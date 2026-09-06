# KakaoTalk PC App Skills

Agent guidance for operating KakaoTalk on macOS. Deterministic CLIs first,
screen automation only where nothing else reaches.

## Architecture

Three execution surfaces, ordered by cost:

| Surface | Handles | Notes |
| --- | --- | --- |
| Kakao official REST API | Calendar events, 할 일, 카카오페이 결제 | TalkCalendar has full CRUD. Always prefer this over the GUI |
| [`kmsg`](https://github.com/channprj/kmsg) | Chat list, read, send text, send image, realtime watch | Accessibility-based CLI with a native stdio MCP server |
| [`katok`](https://github.com/NomaDamas/katok) | History search (keyword, BM25), image extraction, bulk export | Local archive of the KakaoTalk database. Read-only |
| Computer Use | Emoticons, `보이스톡`/`페이스톡`, `메시지 예약`, `채팅방 서랍`, P2P 송금 | Only path for features with no API or CLI |

P2P 송금 (person-to-person transfer) has no API. 카카오페이 결제 (merchant
charge) does. They are different capabilities.

Earlier versions of this skill routed everything through Computer Use. Most of
what agents actually need — listing, reading, searching, sending — is faster and
more reliable through the CLIs, and history search is not reachable by screen
automation at all. Computer Use is now the fallback, not the default.

## Requirements

- Apple Silicon Mac (a `katok` constraint; `kmsg` is universal)
- KakaoTalk for macOS, installed and logged in
- Accessibility permission for `kmsg`
- Full Disk Access for `katok`

Permissions are granted to the terminal application running the agent, in
System Settings. A CLI cannot grant itself macOS TCC permission.

## Install

```bash
brew tap channprj/tap && brew trust channprj/tap && brew install kmsg
brew tap NomaDamas/katok https://github.com/NomaDamas/katok.git \
  && brew trust nomadamas/katok && brew install katok
```

Homebrew 6.x refuses untrusted third-party taps, hence `brew trust`. Inspect
both formulae before trusting them.

Verify:

```bash
kmsg status
katok doctor --json
```

## Supported Agents

- Codex: use `kakaotalk-pc/` as a Codex skill folder.
- Claude Code: root `CLAUDE.md` is loaded as project memory.
  See <https://docs.claude.com/en/docs/claude-code/memory>.
- Any MCP client: `kmsg mcp-server` exposes read, send text, and send image over
  stdio, which covers Tier 1 without shelling out.

## Known Limitations

- `kmsg` reads only what the UI currently exposes; it is not a history tool.
- `katok` cannot send anything.
- `katok sync` is slow. A ~300k message archive took roughly 23 minutes.
- `katok search semantic` is currently broken (`local embedder unavailable`).
  Use `bm25`, which is built during sync and works.
- `katok media get` extracts photos and album frames only. No video.
- Open chat room titles are absent from the `katok` archive, which labels group
  rooms by member nicknames. Titles come from `kmsg chats`.
- Windows is out of scope. The chat database there is separately encrypted and
  the client is packed with Themida, so the local-database path does not
  transfer. Use KakaoTalk's own 대화 내보내기 (`.txt`) instead.

## Safety

KakaoTalk actions communicate with third parties and handle sensitive data. The
guidance requires action-time confirmation before sending messages, images, or
files, making calls, scheduling messages, creating calendar items, inviting
people, changing room settings, leaving rooms, or sending money. A faster tool is
not a lower bar — `kmsg send` requires the same confirmation as a GUI send.

Group and open chat rooms contain messages from people who never consented to
processing. The guidance prefers targeted search over bulk extraction and keeps
third-party content out of model context where a narrower query would answer the
question.

This repository stores no private chat content, real contacts, phone numbers,
payment details, or personal screenshots.

Relevant published terms and the resulting tradeoffs are summarized in
`kakaotalk-pc/SKILL.md` under Privacy And Risk. Keep this tooling on personal
accounts and personal machines.
