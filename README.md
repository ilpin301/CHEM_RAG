# CHEM_RAG

A local GraphRAG knowledge base over German general- and inorganic-chemistry course material
(Allgemeine Anorganische Chemie: Stöchiometrie, Stoffmenge, Reaktionsgleichungen), built on
**LightRAG** + **RAG-Anything** + **MinerU**.

Ask it a question in German, get an answer with formulas and citations back to the source sheet.

## Stack

| Layer | What runs | Where |
|---|---|---|
| Server / Web UI / REST API | LightRAG 1.5.4 | Docker, `http://localhost:9621` |
| Document parsing | RAG-Anything + MinerU | host Python venv (`$env:RAGKIT_HOME\.venv`) |
| LLM + VLM | GLM-5.3 / GLM-4.5V via the Z.ai coding plan | cloud |
| Embeddings | Ollama `bge-m3`, 1024-dim | local |
| KV + graph storage | JSON KV + NetworkX graph | `lightrag/data/rag_storage` |
| Vector storage | Qdrant | docker volume, `127.0.0.1:6353` |

The LightRAG server and the host ingest script write the **same** KV/graph files, so the
`lightrag` service is stopped while ingesting. Qdrant is a server and stays up.

## Contents

```
lightrag/docker-compose.yml     the two services: lightrag + qdrant
lightrag/.env.example           configuration template (no secrets)
lightrag/INGESTED_SOURCES.txt   ledger of ingested SOURCE files, not slice names
lightrag/keepawake.ps1          blocks system sleep for the duration of a long run
rag_sync.ps1                    DB snapshot push/pull via Google Drive
IN/                             curated ingest queue
FOUND/                          harvested source PDFs
CLAUDE.md                       ingest rules and this base's specifics
```

The ingest tooling itself is **not** in this repo. `rag_ingest.py`, `ingest.ps1`,
`check_vectors.py` and `ingest_triage.py` live in the shared ragkit at `$env:RAGKIT_HOME`.

## Quick start

```powershell
Copy-Item lightrag\.env.example lightrag\.env    # fill in LLM_BINDING_API_KEY,
                                                 # VLM_LLM_BINDING_API_KEY, LIGHTRAG_API_KEY
docker compose --project-directory lightrag up -d
```

Then open `http://localhost:9621` and authenticate with `LIGHTRAG_API_KEY`.
Ollama must be running locally (`http://localhost:11434`) before any ingest.

## Ingesting a document

Drop the PDF in `IN\`, then let the shared launcher do the rest:

```powershell
$List = 'lightrag\ingest_list.txt'
Set-Content -LiteralPath $List -Encoding UTF8 -Value (Get-ChildItem IN\*.pdf).FullName
& $env:RAGKIT_HOME\ingest.ps1 -Root $PWD -ListFile $List
```

Use `-ListFile` rather than passing paths as arguments: a filename such as
`Musterlösung zur Übung 2.pdf` is mangled by the ANSI codepage when it crosses a process
boundary, and the ingest then silently targets the wrong file.

The launcher owns `docker compose stop`/`start`, writes `lightrag\LOG\ingest_run.log`, runs the
vector sanity check and archives the log as `ingest_run.<stamp>.ok.log` on success. Verify with a
doc-count delta, a graph-node delta, the Qdrant sanity block, one targeted query, and a grep of the
archived log for permanent VLM losses. See [CLAUDE.md](CLAUDE.md) for the full rules.

## Backup

```powershell
.\rag_sync.ps1 push     # local DB -> J:\My Drive\RAG\CHEM_RAG\rag_storage.tgz
.\rag_sync.ps1 pull     # Drive -> local, full overwrite (old store kept as rag_storage.bak)
```

Run it from native PowerShell, not Git Bash. On this Qdrant base `push` also snapshots each
collection over the API and tars it alongside `rag_storage` - a backup taken any other way
contains no vectors.

## What is not in git

The knowledge-graph store (`lightrag/data/`), the live `.env` with the API keys, and `CHEM.rar`
(29 MB archive of the same corpus already present in `FOUND/`).
