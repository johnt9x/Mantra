# MANTRA
port 12, chain id: mantrachain-1

# Automatic:
```
...
```
# Snapshot:
```
sudo systemctl stop mantrachaind

cp $HOME/.mantrachain/data/priv_validator_state.json $HOME/.mantrachain/priv_validator_state.json.backup
mantrachaind tendermint unsafe-reset-all --home $HOME/.mantrachain
SNAP_RPC="https://mantra-testnet-rpc.itrocket.net:443"

LATEST_HEIGHT=$(curl -s $SNAP_RPC/block | jq -r .result.block.header.height);
BLOCK_HEIGHT=$((LATEST_HEIGHT - 1000));
TRUST_HASH=$(curl -s "$SNAP_RPC/block?height=$BLOCK_HEIGHT" | jq -r .result.block_id.hash) 

echo $LATEST_HEIGHT $BLOCK_HEIGHT $TRUST_HASH && sleep 2

sed -i.bak -E "s|^(enable[[:space:]]+=[[:space:]]+).*$|\1true| ;
s|^(rpc_servers[[:space:]]+=[[:space:]]+).*$|\1\"$SNAP_RPC,$SNAP_RPC\"| ;
s|^(trust_height[[:space:]]+=[[:space:]]+).*$|\1$BLOCK_HEIGHT| ;
s|^(trust_hash[[:space:]]+=[[:space:]]+).*$|\1\"$TRUST_HASH\"| ;
s|^(seeds[[:space:]]+=[[:space:]]+).*$|\1\"\"|" $HOME/.mantrachain/config/config.toml

mv $HOME/.mantrachain/priv_validator_state.json.backup $HOME/.mantrachain/data/priv_validator_state.json

sudo systemctl restart mantrachaind && sudo journalctl -u mantrachaind -f
```
# MONIKER
Replace YOUR_MONIKER_GOES_HERE with your validator name

MONIKER="YOUR_MONIKER_GOES_HERE"

# Update Packages
```
sudo apt update && apt upgrade -y
sudo apt install curl git jq lz4 build-essential unzip fail2ban ufw -y
```
# Install Go
```
ver="1.20.3"
wget "https://golang.org/dl/go$ver.linux-amd64.tar.gz"
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf "go$ver.linux-amd64.tar.gz"
rm "go$ver.linux-amd64.tar.gz"
echo "export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin" >> $HOME/.bash_profile
source $HOME/.bash_profile
go version
```
# Install Cosmovisor
```
go install cosmossdk.io/tools/cosmovisor/cmd/cosmovisor@v1.5.0
```
# Install CosmWasm Library
```
sudo wget -P /usr/lib https://github.com/CosmWasm/wasmvm/releases/download/v1.3.0/libwasmvm.x86_64.so
```
# Download and install binary
```
cd $HOME
wget https://github.com/MANTRA-Finance/public/raw/main/mantrachain-testnet/mantrachaind-linux-amd64.zip
unzip mantrachaind-linux-amd64.zip
```
# Setup Cosmovisor Symlinks
```
mkdir -p $HOME/.mantrachain/cosmovisor/genesis/bin
mv mantrachaind $HOME/.mantrachain/cosmovisor/genesis/bin/
```
```
sudo ln -s $HOME/.mantrachain/cosmovisor/genesis $HOME/.mantrachain/cosmovisor/current
sudo ln -s $HOME/.mantrachain/cosmovisor/current/bin/mantrachaind /usr/local/bin/mantrachaind
```
# Set Configuration for your node
```
mantrachaind config node tcp://localhost:12657
mantrachaind config chain-id mantrachain-1
mantrachaind config keyring-backend file
```
# Init your node
```
mantrachaind init $MONIKER --chain-id mantrachain-1
```
# Add Genesis File and Addrbook
```
wget -O $HOME/.mantrachain/config/genesis.json https://testnet-files.itrocket.net/mantra/genesis.json
wget -O $HOME/.mantrachain/config/addrbook.json https://testnet-files.itrocket.net/mantra/addrbook.json
```

