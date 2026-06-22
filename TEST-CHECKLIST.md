# Rack Survey Power App — Test Checklist

Run through these before signing off on each build phase. Tick each item in order — later items depend on earlier ones passing.

---

## Phase 1 — Dataverse setup

- [ ] Rack Surveys table exists in Dataverse
- [ ] Equipment Items table exists with lookup to Rack Surveys
- [ ] Status choice column has exactly: Pending, In Progress, Complete (Pending = default)
- [ ] Category choice column has exactly: Server, Network Switch, UPS, Storage, PDU, KVM, Other
- [ ] OCR Status choice column has exactly: Pending, Scanned, Error (Pending = default)
- [ ] Notes is enabled on Rack Surveys table (Settings → General → Enable Attachments)
- [ ] Notes is enabled on Equipment Items table
- [ ] Create a test Rack Survey record manually in Dataverse — confirm it saves and appears in the table view
- [ ] Create a test Equipment Item linked to that rack — confirm the lookup resolves correctly

---

## Phase 2 — OCR Flow (Flow 1)

- [ ] Flow is saved and turned ON in Power Automate
- [ ] HTTP trigger URL is copied and saved securely
- [ ] Test with a real equipment label photo using curl or Postman (see FLOW-SPEC.md for command)
- [ ] Response contains a `text` field with readable OCR output
- [ ] Serial number is visible in the OCR text (e.g. `S/N CZJ1234567`)
- [ ] Test with a blank/unreadable image — confirm flow returns HTTP 200 with empty or partial text (not a 500 error)
- [ ] Test with an oversized image (>5MB) — confirm flow does not time out or crash
- [ ] Confirm AI Builder credit consumption appears in the Power Platform admin centre

---

## Phase 3 — Power App: Home Screen (Screen 1)

- [ ] App connects to Rack Surveys table without errors (no yellow warning triangles on the gallery)
- [ ] Gallery shows the manually created test rack from Phase 1
- [ ] Status badge colour is correct: grey (Pending), amber (In Progress), green (Complete)
- [ ] Tapping a rack row sets `gblRackRecord` and navigates to Screen 2
- [ ] "+ New" button sets `gblEditMode = false` and navigates to Screen 2
- [ ] Empty state message appears when no racks exist (temporarily delete the test record to verify)
- [ ] Newest rack appears first (sort order)

---

## Phase 4 — Power App: Rack Detail (Screen 2)

- [ ] Screen opens in **create mode** (via + New): Rack ID field is blank, Save creates a new record
- [ ] Screen opens in **edit mode** (via row tap): fields pre-populated with existing rack data
- [ ] Rack ID is required — Save button shows an error or is disabled if blank
- [ ] Status dropdown defaults to Pending on new record
- [ ] Saving a new rack record creates it in Dataverse (verify in make.powerapps.com → Tables)
- [ ] Editing an existing rack and saving updates the record (not creates a duplicate)
- [ ] Front of Rack photo slot: tap → camera or file picker opens → photo appears in slot
- [ ] Back of Rack photo slot: same as above
- [ ] Photos are saved as Note attachments on the Rack Survey record (verify in Dataverse → record → Timeline)
- [ ] Equipment Items gallery shows only items linked to the current rack
- [ ] "Add Equipment Item" button navigates to Screen 3 with a blank item
- [ ] Tapping an existing equipment item navigates to Screen 3 with that item pre-populated
- [ ] Back button returns to Home Screen

---

## Phase 5 — Power App: Equipment Item (Screen 3)

- [ ] All fields visible: Start U, End U, Category, Serial Number, Part Number, Manufacturer, Notes
- [ ] Category dropdown shows all 7 options
- [ ] Start U is required — Save is blocked if blank
- [ ] End U defaults to same value as Start U
- [ ] Save in create mode creates a new Equipment Item linked to the current rack
- [ ] Save in edit mode updates the existing record
- [ ] Delete button appears in edit mode only — deletes the record and returns to Screen 2
- [ ] Photo attachment slot: tap → camera or file picker → photo saved as Note attachment on Equipment Item
- [ ] Scan button triggers AI Builder Text Recognizer on the attached photo
- [ ] OCR result populates Serial Number and/or Part Number fields
- [ ] OCR Status updates to Scanned on success, Error on failure
- [ ] Back button returns to Rack Detail without saving

---

## Phase 6 — Power App: Scan / OCR (Screen 4)

- [ ] Screen shows all Equipment Items for the current rack where OCR Status = Pending
- [ ] Progress indicator updates as each item is processed
- [ ] Each item's Serial Number and Part Number fields are populated after scanning
- [ ] OCR Status updates to Scanned in Dataverse after successful scan
- [ ] Items that fail OCR show status Error with a visible message
- [ ] Items with OCR Status = Scanned are excluded from the pending list (do not re-scan)
- [ ] Empty state shown when all items are already scanned

---

## Phase 7 — Power App: Review / Export (Screen 5)

- [ ] All equipment items across all racks are visible in the table
- [ ] Rack filter works — selecting a specific rack shows only that rack's items
- [ ] Status filter works
- [ ] Category filter works
- [ ] Inline cell editing saves back to Dataverse on change
- [ ] Export button triggers Flow 2 (Export Flow)
- [ ] Confirmation message shows item count and success/failure status
- [ ] Email arrives with correct Excel attachment
- [ ] Excel file opens cleanly — columns: Rack ID, Location, Start U, End U, Category, Manufacturer, Part Number, Serial Number, Notes, OCR Status
- [ ] Rows are sorted by Rack then by Start U

---

## Phase 8 — End-to-end test (full workflow)

Run this with a real rack or a test rack in the office.

- [ ] Create a new rack survey on a phone (Screen 1 → Screen 2)
- [ ] Add 3 equipment items with different U positions and categories
- [ ] Take a photo for at least one item
- [ ] Change rack status to In Progress — confirm badge updates on Home Screen
- [ ] Run OCR on the photo item (Screen 4) — confirm serial number is populated
- [ ] Manually enter serial number for one item without a photo
- [ ] Leave one item with no serial number (OCR Status = Pending)
- [ ] Go to Review screen — confirm all 3 items appear
- [ ] Export — confirm email received with all 3 items, correct rack ID and U positions
- [ ] Change rack status to Complete — confirm badge turns green on Home Screen
- [ ] A second user on a different device opens the app — confirm they see the same rack data

---

## Phase 9 — Access and permissions

- [ ] A user who is NOT the app maker can open the app (shared correctly in Power Apps)
- [ ] That user can create, edit, and delete their own Rack Survey records
- [ ] That user can see racks created by other users (if shared visibility is intended)
- [ ] Flow 2 (Export) runs under a service account or shared connection — not tied to one user's credentials
- [ ] Flow 1 (OCR) URL is not visible to end users (stored in app config, not exposed in UI)

---

## Known limitations to document for users

- Photos taken on the app are stored as Dataverse Note attachments — they are not directly downloadable in bulk from the app. Use the Export ZIP feature from the original desktop HTML tool if bulk photo export is needed.
- AI Builder Text Recognizer works best on well-lit, in-focus photos of printed labels. Handwritten serials or embossed/engraved text may not scan reliably.
- Each photo OCR scan consumes 1 AI Builder credit.
- Dataverse Note attachments have a default 32MB per-file size limit. Photos taken on modern phones may need to be resized before attaching (the field app HTML tool already resizes to 1600px max).
