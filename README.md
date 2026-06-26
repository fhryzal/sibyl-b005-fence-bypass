# Sibyl Memory Injection Fence Bypass

Reproducer for prompt injection vulnerabilities in sibyl-memory fence implementation (v0.4.15).

## Findings

The fence scrubber has three bypasses:

**Unicode whitespace**  
Regex looks for literal ASCII space in `[UNTRUSTED MEMORY CONTEXT ...]`. Zero-width space (U+200B), tabs, and non-breaking spaces break the pattern but render identically. Forged markers pass through.

**Dict key gap**  
Scrubber walks dict values recursively but keys stay untouched. Put the marker in a key and it survives.

**Nested structure**  
Store a body with `_untrusted_context` field. Output gets two control blocks—one real at root, one fake nested in the body. Model sees conflicting fence markers.

## Running

```bash
pip install sibyl-memory-client==0.4.15 mcp
git clone https://github.com/Sibyl-Labs/Sibyl-Memory
python3 sibyl_fence_bypass_poc.py
```

## Fix

Strip reserved keys before write:

```python
_RESERVED = {"_untrusted_context", "ok", "error"}

def strip_reserved(v):
    if isinstance(v, dict):
        return {k: strip_reserved(val) for k, val in v.items() if k not in _RESERVED}
    if isinstance(v, list):
        return [strip_reserved(x) for x in v]
    return v
```

Apply in `set_entity` before insert. Read-side scrubbing alone isn't enough.
