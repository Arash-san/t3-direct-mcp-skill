---
name: t3-direct-connect
description: Connect Claude, dots, or another tool-capable agent to T3 Connect over MCP; diagnose remote callback failures, pair securely, and use a direct non-model client.
---

# T3 Direct Connect

Use this for T3 Connect, not an unrelated product named T3. A skill supplies instructions, not network access, credentials, or permission. Never claim the connection works until an authenticated read succeeds.

Compatibility: this recipe was checked against T3's current contract on 2026-11-06. Public references are [outside-agent setup](https://github.com/pingdotgg/t3code/blob/main/docs/user/outside-agents.md) and [authentication contracts](https://github.com/pingdotgg/t3code/blob/main/packages/contracts/src/auth.ts). Those links track main, not a pinned release. Inspect the corresponding files at the deployed tag/commit before relying on version-specific behavior.

## Choose the route

Obtain the user's T3 Connect HTTPS origin from their settings or a verified existing configuration. Do not guess a hostname. Prefer a native MCP connector if available; otherwise use a small Python + curl client directly against ORIGIN/mcp. Do not launch an extra agent turn or call a model API merely to relay MCP requests. Direct inventory reads do not invoke a model; starting or messaging a T3 agent can consume its provider quota. The assistant's own usage is separate.

For Claude Code, the native path is claude mcp add --transport http t3 ORIGIN/mcp, then claude mcp login t3. Its documented --no-browser option supports entering the final return URL in the login CLI, not ordinary chat. Adding a connection still needs any required persistent-access approval.

Read ORIGIN/.well-known/oauth-protected-resource/mcp and ORIGIN/.well-known/oauth-authorization-server. Verify the resource is ORIGIN/mcp, the issuer and authorization server match, and PKCE S256 and bearer headers are supported. Pin the approved origin and verified endpoints. Changed hosts, access denials, TLS warnings, or unexpected redirects are reasons to stop, not to alter identity, disable security, or tunnel around a restriction.

## Pair without a callback listener

A remote agent's 127.0.0.1 is not the user's computer. For T3 versions supporting the pairing-code decision endpoint, use the following supported flow instead of waiting for a remote localhost callback. Confirm the deployed version supports it; do not assume these T3-specific routes exist on every OAuth server.

Creating or expanding persistent access needs the user's confirmation for the server, access mode, lifetime, and storage location. Prefer read-only when sufficient. Full-access scopes do not authorize arbitrary actions. Obtain a fresh pairing code through the service's approved login mechanism or a hidden terminal prompt, never ordinary chat. If hidden input is unavailable, stop.

1. Generate an unpredictable state and PKCE verifier; challenge = base64url(SHA256(verifier)) without padding. Use redirect_uri http://127.0.0.1/callback as a registered return value, not a listening socket.
2. POST JSON to the verified registration_endpoint: {"client_name":"t3-direct-client","redirect_uris":[redirect_uri]}. Retain client_id. The observed path is /oauth/mcp/register; use discovery, not an assumed path.
3. POST JSON to the version-verified /oauth/mcp/decision route: {"authorization":{"response_type":"code","client_id":client_id,"redirect_uri":redirect_uri,"code_challenge":challenge,"code_challenge_method":"S256","state":state,"resource":ORIGIN+"/mcp"},"decision":{"_tag":"pairing-code","access":approved_access,"code":pairing_code}}. Example modes are read-only, approval-required, and full-access; this is not an exhaustive enum.
4. Parse redirectTo locally. Do not open it. Require the exact registered scheme/host/port/path, no fragment or control characters, one matching state, one nonempty code, no error, and the expected issuer when supplied or advertised as required. Reject duplicate security fields.
5. POST form-urlencoded to the verified token_endpoint (observed /oauth/mcp/token): grant_type=authorization_code, code, client_id, redirect_uri, code_verifier, resource. Verify Bearer token type, permitted scopes, and positive expiry within the user's approved lifetime. Never print the response. On mismatch, do not use the token; report that server-side access may already exist.

Scope check: read-only grants orchestration:read; approval-required adds orchestration:operate; full-access additionally grants settings:write and source-control:write. The metadata can advertise only read/operate while full-access legitimately returns all four. Validate against the deployed mcpClientScopes definition and approved mode; do not reject the latter two solely because discovery omitted them, or silently accept unrelated extra scopes.

The observed token expires_in is 2592000 seconds (30 days); this request has no lifetime parameter. Verify the actual expiry and server-side session semantics before promising a duration. Do not retain/use an unexpected refresh token. Expiry is not proof that every server-side grant was deleted. For an orphaned or uncertain grant, inspect approved clients and arrange authorized revocation before re-pairing.

## Protect credentials

Use the host's approved secret store. A local POSIX fallback needs an owner-only directory (0700), exclusively created file (0600), ownership checks, and symlink rejection. Obtain any required storage approval. Never overwrite a credential silently. Keep secrets outside the skill and repository, excluded from backups/uploads where appropriate.

Do not put pairing codes, access tokens, authorization codes, or PKCE verifiers in chat, URLs you navigate, command arguments, logs, screenshots, or artifacts. Pass request secrets through process stdin or an approved secret interface. For curl, place the Authorization header in stdin configuration; disable user curl configuration, retain TLS verification and the normal network policy, and do not follow redirects. Use bounded timeouts and no automatic retries for pairing or actions. If a write times out, inspect state before retrying. Never bypass a denied action through another route.

## Use MCP directly

POST JSON-RPC 2.0 to ORIGIN/mcp with Authorization: Bearer, Content-Type: application/json, and Accept: application/json, text/event-stream. Send initialize with protocolVersion 2025-06-18, empty capabilities, and an honest clientInfo name/version; negotiate a mutually supported version. Retain Mcp-Session-Id when returned, use the negotiated MCP-Protocol-Version on subsequent requests, then send notifications/initialized.

Call tools/list, following its pagination, and inspect current schemas before tools/call. Handle JSON and SSE responses; match response IDs and check both JSON-RPC errors and tool isError. Do not log raw credential-bearing requests or responses.

Prove the connection with t3_project_list, then report the read result, never the token. Useful inventory tools are t3_project_list and t3_thread_list; follow nextCursor. Use recentRuns.completedAt for run end time, not thread.updatedAt. Check run outcome and current thread status: an ended run may have failed, and an older completed run may belong to an active thread. Check pagination/truncation before claiming completeness. Some T3 thread reads acknowledge child-result delivery. Inspect current t3_thread_read semantics before using it as a supposedly passive read. Tool names alone do not prove an operation is harmless.

## Diagnose and report honestly

- Metadata works, authenticated read fails: check expiry, issuer/resource binding, scopes, session, and negotiated protocol. Never reveal the token.
- Localhost callback hangs: use supported callback-free pairing above. Do not invent an externally reachable callback or expose a port.
- 401: diagnose invalid/expired credentials; renewing persistent access requires approval. 403: stop and resolve permissions. Do not repeatedly re-pair.
- Missing tool or rejected argument: refresh schemas and check server version. Do not invent tool names.
- Storage, network, proxy, or sandbox denial: report the exact blocked operation and supported next step. Do not switch identity, proxy, network, or environment to evade it.
- Interrupted pairing/action: distinguish known failure from uncertain outcome; check existing sessions/results before retrying.

State what was verified: metadata only, token stored, authenticated read passed, or action completed. Listing a catalog does not prove all tools work. Distinguish an HTTP transport success from a completed T3 task. Record no secrets in memory or reusable artifacts.
