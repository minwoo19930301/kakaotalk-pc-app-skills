# KakaoTalk PC App Guidance

Use this project as operational guidance for controlling KakaoTalk PC on macOS. Keep the work narrow, privacy-aware, and based on visible UI state.

## Scope

- Target app: KakaoTalk PC for macOS.
- Bundle identifier when available: `com.kakao.KakaoTalkMac`.
- Primary knowledge source: `kakaotalk-pc/SKILL.md`.
- Use only the user-designated test chat for exploratory UI testing.
- Do not test workflows in unrelated chats.

## Default Workflow

1. Inspect the current KakaoTalk app state before taking action.
2. Identify whether the screen is the main chat list or an open chat room.
3. Prefer accessibility labels, visible button text, and stable UI labels over pixel coordinates.
4. Do the minimum requested action.
5. Stop and report the result.

## Known UI Patterns

- Main chat list window is usually titled `카카오톡`.
- Open chat window title is usually the chat name.
- Chat tab button can appear as `채팅 (⌘2)`.
- The chat row text usually includes chat name and latest timestamp.
- Clicking a chat row may only select it; pressing `Return` after selection can open the chat.
- The message input usually appears as `텍스트 엔트리 영역 ... 메시지 입력`.
- Pressing `Return` from the focused message input can send.
- Chat room buttons can include `검색`, `보이스톡`, `페이스톡`, `메뉴`, `추가 기능`, `이모티콘 ⌘E`, and `파일전송 ⌘O`.
- `추가 기능` can expose `대화 캡처`, `메시지 예약`, `맞춤법/번역`, `일정`, `할 일`, and `미니게임`.
- `메뉴` can expose `대화상대 초대하기`, `채팅방 서랍`, `톡게시판`, `브리핑 보드`, `챗봇 (beta)`, `알림`, `즐겨찾기`, `항상 위에 유지`, `채팅방 설정`, and `채팅방 나가기`.

## Confirmation Rules

Ask for explicit confirmation immediately before actions that affect other people, cloud state, or user data. Confirm the exact recipient and payload where relevant.

Require confirmation before:

- Sending a text message, link, emoticon, image, or file
- Starting `보이스톡` or `페이스톡`
- Scheduling `메시지 예약`
- Creating `일정` or `할 일`
- Inviting people to a chat
- Downloading, saving, forwarding, or uploading chat content
- Changing chat settings or notification state
- Leaving a chat room
- Initiating or completing 송금

Never complete passwords, OTPs, payment confirmations, or final financial steps. Ask the user to take over when those appear.

## Privacy Rules

- Do not quote, summarize, or analyze private chat content unless the user asks for that specific content.
- Avoid mentioning unrelated visible chat names or message snippets in final responses.
- Do not save screenshots or profile photos unless explicitly requested and safe.
- If a request would transmit sensitive data, confirm the specific data and destination before typing it into KakaoTalk.

## Updating This Guidance

When a new KakaoTalk PC behavior is observed, update `kakaotalk-pc/SKILL.md` first. Keep this `CLAUDE.md` as a concise Claude Code entrypoint that points to the shared skill notes.
