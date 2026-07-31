# KakaoTalk on macOS — Agent Guidance

Operational guidance for driving KakaoTalk on macOS. Keep work narrow,
privacy-aware, and grounded in verifiable tool output rather than guesswork.

Primary knowledge source: `kakaotalk-pc/SKILL.md`. Read it before acting.

## Scope

- Target app: KakaoTalk for macOS, bundle id `com.kakao.KakaoTalkMac`.
- Three execution surfaces: `kmsg` CLI, `katok` CLI, Computer Use.
- Use only the user-designated test chat for exploratory testing.

## Tool Routing

Pick the cheapest surface that can do the job. Screen automation is the last
resort, not the default.

| Need | Surface |
| --- | --- |
| Calendar events, 할 일 | **TalkCalendar REST API** (`kapi.kakao.com/v2/api/calendar/*`, tasks on `v1`) |
| 카카오페이 결제 | `kakaopay-agent-payments` MCP (mock mode needs no key) |
| Chat list, read, send, send image, watch | `kmsg` |
| Search past conversations, extract images, bulk export | `katok` |
| Emoticons, `보이스톡`/`페이스톡`, `메시지 예약`, `채팅방 서랍`, P2P 송금 | Computer Use |

If the task is list / read / search / send / watch, do not touch Computer Use.
If the task is a calendar event or task, use the REST API — never click through
`추가 기능 > 일정`.

P2P 송금 has no API and is Computer Use only; 카카오페이 결제 is a different
capability with an API. Do not conflate them.

## Default Workflow

1. Preflight: `kmsg status` and `katok doctor --json`. Report anything missing
   instead of proceeding.
2. Choose the surface from the routing table.
3. For `kmsg`, prefer `--json` and `--chat-id` over name matching. Use
   `--background-safe` when the user is working on screen.
4. For `katok`, return snippets and chunk ids first; fetch full text only when
   explicitly asked.
5. For Computer Use, inspect app state once per turn and prefer accessibility
   labels over pixel coordinates.
6. Do the minimum requested action, then stop and report.

## Known Constraints

- `kmsg` reads only what the UI currently exposes. It is not a history tool.
- `katok` is read-only and cannot send anything.
- `katok sync` is slow; a ~300k message archive took roughly 23 minutes.
- `katok search semantic` is currently broken. Use `bm25`.
- `katok media get` extracts photos and album frames only. No video.
- Open chat room titles are absent from the katok archive, which labels group
  rooms by member nicknames. Get titles from `kmsg chats` and map by content.
- Apple Silicon only for `katok`.

## Confirmation Rules

Confirm immediately before actions affecting other people, cloud state, or user
data. Confirm the exact recipient and payload. A faster tool is not a lower bar:
`kmsg send` needs the same confirmation as a Computer Use send.

Require confirmation before: sending text, links, emoticons, images, or files;
starting `보이스톡`/`페이스톡`; `메시지 예약`; creating `일정`/`할 일`; inviting
people; downloading, saving, forwarding, or uploading chat content; changing chat
settings or notification state; leaving a chat room; any 송금.

Never complete passwords, OTPs, payment confirmations, or final financial steps.
Hand control back to the user.

## Privacy Rules

- Do not quote, summarize, or analyze private chat content unless asked for that
  specific content.
- Do not surface unrelated chat names or snippets in final responses.
- Group and open chat rooms contain third-party messages from people who never
  consented. Prefer targeted search over bulk extraction.
- Do not save screenshots or profile photos unless explicitly requested.
- Confirm the specific data and destination before typing sensitive data into
  KakaoTalk.

## Risk Notes

- 카카오계정 약관 제12조 ①8호 prohibits reverse engineering. Protocol-level
  clients (LOCO) violate it; accessibility automation and local DB reads do not.
- 카카오계정 약관 제12조 ①10호 prohibits storing other members' personal
  information. Full archives of group rooms touch this. Keep sync scope narrow.
- 카카오 운영정책 4항 permits protective measures on unusual usage patterns, so
  "not explicitly prohibited" is not a defense.
- Never attempt server-side login from an unregistered device. Reported
  consequences include blocked sub-device login and account sanctions.
- Personal accounts and personal machines only. Check company policy on work
  devices.

## Updating This Guidance

Update `kakaotalk-pc/SKILL.md` first. Keep this file a concise entrypoint that
mirrors the routing table and points to the shared skill notes. When a CLI gains
a capability that currently needs Computer Use, move it up a tier.
