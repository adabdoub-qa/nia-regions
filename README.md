# NIA Region Compliance Scanner (`nia_scan.py`)

> AI-generated. Review and verify before sharing externally.

A **read-only** Cloud Shell script that lists every Google Cloud resource deployed **outside the NIA-certified regions** and produces a report grouped by category: **Virtual Machines, Disks & Snapshots, Databases, Data Analytics, Object Storage, GKE, Serverless, AI/ML, Networking, Logging, Security, Backup, and DevOps**.

* **Default allowed regions:** `me-central1` (Doha), `europe-west1` (Belgium), `europe-west4` (Netherlands). Every zone in those regions also counts, for example `me-central1-a` and `europe-west4-b`.
* **Strict matching:** `europe-west10`, `europe-west12`, and `me-central2` are correctly flagged as **non-compliant**.
* **Multi-region / dual-region locations** (`us`, `eu`, `europe`, `asia`, `nam4`, `eur4`, and so on) are flagged as non-compliant.
* **Zero install:** needs only Python 3 and `gcloud`, both preinstalled in Cloud Shell. It never creates, changes, or deletes resources.

---

## 1. Prerequisites (one-time)

| Requirement | Why | Command / Role |
| :--- | :--- | :--- |
| Cloud Asset API enabled in **one** project (the "quota project") | Discovery engine | `gcloud services enable cloudasset.googleapis.com --project=<QUOTA_PROJECT>` |
| `roles/cloudasset.viewer` on the scan scope (organization, folder, or project) | Search all resources | Grant via IAM at the organization level for a full-estate scan |
| `roles/compute.viewer` and `roles/cloudsql.viewer` (or `roles/viewer`) on affected projects | Optional VM / Cloud SQL detail enrichment (machine type, GPUs, DB engine/tier) | Skip with `--no-enrich` |
| `resourcemanager.projects.list` | Maps project numbers to project IDs | Included in `roles/browser` / `roles/viewer` |

## 2. Run it in Cloud Shell

