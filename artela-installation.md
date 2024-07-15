# Artela Installation Guide

## Preparation

### **Install Dependencies**

Update system package and install build tools
```bash
sudo apt -q update
sudo apt -qy install curl git jq lz4 build-essential fail2ban ufw
sudo apt -qy upgrade
```

### **Install Go**

Install go version 1.20.7
```bash
sudo rm -rf /usr/local/go
curl -Ls https://go.dev/dl/go1.20.7.linux-amd64.tar.gz | sudo tar -xzf - -C /usr/local
eval $(echo 'export PATH=$PATH:/usr/local/go/bin' | sudo tee /etc/profile.d/golang.sh)
eval $(echo 'export PATH=$PATH:$HOME/go/bin' | tee -a $HOME/.profile)
```

### **Build Binaries**

Cloning project repository & Compile binaries
```bash
cd $HOME
rm -rf artela
git clone https://github.com/artela-network/artela artela
cd artela
git checkout v0.4.7-rc6
make build
```

Prepare binaries for cosmovisor
```bash
mkdir -p $HOME/.artelad/cosmovisor/genesis/binmv build/artelad $HOME/.artelad/cosmovisor/genesis/bin/rm -rf build
```

Create symlinks
```bash
sudo ln -s $HOME/.artelad/cosmovisor/genesis $HOME/.artelad/cosmovisor/current -fsudo ln -s $HOME/.artelad/cosmovisor/current/bin/artelad /usr/local/bin/artelad -f
```

### **Cosmovisor Setup**
Install cosmovisor
```bash
go install cosmossdk.io/tools/cosmovisor/cmd/cosmovisor@v1.5.0
```

---

## Configuring and Initialization

### **Configure Moniker**

Replace <your-moniker-name> with your own validator name
```bash
MONIKER="<your-moniker-name>"
```

### **Create Service**

Create a systemd service
```bash
sudo tee /etc/systemd/system/artela.service > /dev/null << EOF
[Unit]
Description=artela node service
After=network-online.target
 
[Service]
User=$USER
ExecStart=$(which cosmovisor) run start
Restart=on-failure
RestartSec=10
LimitNOFILE=65535
Environment="DAEMON_HOME=$HOME/.artelad"
Environment="DAEMON_NAME=artelad"
Environment="UNSAFE_SKIP_BACKUP=true"
 
[Install]
WantedBy=multi-user.target
EOF
```

### **Enable Service**

Enable artela systemd service
```bash
sudo systemctl daemon-reloadsudo systemctl enable artela
```

### **Initialize Node**

Setting node configuration
```bash
artelad config chain-id artela_11822-1
artelad config keyring-backend test
artelad config node tcp://localhost:23457
```

Initialize node
```bash
artelad init $MONIKER --chain-id artela_11822-1
```

### **Download Genesis & Addrbook**

Download genesis & addrbook file
```bash
curl -Ls https://snap.orydysst.net/artela-testnet/genesis.json > $HOME/.artelad/config/genesis.json
curl -Ls https://snap.orydysst.net/artela-testnet/addrbook.json > $HOME/.artelad/config/addrbook.json
```

### **Configure Seeds**

Setting up a seed peers
```bash
sed -i -e "s|^seeds *=.*|seeds = \"d1d43cc7c7aef715957289fd96a114ecaa7ba756@testnet-seeds.orydysst.net:23410\"|" $HOME/.artelad/config/config.toml
```

### **Configure Gas Prices**

Setting up a gas prices
```bash
sed -i -e "s|^minimum-gas-prices *=.*|minimum-gas-prices = \"0.025uart\"|" $HOME/.artelad/config/app.toml
```

### **Pruning Setting**

Configure pruning setting
```bash
sed -i \
  -e 's|^pruning *=.*|pruning = "custom"|' \
  -e 's|^pruning-keep-recent *=.*|pruning-keep-recent = "100"|' \
  -e 's|^pruning-keep-every *=.*|pruning-keep-every = "0"|' \
  -e 's|^pruning-interval *=.*|pruning-interval = "19"|' \
  $HOME/.artelad/config/app.toml
```

### **Download Snapshots**

Download latest chain snapshot
```bash
curl -L https://snap.orydysst.net/artela-testnet/artela-latest.tar.lz4 | tar -Ilz4 -xf - -C $HOME/.artelad
[[ -f $HOME/.artelad/data/upgrade-info.json ]] && cp $HOME/.artelad/data/upgrade-info.json $HOME/.artelad/cosmovisor/genesis/upgrade-info.json
```

---
## Final step
### **Start Service**

```bash
sudo systemctl start artela
```
