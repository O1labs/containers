# AGENTS.md

Guidance for AI agents working in this repository.

## What this repo is

**O1 Containers Collection** — a monorepo of independent Docker images for Web3/blockchain node clients (Bitcoin, Ethereum, Cosmos, Avalanche, etc.). There is no root application server or `package.json`. Development and CI mean **building and smoke-testing individual service directories** with Docker Compose.

See `README.md` for container overview and `.github/workflows/CI.yaml` for the canonical build/test flow.

## Service index

| Directory | Client |
|-----------|--------|
| `avalanchego` | Avalanche node |
| `besu` | Ethereum execution (Java) |
| `bitcoind` | Bitcoin Core |
| `bitcoin-abcd` | Bitcoin Cash / eCash |
| `dogecoind` | Dogecoin |
| `erigon` | Ethereum execution (Go) |
| `gaia` | Cosmos Hub (`gaiad`) |
| `litecoin` | Litecoin |
| `lodestar` | Ethereum consensus (beacon + validator) |
| `mev-boost` | MEV-Boost middleware |
| `nethermind` | Ethereum execution (.NET) |
| `nimbus` | Ethereum consensus (beacon + validator) |

## Prerequisites

- **Docker Engine** + **Docker Compose** (v2 plugin: `docker compose`)
- **Python 3** + **`yq`** (CI installs via `pip install yq`; used to read `meta.yml` versions — use pip binary, not `/usr/bin/yq`)
- **Internet access** during image builds (Dockerfiles download upstream binaries/source)

## Starting Docker in Cloud Agent VMs

Docker may not auto-start on boot. If `docker` fails with permission or connection errors:

```bash
sudo dockerd > /tmp/dockerd.log 2>&1 &
sleep 3 && sudo docker version
```

Use `sudo docker` / `sudo docker compose` unless you have reloaded group membership (`newgrp docker`) after being added to the `docker` group. Storage driver is configured as `fuse-overlayfs` (required in nested VM environments).

Ensure the pip-installed `yq` is on your PATH:

```bash
export PATH="$HOME/.local/bin:$PATH"
```

## Build and test a service (matches CI)

Pick a service directory (e.g. `bitcoind/`, `gaia/`, `erigon/`):

```bash
dir=bitcoind
version=$(cat $dir/meta.yml | yq -r .version)
sudo docker build $dir/ --file $dir/Dockerfile --build-arg VERSION=$version --tag 0labs/$dir:$version
sudo docker compose -f $dir/docker-compose.yaml up -d --build
sudo docker compose -f $dir/docker-compose.yaml down
```

CI uses the legacy `docker-compose` command name; the Compose v2 plugin (`docker compose`) is equivalent.

## Compose environment variables

All services support:

| Variable | Default | Purpose |
|----------|---------|---------|
| `network` | `mainnet` | Selects `config/${network}/` for mounted configs |
| `host_data_dir` | named volume | Host path or Docker volume for chain data |

Example — Bitcoin **regtest** (fast local dev, no mainnet sync):

```bash
cd bitcoind
network=local sudo -E docker compose -f docker-compose.yaml up -d --build
sudo docker exec bitcoind bitcoin-cli -regtest -rpcuser=bitcoin -rpcpassword=password \
  -datadir=/data -rpcport=28332 getblockchaininfo
```

Gaia defaults to a local dev chain (`CHAIN_ID=local`) and exposes RPC on port `26657`.

## Hello-world verification (avalanchego)

```bash
cd avalanchego && network=local docker compose up -d --build
curl -s -X POST --data '{"jsonrpc":"2.0","id":1,"method":"info.getNodeVersion"}' \
  -H 'content-type:application/json' http://127.0.0.1:9650/ext/info
docker compose down
```

## Build time notes

- **Fast** (prebuilt binaries): `avalanchego`, `bitcoind`, `bitcoin-abcd`, `dogecoind`, `litecoin`, `besu`, `nethermind`
- **Slow** (compile from source): `gaia`, `erigon`, `lodestar`, `nimbus`, `mev-boost`

## Lint / tests

No repo-wide linter or unit test suite. Validation is image build + compose lifecycle per changed service directory, matching CI. CD/docs workflows additionally use `jinja-cli` and `jq` (see `.github/workflows/CD.yaml`).

## Gotchas

- **Port conflicts**: Ethereum execution clients (`besu`, `erigon`, `nethermind`) default to host port `8545`; only one can bind at a time. Same for `8332` across Bitcoin variants.
- **Beacon stacks** (`lodestar/`, `nimbus/`) expect an execution client at `localhost:8545` for real network operation; that client is not bundled in the same compose file.
- **Mainnet sync**: Running with default `mainnet` config starts real chain sync (slow, disk-heavy). Prefer `network=local` for quick smoke tests.
- `bitcoind/config/local/bitcoin.conf` uses RPC port **28332** inside the container (not the compose-mapped `8332`).
