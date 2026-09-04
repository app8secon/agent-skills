# 8Secon API reference

Use this canonical public base URL for every API and OAuth request:

```text
BASE_URL=https://8secon.com
```

Discover OAuth endpoints from `BASE_URL/.well-known/openid-configuration`. Do not use or expose internal service hostnames in application code, configuration, documentation, or logs.

Keep `BASE_URL` and the configured issuer on HTTPS. Do not configure `http://8secon.com`: an HTTP-to-HTTPS redirect can change a token POST into a GET and return HTML rather than OAuth JSON.

Before coding, the 8Secon user must sign in at [8secon.com/oauth/apps](https://8secon.com/oauth/apps) and create an OAuth App. Configure the external app's exact redirect URI and grant `video.upload`. Keep the returned `clientId` and one-time `clientSecret` in environment variables or a secret manager; never commit or paste them into source code.

## Discovery

```http
GET BASE_URL/.well-known/openid-configuration
```

The current server advertises Authorization Code and refresh-token grants, scopes `openid`, `profile`, `email`, and `video.upload`, and PKCE method `S256`.

## Register an app

Create and manage OAuth Apps in the authenticated web interface at [8secon.com/oauth/apps](https://8secon.com/oauth/apps). Save the one-time client secret immediately. The app-management API is part of the website session and is not an external Bearer-token API.

## Authorize with PKCE

Redirect the user to `/oauth/authorize` with `client_id`, exact `redirect_uri`, `response_type=code`, `scope=openid profile email video.upload`, a random `state`, `code_challenge`, and `code_challenge_method=S256`. Validate `state` in the callback before using the returned code. The redirect URI must exactly match a URI registered on the OAuth App, including scheme, host, port, path, and trailing slash.

Use HTTPS callbacks in production. A loopback HTTP callback such as `http://localhost:3422/api/auth/8secon/callback` is suitable only for local development.

## Exchange the code

```http
POST BASE_URL/oauth/token
Content-Type: application/x-www-form-urlencoded
Authorization: Basic <base64(clientId:clientSecret)>
```

Send `grant_type=authorization_code`, `code`, exact `redirect_uri`, and the original `code_verifier`. Access tokens currently last about 15 minutes; refresh tokens about 30 days and are rotated on use.

Use the returned `access_token` for 8Secon external APIs. Do not send a JWT or login token issued by the integrating application; 8Secon validates access tokens created by this token endpoint.

The successful token response is standard OAuth JSON with top-level `access_token`, `token_type`, `expires_in`, `refresh_token`, `scope`, and `id_token`. An authorization code is single-use. If the exchange fails after the code was consumed, restart authorization instead of replaying it.

## Initialize a direct upload

```http
POST BASE_URL/api/external/videos/upload/init
Authorization: Bearer <oauth_access_token>
Content-Type: application/json
```

The access token must include `video.upload`.

```json
{
  "title": "Example video",
  "fileName": "example.mp4",
  "fileSize": 10485760,
  "contentType": "video/mp4",
  "description": "Optional description",
  "visibility": "private"
}
```

The response contains `data.video` and `data.upload`. Persist `data.upload` before transferring the file. Its required fields are `videoId`, `guid`, `libraryId`, `uploadUrl`, `authorizationSignature`, and `authorizationExpire`.

## Upload with TUS

Create a TUS upload against the returned `uploadUrl`. Set the total upload size, provide ordinary filename/content-type metadata, and include these request headers on the TUS create and upload requests:

```text
AuthorizationSignature: <authorizationSignature>
AuthorizationExpire: <authorizationExpire>
LibraryId: <libraryId>
VideoId: <guid>
```

With `tus-js-client`, use `uploadUrl` as `endpoint`, set `uploadSize` when uploading a Node stream, keep `uploadDataDuringCreation: false`, and use resumable chunks. Treat `onSuccess` as completion of the byte transfer. Never send the file bytes to the 8Secon init endpoint.

Each successful init request creates a private draft. If transfer fails after init, reuse the saved fields and resume/retry that TUS upload. Do not call init again unless the short-lived upload authorization expired or the user explicitly chooses to create a new draft.

## Verification checklist

- Discovery reports issuer `https://8secon.com` and PKCE method `S256`.
- Callback `state` matches and the code is exchanged once with the original verifier.
- Token scope contains `video.upload`.
- Init returns a nonzero `videoId` and every required upload field.
- TUS reports all bytes uploaded.
- The new item appears in the user's 8Secon channel as a private draft.

## Current boundary

`POST /api/video/upload`, `POST /api/video/upload-chunk`, `PUT /api/video/:id`, and first-party status routes use the internal JWT middleware and are not the external OAuth contract. A complete external publish workflow requires additional OAuth-protected status, metadata, and publish endpoints.
