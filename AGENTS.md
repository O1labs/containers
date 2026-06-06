# AGENTS.md

## Cursor Cloud specific instructions

### What this repo is

Monorepo of **Docker images** for Web3/blockchain node clients (Bitcoin, Ethereum, Cosmos, etc.). There is no root application server or `package.json`. Development and CI mean **building and smoke-testing individual service directories** with Docker Compose.

See `README.md` for container overview and `.github/workflows/CI.yaml` for the canonical build/test flow.

### Prerequisites

- **Docker Engine** + **Docker Compose** (v2 plugin: `docker compose`)
- **Python 3** + **`yq`** (CI installs via `pip install yq`; used to read `meta.yml` versions)
- **Internet access** during image builds (Dockerfiles download upstream binaries/source)

### Starting Docker in Cloud Agent VMs

Docker may not auto-start on boot. If `docker` fails with permission or connection errors:

```bash
sudo dockerd > /tmp/dockerd.log 2>&1 &
sleep 2
sudo docker version
```

Use `sudo docker` / `sudo docker compose` unless you have reloaded group membership (`newgrp docker`) after being added to the `docker` group.

Storage driver is configured as `fuse-overlayfs` (required in nested VM environments).

### Build and test a service (matches CI)

Pick a service directory (e.g. `bitcoind/`, `gaia/`, `erigon/`). From repo root:

```bash
dir=bitcoind
version=$(cat $dir/meta.yml | yq -r .version)
sudo docker build $dir/ --file $dir/Dockerfile --build-arg VERSION=$version --tag 0labs/$dir:$version
sudo docker compose -f $dir/docker-compose.yaml up -d --build
sudo docker compose -f $dir/docker-compose.yaml down
```

CI uses the legacy `docker-compose` command name; the Compose v2 plugin (`docker compose`) is equivalent.

### Compose environment variables

All services support:

| Variable | Default | Purpose |
|---|---|---|
| `network` | `mainnet` | Selects `config/${network}/` for mounted configs |
| `host_data_dir` | named volume | Host path or Docker volume for chain data |

Example — Bitcoin **regtest** (fast local dev, no mainnet sync):

```bash
cd bitcoind
network=local sudo -E docker compose -f docker-compose.yaml up -d --build
sudo docker exec bitcoind bitcoin-cli -regtest -rpcuser=bitcoin -rpcpassword=password \
  -datadir=/data -rpcport=28332 getblockchaininfo
```

Gaia defaults to a **local dev chain** (`CHAIN_ID=local`) and exposes RPC on port `26657`.

### Lint / tests

- No repo-wide linter or unit test suite.
- **CI test** = `docker compose up -d --build` then `down` for changed service dirs.
- CD/docs workflows additionally use `jinja-cli` and `jq` (see `.github/workflows/CD.yaml`).

### Gotchas

- Ethereum **beacon** stacks (`lodestar/`, `nimbus/`) expect an execution client at `localhost:8545` for real network operation; that client is not bundled in the same compose file.
- `bitcoind/config/local/bitcoin.conf` uses RPC port **28332** (not the compose-mapped `8332`).
- Image builds can take several minutes for source-compile services (Gaia, Lodestar, Nimbus); binary-download services (bitcoind, besu) are faster.