1. Open [Cloud Shell](https://shell.cloud.google.com).
2. Upload `nia_scan.py` (Cloud Shell menu **⋮ → Upload**).
3. Set the quota project:
   ```bash
   gcloud config set project <QUOTA_PROJECT_ID>
   ```
4. Run one of:
   ```bash
   # Whole organization (recommended): find your org ID with `gcloud organizations list`
   python3 nia_scan.py --organization <ORG_ID>

   # One or more folders
   python3 nia_scan.py --folder <FOLDER_ID>[,<FOLDER_ID>]

   # Specific projects
   python3 nia_scan.py --projects prj-a,prj-b

   # No scope flag: every ACTIVE project your account can list
   python3 nia_scan.py

   # Strict mode: only Doha + Belgium are allowed
   python3 nia_scan.py --organization <ORG_ID> --allowed-regions me-central1,europe-west1

   # Also list global resources for review, and export a full inventory (compliant + non-compliant)
   python3 nia_scan.py --organization <ORG_ID> --include-global --include-compliant
   ```
5. Download the report:
   ```bash
   cloudshell download nia_report_<timestamp>/nia_report.html
   ```

## 3. Output

The script creates a folder `nia_report_<UTC timestamp>/` containing:

| File | Content |
| :--- | :--- |
| `nia_report.html` | Summary cards, **summary by project**, **summary by non-compliant location**, then **one section per category**. Each row shows project ID, resource name, full resource name, type, location, state, and details. VMs include machine type, status, GPUs, provisioning model, and disks. Cloud SQL includes engine, tier, availability, and disk. The page has a live filter box. |
| `nia_findings.csv` | Every finding, ready for Google Sheets or Excel |
| `nia_findings.json` | Machine-readable findings plus scan metadata |

The console also prints a per-category summary. **Exit code** `0` means everything is compliant, `2` means non-compliant resources were found, and `1` means a fatal error. This makes the script usable in CI or scheduled checks.

The **"Scan warnings / errors (coverage gaps)"** section lists every scope or API call that failed, such as missing permissions or a disabled API. Resources in those scopes **may be missing** from the report, so fix them and re-run.

## 4. All options

```text
--organization IDS      Organization ID(s), comma-separated
--folder IDS            Folder ID(s), comma-separated
--projects IDS          Project ID(s), comma-separated
--allowed-regions LIST  Default: me-central1,europe-west1,europe-west4
--quota-project ID      Project used for Cloud Asset API quota (passed as --billing-project)
--all-asset-types       Search every asset type, including BigQuery tables (slower)
--include-global        Add a review section for global / unspecified-location resources
--include-compliant     Also write compliant resources to CSV/JSON
--no-enrich             Skip VM / Cloud SQL enrichment calls
--parallel N            Parallel gcloud calls (default 6)
--timeout SECONDS       Timeout per gcloud call (default 1800)
--max-console-rows N    Rows printed per category on screen (default 25)
--output-dir DIR        Output folder
```

## 5. How it works

1. **Pre-flight:** checks the active `gcloud` account and the quota project, and confirms the Cloud Asset API is enabled. If it isn't, the script prints the exact enable command.
2. **Discovery:** runs `gcloud asset search-all-resources --scope=<org|folder|project>` with a curated list of asset types covering more than 60 services. BigQuery is searched at **dataset** level, because tables inherit the dataset location. If the API rejects an asset type in your environment, that type is skipped automatically and logged under coverage gaps.
3. **Classification:** applies strict region/zone matching against the allowed list, entirely client-side.
4. **Enrichment:** for projects with non-compliant VMs or Cloud SQL instances, runs `gcloud compute instances list` and `gcloud sql instances list` to add sizing details.
5. **Reporting:** writes HTML, CSV, and JSON.

## 6. Known limitations (read before acting on the report)

* **API usage is not a resource.** Calls to *global* or non-NIA API endpoints are not visible to Cloud Asset Inventory, because they create no resource. Examples: Vertex AI / Gemini called with `location="global"` or `us-central1`, Model Garden partner models, Text-to-Speech, Document AI `us` processors. Check these in the **Cloud Billing export** (filter by `location.region`) or in application code.
* **`_Default` / `_Required` log buckets:** log buckets **are** reported with their location. Logs routed by organization-level sinks may also land in another project's bucket, so review sink destinations.
* **Global resources** (VPC networks, firewall rules, global load balancers, IAM) have no region and are shown only with `--include-global`. Global external load balancers terminate traffic at Google's edge, so review whether this matters for your NIA control set.
* **Point-in-time:** the report reflects the moment of the scan. Pair it with the Organization Policy `constraints/gcp.resourceLocations` (allowed values `in:me-central1-locations`, `in:europe-west1-locations`, `in:europe-west4-locations`) to **prevent** new non-compliant resources.
* Cloud Asset Inventory supports most, not all, Google Cloud resource types. See [Supported asset types](https://cloud.google.com/asset-inventory/docs/supported-asset-types).

## 7. Next steps after the scan

For each row in the report: **delete** it if unused (for example old snapshots, test VMs, idle GPU notebooks), or **migrate** it to an allowed region. Typical paths:

* **VMs:** machine image → recreate in an allowed zone.
* **Cloud SQL:** Database Migration Service or export/import.
* **GCS:** Storage Transfer Service to a regional bucket.
* **Snapshots:** regional snapshot schedule with `--storage-location=me-central1`.
* **Logs:** regional log bucket plus updated `_Default` sink.

## 8. Self-test (for maintainers)

```bash
python3 -m unittest discover -s tests -v
```
The test suite covers location edge cases and category mapping. It also includes **static guardrails** that fail if canned payloads, simulated output, or mutating `gcloud` verbs appear in the scanner.
