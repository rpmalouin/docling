# FORK.md — Why this fork exists

**This is `rpmalouin/docling`**, the **engine** fork of upstream
[docling](https://github.com/docling-project/docling) (distribution name `docling-slim`: the Python
SDK and CLI that turns PDFs, Office files, HTML, Markdown, audio, images and XML into a unified
`DoclingDocument`). It is the engine the homelab `docling` stack runs. The request service + React UI
is the sibling fork `/appdata/docling-serve`; the two are combined into one container by
`/appdata/A--docker_stacks/docling/Dockerfile.fork`.

Baseline: our `main` is **upstream `main` at `6fadbae3` (version 2.139.0) plus one documentation
commit** (`e55da31f`). The fork was byte-identical to upstream until 2026-10-14, when Ron
fast-forwarded it to upstream on GitHub (Sync fork, `21588f52` → `6fadbae3`); the local branch was
rebased onto the new upstream. Drift today: **0 behind / 1 ahead** — the only fork commit is docs.

Because the tree tracks upstream that closely, the engine the stack runs is currently one sync
behind the checkout: `dist/` holds the `2.137.0` wheel built from `21588f52`, and the live container
serves it. Rebuilding the wheel is part of deploying an engine change (see Deployment).

## Why a fork

1. **Reproducible engine pin.** The stack installs a wheel built from this checkout, not a PyPI
   artifact: same bytes, rebuildable in place (`uv build --wheel`, ~0.35 s), from a tree we can audit.
   The repo was kept level with upstream precisely so that this pin is cheap to move — the wheel
   deployed today (`dist/docling_slim-2.137.0-py3-none-any.whl`, sha256 `5308bc37…`) was built from
   `21588f52`, five commits past upstream's 2.137.0 release, so it carries fixes the published
   `docling-slim==2.137.0` wheel does not; a fresh build of the current HEAD is
   `docling_slim-2.139.0-py3-none-any.whl` (sha256 `ace0f584…`). A plain `pip install docling-slim==…`
   would be equivalent — and would remove every hook for engine work.
2. **A home for engine-side changes.** Anything that must change below the serve layer — backends,
   pipelines, models, datamodel — belongs here. The serve fork never patches the engine in place;
   it only calls it.
3. **Agent tooling travels with the checkout.** `.code-review-graph/` (self-ignoring graph DB) and
   `MEMORY.md` (gitignored operational notes) live here, next to the code they describe.

## Key changes in this fork vs upstream

| Area | Change | Commits |
|------|--------|---------|
| — | **None yet.** No fork-specific engine change has been made. The wheel installed by the stack is upstream code at `21588f52`. | — |

Nothing in this repository is ours yet; the first commit is documentation (this file plus the
`.gitignore` line that ignores `MEMORY.md`). Keep it that way until a real engine change is needed —
the size of our diff is the cost of the next upstream rebase.

No upstream PRs: these changes are fork-only by convention.

## Deployment (how this fork reaches the running container)

The live stack is `/appdata/A--docker_stacks/docling` (compose is the source of truth; Dockhand
reconciles). The engine arrives as a wheel layer over the digest-pinned serve image:

```sh
uv build --wheel --out-dir /appdata/A--docker_stacks/docling/dist /appdata/docling   # ~0.35 s, reproducible
cd /appdata/A--docker_stacks/docling && docker compose up -d --build docling-serve   # recreates a live container: confirm first
docker exec docling-serve python -c "import importlib.metadata as m; print(m.version('docling-slim'))"   # 2.137.0 today
curl -s http://127.0.0.1:5001/version                                                # no auth needed
```

**Wheel filename trap:** `Dockerfile.fork` line 18 pins the filename
(`COPY dist/docling_slim-2.137.0-py3-none-any.whl /tmp/fork/`) while the install step globs
`docling_slim-*.whl`. Bumping the engine version means editing that one COPY line, or the image build
fails on a missing file.

`/version` reports `importlib.metadata` of the *installed* distribution, not `pyproject.toml` — so it
proves which wheel is installed, and nothing about source content. Do not use it to verify a code
change that the installed wheel does not carry.

Tests run through the built image (there is no host venv, and the image's python is a venv, so
`pip install --user` fails there):

```sh
docker run --rm -v /appdata/docling:/src:ro -w /src --entrypoint sh docling-serve:local -c \
  "pip install -q --target /tmp/td pytest && \
   PYTHONPATH=/src:/tmp/td python -m pytest tests/<file> -q -p no:cacheprovider"
```

## Upstream sync

No upstream remote is configured (only `origin` = this fork). The 2026-10-14 catch-up was done as a
GitHub fast-forward, then rebased locally. Either way works:

```sh
git fetch https://github.com/docling-project/docling main
git rev-list --count HEAD..FETCH_HEAD     # behind
git rebase FETCH_HEAD                     # replay our commits; use --ff-only only while we have none
```

After any sync: rebuild the wheel, recreate the container, re-run the code-review graph
(`build_or_update_graph_tool` with `repo_root=/appdata/docling`) and confirm
`built_at_sha == git rev-parse HEAD`. Expect the version in `pyproject.toml` to move with upstream's
release automation — never hand-edit it (`[tool.semantic_release] version_source = "tag_only"`).

## Repository conventions

- **Remote:** `origin` = `https://github.com/rpmalouin/docling`; branch `main` kept.
- **Git identity:** none configured locally (no global, no repo-local) — commit as
  `Ron Malouin <ron@malouin.com>` per commit.
- **`FORK.md` is tracked** (this file); **`MEMORY.md` is gitignored** — operational notes, never
  upstream content. `.code-review-graph/` self-ignores.
- **`AGENTS.md` governs work here:** ASD-STE100 Simplified Technical English in agent replies,
  `make validate` before a task counts as done, scoped edits, targeted tests, and repo skills routed
  through `.agents/skills/skill-router.json`.
- **Version bumps are upstream automation.** Do not edit `version` in `pyproject.toml`.
- **Verification before pushing:** the image-mounted pytest recipe green for touched tests,
  `uv build --wheel` reproducing the dist artifact, and a live `/version` check if the container was
  rebuilt.
