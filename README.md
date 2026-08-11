# gammalanguage

<img width="1024" height="1024" alt="image" src="https://github.com/user-attachments/assets/511ffa8a-b3d9-480a-94d9-0acba10de17b" />

Gamma Programming Language for Agentic LLMs optimization and MCP Vector Analysis
Created by Lux Aura with code review by KVOID (Barbelith) https://facebook.com/LuxAuraOfficial https://luxaura.bandcamp.com https://youtube.com/LuxAuraOfficial https://github.com/luxauraofficial777 https://kvoid.net 

# Liminal A2A-DSL (Agent-to-Agent Domain Specific Language)
## v1.11B — Deterministic Agent VM with Hardware Interop & Bytecode Target

**Authors:** Lux Aura  
**Target Runtimes:** VoidWalkers HD Engine, Modern_X64, VW Nexus Server (`:8651`), DuckStation/PSX emulators, custom bare-metal micro-runtimes  
**Date:** August 2026  

> **Implementation Status:** The C++ parser, AST arena, and bytecode VM are functional. Some features listed below (Vulkan GPU offload, Caveman compression in C++, SIMD intrinsics) are stubs or software fallbacks — see individual sections for details.  

---

## 1. What This Language Is — and Is Not

**Liminal A2A-DSL is a deterministic, sandboxed virtual machine for agent programs.**

Its value is not token compression. A2A-DSL costs **~20% more** tokens than compact JSON-RPC for the same information (measured: `cl100k_base` 511→615, `o200k_base` 524→620; reproduced by `tests/benchmark_token_reduction.py`). Its value is:

- **Determinism:** A program is a typed, replayable instruction stream. The same program on the same inputs produces the same register state, the same host requests, and the same faults — every time.
- **Validation:** The parser rejects unrecognized input with line/column diagnostics. Register indices are bounds-checked against bank sizes. Unknown opcodes fault instead of silently succeeding.
- **Host-mediated effects:** The VM itself is side-effect-free. External work (tool calls, vector queries, research delegation, native compilation) is recorded in a `HostRequestQueue` that the host drains and performs. The host decides what is safe to execute.
- **A bytecode target:** Programs compile to 32-bit packed A2A-B instructions, enabling a Direct Threaded Code execution path with no string lexing on the hot path.

**Flagship reference application: hardware/emulator-interop scripting.** This repository is a PSX/DuckStation reverse-engineering project. `CP0_REG_MAP`, `PSX_DMA_DISPATCH`, `%HW_INT`, and `#HIVE` research delegation are the real center of gravity — the language pointed at this repo's actual domain.

### Why not JSON-RPC?

JSON-RPC is the right default for agent-to-agent communication. It is universally supported, token-cheap (BPE tokenizers merge its structure into single tokens), and requires no grammar priming. A2A-DSL does not compete with JSON-RPC on token cost — it costs more. What it offers that JSON-RPC cannot is a **typed, replayable, statically-checkable instruction stream** with explicit control flow (`MATCH`), fault taxonomy (`PASSED`/`FAILED`/`UNRESOLVED`), speculative execution (`@PREDICT`/`SPEC_COMMIT`/`SPEC_ROLLBACK`), and a bytecode target. If you do not need determinism, validation, or a compilation target, use JSON-RPC.

### Liminal Lore Toolchain Integration

The language incorporates four toolchain layers. Each is either implemented, a stub, or a host-recorded no-op — see the individual sections and `GAMMA_FINALIZATION_PLAN.md` for honest status:

1. **Caveman Layer:** Strips conversational filler and structural noise. *(C++ implementation is a passthrough stub that copies payload verbatim and records a host request; Python bridge implements a fixed regex filler-strip.)*
2. **Colibri Layer:** MoE expert pinning/unpinning/swap and heartbeat. *(Records host requests; the VM sets an expert index.)*
3. **TurboQuant Layer:** KV-cache quantization guard. *(A single float compared against a threshold in `@INVARIANT` blocks.)*
4. **Gigatoken Layer:** Large-context streaming and chunk-packing. *(Records a host request; no packing logic in the VM.)*

### v1.1A-Robust+ Additions

Building on v1.0's 8-sigil architecture, v1.1A-Robust introduces 7 new feature families and v1.1A-Robust+ adds 5 peak performance additions:

