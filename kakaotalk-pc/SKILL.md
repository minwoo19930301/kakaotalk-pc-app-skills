---
name: kakaotalk-pc
description: Operate KakaoTalk on macOS. Use when the user asks to list chats, read messages, search past conversations, extract chat images, send a message or image after required confirmation, watch a chat in realtime, or drive KakaoTalk PC GUI-only features such as emoticons, voice/face calls, scheduled messages, calendar items, the chat drawer, or 송금. Prefers deterministic CLIs (kmsg, katok) over screen automation, and falls back to Computer Use only for GUI-only features.
---

# KakaoTalk on macOS

## Overview

Three execution surfaces. Pick the cheapest one that can do the job. Screen
automation is the last resort, not the default.

| Need | Use | Why |
| --- | --- | --- |
| Chat list **with room titles** | `kmsg chats` | Only surface that exposes real open chat titles |
| Chat list with ids and types | `katok source chats` | Covers the whole archive, not just recent rooms. No titles |
| Read recent messages | `kmsg read` | JSON, marks own messages as `(me)` |
| Send text or image | `kmsg send` / `kmsg send-image` | One command, verifiable result |
| Watch a chat live | `kmsg watch` | Event stream, no polling |
| Search past conversations | `katok search bm25` | Full local archive. Computer Use cannot do this at all |
| Extract chat images | `katok media get` | Decrypts local cache |
| Bulk export a room | `katok` archive SQLite | Whole history in one query |
| Calendar events, 할 일 | **Kakao TalkCalendar REST API** | Full CRUD. Do not click through the GUI for this |
| Emoticons, calls, 예약 메시지, 채팅방 서랍, P2P 송금 | Computer Use | No API or CLI exposes these |

