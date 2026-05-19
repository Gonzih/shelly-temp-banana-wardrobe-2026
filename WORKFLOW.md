# Shelly Clothings — Photo to Squarespace Workflow

## Overview

Take photos of pre-owned kids clothing → convert → cluster by item → generate product listings → export to Squarespace.

---

## One-Time Setup

### Requirements
- macOS (uses `sips` for HEIC conversion)
- Python 3 with Flask: `pip install flask`
- ComfyUI venv at `/Users/feral/of-stack/ComfyUI/venv/bin/python` with `transformers==4.51.3`
- Florence-2 model at `/Users/feral/of-stack/ComfyUI/models/LLM/Florence-2-base`
- `gh` CLI logged into GitHub (for image hosting)
- Anthropic API key in env (optional — not used in current flow)

### Florence-2 Note
Must use `transformers==4.51.3` — version 5.x breaks Florence-2 with `AttributeError: forced_bos_token_id`.

---

## Photography Guidelines (for Shelly)

Each item needs **2–4 photos**, named descriptively:

| Photo | Filename pattern |
|-------|-----------------|
| Full garment (front) | `ralph lauren polo.jpg` |
| Brand tag | `ralph lauren polo brand tag.jpg` |
| Fabric/fibre tag | `ralph lauren polo fabric tag.jpg` |
| Detail or back (optional) | `ralph lauren polo detail.jpg` |

**Naming rules:**
- Use brand name + item type + color if helpful
- Suffixes `brand tag`, `fabric tag`, `detail`, `back side` are stripped automatically for grouping
- Trailing numbers (`polo 1`, `polo 2`) create separate items — use for items of the same type
- Unrenamed `IMG_XXXX.heic` files are fine but require manual cluster assignment in the UI

---

## Step 1 — Prepare: Convert & Auto-Cluster

Drop all HEICs into `heics/`, then run:

```bash
python3 prepare.py
```

This:
1. Converts all HEICs → JPEGs under 500kb in `jpegs/`
2. Groups files by item name (strips tag suffixes)
3. Writes `state.json` with numbered clusters ready for the server

---

## Step 2 — Run the Server

```bash
python3 server.py
```

Opens at `http://localhost:5555`

---

## Step 3 — Review & Adjust Clusters in UI

Open `http://localhost:5555` in browser.

- **Click an image** → move it to a different cluster or unassign
- **Shift/Cmd+click** → multi-select
- **Drag and drop** between clusters
- **Auto-group** → creates triplets from unassigned images by filename order
- Cluster names are editable (click to rename)
- All changes save automatically

Key things to check:
- Each cluster = one physical item
- Unrenamed `IMG_` files need manual placement
- Unassigned panel (left) should be empty when done
- Image order within a cluster matters — **put the garment photo first**

---

## Step 4 — Run Florence-2 Descriptions

Once clusters look correct, run Florence-2 on all clustered images:

```bash
/Users/feral/of-stack/ComfyUI/venv/bin/python run_florence.py
```

This runs two passes per image (`<MORE_DETAILED_CAPTION>` + `<OCR>`) and saves results to `florence_out.json`. Takes ~5–10 minutes for 120 images on MPS. Resumable — skips already-processed images.

---

## Step 5 — Generate CSV

```bash
python3 make_csv.py
```

This builds `squarespace/shelly_products.csv` using:
- Filenames → brand, item type, color/descriptor
- Florence-2 OCR → size and materials (from fabric tag photos)
- Squarespace v3 format with `Product Page`, `Hosted Image URLs`, prices, etc.

OCR from Florence-2 is noisy — for critical fields (size, materials on premium items) verify against the tag photos manually.

---

## Step 6 — Image Hosting (GitHub Pages)

Squarespace import requires **public URLs** for images.

