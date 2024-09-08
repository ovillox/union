Manual Installation
Official Documentation
Recommended Hardware: 8 Cores, 15GB RAM, 250GB of storage (NVME)

**install dependencies, if needed**
```
sudo apt update && sudo apt upgrade -y
sudo apt install curl git wget htop tmux build-essential jq make lz4 gcc unzip -y
```

**install go, if needed**
```
cd $HOME
VER="1.21.3"
wget "https://golang.org/dl/go$VER.linux-amd64.tar.gz"
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf "go$VER.linux-amd64.tar.gz"
rm "go$VER.linux-amd64.tar.gz"
[ ! -f ~/.bash_profile ] && touch ~/.bash_profile
echo "export PATH=$PATH:/usr/local/go/bin:~/go/bin" >> ~/.bash_profile
source $HOME/.bash_profile
[ ! -d ~/go/bin ] && mkdir -p ~/go/bin
```

**set vars**
```
echo "export WALLET="wallet"" >> $HOME/.bash_profile
echo "export MONIKER="test"" >> $HOME/.bash_profile
echo "export UNION_CHAIN_ID="union-testnet-8"" >> $HOME/.bash_profile
echo "export UNION_PORT="23"" >> $HOME/.bash_profile
source $HOME/.bash_profile
```

**download binary**
```
cd $HOME
wget -O uniond https://testnet-files.itrocket.net/union/uniond
chmod +x uniond
mv uniond $HOME/go/bin/
```

**config and init app**
```
uniond --home $HOME/.union init $MONIKER --chain-id union-testnet-8
```

**download genesis and addrbook**
```
wget -O $HOME/.union/config/genesis.json https://server-4.itrocket.net/testnet/union/genesis.json
wget -O $HOME/.union/config/addrbook.json  https://server-4.itrocket.net/testnet/union/addrbook.json
```

**set seeds and peers**
```
SEEDS="2812a4ae3ebfba02973535d05d2bbcc80b7d215f@union-testnet-seed.itrocket.net:23656"
PEERS="a05dde8737e66c99260edfd45180055fe7f8bd9d@union-testnet-peer.itrocket.net:23656,571b775d217d2e56cd9914134513c14507e30b5b@188.40.126.219:26656,7314fd2235940db754af316446869af4ef2ffe5c@65.108.40.246:26656,71b1273c3e944fc9df0c0d39fe53931ed12b7674@65.108.131.146:27101,9fa3e7248de4dc1f639e0d89470bf443d4a427ba@65.109.113.233:24656,d513bc19be599c2de8fcde0d7b4c997fc5e7b197@65.109.126.24:26671,c51587b3d222faa9a13c0354ec87cdc2902ce987@198.244.179.173:26656,5f498ea2735387e31c5076ea54b1463138d9755f@185.165.170.147:26656,082d00f596f1823041ec8ad570e7da3af4c2c4bd@62.171.160.155:26656,c778da65696e032ff92375c9b3a296d54fa89126@195.201.245.178:56076,19ae23577b8ad3c99e293dfb2d32ba20fdb15d04@95.217.196.224:17156,13abf5bb34c6811514a07855364132b03ad05b3d@93.190.140.5:26656"
sed -i -e "/^\[p2p\]/,/^\[/{s/^[[:space:]]*seeds *=.*/seeds = \"$SEEDS\"/}" \
       -e "/^\[p2p\]/,/^\[/{s/^[[:space:]]*persistent_peers *=.*/persistent_peers = \"$PEERS\"/}" $HOME/.union/config/config.toml
```

**set custom ports in app.toml**
```
sed -i.bak -e "s%:1317%:${UNION_PORT}317%g;
s%:8080%:${UNION_PORT}080%g;
s%:9090%:${UNION_PORT}090%g;
s%:9091%:${UNION_PORT}091%g;
s%:8545%:${UNION_PORT}545%g;
s%:8546%:${UNION_PORT}546%g;
s%:6065%:${UNION_PORT}065%g" $HOME/.union/config/app.toml
```

**set custom ports in config.toml file**
```
sed -i.bak -e "s%:26658%:${UNION_PORT}658%g;
s%:26657%:${UNION_PORT}657%g;
s%:6060%:${UNION_PORT}060%g;
s%:26656%:${UNION_PORT}656%g;
s%^external_address = \"\"%external_address = \"$(wget -qO- eth0.me):${UNION_PORT}656\"%;
s%:26660%:${UNION_PORT}660%g" $HOME/.union/config/config.toml
```

