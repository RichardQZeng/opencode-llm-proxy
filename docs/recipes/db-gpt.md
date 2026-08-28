# Using opencode-llm-proxy with DB-GPT

## What this gives you

Use any model authenticated in OpenCode from DB-GPT's OpenAI-compatible provider. For example, DB-GPT can use `opencode/mimo-v2.5-free` for chat, SQL generation, and agent workflows.

## Network Addressing

The client and the proxy may run in different network domains. `localhost` always means the current process or container, not the machine hosting OpenCode.

| Client location | Proxy URL | Notes |
|---|---|---|
| Native process on the OpenCode machine | `http://127.0.0.1:4010/v1` | Keep the proxy bound to loopback. |
| Docker Desktop container on the OpenCode machine | `http://host.docker.internal:4010/v1` | Docker Desktop resolves this name to the host. |
| Linux Docker container on the OpenCode machine | A shared Docker network service name, or the host gateway | Do not use `localhost`; if the proxy is reachable over the host gateway, bind it beyond loopback and require a token. |
| Client on another machine | `http://<opencode-host>:4010/v1` | Bind the proxy to a trusted LAN interface, require a token, and restrict the firewall. |

Do not expose the proxy to the public internet.

## Configure DB-GPT

Add an LLM entry to the DB-GPT TOML configuration. Set `api_base` to the address for the client location above:

```toml
[[models.llms]]
name = "opencode/mimo-v2.5-free"
provider = "proxy/openai"
api_base = "http://host.docker.internal:4010/v1"
api_key = "${env:OPENCODE_LLM_PROXY_TOKEN}"
```

Pass the token to the DB-GPT process or container as an environment variable. Do not put the token in the TOML file:

```yaml
services:
  webserver:
    environment:
      OPENCODE_LLM_PROXY_TOKEN: ${OPENCODE_LLM_PROXY_TOKEN}
```

Restart DB-GPT after changing its model configuration. The new model appears in DB-GPT's chat model selector alongside existing local or cloud models.

## Verify

From the same network domain as DB-GPT, list the models available through the proxy:

```bash
curl http://127.0.0.1:4010/v1/models \
  -H "Authorization: Bearer $OPENCODE_LLM_PROXY_TOKEN"
```

Confirm `opencode/mimo-v2.5-free` is in the response, then select it in DB-GPT and send a short chat request.

## Troubleshooting

- **401 Unauthorized**: DB-GPT's `OPENCODE_LLM_PROXY_TOKEN` does not match the proxy token.
- **Connection refused from Docker**: use the correct Docker-to-host address from the table. `127.0.0.1` and `localhost` point to the DB-GPT container itself.
- **Model is missing**: run `opencode models opencode --verbose` and use the exact provider/model ID returned by the proxy.

## Security Notes

- The proxy provides access to the model accounts configured in OpenCode. Keep it local or on a trusted network.
- MiMo requests leave the machine through OpenCode's upstream provider. Use a local model, such as Ollama, when the prompt or data must remain local.
- See [security.md](../security.md) for full guidance.
