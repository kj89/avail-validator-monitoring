# Install updates and dependencies
```bash
sudo apt-get update
sudo apt install jq python3-pip -y
```

# Install docker
```bash
sudo apt-get install ca-certificates curl gnupg lsb-release wget -y
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
sudo chmod a+r /usr/share/keyrings/docker-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io -y
```

# Install docker compose
```bash
docker_compose_version=$(wget -qO- https://api.github.com/repos/docker/compose/releases/latest | jq -r ".tag_name")
sudo wget -O /usr/bin/docker-compose "https://github.com/docker/compose/releases/download/${docker_compose_version}/docker-compose-`uname -s`-`uname -m`"
sudo chmod +x /usr/bin/docker-compose
```

# Clone repository
```bash
cd $HOME && rm -rf avail-validator-monitoring
git clone https://github.com/kj89/avail-validator-monitoring.git
```

# Copy _.env.example_ into _.env_
```bash
cp $HOME/avail-validator-monitoring/config/.env.example $HOME/avail-validator-monitoring/config/.env
```

# Update values in _.env_ file
```bash
vim $HOME/avail-validator-monitoring/config/.env
```

| KEY | VALUE |
|---------------|-------------|
| TELEGRAM_ADMIN | Your user id you can get from [@userinfobot](https://t.me/userinfobot). The bot will only reply to messages sent from the user. All other messages are dropped and logged on the bot's console |
| TELEGRAM_TOKEN | Your telegram bot access token you can get from [@botfather](https://telegram.me/botfather). To generate new token just follow a few simple steps described [here](https://core.telegram.org/bots#6-botfather) |

# Export _.env_ file values into _.bash_profile_
```bash
echo "export $(xargs < $HOME/avail-validator-monitoring/config/.env)" > $HOME/.bash_profile
source $HOME/.bash_profile
```

# Adjust IP:PORT to match lava provider metrics endpoint in prometheus config file
```bash
vim $HOME/avail-validator-monitoring/prometheus/prometheus.yml
```

# Run docker-compose
Deploy the monitoring stack
```bash
cd $HOME/avail-validator-monitoring && docker-compose up -d
```

ports used:
- `8080` (alertmanager-bot)
- `9090` (prometheus)
- `9093` (alertmanager)
- `9999` (grafana)

# Install lava-exporter

## Download binaries
```bash
sudo curl -Ls https://github.com/MELLIFERA-Labs/lava-exporter/releases/download/v1.0.0/lava-exporter-linux-v1.0.0-amd64 > /usr/local/bin/lava-exporter
sudo chmod +x /usr/local/bin/lava-exporter
```

## Set config
```bash
mkdir -p $HOME/lava-exporter
sudo tee $HOME/lava-exporter/config.toml > /dev/null << EOF
# array of lava rest api endpoints. Will be used next URL if previous is down
lava_rest_api = ['http://127.0.0.1:14417']
# Your chains to track for your provider.
# Name should be match with the chainID from this list:
# https://rest-public-rpc.lavanet.xyz/lavanet/lava/spec/show_all_chains
# Otherwise provider_frozen always will be 0
chains = ['EVMOS', 'EVMOST', 'AXELAR', 'AXELART', 'AGR', 'AGRT', 'NEAR', 'NEART']
# Your provider address
lava_provider_address = 'lava@13j4r22xd0tegzagwhx39ptzxpzlvgyv7ykgrvw'
EOF
```

## Create service
```bash
sudo tee /etc/systemd/system/lava-exporter.service > /dev/null << EOF
[Unit]
Description=Lava Exporter
After=network-online.target

[Service]
User=$USER
ExecStart=/usr/local/bin/lava-exporter start --config $HOME/lava-exporter/config.toml --port 14440 
Restart=on-failure
RestartSec=3
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
EOF

systemctl enable lava-exporter
systemctl daemon-reload
```

## Start service and check logs
```bash
systemctl restart lava-exporter && journalctl -fu lava-exporter -o cat
```
