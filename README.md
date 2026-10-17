# CWQT water quality map: automatic updates

This repository rebuilds the CWQT map and charts from the master Google Sheet **every morning** and
publishes them to a free website (GitHub Pages). The SWIM Wix site embeds that page, so it stays
current without anyone touching Wix or running code.

```
Google Sheet ──(daily, 7:17 am)──> GitHub Actions runs cwqt_visualize.py ──> GitHub Pages site ──> embedded on Wix
```

| File | What it is |
|---|---|
| `cwqt_visualize.py` | The script. Thresholds, season dates and the wet-weather rule are in its CONFIGURATION section. |
| `requirements.txt` | Python packages the script needs. |
| `.github/workflows/update-map.yml` | The daily schedule: download sheet → build → publish. |
| `status.json` | Written by the workflow: the latest sample date and the sample count. |

**What gets published** (public): the map (`index.html`), the five chart PNGs and `site_timeseries.pdf`.
**Not published:** the workbook, the CSV tables and the data-quality report. To change this, edit the
"Choose what gets published" step in the workflow.

---

## One-time setup (about 20 minutes)

### 1. Create the repository
1. Sign in to GitHub → **New repository**. Name it e.g. `cwqt-map`, set it to **Public** (GitHub Pages
   is free only for public repositories), and create it.
2. Upload every file in this folder, including the hidden `.github/workflows/update-map.yml` file. On
   the web, use **Add file → Upload files** and drag in the whole folder, or use GitHub Desktop.

### 2. Let the workflow read the Google Sheet (pick one)
The sheet ID is the long code in the sheet's address:
`https://docs.google.com/spreadsheets/d/`**`1AbCdEf...xyz`**`/edit`

**Option A: link sharing (simplest).** In the sheet, go to **Share → General access → Anyone with the
link → Viewer**. No other setup is needed. Anyone who has the link can view the sheet, so use this
only if that's acceptable.

**Option B: keep the sheet private (service account).**
1. At <https://console.cloud.google.com>, create a project, then go to **APIs & Services → Enable APIs**
   and enable the **Google Drive API**.
2. Go to **IAM & Admin → Service Accounts → Create**. Give it any name and skip the roles.
3. Open the new account → **Keys → Add key → JSON**. A `.json` file downloads.
4. Share the Google Sheet (Viewer) with the service account's email address, which looks like
   `something@project.iam.gserviceaccount.com`.

### 3. Add the secrets to GitHub
Repository → **Settings → Secrets and variables → Actions → New repository secret**:
- `CWQT_SHEET_ID`: the sheet ID from step 2.
- `GOOGLE_SERVICE_ACCOUNT_JSON`: Option B only. Paste the entire contents of the `.json` key file.

Secrets are never shown publicly, even though the repository is public.

### 4. Turn on GitHub Pages
Repository → **Settings → Pages → Build and deployment → Source: GitHub Actions**.

### 5. Run it once by hand
Repository → **Actions → Update CWQT map → Run workflow**. After 2–3 minutes it should show a green
check. The site address appears on the run's summary page and looks like
`https://<your-username>.github.io/cwqt-map/`. Open it to check the map.

### 6. Embed on Wix
Wix Editor → **Add Elements → Embed Code → Embed a site** → enter the site address from step 5.
Resize the box (around 900 × 650 px works well on desktop) and adjust it separately in Wix's mobile
editor. Wix embeds don't resize automatically, so consider adding a link under it that opens the
map full screen.

---

## Day to day
- **Nothing to do.** Enter data in the sheet as usual. The next morning's run picks it up.
- **Need it sooner?** Actions → Update CWQT map → **Run workflow**.
- **Is it running?** The Actions tab lists every run. The map's title also shows "Data through …".
- **If a run fails,** GitHub emails the account that owns the repository. The usual causes are:
  - the sheet's sharing changed;
  - a tab or column in the sheet was renamed (the script looks columns up by their exact names);
  - the service-account key was deleted.

  Open the failed run to see the script's error message.
- **Off-season:** the workflow keeps running daily (it costs nothing) and makes an occasional
  "keep-alive" commit. GitHub pauses scheduled jobs in repositories with no commits for 60 days, and
  these commits prevent that.

## Handing this over to SWIM
For now this lives on a personal account. To move it to a SWIM account or organization later, go to
**Settings → General → Transfer ownership**. Then:
- re-add the two secrets;
- turn Pages back on;
- **update the Wix embed**, because the site address changes to `https://<new-owner>.github.io/cwqt-map/`.

Before you leave, also add a SWIM staff member as a collaborator (**Settings → Collaborators**) so
someone else can see failures and re-run the workflow.

## Running it on your own computer
```
pip install -r requirements.txt
python cwqt_visualize.py CWQT_Master_Data_Sheet.xlsx          # from a downloaded .xlsx
python cwqt_visualize.py --sheet-id 1AbCdEf...xyz              # or straight from Google
```
Outputs go to `cwqt_outputs/`.
