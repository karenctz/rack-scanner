# Rack Survey — Power Apps Build Handoff

**Platform:** Power Apps canvas app  
**Storage:** Microsoft Dataverse  
**Photos:** Dataverse Notes / Attachments  

---

## 1. Business problem

Datacenter engineers currently survey server racks using a phone camera and a clipboard. Photos get detached from rack/slot context, serial numbers are transcribed manually from photos into Excel, and there is no live visibility into survey progress. A typical 20-rack survey takes a full day to process.

This app replaces that workflow end-to-end: structured field capture on a phone, AI-assisted OCR to extract serial and part numbers from photos, and a single consolidated Dataverse record set accessible to the whole team.

---

## 2. Dataverse schema

Create two tables. Enable **Notes** on both (table Settings → General → Enable Attachments).

### Table: Rack Surveys

| Display name | Type | Required | Notes |
|---|---|---|---|
| Name | Text (primary) | Yes | Rack ID e.g. R01, A-02-L |
| Location | Single line of text | No | e.g. "Level 2 Aisle B" |
| Notes | Single line of text | No | |
| Status | Choice | Yes | Pending (default), In Progress, Complete |

Front and back rack photos stored as Note attachments with subject = "Front" or "Back".

### Table: Equipment Items

| Display name | Type | Required | Notes |
|---|---|---|---|
| Name | Text (primary, auto) | Yes | |
| Rack Survey | Lookup → Rack Surveys | Yes | |
| Start U | Whole number | Yes | Rack unit position top |
| End U | Whole number | No | Omit if 1U device |
| Category | Choice | No | Server, Network Switch, UPS, Storage, PDU, KVM, Other |
| Serial Number | Single line of text | No | Populated by OCR |
| Part Number | Single line of text | No | Populated by OCR |
| Manufacturer | Single line of text | No | |
| Notes | Multiline text | No | |
| OCR Status | Choice | No | Pending (default), Scanned, Error |

Equipment and serial label photos stored as Note attachments with subject = "Equipment" or "Serial".

---

## 3. App screens (5 total)

### Screen 1 — Home ✅ YAML ready

Rack survey list, sorted newest first. Status badge per row (Pending / In Progress / Complete).

- **+ New** button → creates new Rack Survey record, navigates to Screen 2
- **Tap row** → navigates to Screen 2 with existing record
- Empty state message when no surveys exist

### Screen 2 — Rack Detail 🔲 To build

View/edit a single rack survey. Create or edit mode driven by global variable `gblEditMode`.

- Fields: Rack ID, Location, Notes, Status
- Two photo attachment slots: Front of Rack, Back of Rack (via Notes Attachments control)
- Equipment item list (gallery filtered to this rack)
- Add Equipment button → Screen 3
- Save / Back buttons

### Screen 3 — Equipment Item 🔲 To build

Add or edit a single equipment item within the current rack.

- Fields: Start U, End U, Category, Serial Number, Part Number, Manufacturer, Notes
- Photo attachment slot (equipment label / serial tag) via Notes
- Scan button → triggers AI Builder Text Recognizer on attached photo, populates Serial Number and Part Number
- Save / Delete / Back

### Screen 4 — Scan / OCR 🔲 To build

Batch OCR screen. Processes all Equipment Items in a rack where OCR Status = Pending.

- Progress indicator per item
- Uses AI Builder Text Recognizer (predict action on note attachment)
- Writes result back to Serial Number / Part Number columns
- Sets OCR Status to Scanned or Error

### Screen 5 — Review / Export 🔲 To build

Editable table of all equipment items across all racks. Final review before export.

- Filterable by rack, status, category
- Inline edit for any cell
- Export button → triggers Power Automate flow that builds Excel and saves to SharePoint or emails to requestor

---

## 4. Power Automate flow (Excel export)

| Step | Connector | Action |
|---|---|---|
| 1 | Power Apps | Triggered from app (passes filter parameters) |
| 2 | Dataverse | List Equipment Items rows (filtered) |
| 3 | Excel Online | Create table from dynamic content |
| 4 | SharePoint / Email | Save file or email to requestor |

---

## 5. Navigation and global variables

| Variable | Type | Set by | Used by |
|---|---|---|---|
| gblEditMode | Boolean | Screen 1 buttons | Screen 2 (edit vs read-only mode) |
| gblRackRecord | Record | Screen 1 buttons | Screens 2, 3, 4 |
| gblItemRecord | Record | Screen 2 add/select | Screen 3 |

---

## 6. AI Builder setup

Use the built-in **Text recognizer** model — no training required. Add it via Insert → AI Builder → Text recognizer.

On Screen 3, the Scan button formula:

```
Set(gblOcrResult, 'Text recognizer'.Predict(galleryPhotos.Selected.Value));
Patch('Equipment Items', gblItemRecord,
  {'Serial Number': First(Filter(gblOcrResult.results, StartsWith(text, "S/N"))).text}
)
```

> **Note:** The OCR extraction logic will need tuning per label format. Start with the raw text output visible in a label, then build the filter/parse logic around the patterns you see on your equipment.

---

## 7. Licensing requirements

| Feature | Licence needed |
|---|---|
| Canvas app + Dataverse | Power Apps Premium (per user or per app) |
| AI Builder Text Recognizer | AI Builder credits (included in some M365 / Power Platform plans) |
| Power Automate flow | Standard — included with Power Apps Premium |

---

## 8. Screen 1 YAML (ready to paste)

In Power Apps Studio, create a blank screen named `HomeScreen`, click on the canvas, press **Ctrl+Alt+V**, and paste the contents of `HomeScreen.yaml` (in this folder).

> If the paste fails with a schema error, copy any existing control from the canvas (Ctrl+C), paste into Notepad to see the exact format Power Apps expects, and reformat the YAML to match.

---

*Prepared by: karen.yeung@cactoz.com*
