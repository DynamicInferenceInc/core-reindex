# core-reindex

Docker Compose deployment for `document_indexer` profiles:

- `local-reindex` watches a local folder with `CHUNKING__STRATEGY=table_aware`;
- `local-cv` is the same image with `CHUNKING__STRATEGY=resume_project` and collection `docs-cv`;
- `smb-reindex` mirrors an SMB share into the staging directory mapped in the root `.env`.

`document_indexer` is built as a reusable base image. The profile images
inherit it and only add their own `main.py`. Qdrant and Ollama are not
started by this Compose project.

## Requirements

- Docker with BuildKit;
- Docker Compose 2.17 or newer;
- Qdrant reachable from the Docker host on port 6333;
- Ollama reachable from the Docker host on port 11434;
- the `nomic-embed-text` embedding model and the `qwen3.8:27b` extraction LLM in Ollama
  (resume profiles only).

```bash
docker compose version
ollama pull nomic-embed-text
ollama pull qwen3.8:27b-q8_0   # or qwen3.8:27b if the q8_0 tag is unavailable
```

Target host for the resume profiles is an NVIDIA DGX Spark (GB10, 128 GB unified
memory, ARM64). Build the images on the Spark itself (`docker compose build`);
the base image is multi-arch and torch CPU wheels are resolved for `aarch64`.
Recommended Ollama environment there: `OLLAMA_FLASH_ATTENTION=1`,
`OLLAMA_KEEP_ALIVE=-1`, `OLLAMA_NUM_PARALLEL=1`. The indexer does not use the
GPU; it belongs to Ollama.

On Linux, Compose maps `host.docker.internal` to the Docker host gateway.
Qdrant and Ollama must accept connections from the Docker bridge, not only
from `127.0.0.1`. For Ollama this can be `OLLAMA_HOST=0.0.0.0:11434`.

If image descriptions are enabled later, also install the configured VLM,
which is `qwen3-vl:8b` by default.

## Configuration

There are three env files. Compose interpolates the root file; each
container gets only its own profile file.

```bash
cp .env.example .env
cp local-reindex/.env.example local-reindex/.env
cp local-reindex/.env.cv.example local-reindex/.env.cv
cp smb-reindex/.env.example smb-reindex/.env
```

Root `.env` is for volume mounts only. It is not injected into containers:

```dotenv
LOCAL_DOCS_HOST=./local-reindex/docs
LOCAL_DOCS_CONTAINER=/data/docs
LOCAL_CV_HOST=./local-reindex/cv
LOCAL_CV_CONTAINER=/data/cv
SMB_STAGING_HOST=./smb-reindex/staging
SMB_STAGING_CONTAINER=/data/staging
```

`local-reindex/.env` is the default local indexer (`docs-local`).
`SOURCE__WATCH_PATH` must equal `LOCAL_DOCS_CONTAINER`.

`local-reindex/.env.cv` is the resume local indexer (`docs-cv`,
`CHUNKING__STRATEGY=resume_project`). `SOURCE__WATCH_PATH` must equal `LOCAL_CV_CONTAINER`.

`smb-reindex/.env` is the full environment of the SMB indexer, including
share credentials. `SOURCE__STAGING_PATH` must equal `SMB_STAGING_CONTAINER`.
Fill `SOURCE__SERVER`, `SOURCE__SHARE`, `SOURCE__USERNAME`, `SOURCE__PASSWORD`,
`SOURCE__DOMAIN` and `SOURCE__SUBPATH` before starting that service.
Resume-поля (ФИО, должность, проект, `functional_direction`, `solution_platform`,
`extraction_source`) включаются через `CHUNKING__STRATEGY=resume_project`.
Параметры LLM в `.env.cv.example` / `smb-reindex/.env.example` подобраны под Spark:
`MODELS__EXTRACTION_MODEL=qwen3.8:27b-q8_0`, `MODELS__EXTRACTION_NUM_CTX=65536`,
`MODELS__EXTRACTION_NUM_PREDICT=8192`, `MODELS__EXTRACTION_TIMEOUT_SEC=1800`,
блок `RESUME__*`, `QDRANT__INDEX_VERSION=resume-v20`.

Runtime `.env` files are ignored by git.

## Run

Start the default local profile and the resume local profile:

```bash
docker compose up -d --build local-reindex local-cv
docker compose logs -f local-reindex local-cv
```

Place ordinary documents in `LOCAL_DOCS_HOST` and CVs in `LOCAL_CV_HOST`.
Bind mounts are live; no rebuild is needed for new files.

Start the SMB profile:

```bash
docker compose --profile smb up -d --build smb-reindex
docker compose logs -f smb-reindex
```

Start both:

```bash
docker compose --profile smb up -d --build local-reindex smb-reindex
docker compose logs -f local-reindex smb-reindex
```

`docker compose up -d --build` without service names starts `local-reindex`
and `local-cv`. The SMB service is behind the `smb` profile so it does
not start with placeholder credentials. The `document-indexer` service
has `deploy.replicas: 0`: it stays in the build graph and never runs.

The SMB container does not establish a VPN. Host VPN, DNS and the route
to TCP/445 must work from the Docker bridge.

Stop:

```bash
docker compose down
```

## How the indexer reads settings

`local-reindex/main.py` и `smb-reindex/main.py` вызывают `run()`.
Стратегию задаёт `CHUNKING__STRATEGY`: `table_aware` или `resume_project`.
При `resume_project` библиотека сама подключает resume-чанкер с LLM-шагами
и resume-payload (сервисы `local-cv` и `smb-reindex`).

## Resume pipeline and report

For every CV (`resume_project`):

1. parser — projects from Docling tables / labeled blocks (template CVs);
2. LLM-1 — projects from the text the parser did not understand (when there are
   no parsed projects, or a large leftover with work-history hints);
3. LLM-2 — one call per CV: fill empty fields, set `functional_direction` and
   `solution_platform`; parser values are never overwritten;
4. LLM-3 — when there are no projects at all: `experience` chunks (one per job,
   same six fields) plus one `profile` chunk;
5. `prose` windows with `needs_review=true` only when the LLM is disabled or failed.

Every LLM value is checked against the CV text and dropped if it is not there.

Audits without embeddings / Qdrant: `RESUME_PARSE_ONLY=1` (parser) or
`RESUME_LLM_AUDIT=1` (parser + LLM, also writes `resume_chunks.jsonl`). After each
audit and each full reindex the indexer prints and saves `resume_report.txt/.csv`
(`ФИО | Должность | Проектов | из них LLM | Мест работы | Проверить | Файл`, totals,
files without ФИО/должность) next to the watch/staging directory.

`IndexerSettings` reads process environment
(`SOURCE__WATCH_PATH`, `QDRANT__URL`, `MODELS__EMBEDDING_MODEL`, …)
and, if present, a `.env` file in the current working directory.

Docker Compose `env_file` loads `local-reindex/.env` or `smb-reindex/.env`
into the container environment before `python main.py` starts. The image
does not copy `.env` files, so the process environment is the source of
truth.

`SOURCE__KIND` selects local watchdog vs SMB polling.

## Build graph

```text
document_indexer/Dockerfile
          |
          v
 document-indexer:latest
      /             \
     v               v
local-reindex     smb-reindex
```

No base container is created.

The first `--build` after a Dockerfile change still installs torch/docling.
Later code-only rebuilds should show `CACHED` on those steps. Do not pass
`--no-cache`. Compose sets `provenance: false` so export is not doubled.

## Git submodules

Commit and push each submodule first, then record the new SHAs in
`core-reindex`. Do not push submodule contents through the parent
repository.
