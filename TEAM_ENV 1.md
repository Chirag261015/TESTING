# Advise Workbench – team environment settings

> **CONFIDENTIAL – live credentials.** Share only through a private channel with Advise Workbench team members.
> Do not commit this file, paste it into tickets or chat channels, or forward it outside the team.

## How to use

1. Copy the block below into `Advise-Workbench/.env` (replace the whole file).
2. Make sure `config/shared-services-ca.crt` is present (the shared services use it for TLS).
3. Start the app with `./scripts/run_agent.sh`.

All values are common to the whole team, including secrets and tokens.
Windows paths for LibreOffice and Tesseract assume the default winget install locations; change them if yours differ.
For what each setting does, see `.env.example`.

```env
JWT_SECRET=7pFU-RhYwY3-pn_0AWm0dOVWIK0RnIddOoRH4Ycq-aU
AUTH_ALLOW_SELF_SIGNUP=true
WORKSPACE_ROOT=./workspace
LLM_PROVIDER=claude_cli
LLM_MODEL=claude-haiku-4-5                      # required in every mode; claude-sonnet-5-5 is slower but stronger
ANTHROPIC_CLAUDE_MODEL=claude-haiku-4-5
ANTHROPIC_TEMPERATURE=0.2
ANTHROPIC_MAX_TOKENS=4096
QDRANT_URL=https://172.31.65.20:6333
QDRANT_COLLECTION_PREFIX=aw_shared_
S3_ENDPOINT_URL=https://172.31.65.20:8333
S3_BUCKET=aw-shared
EMBEDDING_BASE_URL=https://172.31.65.20:8080
EMBEDDING_MODEL=BAAI/bge-small-en-v1.5
OPENOBSERVE_ENABLED=true
OPENOBSERVE_URL=https://172.31.65.20:5080
OPENOBSERVE_ORG=default
OPENOBSERVE_USER=shared@aw.local
OPENOBSERVE_STREAM=aw_shared
OTEL_EXPORTER_OTLP_ENDPOINT=https://172.31.65.20:4318
SSL_CA_FILE=config/shared-services-ca.crt
OTEL_EXPORTER_OTLP_CERTIFICATE=config/shared-services-ca.crt
OTEL_RESOURCE_ATTRIBUTES=aw.tenant=shared
REDIS_URL=rediss://:8d44784ea48dcbcb742cb3f59e55550ee8839d7d@172.31.65.20:6380/1?ssl_cert_reqs=required&ssl_ca_certs=config/shared-services-ca.crt
CELERY_BROKER_URL=amqps://aw_shared:08ae5bb3ce55d62d6987c67aa16e17d6baa1ccea@172.31.65.20:5671/aw_shared
RUN_STUCK_TIMEOUT_SEC=1800
ANTHROPIC_MAX_TOKENS_PER_RUN=0
DATABASE_URL=postgresql+psycopg://aw_shared:48682468273b0c6ef44fb1af35938bdb9a3fb676@172.31.65.20:5432/aw_shared?sslmode=verify-full&sslrootcert=config/shared-services-ca.crt
S3_CA_FILE=config/shared-services-ca.crt
WORKSPACE_BACKEND=db_s3
RUN_CHECKPOINT_STORE=postgres
OPENOBSERVE_PASSWORD=Aw!9fb44f4f4938e9539d6c
SOFFICE_BINARY_PATH=C:\Program Files\LibreOffice\program\soffice.com
TESSERACT_CMD=C:\Program Files\Tesseract-OCR\tesseract.exe
QDRANT_API_KEY=ef5c6aa0f20238bc301641525c1febf03b7a6597e850bc5b306f65af769b99de
S3_ACCESS_KEY_ID=33F357331E8E9FDA
S3_SECRET_ACCESS_KEY=aed83291ee5490f43b775c01e3007ac7312cc33bea99128f7e05c41df1fbc6e9
EMBEDDING_API_KEY=388b8bcd25c1f039ba14936d77ddb9ef7ba2175d37654f204f47f015695076c7
OTEL_EXPORTER_OTLP_HEADERS=authorization=Bearer%203bb995aba91b6fc48e802e9dba21cd760ec23912e3f32e6bf4f18d8a734b4559
WIKI_LP_ADMIN_EMAILS=sgadwala@deloitte.com,lomurugan@deloitte.com,cjain21@deloitte.com,prgupta19@deloitte.com
```
