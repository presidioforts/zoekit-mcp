Got it—fastest path to get Zoekt running and expose a Python REST API around it.

1) Install & index (single VM)

# deps (Ubuntu/Debian)
sudo apt-get update && sudo apt-get install -y git curl build-essential universal-ctags

# Go (if you don’t already have it)
# https://go.dev/dl — ensure Go ≥ 1.21 installed and on PATH

# Fetch Zoekt tools
go install github.com/sourcegraph/zoekt/cmd/zoekt-git-index@latest
go install github.com/sourcegraph/zoekt/cmd/zoekt-index@latest
go install github.com/sourcegraph/zoekt/cmd/zoekt-webserver@latest

# Create an index dir and index a repo
mkdir -p ~/.zoekt
$GOPATH/bin/zoekt-git-index -index ~/.zoekt /path/to/your/repo

# Run the webserver with JSON API enabled
$GOPATH/bin/zoekt-webserver -index ~/.zoekt -rpc -listen :6070
# UI:  http://localhost:6070
# JSON: POST http://localhost:6070/api/search

Zoekt’s README shows these exact commands and the -rpc flag exposing a simple JSON API at /api/search. It also links the query syntax doc (e.g., panic file:*.py etc.). 


---

2) (Optional) “Push”-indexing service

If you want to push repos to be indexed over HTTP instead of running CLI per host, Zoekt includes a dynamic index server you can POST to. Example flow (ports are illustrative): build/run an indexserver, then:

# Example call to ask the indexer to clone+index a repo
curl -X POST http://127.0.0.1:6060/index \
  -H 'Content-Type: application/json' \
  -d '{"CloneUrl":"https://gitlab.com/gitlab-org/gitlab-elasticsearch-indexer.git","RepoId":2953390}'

That POST shape and port come from a working reference used by GitLab’s chart while the binary itself is documented as “a server to manage dynamic indexing” on pkg.go.dev. 

> For larger fleets you can also run zoekt-indexserver to mirror/index from GitHub orgs on a schedule (token+config). 




---

3) Minimal Python REST wrapper (FastAPI)

This wraps Zoekt’s JSON API so your apps only talk to your Python service. It passes through any JSON body to /api/search (so you aren’t locked to a subset of options).

# app.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import Any, Dict, Optional
import os, httpx

ZOEKT_URL = os.getenv("ZOEKT_URL", "http://localhost:6070")  # zoekt-webserver -rpc
INDEXER_URL = os.getenv("INDEXER_URL", "http://localhost:6060")  # if using dynamic indexer

app = FastAPI(title="Zoekt REST Proxy")

class SearchPayload(BaseModel):
    # Pass-through to Zoekt's /api/search; include at least {"Query": "your query"}.
    payload: Dict[str, Any]

@app.post("/search")
async def search(body: SearchPayload):
    try:
        async with httpx.AsyncClient(timeout=60) as client:
            r = await client.post(f"{ZOEKT_URL}/api/search", json=body.payload)
            r.raise_for_status()
            return r.json()
    except httpx.HTTPError as e:
        raise HTTPException(status_code=502, detail=str(e))

@app.get("/healthz")
async def healthz():
    # Trivial probe
    return {"ok": True}

# If you use the dynamic index server for “push” indexing:
class IndexReq(BaseModel):
    clone_url: str
    repo_id: Optional[int] = None  # your internal id if you use one

@app.post("/index")
async def index_repo(req: IndexReq):
    try:
        payload = {"CloneUrl": req.clone_url}
        if req.repo_id is not None:
            payload["RepoId"] = req.repo_id
        async with httpx.AsyncClient(timeout=120) as client:
            r = await client.post(f"{INDEXER_URL}/index", json=payload)
            r.raise_for_status()
            return {"enqueued": True}
    except httpx.HTTPError as e:
        raise HTTPException(status_code=502, detail=str(e))

Run it:

pip install fastapi uvicorn httpx
ZOEKT_URL=http://localhost:6070 INDEXER_URL=http://localhost:6060 \
uvicorn app:app --host 0.0.0.0 --port 8000

Usage:

# Basic query (Zoekt string query)
curl -X POST localhost:8000/search -H 'content-type: application/json' \
  -d '{"payload":{"Query":"panic file:*.go"}}'

Zoekt’s README confirms /api/search is available when started with -rpc and supports additional options (BM25, context lines, streaming). You can pass those straight through in payload. 


---

4) Containerize (quick way)

git clone https://github.com/sourcegraph/zoekt.git
cd zoekt
docker build -f Dockerfile.webserver -t zoekt-web .
docker build -f Dockerfile.indexserver -t zoekt-indexer .

mkdir -p /data/zoekt/index /data/zoekt/repos

# web (JSON API on 6070)
docker run --rm -p 6070:6070 \
  -v /data/zoekt/index:/data/index \
  zoekt-web -index /data/index -rpc -listen :6070

# (optional) dynamic indexer (HTTP on 6060)
# consult your chosen indexer’s readme for flags

Dockerfiles are in the repo; volumes are mounted under /data. 

> OpenShift/K8s: run each as a Deployment + PVC. The official Dockerfiles run as non-root; set a securityContext with a writable volume at /data. (That matches the Dockerfiles’ expectations.) 




---

5) Notes you’ll care about

Query language: rich filters like file:, boolean ops, regex, etc. 

Auth: Zoekt webserver has no auth; put it behind your gateway/ingress with mTLS or an auth proxy.

Symbols: installing universal-ctags improves ranking/symbol signals. 

Scale: for org-wide search, use zoekt-indexserver (mirror/schedule) or zoekt-dynamic-indexserver (push). 


If you want, I can drop a minimal Docker Compose or OpenShift YAML next that pairs zoekt-webserver (+ -rpc) with the Python proxy and a PVC for /data/index.

