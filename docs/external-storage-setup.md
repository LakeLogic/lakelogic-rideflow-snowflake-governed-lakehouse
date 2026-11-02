# External storage (AWS S3 / Azure / GCP) for the RideFlow landing zone

The project lands files on an internal stage in each domain database
(`RIDEFLOW_DEV_<DOMAIN>.FILES.LANDING`), and Bronze loads them with `COPY INTO`. To keep
landing files in your own bucket instead, point Snowflake at your cloud storage with a
storage integration and an external stage.

> **Not wired in this project.** The steps below set up the Snowflake side. The generated
> runtime still reads the internal stage; switching Bronze to the external stage is not
> built yet.

The same storage also backs **Iceberg** tables later (via an *external volume*) — see
[iceberg-roadmap.md](iceberg-roadmap.md).

Trial accounts block external *access* integrations, but allow storage integrations,
external stages and external tables, so these steps work on a trial.

Two building blocks, per cloud:
1. A **storage integration** (recommended — no secret in SQL) *or* inline stage credentials.
2. An **external stage** over the bucket, then **external tables** the medallion reads.

`CREATE STORAGE INTEGRATION` requires **ACCOUNTADMIN** (or a role with `CREATE INTEGRATION`).

---

## AWS S3

```sql
CREATE STORAGE INTEGRATION s3_int
  TYPE = EXTERNAL_STAGE
  STORAGE_PROVIDER = 'S3'
  ENABLED = TRUE
  STORAGE_AWS_ROLE_ARN = 'arn:aws:iam::<account-id>:role/<snowflake-access-role>'
  STORAGE_ALLOWED_LOCATIONS = ('s3://<bucket>/landing/');

DESC INTEGRATION s3_int;   -- read STORAGE_AWS_IAM_USER_ARN and STORAGE_AWS_EXTERNAL_ID
```
Then in **AWS IAM**: create the role `<snowflake-access-role>` with a **trust policy**
trusting the `STORAGE_AWS_IAM_USER_ARN` and the `STORAGE_AWS_EXTERNAL_ID` from `DESC`, and a
**permissions policy** granting `s3:GetObject`, `s3:GetObjectVersion`, `s3:ListBucket`
(+ `s3:PutObject`, `s3:DeleteObject` if Snowflake also writes) on `arn:aws:s3:::<bucket>/landing/*`.

**Inline-credential alternative** (skip the integration):
```sql
CREATE STAGE landing_ext URL = 's3://<bucket>/landing/'
  CREDENTIALS = (AWS_KEY_ID = '...' AWS_SECRET_KEY = '...');   -- or AWS_TOKEN for temp creds
```
**You need:** the bucket and region, and either the IAM role ARN (integration) or an
access key and secret (inline).

---

## Azure (Blob / ADLS Gen2)

```sql
CREATE STORAGE INTEGRATION azure_int
  TYPE = EXTERNAL_STAGE
  STORAGE_PROVIDER = 'AZURE'
  ENABLED = TRUE
  AZURE_TENANT_ID = '<tenant-id>'
  STORAGE_ALLOWED_LOCATIONS = ('azure://<account>.blob.core.windows.net/<container>/landing/');

DESC INTEGRATION azure_int;   -- read AZURE_CONSENT_URL and AZURE_MULTI_TENANT_APP_NAME
```
Then: open the **AZURE_CONSENT_URL** and approve; in the **Azure portal** → the container →
**Access Control (IAM)** → assign **Storage Blob Data Contributor** to the app named
`AZURE_MULTI_TENANT_APP_NAME`. (ADLS Gen2 uses `.dfs.core.windows.net` in the URL.)

**Inline-credential alternative** (SAS token):
```sql
CREATE STAGE landing_ext URL = 'azure://<account>.blob.core.windows.net/<container>/landing/'
  CREDENTIALS = (AZURE_SAS_TOKEN = '?sv=...');
```
**You need:** the storage account and container, and either the tenant ID (integration,
then approve consent and grant the role) or a SAS token with read, write, list and delete.

---

## GCP (Google Cloud Storage)

```sql
CREATE STORAGE INTEGRATION gcs_int
  TYPE = EXTERNAL_STAGE
  STORAGE_PROVIDER = 'GCS'
  ENABLED = TRUE
  STORAGE_ALLOWED_LOCATIONS = ('gcs://<bucket>/landing/');

DESC INTEGRATION gcs_int;   -- read STORAGE_GCP_SERVICE_ACCOUNT (an email)
```
Then in **GCP IAM**: grant that service account **Storage Object Viewer** (read) or
**Storage Object Admin** (read+write) on the bucket.

> **GCS has no inline-credential stage** — a GCS external stage **must** use a storage
> integration (unlike S3/Azure). So the integration route is required for GCP.

**You need:** the bucket, with IAM granted to the Snowflake service account.

---

## Common: stage → external table (all clouds)

```sql
CREATE STAGE landing_ext
  URL = '<s3://… | azure://… | gcs://…>/landing/'
  STORAGE_INTEGRATION = <s3_int | azure_int | gcs_int>
  DIRECTORY = (ENABLE = TRUE)
  FILE_FORMAT = (TYPE = CSV SKIP_HEADER = 1 FIELD_OPTIONALLY_ENCLOSED_BY = '"');

-- One external table per landing entity; columns are schema-on-read over the files.
CREATE OR REPLACE EXTERNAL TABLE LANDING.<system>_<entity> (
  "COL1" VARCHAR AS (VALUE:c1::VARCHAR),
  "COL2" VARCHAR AS (VALUE:c2::VARCHAR)
  -- … one per column, positional (c1, c2, …) for CSV
)
  LOCATION = @landing_ext/<system>/<entity>/
  FILE_FORMAT = (TYPE = CSV SKIP_HEADER = 1)
  PATTERN = '.*[.]csv'
  AUTO_REFRESH = TRUE;   -- reflects new files as they land
```
A Bronze contract could then read the external table as a table source instead of
loading files with `COPY INTO`. This project does not do that yet.

- **Auto-refresh** needs cloud event notifications (S3 → SNS/SQS, Azure → Event Grid,
  GCS → Pub/Sub). Without them, run `ALTER EXTERNAL TABLE … REFRESH;` (or on a schedule).
- Parquet works too (`TYPE = PARQUET`, `INFER_SCHEMA` to derive columns).

## Iceberg (later): external volume on the same bucket

```sql
CREATE EXTERNAL VOLUME iceberg_vol
  STORAGE_LOCATIONS = ((
    NAME = 'rideflow'
    STORAGE_PROVIDER = '<S3|AZURE|GCS>'
    STORAGE_BASE_URL = '<s3://…|azure://…|gcs://…>/iceberg/'
    /* S3:  STORAGE_AWS_ROLE_ARN = '…'  |  AZURE: AZURE_TENANT_ID = '…' */
  ));
-- then CREATE ICEBERG TABLE … EXTERNAL_VOLUME = iceberg_vol …;  (see iceberg-roadmap.md)
```

## Uploading the demo files to the bucket

Snowflake `PUT` only targets *internal* stages, so the RideFlow files are uploaded to the
external bucket with the cloud's own tools/SDK, then the external tables read them:
- **AWS:** `aws s3 cp --recursive` / boto3
- **Azure:** `az storage blob upload-batch` / azcopy / `azure-storage-blob`
- **GCP:** `gsutil -m cp -r` / `google-cloud-storage`

The synthetic seed writes to the internal stage only. Seeding an external bucket is not
built.
