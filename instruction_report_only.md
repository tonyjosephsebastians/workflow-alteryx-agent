# Copilot Agent Instructions
## Alteryx Workflow Report Generation

You are a Linux-executing Copilot agent using Codex 5.2.

Objective:
Generate **final Alteryx Workflow Reports** as `.docx` files. The output must read like a finished report, not a template.
Do not include template guidance text such as “Write 3–5 sentences…” or “Diagram image placeholder…”.

Inputs:
- `Alteryx_Workflow_Report_Template_Standard.docx` as a structural reference and as a base for styles and headings.
- W/T/C parser outputs when available.
- `.yxmd` workflow files (and optional `.yxmc`).

Output:
For each workflow, produce:
- `<workflow_stem>_Alteryx_Workflow_Report.docx`

Critical rules:
1) Populate only values that exist in W/T/C or workflow XML.
2) If a field is missing, leave it blank.
3) Remove all placeholders (`{{...}}`) from the final report.
4) Remove all template instruction sentences from the final report.
5) Headings must remain the standard report headings:
   Purpose, Workflow Overview, Inputs, Outputs, Workflow Diagram, Tool Inventory, Business Rules, Exceptions and Controls, Dependencies, Operational Notes, Validation and Reconciliation, Appendix.

Diagram requirement:
- Build Mermaid from connections.
- Render Mermaid to PNG using:
  `npx --cache=/dev/shm/.npm70 -y @mermaid-js/mermaid-cli@latest ...`
- Embed PNG into the report under “Workflow Diagram”.
- Mermaid source should be included in Appendix.

Implementation approach:
- Recommended: generate a fresh Word document using python-docx with the headings above.
  Do not copy the template’s guide paragraphs.
- Alternative: start from the template but programmatically delete:
  - paragraphs containing “Write”, “List every”, “Describe”, “Embed”, “Diagram image placeholder”, “Mermaid rendering using mermaid-cli”
  - any paragraph containing `{{...}}` tokens after replacement

Environment:
- Python venv: `python3.12 -m venv venv`
- pip: `python-docx lxml pandas`
- Mermaid: `npx --cache=/dev/shm/.npm70 ...`
- Chromium: set `PUPPETEER_EXECUTABLE_PATH`

Success checks per report:
- No template instructions appear.
- No `{{...}}` placeholders remain.
- Diagram PNG is embedded.
- Tables have headers and only real rows.
