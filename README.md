# Babylon Node Setup Guide

This comprehensive guide is designed to assist you in setting up a Babylon node on the bbn-test-2 network. By following the outlined steps, you'll be well on your way to becoming a validator on the Babylon system.

## Table of Contents
- [Setup Validator Name](#setting-up-validator-name)
- [Install Dependencies](#installing-dependencies)
  - [Update System and Install Build Tools](#update-system-and-build-tools)
  - [Install Go](#install-go)
- [Download and Build Binaries](#download-and-build-binaries)
- [Install Cosmovisor and Create a Service](#install-cosmovisor-and-create-the-service)
- [Initialize the Node](#initialize-the-node)
- [Download Latest Chain Snapshot](#download-latest-chain-snapshot)
- [Start Service and Check the Logs](#start-service-and-check-the-logs)
- [Becoming a Validator](#becoming-a-validator)
  - [Create a Keyring and Get Funds](#1-create-a-keyring-and-get-funds)
    - [Add New Key](#add-new-key)
    - [Optional Recover Existing Key](#optional-recover-existing-key)
    - [Request Funds from the Babylon Testnet Faucet](#request-funds-from-the-babylon-testnet-faucet)
  - [Create a BLS Key](#2-create-a-bls-key)
  - [Modify the Configuration](#3-modify-the-configuration)
  - [Create the Validator](#4-create-the-validator)

## Setting Up Validator Name

Replace `YOUR_MONIKER_GOES_HERE` with your validator name.
```
MONIKER="YOUR_MONIKER_GOES_HERE"
```

## Installing Dependencies
# Update System and Build Tools

```
sudo apt -q update
sudo apt -qy install curl git jq lz4 build-essential
sudo apt -qy upgrade
```

# Install Go
```
sudo rm -rf /usr/local/go
curl -Ls https://go.dev/dl/go1.20.13.linux-amd64.tar.gz | sudo tar -xzf - -C /usr/local
eval $(echo 'export PATH=$PATH:/usr/local/go/bin' | sudo tee /etc/profile.d/golang.sh)
eval $(echo 'export PATH=$PATH:$HOME/go/bin' | tee -a $HOME/.profile)
```

# Download and Build Binaries

```
# Clone project repository
cd $HOME
rm -rf babylon
git clone https://github.com/babylonchain/babylon.git
cd babylon
git checkout v0.7.2

# Build binaries
make build

# Prepare binaries for Cosmovisor
mkdir -p $HOME/.babylond/cosmovisor/genesis/bin
mv build/babylond $HOME/.babylond/cosmovisor/genesis/bin/
rm -rf build

# Create application symlinks
sudo ln -s $HOME/.babylond/cosmovisor/genesis $HOME/.babylond/cosmovisor/current -f
sudo ln -s $HOME/.babylond/cosmovisor/current/bin/babylond /usr/local/bin/babylond -f
```

# Install Cosmovisor and create the service
```
# Download and install Cosmovisor
go install cosmossdk.io/tools/cosmovisor/cmd/cosmovisor@v1.5.0

# Create service
sudo tee /etc/systemd/system/babylon.service > /dev/null << EOF
[Unit]
Description=babylon node service
After=network-online.target

[Service]
User=$USER
ExecStart=$(which cosmovisor) run start
Restart=on-failure
RestartSec=10
LimitNOFILE=65535
Environment="DAEMON_HOME=$HOME/.babylond"
Environment="DAEMON_NAME=babylond"
Environment="UNSAFE_SKIP_BACKUP=true"
Environment="PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin:$HOME/.babylond/cosmovisor/current/bin"

[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload
sudo systemctl enable babylon.service
```

# Initialize the node
```
# Set node configuration
babylond config chain-id bbn-test-2
babylond config keyring-backend test
babylond config node tcp://localhost:16457

# Initialize the node
babylond init $MONIKER --chain-id bbn-test-2

# Download genesis and addrbook
curl -Ls https://snapshots.kjnodes.com/babylon-testnet/genesis.json > $HOME/.babylond/config/genesis.json
curl -Ls https://snapshots.kjnodes.com/babylon-testnet/addrbook.json > $HOME/.babylond/config/addrbook.json

# Add seeds
sed -i -e "s|^seeds *=.*|seeds = \"3f472746f46493309650e5a033076689996c8881@babylon-testnet.rpc.kjnodes.com:16459\"|" $HOME/.babylond/config/config.toml

# Set minimum gas price
sed -i -e "s|^minimum-gas-prices *=.*|minimum-gas-prices = \"0.00001ubbn\"|" $HOME/.babylond/config/app.toml

# Set pruning
sed -i \
  -e 's|^pruning *=.*|pruning = "custom"|' \
  -e 's|^pruning-keep-recent *=.*|pruning-keep-recent = "100"|' \
  -e 's|^pruning-keep-every *=.*|pruning-keep-every = "0"|' \
  -e 's|^pruning-interval *=.*|pruning-interval = "19"|' \
  $HOME/.babylond/config/app.toml

# Set custom ports
sed -i -e "s%^proxy_app = \"tcp://127.0.0.1:26658\"%proxy_app = \"tcp://127.0.0.1:16458\"%; s%^laddr = \"tcp://127.0.0.1:26657\"%laddr = \"tcp://127.0.0.1:16457\"%; s%^pprof_laddr = \"localhost:6060\"%pprof_laddr = \"localhost:16460\"%; s%^laddr = \"tcp://0.0.0.0:26656\"%laddr = \"tcp://0.0.0.0:16456\"%; s%^prometheus_listen_addr = \":26660\"%prometheus_listen_addr = \":16466\"%" $HOME/.babylond/config/config.toml
sed -i -e "s%^address = \"tcp://localhost:1317\"%address = \"tcp://0.0.0.0:16417\"%; s%^address = \":8080\"%address = \":16480\"%; s%^address = \"localhost:9090\"%address = \"0.0.0.0:16490\"%; s%^address = \"localhost:9091\"%address = \"0.0.0.0:16491\"%; s%:8545%:16445%; s%:8546%:16446%; s%:6065%:16465%" $HOME/.babylond/config/app.toml
```

# Download latest chain snapshot
```
curl -L https://snapshots.kjnodes.com/babylon-testnet/snapshot_latest.tar.lz4 | tar -Ilz4 -xf - -C $HOME/.babylond
[[ -f $HOME/.babylond/data/upgrade-info.json ]] && cp $HOME/.babylond/data/upgrade-info.json $HOME/.babylond/cosmovisor/genesis/upgrade-info.json
```

# Start service and check the logs
```
sudo systemctl start babylon.service && sudo journalctl -u babylon.service -f --no-hostname -o cat
```

## Becoming a Validator
# 1. Create a Keyring and Get Funds
Before proceeding, ensure you have sufficient funds for two purposes:

1. Providing a self-delegation
2. Paying for transaction fees for BLS signature transactions
Currently, only the test keyring backend is supported.


### ADD NEW KEY
```
babylond keys add wallet
```

### (OPTIONAL) RECOVER EXISTING KEY
```
babylond keys add wallet --recover
```

### REQUEST FUNDS FROM THE BABYLON TESTNET FAUCET
Head to the #faucet channel of the official Discord server and request funds by providing your wallet address.

# 2. Create a BLS key

Using the address that you created on the previous step.
```
babylond create-bls-key $(babylond keys show wallet -a)
```
This command will create a BLS key and add it to the $HOME/.babylond/config/priv_validator_key.json. This is the same file that stores the private key that the validator uses to sign blocks. Please ensure that this file is secured properly.

Restart your node to load this key into memory:

```
sudo systemctl restart babylon.service
```
# 3. Modify the Configuration
Edit $HOME/.babylond/config/app.toml and set the key name to the one holding funds on your keyring:
```
sed -i -e "s|^key-name *=.*|key-name = \"wallet\"|" $HOME/.babylond/config/app.toml
```
Finally, it is strongly recommended to modify the timeout_commit value under $HOME/.babylond/config/config.toml. This value specifies how long a validator will wait before commiting a block before starting on a new height. Given that Babylon aims to have a 10 second time between blocks, set this value to:
```
sed -i -e "s|^timeout_commit *=.*|timeout_commit = \"10s\"|" $HOME/.babylond/config/config.toml
```
# 4. Create the Validator
Use the following command to create the validator:
```
# Please make sure you have adjusted **moniker**, **identity**, **details** and **website** to match your values.

babylond tx checkpointing create-validator \
--amount 1000000ubbn \
--pubkey $(babylond tendermint show-validator) \
--moniker "YOUR_MONIKER_NAME" \
--chain-id bbn-test-2 \
--commission-rate 0.05 \
--commission-max-rate 0.20 \
--commission-max-change-rate 0.01 \
--min-self-delegation 1 \
--from wallet \
--gas-adjustment 1.4 \
--gas auto \
--gas-prices 0.00001ubbn \
-y
```
Congrats! You are now a (inactive*) validator on the Babylon system.

