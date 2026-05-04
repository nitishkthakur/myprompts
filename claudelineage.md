# ETL Column Lineage Pipeline — GitHub Copilot Agent Mode Blueprint

> **Scope:** A complete implementation plan for the ETL column-lineage extraction system covering 291+ Teradata stored procs and 65 AbInitio binary files, designed exclusively for **GitHub Copilot Agent Mode in VS Code**. No DeepAgents, no direct API access, no Python framework outside what the Copilot agent invokes via its integrated terminal.
>
> **What this means in practice:** The Python tooling layer (sqlglot-based parsers, DAG builders, exporters) still exists — it just runs as a set of local scripts in `/tools/` that the Copilot agent invokes through the terminal. Copilot is the orchestrator and the LLM-fallback parser; the Python scripts are the deterministic engine.
>
> Code snippets are illustrative; design decisions are fully resolved.

---

## Table of Contents

1. [The Operating Model — How Copilot Drives This](#1-the-operating-model)
2. [Foundational Architecture — The 4-Phase Pipeline](#2-foundational-architecture)
3. [The Pre-Processing Layer](#3-the-pre-processing-layer)
4. [The Lineage Graph — Conceptual Model](#4-the-lineage-graph)
5. [Top 2 Approaches Per Deliverable](#5-top-2-approaches-per-deliverable)
6. [Repository Layout](#6-repository-layout)
7. [`.vscode/mcp.json` — MCP Configuration](#7-vscodemcpjson)
8. [`.github/copilot-instructions.md` — Always-On Instructions](#8-githubcopilot-instructionsmd)
9. [Path-Scoped Instruction Files](#9-path-scoped-instruction-files)
10. [The Five Prompt Files — Phase Drivers](#10-the-five-prompt-files)
11. [The Resume Strategy](#11-the-resume-strategy)
12. [Hard Limits and Fallbacks](#12-hard-limits-and-fallbacks)
13. [Cross-Cutting Concerns](#13-cross-cutting-concerns)
14. [Validation & Acceptance Criteria](#14-validation--acceptance-criteria)

---

## 1. The Operating Model

A common mistake on a project like this is asking the agent to *be* the parser. It is not. The agent is an orchestrator that:

- **Invokes deterministic Python tools** for parsing, graph construction, and artifact rendering, via the VS Code integrated terminal.
- **Reads files directly via the filesystem MCP** when context-sensitive reasoning is needed (e.g., interpreting a dynamic SQL block).
- **Writes intermediate state to disk** at every checkpoint, so the run is resumable across sessions.
- **Is invoked phase-by-phase** through five numbered prompt files, not a single mega-prompt.

This separation is essential because:
- The corpus (291+ procs of 70–3000+ lines, 65 AbInitio binaries, hundreds of output columns) **will not fit in any single Copilot session**.
- LLM-driven parsing of every file would burn the premium-request quota and produce inconsistent results across runs.
- Deterministic Python tools (sqlglot + networkx) parse Teradata SQL more reliably than the agent reasoning over raw text.

**The agent's actual cognitive work is concentrated in two places:**
1. Phase 3 — interpreting dynamic SQL blocks that the static parser flagged as `needs_llm_fallback`.
2. Phase 2 — sanity-checking the inferred generation boundaries against expected ETL semantics.

Everything else is orchestration: read this, run this script, check the output, write a checkpoint.

---

## 2. Foundational Architecture

The pipeline has four phases, executed in order, each driven by its own prompt file:

```
┌──────────────────┐   ┌──────────────────┐   ┌──────────────────┐   ┌──────────────────┐
│ PHASE 1          │   │ PHASE 2          │   │ PHASE 3          │   │ PHASE 4          │
│ Pre-process &    │ → │ Build file-DAG   │ → │ Per-column       │ → │ Render artifacts │
│ index every file │   │ & infer          │   │ backward         │   │ (xlsx, json,     │
│                  │   │ generations      │   │ lineage trace    │   │  mermaid, txt)   │
└──────────────────┘   └──────────────────┘   └──────────────────┘   └──────────────────┘
```

**Why this separation matters:** Phase 1 and 2 produce stable, file-scoped JSON artifacts. Phase 3 reads only those JSON artifacts (not raw SQL again), which is what makes hundreds of column traces tractable inside Copilot. **Each raw `.sql` file is read at most twice** — once for static parsing in Phase 1, once for LLM fallback in Phase 3 if dynamic SQL was flagged. After that, all work is on cached structured data.

### 2.1 Output File Layout

Every artifact lives under a single `output/` tree. Copilot writes here via the filesystem MCP and the integrated terminal.

```
output/
├── cache/                                    # Phase 1 outputs — one file per source file
│   ├── ct__proc_ct_bmg_stg.json
│   ├── tf__proc_tf_bmg_xform.json
│   ├── ex__proc_ex_bmg_load.json
│   ├── cv__proc_cv_bmg_view.json
│   └── abinitio__load_raw_txns.json          # minimal stub
├── graph/
│   ├── file_dag.json                         # Phase 2 output: file → file edges
│   ├── table_index.json                      # table/view → producing file(s)
│   ├── inferred_schema.json                  # progressively-built schema for sqlglot
│   ├── execution_order.txt                   # Deliverable 2 (human-readable)
│   └── execution_dag.adjlist                 # Deliverable 2 (machine-readable)
├── lineage.json                              # Phase 3 output (canonical, source of truth)
├── long.xlsx                                 # Phase 4 (derived)
├── wide.xlsx                                 # Phase 4 (derived)
├── diagrams/                                 # Phase 4 (derived)
│   ├── final_table_a/
│   │   ├── net_amount.mmd
│   │   ├── customer_id.mmd
│   │   └── …
│   └── final_table_b/
└── _run_log/
    ├── progress.json                         # checkpoint state for resumes
    ├── columns_to_trace.json                 # Phase 3 input (computed at start of Phase 3)
    ├── manual_review.txt                     # low-confidence hops for human review
    └── extraction_warnings.log
```

---

## 3. The Pre-Processing Layer

Before any lineage logic runs, every `.sql` file is converted into a normalized JSON record. **Every downstream component reads these JSON records, not raw SQL.** This single design choice is what makes the project tractable inside Copilot.

### 3.1 The per-file JSON record

```json
{
  "filename": "tf/proc_tf_bmg_xform.sql",
  "proc_type": "TF",
  "proc_name": "PROC_TF_BMG_XFORM",
  "line_count": 847,
  "has_dynamic_sql": true,
  "statements": [
    {
      "stmt_id": "stmt_001",
      "stmt_type": "INSERT_SELECT",
      "extraction_method": "static_parser",
      "raw_sql": "INSERT INTO stg_bmg_xform SELECT … FROM cv_bmg_prev_gen WHERE …;",
      "input_tables": ["cv_bmg_prev_gen"],
      "output_table": "stg_bmg_xform",
      "column_mapping": [
        {
          "output_column": "net_amount",
          "input_columns": ["gross_amount", "tax_amount"],
          "input_tables": ["cv_bmg_prev_gen"],
          "expression_sql": "gross_amount - tax_amount",
          "transformation_summary": "Subtracts tax_amount from gross_amount"
        }
      ],
      "filter_predicates": ["status <> 'CANCELLED'"]
    },
    {
      "stmt_id": "stmt_002",
      "stmt_type": "DYNAMIC_SQL",
      "extraction_method": "dynamic_reconstructed",
      "raw_sql": "-- Dynamic SQL: best-effort reconstruction. INSERT INTO stg_bmg_xform SELECT … FROM <runtime_param>",
      "reconstruction_confidence": 0.4,
      "input_tables": ["<runtime_param>"],
      "output_table": "stg_bmg_xform",
      "column_mapping": [],
      "needs_llm_fallback": true,
      "fallback_context_lines": [410, 460]
    }
  ],
  "output_tables_written": ["stg_bmg_xform"],
  "input_tables_read": ["cv_bmg_prev_gen"]
}
```

Notes:
- `input_tables_read` and `output_tables_written` at the top level are the **union** of every statement's I/O. Phase 2 uses these for DAG construction.
- `column_mapping` at the statement level is what Phase 3 uses for column tracing.
- `extraction_method` is propagated all the way to the final lineage row.
- `fallback_context_lines` tells Phase 3 exactly which line range to re-read if LLM fallback is triggered — the agent doesn't have to guess.

### 3.2 The Teradata Proc Body Extractor (`tools/teradata_proc_extractor.py`)

`sqlglot` cannot parse a `CREATE PROCEDURE … BEGIN … END` block as a single unit. It can parse the inner statements once they are isolated. The extractor performs these steps:

**Step 1 — Strip the procedure wrapper.** Match `CREATE PROCEDURE …` or `REPLACE PROCEDURE …`, capture the proc name, and isolate the body between the outermost `BEGIN` and matching `END`.

**Step 2 — Skip declarations and control flow scaffolding.** Lines or blocks beginning with these keywords contribute no lineage and are removed before parsing:
- `DECLARE` (variable, cursor, or condition handler)
- `DECLARE … HANDLER FOR …`
- Standalone `IF` / `END IF` / `ELSE` / `ELSEIF` / `WHILE` / `END WHILE` / `LOOP` / `END LOOP` / `LEAVE` / `ITERATE` keywords (the *body* inside these blocks is kept; only the control-flow keywords are stripped)
- `SET <var> = <scalar>` lines that don't construct SQL strings (see §3.3 for the exception)

**Step 3 — Split into statement-level chunks.** Teradata uses `;` as statement terminator. Naïve splitting on `;` breaks if a literal contains a semicolon, so the splitter is aware of single quotes and `--` / `/* */` comments. The output is a list of candidate SQL strings.

**Step 4 — Classify each chunk.** Run a fast regex pass to bucket each chunk into:
- `INSERT_SELECT` (`INSERT INTO … SELECT …`)
- `MERGE` (`MERGE INTO …`)
- `UPDATE_FROM` (`UPDATE … FROM …`)
- `CREATE_VIEW` / `REPLACE_VIEW`
- `DYNAMIC_SQL` (sql-string assignment + later execution)
- `OTHER` (e.g. `COLLECT STATISTICS`, BTEQ directives — discarded for lineage purposes)

**Step 5 — For each parseable bucket, run sqlglot.** Use `sqlglot.parse_one(chunk, dialect="teradata")`. For column-level lineage within the statement, use `sqlglot.lineage(column, sql, dialect="teradata", schema=…, sources=…)`. The schema/sources are populated lazily as more files are pre-processed (see §13.3).

**Step 6 — On parse failure, flag for LLM fallback.** Don't skip silently. Record the chunk in `needs_llm_fallback: true` and let Phase 3 escalate.

### 3.3 Dynamic SQL Reconstruction (`tools/dynamic_sql_reconstructor.py`)

Teradata dynamic SQL has a recognizable shape:

```sql
DECLARE sql_buf VARCHAR(10000);
SET sql_buf = 'INSERT INTO ' || target_tbl || ' SELECT col_a, col_b - col_c AS net FROM ' || source_tbl;
SET sql_buf = sql_buf || ' WHERE batch_dt = ' || batch_param;
CALL DBC.SysExecSQL(sql_buf);
```

The reconstructor performs **intra-procedure variable propagation**:

1. Walk the proc body line-by-line, maintaining a `Dict[var_name, str]` of currently-known variable values.
2. When a `SET` assigns a literal, record it.
3. When a `SET` concatenates literals + parameters, record the partial form with `<param_name>` placeholders.
4. When a `CALL DBC.SysExecSQL(<var>)` or `EXECUTE IMMEDIATE <var>` is hit, freeze the current value of `<var>` as the reconstructed SQL string for that call.
5. Run that reconstructed string through the same sqlglot parser as static SQL. If parse succeeds despite `<param>` placeholders, treat it as parseable but tag `extraction_method = "dynamic_reconstructed"`.
6. If parse fails (string still has too many runtime gaps), tag `needs_llm_fallback: true` and store the best-effort string + the proc body line range for later.

**Confidence scoring:** A simple heuristic — `confidence = (literal_chars) / (literal_chars + placeholder_chars)`. Anything below 0.6 is downgraded to `llm_inferred` even if it parses.

### 3.4 Long Procs (>1500 lines)

These are the danger zone. They typically contain 5–15 separate `INSERT INTO` statements, sometimes feeding the same table conditionally. The strategy:

1. The proc body extractor is **statement-oriented, not line-oriented**. A 3000-line proc with 12 INSERT statements becomes 12 statement records, each ~250 lines on average. This is well within Copilot's context for any LLM fallback.
2. Each statement is processed **independently** by the Python parser. The aggregation happens at `output_tables_written` / `input_tables_read` rollup time.
3. For LLM fallback in Phase 3, **never send the whole proc to Copilot** — the agent's instructions tell it to re-read only the specific line range from `fallback_context_lines`, plus a 20-line surrounding window.

### 3.5 AbInitio Files — The Black Box Stub

Each `.mp` file gets a minimal stub record. The agent is instructed never to open `.mp` binaries.

```json
{
  "filename": "abinitio/load_raw_txns.mp",
  "proc_type": "AbInitio",
  "is_black_box": true,
  "extraction_method": "abinitio_blackbox",
  "input_tables_read": ["<external_source_unknown>"],
  "output_tables_written": []
}
```

`output_tables_written` is initially empty. It's filled in during Phase 2 by pairing each AbInitio file with the CT proc in its parallel block (see §4.4).

---

## 4. The Lineage Graph

The graph has two layers:

### 4.1 Layer 1 — File-DAG (Deliverable 2: Execution Order)

- **Nodes:** files (both `.sql` and `.mp`)
- **Edges:** `F1 → F2` if any table in `F1.output_tables_written` appears in `F2.input_tables_read`
- **Implementation:** `networkx.DiGraph()`. Topological sort gives execution order. Antichains (sets of nodes with no edges between them) form the parallel blocks within a generation.

### 4.2 Layer 2 — Table/View-DAG (Deliverable 1: Column Lineage)

- **Nodes:** tables and views (e.g. `stg_bmg_xform`, `cf_bmg_xform`, `cv_bmg_gen2`)
- **Edges:** `T1 → T2` if a file writes T2 by reading T1
- **Edge metadata:** the file responsible, the column-level mapping JSON from §3.1
- **This is the graph that lineage tracing walks.** Starting from a final-output column, walk edges *backward* through the table-DAG. At each edge, look up the column-level mapping to find which input column(s) feed the current column.

### 4.3 Generation Inference (Scenario B — primary)

Generations are derived **from the file-DAG, not from filenames**:

1. Identify the 4 final-generation SQL files: they are CV procs whose output views are not consumed by any other file in the corpus. (Equivalently: their output tables/views have **out-degree 0** in the file-DAG.)
2. Mark these as `generation = "final"`.
3. For every other file, `generation = max(predecessor_generation) + 1`, computed bottom-up via reverse topological order.

**Important wrinkle:** A CT proc has no inputs from earlier generations; it stands alone as Step 1 of its block. To assign it a generation, pair it with the EX/CV procs in its block (they share the staging table the CT creates) and inherit the block's generation from those.

### 4.4 Parallel Block Detection

Within a single generation, parallel blocks are sets of files that:
- Share the same generation number, AND
- Have no edges between them in the file-DAG, AND
- Each form a complete CT→{TF|AbInitio}→EX→CV chain (the block's 4 files are connected to each other but not to the other blocks)

The detection algorithm: for each generation, run **weakly connected components** on the subgraph induced by that generation. Each component is a parallel block. Within a block, the 4 files are ordered by proc type: CT (Step 1) → TF/AbInitio (Step 2) → EX (Step 3) → CV (Step 4).

### 4.5 AbInitio Stub Patching

After Phase 2 produces the parallel block decomposition, walk every block in Generation 1. For each block:
- Find the AbInitio file (by extension `.mp`)
- Find the CT proc in the same block
- Read the CT proc's cache JSON; copy its single entry from `output_tables_written` into the AbInitio stub's `output_tables_written`

This gives the AbInitio stub a real output table name to participate in the table-DAG, even though its internal transformations remain black-boxed.

---

## 5. Top 2 Approaches Per Deliverable

Both approaches in each pair are feasible inside Copilot Agent Mode. Approach A is the recommended primary in every case; Approach B is a fallback or supplemental option.

### 5.1 Deliverable 1 — Column Lineage Documentation

#### Approach 1A — Pre-cached Graph + Targeted Backward Walk *(recommended primary)*

**Summary:** Pre-process every file once into the JSON cache (Phase 1, via `teradata_proc_extractor.py`). Build the table-DAG with column-level metadata on every edge (Phase 2, via `dag_builder.py`). For each final-gen output column, walk the table-DAG backward (Phase 3, via `lineage_walker.py`), emitting one lineage hop per file traversed. Copilot's role is orchestration, plus LLM fallback on hops the walker flagged as needing it.

**Key technical steps:**
1. Phase 1 — Copilot iterates `/ct`, `/tf`, `/ex`, `/cv`, `/abinitio` directories, invoking `python tools/teradata_proc_extractor.py <file>` per file.
2. Phase 2 — Copilot invokes `python tools/dag_builder.py`, producing the file-DAG and table-index.
3. Phase 3 — for each column: invoke `python tools/lineage_walker.py --column <col>`. The walker handles all static cases. For any hop it returns with `needs_llm_fallback: true`, Copilot re-reads the relevant line range from the original proc, reasons about the dynamic SQL, and patches the hop manually.
4. Each completed column is appended to `output/lineage.json` immediately.

**Failure modes & mitigations at this scale:**
- *Hallucinated joins between unrelated files:* the table-DAG only contains edges that were materially written/read by a file. If a join is hallucinated, no edge exists, the walk terminates cleanly. The DAG is the truth.
- *Lost state on long lineage chains:* the walk is iterative and writes to `lineage.json` after each *column* (not each hop) completes. Resume can pick up at the next unfinished column.
- *Renames across files:* the `OutputColumn` field in each hop tracks the column's name *at that file's output*. The walker uses this name (not `FinalColumnName`) to look up the next hop's mapping. Handles arbitrary rename chains.
- *Dynamic SQL hops:* surfaced as `extraction_method = "dynamic_reconstructed"` or `"llm_inferred"`. The walk continues normally; only the confidence flag changes.
- *Context overflow on Copilot side:* mitigated because Copilot reads JSON cache files (small, structured) for the bulk of the work, not raw SQL. Raw SQL is only re-read for LLM-fallback hops, and only the flagged line range.

**Why this fits Copilot specifically:** Copilot Agent Mode excels at orchestrating tools (terminal commands, file reads) and reasoning about specific bounded problems. It does *not* excel at parsing 3000-line stored procedures from scratch. This approach plays to Copilot's strengths.

#### Approach 1B — Pure LLM-Driven Per-Column Trace *(fallback)*

**Summary:** Skip the Python tooling. For each final-output column, Copilot reads the producing file directly, identifies the input table/column, recurses to the producing file of that input, and emits lineage hops. No pre-processing cache, no DAG.

**Key technical steps:**
1. Read the 4 final SQL files. Extract output columns by reasoning over the SELECT clauses.
2. For each column, read the file that produced its source view, identify the input column, and recurse.
3. Write each completed column to `lineage.json`.

**Failure modes & mitigations:**
- *Hallucinated transformations:* LLMs invent plausible SQL when underspecified. Mitigation: require Copilot to cite the exact line range from the source file in every hop. Reject hops whose `transformation_in_sql` doesn't appear verbatim in the cited file.
- *Inconsistency across columns:* same input file is summarized differently for different columns. Mitigation: cache file summaries between column traces; never let Copilot re-summarize.
- *Cost:* every column trace re-reads multiple full SQL files. The premium-request quota would not survive a hundred-column run.
- *Context overflow on long procs:* unmitigated. A 3000-line proc may not fit in Copilot's working context, and chunked reading without an AST parser misses cross-statement dependencies.

**When to use 1B:** only as an emergency fallback when the Python tooling layer is unavailable (e.g. corporate environment that prohibits running unsigned scripts). Not recommended as a primary approach.

**Recommended pattern:** Use 1A as the spine. Use Copilot's reasoning only for hops where the walker reports `needs_llm_fallback`. The `extraction_method` tag in every output row is the audit trail.

---

### 5.2 Deliverable 2 — Pipeline Execution Order

#### Approach 2A — File-DAG Topological Sort *(recommended primary)*

**Summary:** Build the file-DAG from Phase 1 metadata. Topologically sort it. Group antichains into parallel blocks. Emit the structured text and adjacency-list outputs. All work is done by `tools/dag_builder.py`, which Copilot invokes from the terminal.

**Key technical steps:**
1. From every per-file JSON record, read `output_tables_written` and `input_tables_read`.
2. Build a global `table → producing_file` map.
3. For each file F and each table T in F's `input_tables_read`, look up T's producer F'. Add edge F' → F.
4. Run `nx.topological_sort()`. Group nodes by `generation` (per §4.3).
5. Within each generation, find parallel blocks via weakly connected components.
6. Within each block, order files by proc type: CT, TF/AbInitio, EX, CV.

**Failure modes & mitigations:**
- *Cycles* (would be an ETL bug): detect with `nx.find_cycle()`. Report and abort — don't silently break the cycle, since it indicates real data corruption in the pipeline definition.
- *Orphan tables* (read by a file but never written): these are external source tables. Add a synthetic `<external_source>` node as the predecessor.
- *Ambiguous table references* (same name in different schemas): keep schema-qualified names throughout. Default to `<unqualified>` schema if not specified, and warn.

**Why this fits Copilot:** the agent runs one terminal command and validates its output. The deterministic part lives in Python where it belongs.

#### Approach 2B — Filename-Heuristic Pre-Group + Validation *(Scenario A only)*

**Summary:** When filenames carry a generation token (Scenario A), Copilot pre-groups files by inspecting filenames, then validates by sampling a few SQL files to confirm the heuristic is consistent. If validation fails, fall back to Approach 2A for that file.

**Key technical steps:**
1. Copilot lists every filename, identifies the generation discriminator (e.g. `_g1_`, `_g2_`), and groups them.
2. For a sample of 5 files per group, Copilot reads the SQL and confirms: TF procs in tentative gen N read from CV procs in tentative gen N-1.
3. On mismatch, mark the file as needing 2A re-evaluation.

**Failure modes & mitigations:**
- *Partial naming conventions:* some files conform, others don't. Use 2B for the conformers and 2A for the rest. Don't force-fit.
- *Sampling misses bugs:* validating 5 of 30 files can miss a single mis-named file. The reliance on `dag_builder.py` for the final answer mitigates this.

**When to use 2B:** only as a quick first pass under Scenario A, before running 2A as the authoritative pass. **For Scenario B (the primary case), use 2A directly.**

---

### 5.3 Deliverable 3 — Mermaid Diagrams

#### Approach 3A — Deterministic Generator from `lineage.json` *(strongly recommended primary)*

**Summary:** Copilot invokes `tools/mermaid_renderer.py` from the terminal. The script reads `lineage.json` and emits one `.mmd` file per output column. Pure rendering — no LLM involvement. Identical input always produces identical output. Re-runs are free.

**Key technical steps:**
1. For each entry in `lineage.json["columns"]`, gather the ordered list of hops.
2. For each hop, emit a node for its `output_table` (or stadium shape `([…])` if proc type is AbInitio).
3. For each hop, emit an edge from the previous hop's `output_table` to the current `output_table`, labelled `filename | one-line-summary`. Use `-.->` if `extraction_method` is `dynamic_reconstructed` or `llm_inferred`; use `-->` otherwise.
4. The "one-line summary" is the first sentence of `transformation_in_words`, truncated to 80 chars. Truncation is deterministic.
5. Write to `output/diagrams/<final_table_name>/<column_name>.mmd`.

**Failure modes & mitigations:**
- *Special characters in column names breaking Mermaid syntax:* sanitize. Replace `[]{}<>|` with underscores; quote labels.
- *Very deep lineage producing unreadable diagrams:* if `total_hops > 8`, generate two diagrams: a top-level summary collapsing per-generation, and the detailed full-depth view.

**Why this fits Copilot:** the agent runs one terminal command. Done.

#### Approach 3B — LLM-Drafted with Programmatic Validation *(situational)*

**Summary:** Copilot drafts Mermaid for each column using its own reasoning over `lineage.json`. A programmatic validator then checks every node and edge label matches what's in the JSON. Reject and regenerate on mismatch.

**Key technical steps:**
1. Same input as 3A: a column's hops from `lineage.json`.
2. Copilot receives the hops + the Mermaid template + diagram requirements.
3. Validator parses the Mermaid output, extracts node/edge labels, checks each one is sourced from the JSON.
4. On any drift, regenerate or fall back to 3A.

**Failure modes & mitigations:**
- *Slower than 3A by orders of magnitude:* yes — only useful if narrative-style edge labels are required.
- *Hallucinated nodes:* validator catches these.

**When to use 3B:** reserved for a future enhancement if analysts want richer narrative captions on the diagrams. Not part of the initial cut. **Use 3A.**

---

## 6. Repository Layout

```
<repo-root>/
├── .vscode/
│   └── mcp.json                              # MCP server config
├── .github/
│   ├── copilot-instructions.md               # Always-on instructions
│   ├── instructions/
│   │   ├── teradata-parsing.instructions.md  # Activates on *.sql
│   │   ├── abinitio-handling.instructions.md # Activates on *.mp paths
│   │   └── output-format.instructions.md     # Activates on output/**
│   └── prompts/
│       ├── 01-discover-and-index.prompt.md
│       ├── 02-build-file-dag.prompt.md
│       ├── 03-trace-column-lineage.prompt.md
│       ├── 04-render-diagrams.prompt.md
│       └── 05-export-artifacts.prompt.md
├── ct/                                       # Source SQL — read-only for the agent
├── tf/
├── ex/
├── cv/
├── abinitio/                                 # .mp binaries — never opened
├── tools/                                    # Local Python helpers the agent invokes
│   ├── teradata_proc_extractor.py
│   ├── dynamic_sql_reconstructor.py
│   ├── dag_builder.py
│   ├── lineage_walker.py
│   ├── mermaid_renderer.py
│   ├── xlsx_exporter.py
│   └── requirements.txt                      # sqlglot==<pinned>, networkx, openpyxl, pandas
└── output/                                   # All generated artifacts go here
```

**Setup prerequisite:** before running any prompt, the user (or a one-shot prompt) ensures `pip install -r tools/requirements.txt` has been executed in the workspace's Python environment. Copilot can be asked to do this in a setup prompt; it should not be assumed.

---

## 7. `.vscode/mcp.json`

The minimum viable config uses two MCP servers — filesystem for reading the SQL corpus and writing artifacts, and a memory server for accumulating graph state across sessions.

```jsonc
{
  "servers": {
    "filesystem": {
      "type": "stdio",
      "command": "npx",
      "args": [
        "-y",
        "@modelcontextprotocol/server-filesystem",
        "${workspaceFolder}/ct",
        "${workspaceFolder}/tf",
        "${workspaceFolder}/ex",
        "${workspaceFolder}/cv",
        "${workspaceFolder}/abinitio",
        "${workspaceFolder}/output",
        "${workspaceFolder}/tools"
      ]
    },
    "memory": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-memory"]
    }
  }
}
```

**Notes:**
- `servers` (plural, no camelCase) is the correct VS Code root key. Other clients (Claude Desktop, Cursor) use `mcpServers`. Get this wrong and the file silently fails to load.
- Filesystem is restricted to the directories the agent legitimately needs. Don't expose the user's home directory.
- The memory server is the official MCP reference implementation that stores key-value state to a local SQLite. Use it to persist the file-DAG and lineage progress across sessions.
- **Optional additions:** a `git` MCP for change tracking on the SQL corpus, and a `time` MCP for `generated_at` metadata stamping. Add only if they earn their slot — every server adds tools that count against the 128-tool ceiling.

---

## 8. `.github/copilot-instructions.md`

This is the always-on system prompt. Keep it under ~3000 tokens.

```markdown
# Copilot Instructions — Teradata ETL Lineage Project

## Project goal
Extract column-level lineage from a Teradata ETL pipeline of 291+ stored procs and 65 AbInitio binaries. Produce three artifacts in /output: lineage.json (canonical), long.xlsx, wide.xlsx, plus per-column Mermaid diagrams in /output/diagrams.

## Hard rules
- Never modify any file under /ct, /tf, /ex, /cv, or /abinitio. They are read-only source.
- Never attempt to read or parse any .mp file. AbInitio files are black boxes. Record the filename only.
- Always use the Teradata SQL dialect when invoking sqlglot (the helper scripts in /tools already do this).
- Every lineage hop you emit must include an `extraction_method` field: static_parser | dynamic_reconstructed | llm_inferred | abinitio_blackbox.
- Cite exact filenames in every hop. Never reference a file you have not read in the current session.
- The 4 final-generation SQL files have explicitly-named output columns. Do not infer columns from schema or imagination.

## Workflow
You operate in 5 phases driven by the prompt files in .github/prompts/. Run them in numeric order. Do not skip phases. Each phase reads the previous phase's output from /output and writes its own outputs to /output. After each phase, write a checkpoint to /output/_run_log/progress.json so the run is resumable.

## Tooling preference
- For deterministic work (parsing, DAG construction, Mermaid rendering, xlsx export), invoke the Python scripts in /tools via the integrated terminal. Do not reimplement them.
- For ambiguous work (dynamic SQL reconstruction, column-name resolution across procs), use your own reasoning, but always cross-check by reading the actual proc body.
- When you read a proc body for LLM-fallback reasoning, only read the line range specified in the cache JSON's `fallback_context_lines` field, plus 20 lines on either side. Do not read whole long procs into context.

## Dynamic SQL
Many procs construct SQL strings via concatenation and execute via CALL DBC.SysExecSQL or EXECUTE IMMEDIATE. When a cache JSON marks a hop as `needs_llm_fallback: true`:
1. Read the specified line range from the original proc.
2. Reason about what tables, columns, and transformation the dynamic block produces.
3. Cite the exact line range in `transformation_in_sql` as a comment header.
4. Mark the hop with extraction_method = "llm_inferred".
5. If you cannot determine a value confidently, output `<unknown>` rather than guessing.

## Output schemas
See .github/instructions/output-format.instructions.md.
```

---

## 9. Path-Scoped Instruction Files

These files use VS Code's `applyTo` frontmatter to activate only when relevant files are in scope. They keep the always-on prompt small while ensuring per-file-type rules are enforced.

### 9.1 `.github/instructions/teradata-parsing.instructions.md`

```markdown
---
applyTo: "**/*.sql"
---
# Teradata SQL parsing rules

When reading any .sql file:
- Recognize CREATE PROCEDURE / REPLACE PROCEDURE wrappers. The body lives between the outermost BEGIN and END.
- Inside the body, the statements that produce lineage are: INSERT INTO … SELECT, MERGE INTO, UPDATE … FROM, CREATE VIEW, REPLACE VIEW.
- Statements that DO NOT produce lineage and can be ignored: DECLARE, SET (unless building a SQL string), IF/ELSE/WHILE/LOOP control flow keywords (their bodies are kept but the keywords themselves are not lineage), COLLECT STATISTICS, BTEQ directives.
- QUALIFY clauses (Teradata-specific window-function filter) act like WHERE for lineage purposes — included rows only.
- Volatile tables (CREATE VOLATILE TABLE) are intra-session scratch tables. They DO contribute to lineage if a later statement in the same proc reads from them. Treat them like any other intermediate table.
- MULTISET vs SET only affects duplicate handling, not lineage. Ignore.

When invoking sqlglot, always pass dialect="teradata".
```

### 9.2 `.github/instructions/abinitio-handling.instructions.md`

```markdown
---
applyTo: "**/abinitio/**"
---
# AbInitio file handling

These are .mp binary files. They are black boxes.

NEVER:
- Open the binary contents.
- Attempt to extract column mappings.
- Guess transformations based on filename alone.

ALWAYS:
- Record the filename in the lineage hop.
- Set extraction_method to "abinitio_blackbox".
- Set transformation_in_sql to: "-- AbInitio black box: <filename>"
- Set transformation_in_words to: "Source load via AbInitio. Transformations not parseable. The staging table populated by this AbInitio graph is <staging_table>, which is created by <paired_ct_proc_filename>."
- input_tables: ["<external_source_unknown>"] unless documentation outside the .mp file specifies otherwise.
- output_table: the staging table name from the paired CT proc (look it up in /output/graph/table_index.json).
```

### 9.3 `.github/instructions/output-format.instructions.md`

```markdown
---
applyTo: "output/**"
---
# Output artifact contracts

## lineage.json schema
[paste the full schema from the spec — including metadata block, columns array, hops array]

## long.xlsx columns (in order)
FinalColumnName, FinalTableName, StepNumber, Generation, Filename, ProcType, InputTable, InputColumn, OutputTable, OutputColumn, TransformationInWords, TransformationInSQL, ExtractionMethod

## wide.xlsx columns (in order)
FinalColumnName, FinalTableName, TotalHops, then for each hop k in 1..MAX_HOPS:
InputTable_hopK, InputColumn_hopK, OutputTable_hopK, OutputColumn_hopK, TransformationInWords_hopK, TransformationInSQL_hopK, Filename_hopK, ProcType_hopK, Generation_hopK, ExtractionMethod_hopK

## Rules
- lineage.json is the source of truth. long.xlsx and wide.xlsx are deterministic projections.
- Never edit xlsx files directly. Always regenerate them from lineage.json by running tools/xlsx_exporter.py.
- The JSON is written incrementally — append a column entry as soon as its trace completes. Do not hold all columns in memory.
```

---

## 10. The Five Prompt Files

Each prompt file is invoked in Agent Mode as a self-contained instruction. Copilot reads it, executes the phase, writes a checkpoint when done, and stops.

### 10.1 `01-discover-and-index.prompt.md` — Phase 1 (Pre-processing)

```markdown
---
mode: agent
description: Discover all source files, run the pre-processor, write per-file JSON cache.
---
1. List every file under /ct, /tf, /ex, /cv, /abinitio. Record counts in output/_run_log/progress.json.
2. For each .sql file, invoke `python tools/teradata_proc_extractor.py <file>` from the integrated terminal. Capture its stdout and write to output/cache/<proc_type>__<basename>.json.
3. For each .mp file, write a stub JSON to output/cache/abinitio__<basename>.json with the schema from .github/instructions/abinitio-handling.instructions.md.
4. After every 50 files processed, update output/_run_log/progress.json with the count and the last filename done.
5. When all files are processed, write output/_run_log/progress.json with phase: 1, status: complete.

Important: do NOT proceed to Phase 2 in the same session. Phase 1 alone may take many tool calls — close out and resume in a new session for Phase 2.
```

### 10.2 `02-build-file-dag.prompt.md` — Phase 2 (Graph construction)

```markdown
---
mode: agent
description: Build the file-DAG and table index from the cache. Infer generations and parallel blocks.
---
Prerequisite: output/_run_log/progress.json shows phase 1 complete.

1. Invoke `python tools/dag_builder.py --cache-dir output/cache --out-dir output/graph` from the terminal.
2. Verify output/graph/file_dag.json, output/graph/table_index.json, output/graph/inferred_schema.json, output/graph/execution_order.txt, and output/graph/execution_dag.adjlist were all written.
3. Open execution_order.txt and confirm: the last generation contains exactly 4 CV procs. If not, report the discrepancy as a warning to output/_run_log/extraction_warnings.log.
4. After Phase 2 completes, also fill in output_tables_written for AbInitio stubs:
   - For each AbInitio file F, find the parallel block it belongs to from execution_order.txt.
   - Find the CT proc in that block. Read its cache JSON. Take its output_tables_written.
   - Patch the AbInitio stub in output/cache/.
5. Sanity-check by reading 3 randomly-sampled cv→tf transitions from execution_order.txt: verify the CV proc's output view name matches a FROM-clause table in the next-generation TF proc. Report mismatches as warnings.
6. Write checkpoint: phase: 2, status: complete.
```

### 10.3 `03-trace-column-lineage.prompt.md` — Phase 3 (Per-column trace)

This is the most complex phase. It will require multiple Copilot sessions for any non-trivial corpus.

```markdown
---
mode: agent
description: For each output column in the 4 final SQL files, walk the table-DAG backward and emit lineage hops.
---
Prerequisite: phase 2 complete.

1. If output/_run_log/columns_to_trace.json does not yet exist:
   a. Read the 4 final-generation files (identified in output/graph/execution_order.txt as the last generation).
   b. Extract every explicitly-named output column.
   c. Save the list to output/_run_log/columns_to_trace.json.

2. Read columns_to_trace.json. For each column, in order:
   a. Check output/lineage.json — if this column already has an entry, skip (resumability).
   b. Invoke `python tools/lineage_walker.py --column <col> --final-table <tbl> --graph-dir output/graph --cache-dir output/cache` from the terminal. The walker handles the static cases.
   c. Read its output. If any hop has `needs_llm_fallback: true`:
      - Look at `fallback_context_lines` to identify the line range to read.
      - Use the filesystem MCP to read ONLY that line range from the source proc, plus 20 lines on either side.
      - Reason about the dynamic SQL block. Reconstruct the most likely transformation.
      - Cite the exact line range from the proc in the `transformation_in_sql` field.
      - Set extraction_method to "llm_inferred".
      - If confidence is low, also append the column+hop to output/_run_log/manual_review.txt for analyst review.
   d. Append the completed column entry to output/lineage.json. Update output/_run_log/progress.json with the column count.

3. Process columns in batches of 20. After each batch, write a checkpoint and assess whether the session is approaching context limits.

4. If approaching the agent context limit, save state and stop. Instruct the user: "Context approaching limit. Open a new session and re-invoke 03-trace-column-lineage.prompt.md to resume at column N." The resume in step 2a will handle the actual continuation.

Important: do NOT load multiple proc files into context simultaneously. Read one at a time. The cache JSONs are summaries — they are sufficient for most hops without re-reading the SQL.
```

### 10.4 `04-render-diagrams.prompt.md` — Phase 4a (Mermaid)

```markdown
---
mode: agent
description: Generate one .mmd file per final-output column from lineage.json.
---
Prerequisite: phase 3 complete.

Invoke `python tools/mermaid_renderer.py --lineage output/lineage.json --out-dir output/diagrams` from the terminal.

Verify by sampling 3 random column diagrams and confirming the node count matches the column's total_hops. Report any discrepancies.

Write checkpoint: phase: 4a, status: complete.
```

### 10.5 `05-export-artifacts.prompt.md` — Phase 4b (Excel exports)

```markdown
---
mode: agent
description: Generate long.xlsx and wide.xlsx from lineage.json.
---
Prerequisite: phase 3 complete.

Invoke `python tools/xlsx_exporter.py --lineage output/lineage.json --out-dir output/` from the terminal.

Verify the two files exist and contain the expected number of rows:
- long.xlsx row count = sum of all columns' total_hops
- wide.xlsx row count = number of columns

Write checkpoint: phase: 4b, status: complete.

The full pipeline is now complete. Summarize the run statistics from output/lineage.json["metadata"] for the user, including:
- Total columns traced
- Total hops emitted
- Distribution by extraction_method (static_parser, dynamic_reconstructed, llm_inferred, abinitio_blackbox)
- Number of hops in manual_review.txt
```

---

## 11. The Resume Strategy

The 291-file corpus and hundreds of columns mean a single Copilot session **will not finish in one go**. The architecture treats this as the default, not the exception.

**Three layers of resumability:**

1. **Phase-level resume.** Each phase writes its outputs to disk before the session ends. The next session starts fresh and reads from disk. The next prompt file checks `progress.json` first to see what phase to enter.

2. **Within-phase resume (Phase 1).** Phase 1 processes 350+ files. If a session ends after 200 of them, `progress.json` records the count and the last filename. The resume skips already-processed files by checking for the existing cache JSON.

3. **Within-phase resume (Phase 3, the most critical).** Phase 3 traces hundreds of columns. The resume is column-level — `lineage.json` is appended to incrementally, and the next session checks each column's presence before processing it. A partially-traced column is treated as not-yet-done; its previous (incomplete) entry is overwritten on resume.

**When a session approaches context limits, Copilot must:**
- Finish the in-progress column (do not commit a partial column to lineage.json)
- Update progress.json with `last_completed_column`
- Print a clear handoff message to the user: *"Context approaching limit. Open a new session and re-invoke 03-trace-column-lineage.prompt.md to resume at column N."*

**Handoff hygiene:** the user opens a fresh chat, types `/03-trace-column-lineage` (the prompt file is invokable as a slash command), and the agent picks up where it left off without further instruction.

---

## 12. Hard Limits and Fallbacks

| Limit | Approximate value | Mitigation |
|---|---|---|
| Agent session context window | varies by selected model | Phase split as above; checkpointing per column in Phase 3 |
| Tools per session | 128 | Use only filesystem + memory + terminal; avoid adding more MCPs than necessary |
| Tool call rate limits | varies by plan | Batch Phase 1 file reads through the terminal Python script, not via individual filesystem MCP reads — one terminal invocation per file is far more efficient |
| MCP tool unavailable in Ask/Edit modes | hard restriction | All work must happen in Agent mode |
| Premium request quota | varies by plan | Phase 3's LLM-fallback hops are the only place a premium model is needed. Run Phases 1, 2, 4, and Phase 3's static cases on the base model. |
| Single-prompt token budget | varies by model | Each prompt file is short; the heavy lifting is in tool invocations, not in the prompt itself |
| File read size for very long procs | implementation-dependent | Never read whole 3000-line procs into the agent's context; use `fallback_context_lines` ranges |

**Fallback when limits are hit:** the same Python tools in `/tools` can be invoked directly from the terminal, outside Agent mode entirely. The agent provides orchestration and reasoning; the determinism is in the scripts. If Copilot becomes unavailable mid-run, a developer can finish the run by hand:

```bash
python tools/teradata_proc_extractor.py <file>      # for any unprocessed files
python tools/dag_builder.py --cache-dir output/cache --out-dir output/graph
python tools/lineage_walker.py --column <col> ...   # for each remaining column
python tools/mermaid_renderer.py --lineage output/lineage.json --out-dir output/diagrams
python tools/xlsx_exporter.py --lineage output/lineage.json --out-dir output/
```

The LLM-fallback hops would then need to be marked `<unknown>` in the lineage.json and added to `manual_review.txt` for human attention.

---

## 13. Cross-Cutting Concerns

### 13.1 Dynamic SQL — The Single Largest Source of Lineage Loss

Three layers of defense, in order:

1. **Layer 1 — Variable propagation** (Phase 1, §3.3). Catches probably 50–70% of dynamic SQL where the string is built from literals + a small number of parameters.
2. **Layer 2 — Copilot LLM fallback** (Phase 3, when `needs_llm_fallback: true`). Catches another large fraction by reasoning about the surrounding proc context.
3. **Layer 3 — Manual review queue.** Any hop with `extraction_method = "llm_inferred"` AND `reconstruction_confidence < 0.6` is added to `output/_run_log/manual_review.txt`. Analysts triage these post-run.

Don't aim for 100% automation. Aim for **transparent confidence labelling** so analysts know exactly which 5–15% of hops to verify.

### 13.2 AbInitio — Designed-In Acknowledgement

The pipeline never treats AbInitio gaps as a bug. They appear as legitimate hops in `long.xlsx` with `ProcType = "AbInitio"` and `ExtractionMethod = "abinitio_blackbox"`. Analysts know to:
- Not expect `InputColumn` values
- Treat this as a "data lineage cliff" — provenance below this point requires AbInitio metadata that lives outside the corpus

If at some future point the AbInitio graphs become parseable (e.g. via the AbInitio metadata environment), the same lineage.json schema accepts the upgrade — only the `extraction_method` field changes from `abinitio_blackbox` to `static_parser`.

### 13.3 Schema Awareness

`sqlglot.lineage()` works much better when given a schema. Build one progressively:
- After Phase 1, every file's `column_mapping` reveals the columns of its `output_table`.
- These are aggregated into `output/graph/inferred_schema.json` during Phase 2.
- Phase 3's column walks pass this schema into `sqlglot.lineage(schema=…)` for higher accuracy.

Schemas are **inferred, not authoritative**. If a downstream file references a column that doesn't appear in the inferred schema, mark it as a discrepancy in the warnings log — it's likely a misnamed reference or a column from a table outside the corpus.

### 13.4 Cross-Generation Bridge Validation

The CV→TF linkage is the spine of Scenario B execution-order inference. After Phase 2, the agent runs a validator (built into `dag_builder.py`):
- For every TF proc, verify that every table in its `input_tables_read` either (a) is the output of a CV proc in an earlier generation, or (b) is an external/source table.
- Any orphan TF inputs are warnings — likely a parser miss or a real ETL bug.

### 13.5 Idempotency

Every phase is idempotent. Re-running Phase 1 on the same corpus produces byte-identical cache JSONs. Re-running Phase 4 on the same lineage.json produces byte-identical xlsx files. This is enforced by:
- Sorted iteration order over filesystem listings
- Deterministic Python dictionary ordering (3.7+)
- No timestamps or random IDs in any output except the `metadata.generated_at` block

### 13.6 Versioning

Pin `sqlglot` to a known-good version in `tools/requirements.txt`. Different sqlglot versions produce slightly different ASTs for the same Teradata input, and that drift breaks lineage walks across runs. A version bump is a deliberate decision, not a passive update.

---

## 14. Validation & Acceptance Criteria

Before declaring the run successful, run these checks (all simple to automate; the agent can do them in the final summary step):

| Check | Expectation |
|---|---|
| Total cache JSONs == 291 + 65 (or actual file count) | Phase 1 covered every file |
| Final generation in execution_order.txt has exactly 4 CV procs | Architecture invariant holds |
| Every column in the 4 final SQL files has an entry in lineage.json | No columns dropped |
| Every hop has all 13 long.xlsx fields populated (or explicit `<unknown>`) | No silent gaps |
| Sum(total_hops over all columns) == long.xlsx row count | Long format consistency |
| Number of distinct columns in long.xlsx == wide.xlsx row count | Wide format consistency |
| `extraction_method` distribution: static_parser % is the single largest bucket | Most lineage is high-confidence |
| `extraction_method = "llm_inferred"` rate < 15% | Indicates parser is doing its job |
| Number of mermaid files == number of columns in lineage.json | Diagram coverage is complete |
| No cycles in file_dag.json | DAG invariant holds |

If any check fails, treat as a hard fail and investigate before delivering artifacts to analysts.

---

## Closing Notes

**Read this section once. Then re-read §3 (the pre-processing layer).** The single biggest implementation pitfall on this kind of project is asking the Copilot agent to *be* the parser. It is not. **`sqlglot` is the parser; Copilot is the orchestrator and the fallback for dynamic SQL.** The line between those two roles is what determines whether you ship a deterministic lineage system in a few weeks or chase intermittent hallucinations for months.

**Copilot Agent Mode is well-suited to this design.** Its strengths are:
- Reading and writing files via filesystem MCP
- Invoking terminal commands and parsing their output
- Reasoning about bounded, specific problems (a single dynamic SQL block, not a 3000-line proc)
- Following multi-step prompt-driven workflows

This blueprint puts each of those strengths in its right place. The deterministic Python tooling layer handles what Copilot would do badly; Copilot handles what no Python tool can do — interpreting ambiguous dynamic SQL, validating that the inferred generation structure makes ETL sense, and orchestrating the whole multi-session run.

**One last word on the prompt files.** They are deliberately short. The temptation is to stuff each one with detail — don't. Detailed parsing rules belong in the path-scoped `.github/instructions/*.instructions.md` files (which activate automatically). The prompt files are workflow scripts. Keep them lean and the agent will follow them faithfully across sessions.
