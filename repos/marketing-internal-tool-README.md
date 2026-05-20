# Two Tools Workflow

## Tool 1: Local batch QR generator (CLI)
File: `/Users/xinz/Documents/fig qr/batch_qr_io.py`

### A) CSV mode
```bash
QR_IO_API_KEY="YOUR_KEY" python3 "/Users/xinz/Documents/fig qr/batch_qr_io.py" \
  --input "/Users/xinz/Documents/fig qr/input.sample.csv" \
  --out-dir "/Users/xinz/Documents/fig qr/out" \
  --format jpg --overwrite --workers 1 --insecure
```

### B) School batch mode (one command)
```bash
QR_IO_API_KEY="YOUR_KEY" python3 "/Users/xinz/Documents/fig qr/batch_qr_io.py" \
  --schools "ucla,usc,ucd,ucr,uci,ucsb,cal" \
  --link-template "https://ditto.ai/?ref={school}" \
  --name-template "{school}_qr" \
  --out-dir "/Users/xinz/Documents/fig qr/out" \
  --format jpg --qrtype dynamic --group-by-school --overwrite --workers 1 --insecure
```

Output:
- QR files: `{school}_qr.jpg`
- Grouped folders (with `--group-by-school`): `out/{school}/{school}_qr.jpg`
- Summary CSV: `out/results.csv`

### C) Local GUI mode
File: `/Users/xinz/Documents/fig qr/qr_batch_gui.py`

Run:
```bash
python3 "/Users/xinz/Documents/fig qr/qr_batch_gui.py"
```

GUI supports:
- Select specific schools by checkbox.
- Input your naming suffix.
- Output naming format: `{school}_{your_name}` (example: `ucsd_poster1`).
- Default QR type is `dynamic`.
- Optional grouping into school folders.
- One-click batch generate via qr.io.

## Tool 2: Figma plugin (batch replace + rename + duplicate)
Folder: `/Users/xinz/Documents/fig qr/figma-qr-batch`

Features:
- Multi-select schools.
- Duplicate template frame.
- Replace `QR_SLOT`.
- Rename frame to `school_posterName` (e.g. `ucla_poster1`).
- Rename QR slot layer to `school_posterName_qr`.
- Supports canvas source mode (no network) and qr.io API mode.

---

## New: Marketing Internal Tool (MVP Web App)

Path: `/Users/xinz/Documents/fig qr/marketing-internal-tool`

### What is implemented
- Single website with 4 tabs: `Editor`, `Generate`, `QR Analytics`, `Archive`
- Editor supports:
  - Add `text` layers with placeholders like `{school}`
  - Upload image and add `image` layer on canvas
  - Add `qrcode_slot` layers and configure URL template
  - Save template versions and publish latest version
- Generate supports:
  - Dynamic form from template fields
  - Single poster generation + preview + PNG download
  - School-list batch generation (from managed school list)
  - Generate all published templates for one selected school
- Schools tab supports:
  - Add/delete schools (used by Generate school selector)
- QR analytics:
  - Short link redirect endpoint (`/q/:code`)
  - PV/UV metrics
- Archive:
  - Generation history
  - Replay existing poster instance

### Run
```bash
cd "/Users/xinz/Documents/fig qr/marketing-internal-tool"
npm start
```

Then open:
`http://localhost:8787`

### Batch generation (no CSV)
- Default school list is preloaded and selected by default.
- You can add a custom school in the input box.
- Click `Run Batch by Selected Schools` to generate all selected entries.