- **Dynamic Target Environment Definitions** (`|TARGET_DEFINE`)
- **Multi-Agent Synchronization** (`=LOCK=`, `=SYNC=`)
- **Partitioned Data Hive Targeting** (`#HIVE`)
- **Deterministic Error Recovery** (`@ON_ERROR`, `RETRY`, `SHIFT_TARGET`)
- **Bidirectional Native Code Interop** (`<NATIVE_BRIDGE>`)
- **Skill Hot-Swapping** (`+SKILL`, `-SKILL`)
- **ZHARK/FUBBU Research Delegation** (`?ZHARK`, `¿FUBBU_SYNC`)
- **A2A-B Binary Bytecode** — 32-bit packed instructions for Direct Threaded Code execution
- **Quantized Vector Delta Registers** (`#QVEC`, `#VEC_DELTA`) — 50% bandwidth reduction (FP16 → 8-bit)
- **Speculative Pipeline Branching** (`@PREDICT`, `SPEC_COMMIT`, `SPEC_ROLLBACK`)
- **Hardware Interrupt Frame-Sync** (`%HW_INT`) — VBLANK/DMA/AUDIO_SWAP mapping
- **Sliding-Window Context Eviction** (`^EVICT_SLIDING`) — automatic ring-buffer pruning

---

## 2. Sigil & Symbol Dictionary

| Sigil | Primitive Name | Description & Usage |
| :---: | :--- | :--- |
| `!` | **State Shift** | Asserts an explicit state transition (`!STATE: 0x01 [NODE_INIT]`) |
| `@` | **Invariant Guard** | Hard precondition (`@INVARIANT`), error recovery (`@ON_ERROR`), speculative branch (`@PREDICT`) |
| `$` | **Execution Block** | Scoped logic container evaluating registers, vectors, and signals |
| `\|` | **Target Context** | Sets platform rules (`\|TARGET: [D3DX11]`) or defines custom specs (`\|TARGET_DEFINE`) |
| `+` / `-` | **Skill Swapper** | Equips (`+SKILL`) or unloads (`-SKILL`) agent capabilities dynamically |
| `*` | **Profile Swap** | Atomically swaps the entire active skill array (`*PROFILE`) |
| `#` | **Vector / Hive / QVec** | Embeddings (`#VEC[0]`), hives (`#HIVE["name"]`), quantized vectors (`#QVEC[0]`), deltas (`#VEC_DELTA`) |
| `%` | **Signal / HW Interrupt** | Cross-modulation (`%SIG[0]`), hardware frame-sync (`%HW_INT(VBLANK)`) |
| `~` | **MCP Transceiver Pipe** | Direct high-throughput tool call binding (`~MCP("vwforge_build", REG[0])`) |
| `&` | **Colibri Expert Swapper**| Signals expert pinning or unpinning in MoE RAM (`&PIN("lux-architect")`) |
| `^` | **Gigatoken / Eviction** | Context packing (`^GIGATOKEN_PACK`), sliding-window eviction (`^EVICT_SLIDING`) |
| `?` | **ZHARK Query** | Dispatches asynchronous research tasks (`?ZHARK(#HIVE["name"], "query")`) |
| `¿` | **FUBBU Sync** | Directs context compression and data hive aggregation (`¿FUBBU_SYNC(CTX[0])`) |
| `=` | **Sync Guard** | Multi-agent concurrency control (`=LOCK=`, `=UNLOCK=`, `=BARRIER=`, `=SYNC=`) |
| `<` | **Native Bridge** | Bidirectional native code interop (`<NATIVE_BRIDGE: CPP>`) |

---

## 3. Register & Memory Architecture

