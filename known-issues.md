# Known Issues

This document tracks unresolved issues that are known to cause failures or unexpected behavior in the gemini-cli workspace template. Entries are organized by severity and include workaround instructions where available.

---

## Issue 1: Missing `GEMINI_API_KEY` causes silent startup failure

**Severity:** High

**Description:**

If `GEMINI_API_KEY` is unset when the container starts, gemini-cli initializes without an API key but does not exit immediately. The agent starts, accepts sessions, and then produces no response for every prompt — the platform sees an agent that "never replies."

The underlying cause is that gemini-cli's auth layer attempts to load the key lazily on the first API call, not at startup. No error is raised until the first `stream_response()` call, which then fails with a generic timeout or an auth error that may be swallowed by the platform shim.

**Affected versions:** All template versions prior to the env-validation shim in `adapter.py` (see workaround).

**Workaround:**

Validate the env var before invoking the adapter:

```bash
if [ -z "$GEMINI_API_KEY" ]; then
  echo "ERROR: GEMINI_API_KEY is not set" >&2
  exit 1
fi
```

Or add an early check in `adapter.py`:

```python
import os
def connect(platform_url, workspace_id, timeout=30):
    if not os.environ.get("GEMINI_API_KEY"):
        raise RuntimeError("GEMINI_API_KEY environment variable is not set")
    ...
```

**Tracking:** Internal issue `WKS-001`.

---

## Issue 2: `system-prompt.md` injected after gemini-cli defaults, overriding template's SOUL.md conventions

**Severity:** Medium

**Description:**

gemini-cli loads system prompts in the following order:

1. Built-in defaults (`gemini-cli/resources/defaults/system.txt`)
2. `SOUL.md` in the current working directory (gemini-cli's convention for agent personality files)
3. `system-prompt.md` injected by the workspace template

Because `system-prompt.md` is concatenated last, it overwrites any setting that was already set in `SOUL.md` (or gemini-cli's defaults). This means template authors cannot rely on gemini-cli's `SOUL.md` convention to set agent personality, guardrails, or tool restrictions — anything set there is silently clobbered.

This is particularly problematic for deployments that rely on gemini-cli's default tool list (which includes shell execution, file read/write, and internet access) since the template's `system-prompt.md` must explicitly deny those tools to enforce a tighter scope.

**Workaround:**

Do not use `SOUL.md` for runtime configuration. Put all system-prompt content exclusively in `system-prompt.md` and leave `SOUL.md` absent or empty. The adapter startup script should delete or truncate `SOUL.md` if present:

```bash
# In Dockerfile, after COPY:
RUN rm -f /workspace/SOUL.md
```

**Tracking:** Internal issue `WKS-002`.

---

## Issue 3: `config.yaml` model override not propagated to the gemini-cli config file inside Docker

**Severity:** Medium

**Description:**

The template's `config.yaml` exposes `runtime.model` as the canonical model selection knob. However, gemini-cli reads its own config file (`~/.config/gemini-cli/config.json`) for model selection, not `config.yaml`. The template's `config.yaml` is read by the adapter shim only; it does not rewrite gemini-cli's config file.

As a result, even if `config.yaml` specifies `model: gemini-2.5-pro`, the container may still run the model configured in gemini-cli's internal config (defaulting to `gemini-2.0-flash`).

**Reproduction:**

```bash
# Set a non-default model in config.yaml
sed -i 's/^  model:.*/  model: gemini-2.5-pro/' config.yaml
docker build -t molecule-gemini-cli:dev .
docker run --rm molecule-gemini-cli:dev \
  python -c "from adapter import stream_response; print(stream_response('s', 'what model are you'))"
# Output: "gemini-2.0-flash"  (not gemini-2.5-pro)
```

**Workaround:**

The adapter must sync the model value into gemini-cli's config file before starting the session:

```python
import json, os, pathlib

def sync_model_to_gemini_config(model: str):
    config_path = pathlib.Path(os.path.expanduser("~/.config/gemini-cli/config.json"))
    config_path.parent.mkdir(parents=True, exist_ok=True)
    if config_path.exists():
        cfg = json.loads(config_path.read_text())
    else:
        cfg = {}
    cfg["model"] = model
    config_path.write_text(json.dumps(cfg, indent=2))
```

Call `sync_model_to_gemini_config()` inside `connect()` before instantiating the gemini-cli client.

**Tracking:** Internal issue `WKS-003`.

---

## Issue 4: Template schema version 1 but platform v2 introduces breaking config key renames

**Severity:** High (breaking for platform v2 deployments)

**Description:**

The template ships with `schema_version: "1"` in `config.yaml`. Platform version 2 (v2) renamed several top-level keys:

| v1 key                    | v2 key                           |
|---------------------------|----------------------------------|
| `runtime.agent`           | `agent.runtime`                  |
| `runtime.model`           | `agent.model`                    |
| `runtime.api_key_env`     | `auth.gemini_api_key_env`        |
| `adapter.platform_url`    | `platform.endpoint`             |
| `adapter.workspace_id_env`| `platform.workspace_id_env`      |
| `adapter.timeout_seconds` | `platform.request_timeout_secs`  |

Templates using v1 syntax on a v2 platform silently ignore renamed keys — the adapter gets default values instead of configured ones, leading to runtime failures that are difficult to diagnose.

**Detection:**

```bash
# Check which schema version the platform expects
curl -s https://platform.molecule.ai/api/schema-version | jq .
```

If the platform returns `2` and `config.yaml` has `schema_version: "1"`, the config is incompatible.

**Workaround:**

Maintain separate `config.v1.yaml` and `config.v2.yaml` files and select the correct one at container startup based on the platform's reported schema version:

```bash
# In Dockerfile CMD or entrypoint script:
PLATFORM_SCHEMA=$(curl -s https://platform.molecule.ai/api/schema-version | jq -r '.version')
if [ "$PLATFORM_SCHEMA" = "2" ]; then
  cp /workspace/config.v2.yaml /workspace/config.yaml
else
  cp /workspace/config.v1.yaml /workspace/config.yaml
fi
exec python -m adapter
```

**Tracking:** Internal issue `WKS-004`. Fixed in template v2.0 (pending release).
