# Copilot Agent Instructions
## Alteryx Workflow Report Generator

You are a Linux-executing Copilot agent using Codex 5.2.

Your job is to generate **Alteryx Workflow Report** Word documents (`.docx`) for **one or many** Alteryx workflows (`.yxmd`, optionally `.yxmc`) using:
- The provided Word template: `Alteryx_Workflow_Report_Template_Standard.docx`
- Workflow XML Parser outputs (preferred): W/T/C exported tables **or** the raw workflow XML inside `.yxmd`
- Mermaid diagrams rendered to **PNG** (using mermaid-cli), embedded into the Word report

---

## 1. Inputs

### 1.1 Required inputs
- `input/template/Alteryx_Workflow_Report_Template_Standard.docx`
- `input/workflows/*.yxmd` (1 to 100+ files)

### 1.2 Optional inputs
- `input/workflows/*.yxmc` (macros referenced by `.yxmd`)
- `input/parser_outputs/<workflow_stem>/W.csv|W.tsv`
- `input/parser_outputs/<workflow_stem>/T.csv|T.tsv`
- `input/parser_outputs/<workflow_stem>/C.csv|C.tsv`

> If W/T/C tables are provided, **prefer them** for tool names and parsed configuration fields.  
> If W/T/C are not provided, parse `.yxmd` XML directly.

---

## 2. Output requirements

For each workflow `<workflow_stem>.yxmd`, produce:
- `output/reports/<workflow_stem>_Alteryx_Workflow_Report.docx`
- `output/diagrams/<workflow_stem>.mmd` (Mermaid source)
- `output/diagrams/<workflow_stem>.png` (Mermaid rendered PNG)
- `output/logs/<workflow_stem>.log`

The report must follow the template’s standard headings:
- Purpose
- Workflow Overview
- Inputs
- Outputs
- Workflow Diagram
- Tool Inventory
- Business Rules
- Exceptions and Controls
- Dependencies
- Operational Notes
- Validation and Reconciliation
- Appendix

---

## 3. Environment setup on Linux

### 3.1 Python virtualenv
Use Python 3.12 exactly:

```bash
python3.12 -m venv venv
source venv/bin/activate
python -m pip install --upgrade pip
pip install python-docx lxml pandas networkx pillow
```

### 3.2 Node.js and Mermaid CLI
Mermaid diagram must be rendered to PNG using **mermaid-cli** via npx with the required cache path:

```bash
export PUPPETEER_SKIP_DOWNLOAD=1
export PUPPETEER_EXECUTABLE_PATH=$(which chromium || which chromium-browser)
npx --cache=/dev/shm/.npm70 -y @mermaid-js/mermaid-cli@latest -i input.mmd -o output.png --backgroundColor transparent
```

#### Chromium dependency
If Chromium is missing, install it (choose the right command for your OS):
- Debian/Ubuntu:
```bash
sudo apt-get update
sudo apt-get install -y chromium-browser || sudo apt-get install -y chromium
```
- RHEL/Fedora:
```bash
sudo dnf install -y chromium || sudo yum install -y chromium
```

---

## 4. Workspace layout (must be created)

```bash
mkdir -p input/template input/workflows input/parser_outputs
mkdir -p output/reports output/diagrams output/logs
mkdir -p scripts
```

Place the template at:
- `input/template/Alteryx_Workflow_Report_Template_Standard.docx`

Place workflows at:
- `input/workflows/*.yxmd`
- `input/workflows/*.yxmc` (optional)

---

## 5. Parsing strategy (W/T/C preferred)

### 5.1 If W/T/C tables exist
Use them as the source of truth:

**W table (workflow-level metadata)**
- Populate `Workflow Overview` and `Operational Notes`:
  - Designer version: `Workflow_yxmdVer`
  - AMP enabled: `Workflow_RunE2`
  - Runtime limits and settings (memory, temp folder, record limits, etc.) if present

