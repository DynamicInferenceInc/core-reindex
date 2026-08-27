# core-reindex

Docker Compose deployment for `document_indexer` profiles:

- `local-reindex` watches a local folder with the default Qdrant payload;
- `local-cv` is the same image with `INDEXER_PROFILE=resume` and collection `docs-cv`;
- `smb-reindex` mirrors an SMB share into the staging directory mapped in the root `.env`.

`document_indexer` is built as a reusable base image. The profile images
inherit it and only add their own `main.py`. Qdrant and Ollama are not
started by this Compose project.

## Requirements

- Docker with BuildKit;
- Docker Compose 2.17 or newer;
- Qdrant reachable from the Docker host on port 6333;
- Ollama reachable from the Docker host on port 11434;
- the `nomic-embed-text` embedding model and `qwen3:8b` extraction LLM in Ollama.

```bash
docker compose version
ollama pull nomic-embed-text
ollama pull qwen3:8b
```

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
`INDEXER_PROFILE=resume`). `SOURCE__WATCH_PATH` must equal `LOCAL_CV_CONTAINER`.

`smb-reindex/.env` is the full environment of the SMB indexer, including
share credentials. `SOURCE__STAGING_PATH` must equal `SMB_STAGING_CONTAINER`.
Fill `SOURCE__SERVER`, `SOURCE__SHARE`, `SOURCE__USERNAME`, `SOURCE__PASSWORD`,
`SOURCE__DOMAIN` and `SOURCE__SUBPATH` before starting that service.
Resume-поля (`project_experiences`) подключаются в `smb-reindex/main.py`.

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

`local-reindex/main.py` читает `INDEXER_PROFILE`. По умолчанию это
`run(IndexerSettings())`. При `INDEXER_PROFILE=resume` подключаются
`ResumePayloadBuilder` и `JsonSchemaEnricher` (сервис `local-cv`).

Resume на шаре по-прежнему в `smb-reindex/main.py` через `ProfileSmb`.

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
