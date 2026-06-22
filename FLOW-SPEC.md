# Power Automate Flow Spec — Rack Scanner AI Builder

## Overview

Two flows are required:

| Flow | Trigger | Purpose |
|---|---|---|
| **OCR Flow** | HTTP POST (from app) | Receive a photo, run AI Builder Text Recognizer, return raw OCR text |
| **Export Flow** | Power Apps button | Query Dataverse, build Excel, email or save to SharePoint |

---

## Flow 1 — OCR (AI Builder Text Recognizer)

### Trigger
**When an HTTP request is received**
- Method: POST
- Request body schema: none (body is raw base64 image string, Content-Type: text/plain)
- Authentication: none (URL is secret — keep it private)

### Steps

**Step 1 — Parse the body**
- Action: `Initialize variable`
- Name: `ImageBase64`
- Type: String
- Value: `@{triggerBody()}` (the raw POST body)

**Step 2 — Run AI Builder Text Recognizer**
- Action: `AI Builder → Extract information from documents` (Text recognition model)
- Document type: Image
- Document content: `@{variables('ImageBase64')}` (base64)

> If the AI Builder action requires a file rather than base64, add a Compose step first:
> - Action: `Compose`
> - Input: `@{base64ToBinary(variables('ImageBase64'))}`
> Then pass the Compose output as the document.

**Step 3 — Build the response text**
- Action: `Compose`
- Name: `OcrText`
- Input: Concatenate all recognized text lines from the AI Builder output:
  `@{join(body('Recognize_Text')?['pages']?[0]?['lines'], ' ')}`

> Adjust the path based on the AI Builder Text Recognizer output schema. The goal is a single plain-text string of all recognised text joined with spaces.

**Step 4 — Respond**
- Action: `Response`
- Status code: 200
- Headers: `Content-Type: application/json`
- Body:
```json
{
  "text": "@{outputs('OcrText')}"
}
```

### Error handling
Wrap Steps 2–4 in a **Scope** with a **Configure run after** set to handle failures:
- On failure: Response with status 500, body `{"error": "OCR failed"}`

### Settings
- Timeout: 30 seconds (AI Builder can be slow on first call)
- Concurrency: 10 (allows parallel photo processing from the desktop app)

---

## Flow 2 — Excel Export

### Trigger
**Power Apps (V2)**
- Inputs passed from the app:
  - `RackFilter` (string) — rack ID to export, or "ALL"
  - `RecipientEmail` (string) — email address to send the file to

### Steps

**Step 1 — List Equipment Items from Dataverse**
- Action: `Dataverse → List rows`
- Table: Equipment Items
- Filter rows (OData):
  - If RackFilter = "ALL": *(no filter)*
  - If RackFilter ≠ "ALL": `_cr_racksurveylookup_value eq '<RackFilter>'`
- Expand query: `cr_RackSurvey($select=cr_name,cr_location)`
- Select columns: `cr_name, cr_startu, cr_endu, cr_category, cr_manufacturer, cr_serialnumber, cr_partnumber, cr_notes, cr_ocrstatus`
- Order by: `cr_RackSurvey/cr_name asc, cr_startu asc`

**Step 2 — Create Excel file in SharePoint**
- Action: `SharePoint → Create file`
- Site: *(your SharePoint site)*
- Folder path: `/Shared Documents/Rack Surveys/Exports`
- File name: `rack-assets-@{formatDateTime(utcNow(), 'yyyy-MM-dd')}.xlsx`
- File content: *(see Step 3)*

**Step 3 — Build Excel content**

Use the **Create CSV table** action as an intermediate step, then convert to Excel using the Office Scripts action, OR use the simpler approach:

- Action: `Excel Online (Business) → Run script`
- Script: Create a new workbook via Office Script, write all rows, return the file bytes

> Simpler alternative if Office Scripts is not available:
> - Action: `Create CSV table` from the Dataverse list rows output
> - Save the CSV to SharePoint instead of Excel
> - Columns: Rack ID, Location, Start U, End U, Category, Manufacturer, Part Number, Serial Number, Notes, OCR Status

**Step 4 — Send email**
- Action: `Office 365 Outlook → Send an email (V2)`
- To: `@{triggerBody()?['RecipientEmail']}`
- Subject: `Rack Survey Export — @{formatDateTime(utcNow(), 'dd MMM yyyy')}`
- Body:
  ```
  Your rack survey export is attached.

  Racks included: @{if(equals(triggerBody()?['RackFilter'], 'ALL'), 'All racks', triggerBody()?['RackFilter'])}
  Items exported: @{length(body('List_rows')?['value'])}
  Generated: @{formatDateTime(utcNow(), 'dd MMM yyyy HH:mm')} UTC
  ```
- Attachments: the Excel file from Step 2/3

**Step 5 — Respond to Power Apps**
- Action: `Respond to a Power Apps or flow (V2)`
- Outputs:
  - `Status` (string): "Success"
  - `ItemCount` (integer): `@{length(body('List_rows')?['value'])}`
  - `FileUrl` (string): SharePoint file URL

---

## Column name reference

When the Dataverse tables are created, the logical column names will have a publisher prefix (e.g. `cr_`). Replace `cr_` below with your actual prefix.

| Display name | Logical name (example) |
|---|---|
| Name (Rack ID) | `cr_name` |
| Location | `cr_location` |
| Status | `cr_status` |
| Start U | `cr_startu` |
| End U | `cr_endu` |
| Category | `cr_category` |
| Manufacturer | `cr_manufacturer` |
| Serial Number | `cr_serialnumber` |
| Part Number | `cr_partnumber` |
| Notes | `cr_notes` |
| OCR Status | `cr_ocrstatus` |
| Rack Survey (lookup) | `_cr_racksurveylookup_value` |

To find your actual prefix: open the table in make.powerapps.com → Columns → click any custom column → look at the "Name" field.

---

## Testing the OCR flow

Once published, test with curl or Postman:

```bash
# Encode a test image
base64 -i test-label.jpg -o test-label.b64

# POST to flow URL
curl -X POST "https://prod-xx.logic.azure.com/..." \
  -H "Content-Type: text/plain" \
  --data-binary @test-label.b64
```

Expected response:
```json
{ "text": "HPE ProLiant DL380 Gen10 S/N CZJ1234567 P/N 867960-B21" }
```

---

## Notes

- The OCR flow URL is the only secret in this system. Treat it like a password — share it only with engineers running the desktop scanner.
- AI Builder Text Recognizer consumes AI Builder credits. Each image = 1 credit. Check your tenant's AI Builder credit allocation before a large batch run.
- The export flow can be extended to write directly to a SharePoint list or a Teams channel instead of email.
