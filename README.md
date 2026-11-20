# Obsidian Vault MCP (free, no card, no always-on computer needed)

Gives Claude (mobile, desktop, web) read and write access to an Obsidian vault
stored in a GitHub repo, through a small server hosted free on Render. Your
devices sync the vault to GitHub with the Obsidian Git plugin, so nothing has
to stay powered on.

## Pieces
- **Vault repo** (GitHub, private): the source of truth for your notes.
- **Obsidian Git plugin**: commits, pushes and pulls from phone and desktop.
- **This server**: talks to the vault repo through GitHub's Contents API (no
  local clone) and exposes 15 MCP tools. Deployed free on Render.
- **Graphify** (optional): a GitHub Actions workflow in the vault repo that
  builds a knowledge graph the server's graph tools read.

## Tools (15)
`list_notes`, `read_note`, `write_note`, `append_note`, `delete_note`,
`rename_note`, `search_notes`, `search_by_tag`, `get_backlinks`,
`get_frontmatter`, `list_folders`, `get_daily_note`, `graph_overview`,
`graph_find_node`, `graph_neighbors`.

`write_note` replaces the whole file: read first, then write back the full
updated content. Use `append_note` for pure additions. `delete_note` is a real,
permanent git commit, so confirm before deleting anything.

## Using a different vault (or sharing this with someone)
Nothing about a specific vault is in the code. Which vault it reads comes only
from environment variables on Render. To point it at another vault, change
those, redeploy, then disconnect and reconnect the connector in Claude so it
refreshes its tool list.

Anyone else who wants this should deploy **their own copy** with their own
token and secret. Never share your `GITHUB_TOKEN` or `MCP_SHARED_SECRET`;
either gives write access to your vault.

| Variable | Required | What it does |
|---|---|---|
| `GITHUB_TOKEN` | yes | Fine-grained token, Contents: Read and write, on the vault repo only |
| `GITHUB_OWNER` | yes | GitHub username (or org) that owns the vault repo |
| `GITHUB_REPO` | yes | Vault repo name |
| `GITHUB_BRANCH` | no | Branch to use (default `main`) |
| `MCP_SHARED_SECRET` | no | If set, every request to `/mcp` must send `Authorization: Bearer <secret>` |

## 1. Create the vault repo
1. GitHub → New repository → name it (e.g. `my-vault`) → **Private**.
2. Settings → Developer settings → Personal access tokens → **Fine-grained
   token**. Repository access: only this repo. Permissions: **Contents →
   Read and write**. Copy the token for Render.

## 2. Wire up Obsidian Git (desktop + phone)
1. Install **Obsidian Git** (Settings → Community plugins).
2. Clone the repo as your vault, or `git init` in place and add it as `origin`.
3. Turn on auto commit-and-sync on an interval and auto pull on load.
4. Set **Merge strategy on conflicts** to **Theirs** on every device. With
   "None", a device's stale local commit can silently overwrite writes made
   through this server.
5. **Keep `.obsidian/` out of the repo.** Add `.obsidian/` to `.gitignore`
   before the first push. If it's already tracked, run
   `git rm -r --cached .obsidian` and commit. Tracked plugin and workspace
   files cause "local changes would be overwritten by checkout" errors on
   phones.

## 3. Deploy the server to Render (free)
1. Push this folder (`server.js`, `package.json`, `render.yaml`) to its own
   GitHub repo, separate from the vault repo.
2. render.com → New → Web Service → connect that repo. It should pick up
   `render.yaml`; otherwise use build `npm install`, start `npm start`, plan
   **Free**.
3. Add the environment variables from the table above.
4. Deploy. You get a URL like `https://<name>.onrender.com`.

Free tier: the service sleeps after about 15 minutes idle and takes a few
seconds to wake on the next request. Deploying is just pushing to this repo's
`main` branch.

## 4. Connect it to Claude
Claude → Settings → Connectors → Add custom connector, URL:
`https://<your-app>.onrender.com/mcp`

About `MCP_SHARED_SECRET`: the server only checks an `Authorization: Bearer`
header. The claude.ai custom connector form has not always offered a header
field. If yours doesn't, **leave `MCP_SHARED_SECRET` unset** on Render, or the
server will reject Claude with 401. In that case the URL is the only barrier,
so keep it private. If your client does let you add the header, set the secret
on both sides. After setting or changing it, test with a read (`list_notes`).

Claude caches the tool list per connection, so reconnect the connector after
any deploy that adds or changes tools.

## 5. Graphify (optional knowledge graph)
Graphify builds a graph from your notes using an LLM. The setup that fits a
no-card account under 18 is **OpenRouter's free tier** (Google's Gemini API
requires being 18+).

1. Get an OpenRouter API key and add it to the **vault repo** under Settings →
   Secrets and variables → Actions as `OPENROUTER_API_KEY`.
2. Vault repo → Settings → Actions → General → Workflow permissions → **Read
   and write**.
3. Add `.github/workflows/graphify.yml` to the vault repo (see
   `vault-workflow/.github/workflows/graphify.yml`). It runs daily, on pushes
   to `main`, and manually from the Actions tab.

The workflow must:
- install `graphifyy[openai]` (the bare package lacks the `openai` dependency),
- run `graphify . --backend openai --obsidian`, **then**
  `graphify cluster-only .` (the second step builds `GRAPH_REPORT.md`; skipping
  it gives a green run with no report),
- commit `graphify-out/` back to the repo.

Output lands in `graphify-out/` (`graph.json`, `GRAPH_REPORT.md`) and the
server's three graph tools read it from GitHub. Free OpenRouter models are
pulled and swapped often, so if a run fails with 402, 404 or an empty
response, change the model in the workflow. Graphify is a nice-to-have, not
something the rest of the vault depends on.

## Troubleshooting
- **Some writes return 403 while reads work.** Seen on full-note rewrites and
  on content containing code-formatted text or literal error messages. It
  looks triggered by that content, not by size alone, and the GitHub token and
  Render logs were ruled out. Workaround: write in smaller `append_note`
  chunks and paraphrase error text or commands instead of quoting them.
- **A note reverted to an older version.** A device's pending local commit
  pushed over the new version after pulling it. Set merge strategy to
  **Theirs** on every device and check the repo's commit history to see what
  happened.
- **401 from the server.** `MCP_SHARED_SECRET` is set on Render but the
  connector isn't sending the header. Unset it or add the header.
- **Tools missing after a deploy.** Disconnect and reconnect the connector.

## Local test (optional)
```
npm install
GITHUB_TOKEN=xxx GITHUB_OWNER=you GITHUB_REPO=my-vault npm start
```
Then POST to `http://localhost:3000/mcp` with an MCP client, or deploy and test
through Claude.
