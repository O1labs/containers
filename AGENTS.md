# AGENTS.md

Guidance for AI agents working in this repository.

## Project overview

This is the **O1 Containers Collection**: a monorepo of independent Docker images for Web3 node software (Avalanche, Besu, Bitcoin, Cosmos Gaia, Ethereum clients, etc.). There is no root application server—development means building and running individual service directories.

See [README.md](README.md) for container options (`RUN_ARGS`, config mounts, custom entrypoints).

## Cursor Cloud specific instructions

### Prerequisites

- **Docker** is required for all build/run workflows. CI uses `docker build` and `docker-compose`/`docker compose`.
- **Python 3 + `yq`** (via pip) is used in GitHub Actions to read `meta.yml` version fields during CI/CD.

### Starting Docker in Cloud Agent VMs

If `docker` fails with permission or connection errors:

1. Ensure the daemon is running: `sudo service docker start` (or `sudo dockerd > /tmp/dockerd.log 2>&1 &` if systemd is unavailable).
2. Use `sudo docker` / `sudo docker compose` until the session user is in the `docker` group (requires re-login after `usermod -aG docker $USER`).

This VM uses `fuse-overlayfs` as the Docker storage driver (`/etc/docker/daemon.json`).

### Running a service

Each top-level directory (e.g. `avalanchego/`, `gaia/`, `besu/`) has its own `Dockerfile`, `docker-compose.yaml`, `meta.yml`, and `config/`.

```bash
cd avalanchego
network=local docker compose up -d --build   # local dev config
docker compose down
```

- Set `network=local` to use `./config/local` instead of mainnet defaults.
- Set `host_data_dir` to override the named volume path for chain data.
- Avoid running multiple stacks that bind the same host ports (e.g. `8545` for Besu/Erigon/Nethermind; `8332` for bitcoind/bitcoin-abcd).

### CI-equivalent smoke test

Matches [.github/workflows/CI.yaml](.github/workflows/CI.yaml):

```bash
dir=avalanchego
version=$(cat $dir/meta.yml | yq -r .version)
docker build $dir/ --file $dir/Dockerfile --build-arg VERSION=$version --tag 0labs/$dir:$version
docker compose -f $dir/docker-compose.yaml up -d --build
docker compose -f $dir/docker-compose.yaml down
```

GitHub Actions invokes the standalone `docker-compose` CLI; `docker compose` (Compose plugin) is equivalent.

### Lint / tests

There is no repo-wide linter or unit test suite. Validation is **Docker image build + compose up/down** per changed service directory, as in CI.

### Hello-world verification (example: avalanchego)

```bash
cd avalanchego && network=local docker compose up -d --build
curl -s -X POST --data '{"jsonrpc":"2.0","id":1,"method":"info.getNodeVersion"}' \
  -H 'content-type:application/json' http://127.0.0.1:9650/ext/info
curl -s -X POST --data '{"jsonrpc":"2.0","id":1,"method":"eth_blockNumber","params":[]}' \
  -H 'content-type:application/json' http://127.0.0.1:9650/ext/bc/C/rpc
docker compose down
```

### Build-time notes

- **Fast builds** (prebuilt binaries): `avalanchego`, `bitcoind`, `litecoin`, etc.
- **Slow builds** (compile from source): `gaia`, `erigon`, `besu`, `nethermind`, `lodestar`, `mev-boost`, `nimbus`.
- Validator stacks (Lodestar/Nimbus) may reference external execution clients not defined in this repo.

### Maintainer CI/CD tools (optional on host)

CD and utilities workflows also use `jinja-cli` and `jq` via pip/apt; only needed when editing release automation, not for container smoke tests.
