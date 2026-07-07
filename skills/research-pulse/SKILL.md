---
name: research-pulse
description: Manage advisor-aware Converge Research Pulse workflows through the supported `converge` CLI path, wait for completion, manage share feeds, and write Markdown or JSON artifacts into the workspace. Use when Codex needs to manage Advisors, run a Research Pulse for a user, export results to files, inspect or fetch account share feeds, reuse config profiles, package artifact-only review context, triage model outputs, or drive Research Pulse from an agent workflow without calling internal Go packages directly.
---

# Research Pulse

Use this skill to run Research Pulses through the external CLI and API contract instead of reaching into internal Go packages or HTML endpoints.

## Workflow

1. Ensure the CLI path exists or install it with Homebrew.

- Prefer a `converge` binary already on `PATH`.
- If `converge` is missing and `brew` is available, install the public binary-only CLI:

```bash
brew tap use-converge/cli
brew trust use-converge/cli
brew install converge
```

- If `converge` is already installed with Homebrew but needs the latest available release, run `brew update` and `brew upgrade converge`.
- If Homebrew is unavailable, say that the CLI is missing and ask the user to install `converge` or provide a path to it.

2. Resolve auth, base URL, and defaults before submitting a run.

- Prefer `--config` and `--profile` when a reusable config exists.
- Otherwise use `--base-url` and `--api-key`, or the `CONVERGE_BASE_URL` and `CONVERGE_API_KEY` env vars.
- If selectors are not already known, run `converge pulse models` first.
- Prefer selectors with stronger recent reliability and lower average latency when the task benefits from fast turn time.
- Use `converge pulse models --json` when an agent needs machine-readable selector stats; each item includes recent `reliability_percent`, `avg_latency_ms`, `p50_latency_ms`, `p95_latency_ms`, and grouped/daily breakdowns from the same 7-day stats shown in the Providers UI.
- Read [references/config.md](references/config.md) for config discovery, precedence, and example profiles.
- Before any paid or state-changing command, state the effective config path, profile name, base URL, and API-key source without printing the key. If the resolved environment is ambiguous, or a local/dev/staging base URL is paired with a production-looking profile, stop and ask for explicit approval or a corrected `--config`, `--profile`, or `--base-url`.
- In local checkout testing, use the project-provided environment loader when it exists, for example `direnv exec .`, and clear unrelated profile defaults when you need to force localhost behavior. Do not let saved production profile defaults silently steer a local test.
- For connectivity and selector sanity checks, use `converge pulse models --json` before a paid run. Treat selector failures from an external router or gateway as model/provider availability problems, not as prompt-truncation evidence.

3. Inspect or manage Advisors when the task needs perspective control.

- In advisor-capable Converge builds, prefer the top-level `converge advisors` command family for reusable council lenses.
- Start with `converge advisors templates` or `converge advisors list` when you need to see available options.
- Use `converge advisors create` or `converge advisors update` with `--instructions-file` when the instructions are more than a short line.
- Reuse stored advisors or templates in run commands instead of inlining large instruction blocks into shell arguments.
- If the connected build does not expose advisor CLI support yet, say so plainly and continue with plain Research Pulse commands or existing web-managed advisors.
- When using `--advisor` or `--use-saved-advisors`, verify that the run output echoes the intended resolved advisor mapping before treating the results as final. If an assignment references a model selector that is not in the run, fix the mapping and rerun only after confirming the corrected target list.

4. Stage the question in the workspace.

