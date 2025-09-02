Perfect—VM-first is the fastest way to land this. Here’s a tight, copy-paste plan to MVP Zoekt + a thin search shim + MCP wrapper on a single VM, then containerize later.

A. Target VM (single box)

OS: Ubuntu 22.04 LTS (or similar)

Sizing: 4–8 vCPU, 8–16 GB RAM, 100–200 GB disk (indexes grow ~2–3× repo size)

Open ports: 6070 (Zoekt), 8080 (shim/MCP), or keep localhost only if IDE runs on same host

B. Install Zoekt on the VM (15–20 min)
# deps
sudo apt update
sudo apt install -y golang-go git ripgrep universal-ctags jq

# Go env
echo 'export GOPATH=$HOME/go' >> ~/.bashrc
echo 'export PATH=$PATH:$HOME/go/bin' >> ~/.bashrc
source ~/.bashrc

# Zoekt binaries
go install github.com/sourcegraph/zoekt/cmd/zoekt-git-index@latest
go install github.com/sourcegraph/zoekt/cmd/zoekt-webserver@latest

# Index storage
sudo mkdir -p /opt/zoekt/index
sudo chown -R $USER:$USER /opt/zoekt

# Index a repo (repeat per repo; use your Git URL)
zoekt-git-index -index /opt/zoekt/index \
  -name my-monorepo https://git.example.com/team/my-monorepo.git

# Run webserver with JSON API
nohup zoekt-webserver -index /opt/zoekt/index -rpc -listen 0.0.0.0:6070 >/tmp/zoekt.log 2>&1 &

(Optional) systemd units

/etc/systemd/system/zoekt-web.service

[Unit]
Description=Zoekt Webserver
After=network-online.target
[Service]
ExecStart=/home/ubuntu/go/bin/zoekt-webserver -index /opt/zoekt/index -rpc -listen :6070
User=ubuntu
Restart=always
[Install]
WantedBy=multi-user.target


/etc/systemd/system/zoekt-index.service

[Unit]
Description=Reindex Zoekt
[Service]
Type=oneshot
ExecStart=/home/ubuntu/go/bin/zoekt-git-index -index /opt/zoekt/index -name my-monorepo https://git.example.com/team/my-monorepo.git
User=ubuntu


/etc/systemd/system/zoekt-index.timer

[Unit]
Description=Reindex every 30 min
[Timer]
OnBootSec=5m
OnUnitActiveSec=30m
Unit=zoekt-index.service
[Install]
WantedBy=timers.target

sudo systemctl daemon-reload
sudo systemctl enable --now zoekt-web.service zoekt-index.timer

Sanity check
curl -s http://localhost:6070/api/search \
  -H 'content-type: application/json' \
  -d '{"q":"\"NullPointerException\" file:\\.java$", "num":10, "num_context_lines":20}' | jq '.Results | length'

C. Thin HTTP search shim (fastest integration)

This avoids MCP complexity on day 1. It turns Zoekt JSON into prompt-ready slices.

mkdir -p ~/zoekt-shim && cd ~/zoekt-shim
npm init -y
npm i undici express


shim.js

import express from "express";
import { fetch } from "undici";

const app = express();
app.use(express.json());
const ZOEKT = process.env.ZOEKT_URL || "http://127.0.0.1:6070";

