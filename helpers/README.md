# helpers/ — ephemeral local helper tools

Docker-Compose definitions for tools we only need *temporarily* and that do NOT belong in the cluster (admin UIs,
one-off utilities). They run on the workstation (Docker / Colima) and reach cluster services over the LAN via their
LoadBalancer IPs (`192.168.47.200-209`).

```sh
cd helpers
docker compose up <service> -d      # start just what you need
docker compose down                 # stop it again
```

Keeping these out of the cluster avoids permanently exposing admin interfaces (often with default credentials) and keeps
the cluster to actual workloads.

## QA fleet

The containers `lefthook.yml` runs its gates in. The base fleet comes from
[lefthook-qa-gates](https://github.com/rehsack/lefthook-qa-gates), whose contract lives as a verbatim copy in
`lefthook/base.yml`; AsciiDoc runs in images of its own beside it. lefthook.yml itself names no container; a
per-developer, git-ignored `lefthook-local.yml` in the repository root points the templates at the services:

```yaml
templates:
  gates: docker compose -f helpers/docker-compose.yml exec -T gates
  adoc: docker compose -f helpers/docker-compose.yml exec -T adoc
  py-srv: docker compose -f helpers/docker-compose.yml exec -T python
```

One-time setup, then keep the runners warm for the session. uid/gid let the fixers rewrite your files; name and mail
because gitlint in the container sees no `~/.gitconfig`:

```sh
printf 'GIT_USER_NAME=%s\nGIT_USER_EMAIL=%s\nUSER_UID=%s\nUSER_GID=%s\n' \
    "$(git config user.name)" "$(git config user.email)" "$(id -u)" "$(id -g)" \
    > helpers/.env                     # git-ignored
lefthook install
docker compose -f helpers/docker-compose.yml up -d gates adoc python
lefthook run pre-commit --all-files     # whole tree, not just what is staged
```

| Service  | Image                                            | Gates                                                   |
| -------- | ------------------------------------------------ | ------------------------------------------------------- |
| `gates`  | `ghcr.io/rehsack/lefthook-qa-gates:0.2`          | base contract: yamllint, gitleaks, gitlint, mdformat, … |
| `adoc`   | `ghcr.io/tpo42/adoc:1.0`                         | `validate --strict`                                     |
| `python` | `ghcr.io/astral-sh/uv:0.12.21-python3.13-trixie` | `asciidoc-linter` (via `uvx`, commit-pinned)            |

The image tag of `gates` moves together with the copied contract — bump procedure in the header of `lefthook.yml`.