The runtime operates over eight dedicated register banks:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    A2A-DSL Virtual Machine v1.1A-ROBUST+                │
├───────────────────┬───────────────────┬─────────────────────────────────┤
│ Register Bank     │ Count & Dimensions│ Operational Scope               │
├───────────────────┼───────────────────┼─────────────────────────────────┤
│ REG[0..15]        │ 16 x 64-bit Int   │ General scalar data & pointers  │
│ HW[0..7]          │ 8 x System Def    │ Target-specific GPU/VDP/MMIO    │
│ SKL[0..3]         │ 4 x Skill Pointers│ Active Toolchain/MCP awareness  │
│ #VEC[0..7]        │ 8 x F16[1536]     │ Embeddings & Vector Memory      │
│ #QVEC[0..7]       │ 8 x U8[1536]      │ Quantized vectors (50% less BW) │
│ %SIG[0..7]        │ 8 x Float32       │ VFX & Engine Cross-Modulation   │
│ CTX[0..3]         │ 4 x Token Stream  │ Gigatoken Buffers & Ring Paging │
│ LCK[0..3]         │ 4 x Mutex Handles │ Multi-Agent Lock/Sync States    │
└───────────────────┴───────────────────┴─────────────────────────────────┘
```

---

## 4. Peak Performance Architecture (v1.1A-Robust+)

### PEAK-1: A2A-B Binary Bytecode Dialect

ASCII sigil syntax is compiled to 32-bit packed instructions at the MCP boundary:

```
[31:24] Opcode | [23:20] TargetReg | [19:16] SrcRegA | [15:12] SrcRegB | [11:8] Flags | [7:0] Immediate
```

The C++ VM executes bytecode via Direct Threaded Code jump tables — **64× smaller memory footprint** (4 bytes vs 256 bytes per instruction). Note: this figure compares instruction encoding size only; the bytecode path currently refuses payload-bearing opcodes (see GAP-08 in `GAMMA_FINALIZATION_PLAN.md`).

### PEAK-2: Quantized Vector Delta Registers

8-bit quantized vectors (`#QVEC[0..7]`) alongside differential updates (`#VEC_DELTA`) cut shared-memory ring buffer bandwidth by **50%** during multi-agent vector syncs (3,072 B/slot FP16 → 1,536 B/slot U8).

### PEAK-3: Speculative Pipeline Branching

`@PREDICT` blocks allow sub-agents to speculatively dispatch async RAG queries before acquiring mutex locks. `SPEC_COMMIT` on invariant success; `SPEC_ROLLBACK` for atomic state restoration on failure.

### PEAK-4: Hardware Interrupt Frame-Sync

`%HW_INT(VBLANK)` / `%HW_INT(DMA_CH1)` / `%HW_INT(AUDIO_SWAP)` map engine signals directly to hardware timing channels for synchronous frame-bound execution.

### PEAK-5: Sliding-Window Context Eviction

`^EVICT_SLIDING(SPAN=2048, STRIDE=512)` formalizes explicit ring-buffer directives for CTX registers, guaranteeing automatic pruning of stale conversation turns in long-horizon agent loops.

---

## 5. End-to-End Code Sample

```plaintext
// v1.1A-Robust+: Peak performance demo with all 5 additions
|TARGET: [D3DX11]
!STATE: 0x03 [SHADER_COMPILE]
+SKILL["AtlasForge"]
-SKILL["CombatEngine_Logic"]
@INVARIANT: HW[2] == STATE_READY
@ON_ERROR: {
    ~MCP("log", "D3DX11 compilation failed. Triggering GDI+ fallback.");
    !SHIFT_TARGET: [GDI_DIB];
    RETRY(MAX=2);
}
=LOCK="source_Core_X64/CameraControl.cpp"

$EXEC {
  CTX[0] = CAVEMAN_SHRINK("Compile isometric depth dither shader", LEVEL=FULL);
  ^EVICT_SLIDING(SPAN=2048, STRIDE=512);
  #HIVE["render_pipeline_shaders"]

  @PREDICT: {
    ?ZHARK(#HIVE["render_pipeline_shaders"], "Locate VBlank timing offset for DQ4 boot");
    #VEC[0] = LOAD_EMBEDDING(CTX[0]);
    #QVEC[0] = QUANTIZE(#VEC[0]);
    #QVEC[1] = QUANTIZE(#VEC[1]);
    #VEC_DELTA(#QVEC[0], #QVEC[1]);
    SPEC_COMMIT;
  }

  %HW_INT(VBLANK);
  #VEC[1] = QUERY_VECTOR_DB(#VEC[0], TOP_K=5);
  REG[0] = COSINE_SIM(#VEC[0], #VEC[1]);
  ^GIGATOKEN_PACK(CTX[0], MAX_TOKENS=2048);

  MATCH REG[0] -> {
    0x80: %SIG[0] = 0.95;
          ~MCP("vwforge_shader", CTX[0]);
          EMIT_SIGNAL(0xFF, "HIGH_SIMILARITY_EXECUTE");
    _:    %SIG[0] = 0.10;
          ABORT_TRANSITION(0xE2, "VECTOR_SIMILARITY_BELOW_THRESHOLD");
  };
}

=UNLOCK="source_Core_X64/CameraControl.cpp"

<NATIVE_BRIDGE: CPP>
void FastCompose16BPP(uint16_t* pDest, const uint16_t* pSrc, int count) {
    for(int i = 0; i < count; ++i) {
        if(pSrc[i] & 0x8000) pDest[i] = pSrc[i];
    }
}
</NATIVE_BRIDGE>
```

