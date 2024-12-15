Hardware Requirements

Minimum

4CPU 8RAM 160GB
Recommended

8CPU 16RAM 500GB
Rent On Hetzner | Rent On OVH
Dependencies Installation

**Install dependencies for building from source**
```
sudo apt update
sudo apt install -y curl git jq lz4 build-essential
```

**Install Go**
```
sudo rm -rf /usr/local/go
curl -L https://go.dev/dl/go1.22.7.linux-amd64.tar.gz | sudo tar -xzf - -C /usr/local
echo 'export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin' >> $HOME/.profile
source .profile
```

Node Installation

Node Name

Your Node Name
Port prefix

194
# Clone project repository
cd && rm -rf pylons
git clone https://github.com/Pylons-tech/pylons.git
cd pylons
git checkout v1.1.4

# Build binary
make install

# Prepare cosmovisor directories
mkdir -p $HOME/.pylons/cosmovisor/genesis/bin
ln -s $HOME/.pylons/cosmovisor/genesis $HOME/.pylons/cosmovisor/current -f

# Copy binary to cosmovisor directory
cp $(which pylonsd) $HOME/.pylons/cosmovisor/genesis/bin

# Set node CLI configuration
pylonsd config chain-id pylons-mainnet-1
pylonsd config keyring-backend file
pylonsd config node tcp://localhost:19457

# Initialize the node
pylonsd init "Your Node Name" --chain-id pylons-mainnet-1

# Download genesis and addrbook files
curl -L https://snapshots.nodejumper.io/pylons/genesis.json > $HOME/.pylons/config/genesis.json
curl -L https://snapshots.nodejumper.io/pylons/addrbook.json > $HOME/.pylons/config/addrbook.json

# Set seeds
sed -i -e 's|^seeds *=.*|seeds = "0d25c5db4cbdc4171c8272278040db774011c268@5.161.229.9:26656,e9e64412c3d43de4f2e5f7a3e9289b4190e4ed78@88.198.32.17:33656,030e6a01aef8913bcee33b957e9204986203bc81@135.125.4.73:46656"|' $HOME/.pylons/config/config.toml

# Set minimum gas price
sed -i -e 's|^minimum-gas-prices *=.*|minimum-gas-prices = "0.0025ubedrock"|' $HOME/.pylons/config/app.toml

# Set pruning
sed -i \
  -e 's|^pruning *=.*|pruning = "custom"|' \
  -e 's|^pruning-keep-recent *=.*|pruning-keep-recent = "100"|' \
  -e 's|^pruning-interval *=.*|pruning-interval = "17"|' \
  $HOME/.pylons/config/app.toml

# Enable prometheus
sed -i -e 's|^prometheus *=.*|prometheus = true|' $HOME/.pylons/config/config.toml

# Change ports
sed -i -e "s%:1317%:19417%; s%:8080%:19480%; s%:9090%:19490%; s%:9091%:19491%; s%:8545%:19445%; s%:8546%:19446%; s%:6065%:19465%" $HOME/.pylons/config/app.toml
sed -i -e "s%:26658%:19458%; s%:26657%:19457%; s%:6060%:19460%; s%:26656%:19456%; s%:26660%:19461%" $HOME/.pylons/config/config.toml

# Download latest chain data snapshot
curl "https://snapshots.nodejumper.io/pylons/pylons_latest.tar.lz4" | lz4 -dc - | tar -xf - -C "$HOME/.pylons"

# Install Cosmovisor
go install cosmossdk.io/tools/cosmovisor/cmd/cosmovisor@v1.7.0

# Create a service
sudo tee /etc/systemd/system/pylons.service > /dev/null << EOF
[Unit]
Description=Pylons node service
After=network-online.target
[Service]
User=$USER
WorkingDirectory=$HOME/.pylons
ExecStart=$(which cosmovisor) run start
Restart=on-failure
RestartSec=5
LimitNOFILE=65535
Environment="DAEMON_HOME=$HOME/.pylons"
Environment="DAEMON_NAME=pylonsd"
Environment="UNSAFE_SKIP_BACKUP=true"
Environment="DAEMON_ALLOW_DOWNLOAD_BINARIES=true"
[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload
sudo systemctl enable pylons.service

# Start the service and check the logs
sudo systemctl start pylons.service
sudo journalctl -u pylons.service -f --no-hostname -o cat
Secure Server Setup (Optional)

# generate ssh keys, if you don't have them already, DO IT ON YOUR LOCAL MACHINE
ssh-keygen -t rsa

# save the output, we'll use it later on instead of YOUR_PUBLIC_SSH_KEY
cat ~/.ssh/id_rsa.pub
# upgrade system packages
sudo apt update
sudo apt upgrade -y

# add new admin user
sudo adduser admin --disabled-password -q

# upload public ssh key, replace YOUR_PUBLIC_SSH_KEY with the key above
mkdir /home/admin/.ssh
echo "YOUR_PUBLIC_SSH_KEY" >> /home/admin/.ssh/authorized_keys
sudo chown admin: /home/admin/.ssh
sudo chown admin: /home/admin/.ssh/authorized_keys

echo "admin ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers

# disable root login, disable password authentication, use ssh keys only
sudo sed -i 's|^PermitRootLogin .*|PermitRootLogin no|' /etc/ssh/sshd_config
sudo sed -i 's|^ChallengeResponseAuthentication .*|ChallengeResponseAuthentication no|' /etc/ssh/sshd_config
sudo sed -i 's|^#PasswordAuthentication .*|PasswordAuthentication no|' /etc/ssh/sshd_config
sudo sed -i 's|^#PermitEmptyPasswords .*|PermitEmptyPasswords no|' /etc/ssh/sshd_config
sudo sed -i 's|^#PubkeyAuthentication .*|PubkeyAuthentication yes|' /etc/ssh/sshd_config

sudo systemctl restart sshd

# install fail2ban
sudo apt install -y fail2ban

# install and configure firewall
sudo apt install -y ufw
sudo ufw default allow outgoing
sudo ufw default deny incoming
sudo ufw allow ssh
sudo ufw allow 9100
sudo ufw allow 26656

# make sure you expose ALL necessary ports, only after that enable firewall
sudo ufw enable

# make terminal colorful
sudo su - admin
source <(curl -s https://raw.githubusercontent.com/nodejumper-org/cosmos-scripts/master/utils/enable_colorful_bash.sh)

# update servername, if needed, replace YOUR_SERVERNAME with wanted server name
sudo hostnamectl set-hostname YOUR_SERVERNAME

# now you can logout (exit) and login again using ssh admin@YOUR_SERVER_IP
