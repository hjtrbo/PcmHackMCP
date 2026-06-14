[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://www.apache.org/licenses/LICENSE-2.0)

# PcmHackMCP

PcmHackMCP is a fork of [LaurieWired/GhidraMCP](https://github.com/LaurieWired/GhidraMCP) (Apache-2.0): a Model Context Protocol (MCP) server plus Ghidra plugin that lets MCP clients (Claude, etc.) drive Ghidra for reverse engineering.

It keeps all of the upstream tools and adds one thing: a **server-side `run_python` tool** that executes an arbitrary Jython script inside Ghidra in a single call. Bulk work - mass renames, xref sweeps, batch comments, applying data types over many addresses - runs as one loop inside Ghidra instead of thousands of individual MCP/HTTP round-trips. Only the printed result crosses the wire.

To run alongside the original GhidraMCP without clashing, every identifier is renamed and the default port is changed:

| | Upstream GhidraMCP | PcmHackMCP |
|---|---|---|
| MCP server name | `ghidra-mcp` | `pcmhack-mcp` |
| Ghidra module / extension | `GhidraMCP` | `PcmHackMCP` |
| Java class | `com.lauriewired.GhidraMCPPlugin` | `com.pcmhack.mcp.PcmHackMCPPlugin` |
| Default HTTP port | 8080 | 8765 |
| MCP bridge script | `bridge_mcp_ghidra.py` | `bridge_mcp_pcmhack.py` |

## Features

- Everything in upstream GhidraMCP (decompile, list/rename functions and data, imports/exports, xrefs, strings, set prototypes, and more).
- `run_python(code, timeout=600)` - run an arbitrary Jython script server-side against the current program. Full GhidraScript environment is available (`currentProgram`, the flat API, `monitor`); a program transaction is opened and committed automatically; both `print(...)` and `println(...)` output is captured and returned.

## Install

### Prerequisites
- Ghidra 11.3.2
- Python 3.10+ and the MCP SDK: `pip install "mcp>=1.2.0,<2" "requests>=2,<3"`

### Ghidra plugin
1. Run Ghidra
2. `File` -> `Install Extensions`
3. Click `+` and select `PcmHackMCP-11.3.2.zip`
4. Restart Ghidra
5. Enable **PcmHackMCP** in `File` -> `Configure` -> `Developer`
6. Optional: change the port in `Edit` -> `Tool Options` -> `PcmHackMCP HTTP Server` (default 8765)

### MCP client (Claude Desktop example)
`Claude` -> `Settings` -> `Developer` -> `Edit Config`, then:
```json
{
  "mcpServers": {
    "pcmhack": {
      "command": "py",
      "args": [
        "-3",
        "C:\\ABSOLUTE_PATH_TO\\bridge_mcp_pcmhack.py",
        "--ghidra-server",
        "http://127.0.0.1:8765/"
      ]
    }
  }
}
```
Host/port default to `127.0.0.1:8765` if not set.

### Claude Code
```
claude mcp add pcmhack -- py -3 "C:\ABSOLUTE_PATH_TO\bridge_mcp_pcmhack.py"
```

## run_python example

```
curl -s -X POST http://127.0.0.1:8765/run_python --data-binary @- <<'PY'
count = 0
for f in currentProgram.getFunctionManager().getFunctions(True):
    if f.getName().startswith("FUN_"):
        count += 1
print("auto-named functions:", count)
PY
```

The script runs inside Ghidra and returns its printed output.

> Security: the embedded HTTP server binds to all interfaces and `run_python` executes arbitrary code in your Ghidra session. Run it only on a trusted network.

## Build from source

No Maven required (it is a single source file). With JDK 17-21:

1. Copy these jars from your Ghidra install into `lib/`:
   - `Base.jar`, `Decompiler.jar`, `Docking.jar`, `Generic.jar`, `Project.jar`, `SoftwareModeling.jar`, `Utility.jar`, `Gui.jar`
2. Compile and package:
   ```
   javac --release 17 -cp "lib/*" -d build/classes src/main/java/com/pcmhack/mcp/PcmHackMCPPlugin.java
   jar --create --file target/PcmHackMCP.jar --manifest src/main/resources/META-INF/MANIFEST.MF -C build/classes .
   ```
3. Zip an extension folder `PcmHackMCP/` containing `extension.properties`, `Module.manifest`, and `lib/PcmHackMCP.jar`.

Or, if you have Maven installed: `mvn clean package assembly:single`.

## Credits

Fork of [GhidraMCP](https://github.com/LaurieWired/GhidraMCP) by LaurieWired. Licensed under Apache-2.0; see [LICENSE](LICENSE).
