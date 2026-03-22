# ServiceNow Update Set CI/CD Pipeline

Automated pipeline to export Update Sets from a ServiceNow DEV PDI to GitHub, and auto-deploy them to a UAT PDI via GitHub Actions.

```
DEV PDI (dev296169)
    │
    │  Business Rule (Main: on Complete)
    │  Scripted REST API (Optional: manual trigger)
    ▼
GitHub Repository
    └── update_sets/*.xml
            │
            │  GitHub Actions (deploy-to-uat.yml)
            ▼
    UAT PDI (dev341667)
        Retrieved Update Sets
        (Load → Preview → Commit)
```

---

## Repository Structure

```
servicenow-update-sets/
├── update_sets/          # Exported Update Set XML files
│   └── *.xml
└── .github/
    └── workflows/
        └── deploy-to-uat.yml
```

---

## Components

### DEV PDI — Export Side

| Component | Name | Purpose |
|-----------|------|---------|
| Script Include | `UpdateSetGitExporter` | Core export logic — builds `<unload>` XML and pushes to GitHub via `sn_ws.RESTMessageV2` |
| Business Rule | (on `sys_update_set`) | Auto-triggers export when an Update Set state changes to **Complete** |
| Scripted REST API (Optional) | `POST /api/1838298/update_set_exporter/export` | Manual trigger endpoint — callable from GitHub Actions or REST API Explorer |

#### System Properties

| Property | Type | Description |
|----------|------|-------------|
| `update_set_exporter.github_token` | `password2` | GitHub Personal Access Token |
| `update_set_exporter.github_owner` | `string` | GitHub username / org |
| `update_set_exporter.github_repo` | `string` | Repository name |
| `update_set_exporter.github_branch` | `string` | Target branch (default: `main`) |

### GitHub → UAT — Deploy Side

**Workflow trigger:** any push to `update_sets/**.xml`

Steps:
1. Detect changed XML files (`git diff` with `fetch-depth: 2`)
2. Download each XML from GitHub Raw URL
3. Call UAT PDI Scripted REST API with the raw URL as payload
4. UAT API fetches the XML, inserts it into `sys_remote_update_set` (Retrieved Update Sets)

**GitHub Secrets required:**

| Secret | Value |
|--------|-------|
| `UAT_SN_INSTANCE` | `dev341667.service-now.com` |
| `UAT_SN_USERNAME` | `admin` (Optional)|
| `UAT_SN_PASSWORD` | UAT PDI admin password (Optional)|

### UAT PDI — Import Side

| Component | Name | Purpose |
|-----------|------|---------|
| Scripted REST API | `POST /api/1790158/update_set_importer/import` | Receives GitHub Raw URL, downloads XML, inserts record into `sys_remote_update_set` |

After the workflow completes, a reviewer manually **Previews** and **Commits** the Update Set on the UAT PDI.

---

## Key Design Decisions

**File naming (no timestamp)**
Files are named by Update Set name only (e.g. `My_Update_Set.xml`). This enables clean overwrites on re-export — GitHub's SHA-based update mechanism handles versioning.

**GitHub Raw URL as payload**
Rather than embedding the full XML in the API request body, GitHub Actions sends only the raw file URL. The UAT Scripted REST API then fetches the XML directly from GitHub. This avoids payload size limits and complex escaping.

**Scripted REST API on UAT (not standard endpoints)**
Standard ServiceNow import endpoints (`/api/now/import/sys_remote_update_set`, `/upload`, table API) returned 405 or produced empty records. A custom Scripted REST API on the UAT instance was built to properly parse the payload and insert into `sys_remote_update_set`.

**XML format: `<unload>` wrapper**
ServiceNow's Retrieved Update Sets mechanism requires the standard `<unload>` format. The `_getUpdateSetXML` method in `UpdateSetGitExporter` builds this format by querying `sys_update_xml` records directly via GlideRecord.

---

## Known Issues / In Progress

- [ ] `_getUpdateSetXML` currently produces a custom `<update_set>` format instead of the required `<unload>` format — fix in progress
- [ ] UAT Scripted REST API import logic needs to properly parse and load the XML payload (currently inserts placeholder records)

---

## Issues / Resolutions

| Issue | Root Cause | Resolution |
|-------|-----------|------------|
| `export_update_set.do` returned 401 | Requires browser session cookie — not accessible via server-side HTTP client | Switched to direct GlideRecord XML assembly |
| `git diff` failed on first commit | `github.event.before` is invalid on initial push | Added `fetch-depth: 2` and used `HEAD~1 HEAD` |
| UAT standard import endpoints returned 405 | OOB endpoints don't accept raw XML payload in expected format | Built custom Scripted REST API on UAT |
| `new GlideDateTime()` chaining errors | ES12 mode incompatibility on PDI | Disabled ECMAScript 2021; used `gs.nowDateTime()` |
| UAT API returned 401 | Admin password not explicitly set on UAT PDI | Set admin password directly on UAT PDI |

---

## Manual Trigger (DEV → GitHub)

```bash
curl -X POST \
  "https://dev296169.service-now.com/api/1838298/update_set_exporter/export" \
  -H "Authorization: Basic <base64(user:password)>" \
  -H "Content-Type: application/json" \
  -d '{"sys_id": "<update_set_sys_id>", "name": "<update_set_name>"}'
```

---

## Roadmap

- Update `<unload>` XML format generation on DEV
- Update UAT import to properly load XML payload
- Evaluate `sys_id`-based file naming for cleaner overwrite logic
- Expand to full DEV → UAT → PROD pipeline
- Add XML schema validation step in GitHub Actions
- Explore Azure DevOps port

