# 8Secon API reference

Replace `BASE_URL` with the deployed 8Secon backend URL. Local development uses `http://localhost:3000`; do not assume that value in production.

Before coding, the 8Secon user must sign in at [8secon.com](https://8secon.com) and create an OAuth App from their authenticated session. Configure the external app's exact redirect URI and grant `video.upload`. Keep the returned `clientId` and one-time `clientSecret` in environment variables or a secret manager; never commit or paste them into source code.

## Discovery

```http
GET BASE_URL/.well-known/openid-configuration
```

The current server advertises Authorization Code and refresh-token grants, scopes `openid`, `profile`, `email`, and `video.upload`, and PKCE method `S256`.

## Register an app

This user-authenticated 8Secon management API is called after the user signs in:

```http
POST BASE_URL/api/oauth/clients
Authorization: Bearer <8secon_first_party_access_token>
Content-Type: application/json
```

```json
{
  "name": "Example Video App",
  "homepageUrl": "https://example.com",
  "redirectUris": ["https://example.com/oauth/callback"],
  "scopes": ["openid", "profile", "email", "video.upload"]
}
```

The response contains a `clientId` and one-time `clientSecret`. Store the secret immediately. Client management endpoints are under `/api/oauth/clients`.

## Authorize with PKCE

Redirect the user to `/oauth/authorize` with `client_id`, exact `redirect_uri`, `response_type=code`, `scope=openid profile email video.upload`, a random `state`, `code_challenge`, and `code_challenge_method=S256`. Validate `state` in the callback before using the returned code.

## Exchange the code

```http
POST BASE_URL/oauth/token
Content-Type: application/x-www-form-urlencoded
Authorization: Basic <base64(clientId:clientSecret)>
```

Send `grant_type=authorization_code`, `code`, exact `redirect_uri`, and the original `code_verifier`. Access tokens currently last about 15 minutes; refresh tokens about 30 days and are rotated on use.

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
  "categoryId": 1,
  "visibility": "private"
}
```

The response contains `data.video` and `data.upload`, including `videoId`, `guid`, `libraryId`, `uploadUrl`, `authorizationSignature`, and `authorizationExpire`. Upload the file to the returned TUS endpoint using those values. The 8Secon init endpoint does not receive the file bytes.

## Current boundary

`POST /api/video/upload`, `POST /api/video/upload-chunk`, `PUT /api/video/:id`, and first-party status routes use the internal JWT middleware and are not the external OAuth contract. A complete external publish workflow requires additional OAuth-protected status, metadata, and publish endpoints.
