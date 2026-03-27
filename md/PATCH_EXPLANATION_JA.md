# ComfyUI-DistorchMemoryManager Patch Explanation (Repository-Accurate Version)

## 1. Why this fix was needed

In production usage, the following message was repeatedly emitted and became major log noise:

- `Error running sage attention: Unsupported head_dim: 160, using pytorch attention instead.`

This is not a fatal stop condition. It is a known recoverable path where execution continues via PyTorch attention fallback.  
The real operational issue was that the same recoverable case was logged as `error` over and over.

---

## 2. Root cause

When SageAttention is called with unsupported `head_dim=160` combinations, it can raise an exception.  
The implementation already has a fallback path, so generation can continue.

The problem was the logging policy: **a known recoverable fallback was recorded as `error` every time**.

---

## 3. Why this was fixed in the custom node

Direct patches to ComfyUI core files are overwritten by updates and are hard to maintain.  
So this project uses the following policy:

- No dependency on direct core-file edits
- Apply runtime patching when the custom node is loaded
- Keep behavior stable across ComfyUI updates by reapplying from the custom node side

In short, this is an **external runtime patch strategy optimized for long-term maintainability**.

---

## 4. Files changed and what they mean

## 4.1 `ComfyUI/custom_nodes/ComfyUI-DistorchMemoryManager/nodes/sa.py`

### Change

The exception handling in `attention_sage()` was adjusted so only `Unsupported head_dim: 160` is suppressed.

- Known exception (`Unsupported head_dim: 160`) logs once as `info`
- Other exceptions remain `error` (unchanged behavior)
- Fallback target stays `attention_pytorch` (unchanged behavior)

### Meaning

This suppresses only known noise while preserving visibility of unknown failures.

## 4.2 `ComfyUI/custom_nodes/ComfyUI-DistorchMemoryManager/__init__.py`

### Change (core of this patch)

At startup, `_install_sage_attention_noise_guard()` runs and replaces ComfyUI's `attention_sage` externally via runtime patching.

Implementation highlights:

- Imports `comfy.ldm.modules.attention`
- Gets the current `attention_sage`
- Defines wrapped `attention_sage_guarded` with `@wrap_attn` and swaps it in
- Also swaps these references when needed:
  - `optimized_attention`
  - `optimized_attention_masked`
  - `REGISTERED_ATTENTION_FUNCTIONS["sage"]`
- Prevents double patching via `_dm_sage160_guard`
- Limits suppression strictly to `"Unsupported head_dim: 160"`

### Meaning

Behavior is corrected at runtime without editing core files directly, making it resilient to ComfyUI updates.

---

## 5. Scope and safety

This suppression is narrowly scoped, not blanket suppression.

- Match target: `"Unsupported head_dim: 160"`
- Known case: first occurrence logged as `info`
- All other exceptions: still logged as `error`

As a result:

- Reduces known SD1.5-style log spam
- Preserves observability for other models and unknown errors

---

## 6. Verification checklist

1. After restarting ComfyUI, confirm this startup log:  
   - `[ComfyUI-DistorchMemoryManager] Installed external sage-attention head_dim=160 noise guard`
2. During SD1.5 runs, repeated `error` logs for `Unsupported head_dim: 160` no longer appear
3. Fallback still works and generation continues
4. SA/FA behavior for other models (e.g., SDXL) remains functional
5. Non-known SageAttention exceptions are still visible as `error`

---

## 7. Future improvements (developer ideas)

- Make suppression targets configurable (e.g., list of unsupported head dimensions)
- Add UI toggle to enable/disable suppression
- Explicitly document runtime patch policy in README/CHANGELOG
- Improve startup logs to show patch apply status in more detail

---

## 8. Summary

This patch suppresses only the known log spam caused by the `head_dim=160` fallback path, while keeping unknown error visibility intact.  
By implementing it as a custom-node runtime patch instead of direct core modification, it improves long-term maintainability.

