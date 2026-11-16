# Rune, mirrored for Beamer

This is a mirror of [anomalyco/rune](https://github.com/anomalyco/rune), the UI toolkit
[Beamer](https://github.com/FredeAlexandre/beamer) is built with. It is a plain public repository,
not a GitHub fork. Rune's code and history are upstream's, unchanged. This branch, `beamer`, holds
only this README.

It exists for three reasons:

- **Beamer's build survives upstream.** Every rev Beamer pins is tagged here, so a force-push or a
  deleted branch upstream can't make an old Beamer commit unbuildable.
- **Our patches have a home.** Fixes we make to Rune ourselves (Windows frame pacing, the Windows
  overlay path) live here as branches, never upstream. Upstream doesn't take pull requests.
- **CI can fetch it without credentials.** The repository is public, so hosted runners clone it as an
  ordinary git dependency.

Decided in FredeAlexandre/beamer#13 (ADR 0001) and #22 (ADR 0005); set up in #26.

## What is here

| Ref | What it is |
|---|---|
| upstream's branches (`dev` and the rest) | as upstream had them at the last sync. `dev` is Rune's main line. A branch upstream deletes stays here. |
| `pin/<full sha>` tags | one per rev Beamer has ever pinned. Never moved or deleted. |
| `beamer` (default branch) | this README only, with no history in common with upstream. |
| `patched/<upstream sha>` | none yet. Our own patches on top of a pinned rev (see [Our patches](#our-patches)). |

Issues, the wiki and GitHub Actions are off. Actions must stay off: upstream's workflows are on its
branches and would otherwise run on every sync. Report Rune bugs upstream; track our own work in
FredeAlexandre/beamer.

Two repository rulesets back this up. `pins` stops any `refs/tags/pin/*` tag from being moved or
deleted. `own branches` stops `dev`, `beamer` and `patched/**` from being deleted or force-pushed.
Nobody bypasses them; changing them is a decision for Alex.

## Current pin

| | |
|---|---|
| Rune rev | `0796df5fa269b85d4fc04786bbd37f7349eb6606` on `dev`, "fix(a11y): let a screen reader walk a text field of any length (#1058)", 2026-11-16 |
| Tag | `pin/0796df5fa269b85d4fc04786bbd37f7349eb6606` |
| Toolchain | `1.99.1`, from Rune's `rust-toolchain.toml` at that rev |
| Pinned | 2026-11-16, FredeAlexandre/beamer#26 |

## Depending on Rune at the pin

Cargo doesn't apply a dependency's `[patch]` table to the workspace that depends on it, so Beamer's
workspace root copies Rune's, at the same rev. Every entry uses the same URL and rev as `rune` itself:

```toml
[dependencies]  # or [workspace.dependencies]
rune = { git = "https://github.com/FredeAlexandre/rune", rev = "0796df5fa269b85d4fc04786bbd37f7349eb6606" }

# Rune's [patch.crates-io] at 0796df5fa269b85d4fc04786bbd37f7349eb6606, pointed at this mirror.
[patch.crates-io]
wgpu-hal = { git = "https://github.com/FredeAlexandre/rune", rev = "0796df5fa269b85d4fc04786bbd37f7349eb6606" }
winit = { git = "https://github.com/FredeAlexandre/rune", rev = "0796df5fa269b85d4fc04786bbd37f7349eb6606" }
parley_engine = { git = "https://github.com/FredeAlexandre/rune", rev = "0796df5fa269b85d4fc04786bbd37f7349eb6606" }
accesskit_windows = { git = "https://github.com/FredeAlexandre/rune", rev = "0796df5fa269b85d4fc04786bbd37f7349eb6606" }
accesskit_macos = { git = "https://github.com/FredeAlexandre/rune", rev = "0796df5fa269b85d4fc04786bbd37f7349eb6606" }
```

Each patched crate is the published crate plus a patch, kept under `vendor/` in Rune:

| Crate | What Rune's patch adds |
|---|---|
| `wgpu-hal` | DX12: one adapter exposed without creating devices on the others, FXC loaded lazily, partial presents (`Present1` dirty rectangles) |
| `winit` | Windows: a new window isn't added again to the taskbar (about 7.8 ms off every first frame), and a positioned window is born at that display's scale |
| `parley_engine` | bidi rule L1 for trailing whitespace, and shaping that reuses segmenters and font metrics (about 14% faster) |
| `accesskit_windows` | headings and landmarks reach UI Automation, and long text fields keep their text pattern |
| `accesskit_macos` | headings reach VoiceOver as `AXHeading`, off-main-thread calls don't panic, and long text fields keep their ranges |

A missing entry still builds, but silently drops that patch: without `accesskit_macos`, for example,
VoiceOver doesn't find Rune's headings. The table must match Rune's root `Cargo.toml` exactly at
each pin.

The toolchain matches Rune's channel and moves only with the pin. Beamer's `rust-toolchain.toml`:

```toml
[toolchain]
channel = "1.99.1"
components = ["clippy", "rustfmt", "llvm-tools"]
```

Checked on 2026-11-16 with a throwaway crate (Rune's `examples/hello`,
`default-features = false, features = ["accessibility", "metal-backend"]`) built from this mirror
with exactly the tables above:

| Clean build | Mac (M5, 10 cores, 24 GB) | Windows PC `lebig` (12 threads, 28 GB) |
|---|---|---|
| `cargo build` (dev) | 29 s | 78 s |
| `cargo build --release` | 55 s | 108 s |
| peak memory of one `rustc` | 2.6 GB | 2.9 GB (4.3 GB for all at once) |

## Syncing the mirror

The weekly Rune-follow agent runs this, and so does anyone who bumps the pin. Keep a bare mirror
clone of upstream between runs, or make a new one:

```sh
git clone --mirror https://github.com/anomalyco/rune.git rune.git   # first time only
cd rune.git
git remote add mirror git@github.com:FredeAlexandre/rune.git        # first time only

git fetch --prune origin    # updates the local clone; pruning here never reaches the mirror
git push mirror 'refs/heads/*:refs/heads/*' 'refs/tags/*:refs/tags/*'
```

The push has no `+`, so it only fast-forwards. If it rejects a branch:

- **`dev` rejected**: upstream rewrote `dev`, which its README says never happens. Stop. Don't force
  it. Open an issue on FredeAlexandre/beamer with the old and new tips, for Alex.
- **another upstream branch rejected**: upstream force-pushed that branch. Force that one branch
  alone: `git push mirror '+refs/heads/<branch>:refs/heads/<branch>'`.

Never, against the mirror:

- `git push --mirror`, `--prune` or `--delete`: they would drop branches upstream deleted, and pin tags;
- force-push or delete a `pin/*` tag, or `dev`, `beamer` or a `patched/*` branch;
- push upstream's `refs/pull/*` (a mirror clone has them; GitHub refuses them anyway).

Then check every pin is still there: `git ls-remote mirror 'refs/tags/pin/*'`.

## Bumping Beamer's pin

During the port the pin moves only by hand, when the port needs an upstream fix. Once Alex declares
Beamer stable, the weekly Rune-follow agent follows the rev Anomaly's Hex app pins in its
`Cargo.toml`, since that rev has shipped to real users.

1. Sync the mirror (above), so the new rev is here.
2. Tag the rev and push the tag:
   ```sh
   git tag -a pin/<full sha> -m "Beamer pin: Rune dev at <full sha> (toolchain <channel>)" <full sha>
   git push mirror refs/tags/pin/<full sha>
   ```
3. Read Rune's `[patch.crates-io]` and toolchain at that rev:
   ```sh
   git show <full sha>:Cargo.toml | sed -n '/^\[patch.crates-io\]/,/^\[/p'
   git show <full sha>:rust-toolchain.toml
   ```
4. On Beamer, in **one commit**: the `rev` of `rune` and of every `[patch.crates-io]` entry (adding or
   removing entries so the table matches Rune's), the `channel` in `rust-toolchain.toml`, and the
   `Cargo.lock` that follows. Never bump the pin and the toolchain in separate commits.
5. Build and test on both machines, then open a PR for Alex.
6. Update [Current pin](#current-pin) and the tables above on this branch.

Old `pin/*` tags stay forever: older Beamer commits still build from them.

## Our patches

None yet. When Beamer needs a fix of its own (Windows frame pacing, overlay bugs), it goes on a
branch named `patched/<upstream full sha>`, cut from the pinned upstream rev, one commit per fix. The
pin then becomes that branch's tip, tagged `pin/<tip sha>` like any other pin, and Beamer depends on
that rev.

At the next bump, cut a new `patched/<new upstream sha>` from the new rev and cherry-pick the fixes
that are still needed. The old branch stays as it was: patch branches are never rebased or
force-pushed.
