# cobrain

**This repo holds no application code, and nothing here is meant to grow into
any.** It is a workspace whose subject is somewhere else: the live *cobrain*
workspace, reached over HTTP with a workspace API key. The cobrain **product's**
source lives in `../brain` — if the question is about how a feature is
implemented, that is the repo to open, not this one.

So: no build, no dev server, no tests. Every answer here is a request against a
running deployment, and every write lands on real business data.

## The keys — one per workspace

**The split is by what is secret, not by what is configuration.** `$COBRAIN_URL` is
a deployment address — the same for everybody, no credential — so it lives in the
committed `.claude/settings.json` alongside the permissions. The keys are
credentials and live **only** in `.claude/settings.local.json`, which is
gitignored; `.claude/settings.example.json` is that file's committed shape.

A key belongs to **one workspace**, and this repo may hold several: one variable
per workspace, `COBRAIN_KEY_<WORKSPACE>`, the suffix being the name the user calls
it by (`COBRAIN_KEY_ACME`, `COBRAIN_KEY_GLOBEX`). Adding a workspace is one more
line: a `cobrain_…` key minted on that workspace's **Your account → Your API
keys** page (`/w/<slug>/account`).

There is still only one `$COBRAIN_URL` — `https://api.cobrain.ch` in production.
Every workspace lives on the same deployment, and the key alone tells the server
which workspace a request is for. Pointing the whole repo at another deployment
(a dev one, say) is an optional `COBRAIN_URL` in the local file — which is why
commands always write `$COBRAIN_URL`, never the literal address.

**Pick the workspace before the first call, every session.** List what is
configured — names only, never print a value:

```bash
env | cut -d= -f1 | grep '^COBRAIN_KEY_' | sed 's/^COBRAIN_KEY_//'
```

- The user named one (any casing, `-`/space for `_`) and it is in the list: use it.
- Exactly one is configured: use it, and say which.
- Otherwise — none named, a name that isn't in the list, or a request that could
  mean more than one — **ask**, offering the configured names. Don't guess from
  the content of the question.
- Nothing configured: tell the user to add a `COBRAIN_KEY_<WORKSPACE>` line to
  `.claude/settings.local.json`; don't go looking for a key elsewhere.

Once chosen, it holds for the session until the user switches. Name it in every
answer that touches data, and **name it again in every write confirmation** — the
same path can exist in two workspaces. Spell the variables out in each call (the
shell does not keep state between calls):

```bash
curl -s -H "authorization: Bearer $COBRAIN_KEY_ACME" "$COBRAIN_URL/api/m/discover" | jq
```

There is only one kind of key: it is bound to **one workspace** and acts as **one
person**. **The key acts as the member who created it**, with their exact standing,
and it stops working if that person leaves the workspace. Folder
rules apply, and a folder that member may not read is simply *absent* rather than
refused — an empty tree is not proof the workspace is empty. There is no
read-only key: every key is read **and** write.

## Two calls reach the whole surface

`<WS>` below is the chosen workspace's suffix.

```bash
curl -s -H "authorization: Bearer $COBRAIN_KEY_<WS>" "$COBRAIN_URL/api/m/discover"
curl -s -H "authorization: Bearer $COBRAIN_KEY_<WS>" "$COBRAIN_URL/api/m/<module>/discover"
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
  and neither exists here. Use the cobrain key, like every other module.
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