app.post("/context", async (req, res) => {
  try {
    const { query, limit = 20, context_lines = 20, file_glob, repo } = req.body || {};
    if (!query) return res.status(400).json({ error: "query required" });

    const parts = [query];
    if (file_glob) parts.push(`file:${file_glob}`);
    if (repo) parts.push(`repo:${repo}`);
    const q = parts.join(" ");

    const r = await fetch(`${ZOEKT}/api/search`, {
      method: "POST",
      headers: { "content-type": "application/json" },
      body: JSON.stringify({ q, num: limit, num_context_lines: context_lines }),
    });
    if (!r.ok) return res.status(502).json({ error: `Zoekt ${r.status}` });
    const data = await r.json();

    const items = (data?.Results ?? []).flatMap(r =>
      (r.FileMatches ?? []).flatMap(fm =>
        (fm.LineMatches ?? []).map(lm => ({
          repo: r.Repository || "",
          path: fm.FileName || "",
          startLine: Math.max(1, (lm.LineNumber || 1) - context_lines),
          endLine: (lm.LineNumber || 1) + context_lines,
          snippet: [...(lm.ContextBefore ?? []), lm.Line ?? "", ...(lm.ContextAfter ?? [])].join("\n"),
          why: `match @ line ${lm.LineNumber || "?"}`,
          score: lm.Score ?? fm.Score ?? r.Score ?? 0
        }))
      )
    );

    // rank + dedupe
    const key = x => `${x.path}:${x.startLine}`;
    const seen = new Set();
    const out = items
      .sort((a,b)=> (b.score??0)-(a.score??0))
      .filter(x => (seen.has(key(x)) ? false : (seen.add(key(x)), true)))
      .slice(0, limit);

    res.json({ query: q, items: out });
  } catch (e) {
    res.status(500).json({ error: String(e) });
  }
});

app.get("/healthz", (_req, res) => res.send("ok"));
app.listen(8080, () => console.log("shim on :8080"));


Run:

ZOEKT_URL=http://127.0.0.1:6070 node shim.js


Example call:

curl -s localhost:8080/context -H 'content-type: application/json' \
  -d '{"query":"sym:^PaymentService\\.process$ or \"NullPointerException\"","file_glob":"\\.java$","limit":10,"context_lines":24}' | jq .


Use this in your editor/agent prompts immediately.

D. MCP wrapper (add after shim works)

Once the shim is solid, expose the same logic as an MCP tool (so MCP-capable IDEs can call it directly). Minimal TS sketch (same VM):

mkdir -p ~/mcp-zoekt && cd ~/mcp-zoekt
npm init -y
npm i @modelcontextprotocol/sdk undici zod


mcp-zoekt.js (Node ESM; very small wrapper around the same fetch)

import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { jsonSchema } from "@modelcontextprotocol/sdk/schema.js";
import { fetch } from "undici";

const ZOEKT = process.env.ZOEKT_URL || "http://127.0.0.1:6070";
const server = new Server({ name:"mcp-zoekt", version:"0.1.0" }, { capabilities:{ tools:{} } });

server.tool("zoekt.search", {
  description: "Search code via Zoekt and return prompt-ready snippets.",
  inputSchema: jsonSchema({
    type:"object",
    properties:{
      query:{type:"string"},
      limit:{type:"number"},
      context_lines:{type:"number"},
      file_glob:{type:"string"},
      repo:{type:"string"}
    },
    required:["query"]
  }),
  async handler({ query, limit=20, context_lines=20, file_glob, repo }) {
    const parts=[query]; if(file_glob) parts.push(`file:${file_glob}`); if(repo) parts.push(`repo:${repo}`);
    const q=parts.join(" ");
    const r = await fetch(`${ZOEKT}/api/search`, {
      method:"POST", headers:{ "content-type":"application/json" },
      body: JSON.stringify({ q, num: limit, num_context_lines: context_lines })
    });
    const data = await r.json();
    const items=(data?.Results??[]).flatMap(r =>
      (r.FileMatches??[]).flatMap(fm =>
        (fm.LineMatches??[]).map(lm => ({
          repo:r.Repository||"", path:fm.FileName||"",
          startLine: Math.max(1,(lm.LineNumber||1)-context_lines),
          endLine: (lm.LineNumber||1)+context_lines,
          snippet:[...(lm.ContextBefore??[]),lm.Line??"",...(lm.ContextAfter??[])].join("\n"),
        }))
      )
    ).slice(0,limit);
    return { items, meta:{ query:q } };
  }
});

server.start();  // stdio MCP server


Run:

ZOEKT_URL=http://127.0.0.1:6070 node mcp-zoekt.js


Point your MCP-capable client at this executable. (If your client wants HTTP/SSE transport only, keep using the shim in §C for now.)

E. Sprint plan (2 weeks, VM-only)