---

## 6. C++ Low-Overhead AST Parser Architecture

The C++ parser converts raw A2A-DSL streams directly into memory-mapped AST structs with zero dynamic heap allocation during parsing and execution (verified via `operator new` override in the test harness). v1.1A-Robust+ adds a bytecode compiler and Direct Threaded Code VM:

```cpp
// GammaLanguage/cpp/A2AST.h (v1.1A-Robust+ excerpt)
namespace A2A {

enum class Opcode : uint8_t {
    // v1.0 opcodes (0x01-0xFF)
    STATE_DECL = 0x01, INVARIANT_GUARD = 0x02, COSINE_SIM = 0x12, ...
    // v1.1A-Robust opcodes
    TARGET_DECL = 0x07, SKILL_EQUIP = 0x09, HIVE_SELECT = 0x15,
    LOCK_ACQUIRE = 0x60, ON_ERROR_BLOCK = 0x70, NATIVE_BRIDGE = 0x80, ...
    // v1.1A-Robust+ Peak Performance opcodes
    QVEC_LOAD     = 0x18,   // PEAK-2: 8-bit quantized vector load
    VEC_DELTA     = 0x19,   // PEAK-2: Differential vector update
    PREDICT_BLOCK = 0x73,   // PEAK-3: Speculative pipeline branch
    SPEC_COMMIT   = 0x74,   // PEAK-3: Commit speculative buffer
    SPEC_ROLLBACK = 0x75,   // PEAK-3: Rollback speculative buffer
    HW_INTERRUPT  = 0x90,   // PEAK-4: Hardware interrupt frame-sync
    EVICT_SLIDING = 0x31    // PEAK-5: Sliding-window context eviction
};

// PEAK-1: 32-bit packed bytecode instruction
struct A2AB_Instruction {
    uint32_t packed;  // [31:24]Op|[23:20]Tgt|[19:16]SrcA|[15:12]SrcB|[11:8]Flags|[7:0]Imm
};
static_assert(sizeof(A2AB_Instruction) == 4, "A2AB instruction must be 4 bytes");

} // namespace A2A
```

---

## 7. Performance Benchmark

> **The headline token-reduction claim does not hold. A2A-DSL uses MORE tokens than compact JSON-RPC, not fewer.**
>
> Every figure below is reproduced by `tests/benchmark_token_reduction.py` (tokens, bytes) and
> `cpp/test_a2a_parser.cpp` (parse time) on your own hardware. Earlier revisions of this table
> published a "85.8% token savings" figure that was never measured and is false.

Corpus: 5 real MCP tool calls (filesystem write, GitHub create-issue, Slack post, Postgres query,
HTTP fetch) with realistic argument payloads, written twice carrying **identical information** —
once as compact JSON-RPC 2.0 (no whitespace), once as A2A-DSL.

| Metric | Standard JSON-RPC | A2A-DSL | Result |
| :--- | :--- | :--- | :--- |
| **Payload Size (Bytes)** | 1,778 B | 1,686 B | 5.2% smaller |
| **Token Count (`cl100k_base`)** | 511 | 615 | **20.4% MORE tokens** |
| **Token Count (`o200k_base`)** | 524 | 620 | **18.3% MORE tokens** |
| **Grammar priming cost** | 0 | 2,754 tokens | One-time, per context window |
| **Parse Time (C++)** | 0.124 ms | 0.0073 ms | 17× faster (parse + execute) |
| **Vector Sync Bandwidth** | 3 KB/slot (FP16) | 1.5 KB/slot (QVec 8-bit) | 50% reduction |
| **KV-Cache RAM Footprint** | — | — | Not attributable to this language — see below |

