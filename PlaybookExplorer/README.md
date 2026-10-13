## PlaybookExplorer

`Playbook Explorer` agent: a read-only assistant that answers questions about the tenant's playbooks - which exist, what they do step by step, what they use, who changed them - and writes reports into an issue or case war room in two steps (create, then upload). Files default to formats that open as one page - HTML reports, flowcharts as an HTML page plus the `.mmd` Mermaid source, SVG charts, CSV rows; PDF or DOCX only when the analyst asks for them by name (PDF splits wide flowcharts and long reports across pages). Built with the `xsiam-agent-sheet-builder` skill.

| File | Contents |
|---|---|
| `PlaybookExplorerAgent.xlsx` | Agent (description, guidelines, starters), 4 custom actions with inputs, outputs and source, build checks |
| `PlaybookExplorerAgent_setup.html` | Step-by-step XSIAM setup guide: prerequisites and instance-name checks, script import and chromium image warm-up, agent, actions, attach, test prompts, troubleshooting. Open it in a browser and follow it |

| Custom action | Wraps |
|---|---|
| Cortex - Search Playbooks | `SearchTenantPlaybooks` |
| Cortex - Find Playbook Tasks | `PlaybookTaskFinder` - task rows as context |
| Cortex - Create File | `CreateFile` - uses `GeneratePDF` for PDF; returns `CreateFile.FileID` |
| Cortex - Upload File To War Room | `UploadFileToWarRoom` - a tenant duplicate of OOTB `UploadFile` |

Each action description is a full prompt: argument recipes that map a question to the exact script arguments (job playbooks -> `Tags=job,jobs,operational`, "what does X do" -> `Name=X NameMatch=exact TaskDetail=flow`, ...), how to read the result, and what to do on no match or error. Create File tells the agent how to write the report (HTML structure and style, SVG for charts, JSON rows for tables).

Difference from ContentSearch: ContentSearch's `SearchPlaybooks` action has the agent write the `/playbook/search` query itself. Playbook Explorer moves the filtering into `SearchTenantPlaybooks` (task-level filters, task flow, trimmed output), so the agent only picks arguments.

### How files reach the analyst

Two actions, always in this order:

1. **Cortex - Create File** (`CreateFile`): the agent writes the content (HTML, Markdown, CSV, JSON, SVG, or a Mermaid / draw.io / Excalidraw diagram); the script builds the PDF / DOCX / file in the chat run and returns `CreateFile.FileID`, `CreateFile.FileName`, `CreateFile.Format`.
2. **Cortex - Upload File To War Room** (`UploadFileToWarRoom`, a duplicate of OOTB `UploadFile`): `entryID` = that `CreateFile.FileID`, `incID` = the issue (alert) ID as given, or `INCIDENT-<n>` for a case (incident), `body` = `{"fileName": "<name.ext>", "fileComment": "<note>"}` - without it the file has no name or extension in the war room.

Why two: XSIAM does not let one script both create a file and upload it, and OOTB scripts such as `UploadFile` cannot be registered as actions - only a tenant duplicate can. Files cannot go to a playground: it only accepts writes from its owner, and the Core REST API key is a different user.

Requires the **Core REST API** integration (and the `UploadFile` script it ships, duplicated as `UploadFileToWarRoom`) and four automations imported from the Automations repo: `SearchTenantPlaybooks`, `PlaybookTaskFinder`, `CreateFile`, and `GeneratePDF` (on `demisto/chromium`, used by Create File for PDFs). After importing, replace the script ids in the spec and sheet with the ones the tenant assigns.
