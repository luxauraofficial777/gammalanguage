# gammalanguage
Gamma Programming Language for Agentic LLMs optimization and MCP Vector Analysis
Created by Lux Aura https://facebook.com/LuxAuraOfficial https://luxaura.bandcamp.com https://youtube.com/LuxAuraOfficial https://github.com/luxauraofficial777

# Gamma Language (Gamma-DSL v1.0)
## Complete Standalone Package & Production Toolkit

**Gamma Language** is a token-optimal native domain-specific language engineered for AI Agents, MCP Servers, Vector Databases, and LLM cross-modulation.

---

## Directory Structure

* **`spec/`**: Formal EBNF grammar, token dictionary, sigil definitions, and architectural specification ([`gamma_dsl_spec.md`](spec/gamma_dsl_spec.md)).
* **`cpp/`**: Zero-allocation, cache-line aligned C++ AST parser, SIMD vectorization engine, and unit tests (`A2AST.h`, `A2ALexer.*`, `A2AEvaluator.*`, `test_a2a_parser.cpp`, `CMakeLists.txt`).
* **`python/`**: Python CLI bridge, Caveman compression integration, and TurboQuant/Nexus socket adapters (`gamma_bridge.py`, `gamma_cli.py`).
* **`examples/`**: Ready-to-run `.gamma` script examples (`sample_crossmod.gamma`, `sample_vector_query.gamma`).
* **`install.ps1` / `install.bat`**: One-click installer scripts compiling native binaries and installing CLI tools.

---

## One-Click Installation

### Windows PowerShell
```powershell
.\install.ps1
```

### Windows Command Prompt (cmd)
```cmd
install.bat
```

---

## Quick Command Verification

After installation, verify the Gamma Language CLI:

```bash
python python/gamma_cli.py --test
```

Or run the C++ zero-heap SIMD parser:

```bash
.\cpp\bin\gamma_parser.exe
```