**config pruning**
```
sed -i -e "s/^pruning *=.*/pruning = \"custom\"/" $HOME/.union/config/app.toml
sed -i -e "s/^pruning-keep-recent *=.*/pruning-keep-recent = \"100\"/" $HOME/.union/config/app.toml
sed -i -e "s/^pruning-interval *=.*/pruning-interval = \"50\"/" $HOME/.union/config/app.toml
```

# set minimum gas price, enable prometheus and disable indexing
sed -i 's|minimum-gas-prices =.*|minimum-gas-prices = "0.0muno"|g' $HOME/.union/config/app.toml
sed -i -e "s/prometheus = false/prometheus = true/" $HOME/.union/config/config.toml
sed -i -e "s/^indexer *=.*/indexer = \"null\"/" $HOME/.union/config/config.toml

# create service file
sudo tee /etc/systemd/system/uniond.service > /dev/null <<EOF
[Unit]
Description=Union node
After=network-online.target
[Service]
User=$USER
WorkingDirectory=$HOME/.union
ExecStart=$(which uniond) start --home $HOME/.union
Restart=on-failure
RestartSec=5
LimitNOFILE=65535
[Install]
WantedBy=multi-user.target
EOF

# reset and download snapshot
uniond tendermint unsafe-reset-all --home $HOME/.union --home $HOME/.union
if curl -s --head curl https://server-4.itrocket.net/testnet/union/union_2024-08-27_2680679_snap.tar.lz4 | head -n 1 | grep "200" > /dev/null; then
  curl https://server-4.itrocket.net/testnet/union/union_2024-08-27_2680679_snap.tar.lz4 | lz4 -dc - | tar -xf - -C $HOME/.union
    else
  echo "no snapshot founded"
fi

# enable and start service
sudo systemctl daemon-reload
sudo systemctl enable uniond
sudo systemctl restart uniond && sudo journalctl -u uniond -f
Automatic Installation
pruning: custom: 100/0/10 | indexer: null

source <(curl -s https://itrocket.net/api/testnet/union/autoinstall/)
Create wallet
# to create a new wallet, use the following command. don’t forget to save the mnemonic
uniond keys add $WALLET

# to restore exexuting wallet, use the following command
uniond keys add $WALLET --recover

# save wallet and validator address
WALLET_ADDRESS=$(uniond keys show $WALLET -a)
VALOPER_ADDRESS=$(uniond keys show $WALLET --bech val -a)
echo "export WALLET_ADDRESS="$WALLET_ADDRESS >> $HOME/.bash_profile
echo "export VALOPER_ADDRESS="$VALOPER_ADDRESS >> $HOME/.bash_profile
source $HOME/.bash_profile

# check sync status, once your node is fully synced, the output from above will print "false"
uniond status 2>&1 | jq 

# before creating a validator, you need to fund your wallet and check balance
uniond query bank balances $WALLET_ADDRESS 
Create validator
Moniker
Identity
Details
I love blockchain ❤️
Amount, muno
1000000
Commission rate
0.1
Commission max rate
0.2
Commission max change rate
0.01
Website
uniond tx staking create-validator \
--amount 1000000muno \
--from $WALLET \
--commission-rate 0.1 \
--commission-max-rate 0.2 \
--commission-max-change-rate 0.01 \
--min-self-delegation 1 \
--pubkey $(uniond tendermint show-validator) \
--moniker "test" \
--identity "" \
--website "" \
--details "I love blockchain ❤️" \
--chain-id union-testnet-8 \
--gas auto --gas-adjustment 1.5 \
-y
Monitoring
If you want to have set up a monitoring and alert system use our cosmos nodes monitoring guide with tenderduty

Security
To protect you keys please don`t share your privkey, mnemonic and follow basic security rules

Set up ssh keys for authentication
You can use this guide to configure ssh authentication and disable password authentication on your server

Firewall security
Set the default to allow outgoing connections, deny all incoming, allow ssh and node p2p port

sudo ufw default allow outgoing 
sudo ufw default deny incoming 
sudo ufw allow ssh/tcp 
sudo ufw allow ${UNION_PORT}656/tcp
sudo ufw enable
Delete node
sudo systemctl stop uniond
sudo systemctl disable uniond
sudo rm -rf /etc/systemd/system/uniond.service
sudo rm $(which uniond)
sudo rm -rf $HOME/.union
sed -i "/UNION_/d" $HOME/.bash_profile
