# PlanQ Mainnet Manual node setup
If you want to setup fullnode manually follow the steps below

## Setting up vars
Here you have to put name of your moniker (validator) that will be visible in explorer
```
NODENAME=<YOUR_MONIKER_NAME_GOES_HERE>
```

Save and import variables into system
```
PLANQ_PORT=33
echo "export NODENAME=$NODENAME" >> $HOME/.bash_profile
if [ ! $WALLET ]; then
	echo "export WALLET=wallet" >> $HOME/.bash_profile
fi
echo "export PLANQ_CHAIN_ID=planq_7070-2" >> $HOME/.bash_profile
echo "export PLANQ_PORT=${PLANQ_PORT}" >> $HOME/.bash_profile
source $HOME/.bash_profile
```

## Update packages
```
sudo apt update && sudo apt upgrade -y
```

## Install dependencies
```
sudo apt install curl tar wget clang pkg-config libssl-dev jq build-essential bsdmainutils git make ncdu gcc git jq chrony liblz4-tool -y
```

## Install go
```
if ! [ -x "$(command -v go)" ]; then
  ver="1.19.3"
  cd $HOME
  wget "https://golang.org/dl/go$ver.linux-amd64.tar.gz"
  sudo rm -rf /usr/local/go
  sudo tar -C /usr/local -xzf "go$ver.linux-amd64.tar.gz"
  rm "go$ver.linux-amd64.tar.gz"
  echo "export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin" >> ~/.bash_profile
  source ~/.bash_profile
fi
```

## Download and build binaries
```
cd $HOME
git clone https://github.com/planq-network/planq.git
cd planq
git fetch
git checkout v1.0.2
make install
```

## Config app
```
planqd config chain-id $PLANQ_CHAIN_ID
planqd config keyring-backend file
planqd config node tcp://localhost:${PLANQ_PORT}657

```

## Init app
```
planqd init $NODENAME --chain-id $PLANQ_CHAIN_ID
```

## Download genesis and addrbook

```
wget -qO $HOME/.planqd/config/genesis.json "https://raw.githubusercontent.com/planq-network/networks/main/mainnet/genesis.json"
wget -O $HOME/.planqd/config/addrbook.json "https://raw.githubusercontent.com/irgirival/planq/main/addrbook.json"
```

## Set peers, gas prices and seeds
```
sed -i.bak -e "s/^minimum-gas-prices *=.*/minimum-gas-prices = \"0.025aplanq\"/;" ~/.planqd/config/app.toml
seeds=`curl -sL https://raw.githubusercontent.com/planq-network/networks/main/mainnet/seeds.txt | awk '{print $1}' | paste -s -d, -`
sed -i.bak -e "s/^seeds =.*/seeds = \"$seeds\"/" ~/.planqd/config/config.toml
sed -i 's/max_num_inbound_peers =.*/max_num_inbound_peers = 100/g' $HOME/.planqd/config/config.toml
sed -i 's/max_num_outbound_peers =.*/max_num_outbound_peers = 100/g' $HOME/.planqd/config/config.toml
```
## Custom Port
```
sed -i.bak -e "s%^proxy_app = \"tcp://127.0.0.1:26658\"%proxy_app = \"tcp://127.0.0.1:${PLANQ_PORT}658\"%; s%^laddr = \"tcp://127.0.0.1:26657\"%laddr = \"tcp://127.0.0.1:${PLANQ_PORT}657\"%; s%^pprof_laddr = \"localhost:6060\"%pprof_laddr = \"localhost:${PLANQ_PORT}060\"%; s%^laddr = \"tcp://0.0.0.0:26656\"%laddr = \"tcp://0.0.0.0:${PLANQ_PORT}656\"%; s%^prometheus_listen_addr = \":26660\"%prometheus_listen_addr = \":${PLANQ_PORT}660\"%" $HOME/.planqd/config/config.toml
sed -i.bak -e "s%^address = \"tcp://0.0.0.0:1317\"%address = \"tcp://0.0.0.0:${PLANQ_PORT}317\"%; s%^address = \":8080\"%address = \":${PLANQ_PORT}080\"%; s%^address = \"0.0.0.0:9090\"%address = \"0.0.0.0:${PLANQ_PORT}090\"%; s%^address = \"0.0.0.0:9091\"%address = \"0.0.0.0:${PLANQ_PORT}091\"%" $HOME/.planqd/config/app.toml
```

## Config pruning
```
pruning="custom"
pruning_keep_recent="100"
pruning_keep_every="0"
pruning_interval="10"
sed -i -e "s/^pruning *=.*/pruning = \"$pruning\"/" $HOME/.planqd/config/app.toml
sed -i -e "s/^pruning-keep-recent *=.*/pruning-keep-recent = \"$pruning_keep_recent\"/" $HOME/.planqd/config/app.toml
sed -i -e "s/^pruning-keep-every *=.*/pruning-keep-every = \"$pruning_keep_every\"/" $HOME/.planqd/config/app.toml
sed -i -e "s/^pruning-interval *=.*/pruning-interval = \"$pruning_interval\"/" $HOME/.planqd/config/app.toml
```

## Set indexer "null"
```
indexer="null" && \
sed -i -e "s/^indexer *=.*/indexer = \"$indexer\"/" $HOME/.planqd/config/config.toml
```

## Enable prometheus
```
sed -i -e "s/prometheus = false/prometheus = true/" $HOME/.planqd/config/config.toml
```

## Reset chain data
```
planqd tendermint unsafe-reset-all --home $HOME/.planqd --keep-addr-book
```

## Create service
```
sudo tee /etc/systemd/system/planqd.service > /dev/null <<EOF
[Unit]
Description=planqd
After=network-online.target

