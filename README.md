# CWQT Data Analysis

The **Community Water Quality Testing Data Portal**: an interactive map and charts of Enterococcus
results from the CWQT program, based on the CWQT master Google Sheet.

**Data Portal:** <https://jcuomo4.github.io/CWQT-Data-Analysis/>
**CWQT Master Google Sheet:** <https://docs.google.com/spreadsheets/d/1813b2nagaxZ80xRfyMZNNKySZOitro5Nt7W4E9WNQDA/edit?pli=1&gid=1924583806#gid=1924583806>

```
CWQT master Google Sheet
        │  downloaded daily at 7:17 am ET (GitHub Actions)
        ▼
cwqt_visualize.py + templates/  ──►  map, charts page, PDFs  ──►  GitHub Pages
```

Results entered in the master sheet automatically appear on the map the next morning.

---

## What's in this repository

| File | Purpose |
|---|---|
| `cwqt_visualize.py` | Reads the workbook, calculates the statistics and writes every output. All thresholds and definitions are in its **CONFIGURATION** section near the top. |
| `templates/` | The web pages' design: `map.html` (map page), `charts.html` (charts page), `cwqt.css` (shared colors, fonts, banner). Edit these to change how the portal looks; see [Changing the look](#changing-the-look-of-the-portal). |
| `requirements.txt` | Python packages the script needs. |
| `.github/workflows/update-map.yml` | The daily job: download the sheet → run the script → publish the site. |
| `status.json` | Written by the daily job: latest sample date, sample count and site count. A new commit means new data arrived. |
| `.gitignore` | Keeps the workbook and generated files out of the repository. |

## What the script produces

