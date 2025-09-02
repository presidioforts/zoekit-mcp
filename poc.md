perfect — we’ll do this **Python-first**.

# TL;DR

* Keep Zoekt as-is (Go binary).
* Put a **tiny Python FastAPI shim** in front of it for prompt-ready snippets.
* Wrap the same call in a **Python MCP server (FastMCP)** so Continue/Claude/etc. can use it.

(Zoekt exposes a JSON API when started with `-rpc`; supports context lines, etc. ([GitHub][1], [Go Packages][2]). MCP has an official Python SDK and Python quickstart. ([Model Context Protocol][3], [GitHub][4]))

---

# 1) Zoekt on the VM (quick)

```bash
# deps
sudo apt update && sudo apt install -y golang-go git ripgrep universal-ctags
# install zoekt tools
go install github.com/sourcegraph/zoekt/cmd/zoekt-git-index@latest
go install github.com/sourcegraph/zoekt/cmd/zoekt-webserver@latest
# index & serve
mkdir -p /opt/zoekt/index
$HOME/go/bin/zoekt-git-index -index /opt/zoekt/index -name repo1 https://git.example.com/repo1.git
nohup $HOME/go/bin/zoekt-webserver -index /opt/zoekt/index -rpc -listen :6070 >/tmp/zoekt.log 2>&1 &
```

Zoekt JSON API is then at `http://<vm>:6070/api/search`. ([GitHub][1])

---

# 2) Python FastAPI **shim** (prompt-ready context)

```bash
pip install fastapi uvicorn httpx pydantic
```

```python
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
```

Run:

```bash
ZOEKT_URL=http://127.0.0.1:6070 uvicorn shim:app --host 0.0.0.0 --port 8080
# test:
curl -s localhost:8080/context -H 'content-type: application/json' \
  -d '{"query":"sym:^PaymentService\\.process$ or \"KeyError\"","file_glob":"\\.py$","limit":10,"context_lines":24}'
```

(Zoekt supports regex/boolean queries, `file:` & `sym:` filters; `num_context_lines` returns surrounding lines. ([GitHub][1], [Go Packages][2]))

---

# 3) **Python MCP server** (FastMCP)

```bash
pip install mcp httpx
```

```python
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
```

* FastMCP is the official Python path for MCP servers; it autogenerates tool schemas. ([Model Context Protocol][5])
* In Continue, add this MCP server in config (command + args) per their docs. ([Continue][6])

---

# 4) Minimal 2-week VM sprint (Python-only work)

* **Day 1–2:** Zoekt installed; one repo indexed; JSON API reachable. ([GitHub][1])
* **Day 3–4:** FastAPI shim live (`/context`) and used by your editor agent.
* **Day 5:** Python MCP server (`zoekt_search`) working over stdio. ([Model Context Protocol][5])
* **Week 2:** Reindex timer, auth in front of Zoekt (JWT/proxy), timeouts/limits, golden queries.

When you’re ready to containerize, lift-and-shift these same Python services; the Zoekt bits remain identical. If you want, I can also add a tiny **pytest** suite that checks query→snippet integrity end-to-end.

**Refs:** Zoekt `-rpc` JSON API & context lines; MCP Python SDK & quickstarts; Continue MCP config. ([GitHub][1], [Go Packages][2], [Model Context Protocol][3], [Continue][6])

[1]: https://github.com/sourcegraph/zoekt?utm_source=chatgpt.com "sourcegraph/zoekt: Fast trigram based code search"
[2]: https://pkg.go.dev/github.com/sourcegraph/zoekt?utm_source=chatgpt.com "zoekt package - github.com/sourcegraph ..."
[3]: https://modelcontextprotocol.io/docs/sdk?utm_source=chatgpt.com "SDKs"
[4]: https://github.com/modelcontextprotocol/python-sdk?utm_source=chatgpt.com "The official Python SDK for Model Context Protocol servers ..."
[5]: https://modelcontextprotocol.io/quickstart/server?utm_source=chatgpt.com "Build an MCP Server"
[6]: https://docs.continue.dev/customize/deep-dives/mcp?utm_source=chatgpt.com "How to Set Up Model Context Protocol (MCP) in Continue"
