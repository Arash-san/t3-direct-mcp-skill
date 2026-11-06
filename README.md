# T3 Direct Connect

Connect a tool-capable assistant to **T3 Code through T3 Connect and MCP** when browser approval does not finish terminal sign-in.

In the tested dot setup, the terminal could reach the public T3 MCP endpoint. Pairing was accepted in the browser, but the OAuth return URL pointed at localhost in the **browser's network context**, separate from the terminal waiting for it. This skill explains how the JSON pairing flow available in compatible T3 versions completes the connection without that callback.

Tailscale was also unavailable in that tested dot environment, so the working connection used the public T3 Connect HTTPS address. Other dots or hosted environments may have Tailscale available; check the actual environment rather than assuming the same limitation.

This repository contains an instructional [SKILL.md](SKILL.md). It explains discovery, approved pairing, credential protection, direct MCP requests, and verification. It does **not** include a runnable client, automatically install an integration, or grant network access or permissions.

## Who this helps

Use this with dots, Claude, or another assistant that can read instructions and use an authorized HTTP client. It is especially useful when the assistant's browser and terminal run in separate environments.

That is a specific callback-delivery failure, not evidence that every dot cannot connect. Prefer a working native MCP integration when available. This skill does not bypass a network restriction, access denial, or security warning.

## Prerequisites

- A running T3 environment and its verified, reachable T3 Connect HTTPS MCP URL. In T3, use **Settings → Environments → Copy MCP URL**. For a cloud assistant, use the public T3 Connect address, not your computer's localhost address. See [T3's outside-agent guide](https://github.com/pingdotgg/t3code/blob/main/docs/user/outside-agents.md).
- A client with an authorized terminal or HTTP tool, such as Python and curl, and permission to reach that server.
- A compatible T3 version and approved credential storage. Get a fresh pairing code from **Settings → Sharing & access** in T3 and supply it through an approved secure interface. Never paste the code into ordinary chat.
- Your confirmation of the server, access mode, access lifetime, and storage location before creating or expanding persistent access.

## Load the skill and start

1. Read [SKILL.md](SKILL.md) before using it.
2. **In a skill-capable client:** download this repository, or run `git clone https://github.com/Arash-san/t3-direct-mcp-skill.git t3-direct-connect`, then use that client's documented skill importer or skill directory. Where it expects a named folder, use `t3-direct-connect/SKILL.md`. Reload skills if the client requires it and verify that it can read `t3-direct-connect`. Skill locations and installation commands vary by client.
3. **Without skill installation:** attach `SKILL.md`, or ask the assistant to open and read it, for the current task. This supplies instructions for that conversation; it does not promise a persistent installation or add missing tools.
4. Give the assistant the verified MCP URL and ask it to check prerequisites before pairing. For example:

   > Read SKILL.md and help me connect to https://YOUR-T3-CONNECT-HOST/mcp. Prefer read-only access. Check compatibility, ask for approval before creating persistent access, and use a secure prompt for the pairing code. Verify the connection with a project-list read. Do not start a T3 agent task.

5. Approve the specific connection only after reviewing its details. Finish by checking the authenticated read result, not just a sign-in page or a “connected” label.

## How the callback-free fix works

On T3 versions supporting its JSON pairing decision endpoint, the client can complete OAuth without a browser callback listener. The full request fields and validation rules are in [SKILL.md](SKILL.md).

1. Read the server's OAuth discovery metadata. Verify the issuer, MCP resource, registration/token endpoints, and PKCE S256 support.
2. Generate fresh state and a PKCE verifier/challenge. Register `http://127.0.0.1/callback` as the return value; do not start a listener.
3. Send the authorization details and an approved `pairing-code` decision to the version-verified `/oauth/mcp/decision` route.
4. Receive `redirectTo` as JSON and **parse it locally without navigating to it**. Validate the exact registered destination, matching state, authorization code, issuer requirements, and absence of errors or duplicate security fields.
5. Exchange the code with the original PKCE verifier at the discovered token endpoint. Validate token type, granted scopes, and expiry before securely storing or using it.
6. Use Python/curl to send authenticated MCP JSON-RPC requests directly to the verified `/mcp` endpoint. Initialize the session, negotiate the protocol, send `notifications/initialized`, inspect `tools/list`, then call an appropriate tool.

The browser no longer needs to reach the terminal's localhost. OAuth authorization and PKCE verification still apply. This is T3-specific behavior, not a generic workaround for every OAuth server. See the [T3 authentication contracts](https://github.com/pingdotgg/t3code/blob/main/packages/contracts/src/auth.ts).

## Choose access deliberately

- **Read-only:** `orchestration:read`, suitable for inventory.
- **Approval-required:** `orchestration:read` and `orchestration:operate`, with supervised T3 operation.
- **Full-access:** all four scopes: `orchestration:read`, `orchestration:operate`, `settings:write`, and `source-control:write`.

These are selected modes, not the complete version-dependent list. Full access is not blanket permission for every action. Discovery may advertise only read/operate even when full access legitimately returns all four scopes. Validate against the deployed `mcpClientScopes` contract and the mode you approved, rather than accepting arbitrary extras.

The reviewed flow returned a 30-day token and exposes no requested-lifetime parameter. Check actual expiry and server-side session behavior; do not promise a shorter grant merely because you intend to stop using it sooner. Review or revoke approved clients in T3 when needed.

## What success and usage mean

Success means an authenticated `t3_project_list` call returns a valid result, including an empty list if appropriate. Follow pagination and check tool errors. Metadata discovery, token storage, or a tool catalog alone does not prove a project read or later task succeeded.

Direct inventory reads do not themselves start model turns. No extra model is needed merely to relay MCP traffic. Starting or messaging a T3 agent may consume its provider usage; the assistant's own usage is separate. This repository makes no free or unlimited quota promise.

## Security and compatibility

Keep pairing codes, tokens, authorization codes, and PKCE verifiers out of chat, command arguments, logs, screenshots, and repositories. Use approved secret storage and stdin/secure interfaces. Retain TLS verification, reject unexpected redirects, and stop on permission or network denials. Check uncertain outcomes before retrying pairing or writes.

The skill was checked against T3 contracts on **2026-11-06**. The official links above track `main`, not a pinned release. Verify the deployed tag/commit before relying on route, scope, expiry, or tool behavior. Unsupported versions need a supported connection route, not a bypass.
