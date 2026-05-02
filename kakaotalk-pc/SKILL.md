---
name: kakaotalk-pc
description: Operate the KakaoTalk PC app on macOS with Computer Use. Use when the user asks Codex to open KakaoTalk, find or open a chat room/contact, inspect the visible KakaoTalk UI, type a message, send a message after required confirmation, or perform simple KakaoTalk PC navigation such as switching to the chat list, searching chats, viewing profiles, using emoticons, attaching files, scheduling messages, or learning safe KakaoTalk PC workflows.
---

# KakaoTalk PC

## Overview

Use this skill for practical KakaoTalk PC app control through Computer Use. Keep the workflow narrow, privacy-aware, and based on visible UI state.

## Core Workflow

1. Start with `mcp__computer_use__.list_apps` if KakaoTalk state is unknown.
2. Use `mcp__computer_use__.get_app_state` for `com.kakao.KakaoTalkMac` once per assistant turn before interacting.
3. Identify the visible screen:
   - Main chat list window is usually titled `카카오톡`.
   - Open chat window title is usually the chat name.
   - Chat tab button has help text like `채팅 (⌘2)`.
4. Prefer accessibility element indexes over coordinates.
5. Do the minimum action requested, then stop and report the result.

If the user designates a specific test chat, use only that chat for exploration. Do not test workflows in unrelated rooms.

## Opening A Chat

Use this path when the target chat is already visible in the chat list:

1. Get app state for `com.kakao.KakaoTalkMac`.
2. If the chat list is visible, find the row whose text contains the requested chat name.
3. Click the row, then press `Return` if the row only becomes selected and the room does not open.
4. Verify the new window title matches the requested chat.

Observed working pattern:

- Chat list row text includes the chat name and latest timestamp.
- Clicking the row may only select it.
- Pressing `Return` after row selection can open the chat room.

If the chat is not visible, use the KakaoTalk search button (`검색`, often help text `검색 ⌘F`) or scroll the chat list. Do not read or summarize unrelated chats unless the user explicitly asks.

## Sending Messages

Sending a KakaoTalk message is representational communication to a third party. Confirm at action-time unless the user already gave explicit approval for the exact recipient and exact message.

Confirmation should state the recipient and message plainly:

`<chat name> 채팅방에 "<message>" 이라고 전송할까요?`

After confirmation:

1. Get app state again.
2. Confirm the active window title is the intended chat.
3. Set the message input element value. It usually appears as `텍스트 엔트리 영역 (settable, string) 메시지 입력`.
4. Press `Return` or click the yellow `전송` button.
5. Verify the sent message appears in the transcript and the input field is empty.
6. Report only that it was sent, unless the user asks for more detail.

Observed working pattern:

- The message input can be set directly by accessibility value.
- Pressing `Return` from the focused input can send.
- The transcript then shows a new outgoing yellow bubble with the message and current time.

## Profiles And Photos

Viewing a profile or profile photo is local UI navigation, but it can expose personal information. Keep summaries minimal.

Safe exploration pattern:

1. Open the intended chat.
2. Use `get_app_state` to identify `프로필` buttons or participant controls such as `대화상대`.
3. Click only the intended profile control.
4. Observe available controls and close the profile view without changing settings.

Do not save, screenshot, upload, or describe profile photos unless the user explicitly asks and the action is safe under the active policy.

## Calls

KakaoTalk chat windows can expose call buttons such as `보이스톡` and `페이스톡`.

Do not click these buttons during exploration. Starting a call contacts a third party and requires action-time confirmation for the exact recipient and call type.

If the user asks to call:

1. Open and verify the intended chat.
2. Confirm: `"<chat name>"에게 보이스톡/페이스톡을 걸까요?`
3. Only after confirmation, click the matching button.
4. Stop if a permission prompt, warning, or unexpected dialog appears.

## Emoticons

The chat window usually has an emoticon button labeled `이모티콘 ⌘E`.

Exploration can open the emoticon panel and inspect categories, but sending an emoticon is representational communication and requires confirmation for the exact recipient and selected emoticon.

Safe pattern:

1. Open the intended chat.
2. Click `이모티콘 ⌘E` to open the panel.
3. Identify category or recent items without sending. Observed panel controls include `이모티콘`, `미니 이모티콘`, `카카오 이모티콘샵`, `다음`, `이전`, and a close button.
4. Close the panel or ask for confirmation before selecting an item that sends.

## Files And Images

The chat window usually has a file transfer button labeled `파일전송 ⌘O`, plus an `추가 기능` button.

Uploading or sending files/images transmits user data to the chat. Confirm the specific file path, recipient, and purpose before choosing or sending the file.

Safe pattern:

1. Open and verify the intended chat.
2. Confirm the exact file or image path and recipient.
3. Click `파일전송 ⌘O` or the appropriate attachment control.
4. If a file picker opens, select only the confirmed file.
5. Confirm again if KakaoTalk shows a preview or send button for the upload.

The chat menu can expose `채팅방 서랍` with submenu items `사진/동영상`, `파일`, and `링크`. Use these for browsing already-shared content only when requested; do not download, forward, save, or upload anything without the required confirmation.

## Scheduled Or Reserved Messages

KakaoTalk PC may expose scheduled/reserved message features through `추가 기능`, a send-button dropdown, or a chat menu depending on version. Observed `추가 기능` menu items include `대화 캡처`, `메시지 예약`, `맞춤법/번역`, `일정`, `할 일`, and `미니게임`.

Creating a scheduled message is still communication to a third party. Confirm the recipient, message, scheduled date/time/timezone, and any recurrence before the final scheduling step.

Exploration pattern:

1. Open the intended chat.
2. Inspect `추가 기능`, send-button dropdown, or `메뉴` without finalizing.
3. Record exact labels discovered.
4. Stop before scheduling unless the user confirms the exact message and time.

## Calendar Or Appointments

Calendar/event features may appear under chat menus, plus buttons, or KakaoTalk services. In the observed version, `추가 기능` includes `일정` and `할 일`.

Creating or editing events can notify participants or modify cloud data. Confirm participants, title, date/time/timezone, location, notes, and notification behavior before final creation.

## Chat Menu

The chat room top-right `메뉴` button can expose room-level actions. Observed items include:

- `대화상대 초대하기`
- `채팅방 서랍` with `사진/동영상`, `파일`, `링크`
- `톡게시판`
- `브리핑 보드`
- `챗봇 (beta)`
- `알림`
- `즐겨찾기`
- `항상 위에 유지`
- `채팅방 설정`
- `채팅방 나가기`

Be especially careful with invitation, notification, settings, and leaving actions because they can modify cloud state or affect other participants. Do not click `채팅방 나가기` unless the user explicitly requests it and confirms at action-time.

## Sending Money

Sending money is a financial transaction. Do not initiate or confirm payment during exploration.

If the user asks for 송금:

1. Treat it as a high-risk financial action.
2. Confirm recipient, amount, currency, funding source if visible, memo, and fees before any final transaction.
3. Never enter passwords, OTPs, or complete the final payment step. Ask the user to take over if required by policy or if authentication/payment confirmation appears.

## Privacy And Safety

- Do not quote, summarize, or analyze private chat content unless the user asks for that specific content.
- Avoid exposing unrelated visible chats in the final response.
- Do not send, edit, react, make calls, upload files, schedule messages, create appointments, or share links without the required confirmation.
- If a request would transmit sensitive data, confirm the specific data and destination before typing it into KakaoTalk.
- If a login, permission prompt, CAPTCHA, payment, OTP, password, or OS security prompt appears, follow the active confirmation policy.

## Small Troubleshooting Notes

- If clicking a row only highlights it, press `Return` to open.
- If the input field is not focused after opening a chat, use the settable message input element instead of typing blindly.
- If KakaoTalk is already open but not frontmost, `get_app_state` usually raises the app and returns the usable window.
- If the transcript is scrolled away from the bottom, use the scrollbar value or visible newest rows to verify whether the message sent.

## Update This Skill

When a new KakaoTalk PC pattern is learned, add it here as a short observed note with the exact UI label or element behavior. Keep this file lean and practical.