Rule of thumb: if the task is list / read / search / send / watch, do not touch
Computer Use. Everything else, use the GUI patterns in
[GUI-Only Features](#gui-only-features).

## Preflight

Run once per session before acting. Do not proceed if a required surface is
missing; report what is missing instead.

```bash
kmsg status          # Accessibility permission, KakaoTalk running, auth ready
katok doctor --json  # archive freshness, last sync/index
```

`kmsg status` must report Accessibility granted and KakaoTalk running.
`katok doctor` reports `freshness.last_sync`; if it is `null`, the archive is
empty and history search is unavailable until a sync runs.

### Install

```bash
brew tap channprj/tap && brew trust channprj/tap && brew install kmsg
brew tap NomaDamas/katok https://github.com/NomaDamas/katok.git \
  && brew trust nomadamas/katok && brew install katok
```

Homebrew 6.x refuses untrusted third-party taps, so `brew trust` is required.
`kmsg` installs as a prebuilt universal binary in about a second. `katok`
compiles from Rust and takes a couple of minutes.

### Permissions

Grant these to the terminal application that runs the agent, not to the agent
itself. macOS TCC requires the user to do this in System Settings; a CLI cannot
grant itself permission.

| Tool | Permission | Needed for |
| --- | --- | --- |
| `kmsg` | Privacy & Security > Accessibility | Reading and driving the KakaoTalk UI |
| `katok` | Privacy & Security > Full Disk Access | Reading the KakaoTalk container |

`katok permissions macos` opens the right settings pane.

## Tier 0 — Official REST API

Cheapest and safest surface where it exists. Prefer it over every other tier.

### TalkCalendar

Full CRUD for calendars, events, and tasks. Requires Kakao Login, the
`톡캘린더 및 일정 생성, 조회, 편집/삭제` consent item, and a 사용 권한 신청.
Events are on `v2`, tasks on `v1`.

```bash
# 일정
POST   /v2/api/calendar/create/event      # title, time, rrule, location, reminders, color
GET    /v2/api/calendar/events            # preset=TODAY|THIS_WEEK|THIS_MONTH, or from/to
GET    /v2/api/calendar/event             # event_id
POST   /v2/api/calendar/update/event/host # recur_update_type=ALL|THIS|THIS_AND_FOLLOWING
DELETE /v2/api/calendar/delete/event

# 캘린더
GET    /v2/api/calendar/calendars         # filter=USER|SUBSCRIBE|ALL, primary is fixed id
POST   /v2/api/calendar/create/calendar
GET    /v2/api/calendar/holidays          # admin key

# 할 일
POST   /v1/api/calendar/create/task       # content, due_date(yyyyMMdd), alarm_time(HHmm), recur
GET    /v1/api/calendar/tasks             # task_status=TODO|COMPLETED|ALL
POST   /v1/api/calendar/complete/task
DELETE /v1/api/calendar/delete/task
```

Base host `https://kapi.kakao.com`, `Authorization: Bearer ${ACCESS_TOKEN}`.

Notes that bite:

- Times are UTC in RFC5545 `DATE-TIME` form, settable at 5-minute granularity.
  When `all_day` is true, `start_at`/`end_at` must be `YYYY-MM-DDT00:00:00Z`
  exactly or the request errors.
- `to` must be within 31 days of `from` on list endpoints.
- `reminders` allows at most 2 values, in minutes, at 5-minute steps.
- `rrule` follows RFC5545. Omit it for a one-off event.
- `lunar` supports Korean lunar dates.
- Only `primary` or sub-calendars your own service created can be written to.
- 공개 일정 endpoints need an admin key and a linked 카카오톡 채널, so they are
  service-level, not personal.

Creating or editing an event can notify participants and modify cloud data, so
the confirmation policy still applies: confirm title, date, time, timezone,
location, reminders, and recurrence before the call.

### Payments vs. transfers

Keep these separate; they are not the same capability.

| Action | Surface |
| --- | --- |
| 카카오페이 결제 (charge, subscription, cancel, order lookup) | `kakaopay-agent-payments` MCP — `pay_ready`, `approve`, `cancel`, `order`. Defaults to mock mode with no key; test CIDs `TC0ONETIME` (single) and `TCSUBSCRIP` (recurring) work without a merchant contract |
| P2P 송금 (sending money to a person in a chat) | **No API exists.** Computer Use only, and the final authentication step must be handed to the user |

Never treat a payment API as a substitute for 송금, and never complete an OTP,
password, or final payment step on the user's behalf in either path.

## Tier 1 — kmsg

Use for anything involving the live client. Prefer `--json` so output is
parseable. Prefer `--chat-id` over a name once the id is known; name matching is
substring-based and can hit the wrong room.

```bash
kmsg chats --json
kmsg read --chat-id <chat-id> --limit 20 --json
kmsg read "<chat name>" --limit 20 --json --background-safe
kmsg send "<chat name>" "<message>"
kmsg send-image "<chat name>" "/absolute/path/image.png"
kmsg watch "<chat name>" --json
kmsg mcp-server                    # stdio MCP: read, send text, send image
```

Notes:

- `--background-safe` reads only already-exposed chat windows. It will not
  launch, activate, search, open rows, resize, or close windows. Use it whenever
  the user is working on screen, so their windows are not disturbed.
- Without `--background-safe`, `kmsg read` and `kmsg send` may open a chat window
  via search and close it afterward. That is normal and reported in the output.
- `send-image` has no dry-run. It requires an explicit recipient and an explicit
  image path, and it sends immediately. Confirm before calling it.
- In `read` output, author `(me)` means the message was sent by the account owner.
- `kmsg cache status` shows AX path cache state. It is read-only — there is no
  clear subcommand. kmsg maintains a self-healing path cache, so after a
  KakaoTalk update check the status and retry rather than trying to reset it.
- `kmsg mcp-server` can be registered directly as an MCP server. When it is
  registered, use its tools instead of shelling out.

`kmsg` reads what the UI currently exposes. It is not a history tool. For
anything older than the visible transcript, use Tier 2.

## Tier 2 — katok

Local archive of the KakaoTalk SQLCipher database, with keyword and BM25 search.
Read-only; it cannot send anything.

```bash
katok sync --source macos --json      # build/refresh the archive
katok source chats --source macos --json
katok search keyword "<term>" --json
katok search bm25 "<phrase>" --json
katok chunk get <chunk-id> --json     # full text of one result
katok chunk context <chunk-id> --json # surrounding conversation
katok chunk parent <chunk-id> --json  # the ~5 minute parent window
katok chunks --chat <chat-id> --json
katok media get --chat <chat-id> --no-cdn --json
```

`--source macos` is required on `sync` and `source chats`. The default adapter is
`fixture`, which fails with `fixture source requires a JSONL path`.

Use `chunk context` or `chunk parent` instead of dumping a room when a search hit
needs surrounding conversation to make sense.

### Cost and current limitations

State these to the user before starting a long operation rather than blocking
silently.

- `sync` is slow. A ~300k message archive took about 23 minutes and produced
  roughly 210k chunks.
- `search semantic` is currently unreliable. `katok index` can fail with
  `local embedder unavailable: Failed to retrieve onnx/model_q4.onnx`, after
  which semantic search errors with `semantic index has never been synced`.
  Use `bm25` instead; it is built during `sync` via SQLite FTS5 and works
  without the semantic index.
- Apple Silicon only. Intel Macs are unsupported because the local embedding
  path has no prebuilt ONNX Runtime for `x86_64-apple-darwin`.

### Search discipline

Mirror katok's own privacy design. Do not defeat it.

- Return snippets and chunk ids first.
- Call `katok chunk get` only when the user explicitly asks to open a result or
  supplies a chunk id.
- Never dump a whole room's transcript into a response or into a model context
  when a targeted search would answer the question.

### Room titles do not come from katok

`katok source chats` lists every archived room with its `chat_id` and
`chat_type`, but group rooms are named by concatenating member nicknames, so open
chat room titles are absent. To map a room title to a `chat_id`:

1. Search the archive for the room title as a literal string. Bot summaries and
   members often mention it, and the correct room dominates the hit count.
2. If that is inconclusive, `kmsg read` the room by title, then match the
   returned message texts against `messages.text` and take the `chat_id` with
   the most exact matches. Ten sampled messages is enough; a correct match
   typically lands 8–10 of 10.

### Archive schema

`~/Library/Application Support/katok/archive.sqlite3`, readable with SQLite in
read-only mode for aggregation the CLI does not cover.

```
messages(account_hash, chat_id, chat_name, chat_type, message_id,
         sender_id, sender_nickname, timestamp, text, message_type,
         reply_to_message_id)
chats(chat_id, chat_name, chat_type)          -- chat_type: direct | group
chunks(chunk_id, chat_id, chat_name, sender_nickname,
       started_at, ended_at, text, message_count)
chunk_messages, chunks_fts, parent_chunks, reply_edges, sync_cursors
```

Open it as `file:<path>?mode=ro` so the archive cannot be modified.

Observed `message_type` values include `text`, `type_2` (single photo),
`type_27` (album), `type_3`, `type_18`, `type_71`. Presence of `type_2` or
`type_27` in a room means `media get` has images to extract.

### Media extraction

```bash
katok media get --chat <chat-id> --no-cdn --json
```

Default `--no-cdn`. Without it, katok may fetch attachment bytes over a CDN
presigned GET, which turns a local read into a network call. Output lands in
`~/Library/Application Support/katok/media/<chat-id>/` and can be large; one
self-chat with 21 photo messages produced 63 files at about 100 MB. Results are
tiered `full` or `thumb`. Report counts, not file listings, unless asked.

Videos are not supported. `media get` handles single photos and album frames
only. If the user needs video, say so plainly instead of attempting a workaround.

## GUI-Only Features

Nothing below is reachable from `kmsg` or `katok`. Computer Use is the only path,
so the original UI patterns still apply.

Workflow:

1. `mcp__computer_use__.list_apps` if KakaoTalk state is unknown.
2. `mcp__computer_use__.get_app_state` for `com.kakao.KakaoTalkMac`, once per
   assistant turn before interacting.
3. Prefer accessibility element indexes over pixel coordinates.
4. Do the minimum action requested, then stop and report.

If the user designates a specific test chat, explore only in that chat.

### Observed UI patterns

- Main chat list window is usually titled `카카오톡`.
- An open chat window title is usually the chat name.
- Chat tab button appears as `채팅 (⌘2)`; search as `검색 ⌘F`.
- A chat row's text includes the chat name and latest timestamp. Clicking it may
  only select it; press `Return` to open.
- The message input appears as
  `텍스트 엔트리 영역 (settable, string) 메시지 입력`. It can be set directly by
  accessibility value, and `Return` from the focused input sends.
- Chat room buttons: `보이스톡`, `페이스톡`, `메뉴`, `추가 기능`,
  `이모티콘 ⌘E`, `파일전송 ⌘O`.
- `추가 기능` exposes `대화 캡처`, `메시지 예약`, `맞춤법/번역`, `일정`,
  `할 일`, `미니게임`.
- `메뉴` exposes `대화상대 초대하기`, `채팅방 서랍` (`사진/동영상`, `파일`,
  `링크`), `톡게시판`, `브리핑 보드`, `챗봇 (beta)`, `알림`, `즐겨찾기`,
  `항상 위에 유지`, `채팅방 설정`, `채팅방 나가기`.
- Emoticon panel controls: `이모티콘`, `미니 이모티콘`, `카카오 이모티콘샵`,
  `다음`, `이전`, close.

### Feature notes

**Emoticons.** Opening the panel and inspecting categories is safe. Selecting an
item sends it, which needs confirmation for the exact recipient and item.

**Calls.** Never click `보이스톡` or `페이스톡` during exploration. On request,
open and verify the chat, confirm
`"<chat name>"에게 보이스톡/페이스톡을 걸까요?`, then click. Stop on any
permission prompt or unexpected dialog.

**Files and images.** Prefer `kmsg send-image` for images. Use `파일전송 ⌘O`
only for non-image files or when kmsg fails. Confirm the exact path and
recipient, select only the confirmed file, and confirm again if KakaoTalk shows
a preview or send button.

**Chat drawer.** `채팅방 서랍` browses already-shared `사진/동영상`, `파일`,
`링크`. Use it for browsing on request. For bulk image retrieval prefer
`katok media get`. Do not download, forward, save, or upload without
confirmation.

**Scheduled messages.** `메시지 예약` may live under `추가 기능`, a send-button
dropdown, or the chat menu depending on version. Inspect without finalizing.
Confirm recipient, message, date, time, timezone, and recurrence before the
final step.

**Calendar.** Do not use the GUI for this. `일정` and `할 일` have a complete
official REST API — see [Tier 0](#tier-0--official-rest-api). Reach for
`추가 기능 > 일정` only if the API is unavailable for the account.

**Room-level actions.** Invitation, notification, settings, and
`채팅방 나가기` modify cloud state or affect other participants. Never click
`채팅방 나가기` without an explicit request and action-time confirmation.

**송금.** A financial transaction. Do not initiate or confirm during
exploration. On request, confirm recipient, amount, currency, funding source,
memo, and fees. Never enter passwords or OTPs, and never complete the final
payment step; hand control back to the user.

## Confirmation Policy

Sending a KakaoTalk message is representational communication to a third party.
Confirm at action time unless the user already approved the exact recipient and
exact payload. This applies identically whether the send goes through `kmsg` or
Computer Use — a faster tool is not a lower bar.

```
<chat name> 채팅방에 "<message>" 이라고 전송할까요?
```

Require confirmation before: sending text, links, emoticons, images, or files;
starting `보이스톡`/`페이스톡`; scheduling messages; creating `일정`/`할 일`;
inviting people; downloading, saving, forwarding, or uploading chat content;
changing chat settings or notification state; leaving a room; any 송금.

Never complete passwords, OTPs, payment confirmations, or final financial steps.

After a send, verify: the message appears in the transcript and the input is
empty (Computer Use), or the command reports success (`kmsg`). Report only that
it was sent unless more detail is requested.

## Privacy And Risk

### Handling chat content

- Do not quote, summarize, or analyze private chat content unless the user asks
  for that specific content.
- Do not surface unrelated chat names or snippets in a final response.
- Group and open chat rooms contain messages from people who never consented to
  processing. Prefer targeted search over bulk extraction, and keep third-party
  content out of model context when a narrower query would do.
- Do not save screenshots or profile photos unless explicitly requested.
- Confirm the specific data and destination before typing sensitive data into
  KakaoTalk.
- On a login, permission prompt, CAPTCHA, payment, OTP, password, or OS security
  prompt, follow the active confirmation policy.

### Terms of service

Relevant published terms, so the tradeoff can be stated accurately rather than
guessed:

- 카카오계정 약관 제12조 ①8호 prohibits reverse engineering, source extraction,
  and reproducing or decomposing the service. Protocol-level clients (LOCO) sit
  directly against this. Accessibility automation and local database reads do
  not reverse engineer the protocol.
- 카카오계정 약관 제12조 ①10호 prohibits collecting, storing, or publishing
  other members' personal information. A full local archive of a group room
  touches this clause. Narrow the scope of any sync or export.
- 카카오 운영정책 4항 allows protective measures when an unusual usage
  environment or usage pattern is detected. Automation is not named explicitly,
  so "not explicitly prohibited" is not a defense.

Never attempt server-side login from an unregistered device, and never retry it.
Reported consequences include blocked sub-device login and account sanctions.
Accessibility-based operation does not require it.

Keep this tooling on personal accounts and personal machines. Check company
security policy before using it on a work device.

### Windows

This skill targets macOS. On Windows the chat log database is separately
encrypted and `KakaoTalk.exe` is packed with Themida, so the local-database path
does not transfer. The available route there is KakaoTalk's own
대화 내보내기 (`.txt`) plus an indexer over the exported file.

## Troubleshooting

| Symptom | Action |
| --- | --- |
| `kmsg` read fails after a KakaoTalk update | `kmsg cache status`, then retry; the path cache self-heals |
| Clicking a chat row only highlights it | Press `Return` to open |
| Message input not focused after opening a chat | Set the settable input element value instead of typing blindly |
| KakaoTalk open but not frontmost | `get_app_state` usually raises the app and returns a usable window |
| Transcript scrolled away from bottom | Check the scrollbar value or newest visible rows to verify a send |
| `katok` finds no messages | Confirm `--source macos`; the default adapter is `fixture` |
| `katok search semantic` errors | Expected. Use `bm25` |
| `katok` cannot read the container | Grant Full Disk Access to the terminal app, then `katok doctor --macos-probe --json` |
| Wrong room matched by name | Switch to `--chat-id` from `kmsg chats` |

## Update This Skill

Record new behavior as a short note with the exact UI label, command, or
observed output. When a CLI gains a capability that currently requires Computer
Use, move it up a tier and delete the Computer Use path for it. Keep this file
lean and practical.
