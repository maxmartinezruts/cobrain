# cobrain

**This repo holds no application code, and nothing here is meant to grow into
any.** It is a workspace whose subject is somewhere else: the live *brain*
workspace, reached over HTTP with a workspace API key. The brain **product's**
source lives in `../brain` — if the question is about how a feature is
implemented, that is the repo to open, not this one.

So: no build, no dev server, no tests. Every answer here is a request against a
running deployment, and every write lands on real business data.

## The key

**The split is by what is secret, not by what is configuration.** `$BRAIN_URL` is
a deployment address — the same for everybody, no credential — so it lives in the
committed `.claude/settings.json` alongside the permissions. `$BRAIN_KEY` is the
credential and lives **only** in `.claude/settings.local.json`, which is
gitignored; `.claude/settings.example.json` is that file's committed shape and
holds nothing else. So setup is one paste, from the workspace's **Settings → API
keys** page, and pointing this repo at a different deployment is an optional
`BRAIN_URL` in the local file overriding the shared default.

```bash
curl -s -H "authorization: Bearer $BRAIN_KEY" "$BRAIN_URL/api/m/discover" | jq
```

**The key acts as the member who created it**, with their exact standing. Folder
rules apply, and a folder that member may not read is simply *absent* rather than
refused — an empty tree is not proof the workspace is empty. There is no
read-only key: every key is read **and** write.

## Two calls reach the whole surface

```bash
curl -s -H "authorization: Bearer $BRAIN_KEY" "$BRAIN_URL/api/m/discover"
curl -s -H "authorization: Bearer $BRAIN_KEY" "$BRAIN_URL/api/m/<module>/discover"
```

The first is the **index**: every module this workspace has installed, what it is
for, its `base_url`, and where its own `discover` lives. The second is that
module's operations — method, path, `query_params`, `body`, and a `hint` per
route saying what the thing is *for*.

Both are generated from the same route table the router matches against, so
neither can be stale. **This file therefore lists no modules and no routes**, and
you should not add any: a list written here goes wrong the first time somebody
installs or uninstalls one. Ask the index, every session.

Outside the modules there are two flat routes: `GET /v1/files` (metadata for
every readable document — path, size, mime, sha) and `GET /v1/usage` (model spend
events).

## Rules

- **Every write is live.** No branch, no review, no merge — a `files.commit` or a
  collection mutation is visible to everyone the moment it returns. **Confirm
  with the user before any POST, PATCH or DELETE**, quoting the exact path or
  record the call will touch. Deletes have no undo except `files` history.
- **Never hardcode a module slug, a base URL or a route** into a script or a
  note. The index is the only authority on what exists.
- **Prefer `search` and `tree` to enumerating.** The file store and the
  collections are large — thousands of records. `list`-shaped routes answer
  `matched` (how many the filter actually hit) and `capped` (the scan stopped
  early, so the page may be short); read both before reporting a number, or a
  cap becomes a wrong answer stated confidently.
- **A path is spelled exactly as the tree spells it.** Don't guess casing,
  don't invent `company/` prefixes, don't reconstruct a path from a search
  snippet — copy it from the response.
- **Accounting's `/v1/discover` documents the wrong auth.** It claims
  `Bearer lk_live_…` and a `POST /v1/bootstrap`; both are inherited from Ledger
  and neither exists here. Use the brain key, like every other module.
- **`files/file` returns text only.** An image, a PDF or an office file is not
  readable through it — say so rather than describing a file you never saw.

## Working here

- **Pipe through `jq`.** A discover response is thousands of characters; dumping
  one raw costs the context that was going to hold the answer.
  `| jq '.modules[].module'` first, then narrow.
- **Findings belong in the workspace, not in this repo.** If the user wants notes
  kept, commit them through the files module — that is the durable, shared,
  version-tracked place, and it is the thing they will actually open again. A
  local `.md` here is a note nobody else can see. Scratch files while you work
  are fine; just don't mistake one for the deliverable.
- **Report what the API said.** If a call 4xx'd, quote the status and body rather
  than retrying variations until something returns 200 — a shape you had to guess
  at is usually the wrong route.
