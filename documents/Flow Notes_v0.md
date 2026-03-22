# ServiceNow CI/CD Pipeline — Flow Notes

---

## What triggers the pipeline in ServiceNow?

The pipeline is triggered in two ways:

**Automatic trigger** — when an Update Set's state changes to `Complete`, a Business Rule on the `sys_update_set` table fires. It detects the state transition (`current.state == 'complete' && previous.state != 'complete'`) and directly calls `UpdateSetGitExporter.exportToGit()` inline.

**Manual trigger** — the Scripted REST API endpoint `GET /api/1838298/update_set_exporter/export` on the DEV PDI accepts a `sys_id` query parameter. You can call this directly from the REST API Explorer or from an external system whenever you want to force an export without changing the Update Set state.

Both paths converge on the same `UpdateSetGitExporter` Script Include, which handles the actual XML assembly and GitHub push.

---

## How does the Business Rule trigger the export?

The Business Rule runs **after update** on the `sys_update_set` table and checks for the state transition:

```javascript
if (current.state == 'complete' && previous.state != 'complete') {
    gs.log('Business Rule triggered: ' + current.name, 'GitExporter');

    var exporter = new UpdateSetGitExporter();
    exporter.exportToGit(
        current.sys_id.toString(),
        current.name.toString()
    );
}
```

A few things worth noting about this design:

- The export runs **synchronously inline** inside the Business Rule. This is straightforward and works well for PDI-scale Update Sets. For production environments with very large Update Sets, an asynchronous worker (`GlideScriptedHierarchicalWorker`) could be considered to avoid blocking the UI.
- The condition checks `previous.state != 'complete'` to avoid re-triggering if the record is updated again while already Complete (e.g. someone edits the description).
- `sys_id` and `name` are passed directly as string arguments to `exportToGit()`.

The Business Rule then hands off to `UpdateSetGitExporter.exportToGit()`, which does the actual XML assembly and GitHub API call.

---

## What does UpdateSetGitExporter do step by step?

Here's the step-by-step breakdown of what `UpdateSetGitExporter.exportToGit()` does once called:

### 1. Validate the local Update Set

It queries `sys_update_set` by the given `sys_id` and confirms the record exists and its state is `complete`. If either check fails, it logs an error and returns early.

### 2. Create a `sys_remote_update_set` record

It calls the OOB `UpdateSetExport().exportUpdateSet()` method, which:
- Creates a new `sys_remote_update_set` record linked to the local Update Set
- Copies all `sys_update_xml` records from the local Update Set into `sys_remote_update_set` context
- Returns the `sys_id` of the newly created `sys_remote_update_set` record

### 3. Build `<unload>` format XML

It reads the newly created `sys_remote_update_set` record and all its linked `sys_update_xml` records via GlideRecord, then assembles them into the standard ServiceNow `<unload>` format using `_buildUnloadXml()`. This was one of the hardest parts of the project — the naive approach of hitting `export_update_set.do` directly always returned 401 because it requires a browser session cookie, not Basic Auth. The final solution assembles the XML manually using `_serializeRecordWithOrder()`, which dynamically reads all fields and also captures any fields not explicitly listed in the ordered field arrays.

### 4. Base64-encode the content

The assembled XML string is encoded via `GlideStringUtil.base64Encode()` — required for the GitHub Contents API payload.

### 5. Check if the file already exists on GitHub (get SHA)

It calls `GET /repos/{owner}/{repo}/contents/update_sets/{local_sys_id}.xml` to check whether the file already exists. If it does, GitHub returns the file's current `sha`. This SHA must be included in the next request — without it, GitHub rejects the update with a 422 conflict error.

### 6. PUT to GitHub Contents API

It calls `PUT /repos/{owner}/{repo}/contents/update_sets/{local_sys_id}.xml` via `sn_ws.RESTMessageV2` with:

- the Base64-encoded XML as `content`
- a commit message like `export official-like unload xml: {name} ({local_sys_id})`
- the `sha` if the file already existed (for overwrite), or omitted if it's a new file

