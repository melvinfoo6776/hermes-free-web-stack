# Hermes Agent without paid web extractor APIs

A solution guide for using DDGS search with Jina Reader or self-hosted
Crawl4AI extraction in a Dockerized Hermes Agent Telegram gateway.

This repository is intentionally documentation-only. Hermes, DDGS, Jina, and
Crawl4AI change quickly, so it explains the integration and gives commands that
readers can adapt to their installed versions instead of shipping an installer
that may become obsolete within days.

## The problem

Hermes separates web research into two capabilities:

- `web_search` finds relevant URLs;
- `web_extract` reads and cleans the content of those URLs.

The default and best-supported providers usually combine both capabilities,
but they are commercial services. Their free tiers and Hermes's anonymous
fallbacks are useful for testing and light use, not guaranteed unlimited
capacity. DDGS solves the search side without an API key, but it cannot extract
page content.

The desired result is therefore:

```text
Telegram -> Hermes Agent -> free web search -> free page extraction
```

Connecting the free pieces is not automatic because several separate
environments are involved:

1. the Docker host;
2. the Hermes gateway container;
3. the Python environment used by the gateway;
4. the Hermes plugin registry and saved provider settings; and
5. the optional execution sandbox used by terminal or `execute_code`.

Installing a Python package proves only that it exists in one environment. It
does not automatically register a native Hermes `web_search` or `web_extract`
provider.

## What this document does

This document:

1. explains the cost and capability trade-offs of Hermes's existing web
   providers;
2. recommends DDGS + Jina Reader as the simplest no-paid-extractor path;
3. presents DDGS + self-hosted Crawl4AI as the local-processing alternative;
4. shows how to validate each layer before connecting it to Telegram;
5. explains why the Docker execution sandbox is separate from native web tools;
6. records the important security, privacy, and maintenance limitations; and
7. provides a short error matrix after the main setup instructions.

It does not replace the upstream provider interfaces or promise that copied
plugin code will work forever. Readers should adapt the integration to the
exact Hermes version they run.

## Existing Hermes options and why they may not fit

Hermes's default and mainstream provider recommendations are the easiest path,
but they are generally commercial or managed services. Some have useful free
tiers—and newer Hermes releases may rotate across anonymous provider
fallbacks—but these paths are quota-limited, rate-limited, unguaranteed, or
shift infrastructure costs to the operator. They are not the same as a durable,
unrestricted, no-paid-API setup.

The current Hermes provider landscape is roughly:

| Provider | Hermes capability | Cost/access reality |
|---|---|---|
| Firecrawl (default keyed provider) | Search, extract, crawl | Limited cloud allowance, then paid; self-hosting shifts compute and maintenance to you |
| Tavily | Search, extract, crawl | Limited free credits, then paid |
| Exa | Search and extract | Limited free usage, then paid |
| Parallel | Search and extract | Commercial service; availability in Hermes's keyless fallback depends on the installed version |
| Nous Tool Gateway | Managed Firecrawl and other tools | Pay-as-you-use through a paid Nous Portal subscription; some accounts may receive a small promotional free pool |
| Brave Search | Search only | Account/API key and renewable free credits, then paid usage |
| SearXNG | Search only | Free/open-source when self-hosted, but still needs infrastructure and maintenance |
| DDGS | Search only | Keyless and no provider subscription, but it does not extract page content |
| xAI | Search only | Paid API or subscription path |

Current Hermes documentation also describes a keyless, round-robin fallback
across several vendors for fresh installations. Hermes calls it a strictly
last-resort tier: it can be throttled, the vendor set is version-dependent, and
a configured backend or API key takes precedence. This is convenient for light
testing, but it should not be treated as guaranteed free production capacity.