### First time setup:
```bash
# Create repo (use an absurd throwaway name)
gh repo create gonzih/shelly-temp-banana-wardrobe-2026 --public

cd /tmp && mkdir shelly-gh && cd shelly-gh && git init && git checkout -b main
mkdir images
cp /Users/feral/Downloads/shelly-clothings/jpegs/*.jpg images/
echo "<html><body>temp</body></html>" > index.html
git add . && git commit -m "Add product images"
git remote add origin git@github.com:gonzih/shelly-temp-banana-wardrobe-2026.git
git push -u origin main

# Enable GitHub Pages
gh api repos/gonzih/shelly-temp-banana-wardrobe-2026/pages \
  --method POST -f 'source[branch]=main' -f 'source[path]=/'
```

Images serve at:
`https://gonzih.github.io/shelly-temp-banana-wardrobe-2026/images/{filename}`

GitHub Pages takes ~2 minutes to go live after first push.

### Update CSV with GitHub Pages URLs:
The `make_csv.py` script outputs localhost URLs by default. After setting up GitHub Pages, update `IMG_BASE` in `make_csv.py` to the GitHub Pages base URL and rerun, or run the URL-swap script used in this session.

### Push updated CSV to GitHub:
```bash
cp squarespace/shelly_products.csv /tmp/shelly-gh/
cd /tmp/shelly-gh
git add shelly_products.csv && git commit -m "Update CSV" && git push
```

CSV is then available at:
`https://raw.githubusercontent.com/gonzih/shelly-temp-banana-wardrobe-2026/main/shelly_products.csv`

---

## Step 7 — Import to Squarespace

### Prerequisites
- A **Store page** must already exist in Squarespace (Pages → + → Store)
- Note its URL slug (e.g. `shop`)
- Set `Product Page` column in CSV to that slug (already done in `make_csv.py`)

### Import steps
1. Squarespace admin → **Commerce → Products → Import**
2. Upload `shelly_products.csv`
3. Map columns if prompted
4. Import

### Known Squarespace import rules
- `Product Page` column **must** match an existing store page slug — blank = error
- `Hosted Image URLs` — space-separated URLs (not comma-separated)
- Images must be publicly reachable at import time
- `Price` field must be numeric (no $ sign)
- `Product URL` should be blank for new products — Squarespace generates from title

---

## Step 8 — Cleanup

After Squarespace has imported and images are attached:

```bash
# Delete the throwaway GitHub repo
gh repo delete gonzih/shelly-temp-banana-wardrobe-2026 --yes
```

---

## File Structure

```
shelly-clothings/
├── heics/              # Source HEIC photos (input)
├── jpegs/              # Converted JPEGs (auto-generated)
├── squarespace/
│   └── shelly_products.csv   # Final Squarespace import file
├── state.json          # Cluster assignments (UI state)
├── florence_out.json   # Florence-2 descriptions + OCR
├── prepare.py          # Step 1: convert + auto-cluster
├── server.py           # Step 2-3: clustering UI server
├── run_florence.py     # Step 4: run Florence-2
├── make_csv.py         # Step 5: generate CSV
├── ui.html             # Clustering UI frontend
└── product_import-v3.csv  # Squarespace format reference
```

---

## Pricing Reference (secondhand kids clothing, 2026)

| Brand tier | Price range |
|-----------|-------------|
| H&M, Gap basics, Cat & Jack, Faded Glory | $5–12 |
| Zara, Hanna Andersson, Mini Boden, Joules | $12–22 |
| Ralph Lauren polo shirts | $15–25 |
| Ralph Lauren dresses | $20–35 |
| Janie and Jack, Matilda Jane, Vineyard Vines | $18–35 |
| Patagonia, The North Face | $25–45 |
| Burberry kids | $40–80 |

---

## Next Batch — Quick Checklist

- [ ] Shelly names and drops HEICs into `heics/`
- [ ] `python3 prepare.py`
- [ ] `python3 server.py` → review clusters at localhost:5555
- [ ] Move item photo to first position in each cluster
- [ ] `/Users/feral/of-stack/ComfyUI/venv/bin/python run_florence.py`
- [ ] `python3 make_csv.py`
- [ ] Push images + CSV to GitHub, update GitHub Pages URLs in CSV
- [ ] Import CSV to Squarespace Commerce → Products
- [ ] Delete GitHub repo