- If the user already has a question file, use it with `--question-file`.
- Otherwise write the question into a workspace file such as `.codex/tmp/research-pulse-question.md` and point the CLI at that file.
- Keep secrets out of workspace files.
- Keep the submitted question body at or below 20,000 characters. The CLI, API, and web UI reject longer question text before starting a run.
- Put long source material in Research Pulse document uploads instead of the question body. In the CLI, pass one or more `--document <path>` flags. In the API, send `multipart/form-data` to `/api/v1/research-pulses` with repeated `documents` file parts plus the normal create fields.
- Document uploads support PDF, DOCX, Markdown, and text files, up to 5 documents and 10 MB each. Unsupported, empty, unreadable, or oversized documents should be fixed before rerunning; dry-runs apply the same document admission checks and echo accepted document metadata. Web and CLI warn when any document is larger than 5 MB. Server-side provider-size validation hard-fails Gemini-backed runs when any attached document is larger than 5 MB. Do not paste a whole oversized document into the question as a workaround.
- Treat upload admission, extracted-text retention, and model-visible context as separate limits. Converge accepts up to the documented upload cap and stores extracted text for provenance/fallback, but long documents are sampled into stage-aware prompt excerpts for text-fallback providers, while native-file providers may enforce their own file-token limits. In a five-member production council that included Gemini 3.5 Flash, synthetic ASCII Markdown succeeded through gathering, review, chairman synthesis, and summary at 5,505,024 bytes and failed at 5,570,560 bytes because Gemini rejected the native-file input as over its 1,048,576-token limit. The current Converge provider-size matrix therefore blocks Gemini stacks above 5 MB. For unattended five-member production council runs with Gemini in the stack, keep Markdown uploads at or below 5 MB unless the task explicitly calls for a capped boundary test. The practical ceiling is content-dependent because provider tokenization varies by document format and text.
- Measure the staged file before submitting, for example with `wc -m <question-file>`. Prefer a safe buffer below the hard cap for large packets; if the packet is near the cap, keep only the objective, non-negotiable constraints, key facts, evidence excerpts, and explicit verification requests.
- If you summarize source files into the question instead of uploading them, label summarized sections as summaries and include source paths or line ranges when relevant.
- For near-limit packets, add a final `Completeness Check` or tail sentinel in the last 1-2k characters. Use a distinctive token or exact final sentence that reviewers and later JSON/share inspection can verify.

5. For plan, code, architecture, or artifact review, build an access-aware context packet.

- Assume Research Pulse reviewers cannot inspect the local checkout, private filesystem, screenshots, browser state, or prior Codex reasoning unless the submitted question includes the relevant facts.
- Label the review mode near the top of the question:
  - `artifact-only`: reviewers can only see the submitted question/document.
  - `code-aware`: reviewers can inspect a repo through an explicitly available external tool. Research Pulse is usually not this mode.
  - `hybrid`: Codex or another local agent inspected code and summarized the important code facts into the packet.
- For artifact-only or hybrid review, include:
  - the concrete failure, decision, or artifact being evaluated;
  - exact observed evidence, traces, commands, paths, and outputs that matter;
  - relevant code facts already inspected by Codex, clearly labeled as packet evidence;
  - constraints reviewers must honor;
  - items already reviewed or accepted by other reviewers;
  - the findings that are useful now, such as missed failure modes, ambiguities, missing tests, or rollout risks.
- Tell reviewers to classify claims using:
  - `Confirmed from packet`: directly supported by included evidence.
  - `Gap in submitted design`: missing or ambiguous in the artifact.
  - `Needs repo verification`: plausible, but not provable without code access.
  - `Already covered`: present in the artifact.
  - `Suspect`: depends on truncation, missing context, or an unsupported assumption.
- Tell reviewers not to say "the code does X" unless the packet includes that code fact. Prefer "if the code does X" or "verify whether X" when code access is absent.
- If the question is near the limit, put a short "Completeness Check" near the end that tells reviewers what the final visible section should be. This helps identify provider-side context pressure separately from Converge upload/share handling.
- For near-limit packets, tell reviewers to include the sentinel or final visible sentence when they can see the tail. Absence of that marker in useful model outputs is a truncation or context-pressure signal.

6. Run the pulse through the CLI.

- For a new run that should finish in this turn, use `converge pulse run --wait`.
- Pass `--output` for Markdown, `--json-output` for full JSON, or both.
- For any automated paid run, especially multi-model or council runs, pass `--max-spend-usd <amount>` unless the user explicitly disables caps. Treat each rerun as a new cost center that needs its own cap.
- Add `--model` and `--synth-model` only when overriding config defaults.
- Attach source files with repeatable `--document <path>` flags when the run needs original document fidelity or source material that would crowd the question limit. Treat any CLI warning about documents larger than 5 MB as actionable; remove Gemini from the stack or reduce/split the file before submitting if Gemini is selected.
- In advisor-capable builds, use `--advisor '<selector>=advisor:<id>'` or `--advisor '<selector>=template:<key>'` for explicit assignments.
- In advisor-capable builds, `--use-saved-advisors` is the explicit opt-in path for reusing saved advisor defaults on a direct CLI run.
- When advisors are applied, verify that the CLI echoes the resolved advisor mapping before treating the run input as final.
- Leave progress enabled unless the user explicitly wants quiet output.
- Prefer a low-risk smoke ladder before expensive work: `converge pulse models --json`, then a tiny capped run with non-sensitive content, then the full run. Validate output paths and JSON shape on the smoke run before spending on a large council run.

7. Reuse existing runs when appropriate.

