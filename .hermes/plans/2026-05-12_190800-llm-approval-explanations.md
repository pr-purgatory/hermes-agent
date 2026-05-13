# LLM-Generated Approval Explanations for Dangerous Commands

## Goal

Replace the static, regex-derived pattern descriptions (e.g., *"delete in root path"*) shown to users when a dangerous command is flagged with natural-language, LLM-generated explanations that explain the actual risk in plain English.

## Current Behavior

When a terminal command matches a dangerous pattern in `tools/approval.py`, the user sees:

> ⚠️ This command is potentially dangerous (delete in root path).
> **Command:**
> ```
> rm -rf /tmp/project/
> ```

The description comes from `DANGEROUS_PATTERNS` — a list of regex pairs mapping to static strings. These are pattern-class labels, not explanations. Users get no context about what the command actually does or why it's risky.

## Proposed Approach

Add a new function `_llm_explain_command(command, description)` that calls the auxiliary LLM (via `agent.auxiliary_client.call_llm`) to generate a concise, actionable explanation. Pipe this into the gateway's approval message flow before the message is sent to the user.

Key design decisions:
- **Use the auxiliary LLM** — not the main model, to avoid consuming the main token budget
- **Async where possible** — the gateway already runs in an executor; the LLM call adds ~1-2s latency
- **Graceful fallback** — if the LLM call fails or times out, fall back to the existing static description
- **Configurable** — a new config flag `approvals.llm_explain: true` (default `true`) so users can disable it

## Step-by-Step Plan

### 1. Add `_llm_explain_command()` in `tools/approval.py`

Create a new function after `_smart_approve()`:

```python
def _llm_explain_command(command: str, description: str, timeout: int = 8) -> str:
    """Generate a natural-language explanation of why a command is dangerous.
    
    Returns the explanation string, or the static description on failure.
    """
```

- Uses `agent.auxiliary_client.call_llm` with `task="approval"`
- Prompt: short system prompt + the command and flagged reason
- Max tokens ~150, temperature 0.2
- Timeout via `timeout` parameter (default 8s, configurable)
- Catches all exceptions → returns the original `description` as fallback

### 2. Wire into `check_all_command_guards()` and `check_dangerous_command()` in `tools/approval.py`

Modify the approval dict construction so that:
- When `approvals.llm_explain` config is `true`, call `_llm_explain_command()` before building the `message` field
- Replace `description` with the LLM explanation in the message text
- Keep the original `description` in the `pattern_key` field for allowlist matching

### 3. Add config option `approvals.llm_explain`

In `_get_approval_config()` or wherever config is read, add:

```yaml
approvals:
  llm_explain: true     # default: true
  llm_explain_timeout: 8  # default: 8 (seconds)
```

### 4. Update gateway approval notification in `gateway/run.py`

The gateway sends the approval dict as a message to the user via the platform adapter. No structural changes needed — the `message` field in the approval dict already supports multi-line text. The LLM explanation just makes it longer and more informative.

### 5. Add tests

- Unit test `_llm_explain_command()` with a mocked `call_llm` returning a plausible explanation
- Unit test fallback on `call_llm` failure (timeout, API error)
- Integration test that the approval message in `check_dangerous_command()` contains the LLM explanation when the config flag is on

Existing tests in `tests/` that cover `tools/approval.py` should be updated to account for the new config field.

## Files Likely to Change

| File | Change |
|------|--------|
| `tools/approval.py` | Add `_llm_explain_command()`, integrate into guard functions, add config reading |
| `gateway/run.py` | Wire `llm_explain_timeout` display setting into the approval flow startup |
| `cli-config.yaml.example` | Document new `approvals.llm_explain` option |
| `tests/test_approval.py` or similar | New tests for LLM explanation generation |

## Risks & Tradeoffs

| Risk | Mitigation |
|------|-----------|
| **Latency** — LLM call adds 1-3s before the approval prompt appears | Use auxiliary model (fast, cheap); 8s timeout with fallback |
| **LLM hallucination** — explanation is misleading or wrong | Short, constrained prompt; temperature 0.2; max 150 tokens |
| **Token cost** — every flagged command costs auxiliary credits | Auxiliary model is the smallest/fastest available; cost is negligible |
| **Config bloat** — new `approvals` fields add complexity | Default to `true` so it just works; users who care can tweak |
| **Edge case: very long commands** — full command in prompt could be huge | Truncate command to 500 chars in the LLM prompt |

## Open Questions

1. **Should this be opt-out or opt-in?** Defaulting to `true` means better UX for most users, but adds latency.
2. **Should we cache explanations?** If the same pattern+command combo is flagged twice (e.g., in a loop), skip the LLM and reuse the cached explanation.
3. **Language support?** The LLM prompt assumes English. Should we pass `agent.i18n` locale for multilingual explanations?
4. **Should tirith security scan findings also get LLM explanations?** Currently `_format_tirith_description()` just concatenates findings. LLM could summarize those too, but that's a separate feature.

## Verification Steps

1. Set `approvals.llm_explain: true` in `~/.hermes/config.yaml`
2. Run a command that triggers dangerous detection (e.g., `rm -rf /tmp/something/`)
3. Confirm the approval prompt shows a natural-language explanation instead of the static pattern string
4. Simulate auxiliary LLM failure (set `approvals.llm_explain_timeout: 1`) → confirm fallback works
5. Run `pytest tests/` for approval-related tests