[Service]
User=$USER
ExecStart=$(which planqd) start
Restart=on-failure
RestartSec=3
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
EOF
```

## Register and start service
```
sudo systemctl daemon-reload
sudo systemctl enable planqd
sudo systemctl restart planqd && sudo journalctl -u planqd -f -o cat
```
## Post installation

When installation is finished please load variables into system
```
source $HOME/.bash_profile
```

Next you have to make sure your validator is syncing blocks. You can use command below to check synchronization status
```
planqd status 2>&1 | jq .SyncInfo
```

### (OPTIONAL) State Sync
You can state sync your node in minutes by running commands below
```
N/A
```

### Option (1) Create wallet
To create new wallet you can use command below. Don’t forget to save the mnemonic
```
planqd keys add $WALLET
```

(OPTIONAL) To recover your wallet using seed phrase
```
planqd keys add $WALLET --recover
```

To get current list of wallets
```
planqd keys list
```

### Option (2) Import Metamask wallet
```
planqd keys unsafe-import-eth-key $WALLET <private-key-eth> --keyring-backend file
```

### Save wallet info
Add wallet and valoper address into variables 
```
PLANQD_WALLET_ADDRESS=$(planqd keys show $WALLET -a)
PLANQD_VALOPER_ADDRESS=$(planqd keys show $WALLET --bech val -a)
echo 'export PLANQD_WALLET_ADDRESS='${PLANQD_WALLET_ADDRESS} >> $HOME/.bash_profile
echo 'export PLANQD_VALOPER_ADDRESS='${PLANQD_VALOPER_ADDRESS} >> $HOME/.bash_profile
source $HOME/.bash_profile
```

### Fund your wallet
In order to create validator first you need to fund your wallet with mainnet tokens please refer to this guide : https://docs.planq.network/about/airdrop.html to get mainnet tokens

### Create validator
Before creating validator please make sure that you have at least 1 planq (1 planq is equal to 1000000000000 aplanq) and your node is synchronized

To check your wallet balance:
```
planqd query bank balances $PLANQD_WALLET_ADDRESS
```
> If your wallet does not show any balance than probably your node is still syncing. Please wait until it finish to synchronize and then continue 

To create your validator run command below
```
planqd tx staking create-validator \
  --amount=499000000000000000000aplanq \
  --from=$WALLET \
  --pubkey=$(planqd tendermint show-validator) \
  --moniker=$NODENAME \
  --chain-id=planq_7070-2 \
  --identity=<your-id>\
  --commission-rate="0.10" \
  --commission-max-rate="0.20" \
  --commission-max-change-rate="0.01" \
  --min-self-delegation="1000000" \
  --gas="1000000" \
  --gas-prices="300000000aplanq" \
  --gas-adjustment="1.15"
