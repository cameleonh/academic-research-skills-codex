# Cross-Model Verification Protocol for Codex (v3.0-codex)

## Overview

This protocol enables optional cross-model verification for high-stakes AI
judgments. In the Codex package, the current Codex agent remains the primary
reviewer or verifier. When cross-model verification is explicitly enabled, the
external verifier must use the Anthropic Claude Opus 4.7 API.

Do not route the Codex package's cross-model reviewer through the Codex API,
OpenAI API, or the old upstream GPT/Gemini verifier paths. The Codex port uses
Claude Opus 4.7 as the external reviewer so the verifier is independent from
Codex's own runtime.

**This is entirely optional.** All ARS skills work without cross-model API
calls. Cross-model verification is an additional layer for users who want higher
confidence in integrity checks, devil's advocate challenges, and reviewer
calibration.

## Why Cross-Model Verification

A stress test of 68 AI-generated citations found 31% had problems, and all
passed three rounds of same-model integrity checks. The root cause: the
verifying AI and the generating AI can share training-data distribution and
review habits, so they can share blind spots. A different model family can catch
errors that the primary runtime systematically misses.

**What it improves:** Error-rate reduction by adding an independent review pass.
Different model families can catch different hallucination and reasoning
patterns.

**What it does not solve:** Frame-lock, shared source-corpus gaps, and
sycophancy. These are degree improvements, not kind improvements.

## Supported Cross-Model API

| Model | API ID | Provider | Best For |
|-------|--------|----------|----------|
| Claude Opus 4.7 | `claude-opus-4.7` | Anthropic | Codex package cross-verification reviewer |

Recommended pair for this repo: current Codex primary runtime plus Claude Opus
4.7 via Anthropic API as the external verifier.

If an Anthropic account exposes a different deployment identifier for Claude
Opus 4.7, use the account's exact model ID in `ARS_CROSS_MODEL` and disclose the
display name as "Claude Opus 4.7" in the report. Otherwise, use
`claude-opus-4.7`.

## Setup Guide

### Prerequisites

Cross-model verification requires an Anthropic API key. Do not enable it unless
the user explicitly asks for external model verification and accepts the extra
API cost.

### Step 1: Get API Key

Create an Anthropic API key in the Anthropic Console and keep it outside the
repository.

### Step 2: Set Environment Variables

Set these in the shell running Codex:

```bash
# Optional: cross-model verification for ARS Codex package
export ANTHROPIC_API_KEY="sk-ant-your-key-here"
export ARS_CROSS_MODEL="claude-opus-4.7"
```

### Step 3: Verify Setup

In Codex, ask:

```text
Use $academic-research-suite to check if ARS cross-model verification is
available.
```

The agent should check `ARS_CROSS_MODEL` and `ANTHROPIC_API_KEY`, then report
whether Claude Opus 4.7 verification is available.

### Step 4: Enable Per Session

If the user does not want cross-model verification running all the time, enable
it only for a specific terminal session:

```bash
export ANTHROPIC_API_KEY="sk-ant-your-key-here"
export ARS_CROSS_MODEL="claude-opus-4.7"

# Disable for this session
unset ARS_CROSS_MODEL
```

## How It Works in Each Skill

### Integrity Verification (academic-pipeline, Stage 2.5 / 4.5)

**When `ARS_CROSS_MODEL` is set to Claude Opus 4.7 and
`ANTHROPIC_API_KEY` is available:**

- The current Codex agent runs full Phase A-E verification as normal.
- After Phase A completes, a random 30% sample of references is sent to Claude
  Opus 4.7 for independent verification.
- Claude Opus 4.7 receives only the reference text and paper context, not the
  primary verification result, to prevent anchoring.
- Disagreements are flagged as `[CROSS-MODEL-DISAGREEMENT]` and prioritized for
  human review.

**When cross-model verification is not configured:**

- Standard single-runtime verification runs unchanged.

**Implementation for agents:**

When `integrity_verification_agent` detects `ARS_CROSS_MODEL` in the
environment, it should:

1. Complete Phase A verification normally.
2. Select 30% of references randomly (minimum 5, maximum 15). If total
   references are fewer than 5, sample all of them.
3. Batch up to 5 references per API call to reduce latency. For each batch,
   construct a verification prompt:

   ```text
   Verify each of these academic references independently. For each,
   check: Does it exist? Are the author names, year, title, journal,
   and DOI correct? Search the web to confirm.

   For each reference, respond with:
   VERIFIED / NOT_FOUND / MISMATCH (with details)

   Reference 1: [full reference text] -- Context: [sentence where cited]
   Reference 2: [full reference text] -- Context: [sentence where cited]
   ... (up to 5 per batch)
   ```

4. Send to Claude Opus 4.7 through the Anthropic Messages API.
5. Compare results. If the primary verifier said `VERIFIED` but Claude Opus 4.7
   said `NOT_FOUND` or `MISMATCH`, flag
   `[CROSS-MODEL-DISAGREEMENT]`.
6. Include disagreements in the integrity report:

   ```markdown
   ### Cross-Model Verification Results
   - External verifier: Claude Opus 4.7 via Anthropic API
   - References sampled: X/Y (Z%)
   - Agreements: N
   - Disagreements: M (listed below, prioritized for human review)

   | # | Reference | Primary | Claude Opus 4.7 | Status |
   |---|-----------|---------|-----------------|--------|
   ```

