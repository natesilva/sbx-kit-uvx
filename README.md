# uvx kit

A [Docker Sandbox](https://docs.docker.com/ai/sandboxes/) mixin kit that installs
[Astral `uv`](https://docs.astral.sh/uv/) and its `uvx` front-end, so an agent can
run Python tools straight from PyPI.

## Usage

Add to an existing sandbox (its state is preserved across the recreate):

```sh
sbx kit add my-sandbox ./sbx-kit-uvx
```

Or create a sandbox with it:

```sh
sbx create --name my-sandbox --kit ./sbx-kit-uvx claude .
```

Or straight from this repo, without cloning:

```sh
sbx create --name my-sandbox --kit "git+https://github.com/natesilva/sbx-kit-uvx.git" claude .
```

Remote kit sources are allowlisted, and `sbx` ships allowing only
`docker.io/`, so that form fails until you opt in to this publisher:

```sh
sbx settings set kit.allowedSources '["docker.io/","github.com/natesilva/"]'
```

Verify from the host:

```sh
sbx exec my-sandbox uvx --version
sbx exec my-sandbox uvx ruff --version
```

Or from inside the sandbox (`sbx exec -it my-sandbox bash`), just
`uvx --version`.

## What it does

- Installs uv 0.11.32 with Astral's official installer, into `/usr/local/bin`
  (`UV_INSTALL_DIR`). The installer script is pinned by SHA256. Nothing touches
  the sandbox's Python — uv is a standalone binary.
- Adds `~/.local/bin` to `PATH` for the agent user, so `uv tool install` results
  are runnable.
- Allowlists Astral's release hosts, PyPI, and GitHub for uv's standalone
  CPython downloads.
- Appends a short usage note to the agent's memory file.

## Network

`uvx` only reaches hosts in `caps.network.allow`. The defaults cover resolving
and downloading tools from PyPI. If a tool you run needs to talk to something
else at runtime — a private index, an API, a model provider — add that host to
`caps.network.allow` in `spec.yaml` and re-add the kit.

## Bumping the uv version

Change the version in the installer URL in `spec.yaml`, update
`INSTALLER_SHA256` alongside it, and fix the version in the install step's
`description` and in this README. The version is pinned in the URL
(`https://astral.sh/uv/<version>/install.sh`); the unversioned
`https://astral.sh/uv/install.sh` would install whatever is current at the time
each sandbox is built.

To get the new digest:

```sh
curl -sSL https://astral.sh/uv/<version>/install.sh | shasum -a 256
```

## Supply chain

The installer script embeds a SHA256 for each platform's uv tarball and fails
the install on mismatch, so pinning the script's own digest transitively pins
the binary — one constant covers both hops, and `releases.astral.sh` is trusted
once, when you record the hash, rather than on every sandbox build.

This is still trust-on-first-use: it protects against later tampering with a
release, not against a compromised release being published in the first place.
Verify the digest against a source you trust when bumping.

If the check fails, `sbx create` aborts with a bare
`500 Internal Server Error`. The actual `sha256sum` mismatch is only in the
daemon log:

```sh
tail -f "$HOME/Library/Application Support/com.docker.sandboxes/sandboxes/sandboxd/daemon.log"
```

## License

MIT — see [LICENSE](LICENSE).
