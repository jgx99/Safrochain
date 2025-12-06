# Safrochain Testnet Validator Guide (Dec 06, 2025) - Full Node + Validator

The most detailed guide, latest updates, run it once and get it right away.

## Recommended VPS Requirements
- CPU: 4–8 cores
- RAM: 16–32 GB
- SSD NVMe: 500 GB+ (the chain is growing)
- Bandwidth: 1 Gbps
- OS: Ubuntu 22.04 LTS

You can rent VPS here :
https://my.hostbrr.com/order/forms/a/MTQxMTk=

<img width="944" height="674" alt="1" src="https://github.com/user-attachments/assets/64d98e73-62af-4cd7-a8ef-6a07b64ef7c4" />


## All Commands (copy-paste each section in order)

```bash
# 1. Update system + install tools
sudo apt update && sudo apt upgrade -y
sudo apt install curl tar wget clang pkg-config libssl-dev jq build-essential bsdmainutils git make gcc git lz4 ncdu -y

# 2. Install Go 1.22.8 (the best running version)
wget https://go.dev/dl/go1.22.8.linux-amd64.tar.gz
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf go1.22.8.linux-amd64.tar.gz
echo ‘export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin’ >> ~/.bash_profile
source ~/.bash_profile
go version # should display go1.22.8

# 3. Set variables (change MONIKER to the name you want to display in Explorer)
echo “export WALLET=wallet” >> ~/.bash_profile
echo “export MONIKER=YourName” >> ~/.bash_profile # change to your name
echo “export SAFROCHAIN_CHAIN_ID=safro-testnet-1” >> ~/.bash_profile
echo “export SAFROCHAIN_PORT=13” >> ~/.bash_profile
source ~/.bash_profile

# 4. Clone repo + build binary (latest tag v0.1.0 – currently the most stable)
cd $HOME
rm -rf safrochain-node
git clone https://github.com/Safrochain-Org/safrochain-node.git
cd safrochain-node
git checkout v0.1.0
make install

safrochaind version # must output v0.1.0

# 5. Init node
safrochaind init $MONIKER --chain-id $SAFROCHAIN_CHAIN_ID

# 6. Latest Genesis + Addrbook
curl -Ls https://rpc.testnet.safrochain.com/genesis | jq -r .result.genesis > ~/.safrochain/config/genesis.json
curl -Ls https://cdn.crxanode.me/safrochain/addrbook.json > ~/.safrochain/config/addrbook.json

# 7. Set seeds & peers (automatically fetch live peers)
sed -i -e “/^seeds *=/{s/= .*/= \”43765518cb55514152d9eeb4072e1d6859cfe428@safrochain-t.nodevism.com:13656\“/}” ~/.safrochain/config/config.toml

PEERS="$(curl -sS https://rpc.safrochain-t.nodevism.com/net_info | jq -r '.result.peers[] | “$$ .node_info.id)@\(.remote_ip):\(.node_info.listen_addr)”‘ | awk -F ’:' ‘{print $1“:”$(NF)}’ | paste -sd, -)"
sed -i -e “s/^persistent_peers *=.*/persistent_peers = \”$PEERS\“/” ~/.safrochain/config/config.toml

# 8. Custom port (13xxx)
sed -i.bak -e "s%:26658%:\( {SAFROCHAIN_PORT}658%g; s%:26657%: $${SAFROCHAIN_PORT}657%g; s%:6060%:$$ {SAFROCHAIN_PORT}060%g; s%:26656%: $${SAFROCHAIN_PORT}656%g; s%^external_address = \“\”%external_address = \“$$ (wget -qO- eth0.me): $${SAFROCHAIN_PORT}656\”%g" ~/.safrochain/config/config.toml
sed -i.bak -e “s%:1317%:$$ {SAFROCHAIN_PORT}317%g; s%:9090%: $${SAFROCHAIN_PORT}090%g; s%:9091%:${SAFROCHAIN_PORT}091%g” ~/.safrochain/config/app.toml

# 9. Optimal configuration + low gas
sed -i -e “s/^minimum-gas-prices *=.*/minimum-gas-prices = \”0.0001usaf\“/” ~/.safrochain/config/app.toml
sed -i ‘s/^pruning *=.*/pruning = “custom”/; s/^pruning-keep-recent *=.*/pruning-keep-recent = ‘200’/; s/^pruning-interval *=.*/pruning-interval = “17”/’ ~/.safrochain/config/app.toml
sed -i ‘s/prometheus = false/prometheus = true/’ ~/.safrochain/config/config.toml
sed -i ‘s/^indexer *=.*/indexer = “null”/’ ~/.safrochain/config/config.toml

# 10. Systemd service (run 24/7)
sudo tee /etc/systemd/system/safrochaind.service > /dev/null <<EOF
[Unit]
Description=Safrochain Validator Node
After=network-online.target

[Service]
User=$USER
ExecStart=$(which safrochaind) start
Restart=always
RestartSec=3
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
EOF

# 11. Download the latest snapshot (sync takes only 5–10 minutes)
cd $HOME
wget -O safrochain-latest.tar.lz4 https://cdn.crxanode.me/safrochain/safrochain-latest.tar.lz4
lz4 -dc safrochain-latest.tar.lz4 | tar -xf - -C ~/.safrochain

# 12. Start the node :
sudo systemctl daemon-reload
sudo systemctl enable safrochaind
sudo systemctl start safrochaind

# Check the log in real-time :
journalctl -u safrochaind -f --no-hostname -o cat

After the node finishes syncing (catching_up = false)

# Create or recover wallet :
safrochaind keys add wallet --recover --keyring-backend test # paste mnemonic

# Faucet (choose 1 of 3 methods)
# Method 1: https://faucet.safrochain.com (250 SAF/time, 12h cooldown)
# Method 2: Discord #testnet-faucet → !faucet address:safro1...
# Method 3: https://faucet.astrostake.xyz/safrochain (5 SAF per transaction, multiple times per day)

# Check balance :
safrochaind query bank balances $(safrochaind keys show wallet -a --keyring-backend test) --node https://rpc.safrochain-t.nodevism.com

# Create validator (stake almost all) :
VALOPER=$(safrochaind keys show wallet --bech val -a --keyring-backend test)

safrochaind tx staking create-validator \
--amount 9000000usaf \
--pubkey $(safrochaind tendermint show-validator) \
--moniker “$MONIKER” \
--chain-id safro-testnet-1 \
--commission-rate=“0.05” \
--commission-max-rate=“0.10” \
--commission-max-change-rate=“0.01” \
--min-self-delegation=“1000000” \
--gas=“auto” \
--gas-adjustment=“1.5” \
--gas-prices=“0.001usaf” \
--from wallet \
--keyring-backend test \
--node https://rpc.safrochain-t.nodevism.com \
--yes

I customized each delegate to be 9 SAF because you can only faucet 10 SAF each time.

Delegate order each time faucet (9 SAF/time) :

VALOPER=$(safrochaind keys show wallet --bech val -a --keyring-backend test)
safrochaind tx staking delegate $VALOPER 9000000usaf --from wallet --keyring-backend test --node https://rpc.safrochain-t.nodevism.com --chain-id safro-testnet-1 --gas-prices 0.001usaf --gas auto --gas-adjustment 1.5 --yes

Rename + details :

safrochaind tx staking edit-validator \
--new-moniker “New_Name” \
--details “Your description here <3” \
--from wallet --keyring-backend test --node https://rpc.safrochain-t.nodevism.com --chain-id safro-testnet-1 --gas-prices 0.001usaf --gas auto --gas-adjustment 1.5 --yes

If you are jailed → unjail immediately : 

safrochaind tx slashing unjail --from wallet --keyring-backend test --node https://rpc.safrochain-t.nodevism.com --chain-id safro-testnet-1 --gas-prices 0.001usaf --gas auto --gas-adjustment 1.5 --yes


Avoid getting jailed next time (must do this immediately):
Add this segment to the service so that the node restarts super fast if there is a crash:

sudo systemctl edit safrochaind

Paste :

[Service]
Restart=always
RestartSec=2
LimitNOFILE=65535

Then :

sudo systemctl daemon-reload
sudo systemctl restart safrochaind

Goodluck !
 