```

## Security
To protect you keys please make sure you follow basic security rules

### Set up ssh keys for authentication
Good tutorial on how to set up ssh keys for authentication to your server can be found [here](https://www.digitalocean.com/community/tutorials/how-to-set-up-ssh-keys-on-ubuntu-20-04)

### Basic Firewall security
Start by checking the status of ufw.
```
sudo ufw status
```

Sets the default to allow outgoing connections, deny all incoming except ssh and 26656. Limit SSH login attempts
```
sudo ufw default allow outgoing
sudo ufw default deny incoming
sudo ufw allow ssh/tcp
sudo ufw limit ssh/tcp
sudo ufw allow ${PLANQ_PORT}656,${PLANQ_PORT}660/tcp
sudo ufw enable
```

### Check your validator key
```
[[ $(planqd q staking validator $PLANQD_VALOPER_ADDRESS -oj | jq -r .consensus_pubkey.key) = $(planqd status | jq -r .ValidatorInfo.PubKey.value) ]] && echo -e "\n\e[1m\e[32mTrue\e[0m\n" || echo -e "\n\e[1m\e[31mFalse\e[0m\n"
```

### Get list of validators
```
planqd q staking validators -oj --limit=3000 | jq '.validators[] | select(.status=="BOND_STATUS_BONDED")' | jq -r '(.tokens|tonumber/pow(10; 6)|floor|tostring) + " \t " + .description.moniker' | sort -gr | nl
```

## Usefull commands
### Service management
Check logs
```
journalctl -fu planqd -o cat
```

Start service
```
sudo systemctl start planqd
```

Stop service
```
sudo systemctl stop planqd
```

Restart service
```
sudo systemctl restart planqd
```

### Node info
Synchronization info
```
planqd status 2>&1 | jq .SyncInfo
```

Validator info
```
planqd status 2>&1 | jq .ValidatorInfo
```

Node info
```
planqd status 2>&1 | jq .NodeInfo
```

Show node id
```
planqd tendermint show-node-id
```

### Wallet operations
List of wallets
```
planqd keys list
```

Recover wallet
```
planqd keys add $WALLET --recover
```

Delete wallet
```
planqd keys delete $WALLET
```

Get wallet balance
```
planqd query bank balances $PLANQD_WALLET_ADDRESS
```

Transfer funds
```
planqd tx bank send $PLANQD_WALLET_ADDRESS <TO_PLANQ_WALLET_ADDRESS> 1000000000000aplanq
```

### Voting
```
planqd tx gov vote 1 yes --from $WALLET --chain-id=$PLANQ_CHAIN_ID
```

### Staking, Delegation and Rewards
Delegate stake
```
planqd tx staking delegate $PLANQD_VALOPER_ADDRESS 1000000000000aplanq --from=$WALLET --chain-id=$PLANQ_CHAIN_ID --gas=auto
```

Redelegate stake from validator to another validator
```
planqd tx staking redelegate <srcValidatorAddress> <destValidatorAddress> 1000000000000aplanq --from=$WALLET --chain-id=$PLANQ_CHAIN_ID --gas=auto
```

Withdraw all rewards
```
planqd tx distribution withdraw-all-rewards --from=$WALLET --chain-id=$PLANQ_CHAIN_ID --gas=auto
```

Withdraw rewards with commision
```
planqd tx distribution withdraw-rewards $PLANQD_VALOPER_ADDRESS --from=$WALLET --commission --chain-id=$PLANQ_CHAIN_ID
```

### Validator management
Edit validator
```
planqd tx staking edit-validator \
  --moniker=$NODENAME \
  --identity=<your_keybase_id> \
  --website="<your_website>" \
  --details="<your_validator_description>" \
  --chain-id=$PLANQ_CHAIN_ID \
  --from=$WALLET
```

Unjail validator
```
planqd tx slashing unjail \
  --broadcast-mode=block \
  --from=$WALLET \
  --chain-id=$PLANQ_CHAIN_ID \
  --gas=auto
```

### Delete node
This commands will completely remove node from server. Use at your own risk!
```
sudo systemctl stop planqd && \
sudo systemctl disable planqd && \
rm /etc/systemd/system/planqd.service && \
sudo systemctl daemon-reload && \
cd $HOME && \
rm -rf planqd && \
rm -rf .planqd && \
rm -rf $(which planqd)
```

