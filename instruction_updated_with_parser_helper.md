# Alteryx Workflow Report Generator

You are a Linux-executing Copilot agent using Codex 5.2.

Your job is to generate **Alteryx Workflow Report** Word documents (`.docx`) for **one or many** Alteryx workflows (`.yxmd`, optionally `.yxmc`) using:
- The provided Word template: `Alteryx_Workflow_Report_Template_Standard.docx`
- Workflow XML Parser outputs (preferred): W/T/C exported tables **or** the raw workflow XML inside `.yxmd`
- Mermaid diagrams rendered to **PNG** (using mermaid-cli), embedded into the Word report

This runbook is written to be executed end-to-end by an agent: it includes environment setup, parsing logic, diagram rendering, and Word output generation.

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

### 1.3 Parser field dictionary (highly recommended)
Add this file as context so the agent knows **exactly** what each W/T/C column means and which tools it applies to:

- `input/context/DesignerToolsParse_Workflow_XML_Parser_Tool.txt`

This “field dictionary” is used to:
- map parser columns to report sections (Inputs/Outputs/Rules/Dependencies/etc.)
- identify which columns contain file paths, expressions, scripts, macros, join configs, and connections
- avoid guessing when a workflow tool is not parsed in a simple format

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
mkdir -p input/template input/workflows input/parser_outputs input/context
mkdir -p output/reports output/diagrams output/logs
mkdir -p scripts
```

Place the template at:
- `input/template/Alteryx_Workflow_Report_Template_Standard.docx`

Place parser field dictionary at:
- `input/context/DesignerToolsParse_Workflow_XML_Parser_Tool.txt`

Place workflows at:
- `input/workflows/*.yxmd`
- `input/workflows/*.yxmc` (optional)

---

## 5. Parsing strategy

### Rule 1: Prefer W/T/C tables if available
If `W/T/C` tables exist for a workflow, use them as the primary source of truth.
If not, parse `.yxmd` XML directly.

### Rule 2: Use the parser field dictionary for mapping
When parsing W/T/C, consult the dictionary file to understand:
- which fields belong to **workflow**, **tool**, or **connection** outputs
- which fields contain **expressions**, **file paths**, **scripts**, **macro paths**, and **join config**

---

## 6. Using W/T/C tables

### 6.1 W table (workflow-level)
Use for:
- Workflow Overview:
  - Designer version: `Workflow_yxmdVer`
  - AMP enabled: `Workflow_RunE2`
  - MetaInfo fields: name/description/author/company
- Operational Notes:
  - memory settings, record limits, temp folder settings, and runtime flags

### 6.2 T table (tool-level)
Use for:
- Tool Inventory: `ToolID`, `DesignerToolName`, `DesignerToolCategory`, `Origin_Tools`, `Destination_Tools`, annotations
- Inputs and Outputs:
  - file/db columns like `File_File`, `File_Table`, `File_PreSQL`, `File_PostSQL`, formats, limits
- Business Rules:
  - expressions like `Expression`, filter fields, formula fields, and parsed join settings
- Dependencies:
  - file paths (`File_File`)
  - macro paths (`EngineSettings_Macro`)
  - scripts (`Python_productionModeScript`, `RScript`)
  - any additional dependency xml (`Dependencies_OuterXML`)
- When a tool’s config is not directly parsed: fall back to `Configuration_OuterXML`

### 6.3 C table (connection-level)
Use for:
- Workflow Diagram and graph ordering
  - `Origin_ToolID`, `Origin_Connection`, `Destination_ToolID`, `Destination_Connection`, `Wireless`

---

## 7. Fallback parsing from `.yxmd` XML

If W/T/C are not available, parse `.yxmd` XML directly:
- Nodes: `//Nodes/Node[@ToolID]`
- Connections: `//Connections/Connection/Origin` and `Destination`
- MetaInfo: `//Properties/MetaInfo/*`
- Tool config: `//Node/Properties/Configuration`
- Schema when available: `//Properties/MetaInfo[@connection='Output']/RecordInfo/Field`

---

## 8. Mermaid generation and PNG embedding

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

4) Embed the PNG into the Word report:
- Replace `{{DiagramPNG}}` placeholder with the PNG image (width ~6.5–7 inches)
- Replace `{{MermaidDiagramSource}}` with the Mermaid source text (for wiki copy/paste)

---

## 9. Word report generation rules

### 9.1 Use the provided template as the base
Open the template and populate placeholders and tables.

Do not change headings. Keep them exactly as in the template.

### 9.2 Replace template placeholders
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

### 9.3 Populate the existing tables
The template contains pre-built tables with placeholder rows. For each table:
- Keep the header row
- Remove the placeholder row (the row containing `{{...}}`)
- Append the real rows extracted from W/T/C or `.yxmd`

Tables to populate:
- Inputs and Input schema summary
- Outputs and Output schema summary
- Tool Inventory
- Business Rules
- Exceptions and Controls
- Dependencies
  - Data dependencies
  - Macro and asset dependencies
  - External connections and credentials
- Operational Notes
- Validation and Reconciliation

### 9.4 Missing information rule
If a value cannot be extracted from W/T/C or `.yxmd` XML:
- fill it with `Not defined`

Do not invent business owners, SLAs, schedules, or volumes.

---

## 10. Extraction rules by section

### 10.1 Purpose
If W contains a description field (MetaInfo description), use it as the base.
Otherwise:
- generate a short 3–5 sentence summary from main inputs + main transforms + outputs
If you cannot infer confidently, set to `Not defined`.

### 10.2 Inputs
Inputs come from:
- input tools and file/db configs (`File_File`, `File_Table`, `File_PreSQL`, `File_PostSQL`)
- schema from RecordInfo if available

### 10.3 Outputs
Outputs come from:
- output tools and file/db configs (output file paths, output table names, write mode if available)

### 10.4 Tool Inventory
Prefer tool naming from T:
- `DesignerToolName`, `DesignerToolCategory`
Fallback:
- infer from plugin id or configuration patterns

### 10.5 Business Rules
Include at minimum:
- Rule type
- Plain meaning
- Exact logic
- Location (ToolID)
- Impact if changed (generic, factual)

Rule-bearing tools:
- Formula, Filter, Multi-Row Formula, Multi-Field Formula, Join, Summarize, Sort, Record ID

### 10.6 Exceptions and Controls
Exceptions come from:
- explicit mismatch logic (e.g., validation filter)
- filters outputting rejects/errors
- browse outputs labelled/annotated as exceptions

If none:
- write one row stating exceptions are `Not defined`

### 10.7 Dependencies
Dependencies include:
- file paths and db tables
- macro calls
- python/r scripts
- external connections (without exposing secrets)

### 10.8 Operational Notes
Populate from W runtime settings when available:
- memory limits, record limits, temp files, AMP enabled
Add monitoring/escalation fields as `Not defined` if missing.

### 10.9 Validation and Reconciliation
If the workflow contains explicit checks, document them.
If not, add up to two suggested checks and label them “Suggested”.

---

## 11. Implementation requirement

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

## 12. Success criteria

For each `.yxmd` file:
- A `.docx` report exists in `output/reports`
- A `.mmd` and `.png` diagram exist in `output/diagrams`
- The `.docx` contains an embedded diagram image (not just placeholder text)
- All major sections are populated (or explicitly `Not defined`)
- Logs exist in `output/logs`

---

## 13. Safety and privacy rules
- Do not print secrets in logs or reports.
- Do not embed passwords/tokens.
- If paths reveal sensitive internal shares, keep them as-is unless the implementation adds a masking flag.
