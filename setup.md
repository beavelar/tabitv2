# Setup Steps

The following assumes the installation is being done in a Ubuntu server. For other distros check if the command running is the right one for the distro

## Base Setup

The minimum setup to get a Minecraft server running. This will not include extra setup like setting up Grafana dashboards for health metrics

### General update

- Update and upgrade packages

  ```bash
  sudo apt update
  sudo apt upgrade
  ```

### Changing SSH port

- Open the SSH daemon configuration file to edit the port

  - Change "Port" to 1738

  ```bash
  sudo vi /etc/ssh/sshd_config
  ```

- Restart the SSH service

  ```bash
  sudo systemctl restart ssh
  ```

### Enabling firewall and configuration

- Enable ufw firewall, add firewall rules and reload

  ```bash
  sudo ufw allow 1738/tcp # SSH port
  sudo ufw allow 25565/tcp # Minecraft port
  sudo ufw allow 80/tcp # Needed for certbot potentially
  sudo ufw allow 443/tcp # nginx server for reverse proxy
  sudo ufw default deny incoming # Deny all incoming by default
  sudo ufw default allow outgoing # Allow all outgoing by default - TODO: Investigate which ones to restrict to not leave open
  sudo ufw enable
  sudo ufw reload
  ```

- `EXTERNAL STEP`
  - Add the firewall rules above within the hosting provider as well

### Creating sudo user account and removing root account

- Create sudo user account and group

  ```bash
  sudo groupadd -g 8371 beamcs
  sudo adduser -u 8371 --gid 8371 beamcs
  sudo usermod -aG sudo beamcs
  ```

- Open the SSH daemon configuration file

  - Change `PermitRootLogin` to **no**

  ```bash
  sudo vi /etc/ssh/sshd_config
  ```

- Lock the root account

  ```bash
  sudo passwd -l root
  ```

### Creating non-sudo accounts for the services

- Create beamc and beaprom user and group

  - beamc - Used to start/stop the minecraft server and any installations necessary for minecraft

  - beaprom - Used to start/stop the prometheus servers and any installations necessary for prometheus

  ```bash
  sudo groupadd -g 1738 beamc
  sudo useradd --uid 1738 --gid 1738 -m beamc
  sudo groupadd -g 6251 beaprom
  sudo useradd --uid 6251 --gid 6251 -m beaprom
  ```

- Add a password to the user accounts

  ```bash
  sudo passwd beamc
  sudo passwd beaprom
  ```

- Update the bash used for those accounts to when ssh'ing the bash is right

  ```bash
  sudo chsh -s /bin/bash beamc
  sudo chsh -s /bin/bash beaprom
  ```

- Enable lingering for the user accounts

  ```bash
  sudo loginctl enable-linger beamc
  sudo loginctl enable-linger beaprom
  ```

### Setting up pubkey authentication and removing password authentication

- `EXTERNAL STEP`

  - In the host machine, generate a key for each account (`beamc`, `beamcs`, and `beaprom`)

  ```bash
  ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519_linodemc_beamc
  ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519_linodemc_beamcs
  ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519_linodemc_beaprom
  ```

- In each account create the `.ssh` directory

  ```bash
  mkdir -p ~/.ssh
  ```

- In each account create a `authorized_keys` file

  - Copy/paste the content of the `id_ed25519_linodemc_<user>.pub` file created above into this `authorized_keys` file

  ```bash
  vi ~/.ssh/authorized_keys
  ```

- In each account update the `.ssh` directory and file permissions

  ```bash
  chmod 700 ~/.ssh
  chmod 600 ~/.ssh/authorized_keys
  ```

- In the sudo account open the SSH daemon configuration file

  - Set `PasswordAuthentication` to **no**

  - Set `PubkeyAuthentication` to **yes**

  ```bash
  sudo vi /etc/ssh/sshd_config
  ```

- In the sudo account restart the SSH service

  ```bash
  sudo systemctl restart ssh
  ```

