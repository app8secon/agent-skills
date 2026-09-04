---
name: 8secon-api-upload
description: Integrate an external application with the 8Secon OAuth API and direct video-upload initialization flow when building or debugging an app that needs a user's authorization to upload videos to their 8Secon channel.
---

# 8Secon external video upload

Use the API documented in [references/api.md](references/api.md). Keep OAuth credentials and user tokens in the external app's secret store; never place them in source code, logs, prompts, or this skill package.

Use `https://8secon.com` as the public API base URL and OAuth issuer. Discover OAuth endpoints from `https://8secon.com/.well-known/openid-configuration`. Do not use, expose, or derive internal service hostnames.

## Prerequisites

Before implementing or testing, tell the user to sign in at [8secon.com](https://8secon.com) and create an OAuth App for the external application. They must configure:

- an exact callback/redirect URI for the external app;
- the `video.upload` scope (plus only the identity scopes the app needs);
- a `client_id` and `client_secret` stored as environment variables in the external app.

If these values are not available, stop at configuration guidance and do not invent credentials, request a user's secret in chat, or reuse the first-party `ac` cookie/internal JWT. The app creation and credential rotation details are in [references/api.md](references/api.md).

## Required workflow

1. Sign in at [8secon.com](https://8secon.com) and register an OAuth App for the 8Secon user, with an exact HTTPS redirect URI in production.
2. Request Authorization Code + PKCE (`S256`) and the `video.upload` scope.
3. Exchange the authorization code for an access token and refresh token.
4. Call `POST /api/external/videos/upload/init` with the OAuth bearer token.
5. Upload the bytes to the returned TUS `uploadUrl` using the returned upload identifier, library ID, signature, and expiry.
6. Treat the returned `videoId` as the 8Secon draft identifier and persist it with the external upload job.

Do not use the first-party `ac` cookie or the internal HS256 JWT for an external integration. Do not send the video file to the 8Secon init endpoint; it only creates the draft and returns a short-lived direct-upload authorization.

## Important current limitation

The current external surface exposes upload initialization only. The normal metadata update, upload-status, and publish endpoints are protected by the first-party JWT middleware. If the external app must publish a video publicly or monitor processing without a first-party session, stop and report that the 8Secon backend needs dedicated external status/publish endpoints; do not silently substitute internal credentials.

## Implementation expectations

- Generate and verify `state` and PKCE `code_verifier` in the external app.
- Validate the callback's `state`, exact `redirect_uri`, token expiry, and refresh-token errors.
- Request only the scopes needed by the app.
- Use idempotent external upload jobs keyed by the 8Secon `videoId` and returned upload identifier.
- Redact `client_secret`, access tokens, refresh tokens, authorization codes, and upload signatures from logs.
- Verify the API's discovery document before integration tests and keep the configured base URL equal to `https://8secon.com`.
