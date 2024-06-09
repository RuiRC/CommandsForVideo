## Prerequisites:
- A Domain
- An Oracle Account (preferably Pay as You Go)

## Steps:

### 1 - Update the Host Machine
```sh
apt update -y && apt upgrade -y
```

### 2 - Install Basic Dependencies and Apps
```sh
apt install netcat-openbsd resolvconf wireguard curl sudo software-properties-common build-essential git -y
```

### 2.5 - (Optional/DO NOT DO) Enable Root SSH for Easy Configuration
```sh
nano /etc/ssh/sshd-config
```

### 3 - Enter Root in VPS
```sh
sudo su
```

### 4 - Update Machine and Install Nginx
```sh
apt update -y && apt upgrade -y && apt install nginx -y
```

### 5 - Download and Run the [Wireguard Install Script](https://github.com/angristan/wireguard-install)
```sh
curl -O https://raw.githubusercontent.com/angristan/wireguard-install/master/wireguard-install.sh
chmod +x wireguard-install.sh
./wireguard-install.sh
```

### 6 - Rename Your Client Config File to `wg0.config`
```sh
mv yourfilename.conf wg0.conf
```

### 7 - Edit the File and Add Keep Alive
Open `wg0.conf` and add:
```sh
nano wg0.conf

PersistentKeepalive = 25
```

### 8 - Edit Nginx Configuration File and Reload Nginx
```sh
nano /etc/nginx/nginx.conf
```
Add the following configuration:
```nginx
stream {
    # TCP traffic on port 25565
    server {
        listen 25565;
        proxy_pass 10.66.66.2:25565;
        proxy_timeout 3s;
    }
}
```
Then test and reload Nginx:
```sh
nginx -t
systemctl reload nginx
```

### 9 - Allow Ports through Firewall
```sh
ufw allow 22 && ufw allow 25565 && ufw allow 58000
ufw enable
```

### 10 - Create Service File for Auto-Connection to Wireguard VPN
```sh
nano /etc/systemd/system/wg-quick@.service
```
Add the following content:
```ini
[Unit]
Description=WireGuard via wg-quick(8) for %I
After=network.target

[Service]
Type=oneshot
ExecStart=/usr/bin/wg-quick up %i
ExecStartPost=/usr/bin/curl ifconfig.io
ExecStartPost=/usr/bin/nc -vz 10.66.66.1 25565
ExecStop=/usr/bin/wg-quick down %i
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```
Enable and start the service:
```sh
systemctl enable wg-quick@wg0.service
systemctl start wg-quick@wg0.service
systemctl status wg-quick@wg0.service
```

### 11 - Workaround to Keep Nginx Alive
Create a timer:
```sh
nano /etc/systemd/system/nc-timer.timer
```
Add the following content:
```ini
[Unit]
Description=Run nc every 1 minute

[Timer]
OnCalendar=*:0/1
Persistent=true

[Install]
WantedBy=timers.target
```
Create a service:
```sh
nano /etc/systemd/system/nc-timer.service
```
Add the following content:
```ini
[Unit]
Description=Run nc command on port 25565

[Service]
Type=simple
ExecStart=/usr/bin/nc -vz 10.66.66.1 25565
```
Enable the timer:
```sh
systemctl enable --now nc-timer.timer
```

### 12 - Make OpenSSL Certs for Pelican Panel
```sh
mkdir -p /etc/certs
openssl req -new -newkey rsa:4096 -days 3650 -nodes -x509 -subj "/C=NA/ST=NA/L=NA/O=NA/CN=Generic SSL Certificate" -keyout /etc/certs/privkey.pem -out /etc/certs/fullchain.pem
```

### 13.1 - Follow the Pelican Documentation! Anything from the Documentation should be taken from it in the correct order
Refer to the [Pelican Panel Documentation](https://pelican.dev/docs/panel/getting-started/).

### 13.2 - Dependencies for Pelican
```sh
apt -y install php8.3 php8.3-{common,cli,gd,mysql,mbstring,bcmath,xml,fpm,curl,zip,intl,sqlite3} mariadb-server nginx tar unzip git
```

### 13.3 - Join the Pelican Panel Discord for Help
Join the [Pelican Panel Discord](https://discord.gg/pelican-panel).

### 14 - Tunnel Docker Command Edit
```sh
docker run -d --restart unless-stopped [rest of cloudflare stuff]
```