Day 1 – VM, ports, Zoekt install, index one repo, search works via curl.
Day 2 – Add systemd timer for reindex; index 2–3 more repos.
Day 3 – Stand up shim (/context) and verify prompt-ready slices.
Day 4 – Hook your IDE/agent to call the shim; run golden queries (stack traces, symbols).
Day 5 – MCP wrapper (stdio) online; test with any MCP client.
Week 2 – Harden: auth (reverse proxy/JWT), request timeouts, query/log audit, backup /opt/zoekt/index. Performance tune (no CPU limits, ensure fast SSD).

Exit criteria

P95 search < 200 ms on common queries.

/context returns top-K snippets with correct lines + 20–40 context lines.

Reindex every 30 min; incremental runs < 2–5 min.

IDE can fix at least 2–3 real bugs using Zoekt-fed context.

F. When you’re ready to containerize

Keep exactly the same binaries and paths; just wrap them.

Zoekt webserver (multi-stage)

# build
FROM golang:1.22 AS build
RUN go install github.com/sourcegraph/zoekt/cmd/zoekt-webserver@latest
# run (non-root)
FROM gcr.io/distroless/static:nonroot
COPY --from=build /go/bin/zoekt-webserver /usr/local/bin/zoekt-webserver
USER 65532:65532
EXPOSE 6070
ENTRYPOINT ["zoekt-webserver","-index","/data/index","-rpc","-listen",":6070"]


Indexer job image = same pattern but with zoekt-git-index.
Shim/MCP image = node:20-alpine, copy app, run as non-root, expose 8080, add basic healthcheck.

If you want, I’ll tailor the exact commands for your Git server auth (SSH key vs token), and give you Nginx snippets to put both endpoints behind a single /zoekt prefix with JWT.

You said:
We are python house 
ChatGPT said:
Thought for 1m 17s

perfect — we’ll do this Python-first.

TL;DR

Keep Zoekt as-is (Go binary).

Put a tiny Python FastAPI shim in front of it for prompt-ready snippets.

Wrap the same call in a Python MCP server (FastMCP) so Continue/Claude/etc. can use it.

(Zoekt exposes a JSON API when started with -rpc; supports context lines, etc. 
GitHub
Go Packages
. MCP has an official Python SDK and Python quickstart. 
Model Context Protocol
+1
GitHub
)

1) Zoekt on the VM (quick)
# deps
sudo apt update && sudo apt install -y golang-go git ripgrep universal-ctags
# install zoekt tools
go install github.com/sourcegraph/zoekt/cmd/zoekt-git-index@latest
go install github.com/sourcegraph/zoekt/cmd/zoekt-webserver@latest
# index & serve
mkdir -p /opt/zoekt/index
$HOME/go/bin/zoekt-git-index -index /opt/zoekt/index -name repo1 https://git.example.com/repo1.git
nohup $HOME/go/bin/zoekt-webserver -index /opt/zoekt/index -rpc -listen :6070 >/tmp/zoekt.log 2>&1 &


Zoekt JSON API is then at http://<vm>:6070/api/search. 
GitHub

2) Python FastAPI shim (prompt-ready context)
pip install fastapi uvicorn httpx pydantic

# shim.py
import os, httpx
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel

ZOEKT = os.getenv("ZOEKT_URL", "http://127.0.0.1:6070")
app = FastAPI()

class SearchReq(BaseModel):
    query: str
    limit: int = 20
    context_lines: int = 20
    file_glob: str | None = None   # e.g. "\\.py$"
    repo: str | None = None        # e.g. "^my-monorepo$"

