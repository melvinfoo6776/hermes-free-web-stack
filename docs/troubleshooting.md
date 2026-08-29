# Troubleshooting notes

Diagnose the first failing layer. Do not repeat the same Telegram tool call
without changing anything.

## Collect basic evidence

Replace `hermes-agent` if your container has another name:

```bash
docker ps --format 'table {{.Names}}\t{{.Image}}\t{{.Status}}'
docker exec -u hermes hermes-agent hermes plugins list
docker exec -u hermes hermes-agent hermes config show
docker logs --tail 200 hermes-agent
```

Check DDGS with the gateway interpreter, not the host interpreter:

```bash
docker exec -u hermes hermes-agent sh -lc \
  'PYTHONPATH=/opt/data/lazy-packages /opt/hermes/.venv/bin/python -c \
  "from ddgs import DDGS; print(list(DDGS().text(\"Python\", max_results=1)))"'
```

Check Jina independently:

```bash
curl -fsSL 'https://r.jina.ai/https://example.com'
```

## Interpret the result

- **DDGS import fails:** the dependency is absent from the gateway's Python
  path, even if it is installed on the host or inside another environment.
- **DDGS shell test passes but native search fails:** inspect provider
  selection, plugin discovery, gateway logs, and the active Hermes version.
- **Jina curl passes but `jina` is unregistered:** the remote service works;
  the missing piece is the Hermes provider plugin.
- **Provider is listed but extraction fails:** compare its `extract()` return
  shape with the interface shipped by the exact Hermes version in use.
- **Docker exit 125:** terminal or `execute_code` failed before the test ran.
  Test native `web_search`/`web_extract` directly.
- **Only Telegram fails:** restart the gateway and send `/new` so the session
  sees the new provider/tool configuration.

## Verify the Docker socket separately

Only do this when you intentionally use Docker-backed execution tools:

```bash
docker exec -u hermes hermes-agent sh -lc \
  'id; ls -ln /var/run/docker.sock 2>/dev/null || true; docker version'
```

Native DDGS/Jina web tools should not require the socket. A Docker socket mount
is highly privileged and is not a general fix for web-provider configuration.

## Before sharing logs

Redact Telegram IDs, bot tokens, API keys, bearer tokens, `.env` contents,
signed URLs, private hostnames, and configuration values that may identify your
deployment.