### 7. Log the result

It logs the HTTP status code and file name via `gs.log()` for visibility in the system logs (`syslog.list`, Source: `GitExporter`).

The filename is based on the **local Update Set `sys_id`** — not the name — so re-exporting the same Update Set always overwrites the same file regardless of name changes. GitHub's SHA mechanism handles version history under the hood.

---

## What XML format does the file use and why does it matter?

The file uses ServiceNow's standard **`<unload>` format**, which looks like this:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<unload unload_date="2026-03-21 00:00:00">
    <sys_remote_update_set action="INSERT_OR_UPDATE">
        <name>My Update Set</name>
        <state>loaded</state>
        <application_scope>global</application_scope>
        ...
    </sys_remote_update_set>
    <sys_update_xml action="INSERT_OR_UPDATE">
        <sys_id>abc123...</sys_id>
        <name>sys_script_include_abc123</name>
        <type>Script Include</type>
        <target_name>UpdateSetGitExporter</target_name>
        <payload>...actual record XML...</payload>
        <remote_update_set>def456...</remote_update_set>
        ...
    </sys_update_xml>
    <sys_update_xml action="INSERT_OR_UPDATE">
        ...
    </sys_update_xml>
</unload>
```

**Why it matters**

ServiceNow's Retrieved Update Sets mechanism — the Load → Preview → Commit flow — expects exactly this format when it parses an imported XML file. It looks for the `<unload>` root element, reads the `<sys_remote_update_set>` block for metadata, and iterates over `<sys_update_xml>` children to reconstruct each record. If the wrapper is wrong or missing, the import either fails silently or creates an empty/broken record.

The XML is assembled by `_buildUnloadXml()` using `_serializeRecordWithOrder()`, which reads field values dynamically from GlideRecord. Fields not listed in the ordered arrays are automatically appended at the end, giving partial resilience to future ServiceNow version upgrades that add new fields.

---

## What does the deploy-to-uat.yml workflow do exactly?

Here's exactly what `deploy-to-uat.yml` does from top to bottom:

**Trigger** — it fires on any `push` to the repo that touches a file matching `update_sets/**.xml` — so it only runs when an actual Update Set XML lands in that folder, not on unrelated commits.

### Step 1 — Checkout

`actions/checkout@v3` with `fetch-depth: 2` pulls the repo. The `fetch-depth: 2` is important — without it, `git diff HEAD~1 HEAD` fails on the very first commit because there's no parent to diff against. This was a real bug that caused silent failures early in the project.

### Step 2 — Detect changed XML files

```bash
git diff --name-only HEAD~1 HEAD | grep 'update_sets/.*\.xml' > changed_files.txt
```

This isolates only the XML files that actually changed in this push, so the workflow doesn't re-deploy every file in the folder on every run — just the ones that were added or modified.

### Step 3 — Deploy each changed file to UAT

For each file in `changed_files.txt` it:

1. Constructs the **GitHub Raw URL** for that file — e.g. `https://raw.githubusercontent.com/jeannjang/servicenow-update-sets/main/update_sets/06a9d0f793633250e502fbf08bba109d.xml`
2. Sends a `POST` to the UAT Scripted REST API (`/api/1790158/update_set_importer/import`) with that URL as the `github_url` payload, authenticated via `UAT_SN_USERNAME` and `UAT_SN_PASSWORD` from GitHub Secrets
3. Checks the HTTP response code — exits with failure if it's not 200/201

The key design decision here is sending the **Raw URL rather than the XML content itself**. Earlier attempts at base64-encoding the full XML into the request body ran into payload size limits, escaping complexity, and instability. Having the UAT instance fetch the file directly from GitHub is simpler and more reliable.

### Step 4 — Summary

Prints a confirmation message with the target instance URL.

The three GitHub Secrets it depends on are `UAT_SN_INSTANCE`, `UAT_SN_USERNAME`, and `UAT_SN_PASSWORD` — all registered under the repo's Settings → Secrets and variables → Actions.

---

## How does the UAT Scripted REST API fetch and load the XML?

Here's what the UAT Scripted REST API (`/api/1790158/update_set_importer/import`) does when it receives a request from GitHub Actions:

### 1. Receive the Raw URL

The endpoint accepts a `POST` with a JSON body containing the GitHub Raw URL:

```json
{
  "github_url": "https://raw.githubusercontent.com/jeannjang/servicenow-update-sets/main/update_sets/06a9d0f793633250e502fbf08bba109d.xml"
}
```

### 2. Fetch the XML from GitHub

It uses `sn_ws.RESTMessageV2` to make an outbound GET request to that Raw URL:

```javascript
var rm = new sn_ws.RESTMessageV2();
rm.setEndpoint(githubRawUrl);
rm.setHttpMethod('GET');
rm.setRequestHeader('Accept', 'application/xml');
var xmlContent = rm.execute().getBody();
```

This is why the Raw URL approach was chosen over sending the XML directly — the UAT instance pulls the file itself, keeping the GitHub Actions request payload tiny and avoiding any Base64/escaping complexity.

### 3. Parse the XML

It parses the fetched XML using ServiceNow's `XMLDocument2`:

```javascript
var xmlDoc = new XMLDocument2();
xmlDoc.parseXML(xmlContent);
```

Fields are accessed via full XPath expressions:
```javascript
xmlDoc.getNodeText('//sys_remote_update_set/name')
xmlDoc.getNodeText('//sys_update_xml[1]/name')
xmlDoc.getNodeText('//sys_update_xml[2]/type')
// ...and so on by index
```

This was a significant troubleshooting point — `getNodes().size()` and `node.getNodeText()` both came back `undefined`. The working approach turned out to be using full XPath-style access directly on the `xmlDoc` object, with index-based iteration for repeated `sys_update_xml` nodes.

### 4. Validate application scope

Before inserting, it looks up the `application_scope` value from the XML in the UAT instance's `sys_scope` table:

```javascript
var scopeGR = new GlideRecord('sys_scope');
scopeGR.addQuery('scope', applicationScope);
scopeGR.query();
if (!scopeGR.next()) {
    // scope not found → log error and abort import
    gs.logError('Scope not found: ' + applicationScope + '. Import aborted.', 'UpdateSetImporter');
    response.setStatus(400);
    return;
}
```

If the required scope doesn't exist on the UAT instance, the import is aborted and logged — there is no fallback to `global`. This prevents silently importing an Update Set into the wrong application scope.

### 5. Check for duplicates

Before inserting, it checks whether a `sys_remote_update_set` with the same `remote_sys_id` already exists:

```javascript
var existing = new GlideRecord('sys_remote_update_set');
existing.addQuery('remote_sys_id', remoteSysIdFromXml);
existing.query();
if (existing.next()) {
    // already imported → return existing record, skip insert
}
```

### 6. Insert into `sys_remote_update_set`

It creates a new GlideRecord on `sys_remote_update_set` and populates it with values parsed from the XML:

```javascript
var remoteUs = new GlideRecord('sys_remote_update_set');
remoteUs.initialize();
remoteUs.setValue('name', remoteName);
remoteUs.setValue('state', 'loaded');
remoteUs.setValue('description', description);
remoteUs.setValue('application', scopeGR.getValue('sys_id'));
remoteUs.setValue('application_name', applicationName);
remoteUs.setValue('application_scope', applicationScope);
remoteUs.setValue('origin_sys_id', originSysId);
remoteUs.setValue('remote_sys_id', remoteSysIdFromXml);
// ...
var remoteSysId = remoteUs.insert();
```

This is what makes the Update Set appear under **Retrieved Update Sets** in the UAT PDI.

### 7. Insert `sys_update_xml` records

It iterates through each `<sys_update_xml>` block using index-based XPath and inserts individual records:

```javascript
var i = 1;
var updateName = xmlDoc.getNodeText('//sys_update_xml[' + i + ']/name');
while (updateName) {
    var updateXml = new GlideRecord('sys_update_xml');
    updateXml.initialize();
    updateXml.setValue('name', updateName);
    updateXml.setValue('type', xmlDoc.getNodeText('//sys_update_xml[' + i + ']/type') || '');
    updateXml.setValue('payload', xmlDoc.getNodeText('//sys_update_xml[' + i + ']/payload') || '');
    updateXml.setValue('remote_update_set', remoteSysId);
    // ...
    updateXml.insert();
    i++;
    updateName = xmlDoc.getNodeText('//sys_update_xml[' + i + ']/name');
}
```

### 8. Return a response

It returns a summary of what was imported:

```json
{
  "status": "success",
  "message": "Update set imported successfully",
  "remote_update_set_sys_id": "...",
  "name": "My Update Set",
  "imported_updates": 2,
  "reused_existing": false
}
```

---

## What is sys_remote_update_set and what gets inserted there?

`sys_remote_update_set` is the ServiceNow table that backs the **Retrieved Update Sets** module — it's where Update Sets from external sources land before they get applied to an instance.

**What it represents**

When you manually go to System Update Sets → Retrieved Update Sets and upload an XML file, ServiceNow creates a record in this table. That record holds the metadata of the incoming Update Set and links to `sys_update_xml` records for each individual change, but hasn't touched the actual instance configuration yet — it's a staging area. The Load → Preview → Commit steps all operate on this record.

**What gets inserted**

When the UAT Scripted REST API runs, it inserts a record with roughly these fields:

| Field | Value |
|-------|-------|
| `name` | Update Set name parsed from the `<unload>` XML |
| `description` | Description from the XML |
| `state` | `loaded` — signals it's ready for Preview |
| `application` | `sys_id` of the matching `sys_scope` record on UAT |
| `application_name` | Application name from the XML |
| `application_scope` | Scope string (e.g. `global`) from the XML |
| `remote_sys_id` | The original local `sys_id` from the DEV instance |
| `origin_sys_id` | Same — used for duplicate detection |

**Why `state: loaded` matters**

The `state` field controls what actions are available in the UI. Setting it to `loaded` on insert means the record immediately shows up as ready to Preview — the reviewer can go straight to Preview and then Commit without any additional manual load step.

**How it differs from `sys_update_set`**

`sys_update_set` is the local table — it holds Update Sets that were *created* on this instance. `sys_remote_update_set` holds Update Sets that *came from somewhere else*. They're kept separate precisely because remote ones need the Preview step to check for conflicts before being committed locally.

**Important distinction from OOB import**

Our pipeline inserts into `sys_remote_update_set` and `sys_update_xml` directly via GlideRecord — the same end state as manually uploading an XML file through the UI. The actual payload application (making Script Includes, Business Rules, etc. appear on the instance) happens later through the standard Preview → Commit flow, which uses the OOB Import Engine. Our code only handles the "get the data into the staging table" step.

---

## What does the Load step do in Retrieved Update Sets?

The Load step is the first of the three manual steps after a record lands in `sys_remote_update_set`, and it's essentially the **XML parsing and unpacking phase**.

**What it does technically**

ServiceNow reads the `<unload>` XML structure and parses each `<sys_update_xml>` child element. For each one, it creates a corresponding record in the `sys_update_xml` table on the UAT instance, linked back to the `sys_remote_update_set` record. These are the individual update records — each one representing a single artifact like a Script Include, Business Rule, field change, and so on.

In our pipeline, this step is effectively already done by the UAT Scripted REST API — it inserts the `sys_update_xml` records directly. So by the time the record appears in Retrieved Update Sets, it is already in `loaded` state and ready for Preview without a separate Load click.

**What it does not do**

It does not apply anything to the instance yet. No Script Includes are created, no Business Rules are modified. It purely unpacks the XML into individual records so Preview has something to work with.

**Why this step exists**

The separation between Load and Preview is deliberate. Load is the cheap mechanical step — just parse and store. Preview is the expensive step — it checks every single unpacked record against what currently exists on the instance and flags conflicts. Keeping them separate means if the XML itself is malformed or corrupt, you find out at Load before wasting time on conflict detection.