- Use `converge pulse watch <run-id>` to wait on an existing run.
- Use `converge pulse export <run-id>` to fetch artifacts after completion.

8. Triage review quality before summarizing important findings.

- Prefer writing both Markdown and JSON when review quality matters.
- Inspect the JSON first when a model claims the input was truncated, incomplete, or missing context. Confirm `.question` length and tail match the staged question when the JSON shape exposes the submitted question, and verify the `Completeness Check` marker when one was used.
- For document-size experiments, inspect `.documents[]` for `byte_size`, `extracted_char_count`, `text_truncated`, and `blob_available`, then inspect each response/review/summary status. A run that reaches chairman output can still be `partial_failed` if one council member failed in gathering; do not call that a clean no-error size.
- If a public share exists, fetch it with `converge share markdown <slug>` and compare the embedded question against the local staged question before concluding that Converge truncated the upload or share rendering.
- Check per-model status and output length in the JSON. Mark tiny outputs, truncation-only outputs, policy-only outputs, or outputs that ignore the access model as suspect.
- If JSON/share inspection confirms a tail mismatch, mark the run as packet-truncated and rebuild a smaller complete packet before rerunning.
- Do not rerun solely because one model reports truncation if the JSON/share contain the full question. Rerun at most once automatically, and only when multiple useful models appear context-limited, the chair output is dominated by suspect responses, or the missing content includes objective, constraints, evidence, or the tail marker.
- When reporting back, separate:
  - genuine design gaps;
  - findings already covered locally;
  - findings that need repo verification;
  - suspect model output.
- Treat Research Pulse as gap discovery and artifact review, not as a substitute for local checkout verification or code-aware reviewers.

9. Report the outcome precisely.

- If Markdown or JSON files were written, report the exact paths.
- Exit code `0` means success; proceed with normal artifact and quality triage.
- Exit code `2` means partial failure with a usable result; preserve artifacts, report failed providers separately from successful synthesis/review output, and decide whether a rerun is warranted by packet integrity or model availability.
- Exit code `1` means request, auth, config, billing, or terminal failure without a usable final result. Fix the specific cause before retrying.
- Capture the run ID exactly from CLI output and reuse it for `watch`, `export`, `share status`, and post-run triage.
- If the CLI reports insufficient prepaid credits, do not retry the same paid operation blindly. Check `converge billing balance`, top up if appropriate, then rerun with an explicit `--max-spend-usd` cap.
- If the CLI reports `max_spend_exceeded`, do not remove or raise the cap reflexively. Either reduce scope, choose cheaper models, or raise it intentionally after user approval.

10. Check or top up prepaid billing credits when needed.

- Use `converge billing balance` to inspect available and reserved prepaid credits.
- Use `converge billing skus` before choosing a top-up SKU. Enabled top-up SKUs enforce the current $10 minimum, so do not assume smaller top-ups exist.
- Use `converge billing top-up --sku <sku-id>` to request a scoped x402 payment challenge.
- Use `converge billing ledger --limit <n>` when you need recent credit/hold/capture/release history. There is no current `converge billing reservations` command; reservation detail is available through browser Settings/Admin billing surfaces.
- In headless environments, use `--signer-command <command>` only when the runtime provides a trusted signer. The CLI sends a JSON payload on stdin and expects the signed payment payload on stdout.
- Treat `--payment-signature` as an explicit manual/debug path, and pair it with a stable `--idempotency-key`.
- Never store wallet private keys or raw payment signatures in the Converge config file or workspace notes.

11. Use supported share and share-feed commands for published Research Pulse outputs.

- Use `converge share status <run-id>` to inspect whether a run has an active or revoked public share and to print both the HTML and Markdown URLs.
- Use `converge share enable <run-id>` to create or re-enable a public share for a completed run.
- Use `converge share disable <run-id>` to revoke an active public share.
- Share status/enable/disable use the authenticated API-v1 Research Pulse share contract and require `api.full`.
- Confirm the target environment before `share enable`, `share disable`, `feed enable`, `feed disable`, `feed rotate`, `feed set-password`, or `feed clear-password`. These are state-changing account operations.
- Use `converge feed status` to inspect whether the account feed exists, is enabled, and has password protection.
- Use `converge feed enable`, `converge feed disable`, `converge feed set-password`, `converge feed clear-password`, and `converge feed rotate` for management.
- Use `converge feed urls` to print Atom/RSS/JSON URLs. URL discovery requires `api.full` or a browser session; a `shares.feed`-only key can read an already-known URL but cannot discover it.
- Use `converge feed fetch --format atom|rss|json` to fetch or validate the feed. Add `--url` when operating with a least-privilege `shares.feed` key.
- Use `converge share markdown <slug>` to fetch a public share as Markdown. Do not scrape the HTML page.
- Feed entries are intentionally link-only. Fetch the linked share page, or `converge share markdown <slug>`, for the complete public artifact.
- If the connected CLI build does not expose `feed`, `share status`, `share enable`, `share disable`, or `share markdown`, say so plainly and fall back only to existing public share URLs.

