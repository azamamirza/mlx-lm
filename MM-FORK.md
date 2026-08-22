# MM fork of mlx-lm — how this fork is organised

This is `azamamirza/mlx-lm`, our patched fork of `ml-explore/mlx-lm`. The fleet
**installs from this fork**, not from PyPI. That is deliberate: it ends the
practice of hand-editing `site-packages` on every node and makes the
environment reproducible as the fleet grows from 2 to 7+ machines.

## The two-track rule

Every fix lives on **two** branches, and the distinction matters:

| Track | Branch(es) | Purpose | May contain |
|-------|-----------|---------|-------------|
| **Submission** | one branch per fix, each off `upstream/main` | what the operator opens a PR from | exactly one code commit + a `PR_DRAFT.md`/`ISSUE_DRAFT.md` commit |
| **Consumption** | `mm-integration` only | what the fleet installs | every fix's code commit, cherry-picked; **no** draft files; a version marker |

Rules:

- **Never** merge `mm-integration` into a submission branch, and never submit a
  PR from `mm-integration`. Upstream must see one self-contained change at a
  time.
- **Never** push to the `upstream` remote. `origin` (the fork) only.
- Opening PRs is the operator's manual act. Automation stages branches; it does
  not submit them.
- `mm-integration` is rebuilt, not patched in place. It has no unique work of
  its own except the version marker, so it can always be thrown away and
  regenerated from the submission branches (see *Rebase procedure*).

## What is in `mm-integration`