@app.post("/context")
async def context(req: SearchReq):
    parts = [req.query]
    if req.file_glob: parts.append(f"file:{req.file_glob}")
    if req.repo: parts.append(f"repo:{req.repo}")
    payload = {
        "q": " ".join(parts),
        "num": max(1, min(req.limit, 100)),
        "num_context_lines": max(0, min(req.context_lines, 200)),
    }
    try:
        async with httpx.AsyncClient(timeout=5.0) as cli:
            r = await cli.post(f"{ZOEKT}/api/search", json=payload)
    except Exception as e:
        raise HTTPException(502, f"Zoekt unreachable: {e}")
    if r.status_code != 200:
        raise HTTPException(502, f"Zoekt HTTP {r.status_code}")
    data = r.json()

    # Flexible parse of Zoekt results => prompt-ready slices
    out = []
    for res in (data.get("Results") or []):
        repo = res.get("Repository", "")
        for fm in res.get("FileMatches", []):
            path = fm.get("FileName", "")
            for lm in fm.get("LineMatches", []):
                ln = lm.get("LineNumber", 1)
                before = "\n".join(lm.get("ContextBefore", []))
                line = lm.get("Line", "")
                after = "\n".join(lm.get("ContextAfter", []))
                out.append({
                    "repo": repo,
                    "path": path,
                    "startLine": max(1, ln - req.context_lines),
                    "endLine": ln + req.context_lines,
                    "snippet": "\n".join([before, line, after]).strip(),
                    "why": f"match @ line {ln}",
                    "score": lm.get("Score", 0)
                })
    # rank + dedupe
    seen, dedup = set(), []
    for item in sorted(out, key=lambda x: x["score"], reverse=True):
        k = (item["path"], item["startLine"])
        if k in seen: continue
        seen.add(k); dedup.append(item)
        if len(dedup) >= req.limit: break
    return {"query": payload["q"], "items": dedup}


Run:

ZOEKT_URL=http://127.0.0.1:6070 uvicorn shim:app --host 0.0.0.0 --port 8080
# test:
curl -s localhost:8080/context -H 'content-type: application/json' \
  -d '{"query":"sym:^PaymentService\\.process$ or \"KeyError\"","file_glob":"\\.py$","limit":10,"context_lines":24}'


(Zoekt supports regex/boolean queries, file: & sym: filters; num_context_lines returns surrounding lines. 
GitHub
Go Packages
)

3) Python MCP server (FastMCP)
pip install mcp httpx

# mcp_zoekt.py
import os, httpx
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("mcp-zoekt")
ZOEKT = os.getenv("ZOEKT_URL", "http://127.0.0.1:6070")

@mcp.tool()
def zoekt_search(query: str, limit: int = 20, context_lines: int = 20,
                 file_glob: str | None = None, repo: str | None = None):
    """Search code via Zoekt and return prompt-ready snippets."""
    parts = [query]
    if file_glob: parts.append(f"file:{file_glob}")
    if repo: parts.append(f"repo:{repo}")
    body = {"q": " ".join(parts), "num": min(max(limit,1),100),
            "num_context_lines": min(max(context_lines,0),200)}
    r = httpx.post(f"{ZOEKT}/api/search", json=body, timeout=5.0)
    r.raise_for_status()
    data = r.json()

    items = []
    for res in (data.get("Results") or []):
        repo_name = res.get("Repository","")
        for fm in res.get("FileMatches",[]):
            path = fm.get("FileName","")
            for lm in fm.get("LineMatches",[]):
                ln = lm.get("LineNumber",1)
                snippet = "\n".join(( "\n".join(lm.get("ContextBefore",[])),
                                      lm.get("Line",""),
                                      "\n".join(lm.get("ContextAfter",[])) )).strip()
                items.append({"repo": repo_name, "path": path,
                              "startLine": max(1, ln - context_lines),
                              "endLine": ln + context_lines,
                              "snippet": snippet})
    return {"items": items[:limit], "meta": {"query": body["q"]}}

if __name__ == "__main__":
    mcp.run()  # stdio transport


FastMCP is the official Python path for MCP servers; it autogenerates tool schemas. 
Model Context Protocol
+1

In Continue, add this MCP server in config (command + args) per their docs. 
Continue

4) Minimal 2-week VM sprint (Python-only work)

Day 1–2: Zoekt installed; one repo indexed; JSON API reachable. 
GitHub

Day 3–4: FastAPI shim live (/context) and used by your editor agent.

Day 5: Python MCP server (zoekt_search) working over stdio. 
Model Context Protocol

Week 2: Reindex timer, auth in front of Zoekt (JWT/proxy), timeouts/limits, golden queries.

When you’re ready to containerize, lift-and-shift these same Python services; the Zoekt bits remain identical. If you want, I can also add a tiny pytest suite that checks query→snippet integrity end-to-end.

Refs: Zoekt -rpc JSON API & context lines; MCP Python SDK & quickstarts; Continue MCP config. 
GitHub
