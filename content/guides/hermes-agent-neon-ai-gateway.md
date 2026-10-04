---
title: 'Run Hermes Agent on Neon AI Gateway'
subtitle: 'Point Hermes Agent, the open-source OpenClaw alternative from Nous Research, at Neon AI Gateway so its chats, messaging bots, cron jobs, and subagents use Claude, GPT, Gemini, and open-weight models through one Neon credential.'
author: neon-team
enableTableOfContents: true
createdAt: '2026-10-03T00:00:00.000Z'
---

You set up a personal agent on a small VPS. It answers you on Telegram, runs a report every morning, and spins up subagents when a task gets big. A month later, `~/.hermes/.env` holds an OpenRouter key, an OpenAI key, and an Anthropic key, and you're not sure which job spends what.

[Hermes Agent](https://github.com/NousResearch/hermes-agent) is an open-source agent from [Nous Research](https://nousresearch.com). It runs in your terminal or as a gateway process that you talk to from Telegram, Discord, Slack, WhatsApp, or Signal. It keeps memory across sessions, writes its own skills, and has a built-in cron scheduler. If you're coming from OpenClaw, Hermes can import your OpenClaw settings, memories, and skills.

[Neon AI Gateway](/docs/ai-gateway/overview) gives you one endpoint for models from OpenAI, Anthropic, Google, and others. You authenticate with a Neon credential instead of provider API keys. Hermes works with any OpenAI-compatible endpoint, so you can add Neon AI Gateway as a custom provider in `config.yaml`.

In this guide, you'll install Hermes, connect it to Neon AI Gateway, test a Claude model and a GPT model with tool calls, and use the gateway in messaging, cron jobs, and subagents.

## What you get from the combination

- **One credential for every model.** Claude, GPT, Gemini, Llama, Qwen, Kimi, and GLM models all authenticate with the same Neon token. Switching models mid-session with `/model` doesn't need another provider key.
- **Inference on your Neon bill.** Every request draws on your Neon [prepaid credits](/docs/ai-gateway/prepaid-credits), including the ones your cron jobs and messaging bots make while you're not watching.
- **One place to cut access.** If the VPS is compromised or you hand the agent to someone else, revoke one Neon credential instead of rotating keys at three providers.

## Prerequisites

- A Neon project on a paid plan with prepaid credits, in a region that supports AI Gateway. See [Get started with Neon AI Gateway](/docs/ai-gateway/get-started#get-access) for the current list.
- A recent version of the [Neon CLI](/docs/cli) that includes the `neon credentials` command. You can also create the credential in the Neon Console.
- macOS, Linux, or WSL2 for the Hermes install command below. Hermes also runs natively on Windows; see the [Hermes installation docs](https://hermes-agent.nousresearch.com/docs/getting-started/installation).

<Steps>

## Install Hermes Agent

Run the official installer:

```bash
curl -fsSL https://hermes-agent.nousresearch.com/install.sh | bash
```

The installer sets up its own Python and Node.js under `~/.hermes` and adds the `hermes` command to your `PATH`. Open a new terminal, or reload your shell profile, before you continue.

```bash
hermes --version
```

If the installer stops with an error while preparing Node dependencies, run it again. A second run picks up where the first one stopped.

## Create an AI Gateway credential

Hermes needs a Neon credential with the `ai_gateway:invoke` scope. Name it after the machine or bot it belongs to, so you know which token to revoke later.

<Tabs labels={["CLI", "Console"]}>
<TabItem>

Run this in a directory [linked](/docs/cli/link) to your Neon project, or pass `--project-id` and `--branch`:

```bash
neon credentials create --scope ai_gateway:invoke --name hermes
```

The CLI prints the `api_token` (it starts with `nt_live_`) once. Copy it for the next step.

</TabItem>
<TabItem>

In the Neon Console, click **Connect** at the top of the sidebar and open the **AI Gateway** tab. Click **Reveal credential** to show the token.

</TabItem>
</Tabs>

The **Connect** dialog also shows your branch's gateway host as `NEON_AI_GATEWAY_BASE_URL`. It looks like this:

```text
https://br-winter-pond-aptw82ef-api.ai.c-2.us-east-2.aws.neon.tech
```

This is different from your database connection string. A credential works on the branch you created it on and on that branch's descendants. See [How branch binding works](/docs/ai-gateway/authentication#how-branch-binding-works) for details.

## Store the token in Hermes

Hermes reads secrets from `~/.hermes/.env`. Add the token there:

```bash filename="~/.hermes/.env"
NEON_AI_GATEWAY_TOKEN=nt_live_...
```

The `config.yaml` entries in the next step refer to this variable by name, so the token itself never appears in your config file.

## Add Neon AI Gateway as a provider

Open `~/.hermes/config.yaml` and add two named custom providers, then point the main model at the first one. Replace the host with your own.

```yaml filename="~/.hermes/config.yaml"
providers:
  neon:
    name: Neon AI Gateway
    api: https://br-winter-pond-aptw82ef-api.ai.c-2.us-east-2.aws.neon.tech/v1
    key_env: NEON_AI_GATEWAY_TOKEN
    transport: chat_completions
    default_model: claude-sonnet-4-6
  neon-openai:
    name: Neon AI Gateway (OpenAI Responses)
    api: https://br-winter-pond-aptw82ef-api.ai.c-2.us-east-2.aws.neon.tech/openai/v1
    key_env: NEON_AI_GATEWAY_TOKEN
    transport: codex_responses
    default_model: gpt-5-4-mini

model:
  provider: custom:neon
  default: claude-sonnet-4-6
```

Keep any other keys that are already in the file, such as `plugins` or `_config_version`.

The two entries use the same token and host but different endpoints:

- `neon` uses the [chat completions endpoint](/docs/ai-gateway/chat-completions) at `/v1`. Use it for Claude, Gemini, and open-weight models.
- `neon-openai` uses the [OpenAI Responses endpoint](/docs/ai-gateway/openai-responses) at `/openai/v1`. Use it for GPT models.

GPT models need the second entry because Hermes sends a `reasoning_effort` value with every request that includes tools. OpenAI rejects that combination on `/v1/chat/completions` for GPT-5 models, and accepts it on the Responses API. Codex models such as `gpt-5-3-codex` only work on the Responses endpoint.

## Test the connection

Run a one-shot prompt that also makes a tool call:

```bash
hermes chat -q "Use your terminal tool to run: echo neon-tool-check. Then tell me the output and which model you are." --oneshot -Q
```

You should see the command output and the model name:

```text
neon-tool-check

Model: claude-sonnet-4-6
```

Then test a GPT model on the Responses entry:

```bash
hermes chat -q "Use your terminal tool to run: echo responses-check. Then tell me the output and which model you are." --provider custom:neon-openai -m gpt-5-4-mini --oneshot -Q
```

```text
responses-check

Model: gpt-5-4-mini
```

</Steps>

## Switch models in a session

Inside a Hermes session, switch models with `/model` and the `custom:<provider>:<model>` syntax:

```text
/model custom:neon:claude-opus-4-8
/model custom:neon:gemini-3-5-flash
/model custom:neon:kimi-k3
/model custom:neon-openai:gpt-5-5
```

To see which model IDs your gateway serves, list them:

```bash
curl -s "$NEON_AI_GATEWAY_BASE_URL/v1/models" \
  -H "Authorization: Bearer $NEON_AI_GATEWAY_TOKEN"
```

The [model catalog](/docs/ai-gateway/models) lists the same IDs and the endpoint each provider needs. Embedding models such as `qwen3-embedding-0-6b` appear in the list too, but Hermes can't chat with them.

## Use the gateway everywhere Hermes runs

Hermes uses your main model for more than the chat in front of you. Once `model.provider` points at Neon, the following all run through the gateway too.

### Telegram, Discord, Slack, WhatsApp, and Signal

The messaging gateway runs Hermes as a bot on the platforms you connect. Set it up and start it with:

```bash
hermes gateway setup
hermes gateway start
```

Each message to the bot is an agent turn on your main model. Inside a chat, `/model custom:neon:claude-haiku-4-5` works the same way it does in the terminal, which is useful if you want a cheaper model for quick questions from your phone.

### Cron jobs

Hermes has a built-in scheduler. You describe a job in plain language, for example a daily summary of your GitHub notifications, and Hermes runs it and delivers the result to any connected platform. Nobody watches these runs as they happen, so their cost is easy to miss. With Neon as the provider, they show up as gateway usage on the same Neon project as everything else.

### Subagents and auxiliary tasks

Hermes can spawn subagents to work on parallel tasks, and it uses a model for side tasks such as context compression, session titles, and vision. By default, those auxiliary tasks use your main model, so they also go through Neon. To send them to a cheaper gateway model, set the auxiliary provider to the same named entry. See [Auxiliary models](https://hermes-agent.nousresearch.com/docs/user-guide/configuration) in the Hermes docs.

## Coming from OpenClaw

If you already run OpenClaw, migrate first and switch the provider afterward:

```bash
hermes claw migrate --dry-run
hermes claw migrate
```

The migration imports your persona, memories, skills, command allowlist, and messaging settings. It can also import provider keys for OpenRouter, OpenAI, and Anthropic. If you want all inference on Neon, run `hermes claw migrate --preset user-data` to skip secrets, or delete those keys from `~/.hermes/.env` after migrating. Then add the Neon providers from the steps above.

## Rotate or revoke the credential

To rotate the token in place, find its ID and rotate it:

```bash
neon credentials list
neon credentials rotate <token_id>
```

Then replace `NEON_AI_GATEWAY_TOKEN` in `~/.hermes/.env` and restart any running `hermes gateway` process.

To remove the agent's access, revoke the credential:

```bash
neon credentials revoke <token_id>
```

See [Rotating credentials](/docs/ai-gateway/authentication#rotating-credentials) for the API equivalents.

## Troubleshooting

- **`401 Unauthorized`.** Hermes isn't sending the token. Check that `NEON_AI_GATEWAY_TOKEN` is in `~/.hermes/.env` and that the provider entry uses `key_env: NEON_AI_GATEWAY_TOKEN`. Use a named entry under `providers:` as shown above. Some Hermes versions ignore `key_env` on a bare `provider: custom` model block.
- **`403 Forbidden`.** The credential is missing the `ai_gateway:invoke` scope, or you created it on a branch outside the lineage of the host in your config.
- **`Function tools with reasoning_effort are not supported for gpt-5...`.** You're calling a GPT model through the `neon` entry. Use `custom:neon-openai:<model>` instead.
- **`Streaming is not supported for this model/provider`.** Hermes prints this before it falls back to a non-streaming request. On the gateway, it usually means the streaming request failed for another reason. Check `~/.hermes/logs/errors.log` for the underlying error, which is often the GPT error above.
- **`model "<model-id>" is not available`.** Check the ID against `/v1/models`. The gateway uses its own IDs, such as `claude-sonnet-4-6` and `gpt-5-4-mini`, not provider-prefixed names like `anthropic/claude-sonnet-4`.

For other errors, see [AI Gateway troubleshooting](/docs/ai-gateway/troubleshooting).

## Try it on one recurring job

Pick something you check by hand every day, such as open pull requests or a status page, and ask Hermes to turn it into a cron job that messages you the result. Let it run for a few days on a small model like `claude-haiku-4-5` or `gpt-5-4-mini`, then check your Neon AI Gateway usage to see what the job costs.

## Resources

- [Hermes Agent on GitHub](https://github.com/NousResearch/hermes-agent)
- [Hermes Agent documentation](https://hermes-agent.nousresearch.com/docs/)
- [Hermes LLM and model providers](https://hermes-agent.nousresearch.com/docs/integrations/providers)
- [Get started with Neon AI Gateway](/docs/ai-gateway/get-started)
- [AI Gateway authentication](/docs/ai-gateway/authentication)
- [AI Gateway models](/docs/ai-gateway/models)

<NeedHelp/>
