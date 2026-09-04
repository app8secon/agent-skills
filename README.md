# 8Secon API Upload Agent Skill

[![skills.sh](https://skills.sh/b/app8secon/agent-skills)](https://skills.sh/app8secon/agent-skills)

An agent skill for integrating external applications with the **8Secon video platform API** using **OAuth 2.0 Authorization Code + PKCE** and secure, resumable **TUS video uploads**.

Use this skill with coding agents such as Codex, Claude Code, Cursor, GitHub Copilot, Gemini CLI, Windsurf, and other agents that support the Agent Skills format.

All public API and OAuth requests use the canonical base URL `https://8secon.com`. Integrations should discover OAuth endpoints from `https://8secon.com/.well-known/openid-configuration` and must not depend on internal service hostnames.

## What this skill helps with

- Registering an OAuth application in 8Secon.
- Implementing Authorization Code flow with PKCE (`S256`).
- Requesting the least-privilege `video.upload` scope.
- Exchanging and refreshing OAuth access tokens safely.
- Initializing an external video upload through the 8Secon API.
- Uploading video bytes directly with the resumable TUS endpoint returned by 8Secon.
- Protecting client secrets, access tokens, refresh tokens, and upload signatures.
- Avoiding unsupported first-party cookies and internal JWT authentication.

## Install

Install from GitHub with the Skills CLI:

```bash
npx skills add app8secon/agent-skills --skill 8secon-api-upload
```

Then invoke it in a supported agent:

```text
Use $8secon-api-upload to integrate my application with the 8Secon OAuth and video upload APIs.
```

## Integration flow

```text
User creates an OAuth App in 8Secon
        ↓
External app starts Authorization Code + PKCE
        ↓
External app receives an OAuth access token with video.upload
        ↓
POST https://8secon.com/api/external/videos/upload/init
        ↓
Client uploads the video directly through the returned TUS endpoint
```

The video file does not pass through the 8Secon upload-initialization endpoint. The API creates a draft and returns short-lived authorization values for a direct upload, reducing bandwidth and memory usage on the application server.

## Before using the skill

The 8Secon user must first sign in at [8secon.com](https://8secon.com), create an OAuth App, and configure:

- the application's exact callback URL;
- the `video.upload` scope;
- a `client_id` and one-time `client_secret` stored in environment variables or a secret manager.

Never commit OAuth credentials, user tokens, upload signatures, or first-party session cookies.

## Current API boundary

The current external OAuth API supports upload initialization. External status polling, metadata updates, and public publishing still require dedicated OAuth-protected endpoints. The skill reports this boundary instead of attempting to reuse internal 8Secon credentials.

See [`references/api.md`](references/api.md) for endpoint details and implementation requirements.

## Repository structure

```text
agent-skills/
├── SKILL.md
├── README.md
├── agents/
│   └── openai.yaml
└── references/
    └── api.md
```

## About 8Secon

8Secon is a video platform for publishing and distributing creator content. This repository provides the official agent instructions for building secure integrations with the 8Secon API.

Keywords: 8Secon API, video upload API, OAuth 2.0 PKCE, TUS resumable upload, direct video upload, AI agent skill, Codex skill, Claude Code skill, Cursor skill.