#Configure Seeds and Peers
```
peers="1a46b1db53d1ff3dbec56ec93269f6a0d15faeb4@mantra-testnet-peer.itrocket.net:22656,9c1c4e34e90273f9e99f4c0ea319efd08f872510@167.235.14.83:32656,c533d7ee2037ee6d382f773be04c5bbf27da7a29@34.70.189.2:26656,a435339f38ce3f973739a08afc3c3c7feb862dc5@35.192.223.187:26656,114988f9a053f594ab9592beb79b924430d355ba@34.123.40.240:26656,0e687ef17922361c1aa8927df542482c67fb7571@35.222.198.102:26656,a9a71700397ce950a9396421877196ac19e7cde0@65.108.231.124:22656,5cf36b98453a2f099e86cb0654d30d4763b21d19@62.171.130.196"
sed -i -e "s|^persistent_peers *=.*|persistent_peers = \"$peers\"|" $HOME/.mantrachain/config/config.toml
```
# Set Pruning, Enable Prometheus, Gas Price, and Indexer
```
PRUNING="custom"
PRUNING_KEEP_RECENT="100"
PRUNING_INTERVAL="19"

sed -i -e "s/^pruning *=.*/pruning = \"$PRUNING\"/" $HOME/.mantrachain/config/app.toml
sed -i -e "s/^pruning-keep-recent *=.*/pruning-keep-recent = \
\"$PRUNING_KEEP_RECENT\"/" $HOME/.mantrachain/config/app.toml
sed -i -e "s/^pruning-interval *=.*/pruning-interval = \
\"$PRUNING_INTERVAL\"/" $HOME/.mantrachain/config/app.toml
sed -i -e 's|^indexer *=.*|indexer = "null"|' $HOME/.mantrachain/config/config.toml
sed -i 's|^prometheus *=.*|prometheus = true|' $HOME/.mantrachain/config/config.toml
sed -i -e "s/^minimum-gas-prices *=.*/minimum-gas-prices = \"0uaum\"/" $HOME/.mantrachain/config/app.toml
```
# Set Service file
```
sudo tee /etc/systemd/system/mantrachaind.service > /dev/null << EOF
[Unit]
Description=mantrachaind testnet node service
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

[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload
sudo systemctl enable mantrachaind
```
```
CUSTOM_PORT=12
sed -i.bak -e "s%^proxy_app = \"tcp://127.0.0.1:26658\"%proxy_app = \"tcp://127.0.0.1:${CUSTOM_PORT}658\"%; s%^laddr = \"tcp://127.0.0.1:26657\"%laddr = \"tcp://127.0.0.1:${CUSTOM_PORT}657\"%; s%^pprof_laddr = \"localhost:6060\"%pprof_laddr = \"localhost:${CUSTOM_PORT}060\"%; s%^laddr = \"tcp://0.0.0.0:26656\"%laddr = \"tcp://0.0.0.0:${CUSTOM_PORT}656\"%; s%^prometheus_listen_addr = \":26660\"%prometheus_listen_addr = \":${CUSTOM_PORT}660\"%" $HOME/.mantrachain/config/config.toml
sed -i.bak -e "s%^address = \"tcp://0.0.0.0:1317\"%address = \"tcp://0.0.0.0:${CUSTOM_PORT}317\"%; s%^address = \":8080\"%address = \":${CUSTOM_PORT}080\"%; s%^address = \"0.0.0.0:9090\"%address = \"0.0.0.0:${CUSTOM_PORT}090\"%; s%^address = \"0.0.0.0:9091\"%address = \"0.0.0.0:${CUSTOM_PORT}091\"%; s%^address = \"0.0.0.0:8545\"%address = \"0.0.0.0:${CUSTOM_PORT}545\"%; s%^ws-address = \"0.0.0.0:8546\"%ws-address = \"0.0.0.0:${CUSTOM_PORT}546\"%" $HOME/.mantrachain/config/app.toml
```
# Start the Node
```
sudo systemctl restart mantrachaind
sudo journalctl -fu mantrachaind -o cat
```