Provider quotas and prices change frequently. Check the current official
[Hermes comparison](https://hermes-agent.nousresearch.com/docs/user-guide/features/web-search),
[Firecrawl pricing](https://www.firecrawl.dev/pricing),
[Tavily pricing](https://www.tavily.com/pricing),
[Brave Search API pricing](https://brave.com/search/api/), and
[Nous Tool Gateway terms](https://hermes-agent.nousresearch.com/docs/user-guide/features/tool-gateway)
instead of relying on hard-coded credit counts in this guide.

## Solution described in this document

The primary solution fills the extraction gap left by DDGS without requiring a
paid extractor subscription:

```text
DDGS (keyless search) + Jina Reader (keyless basic extraction)
```

The alternative, when local processing is preferred, is:

```text
DDGS (keyless search) + self-hosted Crawl4AI (local extraction)
```

Here, “free” means **no paid web-extractor subscription is required for the
documented path**. It does not mean unlimited or costless: Jina is a third-party
service with changeable rate limits, while Crawl4AI consumes your own compute,
memory, bandwidth, and maintenance time. Your LLM provider may also charge for
processing or summarizing extracted content.

## Which solution to choose

| Need | Suggested option |
|---|---|
| Free search | Hermes's built-in DDGS provider |
| Simple public-page extraction | Jina Reader through a Hermes extraction plugin |
| JavaScript-heavy or locally processed pages | A separately deployed Crawl4AI service plus a Hermes extraction plugin |

Start with DDGS + Jina. Add Crawl4AI only if Jina cannot handle your pages or
you do not want URLs sent to Jina.

If a commercial provider's free tier is sufficient for your workload, use the
built-in Hermes setup—it is simpler and better supported. This guide is for
people who specifically want to avoid depending on that quota or paying for an
extractor service.

## 1. Confirm the Hermes container

The examples assume the container is named `hermes-agent` and its application
user is `hermes`:

```bash
docker ps --format 'table {{.Names}}\t{{.Image}}\t{{.Status}}'
docker exec -u hermes hermes-agent sh -lc 'id; command -v hermes; python --version'
```

Change the container name in later commands if yours is different.

## 2. Configure and test DDGS as a native search provider

Use Hermes's supported configuration flow first:

```bash
docker exec -it hermes-agent hermes tools
```

Select DDGS for web search, then restart the gateway:

```bash
docker restart hermes-agent
```

Open a new Telegram session with `/new` and ask:

> Use only the native web_search tool to search for the Python homepage.

If Hermes reports that `ddgs` is missing, determine which Python interpreter
the gateway uses and test that exact interpreter:

```bash
docker exec -u hermes hermes-agent sh -lc \
  'command -v python; python -c "from ddgs import DDGS; print(DDGS)"'
```

On the reference Docker image, the interpreter was
`/opt/hermes/.venv/bin/python` and persistent lazy packages were stored under
`/opt/data/lazy-packages`:

```bash
docker exec -u hermes hermes-agent sh -lc \
  'PYTHONPATH=/opt/data/lazy-packages /opt/hermes/.venv/bin/python -c \
  "from importlib.metadata import version; print(version(\"ddgs\"))"'
```

Prefer Hermes's lazy-dependency mechanism when the installed version supports
it. If you manually install DDGS, install it into a persistent path used by the
gateway—not only on the Docker host—and re-test after recreating or restarting
the container. Consult the current Hermes DDGS implementation before choosing
the target path.

## 3. Test Jina Reader independently

Jina Reader accepts a public URL after `https://r.jina.ai/http://...` or
`https://r.jina.ai/https://...`. A quick host-side test is:

```bash
curl -fsSL 'https://r.jina.ai/https://example.com'
```

This confirms network access to Jina. It does **not** confirm that Hermes has a
provider named `jina`.

## 4. Connect Jina to Hermes

Hermes needs a registered extraction provider; installing a `jina` Python
package alone is not enough. Follow the current upstream
[Hermes web-provider plugin guide](https://github.com/NousResearch/hermes-agent/blob/main/website/docs/developer-guide/web-search-provider-plugin.md).

The durable procedure is:

1. Copy the smallest current extraction-capable provider from the same Hermes
   version as your running container.
2. Place the custom plugin in Hermes's documented user-plugin directory.
3. Give the provider a unique name such as `jina` and implement extraction by
   requesting `https://r.jina.ai/<target-url>`.
4. Match the `extract()` return contract from your installed Hermes version.
   Do not rely on an older example—the contract has changed between releases.
5. Enable the plugin with `hermes plugins enable ...`.
6. Set `web.extract_backend` to the provider's registered name through
   `hermes tools` or `hermes config set`.
7. Restart the gateway and start a new Telegram session with `/new`.

Verify registration and saved configuration:

```bash
docker exec -u hermes hermes-agent hermes plugins list
docker exec -u hermes hermes-agent hermes config show
docker logs --tail 200 hermes-agent 2>&1 | grep -iE 'plugin|jina|provider|error'
```

Do not publish the output of `hermes config show` until you have checked it for
tokens, user IDs, URLs with credentials, and other secrets.

Test the native path from Telegram:

> Use only the native web_extract tool to extract and summarize
> https://example.com. Do not use terminal, browser, or execute_code.

If that succeeds, test the site you actually care about.

## 5. Optional Crawl4AI path

Crawl4AI is a separate crawler service. Installing its Python package in the
Hermes container does not register it as a Hermes provider.

Use the current [Crawl4AI Docker documentation](https://github.com/unclecode/crawl4ai/blob/main/deploy/docker/README.md)
to deploy it, keep its API off the public internet, and test it directly. Then
create a Hermes extraction-provider adapter using the current Hermes plugin
guide, exactly as for Jina, but call the Crawl4AI API instead.

Before selecting it in Hermes, verify all three layers independently:

1. the Crawl4AI container is healthy;
2. the Hermes container can reach its Docker hostname and port; and
3. the Hermes provider is registered and selected as `web.extract_backend`.

Crawl4AI adds Chromium, container networking, memory usage, and another API to
maintain. It is not required for ordinary public pages.

## Docker socket: a separate issue

Native `web_search` and `web_extract` providers run through the Hermes gateway.
They do not normally require `/var/run/docker.sock`.

The socket is relevant when Hermes tries to start a Docker-backed terminal or
`execute_code` sandbox. Therefore this failure:

```text
Docker command is available but docker version failed
```

does not prove DDGS or Jina is broken. It means the execution path could not
reach the Docker daemon. Test native web tools directly before changing Docker
socket permissions.

Mounting the Docker socket gives the container powerful control over Docker and
often the host. Do not add it merely to make web search work.

## Troubleshooting appendix

| Error or symptom | What it usually means | Next action |
|---|---|---|
| `ddgs package is not installed` | Missing from the gateway's Python path | Test the exact gateway interpreter and persistent package path |
| `no registered web extract provider has that name` | Config names a provider that Hermes did not discover | Check plugin location, manifest, enablement, logs, and provider name |
| Direct Jina request works but `web_extract` fails | Jina is reachable but the Hermes adapter or return contract is wrong | Compare the plugin with the provider interface in the installed Hermes version |
| `same_tool_failure_halt` | Hermes stopped repeating the same failed call | Read the first tool error and fix that layer before retrying |
| Docker exit status 125 | Optional execution sandbox failed before code ran | Do not use `execute_code` to validate native web tools |
| Shell test works but Telegram fails | Gateway or conversation has stale state | Restart Hermes and send `/new` |
| Crawl4AI connection refused | Service/networking problem | Test container health, DNS, port, and shared network |

More evidence commands are in [the troubleshooting notes](docs/troubleshooting.md).

## Security and privacy

- Treat extracted webpages as untrusted input that may contain prompt
  injection.
- Use an explicit Telegram user allowlist or audited pairing.
- Keep terminal, file-write, browser, and code-execution tools out of a
  research-only bot where possible.
- Jina receives every URL you send it. Do not submit private, authenticated,
  signed, personal, embargoed, or regulated URLs.
- Keep Crawl4AI private and updated; it runs a browser against hostile pages.
- Never paste bot tokens, `.env` contents, signed URLs, or unredacted
  configuration into a public issue.

See [SECURITY.md](SECURITY.md) for the full checklist.

## Applicability

The commands were derived from a macOS Docker Desktop host running a
Linux-based Hermes Agent container and using Telegram. The concepts also apply
to Linux Docker Engine and Windows/WSL2, but paths, users, image contents, and
networking can differ.

Always compare this guide with the documentation and source for the exact
Hermes version you run.

## Upstream references

- [Hermes web-provider plugin guide](https://github.com/NousResearch/hermes-agent/blob/main/website/docs/developer-guide/web-search-provider-plugin.md)
- [Hermes provider interface](https://github.com/NousResearch/hermes-agent/blob/main/agent/web_search_provider.py)
- [Hermes web tools](https://github.com/NousResearch/hermes-agent/blob/main/tools/web_tools.py)
- [Hermes Web Search & Extract provider comparison](https://github.com/NousResearch/hermes-agent/blob/main/website/docs/user-guide/features/web-search.md)
- [Jina Reader](https://jina.ai/reader/)
- [Crawl4AI Docker deployment](https://github.com/unclecode/crawl4ai/blob/main/deploy/docker/README.md)

## License

MIT.
