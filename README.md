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
        (Preview → Commit)
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
| Business Rule | `Export Update Set to GitHub` (on `sys_update_set`) | Auto-triggers export when an Update Set state changes to **Complete** |
| Scripted REST API (Optional) | `GET /api/1838298/update_set_exporter/export` | Manual trigger endpoint — callable from REST API Explorer with `sys_id` query param |

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
1. Detect changed XML files (`git diff HEAD~1 HEAD` with `fetch-depth: 2`)
2. Construct GitHub Raw URL for each changed file
3. Call UAT PDI Scripted REST API with the raw URL as `github_url` payload
4. UAT API fetches the XML from GitHub, parses it, and inserts into `sys_remote_update_set` and `sys_update_xml` (Retrieved Update Sets)

**GitHub Secrets required:**

| Secret | Value |
|--------|-------|
| `UAT_SN_INSTANCE` | `dev341667.service-now.com` |
| `UAT_SN_USERNAME` | `admin` |
| `UAT_SN_PASSWORD` | UAT PDI admin password |

### UAT PDI — Import Side

| Component | Name | Purpose |
|-----------|------|---------|
| Scripted REST API | `POST /api/1790158/update_set_importer/import` | Receives GitHub Raw URL, downloads XML, parses it, and inserts records into `sys_remote_update_set` and `sys_update_xml` |

After the workflow completes, a reviewer manually **Previews** and **Commits** the Update Set on the UAT PDI.

---

## Key Design Decisions

**File naming by `sys_id`**
Files are named by the local Update Set `sys_id` (e.g. `06a9d0f793633250e502fbf08bba109d.xml`). This ensures the same file is always overwritten on re-export even if the Update Set name changes — GitHub's SHA-based update mechanism handles versioning.

**GitHub Raw URL as payload**
Rather than embedding the full XML in the API request body, GitHub Actions sends only the raw file URL. The UAT Scripted REST API then fetches the XML directly from GitHub. This avoids payload size limits and complex escaping.

**Scripted REST API on UAT (not standard endpoints)**
Standard ServiceNow import endpoints (`/api/now/import/sys_remote_update_set`, `/upload`, table API) returned 405 or produced empty records. A custom Scripted REST API on the UAT instance was built to properly parse the `<unload>` XML and insert into `sys_remote_update_set` and `sys_update_xml` via GlideRecord.

**XML format: `<unload>` wrapper**
ServiceNow's Retrieved Update Sets mechanism requires the standard `<unload>` format. `UpdateSetGitExporter` uses OOB `UpdateSetExport().exportUpdateSet()` to create a `sys_remote_update_set` record, then reads it back via GlideRecord to assemble the `<unload>` XML using `_buildUnloadXml()` and `_serializeRecordWithOrder()`. Fields are read dynamically — any fields not listed in the ordered arrays are automatically appended, giving partial resilience to future platform upgrades.

**`export_update_set.do` not usable server-side**
The OOB export endpoint requires a browser session cookie and cannot be called via `sn_ws.RESTMessageV2` (returns 401 regardless of auth method). All approaches were exhausted — Basic Auth, session token, JSESSIONID cookie injection, `GlideUpdateSetExport`, `GlideSysUpdateXML`, `ExportWithRelatedLists` + `ByteArrayOutputStream`. The GlideRecord-based XML assembly in `_buildUnloadXml()` is the only viable server-side approach in a PDI environment.

**Preview and Commit remain manual**
UAT import stops at `state: loaded`. Preview and Commit are intentionally left as manual steps so a reviewer can inspect changes and catch conflicts before they are applied to the instance.

---

## Issues / Resolutions

| Issue | Root Cause | Resolution |
|-------|-----------|------------|
| `export_update_set.do` returned 401 | Requires browser session cookie — not accessible via server-side HTTP client (`sn_ws.RESTMessageV2`) | Built `_buildUnloadXml()` to assemble `<unload>` XML directly from GlideRecord |
| `git diff` failed on first commit | `github.event.before` is invalid on initial push | Added `fetch-depth: 2` and used `git diff HEAD~1 HEAD` |
| GitHub Actions detected no changed files | `github.event.commits[0].added` unreliable | Switched to `git diff --name-only HEAD~1 HEAD` |
| UAT standard import endpoints returned 405 | OOB endpoints don't accept raw XML payload in expected format | Built custom Scripted REST API on UAT |
| `new GlideDateTime()` chaining errors | ES12 mode incompatibility on PDI | Disabled ECMAScript 2021; used `gs.nowDateTime()` |
| UAT API returned 401 | Admin password not explicitly set on UAT PDI (SSO-only login) | Set admin password directly on UAT PDI; stored in GitHub Secrets |
| UAT inserted empty records | `XMLDocument2.getNodes()` and `node.getNodeText()` returned `undefined` | Switched to full XPath access: `xmlDoc.getNodeText('//sys_update_xml[i]/field')` |
| UAT `Application` field blank | `application` is a reference field requiring `sys_scope.sys_id` | Look up `sys_scope` by `application_scope` value; abort import if scope not found on UAT |

---

## Manual Trigger (DEV → GitHub)

Via REST API Explorer on DEV PDI:

```
GET /api/1838298/update_set_exporter/export?sys_id=<update_set_sys_id>
```

Or via curl:

```bash
curl -X GET \
  "https://dev296169.service-now.com/api/1838298/update_set_exporter/export?sys_id=<update_set_sys_id>" \
  -H "Authorization: Basic <base64(user:password)>" \
  -H "Accept: application/json"
```

---

## Limitations

- **PDI only:** `export_update_set.do` is blocked server-side in PDI. In a real instance where Basic Auth is permitted, `_buildUnloadXml()` could be replaced with a simple HTTP call to that endpoint, significantly reducing code complexity.
- **Basic Auth on UAT:** GitHub Actions authenticates to the UAT PDI using admin credentials. In SSO-enforced environments, a dedicated service account (SSO-excluded) or OAuth 2.0 would be required.
- **Preview / Commit not automated:** The pipeline stops at `loaded` state. Full automation would require additional conflict detection and resolution logic before Commit can be safely automated.

---

## Roadmap

- Expand to full DEV → UAT → PROD pipeline (requires additional ServiceNow instances)
- Add XML schema validation step in GitHub Actions
- Explore PR-based deployment approval flow
- Explore Azure DevOps port
- Evaluate OAuth 2.0 for UAT authentication
