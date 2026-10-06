---
title: 'Run a multiplayer coding agent on Neon AI Gateway with Hoplite'
subtitle: 'Connect Neon AI Gateway to Hoplite so cloud coding agent threads, Slack requests, automations, and software factory runs in your organization use models billed through your Neon project.'
author: neon-team
enableTableOfContents: true
createdAt: '2026-10-03T00:00:00.000Z'
---

Your team has three coding agents running. One is fixing a flaky test, one is halfway through a migration, and one was started from Slack by someone who's now in a meeting. Each one calls a model provider with its own key, and nobody checks the bill until the end of the month.

[Hoplite](https://hoplite.sh) is a multiplayer coding agent backed by YCombinator. Every task runs as a thread in its own cloud sandbox, a Linux machine with your repository cloned and your setup script applied, and ends in a pull request. Threads are shared with the whole organization, so teammates can watch a run, steer it, approve sensitive steps, and pick up where someone left off.

[Neon AI Gateway](/docs/ai-gateway/overview) gives you one endpoint for models from OpenAI, Anthropic, Google, and others. You authenticate with a Neon credential instead of provider API keys. Hoplite has Neon AI Gateway built in as a gateway provider, so connecting the two takes a branch host and a token.

In this guide, you'll connect Neon AI Gateway to Hoplite, pick a gateway model for your threads, and use it in shared threads, parallel runs, Slack and iMessage, automations, and the Hoplite API.

## What you get from the combination

Hoplite runs the agents and controls who can work with them. Neon AI Gateway controls which models they can call and who pays for them. Together you get:

- **One credential for every model.** Hoplite discovers the models on your gateway and adds them to the model picker for everyone in the organization. Switching from a Claude model to a GPT model in the middle of a thread doesn't need a second provider key.
- **Inference on your Neon bill.** The account behind a gateway pays for its usage, so these runs draw on your Neon [prepaid credits](/docs/ai-gateway/prepaid-credits) instead of Hoplite credits. Hoplite marks them with a BYOK badge in the cost breakdown.
- **One place to cut access.** You can rotate or revoke the Neon credential at any time, and no developer has a provider key on their laptop to clean up.

## Prerequisites

- A Neon project on a paid plan with prepaid credits, in a region that supports AI Gateway. See [Get started with Neon AI Gateway](/docs/ai-gateway/get-started#get-access) for the current list.
- The [Neon CLI](/docs/cli). It's optional, but it's the fastest way to create a credential.
- A Hoplite organization with a connected GitHub repository. You need to be an organization owner or admin to add a gateway. If you don't see a **Gateways** section under **Settings** > **Organization** > **Models**, ask Hoplite to enable custom gateways for your organization.

<Steps>

## Create an AI Gateway credential

Hoplite needs a Neon credential with the `ai_gateway:invoke` scope. Give it a name you'll recognize later, so you know which token to rotate or revoke.

<Tabs labels={["CLI", "Console"]}>
<TabItem>

Run this in a directory [linked](/docs/cli/link) to your Neon project, or pass `--project-id` and `--branch`:

```bash
neon credentials create --scope ai_gateway:invoke --name hoplite
```

The CLI prints the `api_token` (it starts with `nt_live_`) once. Copy it somewhere safe for the next step.

</TabItem>
<TabItem>

In the Neon Console, click **Connect** at the top of the sidebar and open the **AI Gateway** tab. Click **Reveal credential** to show the token.

</TabItem>
</Tabs>

A credential works on the branch you created it on and on that branch's descendants. For a team setup, create it on your project's default branch. See [How branch binding works](/docs/ai-gateway/authentication#how-branch-binding-works) for details.

## Find your branch host

Hoplite also needs your branch's AI Gateway host. It's in the same **Connect** dialog, on the **AI Gateway** tab, as `NEON_AI_GATEWAY_BASE_URL`. It looks like this:

```text
https://br-winter-pond-aptw82ef-api.ai.c-2.us-east-2.aws.neon.tech
```

Use the bare host. Don't append `/v1` or `/anthropic`, and don't use your database connection string or an `ep-` endpoint host.

## Add Neon AI Gateway to Hoplite

In Hoplite, click **Settings** in the sidebar, then open **Models** under **Organization**. Scroll to **Gateways** and click **Add gateway**.

In the **Add gateway** dialog:

1. Under **Provider**, select **Neon AI Gateway**.
2. Enter a **Name** for the gateway, for example `Neon`.
3. Paste the URL from the previous step into **Neon branch host**.
4. Paste the `nt_live_...` token into **Neon AI Gateway API key**.
5. Click **Verify and add**.

![The Add gateway dialog in Hoplite with Neon AI Gateway selected](/guides/images/hoplite-neon-ai-gateway/hoplite-add-gateway.png)

Hoplite checks the key against Neon and adds the gateway's tool-capable models. The gateway then appears in the **Gateways** list with its number of verified models and an **Active** status.

![The Neon gateway listed as Active under Gateways in Hoplite](/guides/images/hoplite-neon-ai-gateway/hoplite-gateway-active.png)

Open the gateway to review its verified models.

![Verified models for the active Neon AI Gateway in Hoplite](/guides/images/hoplite-neon-ai-gateway/hoplite-gateway-models.png)

If verification fails, check that the host has no trailing path and that the token has the `ai_gateway:invoke` scope.

## Pick a gateway model for your threads

Verified gateway models show up in the model picker in every thread. Pick one in the new-thread dialog, or switch an existing thread with the `/model` slash command. To make a gateway model the default for a project, set it under **Settings** > **Project** > **Agents**.

Start with a small task, such as a failing test, and open the run's cost breakdown when it finishes. A BYOK badge means the run went through Neon AI Gateway.

</Steps>

## Use the gateway across Hoplite

Anything that starts a Hoplite thread can run on a gateway model.

### Multiplayer AI puts the team in one thread

A Hoplite thread shows up for everyone in the organization, not only the person who started it. Tool calls and diffs stream to whoever has it open, and any owner, admin, or member can answer an approval. Hoplite calls this multiplayer AI, meaning one agent context that teammates, integrations, and automations can all write into.

The thread keeps its model when someone else takes over. If a teammate picks it up at 9 a.m., it keeps running on the same gateway model and bills the same Neon project, with no personal API key involved.

### Swarm coding with parallel threads

Every Hoplite thread gets its own sandbox, so you can run several at once without port conflicts or a shared checkout. Three approaches to the same refactor means three machines and three pull requests.

Parallel threads can use different models from the same gateway. You might give test fixes to a fast model and a hard refactor to a larger one, and all of the usage still lands in one Neon project. Before you start the threads, split the work so they don't edit the same files. Check the combined result before you merge.

### A Slack coding agent and an iMessage coding agent

Mention Hoplite in Slack to start a coding thread in the right project, steer running work, and get the pull request link back in the same Slack thread. The Phone and iMessage integration lets you check on agents, relay instructions, and start threads by text.

Threads started from Slack or iMessage use the project's default model, so set a gateway model as the default if you want them on Neon. Hoplite pays for the Slack and iMessage assistant's own conversation. The coding thread it starts uses your model.

### Automations with AI

Hoplite Automations run coding agents on a schedule or when something happens in GitHub, Sentry, Linear, Slack, or a webhook. Each run is an ordinary thread your team can open, steer, and review.

Nobody watches a scheduled run as it happens, so its cost is easy to miss. Running automations on a gateway model puts that usage on your Neon bill with everything else. A nightly Sentry automation, for example, can reproduce new errors with a failing test and open a pull request, and each of those runs shows up as gateway usage.

### Build a software factory

Hoplite's API lets you build a software factory, a backend in your own product or internal tools that starts a budgeted task and gets back a draft pull request. Hoplite's [Build a software factory](https://hoplite.sh/docs) guide covers the full integration.

To use the gateway, call Hoplite's List models endpoint to find the current model IDs instead of hard-coding them. Pass the gateway model when you create a thread, and Neon AI Gateway bills every task your factory starts.

## Review the work and the model usage

People often describe tools like Hoplite as an AI software engineer. In practice, a cloud coding agent does the mechanical work in a sandbox and returns a pull request, and a person decides what merges. Hoplite keeps the diff, the transcript, and the test output for that review. Neon AI Gateway records which models each run called and what they cost.

## Rotate or revoke the credential

To rotate the token in place, find its ID and rotate it:

```bash
neon credentials list
neon credentials rotate <token_id>
```

Then update the API key on the Neon gateway under **Settings** > **Organization** > **Models** > **Gateways** in Hoplite.

To remove Hoplite's access, revoke the credential:

```bash
neon credentials revoke <token_id>
```

Runs on gateway models fail until you add a new credential. See [Rotating credentials](/docs/ai-gateway/authentication#rotating-credentials) for the API equivalents.

## Troubleshooting

- **`401 Unauthorized` during verification.** The token is missing or incomplete. Copy the full `nt_live_...` value again.
- **`403 Forbidden`.** The credential is missing the `ai_gateway:invoke` scope, or you created it on a branch outside the lineage of the host you entered. Create the credential on the same branch as the host, or on one of its ancestors.
- **A model you expected isn't in the picker.** Check the [model catalog](/docs/ai-gateway/models) and your project's region. Hoplite lists only the tool-capable models it verified on your gateway.

For other errors, see [AI Gateway troubleshooting](/docs/ai-gateway/troubleshooting).

## Try it on one ticket

Pick a ticket you've been putting off, start a Hoplite thread on a Neon AI Gateway model, and share the thread link with a teammate. When the pull request comes back, open the cost breakdown and check for the BYOK badge. If it's there, set the gateway model as the project default so your automations use it too.

## Resources

- [Hoplite documentation](https://hoplite.sh/docs)
- [Hoplite models and gateways](https://hoplite.sh/docs/agent/models)
- [Hoplite multiplayer coding agent](https://hoplite.sh/product/multiplayer)
- [Get started with Neon AI Gateway](/docs/ai-gateway/get-started)
- [AI Gateway authentication](/docs/ai-gateway/authentication)
- [AI Gateway models](/docs/ai-gateway/models)

<NeedHelp/>
