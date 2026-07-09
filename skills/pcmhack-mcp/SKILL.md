---
id: pcmhack-mcp
name: pcmhack-mcp
description: Use when writing or running code through the PcmHackMCP `run_python` tool (or POSTing to the Ghidra plugin's :8765/run_python endpoint) - i.e. executing an arbitrary Jython script server-side inside a live Ghidra session via MCP. Covers the whole-script execution model, print/JSON output discipline, the automatic (commit-on-error) transaction, and how this differs from the ghidra_bridge client. Do NOT trigger for the ghidra_bridge/jfx_bridge RPC client (that has its own skill) or plain Ghidra GUI/headless questions.
disable-model-invocation: true
---

# PcmHackMCP run_python - the things that bite

`run_python` ships a **whole Jython 2.7 script** to a live Ghidra session and runs it as a GhidraScript in **one call**, returning only what the script prints. Same interpreter and same Ghidra API as `ghidra_bridge`, but the transport and a few rules are different.

If you forget everything else:

1. **It's Jython 2.7 + the Ghidra Flat API.** Every Jython/API gotcha from the `ghidra-bridge` skill applies to the script you send. The critical three are inlined below; the full trap shelf + API method-name table live in that skill (Nathan: `~/.claude/skills/ghidra-bridge/SKILL.md`) - read it for anything non-trivial.
2. **Send one complete script; `print` what you want back.** No return value crosses the wire - only stdout/stderr. Emit results with `print(json.dumps(...))`.
3. **The transaction is automatic - and commits even on error.** Don't open one. But guard each row (bare `except:`), because work done before an uncaught exception is *saved*, not rolled back.
4. **Imports work directly.** `from ghidra.program.model.symbol import SourceType` just works - you are already server-side. There is no `remote_eval`, `remote_exec`, or `remote_import` here.

Repo: https://github.com/hjtrbo/PcmHackMCP

## The dedicated tools can overflow - page, filter, or summarize

`run_python` is the bulk workhorse, but the same MCP server also exposes dedicated listing
tools (`list_functions`, `list_data_items`, `list_strings`, `list_segments`, ...). The trap:
**a whole-program listing can return thousands of rows and blow the single tool-response
limit.** A typical automotive ECU image has many thousands of functions and tens of thousands
of data items, so an unfiltered `list_functions` or `list_data_items` will overflow in one shot.

When you need a slice, do not dump the whole list:
- Use the tool's own paging/filter args, or `search_functions_by_name` for a targeted lookup.
- For counts, scans, or aggregates over the whole program, push the work into `run_python`
  and `print` only the count plus a bounded sample - never round-trip thousands of rows just
  to filter them client-side.

Same discipline as "bound the volume" below: shape the result on the server, return the answer.

## Multiple Ghidra instances

You can run two Ghidra sessions simultaneously and talk to each from one opencode session.
Each instance runs its own bridge process, with the second Ghidra's pcmhack plugin
configured to listen on a different port (e.g. 8766 instead of 8765).

Tool names are prefixed by the server name, so there is no collision:

```
pcmhack_run_python          # primary instance (:8765)
pcmhack_alt_run_python       # alternate instance (:8766)
pcmhack_list_functions       # primary instance
pcmhack_alt_list_functions   # alternate instance
```

`currentProgram` refers to the program loaded in the Ghidra session targeted by the chosen
MCP tool. The script runs inside that session's interpreter, so `currentProgram` is always
the correct one for that port.

## Invoke

Via the MCP tool (preferred):
```
pcmhack_run_python(code="...", timeout=600)
```
Or, for the alternate instance:
```
pcmhack_alt_run_python(code="...", timeout=600)
```
Or raw HTTP (PowerShell) - primary on port 8765:
```powershell
$code = @'
print(currentProgram.getName())
'@
Invoke-RestMethod -Uri http://127.0.0.1:8765/run_python -Method Post -Body $code
```
Or raw HTTP - alternate on port 8766:
```powershell
Invoke-RestMethod -Uri http://127.0.0.1:8766/run_python -Method Post -Body $code
```

`currentProgram`, `currentAddress`, `monitor`, and the Flat API helpers (`toAddr`, `getFunctionAt`, `getInstructionAt`, `createLabel`, ...) are pre-injected as GhidraScript globals. No connect step, no `namespace=globals()`.

## Output: print it, shape it, bound it

The response body is exactly what the script wrote to stdout/stderr. So:

- Return data by printing JSON, then parse it client-side:
  ```python
  import json
  fm = currentProgram.getFunctionManager()
  def collect():
      out = []
      for f in fm.getFunctions(True):
          if f.getName().startswith("FUN_"):
              out.append({"name": f.getName(), "ep": str(f.getEntryPoint())})
      return out
  print(json.dumps(collect()))
  ```
- **Bound the volume.** Do not print 50k rows - summarize, page (`rows[offset:offset+N]`), or print a count plus a sample. Oversized responses are slow and can blow the tool result limit.
- A script that prints nothing returns `(script finished with no output)` - that is success, not an error.

## Transactions: automatic, no rollback

`execute()` wraps your script in `start()` / `run()` / `end(commit=true)`, and the commit happens in a `finally`, so:

- You do **not** open a transaction for mutations. Just mutate.
- An uncaught exception still **commits** whatever ran before it. There is no auto-rollback.
- For batch mutation: use a **per-row `try` with a bare `except:`** (Java exceptions do not subclass `Exception`) so one bad row does not abort the rest, and collect failures to print at the end.
- If you need all-or-nothing, **validate everything first, then mutate** - you cannot rely on rollback.

## The three Jython 2.7 traps you will hit (full list in the ghidra-bridge skill)

1. **No f-strings.** `f"{x}"` is a Jython 2.7 parse error. Use `"{}".format(x)` or `"%x" % x`.
2. **Comprehension loop variables leak** and can crash on serialize. Wrap any non-trivial comprehension in a `def`.
3. **`except Exception:` does not catch Java exceptions** (`InvalidInputException`, `CodeUnitInsertionException`, ...). Use bare `except:` + `sys.exc_info()[1]`.

**Reading raw bytes:** do **not** use `mem.getBytes(addr, bytearray(n))` - a Python `bytearray` reads back all zeros (silent, no exception, looks like blank/erased memory). Use the returning accessors `mem.getInt/getShort/getByte/getLong(addr)`, or the flat-API `getBytes(addr, n)` which returns a fresh `byte[]`; if you must fill a buffer use `jarray.zeros(n, 'b')`. Full note in the ghidra-bridge trap shelf.

The Ghidra-API method-name table (e.g. `getCallingConventions`, `AddressFactory.getAddress`, `getNumComponents`, `getComment(type)`, symbol-name sanitization) is in the ghidra-bridge skill and applies unchanged.

## Worked pattern: apply N labels, per-row guarded, JSON result

No manual transaction (it is automatic); errors captured per row; result printed as JSON.

```python
import json, sys
from ghidra.program.model.symbol import SourceType

ROWS = [{"id": 1, "address": "0x3f83c8", "label": "SPK_BDL_CLP"}]  # bake big inputs into the script
af = currentProgram.getAddressFactory()
symtab = currentProgram.getSymbolTable()
applied, errors = 0, []
for r in ROWS:
    try:
        addr = af.getAddress("%x" % int(r["address"], 16))
        sym = symtab.createLabel(addr, r["label"], SourceType.USER_DEFINED)
        if not sym.isPrimary():
            sym.setPrimary()
        applied += 1
    except:
        errors.append([r["id"], r["label"], str(sys.exc_info()[1])])
print(json.dumps({"applied": applied, "errors": errors}))
```

## Worked pattern: scan/collect in one call

```python
import json
rm = currentProgram.getReferenceManager()
af = currentProgram.getAddressFactory()
def collect(targets):
    out = {}
    for name, hexs in targets.items():
        a = af.getAddress("%x" % int(hexs, 16))
        out[name] = {"addr": str(a),
                     "xrefs": [str(x.getFromAddress()) for x in rm.getReferencesTo(a)]}
    return out
print(json.dumps(collect({"SPK_BDL_CLP": "0x3f83c8"})))
```

For big inputs, bake them into the script text (it is one payload anyway) rather than threading them through a separate channel.

## Reading errors

On failure the response is the **plain-text Jython traceback** (class, message, stack) - there is no `BridgeException` wrapper to unwrap. The last line is usually the real cause. If the script is silent or wrong, comment the body down to `print("alive")`, confirm it runs, then add lines back.

## Timing

- Long sweeps: pass a bigger `timeout` (the tool defaults to 600s).
- The plugin's HTTP server is single-threaded, so a long `run_python` blocks other PcmHackMCP tool calls until it returns. Prefer one well-shaped call over many.

## Checklist before sending a non-trivial script

1. No f-strings in the script body - use `.format()` / `%`.
2. Non-trivial comprehensions are inside a `def`.
3. Per-row `try` uses bare `except:` + `sys.exc_info()[1]`.
4. Results are `print`ed, ideally `json.dumps`, and bounded in size.
5. No manual transaction (and no reliance on rollback - validate first if it must be atomic).
6. Imports are plain `from ghidra... import ...` (no remote_import).
7. Long jobs pass an explicit `timeout`.
