Yes—if Zoekt has indexed your repo, you can search essentially any stack trace (Java, Python, Node, .NET) and pull back the exact code slices around each frame. The trick is: parse frames → build precise Zoekt queries (`sym:`, `file:`, exact error text) → request context lines.

Here’s a minimal Python helper that turns a raw stack trace into Zoekt queries and hits your FastAPI `/context` shim:

```python
import re, json, httpx

ZOEKT_CTX = "http://localhost:8080/context"  # your FastAPI shim

EXT_BY_LANG = {"java": r"\.java$", "py": r"\.py$", "js": r"\.(js|ts|tsx)$", "cs": r"\.cs$"}

def build_queries_from_stacktrace(s: str, repo_regex: str | None = None, context_lines: int = 24, limit: int = 12):
    queries = []

    # 1) Error names / phrases (generic boost)
    errs = set(re.findall(r'([A-Z][A-Za-z]+Exception|Error: [^\n:]+)', s))
    for e in errs:
        queries.append({"query": f"\"{e}\"", "file_glob": None})

    # 2) Java: at pkg.Class.method(File.java:123)
    for pkg, cls, meth, file_, line in re.findall(r'at\s+([\w.$]+)\.([\w$<>]+)\(([^:()]+):(\d+)\)', s):
        sym = f"{cls}\\.{meth}".replace("$", r"\$")
        q = f"sym:^{sym}$ or \"{cls}\""
        queries.append({"query": q, "file_glob": EXT_BY_LANG["java"]})

    # 3) Python: File "/path/mod.py", line 42, in func
    for file_, line, func in re.findall(r'File\s+"([^"]+)",\s+line\s+(\d+),\s+in\s+([\w<>]+)', s):
        base = file_.split("/")[-1]
        q = f"file:{base}$ or sym:^{func}$"
        queries.append({"query": q, "file_glob": EXT_BY_LANG["py"]})

    # 4) Node/JS: at func (/path/file.js:10:5) or at /path/file.js:10:5
    for func, file_, line in re.findall(r'at\s+(?:([\w$.<>]+)\s+\()?(.*?\.js|\.ts|\.tsx):(\d+)', s):
        base = file_.split("/")[-1]
        q = f"file:{base}$" + (f" or sym:^{func}$" if func else "")
        queries.append({"query": q, "file_glob": EXT_BY_LANG["js"]})

    # 5) .NET: at Ns.Class.Method(...) in C:\path\file.cs:line 123
    for ns, cls, meth, file_, line in re.findall(r'at\s+([\w.]+)\.([\w`]+)\(.*\)\s+in\s+(.*?\.cs):line\s+(\d+)', s):
        q = f"sym:^{cls}\\.{meth}$ or file:{file_.split('/')[-1]}$"
        queries.append({"query": q, "file_glob": EXT_BY_LANG["cs"]})

    # Attach repo, limits
    for q in queries:
        q["repo"] = repo_regex
        q["context_lines"] = context_lines
        q["limit"] = limit

    # Dedup by (query,file_glob)
    seen, out = set(), []
    for q in queries:
        k = (q["query"], q.get("file_glob"))
        if k in seen: continue
        seen.add(k); out.append(q)
    return out

def search_stacktrace(stacktrace: str, repo_regex: str | None = None):
    results = []
    for q in build_queries_from_stacktrace(stacktrace, repo_regex):
        payload = {k: v for k, v in q.items() if v is not None}
        r = httpx.post(ZOEKT_CTX, json=payload, timeout=8.0)
        r.raise_for_status()
        data = r.json()
        results.extend(data.get("items", []))
    # Rank by simple heuristics (symbol hits first if your shim sets score; else length)
    results = sorted(results, key=lambda x: (-len(x.get("snippet","")), x.get("path","")))
    # Keep top-K overall
    return results[:20]

# Example:
if __name__ == "__main__":
    stack = """
    java.lang.NullPointerException: cannot read field id
        at com.acme.payments.PaymentService.process(PaymentService.java:128)
        at com.acme.web.CheckoutController.submit(CheckoutController.java:54)
    """
    hits = search_stacktrace(stack, repo_regex="^my-monorepo$")
    print(json.dumps(hits, indent=2)[:2000])
```

How this behaves:

* For each frame, it builds a **precise Zoekt query**:

  * `sym:^Class.method$` (symbol search) and/or `file:<basename>$`
  * plus any **error phrase** (e.g., `"NullPointerException"`) to widen recall.
* Sends each query to your `/context` shim, which asks Zoekt for **20–40 context lines** around the hit.
* You get a ranked list of `{repo, path, startLine, endLine, snippet}` ready to drop into your LLM prompt or IDE panel.

Tips to make it reliable:

* Enable **ctags** when indexing so `sym:` queries are strong.
* If runtime paths differ from repo paths, add a **path mapper** (e.g., strip `/workspace/app/` → repo root).
* Keep a fallback: if no symbol match, search **class/file name + error text**.
* Use **repo:** filter when your Zoekt instance indexes multiple repos.

Bottom line: yes—you can point this at your repo and search arbitrary stack traces effectively. This MVP gets you usable results in days; you can refine patterns later (more languages, better ranking, path mappings).
