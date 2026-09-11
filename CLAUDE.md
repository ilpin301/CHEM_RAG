# CHEM_RAG

German-language LightRAG base for general and inorganic chemistry (Allgemeine Anorganische Chemie).
Sources are lecture exercises, worked solutions and harvested papers, so filenames carry umlauts
(`Musterlösung`, `Übung`) - every path must move through UTF-8, never the ANSI codepage.

## RAG Ingest Rules
- **One SOURCE per ingest run.** If `IN\` has several new files, ingest them one at a time:
  full cycle per source (probe, launch, verify, error-correction pass, cleanup, ledger, memory),
  and only then start the next. Never batch several sources into one list file - a failure then
  cannot be attributed and the delete/re-ingest repair costs minutes per document. Slices of ONE
  source still go in ONE run. A failed or lossy source STOPS the queue; report and wait.
- **After each source, `rag_sync.ps1 push` before the next one launches.** The per-source
  cleanup is not done until that push succeeds. If `J:` is missing, Google Drive is not
  running - start `GoogleDriveFS.exe` (newest folder under
  `C:\Program Files\Google\Drive File Stream`), wait for `J:\My Drive` to appear, then push.
  Native PowerShell only. A failed push stops the queue, same as a failed ingest.
- Always launch ingests via the ragkit launcher, detached, never inline:
  `& $env:RAGKIT_HOME\ingest.ps1 -Root X:\RAG_MAIN\CHEM_RAG -ListFile <utf8 list>`. It produces
  `lightrag\LOG\ingest_run.log`, which progress checks depend on, and it owns `docker compose`
  `stop`/`start` - do not stop the container yourself. This base has no launcher of its own.
- `-ListFile` is mandatory and always UTF-8. Never pass PDF paths as bare process arguments: the
  German filenames in this base mangle through the ANSI codepage and the ingest silently targets
  the wrong file.
- Delete/wipe `ingest_run.log` ONLY after the run reports `EXITCODE=0`, never before the run starts.
  On success the launcher archives it as `ingest_run.<stamp>.ok.log`; that archive is the input to
  the error-correction pass and must survive until the pass reports zero permanent losses.
- Probe every PDF before routing: use full image detection (embedded images AND vector figures),
  not just `page.get_images()`. Image-heavy → MinerU/VLM path; text-only → text path.
- Slice PDFs over 10 pages before ingest; delete slice PDFs only after all slices are confirmed
  processed.
- Record every ingested SOURCE in `lightrag\INGESTED_SOURCES.txt`, not the slice names.
- Verify every ingest with: doc count delta, graph node delta, vector sanity check, one targeted
  query, and the permanent-loss grep over the archived log.

## This base
- Server `chem_rag-lightrag-1`, port 9621, `COMPOSE_PROJECT_NAME=chem_rag` in `lightrag\.env`.
- The tooling lives in the ragkit repo, not here: `rag_ingest.py`, `ingest_merged.py`,
  `check_vectors.py`, `ingest_triage.py` and the launcher are all under `$env:RAGKIT_HOME`
  (`X:\RAG_MAIN\RAG`). The RAG skills are user-level too. Nothing RAG-specific is version-controlled
  in this repo except the compose file, `IN\`, `FOUND\`, the ledger, `keepawake.ps1` and
  `rag_sync.ps1`.
- Unlike MECH_RAG, `lightrag\` here is NOT a clone of hkuds/lightrag. It is a plain data directory:
  `data\`, `LOG\`, `.env`, `docker-compose.yml`, `INGESTED_SOURCES.txt`, `keepawake.ps1`.
- The store (`lightrag\data\rag_storage\`) is NOT in git. It is backed up to Google Drive with
  `rag_sync.ps1`. Never `git add -A` here.
- `rag_sync.ps1 push` must run from a native PowerShell. Invoked from Git Bash, `tar` resolves
  to the msys build, which reads `C:\...` as a remote host and fails with
  `Cannot connect to C: resolve failed`. Git Bash also cannot see the `J:` Drive mount at all -
  a `ls` there returning nothing does NOT mean Drive is down; check from PowerShell.

## Vector storage: Qdrant
- Vectors are NOT under `lightrag\data\rag_storage`. They live in a docker named volume served by
  `chem_rag-qdrant-1` on `127.0.0.1:6353` (6333 is PCM_RAG's, 6343 is taken).
  `LIGHTRAG_VECTOR_STORAGE=QdrantVectorDBStorage` in `lightrag\.env`.
- **Host-side URLs must be `127.0.0.1`, never `localhost`.** localhost resolves to `::1` first and
  the failed IPv6 attempt costs ~2 s per request - it looks like slow throughput, not a DNS problem.
- Do NOT stop the whole compose project to ingest. `ingest.ps1` stops only the `lightrag` service.
- The il-rag-ingest skill's `vdb_*.json` size/mtime check does NOT apply here - there are no such
  files. The launcher's own `VECTOR SANITY CHECK (qdrant)` block is the evidence instead.
- `rag_sync.ps1 push` exports a Qdrant snapshot per collection into `lightrag\data\qdrant_snapshots\`
  and tars it alongside `rag_storage`; `pull` uploads them back. A backup taken any other way
  contains no vectors.

## Baseline (2026-09-11)
First ingest: `Musterlösung zur Übung 2.pdf` (10 pages, 19 images, MinerU route). 1 document,
`processed`, 72 chunks; graph 899 nodes / 3735 edges; Qdrant collections chunks 72 / entities 899 /
relationships 3735, all green, 0 nonfinite, 0 zero vectors. Zero permanent VLM losses in the run log.