| Output | Shows | Published on the site? |
|---|---|---|
| `index.html` | The map page. It has a SWIM banner with links to the charts page and a **Layers** menu: CWQT results by season, "Latest 30 days", "All seasons" and earlier seasons, plus basemap choices and a streams overlay. Clicking a site shows its stats, a mini trend chart and a link to its full time series. (`cwqt_map.html` is a copy, so the old address still works.) | Yes |
| `charts.html` | Charts page: summary numbers, each chart with a short description, and a list of every site's time-series PDF | Yes |
| `cwqt.css` | Shared styles for both pages (copied from `templates/`) | Yes |
| `1_program_trend.png` | Typical site geomean by season (all sites, and long-running sites only) and % of samples over 104, wet vs dry weather | Yes |
| `2_site_by_season_heatmap.png` | Every site × every season, colored good / advisory / poor | Yes |
| `3_rainfall_effect.png` | Bacteria levels by rainfall in the ~72 hours before sampling | Yes |
| `4_waterbody_comparison.png` | Geomean by waterbody over the last 5 seasons, with one dot per site | Yes |
| `5_latest_30_day_status.png` | 30-day geomean for each site (same calculation as the sheet's GeoMean tabs) | Yes |
| `site_timeseries.pdf` | One page per site: every regular-season sample, by season | Yes |
| `sites/site-NNN.pdf` | The same page for each site on its own (NNN = Site ID), linked from the map popups and the charts page | Yes |
| `tables/*.csv` | The numbers behind every chart | No |
| `data_quality_report.txt` | Problems found in the sheet (bad coordinates, duplicate rows, names out of sync, lab QC notes…) | No |
| `status.json` | Latest sample date and counts | Committed to the repository |

Published files are reachable by name, e.g. `…/CWQT-Data-Analysis/1_program_trend.png`. What gets
published is set in the "Choose what gets published" step of the workflow.

---

## How the numbers are calculated

The script follows the sheet's FAQ and its GeoMean tabs. Each run, it recalculates the
**30 Day GeoMean Primary Contact** tab and prints whether every site matches (it should report
79/79 or similar).

- **Censored results:** `<10` counts as 10, `>24196` as 24196 and `>6000` as 6000.
- **Geometric mean** = exp(mean(ln(MPN))).
- **Categories** (NYC DOH primary contact, same as the sheet's green/yellow/red):
  - Good: geomean ≤ 35
  - Advisory: 35–104
  - Poor: > 104
- **Single-sample limit** for "% of samples over": 104.
- **Fewer than 5 samples:** a geomean from fewer than 5 samples is shown with a dashed / hollow
  marker, because standards require at least 5.
- **NYS DEC criteria** shown in map popups:
  - Class I: geomean 72, threshold 266
  - Classes SB and SA: geomean 35, threshold 130
- **Regular season** = May 1 – Oct 5. Seasonal statistics use only these samples. Winter and
  post-season samples appear in the "Latest 30 days" layer but not in seasonal geomeans.
- **Wet weather** = at least 0.25" of rain at Central Park on the sampling day plus the 2 days
  before. Precipitation text is cleaned up first: "Trace" counts as 0, and "0.45 (3.2 snow)" counts
  as 0.45.
- **Lab/field splits** recorded in Notes are ignored by default, so the numbers match the GeoMean
  tabs. Run with `--average-splits` to average them as the FAQ describes.

To change any of these, edit the CONFIGURATION section of `cwqt_visualize.py`
(`GOOD_MAX`, `ADVISORY_MAX`, `SEASON_START`, `WET_WEATHER_INCHES`, `RAIN_DAYS`, `DEC_CRITERIA`, …).
The next daily run uses the new values.

### What the script expects in the sheet
Columns are found **by their header names**, so moving or inserting columns is fine.
- **Required.** Renaming any of these stops the daily run:
  - Data tab: `Site`, `Full Date` and
    `Most Probable Number (MPN) of Enterococcus colonies per 100 ml`.
  - Information tab: a header row containing `Table of Contents - Site (Listed Alphabetically)`
    followed by `Site ID`, plus `Currently Testing`, `Waterbody`, `Borough/State`,
    `Neighborhood/Town`, `Latitude`, `Longitude` and `NYS DEC Water Body Classification`.
- **Used if present.** Renaming one of these quietly turns off the feature that depends on it:
  - `Site ID` (otherwise sites are matched by name);
  - `Notes` (lab splits and QC notes);
  - the seven precipitation columns, `Day of Collection Precipitation (Thursday)`,
    `Previous Day Precipitation (Wednesday)`, `Previous Tuesday Precipitation`, … (wet/dry weather).

New sites only need a row on the Information tab (with coordinates) and a matching Site ID in Data.

---

## The daily update

`.github/workflows/update-map.yml` runs every day at 11:17 UTC (7:17 am EDT / 6:17 am EST) on
Ubuntu 24.04:

1. Downloads the sheet as .xlsx, using the `CWQT_SHEET_ID` secret (plus
   `GOOGLE_SERVICE_ACCOUNT_JSON` if the sheet is private).
2. Runs `python cwqt_visualize.py --out build`.
3. Copies the public files to `site/` and deploys them to GitHub Pages.
4. Commits `status.json` if the data changed. If 45 days pass with no commit (e.g. off-season), it
   makes an empty "keep-alive" commit instead, because GitHub pauses scheduled jobs in repositories
   with no commits for 60 days.

**Update now:** Actions → *Update CWQT map* → **Run workflow**.

**Check it's working:**
- The map title says "Data through <date>".
- The Actions tab shows a green check for each morning.
- The commit history shows "Data updated: …" after new results are entered.

### If a run fails
GitHub emails the account that owns the repository. Open the failed run → `update` job to see the
error, then check:

| Message | Likely cause / fix |
|---|---|
| "Could not download the Google Sheet" / "Google returned a web page" | The sheet's sharing changed, the sheet ID secret is wrong, or the service-account key was deleted. |
| "Data tab is missing expected column(s)" / "Could not find a header row" | A column or tab in the sheet was renamed. Rename it back, or update the names in the CONFIGURATION section. |
| Fails in "deploy" | Settings → Pages → Source must be **GitHub Actions**. |
| No runs at all for weeks | The schedule was paused. Open Actions → *Update CWQT map* → **Enable workflow**. |

GitHub occasionally retires old versions of the actions used in the workflow (`actions/checkout`,
`setup-python`, …). Warnings about this appear as "annotations" on a run. Updating the version
numbers in the `uses:` lines fixes them.

---

## Changing the look of the portal

The design lives in `templates/` and is separate from the calculations. Edit those files on GitHub
(pencil icon → commit), then run the workflow (or wait for the next morning) to publish.

| To change… | Edit |
|---|---|
| Colors, fonts, button style, banner height | the **SWIM BRAND** block at the top of `templates/cwqt.css` |
| Banner title, button labels, popup wording, legend text | `templates/map.html` |
| Chart descriptions and page text on the charts page | `templates/charts.html` |
| The logo | It's loaded from SWIM's Wix site (`LOGO_URL` in `cwqt_visualize.py`). To host it here instead, save it as `templates/assets/swim-logo.png`. |
| Basemap choices or the streams overlay | the `BASEMAPS` and `WATERWAYS` lists near the top of the script in `templates/map.html` |
| Thresholds, categories, season dates | the CONFIGURATION section of `cwqt_visualize.py` (used by the charts *and* the map) |

Words in double curly braces in the templates (e.g. `{{DATA}}`, `{{N_SITES}}`) are filled in by the
script. Keep them where you want those values to appear.

**Adding map layers later** (e.g. CSO outfalls, green infrastructure): put the data file (GeoJSON) in
`templates/assets/`, then add an entry to the `EXTRA_LAYERS` list in `templates/map.html`. There's a
commented example there. Each entry appears as its own group with a checkbox in the Layers menu.

**Current design** (October 2026), matched to swimmablenyc.org:
- Poppins font, SWIM navy `#092e6e` with square-cornered buttons, SWIM blue `#0d4ab2` links, cream
  `#f7f7f3` page background.
- Light gray basemap with the USGS streams and rivers overlay on top, so narrow waterways like the
  Bronx River stand out. "Streets" and "Satellite" basemaps are also available.
- On phones the legend starts collapsed, button labels shorten, and tap targets are larger.

---

## Running it on your own computer

```bash
pip install -r requirements.txt

# from a downloaded copy (Google Sheets → File → Download → Microsoft Excel)
python cwqt_visualize.py CWQT_Master_Data_Sheet.xlsx

# or straight from Google Sheets
python cwqt_visualize.py --sheet-id <long code from the sheet's URL>
```

Options:

| Option | Effect |
|---|---|
| `--out FOLDER` | Where outputs go (default `cwqt_outputs/`). |
| `--window-end YYYY-MM-DD` | End date of the 30-day window (default: latest sample). |
| `--average-splits` | Average lab/field split results recorded in Notes. |
| `--site-pngs` | Also save one PNG per site. |
| `--no-map` | Skip the web pages (map and charts page). |

Outputs go to `cwqt_outputs/` (or the `--out` folder). Open `index.html` there in a browser to
preview the portal before publishing.

---

## Setup (for a new copy or after transferring the repository)

1. **Public repository** with these files. Free GitHub Pages requires a public repository.
2. **Sheet access.** Either:
   - share the sheet as *Anyone with the link → Viewer*; or
   - keep it private: create a Google Cloud service account, enable the Drive API, share the sheet
     with the service account's email, and download its JSON key.
3. **Secrets** (Settings → Secrets and variables → Actions):
   - `CWQT_SHEET_ID`
   - `GOOGLE_SERVICE_ACCOUNT_JSON` (private sheet only)
4. **Pages:** Settings → Pages → Source: **GitHub Actions**.
5. **First run:** Actions → *Update CWQT map* → **Run workflow**. The site address appears under
   Settings → Pages.
6. **Wix:** Add Elements → Embed Code → **Embed a site** → the Pages address.

**Moving the repository to a SWIM account:** use Settings → General → Transfer ownership. Then:
- re-add the secrets;
- turn Pages back on;
- **update the Wix embed**, because the site address changes to
  `https://<new-owner>.github.io/CWQT-Data-Analysis/`;
- add SWIM staff as collaborators so failure emails reach someone who can act on them.

---

## Known data issues to fix in the sheet
Run the script locally and open `data_quality_report.txt` for the current list. As of October 2026:
- Site 160 (Hackensack) has a positive longitude. The script corrects it for the map.
- 8 duplicate site+date rows.
- 5 rows with no MPN result.
- 4 site names in Data that differ from Information.
- No 2026 rainfall entered yet, so 2026 has no wet/dry split.

