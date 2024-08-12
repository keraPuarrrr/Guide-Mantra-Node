# Guide-Mantra-Node

# Hardware Requirement
```bash
- Memory: 16 GB RAM
- CPU: 12 cores
- Disk: 1 TB NVME SSD
```

# Installation

## Install Dependencies

```bash
sudo apt -q update
sudo apt -qy install curl git jq lz4 build-essential fail2ban ufw unzip
sudo apt -qy upgrade
```

## Configure Moniker

Replace MONIKERNAME with your own

```bash
MONIKER="MONIKERNAME"
```

## Install Go

```bash
sudo rm -rf /usr/local/go
curl -Ls https://go.dev/dl/go1.20.10.linux-amd64.tar.gz | sudo tar -xzf - -C /usr/local
eval $(echo 'export PATH=$PATH:/usr/local/go/bin' | sudo tee /etc/profile.d/golang.sh)
eval $(echo 'export PATH=$PATH:$HOME/go/bin' | tee -a $HOME/.profile)
```

## Download Binaries

```bash
sudo wget -O /usr/lib/libwasmvm.x86_64.so https://github.com/CosmWasm/wasmvm/releases/download/v1.3.0/libwasmvm.x86_64.so
```

### Download mantrachain binaries

```bash
cd $HOME
wget https://snap.warlord44.host/mantra-testnet/mantrachaind-linux-amd64.zip
unzip mantrachaind-linux-amd64.zip
chmod +x mantrachaind
```

### Prepare binaries for cosmovisor

```bash
mkdir -p $HOME/.mantrachain/cosmovisor/genesis/bin
mv mantrachaind $HOME/.mantrachain/cosmovisor/genesis/bin/
```

### Create symlinks

```bash
sudo ln -s $HOME/.mantrachain/cosmovisor/genesis $HOME/.mantrachain/cosmovisor/current -f
sudo ln -s $HOME/.mantrachain/cosmovisor/current/bin/mantrachaind /usr/local/bin/mantrachaind -f
```

## Install Cosmovisor

```bash
go install cosmossdk.io/tools/cosmovisor/cmd/cosmovisor@v1.5.0
```

## Create Service

```bash
sudo tee /etc/systemd/system/mantra.service > /dev/null << EOF
[Unit]
Description=mantra node service
After=network-online.target
 
[Service]
User=$USER
ExecStart=$(which cosmovisor) run start
Restart=on-failure
RestartSec=10
LimitNOFILE=65535
Environment="DAEMON_HOME=$HOME/.mantrachain"
Environment="DAEMON_NAME=mantrachaind"
Environment="UNSAFE_SKIP_BACKUP=true"
Environment="PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin:$HOME/.mantrachain/cosmovisor/current/bin"
 
[Install]
WantedBy=multi-user.target
EOF
```

## Enable Service

Enable mantra systemd service

```bash
sudo systemctl daemon-reload
sudo systemctl enable mantra
```

## Initialize Node

### Setting node configuration

```bash
mantrachaind config chain-id mantra-hongbai-1
mantrachaind config keyring-backend test
mantrachaind config node tcp://localhost:22257
```

### Initialize node

```bash
mantrachaind init $MONIKER --chain-id mantra-hongbai-1
```

## Download Genesis & Addrbook

```bash
curl -Ls https://snap.warlord44.host/mantra-testnet/genesis.json > $HOME/.mantrachain/config/genesis.json
curl -Ls https://snap.warlord44.host/mantra-testnet/addrbook.json > $HOME/.mantrachain/config/addrbook.json
```

## Configure Seeds

Setting up a seed peers

```bash
sed -i -e "s|^seeds *=.*|seeds = \"d1d43cc7c7aef715957289fd96a114ecaa7ba756@testnet-seeds.warlord44.host:22210\"|" $HOME/.mantrachain/config/config.toml
```

## Configure Min Gas Prices

```bash
sed -i -e "s|^minimum-gas-prices *=.*|minimum-gas-prices = \"0.025uaum\"|" $HOME/.mantrachain/config/app.toml
```

## Configure pruning setting

```bash
sed -i \
  -e 's|^pruning *=.*|pruning = "custom"|' \
  -e 's|^pruning-keep-recent *=.*|pruning-keep-recent = "100"|' \
  -e 's|^pruning-keep-every *=.*|pruning-keep-every = "0"|' \
  -e 's|^pruning-interval *=.*|pruning-interval = "19"|' \
  $HOME/.mantrachain/config/app.toml
```

## Download Snapshots

```bash
curl -L https://snap.warlord44.host/mantra-testnet/mantra-latest.tar.lz4 | tar -Ilz4 -xf - -C $HOME/.mantrachain
[[ -f $HOME/.mantrachain/data/upgrade-info.json ]] && cp $HOME/.mantrachain/data/upgrade-info.json $HOME/.mantrachain/cosmovisor/genesis/upgrade-info.json
```

## Start Service

```bash
sudo systemctl start mantra
```
