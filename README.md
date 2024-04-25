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

# Adjust IP:PORT to match avail validator metrics endpoint in prometheus config file
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

![image](https://github.com/notional-labs/subnode/assets/50621007/b0da0a66-911c-421f-8efb-325495c937d2)


# (OPTIONAL) Install node exporter on a validator host

## Download binaries
```bash
wget https://github.com/prometheus/node_exporter/releases/download/v1.8.0/node_exporter-1.8.0.linux-amd64.tar.gz
tar xvfz node_exporter-*.*-amd64.tar.gz
sudo mv node_exporter-*.*-amd64/node_exporter /usr/local/bin/
rm node_exporter-* -rf

sudo useradd -rs /bin/false node_exporter
```

## Create service
```bash
sudo tee <<EOF >/dev/null /etc/systemd/system/node_exporter.service
[Unit]
Description=Node Exporter
After=network.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=multi-user.target
EOF

systemctl enable node_exporter
systemctl daemon-reload
```

## Start service and check logs
```bash
systemctl restart node_exporter && journalctl -fu node_exporter -o cat
```