- `EXTERNAL STEP`

  - Create or edit the `~/.ssh/config` to copy the following content

  ```
  Host linodemc-beamc
      HostName <ip-address>
      User beamc
      IdentityFile ~/.ssh/id_ed25519_linodemc_beamc
      Port 1738

  Host linodemc-beamcs
      HostName <ip-address>
      User beamcs
      IdentityFile ~/.ssh/id_ed25519_linodemc_beamcs
      Port 1738

  Host linodemc-beaprom
      HostName <ip-address>
      User beaprom
      IdentityFile ~/.ssh/id_ed25519_linodemc_beaprom
      Port 1738
  ```

  _When ssh'ing or scp'ing, use the "Host" value to connect_

### Setup Java

- In the beamc account, create `~/installs` directory (if it doesn't already exist)

  ```bash
  mkdir ~/installs
  ```

- Create `java` directory and navigate to it

  ```bash
  mkdir -p ~/installs/java
  cd ~/installs/java
  ```

- Download Java

  - The following command will download a specific version from Adoption

  - Verify this version is the right version and if it's the latest

  ```bash
  wget https://adoptium.net/download?link=https%3A%2F%2Fgithub.com%2Fadoptium%2Ftemurin21-binaries%2Freleases%2Fdownload%2Fjdk-21.0.9%252B10%2FOpenJDK21U-jdk_x64_linux_hotspot_21.0.9_10.tar.gz&vendor=Adoptium
  ```

- Open the .bashrc file

  ```bash
  vi ~/.bashrc
  ```

- Add the following to the end of the file

  - Verify the path correctly points to Java

  ```
  # Java JDK
  export JAVA_HOME=~/installs/java/jdk-21.0.9+10
  export PATH=$PATH:~/installs/java/jdk-21.0.9+10/bin
  ```

- Source the .bashrc file and verify Java command works

  ```bash
  source ~/.bashrc
  java --version
  ```

### Setup NeoForge

The following steps will be for a NeoForge server (specifically for 21.1.216). If using a different mod lodder verify the setup steps specifically when updating the run script

- In the beamc account, create the directory where all the Minecraft specific files should be

  ```bash
  mkdir -p ~/minecraft/installs/mods
  mkdir -p ~/minecraft/installs/neoforge
  mkdir -p ~/minecraft/tabitv2
  ```

- Navigate to the `~/minecraft/installs/neoforge` directory and download the NeoForge installer

  ```bash
  mkdir -p ~/minecraft/installs/neoforge
  cd ~/minecraft/installs/neoforge
  wget https://maven.neoforged.net/releases/net/neoforged/neoforge/21.1.216/neoforge-21.1.216-installer.jar
  ```

- Copy the installer to the `~/minecraft/tabitv2` directory and run the installer

  ```bash
  cp neoforge-21.1.216-installer.jar ~/minecraft/tabitv2/neoforge-21.1.216-installer.jar
  cd ~/minecraft/tabitv2
  java -jar neoforge-21.1.216-installer.jar --installServer
  ```

- Update the run.sh to add the following line above the Java command

  ```
  export JAVA_HOME="/home/beamc/installs/java/jdk-21.0.9+10"
  ```

- Update the run.sh to change the "java" command to provide the full path to the Java file

  ```
  "/home/beamc/installs/java/jdk-21.0.9+10/bin/java"
  ```

- Update the user_jvm_args.txt file to add the following line

  - You may need to adjust depending on the server spec or Minecraft/mod loader versions used

  ```
  -Xms10G -Xmx10G -XX:+UseG1GC -XX:+ParallelRefProcEnabled -XX:MaxGCPauseMillis=200 -XX:+UnlockExperimentalVMOptions -XX:+DisableExplicitGC -XX:+AlwaysPreTouch -XX:G1NewSizePercent=30 -XX:G1MaxNewSizePercent=40 -XX:G1HeapRegionSize=8M -XX:G1ReservePercent=20 -XX:G1HeapWastePercent=5 -XX:G1MixedGCCountTarget=4 -XX:InitiatingHeapOccupancyPercent=15 -XX:G1MixedGCLiveThresholdPercent=90 -XX:G1RSetUpdatingPauseTimePercent=5 -XX:SurvivorRatio=32 -XX:+PerfDisableSharedMem -XX:MaxTenuringThreshold=1 -Dusing.aikars.flags=https://mcflags.emc.gs -Daikars.new.flags=true
  ```

- Run the run.sh script

  ```bash
  bash run.sh
  ```

- Open the eula.txt file and change `eula=false` to `eula=true`

  ```bash
  vi eula.txt
  ```

- Open the `server.properties` file and adjust any properties that need to be updated

  ```bash
  vi server.properties
  ```

- Depending on the NeoForge server the `server.properties` file may have different fields, but the following can be used as a guide

  ```
  #Minecraft server properties
  #Thu Dec 18 05:44:45 UTC 2025
  accepts-transfers=false
  allow-flight=false
  allow-nether=true
  broadcast-console-to-ops=true
  broadcast-rcon-to-ops=true
  bug-report-link=
  difficulty=normal
  enable-command-block=true
  enable-jmx-monitoring=false
  enable-query=false
  enable-rcon=false
  enable-status=true
  enforce-secure-profile=true
  enforce-whitelist=true
  entity-broadcast-range-percentage=100
  force-gamemode=true
  function-permission-level=2
  gamemode=survival
  generate-structures=true
  generator-settings={}
  hardcore=false
  hide-online-players=false
  initial-disabled-packs=
  initial-enabled-packs=vanilla
  level-name=tabitv2
  level-seed=-8021755361276700313
  level-type=minecraft\:normal
  log-ips=true
  max-chained-neighbor-updates=1000000
  max-players=10
  max-tick-time=60000
  max-world-size=29999984
  motd=<3
  network-compression-threshold=256
  online-mode=true
  op-permission-level=4
  player-idle-timeout=30
  prevent-proxy-connections=false
  pvp=true
  query.port=25565
  rate-limit=0
  rcon.password=
  rcon.port=25575
  region-file-compression=deflate
  require-resource-pack=false
  resource-pack=
  resource-pack-id=
  resource-pack-prompt=
  resource-pack-sha1=
  server-ip=
  server-port=25565
  simulation-distance=8
  spawn-animals=true
  spawn-monsters=true
  spawn-npcs=true
  spawn-protection=0
  sync-chunk-writes=true
  text-filtering-config=
  use-native-transport=true
  view-distance=8
  white-list=true
  ```

- EXTERNAL STEP

  - Download all the mods we're using locally and tar.gz the directory if it's not already

  - Copy the local files to the `minecraft` installs files

  ```bash
  scp -c aes128-ctr ./mods.tar.gz linodemc-beamc:/home/beamc/minecraft/installs/mods/mods.tar.gz
  ```

- Navigate to the `~/minecraft/installs/mods` directory, copy the `mods.tar.gz` file to the `~/minecraft/tabitv2/mods`, untar file and cleanup

  ```bash
  cd ~/minecraft/installs/mods
  cp mods.tar.gz ~/minecraft/tabitv2/mods
  cd ~/minecraft/tabitv2/mods
  tar -xvzf mods.tar.gz
  rm mods.tar.gz
  ```

- Run the run.sh script and verify the server starts

  - You should be able to connect to the server through Minecraft at this point

  ```bash
  bash run.sh
  ```

### Setup service

- In the beamc account, create a service file, change `tabitv2` to something cute but short related to the minecraft server

  ```bash
  vi ~/.config/systemd/user/tabitv2.service
  ```

- Paste the following into the file

  ```
  [Unit]
  After=network.target
  Description=TabitV2

  [Service]
  ExecStart=%h/minecraft/tabitv2/run.sh
  Restart=always
  WorkingDirectory=%h/minecraft/tabitv2/

  [Install]
  WantedBy=default.target
  ```

- Reload the systemctl daemon, enable the service and start

  ```bash
  systemctl --user daemon-reload
  systemctl --user enable tabitv2.service
  systemctl --user start tabitv2.service
  ```

- Verify status

  ```bash
  systemctl --user status tabitv2.service
  ```

### Setup Cloudflare

- In Cloudflare, select the domain for the Minecraft server

- Navigate to DNS -> Records

- Add a new record with the following

  - `Type`: A

  - `Name (required)`: `minecraft.tabitv2.com`

  - `IPv4 address (required)`: Server IP address

  - `Proxy status`: Disabled

- Save

## Extra Setup

Extra setup step not needs to get the Minecraft server running but would be nice to setup

### Install and Configuring Node Exporter

- In the `beaprom` account, create and navigate to the `~/installs/node_exporter` directory and download Node Exporter

  - The download link is for a specific version, if a newer version is available use the newer version

  ```bash
  mkdir -p ~/installs/node_exporter
  cd ~/installs/node_exporter
  wget https://github.com/prometheus/node_exporter/releases/download/v1.7.0/node_exporter-1.7.0.linux-amd64.tar.gz
  tar xvfz node_exporter-1.7.0.linux-amd64.tar.gz
  ```

- Create a `~/.config/systemd/user/node_exporter.service` file and paste the following content

  - Verify the path and ports are correct and desired

  ```
  [Unit]
  Description=Node Exporter (TabitV2)
  After=network.target

  [Service]
  ExecStart=%h/installs/node_exporter/node_exporter-1.10.2.linux-amd64/node_exporter --web.listen-address=":55500"
  Restart=always

  [Install]
  WantedBy=default.target
  ```

- Start and enable the service and check the status

  ```bash
  systemctl --user daemon-reload
  systemctl --user enable node_exporter.service
  systemctl --user start node_exporter.service
  systemctl --user status node_exporter.service
  ```

- Verify ability to request endpoint

  ```bash
  curl http://localhost:55500/metrics
  ```

### Install and Configuring minecraft-prometheus-exporter

- In the beamc account, navigate to the `~/minecraft/installs/prometheus_exporter` directory, download the minecraft-prometheus-exporter file, and copy to the `~/minecraft/tabitv2/mods` directory

  - The download link is for a specific version of the file. If there's a newer version use the latest verion

  ```bash
  cd ~/minecraft/installs/prometheus_exporter
  wget https://github.com/cpburnz/minecraft-prometheus-exporter/releases/download/1.21.8-neoforge-1.2.1/Prometheus-Exporter-1.21.8-neoforge-1.2.1.jar
  cp Prometheus-Exporter-1.21.8-neoforge-1.2.1.jar ~/minecraft/tabitv2/mods/Prometheus-Exporter-1.21.8-neoforge-1.2.1.jar
  ```

- Restart the Minecraft server

  ```bash
  systemctl --user restart tabitv2.service
  ```

- Verify ability to request endpoint

  ```bash
  curl http://localhost:55501/metrics
  ```

### Install and Configuring Prometheus

- In the beaprom account, create and navigate to the `~/installs/prometheus` directory and download the Prometheus file

  - The download link is for a specific version, if there is a newer version use the latest version

  ```bash
  cd ~/installs
  wget https://github.com/prometheus/prometheus/releases/download/v2.50.1/prometheus-3.8.1.linux-amd64.tar.gz
  tar xvfz prometheus-2.50.1.linux-amd64.tar.gz
  ```

- Replace the contents of the `~/installs/prometheus/prometheus-3.8.1.linux-amd64/prometheus.yml` file with the following

  ```
  global:
    scrape_interval: 15s

  scrape_configs:
    - job_name: 'node_exporter'
      static_configs:
        - targets: ['localhost:55500']
          labels:
            instance: 'tabitv2'
    - job_name: 'minecraft_server'
      static_configs:
        - targets: ['localhost:55501']
          labels:
            instance: 'tabitv2'
  ```

- Create a `~/.config/systemd/user/prometheus.service` file with the following content

  ```
  [Unit]
  After=network-online.target
  Description=Prometheus (TabitV2)

  [Service]
  ExecStart=%h/installs/prometheus/prometheus-3.8.1.linux-amd64/prometheus \
      --config.file=%h/installs/prometheus/prometheus-3.8.1.linux-amd64/prometheus.yml \
      --storage.tsdb.path=%h/installs/prometheus/prometheus_data \
      --web.console.templates=%h/installs/prometheus/prometheus-3.8.1.linux-amd64/consoles \
      --web.console.libraries=%h/installs/prometheus/prometheus-3.8.1.linux-amd64/console_libraries \
      --web.listen-address=:55502
  Restart=always

  [Install]
  WantedBy=default.target
  ```

- Start and enable the service and check status

  ```bash
  systemctl --user daemon-reload
  systemctl --user enable prometheus.service
  systemctl --user start prometheus.service
  systemctl --user status prometheus.service
  ```

- Verify ability to request UI

  ```bash
  curl http://localhost:55502
  ```

### Installing and Configuring Grafana

- In the beamcs account, install packages and configure keyring

  ```bash
  sudo apt install -y apt-transport-https software-properties-common wget
  sudo mkdir -p /etc/apt/keyrings/
  wget -q -O - https://apt.grafana.com/gpg.key | gpg --dearmor | sudo tee /etc/apt/keyrings/grafana.gpg > /dev/null
  echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | sudo tee /etc/apt/sources.list.d grafana.list
  sudo apt update
  sudo apt install grafana
  ```

- Start and enable the service and check status

  ```bash
  sudo systemctl daemon-reload
  sudo systemctl enable grafana-server
  sudo systemctl start grafana-server
  sudo systemctl status grafana-server
  ```

- Open to the `/etc/grafana/grafana.ini` file, uncomment `http_port` line (if commented) and set **55503** as the value

  ```bash
  sudo vi /etc/grafana/grafana.ini
  ```

- In the `/etc/grafana/grafana.ini` file set the following blocks

  ```
  [auth.anonymous]
  enabled = false

  [users]
  allow_sign_up = false
  ```

- Restart the service and check status

  ```bash
  sudo systemctl restart grafana-server
  sudo systemctl status grafana-server
  ```

- Verify ability to request UI

  ```bash
  curl http://localhost:55503
  ```

### Setup Cloudflare

- In Cloudflare, select the domain for the Minecraft server

- Navigate to DNS -> Records

- Add a new record with the following

  - `Type`: A

  - `Name (required)`: `grafana.tabitv2.com`

  - `IPv4 address (required)`: Server IP address

  - `Proxy status`: Enabled

- Save

- Navigate to SSL/TLS -> Overview

- Set SSL/TLS encryption to `Full (strict)`

- Navigate to Security -> Settings

  - Turn on `Bot fight mode`

  - Update `Manage your robots.txt` configuration to restrict the bot policy

### Installing and Configuring nginx With Certbot (Let's Encrypt)

- In the `beamcs` account, install nginx

  ```bash
  sudo apt install nginx
  ```

- Create a `/etc/nginx/sites-available/grafana.tabitv2.com` file (change domain to the domain you own) and paste the following

  ```bash
  server {
      listen 80;
      listen [::]:80;

      server_name grafana.tabitv2.com;

      location / {
          proxy_pass http://localhost:55503;
          proxy_set_header Host $host;
          proxy_set_header X-Real-IP $remote_addr;
          proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
          proxy_set_header X-Forwarded-Proto $scheme;
      }
  }
  ```

- Create the symlink

  ```bash
  sudo ln -s /etc/nginx/sites-available/grafana.tabitv2.com /etc/nginx/sites-enabled/
  ```

- Test nginx configuration

  ```bash
  sudo nginx -t
  ```

- Restart nginx

  ```bash
  sudo systemctl restart nginx
  ```

- Install Certbot and nginx plugin

  ```bash
  sudo apt install certbot python3-certbot-nginx
  ```

- Run Certbot to get and install the certificate

  ```bash
  sudo certbot --nginx -d grafana.tabitv2.com
  ```

- Verify service status

  ```bash
  sudo systemctl status certbot.timer
  ```

- Open the `/etc/grafana/grafana.ini` file, uncomment the `root_url` field (if commented) and set the value to be `https://grafana.tabitv2.com`

  ```bash
  sudo vi /etc/grafana/grafana.ini
  ```

- Restart grafana

  ```bash
  sudo systemctl restart grafana-server
  ```

- Verify ability to view grafana UI through `https://grafana.tabitv2.com`

- Change password immediately

### Setting Up Grafana Dashboards

- Log into Grafana

- Navigate to Connections -> Data sources -> `+ Add new data source`

- Add prometheus and update the configuration for the prometheus server

- Click "Save & Test" and verify it says "Data source is working".

- Navigate to Dashboards -> `New` -> `Import`

- Import `Node Exporter Full` dashboard via ID `1860` and `Load`

- Navigate to Dashboards -> `New` -> `Import`

- Import `Node Exporter Full` dashboard via ID `16508` and `Load`

### Setup fail2ban

- In the `beamcs` account run the following

  ```bash
  sudo apt install fail2ban
  sudo systemctl enable fail2ban
  sudo systemctl start fail2ban
  ```

- Open the `/etc/fail2ban/jail.local` file and paste the following

  ```
  [DEFAULT]
  # Ban hosts for 1 hour
  bantime = 1h

  # A host is banned if it makes maxretry attempts in findtime seconds.
  findtime = 10m

  # Number of attempts before a ban
  maxretry = 5

  # Your server's public IP addresses or local IP ranges that should never be banned
  # Space-separated values.
  ignoreip = 127.0.0.1/8 ::1

  # Email address to send notifications
  # destemail = your_email@example.com

  # Action to take (e.g., ufw, iptables)
  # 'ufw' is good for Ubuntu systems using UFW
  banaction = ufw

  [nginx-bad-request]
  enabled = true

  port = http,https

  filter = nginx-bad-request

  logpath = /var/log/nginx/access.log

  backend = auto

  maxretry = 6

  findtime = 600

  bantime = 3600

  [nginx-limit-req]
  enabled = true

  port = http,https

  filter = nginx-limit-req

  logpath = /var/log/nginx/error.log

  maxretry = 2

  findtime = 60

  bantime = 3600

  [sshd]
  enabled = true

  port = 1738

  logpath = /var/log/auth.log

  backend = system
  ```

- Restart fail2ban and verify status

  ```bash
  sudo systemctl restart fail2ban
  sudo systemctl status fail2ban
  sudo fail2ban-client status
  sudo fail2ban-client status nginx-bad-request
  sudo fail2ban-client status nginx-limit-req
  sudo fail2ban-client status sshd
  ```

### Further Setup nginx

- Open the `/etc/nginx/nginx.conf` file and set the content in the `http` block

  ```
  server_tokens off;
  client_max_body_size 10M;
  client_body_buffer_size 128k;
  limit_req_zone $binary_remote_addr zone=ratelimit:10m rate=10r/s;
  ```

- Open the `/etc/nginx/sites-available/grafana.tabitv2.com` file and set the content in the `location` block

  ```
  limit_req zone=ratelimit burst=5 nodelay;
  ```

## Useful Commands

Ussful commands for the ease of copying/pasting

### SSH specifying the host in the config

- beamc

  ```bash
  ssh linodemc-beamc
  ```

- beamcs

  ```bash
  ssh linodemc-beamcs
  ```

- beaprom

  ```bash
  ssh linodemc-beaprom
  ```

### Create a backup tar.gz

```bash
tar --use-compress-program="pigz -p 8" -cf <destination-file>.tar.gz <source-directory>
```

### Untar a backup tar.gz

```bash
tar --use-compress-program="pigz -p 8" -xvf <source-file>.tar.gz
```

### Copying files with a fast cipher and custom port

- Copy from remote server to local machine

  ```bash
  scp -c aes128-ctr linodemc-<user>:/file ./file
  ```

- Copy from local machine to remote server

  ```bash
  scp -c aes128-ctr ./file linodemc-<user>:/file
  ```

### Starting the service

```bash
systemctl --user start tabitv2.service
```

### Stopping the service

```bash
systemctl --user stop tabitv2.service
```

### Checking service status

```bash
systemctl --user status tabitv2.service
```
