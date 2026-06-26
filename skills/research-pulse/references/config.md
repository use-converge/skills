# Config

The `converge` CLI uses a TOML config file for reusable endpoint, auth, model, advisor, progress, and timeout defaults.

## Discovery

The CLI discovers config files in this order:

1. `--config <path>`
2. `.converge.toml` in the current working directory
3. `~/.config/converge/config.toml`

Use flags for one-off overrides. Use config files and profiles for stable defaults that should be reused by people or agents.

## Precedence

For endpoint and auth values:

1. Command flags such as `--base-url` and `--api-key`
2. Environment variables `CONVERGE_BASE_URL` and `CONVERGE_API_KEY`
3. The selected profile in the config file
4. Top-level config file values

For profile selection:

1. `--profile <name>`
2. `CONVERGE_PROFILE`
3. Top-level `profile = "<name>"`

## Format

Supported top-level keys:

| Key | Description |
|-----|-------------|
| `version` | Config format version. Use `1`. |
| `profile` | Default profile name when `--profile` and `CONVERGE_PROFILE` are not set. |
| `base_url` | Default Converge API base URL. Usually better set per profile. |
| `api_key` | Default API key. Environment variables are safer for shared workspaces. |

Supported keys inside `[defaults]` and `[profiles.<name>]`:

| Key | Description |
|-----|-------------|
| `base_url` | Converge API base URL for this profile. |
| `api_key` | API key for this profile. Prefer `CONVERGE_API_KEY` when possible. |
| `models` | Selector strings returned by `converge pulse models`. |
| `synth_model` | Selector string for the synthesizer. Requires `include_synthesis = true`. |
| `include_synthesis` | Whether to request synthesis. |
| `include_summary` | Whether to include a summary. Requires synthesis. |
| `council_mode` | Whether to run selected models as an advisor council. Requires synthesis. |
| `use_saved_advisors` | Whether direct CLI runs should reuse saved advisor defaults. |
| `wait` | Whether run commands should wait for completion by default. |
| `progress` | One of `auto`, `always`, or `never`. |
| `poll_interval` | Poll interval as a duration, such as `2s`. |
| `timeout` | Overall wait timeout as a duration, such as `30m`. |

Supported profile-only key:

| Key | Description |
|-----|-------------|
| `extends` | Name of another profile to inherit before applying this profile's overrides. |

Advisor assignments use repeated `[[...council_assignments]]` blocks under `[defaults]` or a profile:

| Key | Description |
|-----|-------------|
| `model_selector` | Selector this assignment applies to. |
| `advisor_id` | Stored advisor ID to use for that selector. |
| `template_key` | Advisor template key to use for that selector. |

Use either `advisor_id` or `template_key` in a single assignment.

## Example Config

```toml
version = 1
profile = "default"

[defaults]
progress = "auto"
wait = true
poll_interval = "2s"
timeout = "30m"
use_saved_advisors = true

[profiles.default]
base_url = "https://converge.example.com"
# Use exact selector strings from `converge pulse models`.
models = ["gpt-5-mini", "claude-sonnet-4", "gemini-3-flash-preview"]
synth_model = "gpt-5-mini"
include_synthesis = true
include_summary = false
council_mode = true

[profiles.production]
extends = "default"
base_url = "https://converge.example.com"

[profiles.fast]
extends = "default"
models = ["gpt-5-mini"]
include_synthesis = false
council_mode = false

[profiles.agent]
extends = "default"
progress = "never"
wait = false

[[profiles.default.council_assignments]]
model_selector = "claude-sonnet-4"
template_key = "contrarian"

[[profiles.default.council_assignments]]
model_selector = "gpt-5-mini"
advisor_id = "adv_123"
```

If a profile extends another profile, list values such as `models` and `council_assignments` are replaced by the child profile when present; they are not appended.

## Minimal Local Config

```toml
version = 1
profile = "local"

[profiles.local]
base_url = "http://localhost:8080"
models = ["gpt-5-mini", "claude-sonnet-4"]
synth_model = "gpt-5-mini"
include_synthesis = true
```

## Agent Profile Pattern

For unattended agent runs, prefer a profile that disables terminal progress and avoids waiting unless the command explicitly asks to wait:

```toml
[profiles.agent]
extends = "default"
progress = "never"
wait = false
```

## Notes

- Storing `api_key` in the config file is supported, but `CONVERGE_API_KEY` is safer when the workspace is shared.
- `--model` flags on `converge pulse run` override configured `models`.
- `--synth-model` overrides the configured `synth_model`.
- `--advisor` assignments override config assignments for the same selector.
- `--use-saved-advisors` opts into saved advisor defaults for that direct run.
- `--quiet` forces progress off even if the profile sets `progress = "auto"` or `"always"`.
- `CONVERGE_BASE_URL` overrides profile-specific `base_url`.
- `CONVERGE_PROFILE` selects a profile unless `--profile` is provided.