### Devil's Advocate (deep-research + academic-paper-reviewer)

**When `ARS_CROSS_MODEL` is set to Claude Opus 4.7 and
`ANTHROPIC_API_KEY` is available:**

- After the devil's advocate completes its standard review or checkpoint,
  Claude Opus 4.7 receives the same material and generates an independent
  critique.
- The devil's advocate compares results. Any CRITICAL or MAJOR issues found by
  Claude Opus 4.7 but not by the primary report are added as
  `[CROSS-MODEL-FINDING]`.

**When cross-model verification is not configured:**

- Standard single-runtime devil's advocate review runs unchanged.

**Implementation:**

The devil's advocate agent, after completing its checkpoint report, should:

1. Send the reviewed material plus a simplified devil's advocate prompt to
   Claude Opus 4.7:

   ```text
   You are a devil's advocate reviewing this [research/paper].
   Find the 3 most serious weaknesses. For each, state:
   - What the weakness is
   - Why it matters
   - What the strongest counter-argument would be

   Material: [the reviewed content]
   ```

2. Compare Claude Opus 4.7 findings with the primary findings.
3. Add any novel CRITICAL or MAJOR issue to the report as
   `[CROSS-MODEL-FINDING]`.
4. Log:
   `[CROSS-MODEL: Claude Opus 4.7 returned X findings, Y novel]`.

### Peer Review (academic-paper-reviewer)

Calibration mode can run one of the five review passes through Claude Opus 4.7
when the API is configured. The external result must be shown separately in the
calibration report; do not silently average Claude Opus 4.7 scores into the
primary Codex scores.

Future sixth-reviewer behavior may add Claude Opus 4.7 as a separate reviewer
inside `eic_agent.md`. Until then, cross-model peer review remains limited to
devil's advocate cross-checking and calibration-mode verification.

## API Call Pattern

### Anthropic Claude Opus 4.7

Use the Anthropic Messages API. `PROMPT` must be set before calling. The example
uses `jq` to build valid JSON; if `jq` is unavailable, use Python's `json`
module for equivalent escaping.

```bash
MODEL="${ARS_CROSS_MODEL:-claude-opus-4.7}"

jq -n \
  --arg model "$MODEL" \
  --arg prompt "$PROMPT" \
  '{
    model: $model,
    max_tokens: 2000,
    temperature: 0.1,
    system: "You are an independent academic verification assistant.",
    messages: [
      {
        role: "user",
        content: $prompt
      }
    ]
  }' |
curl -s https://api.anthropic.com/v1/messages \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d @- |
jq -r '.content[] | select(.type == "text") | .text'
```

### Detecting Availability

Agents should check at the start of a verification or review session:

```bash
if ! command -v jq >/dev/null 2>&1; then
  echo "WARNING: jq not installed. Use python3 JSON parsing fallback for API calls."
fi

if [ -n "$ARS_CROSS_MODEL" ]; then
  case "$ARS_CROSS_MODEL" in
    claude-opus-4.7|claude-opus-4-7)
      if [ -n "$ANTHROPIC_API_KEY" ]; then
        echo "CROSS_MODEL_AVAILABLE=anthropic"
        echo "CROSS_MODEL_REVIEWER=Claude Opus 4.7"
      else
        echo "WARNING: ARS_CROSS_MODEL=$ARS_CROSS_MODEL but ANTHROPIC_API_KEY is not set"
        echo "CROSS_MODEL_AVAILABLE=none"
      fi
      ;;
    *)
      echo "WARNING: ARS_CROSS_MODEL=$ARS_CROSS_MODEL is not supported in the Codex package."
      echo "Supported cross-model reviewer: claude-opus-4.7"
      echo "CROSS_MODEL_AVAILABLE=none"
      ;;
  esac
else
  echo "CROSS_MODEL_AVAILABLE=none"
fi
```

If `ARS_CROSS_MODEL` is set but the Anthropic key is missing or the model name is
unsupported, warn the user and proceed with single-runtime verification.

## Cost Considerations

Cross-model verification adds Anthropic API calls:

| Scenario | Additional Calls | Cost Note |
|----------|------------------|-----------|
| Integrity verification (30% of 60 refs, batched 5/call) | About 4 calls | Depends on Anthropic account pricing and prompt size |
| Devil's advocate cross-check (1 per checkpoint, 3 checkpoints) | 3 calls | Depends on Anthropic account pricing and prompt size |
| Calibration mode | At least 1 external run per gold paper set | Depends on number and length of papers |

Always tell the user when cross-model verification will make external API calls.

## Limitations

1. **Does not solve frame-lock fully.** Cross-model review catches different
   surface errors, but models can still share source-corpus gaps.
2. **API latency.** External calls add latency. Batched reference checks are
   usually faster than one call per reference.
3. **Response format differences.** Claude Opus 4.7 may structure responses
   differently from the primary Codex output. Keep prompts simple and structured.
4. **Cost scales with paper size.** Longer papers with more references require
   more external calls.

## Graceful Degradation

If cross-model verification fails because of API error, rate limit, missing key,
or unsupported model:

- Log the failure: `[CROSS-MODEL-ERROR: reason]`.
- Continue with single-runtime verification. Never block the pipeline on
  cross-model failure.
- Include this report note: "Cross-model verification was configured for Claude
  Opus 4.7 but unavailable for this run. Results are single-runtime only."
