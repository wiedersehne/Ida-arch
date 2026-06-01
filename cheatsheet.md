"""
Cheatsheet Agent - Continuous agent that automatically updates cheatsheets.

Trigger: subscribes to `system.historian.record_saved`; filters for AGENT RESPONSE records.
A 120-second fallback poll recovers events missed across restarts.

Cursor: per-chat `chat.cheatsheet_cursor` column — no global state, no cross-chat races.

Tail design:
  Phase 1 — young chat (total <= tail_n): curated one by one immediately as each arrives.
  Phase 2 — accumulation (total > tail_n, unprocessed <= tail_n): deferred. Curation stops
    until tail_n new records accumulate as unprocessed records in the tail.
  Phase 3 — sliding window (unprocessed > tail_n): the oldest unprocessed has scrolled past
    the tail boundary. Curate it immediately. Each new arrival advances the window by one.
  Idle past IDLE_BYPASS_THRESHOLD flushes remaining tail records in any phase.

Threading:
  Each event spawns a per-chat thread that curates all outside-tail records in one run.
  _dirty_chats: if a new response arrives while the thread is running, the thread
  re-submits itself on exit so no events are dropped.
"""

import logging
from concurrent.futures import ThreadPoolExecutor
from datetime import datetime, timezone, timedelta
from typing import Optional

from app.db.project import ChatRecord, Sender
from app.msgbus.message import MessageType, Message
from app.agents.factory.agent_base import AgentBase, AgentCapabilities, TriggerType
from app.services.cheatsheet.cheatsheet_service import CheatsheetService
from app.services.cheatsheet.cheatsheet_curator_prompt import CURATOR_PROMPT

logger = logging.getLogger(__name__)

DEFAULT_TAIL_WINDOW_SIZE = 6
FALLBACK_POLL_INTERVAL = 120  # seconds
IDLE_BYPASS_THRESHOLD = timedelta(minutes=40)


