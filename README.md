# Webshare Proxy Setup Instructions for AI Agent

## Purpose

Use these instructions to set up and manage Webshare proxy services for an AI agent.

This guide covers:

- Retrieving proxy lists from the Webshare API
- Configuring direct proxy connections
- Configuring backbone proxy connections
- Managing IP authorization
- Applying reliability best practices in sandboxed or dynamic environments

---

## Security Requirements

Do not hard-code API keys, proxy usernames, or proxy passwords in source code, prompts, logs, or public files.

Store credentials in a secure environment variable store, `.env` file, or secrets manager.

Required secrets:

```bash
WEBSHARE_API_KEY=<YOUR_API_KEY>
WEBSHARE_PROXY_USERNAME=<YOUR_PROXY_USERNAME>
WEBSHARE_PROXY_PASSWORD=<YOUR_PROXY_PASSWORD>
```

The agent should load credentials from the environment at runtime.

---

## Webshare API Authentication

All Webshare REST API requests must include the following header:

```http
Authorization: Token <YOUR_API_KEY>
```

Example using an environment variable:

```bash
curl -X GET "https://proxy.webshare.io/api/v2/proxy/list/?mode=direct&page_size=25" \
  -H "Authorization: Token ${WEBSHARE_API_KEY}"
```

---

## Skill Metadata

```yaml
name: webshare-setup
description: >
  Setup and manage Webshare proxy services. Use this skill to retrieve proxy
  lists, configure IP authorization, and set up backbone or direct connections.
```

---

## Setup Workflow

### 1. Load Credentials

The agent must load the following values before making requests:

```bash
WEBSHARE_API_KEY
WEBSHARE_PROXY_USERNAME
WEBSHARE_PROXY_PASSWORD
```

If any credential is missing, stop and ask the operator to provide it securely.

Do not continue with placeholder or empty credentials.

---

### 2. Retrieve Proxy List

Use the Webshare Proxy List API to retrieve available proxies.

Endpoint:

```http
GET https://proxy.webshare.io/api/v2/proxy/list/
```

Supported parameters:

| Parameter | Required | Description |
|---|---:|---|
| `mode` | Yes | Use `direct` for specific IP proxies or `backbone` for Webshare entry nodes. |
| `page` | No | Page number. Default is `1`. |
| `page_size` | No | Number of proxies per page. Default is `25`. |
| `country_code__in` | No | Comma-separated ISO country codes, such as `US,GB`. |

Example request for direct proxies:

```bash
curl -X GET "https://proxy.webshare.io/api/v2/proxy/list/?mode=direct&page_size=25" \
  -H "Authorization: Token ${WEBSHARE_API_KEY}"
```

Example request for backbone proxies:

```bash
curl -X GET "https://proxy.webshare.io/api/v2/proxy/list/?mode=backbone&page_size=25" \
  -H "Authorization: Token ${WEBSHARE_API_KEY}"
```

---

## Proxy Connection Modes

Webshare supports two primary connection modes:

1. Direct connection
2. Backbone connection

Choose the mode based on the task requirements.

---

## Direct Proxy Connection

Use direct mode when the agent needs to connect through specific proxy IP addresses and ports.

Direct proxy format:

```text
proxy_address:port:username:password
```

Example template:

```text
38.154.203.95:5863:<USERNAME>:<PASSWORD>
198.105.121.200:6462:<USERNAME>:<PASSWORD>
64.137.96.74:6641:<USERNAME>:<PASSWORD>
209.127.138.10:5784:<USERNAME>:<PASSWORD>
38.154.185.97:6370:<USERNAME>:<PASSWORD>
84.247.60.125:6095:<USERNAME>:<PASSWORD>
142.111.67.146:5611:<USERNAME>:<PASSWORD>
191.96.254.138:6185:<USERNAME>:<PASSWORD>
31.58.9.4:6077:<USERNAME>:<PASSWORD>
104.239.107.47:5699:<USERNAME>:<PASSWORD>
```

The agent must replace `<USERNAME>` and `<PASSWORD>` with secure runtime values.

