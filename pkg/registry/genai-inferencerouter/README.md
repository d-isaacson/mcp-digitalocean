# GenAI Inference Router MCP Tools

## What is a model router?

A **model router** (sometimes called an inference router) is a **named GenAI configuration** in your DigitalOcean account. It does not run inference by itself; it defines **how your app or agent should choose models** for different kinds of work. Concretely, a router has:

- **Policies** — Each policy ties a **task** to an **ordered list of model ids** and a **selection policy** (for example, prefer the fastest or cheapest model among the candidates for that task). A task is either a **built-in** `task_slug` (e.g. code generation, summarization) or a **custom task** you describe with a name and short `description`.
- **Fallback models** — A **required** ordered list the API can use when primary model choices in a policy are not available, so traffic still has a path to complete.

When you use these MCP tools, you are creating, reading, or deleting that **routing configuration** through the typed **`godo.GradientAI`** client (same auth, base URL, and transport as the rest of this MCP server). This MCP does not expose **update**; use the control panel, `godo.GradientAI.UpdateInferenceRouter`, or another client to change an existing router. List and get responses are formatted JSON matching the API shape (`model_routers` / `model_router` and `config`).

## godo surface

Calls map to:

- `GradientAI.CreateInferenceRouter` — create (`POST /v2/gen-ai/models/routers`)
- `GradientAI.ListInferenceRouters` — list (`GET …` with `page`, `per_page`)
- `GradientAI.GetInferenceRouter` — get by UUID
- `GradientAI.DeleteInferenceRouter` — delete by UUID

Preset tasks for policies are available on the client as `GradientAI.ListInferenceRouterTaskPresets` (not wrapped as an MCP tool here).

## Built-in `task_slug` values (how to choose)

**There is no MCP tool in this project that returns the full catalog of valid `task_slug` strings**, and the router HTTP surface above does not include a “list task types” route in this client. The API is the source of truth: an unknown slug typically fails at create with an error (for example, task slug not found), and the set can grow over time.

**Practical ways to pick a slug:**

1. **Copy from an existing router** — Call `genai-inference-router-list` or `genai-inference-router-get` on a router that already has built-in tasks and read each policy’s `task_slug` under `model_router.config.policies`.
2. **Avoid slugs for one-off work** — Use a **`custom_task`** policy with `name` and `description` instead of `task_slug` (the [e2e test](../../../testing/e2e_genai_inferencerouter_test.go) does this so tests do not depend on a specific catalog in every environment).
3. **Check product / API documentation** — If DigitalOcean publishes a definitive list in the GenAI or Gradient docs, treat that as authoritative for your environment.

**Built-in `task_slug` values** (the platform’s `model_router_task_presets` set—23 slugs, grouped for scanning). The API is still authoritative if a slug is missing in your account or region:

- **General:** `brainstorming-ideation`, `classification-labeling`, `opinion-advice-recommendation`, `planning-task-decomposition`, `summarization`, `text-extraction-structured-output`, `translation`
- **Writing:** `creative-writing`, `email-professional-communication-drafting`, `long-form-article-blog-writing`, `rewriting-editing`, `social-media-short-form-content`
- **Software engineering:** `bug-fixing`, `code-completion-inline`, `code-generation`, `code-performance-optimization`, `test-writing-code-verification`
- **Knowledge base & document intelligence:** `knowledge-base-customer-support`, `long-context-retrieval-aggregation`, `long-document-qa`, `rag-system-quality-evaluation`, `retrieval-quality-cross-domain-ir`, `text-and-table-grounded-reasoning`

If the platform exposes a **documented** HTTP route to list task slugs, a read-only MCP tool that proxies it would be preferable to a static list. The `godo` client exposes **`ListInferenceRouterTaskPresets`** for preset tasks (not wired as an MCP tool in this repo).

## Tools

### `genai-inference-router-create`

**Arguments**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `Name` | string | yes | Router name |
| `PoliciesJson` | string | no | JSON array of policies (omit or `[]` if the API allows); see below |
| `FallbackModels` | string[] | **yes** | At least one model id, sent as `fallback_models` (required by the API) |

**Create request body (via godo):** `name`, optional **`policies`**, and required non-empty **`fallback_models`**. Each policy must define a **task**:

- **Built-in task:** set **`task_slug`** (e.g. `code-generation`, `summarization`, `bug-fixing`) and **`models`** (ordered model id strings). Include **`selection_policy`** with **`prefer`** set to `fastest` or `cheapest`.
- **Custom task:** use **`custom_task`** with **`name`** and **`description`** instead of `task_slug`, and still set **`models`** and **`selection_policy`** as needed.

Policies that only set `model` plus `usecase_class` (with no task) are rejected by the API with an error like `policy 0 task is required`.

**Example** (equivalent JSON body sent by godo; `PoliciesJson` is only the `policies` array):

```json
{
  "name": "my-router",
  "policies": [
    {
      "task_slug": "code-generation",
      "models": ["openai-gpt-5", "anthropic-claude-4.6-sonnet"],
      "selection_policy": { "prefer": "fastest" }
    }
  ],
  "fallback_models": ["openai-gpt-oss-120b"]
}
```

**Custom task** policy example:

```json
{
  "custom_task": {
    "name": "Code reviewer",
    "description": "Review patches for correctness and style."
  },
  "models": ["openai-gpt-5.2"],
  "selection_policy": { "prefer": "cheapest" }
}
```

Pass the contents of `policies` (a JSON array) as the `PoliciesJson` string. **List** and **get** return `model_router.config.policies` in the same general shape (`task_slug` or `custom_task`, `models`, `selection_policy`). List summaries and full router payloads may include **`regions`** when the API returns them; that field is not set by this MCP on create (see current `godo.InferenceRouterCreateRequest`).

### `genai-inference-router-list`

**Arguments**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `Page` | number | no | Page (default 1) |
| `PerPage` | number | no | Page size (default 1000, max 1000) |

### `genai-inference-router-get`

**Arguments**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `UUID` | string | yes | Model router UUID |

### `genai-inference-router-delete`

**Arguments**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `UUID` | string | yes | Model router UUID to delete |

Returns formatted JSON (typically `{"uuid":"..."}`). If the API responds with an empty body on success, the tool still returns a JSON object containing the requested UUID.

## Enabling the service

Register with `--services genai-inferencerouter` (or include it in `SERVICES`). A valid `DIGITALOCEAN_API_TOKEN` is required.

## Notes

- The GenAI model router API **requires** at least one `fallback_models` entry on create, so the MCP enforces a non-empty `FallbackModels` list for create (mirroring `godo` client validation).
- Preview / unreleased APIs may only work on specific API hosts or accounts.
- Response bodies are returned as formatted JSON text.
