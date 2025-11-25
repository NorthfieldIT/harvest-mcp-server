# OAuth flows (diagrams)

This file contains simple ASCII diagrams showing:

- The OAuth 2.0 Authorization Code flow used to obtain access tokens from Harvest.
- The runtime flow for an MCP tool (e.g. Claude) calling the Harvest API via this application.

---

1) OAuth 2.0 - Authorization Code flow (high level)

```
             +-----------------+                       +--------------------+
             |     Browser     |                       |    Harvest Auth    |
             |   (End User)    |                       |   (Authorization)  |
             +--------+--------+                       +---------+----------+
                      |                                          |
  1) User clicks      |                                          |
     "Sign in"        |                                          |
     (to /auth/harvest)|                                          |
                      |     2) Redirect to Authorization URL     |
                      +----------------------------------------->|
                      |    (client_id, redirect_uri, scope, etc) |
                      |                                          |
                      |   3) User authenticates & authorizes     |
                      |<-----------------------------------------+
                      |                                          |
  4) Harvest redirects|                                          |
     back to          |   4) Redirect to `OAUTH_REDIRECT_URI`    |
     `/auth/callback` |      with `?code=AUTHORIZATION_CODE`     |
     (contains code)  |<-----------------------------------------+
                      |                                          |
                      |                                          |
             +--------v--------+                       +---------v----------+
             |  MCP Server /   |                       |  Harvest Token     |
             |  OAuth Client   |                       |  Endpoint (Token)  |
             +--------+--------+                       +---------+----------+
                      |                                          |
  5) Server receives  |                                          |
     code at /auth/callback                                          |
                      |   6) Exchange code + client_secret for    |
                      +----------------------------------------->|
                      |             access_token (+refresh)     |
                      |                                          |
                      |   7) Token response (access_token,...)  |
                      |<-----------------------------------------+
                      |                                          |
  8) Server stores    |                                          |
     tokens in session |                                          |
     or issues an      |                                          |
     encrypted JWT     |                                          |
                      |                                          |
```

Notes:
- This application is an OAuth *client* when talking to Harvest — it is not an OAuth provider.
- Client credentials (Client ID and Client Secret) are stored on the machine: `.env` in local dev or in a secret store (Kubernetes Secret, etc.) in production.
- Tokens obtained from Harvest are stored either in session storage (Redis or in-memory) for browser sessions, or encrypted into stateless JWTs for MCP clients.
- The server should perform token refresh when the access token expires (using the refresh token returned by Harvest).

---

2) MCP tool → Harvest API flow (runtime)

```
  +---------+        +------------------+        +-----------------+
  |  Claude |  HTTP  |  MCP HTTP Server |  HTTP  |  Harvest API    |
  |  (MCP)  | -----> |  (this app)      | -----> |  (Resource API) |
  +----+----+        +--------+---------+        +--------+--------+
       |                     |                           |
       | 1) POST /mcp         |                           |
       |    (tool request)    |                           |
       |--------------------->|                           |
       |                     |                           |
       |                     | 2) Authenticate request    |
       |                     |    - check session cookie  |
       |                     |      (Redis / in-memory)   |
       |                     |    OR                     |
       |                     |    - validate stateless JWT|
       |                     |      (contains encrypted   |
       |                     |       Harvest access token)
       |                     |                           |
       |                     | 3) Prepare Harvest API call|
       |                     |    - attach `Authorization: Bearer <access_token>`
       |                     |                           |
       |                     | 4) Call Harvest API         |
       |                     |    (GET/POST to /projects, /time_entries, etc.)
       |                     |--------------------------->|
       |                     |                           |
       |                     | 5) Harvest responds        |
       |                     |<---------------------------|
       | 6) Server formats/   |                           |
       |    maps response     |                           |
       |<-------------------- |                           |
       |                     |                           |
```

Notes:
- The MCP server acts on behalf of an authenticated user: it uses the access token previously obtained during the OAuth flow.
- For MCP clients that are stateless, the server may accept an encrypted JWT that contains the user's Harvest tokens; the server decrypts/validates it and performs API calls.
- Access tokens should never be logged. Refresh tokens should be kept secure and rotated when used.
- In production, use HTTPS end-to-end (clients → MCP server and MCP server → Harvest).

---

If you want different diagram styles (more compact, ASCII art with boxes, or Mermaid diagrams), tell me which style and I can add alternate representations.
