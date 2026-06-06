# AGENTS.md

## Cursor Cloud specific instructions

### What this repo is

Monorepo of **12 independent Docker images** for Web3 blockchain node clients (`avalanchego`, `besu`, `bitcoind`, etc.). There is no root-level app server or shared `package.json`/`Makefile`. Development and CI are Docker-centric: build an image, then `docker compose up` in the service subdirectory.

See [README.md](README.md) and [.github/workflows/CI.yaml](.github/workflows/CI.yaml) for the canonical workflow.

### Prerequisites

- **Docker** and **Docker Compose** (`docker compose`) must be running. In Cloud Agent VMs, start the daemon if needed:
  ```bash
  sudo dockerd > /tmp/dockerd.log 2>&1 &
  sleep 3 && sudo docker info
  ```
- **CI YAML parsing**: GitHub Actions installs `yq` via pip. Use the pip-installed binary (not `/usr/bin/yq`):
  ```bash
  export PATH="$HOME/.local/bin:$PATH"
  ```

`jq` is available at `/usr/bin/jq`.

### Build and test (matches CI)

From a service directory (example: `bitcoind`):

```bash
export PATH="$HOME/.local/bin:$PATH"
cd bitcoind
version=$(cat meta.yml | yq -r .version)
sudo docker build . --file Dockerfile --build-arg VERSION=$version --tag 0labs/bitcoind:$version
sudo docker compose -f docker-compose.yaml up -d --build
sudo docker compose -f docker-compose.yaml down
```

Use `sudo` for Docker commands unless your user is in the `docker` group.

### Local / regtest networks

Several services support `network=local` to mount `config/local` instead of `config/mainnet` (e.g. Bitcoin regtest). Example:

```bash
cd bitcoind
network=local sudo -E docker compose -f docker-compose.yaml up -d --build
sudo docker exec bitcoind bitcoin-cli -conf=/config/bitcoin.conf -datadir=/data getblockchaininfo
```

Regtest RPC listens on port **28332** inside the container; default compose port mappings target mainnet (8332). Use `docker exec` + `bitcoin-cli`, or override compose ports for host RPC access.

### Gotchas

- **No repo-wide lint or unit tests** — validation is image build + compose lifecycle.
- **Port conflicts**: Ethereum execution clients (`besu`, `erigon`, `nethermind`) default to host port `8545`; only one can bind it at a time.
- **Build time**: Dockerfiles download upstream release binaries; first build needs network and can take several minutes per service.
- **Mainnet sync**: Running with default `mainnet` config starts real chain sync (slow, disk-heavy). Prefer `network=local` for quick smoke tests.

### Service index

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
