# Security notes

This repository contains documentation, not executable installation code.
Commands are examples and must be reviewed against the current Hermes, Jina,
Crawl4AI, Docker, and operating-system documentation.

## Minimum precautions

- Restrict the Telegram gateway to an explicit user allowlist or audited
  pairing list.
- Treat every extracted page as untrusted, prompt-injection-capable input.
- Give a research-only bot only the tools it needs. Avoid terminal, file-write,
  browser, and code-execution access where possible.
- Do not mount `/var/run/docker.sock` solely for native web search or
  extraction. Docker socket access is effectively administrative access to the
  Docker daemon and often the host.
- Jina is a third-party service and receives submitted URLs. Never send private,
  authenticated, signed, personal, embargoed, or regulated URLs.
- Keep a self-hosted Crawl4AI API private, authenticated, resource-limited, and
  updated. It runs a browser against potentially hostile pages.
- Review custom Hermes provider code before installation; it executes in the
  gateway process.
- Do not publish `.env` files, bot tokens, API keys, user IDs, pairing records,
  private configuration, signed URLs, or unredacted diagnostic output.

For a suspected vulnerability, contact the repository owner privately or use
GitHub private vulnerability reporting. Do not place secrets or exploit details
for a live deployment in a public issue.
