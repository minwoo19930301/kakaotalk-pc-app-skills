# KakaoTalk PC App Skills

Practical agent guidance for operating the KakaoTalk PC app on macOS.

This repository starts small and is meant to grow from observed UI patterns. The skill focuses on:

- Opening KakaoTalk PC chats
- Navigating visible chat list and chat windows
- Sending messages only after required confirmation
- Learning safe procedures for attachments, calls, emoticons, scheduled messages, and other KakaoTalk PC actions

The guidance avoids storing private chat content, real contacts, phone numbers, payment details, or personal screenshots.

## Supported Agents

- Codex: use `kakaotalk-pc/` as a Codex skill folder.
- Claude Code: use root `CLAUDE.md` as the project memory/instructions file.

## Safety

KakaoTalk actions often communicate with third parties or handle sensitive data. The guidance requires confirmation before sending messages, making calls, uploading files, scheduling messages, sharing links, sending money, or otherwise transmitting data.

## Notes

Claude Code loads `CLAUDE.md` project memory files from the working directory tree. See Anthropic's Claude Code memory documentation for the current behavior: <https://docs.claude.com/en/docs/claude-code/memory>.
