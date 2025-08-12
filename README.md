# Cysis-node-run-
Cysis node run step by step full guide, provide all types cysis node run details like verifier, prover or validator

TL;DR (pick one)
1) Want an easy, low-resource role? Run a Verifier (works on Linux / mac / Windows; quick scripted installers). 

2) Heavy compute / you have GPUs and lots of RAM? Run a Prover (big hardware needs, uses setup_prover script). 

3) Want to validate blocks / be a validator? Run a Validator (uses Docker Compose, requires whitelist & staking).

0) Common prep (applies to all node types)
1. Use a fresh Ubuntu 20.04+ server (most docs reference Linux; verifier also supports mac/windows). 

2. Install common tools (example for Ubuntu):

sudo apt update && sudo apt upgrade -y
sudo apt install -y build-essential git curl jq wget

3. Keep a secure place to store mnemonics/backups (do not share them). Always back up the mnemonic the first time a key is generated.

A)  Verifier node — quick (recommended first step)
Verifier nodes are lightweight and intended to verify proofs and earn credits. There are one-line installers for Linux/Mac/Windows.

Linux (TL;DR):

# replace 0x-... with your reward address (from your wallet)
curl -L https://github.com/cysic-labs/cysic-phase3/releases/download/v1.0.0/setup_linux.sh > ~/setup_linux.sh && bash ~/setup_linux.sh 0x-FILL-YOUR-REWARD-ADDRESS
cd ~/cysic-verifier && bash start.sh

Mac / Windows steps are analogous (docs include setup_mac.sh and setup_windows.ps1 variants). The installer downloads the verifier binary, libraries, creates config.toml and a start.sh/start.ps1. The verifier creates mnemonic files in ~/.cysic/keys/ — back those up. 

2) Prover node — quick + hardware notes
Provers do the heavy ZK proving. Requirements are large and vary by proof type:
Scroll proof (very large): ~256 GB RAM, 32 cores, GPU ≥ 20 GB VRAM.
ETH proof: ~32 GB RAM, 8 cores, GPU ≥ 16 GB VRAM.
The prover uses a setup script that needs your reward address and an RPC URL (e.g., Alchemy).

Linux (TL;DR):

# replace address and RPC
curl -L https://github.com/cysic-labs/cysic-phase3/releases/download/v1.0.0/setup_prover.sh > ~/setup_prover.sh && bash ~/setup_prover.sh 0x-FILL-YOUR-REWARD-ADDRESS YOUR_RPC_URL
cd ~/cysic-prover && bash start.sh

The script installs the prover binary, libraries, creates config files and start.sh. You’ll be asked whether to set up ETH proof dependencies during install. Back up any mnemonic files created (e.g. ~/.cysic/assets/). 


3) Validator node — full step-by-step (Docker Compose)
Validators are run in Docker; validators are permissioned in the testnet phases (you must be whitelisted and stake). This is the longer setup.

A. Minimum recommended validator hardware & network
CPU: 4 cores (8 recommended)
RAM: 8 GB (16 GB recommended)
Storage: 200 GB SSD (500 GB recommended)
Ports: 26657 (RPC), 26656 (P2P), 8545/8546 (eth RPC/WebSocket), 1317 (REST), 9090 (gRPC), 6065 (metrics). 

B. Install Docker & Docker Compose:

# Ubuntu example
sudo apt update
sudo apt install -y ca-certificates curl gnupg lsb-release
# Docker repo + install
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] \
  https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" \
  | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
sudo usermod -aG docker $USER

Log out/in or newgrp docker to pick up docker group.

C. Download config & snapshot (official):

mkdir -p /data && cd /data
wget https://statics.prover.xyz/testnet_node.tar.gz
tar -zxvf testnet_node.tar.gz
# snapshot (faster initial sync)
wget https://statics.prover.xyz/data.tar.gz
rm -rf node/cysicmintd/data
tar -zxvf data.tar.gz -C node/cysicmintd

D. Docker Compose example:
Edit docker-compose.yml to match this example (the docs show one similar to below). Pay attention to volumes, ports and entrypoint:

version: "3"
services:
  node:
    container_name: node
    image: "ghcr.io/cysic-tech/cysic-network:testnet"
    ports:
      - "26657:26657"
      - "26656:26656"
      - "8545:8545"
      - "8546:8546"
      - "1317:1317"
      - "9090:9090"
      - "6065:6065"
    environment:
      - ID=YOUR_NODE_ID
      - LOG=cysicmintd.log
      - LOGLEVEL=info
      - TRACE=false
    volumes:
      - ./node/cysicmintd:/cysicmint
    networks:
      - cysic-network
    restart: unless-stopped
    entrypoint: "bash start-docker.sh"
networks:
  cysic-network:
    driver: bridge


E. Whitelist requirement — IMPORTANT
Before starting you must send the node’s exit IP address to the Cysic team so they can whitelist your validator (permissioning applies in testnet phases). Do that before docker-compose up. 

F. Start the node:

docker-compose pull
docker-compose up -d node
# follow logs
docker-compose logs -f node

check node health:

curl http://localhost:26657/status
curl http://localhost:26657/abci_info

(If you used snapshot, sync should be fast.) 

G. Create/generate validator key inside container

docker exec -it node bash
# inside container
./cysicmintd keys add validator-xxx --home ./cysicmint --keyring-backend test
# check node ID & validator pubkey
./cysicmintd tendermint show-node-id --home ./cysicmint --keyring-backend test
./cysicmintd tendermint show-validator --home ./cysicmint --keyring-backend test


Save the mnemonic printed during keys add securely — it’s the only recovery method. 


H. Stake / create validator (example)

(Replace amounts and moniker as appropriate.)
./cysicmintd tx staking create-validator \
  --amount=100000000000000000000CGT \
  --moniker="my-validator" \
  --details="my validator node" \
  --chain-id cysicmint_9001-1 \
  --commission-rate="0.05" \
  --commission-max-rate="0.20" \
  --commission-max-change-rate="0.01" \
  --min-self-delegation=1000000000000000000 \
  --pubkey=$(./cysicmintd tendermint show-validator --home ./cysicmint) \
  --from validator-xxx \
  --home=./cysicmint \
  --keyring-backend test \
  --gas auto \
  --gas-adjustment 1.2 \
  --gas-prices=300000CYS \
  --yes


  Docs show CGT & CYS token model and example parameters for gas/prices and chain-id. Make sure you have testnet tokens (contact the Cysic team / faucet for test tokens during testnet). 

  Monitoring & common commands:
Logs: docker-compose logs -f node or docker-compose logs --tail=100 node 

Node status: curl http://localhost:26657/status → check sync_info. 

Prometheus metrics: http://localhost:6065/debug/metrics/prometheus (if enabled). 


Troubleshooting (quick):
Node won’t sync: check snapshot, verify config.toml, ensure ports open, check peers in config/addrbook.json. If necessary, clear data/ and re-extract snapshot. 

Validator not active: ensure enough stake, check jailed status, confirm whitelist and staking tx succeeded. 

Container fails: docker-compose logs node | grep -i error and increase LOGLEVEL if needed. 

Follow me on x <https://x.com/ashisofficial_?t=hJa4qBqdtSFtiuWxeSfIWg&s=09>

