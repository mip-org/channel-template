# mip channel template

Template for a [mip](https://mip.sh) package channel. A channel is a GitHub repo that builds MATLAB packages on GitHub Actions, publishes each build as a GitHub Release asset, and serves a package index from GitHub Pages. The build pipeline itself lives in [mip-org/mip_channel_tools](https://github.com/mip-org/mip_channel_tools); the workflows in this repo are thin callers, so you never write or maintain build scripts yourself.

The example packages show the layouts a channel can hold: in-channel source (`hello_inline`), a git-sourced package whose `mip.yaml` lives upstream (`hello_mip`), a native MEX package (`hello_mip_mex`), and a numbl/WASM package (`hello_mip_wasm`). They are the same packages as in [mip-org/mip-hello](https://github.com/mip-org/mip-hello).

## Creating your channel

1. Click **Use this template → Create a new repository**. Name the repo `mip-<channel_name>`, for example `mip-mylab`, and make it **public**.
   - The `mip-` prefix is required. Users install with `mip install --channel <owner>/mylab ...`, and mip looks for the index at the GitHub Pages site of `<owner>/mip-mylab`. A repo without the prefix cannot be reached as a channel.
   - The repo must be public: MATLAB on GitHub's CI runners is licensed only for public repos, and users download packages anonymously.
2. In the new repo, go to **Settings → Pages** and set the source to **GitHub Actions**.
3. Replace the example packages under `packages/` with your own (see [Adding a package](#adding-a-package)), or keep one or two while you get started. Then commit and push to `main`. Each package you add or change is built automatically.
4. Rewrite this README to describe your channel.

Keep `.github/workflows/`, `.gitattributes`, and `.gitignore` as they are; they connect the channel to the build engine.

Once a build has finished, users can install from the channel:

```matlab
mip avail --channel <owner>/mylab
mip install --channel <owner>/mylab my_package
```

There is no central registry, so nothing needs to be registered. See [Hosting a Channel](https://mip.sh/docs/hosting-a-channel) for more detail.

## Adding a package

Each package release lives at `packages/<name>/<release>/` with a `source.yaml` (where the source comes from) and usually a `mip.yaml` (package metadata and supported architectures). The release directory name is the version: `main` tracks the latest commit on a branch, and a name like `1.0.0` is a fixed version, typically pointing at a tag.

```
packages/<name>/<release>/
├── source.yaml        # Required: where to get the source
├── mip.yaml           # Optional: provides or overrides the package metadata
└── compile.m          # Optional: channel-provided compile script
```

The recommended layout keeps the package in its own repo with a `mip.yaml` at the root, and points to it from `source.yaml`:

```yaml
source:
  git: "https://github.com/youruser/my_package"
  branch: "main"
```

If the source repo has no `mip.yaml`, or you want to override it, put one next to `source.yaml`. For very small packages, the source can live directly in the channel with an empty `source.yaml` (see `hello_inline`). Package names use underscores, not hyphens. For the full set of `source.yaml` options and many real examples, see the [packages in mip-core](https://github.com/mip-org/mip-core/tree/main/packages).

## How builds run

Builds run one (package, architecture) pair at a time. They are triggered automatically on push to `main`, daily via a scheduled probe, or manually via a GitHub issue.

### Auto-build on push

Pushes to `main` run the `push-build.yml` workflow, which diffs the push and dispatches `build-package.yml` once per (package, architecture) pair affected by the change.

A file affects `packages/<name>/<release>` if and only if its path lies inside that directory. Each affected package expands to every architecture declared in its `mip.yaml`, intersected with the channel's supported architectures (`any`, `linux_x86_64`, `macos_arm64`, `windows_x86_64`, `numbl_wasm`). Packages with no channel-side `mip.yaml` expand to all of them.

Changes outside `packages/` (workflows, README) do not trigger builds, and deleted packages are skipped. A push that does not change a package's source hash short-circuits at the prepare step.

### Scheduled rebuild

Daily at 06:00 UTC, `scheduled-build.yml` probes every (package, architecture) pair. A pair needs rebuilding if its `.mhl` is missing from GitHub Releases or its source hash no longer matches, typically because an upstream branch advanced. Pairs that need rebuilding are dispatched to `build-package.yml`. The probe can also be run manually:

```bash
gh workflow run scheduled-build.yml
```

### Submitting a build

Open an issue whose title starts with `Build` (case-insensitive). The body lists one or more build lines:

```
<name>@<release> <architecture>
```

Multiple architectures on one line dispatch multiple builds for that package, and multiple lines dispatch multiple packages. Lines without a package reference are ignored. For example:

```
foo@1.0.0 any
bar@2.0 linux_x86_64 macos_arm64
```

Within about 30 seconds the request bot replies with the (package, architecture) pairs it parsed, or a list of errors. If an admin (anyone with write access to the repo) opened the issue, the builds dispatch automatically. Otherwise an admin must reply `approve` on its own line; emoji reactions and `approve` embedded in prose do not count. Editing a submitted issue does not re-validate it, so open a new issue to change anything.

Architecture keywords:

- `any`: pure MATLAB; runs on Ubuntu.
- `linux_x86_64`, `macos_arm64`, `windows_x86_64`: native; run on the matching OS.
- `numbl_wasm`: WebAssembly build for numbl.
- `all`: every architecture declared in the package's `mip.yaml`. A package with no channel-side `mip.yaml` cannot expand `all`.

A build for an architecture the package does not declare exits cleanly with nothing to do.

To build every package, use the keyword `all-packages` as the first token of a line:

```
all-packages linux_x86_64
all-packages all
```

By default, a build that would produce a `.mhl` identical to the published one (same source hash and metadata) is skipped. To rebuild anyway, append `force` to the line; it applies only to that line:

```
foo@1.0.0 linux_x86_64 force
```

### Direct dispatch

The same builds can be dispatched from the command line:

```bash
gh workflow run build-package.yml \
  -f package_path=packages/<name>/<release> \
  -f architecture=<arch> \
  -f force=false
```

To regenerate the channel index and redeploy Pages without rebuilding anything:

```bash
gh workflow run assemble-index.yml
```