## Command Patterns

Install the CLI outside this repo, when Homebrew is available:

```bash
brew tap use-converge/cli
brew trust use-converge/cli
brew install converge
```

Discover selectors:

```bash
converge pulse models --config <path> --profile <name>
```

Discover selectors plus machine-readable recent stats:

```bash
converge pulse models --config <path> --profile <name> --json
```

Submit a run and wait for Markdown plus JSON:

```bash
converge pulse run \
  --wait \
  --question-file <workspace-question-path> \
  --max-spend-usd 2.50 \
  --output <workspace-result-path>.md \
  --json-output <workspace-result-path>.json \
  --config <path> \
  --profile <name>
```

Override models for a one-off run:

```bash
converge pulse run \
  --wait \
  --question-file <workspace-question-path> \
  --model gpt-5-mini \
  --model claude-sonnet-4 \
  --synth-model gpt-5-mini \
  --output <workspace-result-path>.md
```

Advisor-aware commands, when supported by the connected Converge build:

```bash
converge advisors templates
```

```bash
converge advisors create \
  --name "Contrarian" \
  --description "Looks for likely failure modes." \
  --instructions-file <workspace-advisor-path>.md
```

```bash
converge pulse run \
  --wait \
  --question-file <workspace-question-path> \
  --council \
  --model gpt-5 \
  --model claude-sonnet-4 \
  --advisor 'gpt-5=template:contrarian' \
  --advisor 'claude-sonnet-4=advisor:adv_123' \
  --output <workspace-result-path>.md
```

```bash
converge pulse run \
  --wait \
  --question-file <workspace-question-path> \
  --council \
  --model gpt-5 \
  --model claude-sonnet-4 \
  --use-saved-advisors \
  --output <workspace-result-path>.md
```

```bash
converge pulse helper start \
  --message "Help me refine this question." \
  --model gpt-5 \
  --model claude-sonnet-4 \
  --council
```

Submit a Question Helper session with a prepaid-credit cap:

```bash
converge pulse helper submit <session-id> --max-spend-usd 2.50
```

Check prepaid credits:

```bash
converge billing balance --config <path> --profile <name>
```

List top-up SKUs:

```bash
converge billing skus --config <path> --profile <name>
```

Request a top-up challenge:

```bash
converge billing top-up --sku usd_10 --config <path> --profile <name>
```

Settle a top-up with a headless signer, when the runtime provides one:

```bash
converge billing top-up \
  --sku usd_10 \
  --idempotency-key <stable-key> \
  --signer-command <payment-signer-command>
```

Wait on an existing run:

```bash
converge pulse watch <run-id> \
  --output <workspace-result-path>.md \
  --json-output <workspace-result-path>.json
```

Export artifacts for a completed run:

```bash
converge pulse export <run-id> \
  --output <workspace-result-path>.md \
  --json-output <workspace-result-path>.json
```

Create or re-enable a public share for a completed run:

```bash
converge share enable <run-id> --config <path> --profile <name>
```

Inspect a run's public-share status and links:

```bash
converge share status <run-id> --config <path> --profile <name>
```

Disable a run's public share:

```bash
converge share disable <run-id> --config <path> --profile <name>
```

Inspect account share-feed status:

```bash
converge feed status --config <path> --profile <name>
```

Enable the share feed:

```bash
converge feed enable \
  --title "Research Pulse Shares" \
  --description "Published Converge Research Pulse outputs." \
  --config <path> \
  --profile <name>
```

Print feed URLs:

```bash
converge feed urls --config <path> --profile <name>
```

Fetch a feed with URL discovery:

```bash
converge feed fetch --format atom --config <path> --profile <name>
```

Fetch a known feed URL with a least-privilege key:

```bash
converge feed fetch \
  --format json \
  --url <feed-url> \
  --base-url <converge-base-url> \
  --api-key <shares.feed-token>
```

Rotate the feed URL only after explicit confirmation:

```bash
converge feed rotate --confirm --config <path> --profile <name>
```

Fetch a public share as Markdown:

```bash
converge share markdown <share-slug> --output <workspace-share-path>.md
```
