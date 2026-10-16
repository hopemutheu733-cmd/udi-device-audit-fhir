# Surgical Implant UDI Audit API (FHIR)

An n8n workflow that takes a scanned implant barcode (UDI), verifies the device against the public **openFDA UDI database**, returns a standard **FHIR R4 `DeviceUseStatement`**, and writes every scan to a **Supabase** audit log.

Built from a perioperative theatre perspective: in a fast-moving theatre list, implants can go unrecorded. This workflow shows how a barcode scan could be turned into a verified, structured, auditable record in seconds.

> **All data in this project is synthetic.** No real patient information is used. This is a portfolio demo and is **not** a medical device or a clinical system.

---

## Workflow

<img width="600" height="270" alt="Surgical Implant UDI Audit API (FHIR)" src="https://github.com/user-attachments/assets/3f8ee46d-39ae-4483-9e21-9f7cf420198c" />


| Step | Node | What it does |
|---|---|---|
| 1 | **Webhook** (`POST /udi-audit`) | Receives a JSON body containing the scanned UDI |
| 2 | **HTTP Request** | Looks the device up in openFDA (`api.fda.gov/device/udi.json`), with retry on failure |
| 3 | **Device Found?** (If) | Checks that openFDA returned a matching record |
| 4 | **Build FHIR DeviceUseStatement** (Code) | Maps the openFDA record to a FHIR R4 `DeviceUseStatement` with a contained `Device` |
| 5 | **Log Scan** (Supabase) | Saves the scan (UDI, brand, manufacturer, model, status, timestamp) to `device_scans` |
| 6 | **Respond to Webhook** | Returns the FHIR resource to the caller |
| 3b | **Respond Not Found** | If no device matches, returns HTTP `404` with a clear error message |

A separate **error workflow** (Error Trigger → Gmail) emails an alert naming the failed node whenever this workflow fails in production.

---

## Execution history

Every call is recorded in n8n's Executions tab. Successful runs show every node on the happy path completing, and failures are visible and alerted on.

<img width="600" height="270" alt="execution log error" src="https://github.com/user-attachments/assets/974bf79a-33db-4030-aeae-81011bc3b0b8" />


---

## API

**Endpoint**

```
POST /webhook/udi-audit
Content-Type: application/json
```

**Request body**

```json
{
  "udi": "10198459305313"
}
```

**Example request (PowerShell)**

```powershell
Invoke-RestMethod -Method Post -Uri "http://localhost:5678/webhook/udi-audit" -ContentType "application/json" -Body '{"udi":"10198459305313"}'
```

**Example request (curl)**

```bash
curl -X POST http://localhost:5678/webhook/udi-audit \
  -H "Content-Type: application/json" \
  -d '{"udi":"10198459305313"}'
```

**Success response (`200`)** - a FHIR `DeviceUseStatement`

```json
{
  "resourceType": "DeviceUseStatement",
  "contained": [
    {
      "resourceType": "Device",
      "id": "device1",
      "udiCarrier": [
        {
          "deviceIdentifier": "10198459305313",
          "issuer": "urn:oid:1.3.160"
        }
      ],
      "deviceName": [
        { "name": "Medline Industries, Inc.", "type": "user-friendly-name" }
      ],
      "manufacturer": "MEDLINE INDUSTRIES, INC.",
      "modelNumber": "DYNJ911253"
    }
  ],
  "status": "completed",
  "subject": { "reference": "Patient/synthetic-patient-001" },
  "device": { "reference": "#device1" },
  "timingDateTime": "2026-10-16T08:30:52.929Z",
  "recordedOn": "2026-10-16T08:30:52.929Z"
}
```

**Not found response (`404`)**

```json
{
  "error": "Device not found in openFDA",
  "udi": "000000000000"
}
```

An optional `patient_id` field in the request body replaces the default synthetic patient reference.

---

## Audit log

Each successful scan is saved to a Supabase table:

```sql
create table device_scans (
  id bigint generated always as identity primary key,
  scanned_udi text not null,
  brand_name text,
  manufacturer text,
  model_number text,
  status text default 'recorded',
  scanned_at timestamptz default now()
);
```

The workflow connects with a Supabase **service role** key, so that role needs table permissions:

```sql
grant select, insert, update, delete on table public.device_scans to service_role;
grant usage, select on all sequences in schema public to service_role;
```

---

## Tech stack

- **n8n** (self-hosted in Docker) for orchestration
- **openFDA UDI API** for device verification (public, no key required)
- **HL7 FHIR R4** (`DeviceUseStatement`, `Device`) for the output format
- **Supabase (Postgres)** for the audit log
- **Gmail** for failure alerts via an n8n error workflow

---

## Run it yourself

1. **Start n8n in Docker** (the `n8n_data` volume keeps your workflows and credentials):

```bash
   docker volume create n8n_data
   docker run -d --name n8n --restart unless-stopped -p 5678:5678 \
     -e GENERIC_TIMEZONE=Africa/Nairobi -e TZ=Africa/Nairobi \
     -v n8n_data:/home/node/.n8n docker.n8n.io/n8nio/n8n
```

2. **Create the Supabase table** using the SQL above.
3. **Import the workflow**: in n8n choose *Import from file* and select the exported workflow JSON from this repo.
4. **Add credentials** in n8n: a Supabase credential (host and service role key) and a Gmail credential for the error workflow. No secrets are included in the exported JSON.
5. **Publish** the workflow, then call the production webhook URL shown on the Webhook node.
6. Optional: import the error workflow and set it as the **Error Workflow** in this workflow's settings.

---

## Design notes

- **Fail loudly in production.** Failures trigger an email alert with the failed node and a link to the execution.
- **Retry on transient errors.** The openFDA request retries up to 3 times, which covers brief network or DNS failures.
- **Clear error handling.** An unknown device returns a proper `404` instead of an empty or misleading success response.

## Limitations and next steps

- Only the UDI lookup is implemented. There is no PHI redaction, EHR integration or billing link, so this should not be used with real patient data.
- The patient reference is a synthetic placeholder.
- The `Log Scan` node continues on error so the caller still gets a response; a stronger version would alert on logging failures separately.
- Planned improvements: separate alert for logging failures, distinguishing "lookup service unavailable" (`502`) from "device not found" (`404`), request validation, and hosting on a VPS so the endpoint runs 24/7.

---

## About

Built by **Hope Mutheu** - perioperative theatre technology background (HIPAA compliance certificate, medical billing and coding with ICD-10) moving into healthcare AI automation.
