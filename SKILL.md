---
name: 8secon-api-upload
description: Integrate an external application with the 8Secon OAuth API and direct video-upload initialization flow when building or debugging an app that needs a user's authorization to upload videos to their 8Secon channel.
license: MIT
---

# 8Secon external video upload

Use the API documented in [references/api.md](references/api.md). Keep OAuth credentials and user tokens in the external app's secret store; never place them in source code, logs, prompts, or this skill package.

Use `https://8secon.com` as the public API base URL and OAuth issuer. Discover OAuth endpoints from `https://8secon.com/.well-known/openid-configuration`. Require HTTPS in configuration: posting a token request to `http://8secon.com` can redirect to HTTPS and turn the POST into a GET/HTML response. Do not use, expose, or derive internal service hostnames.

## Prerequisites

Before implementing or testing, tell the user to sign in and create an OAuth App at [8secon.com/oauth/apps](https://8secon.com/oauth/apps). They must configure:

- an exact callback/redirect URI for the external app;
- the `video.upload` scope (plus only the identity scopes the app needs);
- a `client_id` and `client_secret` stored as environment variables in the external app.

If these values are not available, stop at configuration guidance and do not invent credentials, request a user's secret in chat, or reuse the first-party `ac` cookie/internal JWT. The app creation and credential rotation details are in [references/api.md](references/api.md).

## Required workflow

1. Sign in and register an OAuth App at [8secon.com/oauth/apps](https://8secon.com/oauth/apps), with an exact redirect URI. HTTP is acceptable only for a loopback localhost callback during development; use HTTPS in production.
2. Request Authorization Code + PKCE (`S256`) and the `video.upload` scope.
3. Exchange the authorization code for an access token and refresh token.
4. Call `POST /api/external/videos/upload/init` with the OAuth bearer token.
5. Upload the bytes to the returned TUS `uploadUrl` using the returned upload identifier, library ID, signature, and expiry.
6. Treat the returned `videoId` as the 8Secon draft identifier and persist it with the external upload job.

Only an OAuth access token issued by `https://8secon.com/oauth/token` is valid for the external upload API. A login token issued by the external application, or an 8Secon website-session token, is not a substitute. Do not send the video file to the 8Secon init endpoint; it only creates the draft and returns a short-lived direct-upload authorization.

## Important current limitation

The current external surface exposes upload initialization only. The normal metadata update, upload-status, and publish endpoints are protected by the first-party JWT middleware. If the external app must publish a video publicly or monitor processing without a first-party session, stop and report that the 8Secon backend needs dedicated external status/publish endpoints; do not silently substitute internal credentials.

## Implementation expectations

- Generate and verify `state` and PKCE `code_verifier` in the external app.
- Validate the callback's `state`, exact `redirect_uri`, token expiry, and refresh-token errors.
- Treat authorization codes as one-time values. Start a new authorization request after any failed exchange; never replay a code.
- Request only the scopes needed by the app.
- Persist the init response before transferring bytes. Do not retry a successful init call blindly because every successful call creates another draft; resume or retry the TUS transfer for that `videoId` instead.
- Redact `client_secret`, access tokens, refresh tokens, authorization codes, and upload signatures from logs.
- Verify the API's discovery document before integration tests and keep the configured base URL equal to `https://8secon.com`.
