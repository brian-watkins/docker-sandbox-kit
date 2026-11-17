# docker-sandbox-kit

## Getting Started

First, install [docker sandbox](https://docs.docker.com/ai/sandboxes/):

```sh
brew trust docker/tap
brew install docker/tap/sbx
```

## Creating a sandbox

Create a sandbox that uses claude and applies the dev and volta mixins like so:

```sh
sbx create --name my-project --clone \
  --kit ~/workspace/docker-sandbox-kit/dev \
  --kit ~/workspace/docker-sandbox-kit/volta \
  claude ~/workspace/my-project
```

Now you can setup claude desktop to access this sandbox via ssh at `my-project.sbx`. You can do
the same with an editor like Zed or just ssh to this in a terminal.

Note: on macOS, use `--clone`. Without it the sandbox bind-mounts the host checkout, so the sandbox
(Linux) and the host (macOS) share one `node_modules`, and packages with native builds or
platform-specific binaries only work on whichever side installed them. With `--clone` the
sandbox gets its own Git clone and its own dependencies:

- The host repo is mounted read-only at `/run/sandbox/source`. Only committed files are in the
  clone, so copy anything untracked you need from there, e.g.
  `cp /run/sandbox/source/.env .`.
- Run `npm ci` (or similar) inside the sandbox after creating it.
- To get work back, push from the sandbox to `origin` (the `dev` mixin forwards your SSH
  agent), or fetch on the host from the `sandbox-<name>` remote that `sbx` adds:
  `git fetch sandbox-funmail`.

## Mixins

Mixins for Docker Sandboxes (`sbx`).

- `dev/`: everyday setup for local development. It does four things:
  - Forwards the host SSH agent to `/run/ssh-agent.sock`. The built-in agent kits don't request
    this, so without it there is no ssh agent in the sandbox
    ([docker/sbx-releases#835](https://github.com/docker/sbx-releases/issues/835)).
  - Allows network access to `github.com:22`, so git can push and pull over SSH. Without
    this, a `deny-all` global policy blocks that port.
  - Copies the host's global git `user.name` and `user.email` into the sandbox. The built-in
    agent kits don't request this either, so without it commits have no author. Creating the
    sandbox fails if the host has no global name and email set.
  - Registers the Debian alternatives `vi`, `vim`, `view`, `editor` and `pager`. The sandbox
    image includes vim, less and nano but never registers these names, and Debian's git runs
    `editor` and `pager` by default, so `git commit`, `git rebase -i` and `git log` fail without
    them. `editor` is vim; change it with `--kit-arg dev.editor=nano`.
- `volta/`: installs [Volta](https://volta.sh). `node`, `npm` and `npx` are Volta shims, so a
  project gets the Node version pinned in the `volta` field of its `package.json` (downloaded on
  first use). Outside a pinned project the default is Node LTS, installed when the sandbox is
  created (change it with `--kit-arg volta.defaultNode=24`). `npm i -g` installs into the
  sandbox's npm prefix, which is already on `PATH`.

Note: building a mixin that has a `build:` section (only `volta/` here) needs Docker Desktop running
with the containerd image store turned on. Without it, Docker Desktop's builder can't export the
OCI image `sbx` needs. An alternative is to create a special buildx builder for sbx.

To check a mixin without creating a sandbox: `sbx kit validate ./volta`.