class CheatsheetAgent(AgentBase):
    """
    Continuous agent that automatically updates cheatsheets as new chat exchanges occur.

    Each event spawns a per-chat thread that curates all outside-tail records in one
    run. A dirty flag re-submits the thread if new responses arrive during processing.
    """

    @classmethod
    def capabilities(cls) -> AgentCapabilities:
        return AgentCapabilities(
            agent_id="cheatsheet_agent",
            name="Cheatsheet Agent",
            trigger_type=TriggerType.CONTINUOUS,
            description="Automatically maintains and updates dynamic cheatsheets for each chat as conversations progress.",
            llm_description="The Cheatsheet Agent is a continuous service that analyzes chat exchanges and maintains curated memory of reusable strategies and heuristics for each conversation.",
        )

    def __init__(self):
        super().__init__()
        self._cheatsheet_service: Optional[CheatsheetService] = None
        self._executor: Optional[ThreadPoolExecutor] = None
        self._active_chats: set[int] = set()
        self._dirty_chats: set[int] = set()
        self._tail_window_size = DEFAULT_TAIL_WINDOW_SIZE

    def init_agent(self):
        if self.project_service is None or self.llm_manager is None:
            raise ValueError(
                "Project service and LLM manager must be initialized. Call set_base_dependencies() first."
            )

        self._cheatsheet_service = CheatsheetService(
            project_service=self.project_service,
            llm_manager=self.llm_manager,
        )
        self._executor = ThreadPoolExecutor(
            max_workers=4, thread_name_prefix="cheatsheet_curate"
        )
        self._tail_window_size = int(
            self.agent_config.get("tail_window_size", DEFAULT_TAIL_WINDOW_SIZE)
        )
        self.message_bus.subscribe_sync(
            topic="system.historian.record_saved",
            client_id="cheatsheet_agent_trigger",
            callback=self._on_new_response,
        )
        logger.info(
            f"CheatsheetAgent initialized (tail_window_size={self._tail_window_size})"
        )

    # ── Event callback ─────────────────────────────────────────────────────

    def _on_new_response(self, message: Message) -> None:
        if not message.metadata:
            return
        if message.metadata.get("sender_role") != Sender.AGENT:
            return
        if message.metadata.get("msg_type") != MessageType.RESPONSE:
            return
        chat_id = message.metadata.get("chat_id")
        project_id = message.metadata.get("project_id")
        if not chat_id or not project_id:
            return

        if chat_id in self._active_chats:
            # Thread already running for this chat; mark for re-run on exit.
            self._dirty_chats.add(chat_id)
            return

        self._active_chats.add(chat_id)
        self._executor.submit(self._process_chat, chat_id, project_id)

    # ── Main loop (fallback poll only) ─────────────────────────────────────

    def _main_loop(self, request_message: Message | None = None):
        logger.info("CheatsheetAgent is running")
        while not self._stop_event.is_set():
            try:
                self._fallback_poll()
            except Exception:
                logger.exception("CheatsheetAgent: fallback poll error")
            self._stop_event.wait(timeout=FALLBACK_POLL_INTERVAL)
        if self._executor:
            self._executor.shutdown(wait=False)
        logger.info("CheatsheetAgent stopped")

    # ── Fallback poll ──────────────────────────────────────────────────────

    def _fallback_poll(self) -> None:
        if not self.project_service or not self._executor:
            return
        chats = self.project_service.get_chats_needing_cheatsheet()
        submitted = 0
        for chat_id, project_id, _ in chats:
            if chat_id in self._active_chats:
                continue
            self._active_chats.add(chat_id)
            self._executor.submit(self._process_chat, chat_id, project_id)
            submitted += 1
        if submitted:
            logger.info(
                f"CheatsheetAgent fallback poll: {submitted}/{len(chats)} chats submitted"
            )

    # ── Per-chat thread ────────────────────────────────────────────────────

    def _process_chat(self, chat_id: int, project_id: int) -> None:
        """Curate all outside-tail records for this chat, then exit.

        If new responses arrived while this thread was running (_dirty_chats),
        re-submit immediately so those records are not lost.
        """
        try:
            while True:
                record = self._get_next_record(chat_id)
                if record is None:
                    break
                try:
                    self._curate_record(chat_id, project_id, record)
                except Exception:
                    logger.exception(
                        f"CheatsheetAgent: error curating record={record.id} chat={chat_id}"
                    )
                    break
        except Exception:
            logger.exception(f"CheatsheetAgent: unexpected error in chat={chat_id}")
        finally:
            self._active_chats.discard(chat_id)
            if chat_id in self._dirty_chats:
                self._dirty_chats.discard(chat_id)
                self._active_chats.add(chat_id)
                self._executor.submit(self._process_chat, chat_id, project_id)

    # ── Tail logic ─────────────────────────────────────────────────────────

    def _get_next_record(self, chat_id: int) -> Optional[ChatRecord]:
        """Return the oldest unprocessed record that is safe to curate, or None.

        Phase 1 — young (total <= tail_n):        curate immediately.
        Phase 2 — accumulating (unprocessed <= tail_n, total > tail_n): defer.
        Phase 3 — sliding window (unprocessed > tail_n): curate oldest immediately.
        Any phase: idle > IDLE_BYPASS_THRESHOLD bypasses the tail and flushes.
        """
        if not self.project_service:
            return None

        tail_n = self._tail_window_size
        cursor_ts = self.project_service.get_cheatsheet_cursor_ts(chat_id)

        # Fetch the oldest tail_n+1 unprocessed records.
        # len == tail_n+1 → Phase 3 (oldest scrolled past tail boundary).
        unprocessed = self.project_service.get_chat_records(
            chat_id=chat_id,
            starting_from_ts=cursor_ts,
            limit=tail_n + 1,
            sender_role_filter=[Sender.AGENT],
            msg_role_filter=[MessageType.RESPONSE],
            oldest_first=True,
        )
        if not unprocessed:
            return None

        record = unprocessed[0]  # oldest unprocessed

        if len(unprocessed) > tail_n:
            # Phase 3: oldest unprocessed has scrolled past the tail boundary.
            return record

        # Fewer than tail_n+1 unprocessed. Check total to distinguish Phase 1 vs 2.
        recent = self.project_service.get_chat_records(
            chat_id=chat_id,
            limit=tail_n + 1,
            sender_role_filter=[Sender.AGENT],
            msg_role_filter=[MessageType.RESPONSE],
            oldest_first=False,
        )

        if len(recent) <= tail_n:
            # Phase 1: young chat — curate immediately.
            return record

        # Phase 2: mature chat, tail not yet full of unprocessed records.
        # Defer unless the chat has gone idle.
        most_recent_ts = recent[-1].timestamp
        if most_recent_ts.tzinfo is None:
            most_recent_ts = most_recent_ts.replace(tzinfo=timezone.utc)
        idle_duration = datetime.now(timezone.utc) - most_recent_ts
        if idle_duration > IDLE_BYPASS_THRESHOLD:
            logger.info(
                f"CheatsheetAgent: chat={chat_id} idle "
                f"{int(idle_duration.total_seconds())}s, flushing tail"
            )
            return record

        logger.info(
            f"CheatsheetAgent: chat={chat_id} record={record.id} accumulating "
            f"({len(unprocessed)}/{tail_n} unprocessed), idle={int(idle_duration.total_seconds())}s, deferring"
        )
        return None

    # ── Curation ───────────────────────────────────────────────────────────

    def _curate_record(self, chat_id: int, project_id: int, record: ChatRecord) -> None:
        """Run the curator LLM on one exchange and advance the cursor."""
        user_query = self._find_preceding_user_query(chat_id, record.id)
        if not user_query:
            logger.info(
                f"CheatsheetAgent: no user query before record={record.id}, skipping"
            )
            self.project_service.update_cheatsheet_cursor(chat_id, record.timestamp)
            return

        agent_response = record.compressed_message or record.message or ""
        if not agent_response:
            logger.info(f"CheatsheetAgent: empty response record={record.id}, skipping")
            self.project_service.update_cheatsheet_cursor(chat_id, record.timestamp)
            return

        logger.info(f"CheatsheetAgent: curating chat={chat_id} record={record.id}")
        try:
            self._cheatsheet_service.update_cheatsheet(
                chat_id=chat_id,
                project_id=project_id,
                user_query=user_query,
                model_answer=agent_response,
                curator_prompt=CURATOR_PROMPT,
                record_id=record.id,
            )
            self.project_service.update_cheatsheet_cursor(chat_id, record.timestamp)
            logger.info(f"CheatsheetAgent: done chat={chat_id} record={record.id}")
        except Exception:
            logger.exception(
                f"CheatsheetAgent: curator failed chat={chat_id} record={record.id}"
            )
            # Advance cursor anyway to avoid getting stuck on a permanently failing record.
            self.project_service.update_cheatsheet_cursor(chat_id, record.timestamp)

    def _find_preceding_user_query(
        self, chat_id: int, before_record_id: int
    ) -> Optional[str]:
        if not self.project_service:
            return None
        records = self.project_service.get_chat_records(
            chat_id=chat_id,
            before_record_id=before_record_id,
            limit=20,
            sender_role_filter=[Sender.USER],
        )
        for record in reversed(records):
            if record.id and record.sender_role == Sender.USER:
                return record.compressed_message or record.message or ""
        return None

    def register_default_prompts(self):
        pass