**T table (tool-level metadata and configurations)**
- Build:
  - Inputs / Outputs: from file fields such as `File_File`, `File_Table`, `File_PreSQL`, `File_PostSQL` (when present)
  - Tool Inventory: `ToolID`, `DesignerToolName`, `DesignerToolCategory`, `Origin_Tools`, `Destination_Tools`, annotation text
  - Business Rules: expressions and config fields, including:
    - `Expression` (formula/filter/multi-row formula etc.)
    - Join details (from parsed fields or `Configuration_OuterXML`)
  - Dependencies:
    - `File_File` and DB table references
    - Macro usage: `EngineSettings_Macro`
    - Scripts: `Python_productionModeScript`, `RScript`
    - Additional dependency XML: `Dependencies_OuterXML`

**C table (connections/graph)**
- Build the workflow graph for Mermaid:
  - `Origin_ToolID`, `Origin_Connection`, `Destination_ToolID`, `Destination_Connection`, `Wireless`

### 5.2 If W/T/C not available
Parse `.yxmd` XML directly:
- Nodes: `//Nodes/Node[@ToolID]`
- Connections: `//Connections/Connection/Origin` and `Destination`
- MetaInfo: `//Properties/MetaInfo/*`
- Tool config: `//Node/Properties/Configuration`
- Schema when available: `//Properties/MetaInfo[@connection='Output']/RecordInfo/Field`

---

## 6. Mermaid generation and PNG embedding

For each workflow:
1) Construct Mermaid source from the connection graph:
   - Use `flowchart LR`
   - Create a node per tool: `T<id>["<id>: <tool name>"]`
   - Create edges: `T1 --> T2`
   - Keep labels short; include annotation text only if it is short (truncate ~50 chars)

2) Write Mermaid to:
- `output/diagrams/<workflow_stem>.mmd`

3) Render to PNG:
```bash
export PUPPETEER_SKIP_DOWNLOAD=1
export PUPPETEER_EXECUTABLE_PATH=$(which chromium || which chromium-browser)

npx --cache=/dev/shm/.npm70 -y @mermaid-js/mermaid-cli@latest \
  -i output/diagrams/<workflow_stem>.mmd \
  -o output/diagrams/<workflow_stem>.png \
  --backgroundColor transparent
```

4) Embed the PNG into the Word report at the template placeholder:
- Replace `{{DiagramPNG}}` paragraph text with the PNG image (width ~6.5–7 inches)
- Also set `{{MermaidDiagramSource}}` to the Mermaid text so it can be copied to a wiki

---

## 7. Word report generation rules

### 7.1 Use the provided template as the base
Open the template, then populate its placeholders and tables.

**Do not change headings**. Keep them exactly as in the template.

### 7.2 Replace template placeholders
Replace these placeholders (write `Not defined` if missing):
- `{{PurposeStatement}}`
- `{{WorkflowName}}`
- `{{FileName}}`
- `{{BusinessOwner}}`
- `{{TechnicalOwner}}`
- `{{BusinessDomain}}`
- `{{DecisionOrAction}}`
- `{{Frequency}}`
- `{{Trigger}}`
- `{{Volume}}`
- `{{LastModified}}`
- `{{Workflow_yxmdVer}}`
- `{{Workflow_RunE2}}`
- `{{MermaidDiagramSource}}`

### 7.3 Populate the existing tables (preferred approach)
The template contains pre-built tables with placeholder rows. For each table:
- Keep the header row
- Remove the single placeholder row (the row containing `{{...}}`)
- Append as many rows as needed for the workflow

Tables to populate:
- Inputs (plus Input schema summary)
- Outputs (plus Output schema summary)
- Tool Inventory
- Business Rules
- Exceptions and Controls
- Dependencies
  - Data dependencies
  - Macro and asset dependencies
  - External connections and credentials
- Operational Notes (key/value table)
- Validation and Reconciliation

### 7.4 What counts as “Not defined”
If a value cannot be extracted from W/T/C or `.yxmd` XML:
- Fill it with: `Not defined`

Do not invent business owners, SLAs, schedules, or volumes.

---

## 8. Extraction rules by section