Built off `upstream/main` at `74e7cf9` (mlx-lm 0.32.0 line, "Support for
Poolside LagunaXS ... (#1334)").

| Commit | Fix | Submission branch | Upstream PR | Why the fleet needs it |
|--------|-----|-------------------|-------------|------------------------|
| `bf99a75` | Tensor parallel sharding for Qwen3 MoE (`Model.shard()`) | `qwen3-moe-tensor-parallel` | not yet filed — ledger candidate 2 | **Load-bearing.** Without it the 2-node Qwen3-235B-A22B-4bit TP ring cannot run at all. |
| `06fb30f` | Pipeline parallelism for Qwen3 MoE (`PipelineMixin`, `pipeline_layers`) | `qwen3-moe-pipeline` | not yet filed — ledger candidate 3 | Enables the PP lane for the same checkpoint. Measured 0.87x of TP, so TP stays the quality lane, but the code path must exist. |
| — | ~~`ChunkedKVCache.maybe_trim_front`~~ | ~~`fix-chunked-kv-trim`~~ | **LANDED UPSTREAM #1673** (2026-08-21) | **REMOVED from this branch 2026-08-22.** Upstream fixed it independently with the same reasoning (trim on valid length, not buffer length) and additionally bounds the slice end. Keeping ours would have put a duplicate patch in the fleet's dependency path — see PR-LEDGER candidate 1. |
| (tip) | Version marker `0.32.0+mm.3` | *integration-only, never submitted* | n/a | Lets `pip list` / `uv pip list` distinguish a fork install from a PyPI install on any node. |

The version marker is a PEP 440 local version. It satisfies every downstream
`mlx-lm>=0.31.3` constraint (including `mlx-vlm`'s), and it is the one commit
that must **not** be cherry-picked onto a submission branch.

### Provenance of the Qwen3 MoE work

The `shard()` and `pipeline()` implementations previously existed **only** as
hand-edits to
`~/mlx-mesh/lib/python3.12/site-packages/mlx_lm/models/qwen3_moe.py` on each
node, with `.orig` and `.tp-only` backups alongside, plus a commented copy at
`conductor/docs/patches/qwen3_moe_shard.py` (TP only — the PP half was never
copied out).

`mm-integration`'s `mlx_lm/models/qwen3_moe.py` is **byte-identical** to those
hand-patched venv files on both the primary and the peer. That was verified by
diff, not assumed. The hand-patches are therefore fully subsumed: the venv
`.orig`/`.tp-only` backups are kept as an escape hatch, but nothing depends on
the live hand-edit any more.

### Not in `mm-integration`

- **`rfc-streaming-convert`** — issue text only, no code. Nothing to consume.
- **`fix-mlxlm-031-compat`** — lives in the `vllm-mlx` fork (`conductor`), not
  here.
- **"Prefer the venv `mlx.launch` in the TP ring parity test"**
  (`inkling-mlx` `45d2d95`) — this is a fix to *our* test harness, working
  around a stale `~/Library/Python/3.13/bin/mlx.launch` on PATH. It is not an
  mlx-lm defect and has no upstream home. Already committed in `inkling-mlx`;
  do not stage it here.

## The fork boundary: what we do NOT fork

**`mlx` core is not forked.** It is C++/Metal and would require a source build
on every node — a fleet-wide build toolchain, Metal shader compilation, and
hours per machine. That cost is not justified for two findings, neither of
which needs a code change on our side:

1. **`MLX_METAL_FAST_SYNCH=1` + jaccl + pipeline-parallel deadlock.** Upstream
   issue report + documented workaround: never set `MLX_METAL_FAST_SYNCH` when
   running in pipeline mode. TP is unaffected.
2. **GPU timeout on concurrent mmap loads.** Upstream issue report (draft) +
   documented workaround.

Both stay as **issue reports plus operational workarounds**, tracked in
`../PR-LEDGER.md`. If a future finding genuinely requires an mlx-core patch,
that is a decision to escalate, not to absorb quietly.

**`mlx-vlm` is not in the dependency path either.** We consume it (heavily —
`conductor`'s MLLM engine imports it across 16 modules), but we have **zero
patches** to it: the `azamamirza/mlx-vlm` clone has a clean `main` at the fork
point and no candidate branches. Forking a package we have not patched adds a
rebase obligation and a moving pin for no benefit, so `mlx-vlm` stays pinned to
PyPI (`>=0.6.2`, currently resolving 0.6.8). The clone stays ready if a patch
ever lands. (`inkling-mlx`'s `vlm` extra declares `mlx-vlm` but no
`inkling_mlx` module imports it — that extra is unused today.)

## How consumers resolve this fork

Consumers install from the **public fork over plain HTTPS**, pinned to an
immutable tag:

```toml
mlx-lm = { git = "https://github.com/azamamirza/mlx-lm.git", tag = "mm-v0.32.0-mm.3" }
```

Why this mechanism and not a local path or a hand-distributed wheel:

- The fork is **public and readable without credentials** — verified by
  resolving `HEAD` from the credential-free peer with `GIT_TERMINAL_PROMPT=0`
  and no credential helper. No token to provision on 7 machines, and no token
  to rotate.
- **One source of truth, one bump.** Adding a machine is `uv sync`. Rolling the
  fleet forward is a new tag plus a lockfile bump — not an rsync fan-out to 7
  hosts that can silently half-fail and leave nodes disagreeing about what they
  are running.
- **The tag is immutable; `mm-integration` moves.** Consumers pin the tag so a
  rebuild of `mm-integration` never silently changes what a node installs. The
  branch is for building the next tag, never for consumption.
- A local path dep plus rsync was rejected: it reintroduces exactly the
  per-node drift this change exists to remove. A prebuilt wheel was rejected as
  strictly more machinery (a place to host it, a naming scheme, a sync step)
  for the same result, since mlx-lm is pure Python and a git install is a
  sub-second wheel build.

### Ad-hoc `~/mlx-mesh` ring venvs

These are not uv projects, so they are provisioned imperatively. Use
`--no-deps`: the venv already has a **known-good, jaccl-capable `mlx` /
`mlx-metal` 0.32.0 pair**, and letting the resolver touch it is a real risk to
the ring.

```sh
uv pip install --python ~/mlx-mesh/bin/python --no-deps --reinstall-package mlx-lm \
  "mlx-lm @ git+https://github.com/azamamirza/mlx-lm.git@mm-v0.32.0-mm.3"
```

Confirm with `uv pip list --python ~/mlx-mesh/bin/python | grep mlx-lm` —
it must read `0.32.0+mm.3`. A bare `0.32.0` means the node is on PyPI and is
missing `shard()`.

## Rebase procedure — when upstream releases

`mm-integration` is **regenerated**, never rebased in place.

1. `git fetch upstream && git log upstream/main --oneline -20`
2. For each submission branch still open upstream, rebase it individually:
   `git checkout <branch> && git rebase upstream/main`. Resolve there — that is
   the branch the operator will submit, so its conflict resolution is the one
   that matters.
3. **Check whether upstream landed any of our fixes.** If a fix is now in
   `upstream/main`, mark it dead in `../PR-LEDGER.md`, delete its submission
   branch, and simply omit it from step 4 — do not carry a duplicate.
4. Rebuild the integration branch from the (rebased) code commits:

   ```sh
   git branch -D mm-integration
   git checkout -b mm-integration upstream/main
   git cherry-pick <code commit of each live submission branch, in ledger order>
   # then re-apply the version marker, bumping the local segment:
   #   0.32.0+mm.3 -> <new upstream version>+mm.<n>
   ```

   Cherry-pick the **code** commit only, never the `PR_DRAFT.md` commit. Order
   matters only in that the two Qwen3 MoE commits must go TP then PP (PP
   auto-merges onto TP; the reverse has been unnecessary so far).
5. Tag and push: `git tag mm-v<version> && git push origin mm-integration --force && git push origin mm-v<version>`.
   Force-push is expected and safe here — the branch is disposable by design.
   **Tags are never moved or reused**; a new build gets a new tag.
6. Bump consumers to the new tag (`inkling-mlx`, `conductor` pyprojects), then
   re-run the two acceptance gates: `inkling-mlx`'s full test suite, and a
   short Qwen3-235B TP ring generation proving `shard()` is live.

## Verifying a node is on the fork

```sh
python -c "import mlx_lm, mlx_lm.models.qwen3_moe as q; \
print(mlx_lm.__version__, hasattr(q.Model, 'shard'), hasattr(q.Qwen3MoeModel, 'pipeline'))"
```

Expected: `0.31.3+mm.1 True True`. Anything else and that node will fail the
235B TP run.