Do not print the fully resolved proxy strings in logs.

---

## Backbone Proxy Connection

Use backbone mode when the agent needs more reliable proxy behavior in dynamic or sandboxed environments.

Backbone address:

```text
p.webshare.io
```

Supported ports:

```text
80, 1080, 3128, 9999-19999
```

Backbone username format:

```text
{username}-{country_code}-{geo_params}-{session_id}
```

### Sticky Sessions

Use a numeric session ID to keep the same exit IP for a session.

Example:

```text
user-us-1234
```

Expected behavior:

```text
US exit IP with sticky session ID 1234
```

### Rotating Sessions

Use `rotate` to request a new exit IP on each request.

Example:

```text
user-us-rotate
```

Expected behavior:

```text
US exit IP with rotation enabled
```

---

## IP Authorization

If the agent is not using username/password authentication, it must authorize its source IP address before using the proxies.

### List Authorized IPs

```http
GET https://proxy.webshare.io/api/v2/ip_authorization/
```

Example:

```bash
curl -X GET "https://proxy.webshare.io/api/v2/ip_authorization/" \
  -H "Authorization: Token ${WEBSHARE_API_KEY}"
```

### Add Authorized IP

```http
POST https://proxy.webshare.io/api/v2/ip_authorization/
```

Body:

```json
{
  "ip_address": "1.2.3.4"
}
```

Example:

```bash
curl -X POST "https://proxy.webshare.io/api/v2/ip_authorization/" \
  -H "Authorization: Token ${WEBSHARE_API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{"ip_address": "1.2.3.4"}'
```

### Delete Authorized IP

```http
DELETE https://proxy.webshare.io/api/v2/ip_authorization/<id>/
```

Example:

```bash
curl -X DELETE "https://proxy.webshare.io/api/v2/ip_authorization/<id>/" \
  -H "Authorization: Token ${WEBSHARE_API_KEY}"
```

---

## Agent Decision Rules

### Use Direct Mode When

- A specific proxy IP is required
- The proxy list has already been retrieved
- Username/password authentication is available
- The agent's source IP has already been authorized, if IP authorization is required

### Use Backbone Mode When

- The environment is dynamic or sandboxed
- Source IP may change between runs
- Direct proxies are failing intermittently
- The task benefits from rotation or sticky sessions
- Reliability is more important than using a specific proxy IP

### Prefer Username/Password Authentication When

- The agent operates from infrastructure with changing outbound IPs
- IP authorization cannot be reliably maintained
- The task runs in short-lived containers or sandboxes

### Prefer IP Authorization When

- The agent runs from a stable server
- Username/password authentication is unavailable
- The network source IP is predictable

---

## Reliability Notes

Direct proxy connections from sandboxed or dynamic environments may be inconsistent.

Common causes:

- The outbound sandbox IP is not authorized
- Free-tier proxies have stricter limits
- The selected direct proxy is unavailable
- The proxy requires username/password authentication
- The source IP changed between setup and use

Recommended fallback sequence:

1. Verify credentials are present.
2. Confirm the requested proxy mode is valid.
3. If using direct mode, test another direct proxy.
4. If using IP authorization, confirm the current source IP is authorized.
5. If direct mode remains unreliable, switch to backbone mode.
6. Use sticky sessions when the task requires session continuity.
7. Use rotating sessions when the task benefits from frequent IP changes.

---

## Minimal Agent Procedure

1. Load `WEBSHARE_API_KEY`, `WEBSHARE_PROXY_USERNAME`, and `WEBSHARE_PROXY_PASSWORD`.
2. Decide whether the task requires direct or backbone proxy mode.
3. If direct mode is required, retrieve the proxy list with `mode=direct`.
4. If backbone mode is acceptable, use `p.webshare.io` with the appropriate port.
5. If IP authorization is required, authorize the current source IP first.
6. Configure the HTTP client with the selected proxy.
7. Test the proxy with a lightweight request.
8. If the connection fails, follow the fallback sequence above.

---

## Reference

Webshare API documentation:

```text
https://apidocs.webshare.io/
```