### 8.1 Purpose
If W contains `MetaInfo_Description`, use it as the base of Purpose statement.
Otherwise:
- Create a short 3–5 sentence summary derived from:
  - main input sources
  - main transforms (rules-bearing tools)
  - main outputs and exception outputs

If you cannot infer confidently, set to `Not defined`.

### 8.2 Inputs
Populate Inputs from:
- Input tools (Input Data, Dynamic Input, Macro Input, In-DB input tools)
- Fields from T (preferred):
  - `File_File`, `File_Table`, `File_PreSQL`, `File_PostSQL`, record limits, etc.
- Schema:
  - Use `RecordInfo` fields if available; otherwise `Not defined`

### 8.3 Outputs
Populate Outputs from:
- Output tools (Output Data, Browse, In-DB output tools)
- Fields from T:
  - output file paths, table names, output format, max records, write mode if available

### 8.4 Tool Inventory
Use T fields where available:
- `ToolID`, `DesignerToolName`, `DesignerToolCategory`, `Origin_Tools`, `Destination_Tools`, `AnnotationText`/`DefaultAnnotationText`

If only XML exists, infer the tool name from plugin id and config.

### 8.5 Business Rules
Rules must include:
- Plain-language meaning
- Exact logic
- Where in workflow
- Impact if changed

Populate rule-bearing tools:
- Formula, Filter, Multi-Row Formula, Multi-Field Formula, Join, Summarize, Sort, Record ID
Use:
- `Expression` for formula/filter expressions
- join keys and selection logic from `Configuration_OuterXML` if join isn’t parsed cleanly

Impact if changed must be written generically (no guessing data):
- “May change downstream results”
- “May increase/decrease exceptions”
- “May change row counts”
Keep it factual and general.

### 8.6 Exceptions and Controls
Identify exceptions from:
- Filter tools where True/False output semantics represent errors/rejects/exceptions
- Browse tools named/annotated as exceptions
- Comparison-style rules (RecordID mismatch, reconciliation joins, etc.)

If no explicit exceptions:
- Create one row with `Exception name = None` and `Definition = Not defined`

### 8.7 Dependencies
Capture:
- File paths and DB tables from File_* fields
- Macro usage from `EngineSettings_Macro`
- Script dependencies from Python/R script fields
- Any `Dependencies_OuterXML` contents as supporting evidence

Populate:
- Data dependencies table
- Macro and asset dependencies table
- External connections and credentials table (do not expose secrets; note secret location only)

### 8.8 Operational Notes
Populate from W fields when available:
- memory limits, record limits, temp files
- AMP enabled
Also include:
- Failure impact (if not known: `Not defined`)
- Monitoring / alerts (if not known: `Not defined`)
- Escalation contact (if not known: `Not defined`)

### 8.9 Validation and Reconciliation
If the workflow has explicit validation logic (e.g., mismatch filter), document it.
If not, add up to **two suggested checks** but clearly label them as “Suggested”:
- Suggested row count check
- Suggested key uniqueness check

---

## 9. Implementation requirements

Create and run:
- `scripts/generate_reports.py`

It must accept:
- `--template input/template/Alteryx_Workflow_Report_Template_Standard.docx`
- `--workflows input/workflows`
- `--parser_outputs input/parser_outputs` (optional)
- `--out output/reports`

Run command:
```bash
python scripts/generate_reports.py \
  --template input/template/Alteryx_Workflow_Report_Template_Standard.docx \
  --workflows input/workflows \
  --parser_outputs input/parser_outputs \
  --out output/reports
```

---

## 10. Success criteria

For each `.yxmd` file:
- A `.docx` report exists in `output/reports`
- A `.mmd` and `.png` diagram exist in `output/diagrams`
- The `.docx` contains an embedded diagram image (not just placeholder text)
- All major sections are populated (or explicitly say `Not defined`)
- Logs exist in `output/logs`

---

## 11. Safety and privacy rules
- Do not print secrets in logs or reports.
- Do not embed passwords/tokens.
- If a path reveals sensitive internal shares, keep it but allow optional masking via a flag in the script.