**Why the token claim fails.** BPE tokenizers were trained on corpora saturated with JSON, so JSON's
structure is pre-merged and nearly free: `arguments` is 1 token, and `{"`, `":"`, `","`, `":{"` are
1 token each. A2A-DSL's invented identifiers are not: `EVICT_SLIDING` costs 5 tokens,
`CAVEMAN_SHRINK` 7, `FUBBU` 3, `ZHARK` 2, and every `[0]` register index costs 3 (`[`, `0`, `]`).
The design assumed sigils are cheap and JSON punctuation is expensive; the tokenizer says the
opposite. The `¿` sigil is fine (1 token); the English-like keywords are the cost.

**Parse time.** The JSON figure is an intentionally allocation-heavy `shared_ptr` + `std::map` DOM
parser built with the same compiler and flags — a conservative stand-in that is *slower* than
nlohmann/json, not faster. A previous revision of this table cited "1.85 ms (nlohmann/json)", which
implies 0.77 MB/s; mainstream JSON parsers run 100–250 MB/s. nlohmann/json has never been vendored
in this repository and that number was never measured.

**Vector sync.** `RegisterBank::Vec` is `uint16_t[1536]` = 3,072 B/slot and `QVec` is
`uint8_t[1536]` = 1,536 B/slot (`cpp/A2AST.h`). That is a 50% reduction. Earlier revisions claimed
75% / 0.75 KB per slot, which would require 4-bit quantization that the code does not implement.

**KV-cache.** There is no KV-cache quantization code in this repository. `TurboQuantRamUtil` is a
single float compared against a threshold. Any KV-cache reduction comes from TurboQuant operating
independently of A2A-DSL and should be attributed to TurboQuant, not to this language.

---

## 8. Directory Structure

* **`GAMMA_LANGUAGE_EBNF_SPEC.md`**: Formal EBNF grammar, sigil dictionary, register architecture, and full specification.
* **`cpp/`**: Zero-allocation, cache-line aligned C++ AST parser, auto-vectorized cosine similarity, A2A-B bytecode VM, GPU compute stub, shared memory bus, and unit tests.
  * `A2AST.h` — AST nodes, opcodes, register banks, bytecode format
  * `A2ALexer.h/.cpp` — Zero-alloc sigil lexer
  * `A2AEvaluator.h/.cpp` — AST + bytecode execution engine
  * `A2AGPUCompute.h/.cpp` — GPU offload stub (software fallback; Vulkan not yet wired)
  * `A2ASharedMemory.h/.cpp` — Inter-process ring buffer for multi-agent sync
  * `test_a2a_parser.cpp` — Unit tests + throughput benchmarks
  * `CMakeLists.txt` — Build configuration
* **`python/`**: Python CLI bridge, Caveman compression, TurboQuant/Nexus adapters.
  * `gamma_bridge.py` — GammaBridge class, v1.1A-Robust+ parser
  * `gamma_cli.py` — CLI with `--test`, `--script`, `--json`, `--agent` flags
  * `requirements.txt` — Zero external dependencies (stdlib only)
* **`examples/`**: Ready-to-run `.gamma` script examples.
  * `sample_crossmod.gamma` — v1.0 cross-modulation demo
  * `sample_vector_query.gamma` — v1.0 vector query demo
  * `sample_v11a_robust.gamma` — v1.1A-Robust multi-agent shader compile
  * `sample_psx_research.gamma` — PSX disassembly research delegation
  * `sample_peak_performance.gamma` — All 5 peak performance additions
  * `sample_rust_peak_interop.gamma` — v1.1C Rust FFI / CyberGrime / PSXMatrix interop
* **`install.ps1` / `install.bat`**: One-click installer scripts.

---

## 9. Quick Start

### Python CLI

```bash
python python/gamma_cli.py --test
python python/gamma_cli.py --script examples/sample_peak_performance.gamma
python python/gamma_cli.py --json
```

### C++ Parser

```bash
cd cpp && cmake -B build && cmake --build build
.\build\gamma_parser.exe
```

---

*Liminal A2A-DSL v1.1C — A deterministic, sandboxed VM for agent programs with a bytecode target and hardware/emulator-interop scripting as its flagship reference application.*


