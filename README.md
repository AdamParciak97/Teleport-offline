# Teleport-offline

### Architecture:
* ORACLE LINUX 8

### Download teleport docker image on docker desktop
```bash
docker pull public.ecr.aws/gravitational/teleport-distroless:18.2.2
docker save -o teleport-18.2.2.tar public.ecr.aws/gravitational/teleport-distroless:18.2.2
```
### Install docker rpm from and start docker
```bash
dnf install -y ./*rpm --allowerasing
systemctl enable --now docker
```

### Load docker image
```bash
docker load -i teleport-18.2.2.tar
```

### Create directory
```bash
mkdir -p ~/teleport/{config,data,certs,desktop-config,desktop-data}
```

### Create RootCA and server certificate
```bash
openssl genrsa -out rootCA.key 4096
openssl req -x509 -new -nodes -key rootCA.key -sha256 -days 3650   -out rootCA.crt -subj "/CN=TeleportRootCA"
openssl genrsa -out teleport.key 4096
openssl req -new -key teleport.key -out teleport.csr -subj "/CN=192.168.10.187"
openssl x509 -req -in teleport.csr -CA rootCA.crt -CAkey rootCA.key -CA createserial -out teleport.crt -days 825 -sha256 -extfile <(printf "subjectAltName=DNS:192.168.10.187")

cat /root/teleport/certs/teleport.crt /root/teleport/certs/rootCA.crt   > /root/teleport/certs/teleport-fullchain.crt
sed -i 's#cert_file: /etc/teleport/certs/teleport.crt#cert_file: /etc/teleport/certs/teleport-fullchain.crt#'   /root/teleport/config/teleport.yaml
```
### Create docker-compose.yml file
```bash
vi /root/teleport/docker-compose.yml
```

```bash
services:
  teleport:
    image: public.ecr.aws/gravitational/teleport-distroless:18.2.2
    container_name: teleport
    hostname: teleport
    restart: unless-stopped
    environment:
      - SSL_CERT_FILE=/etc/teleport/certs/rootCA.crt
      - TELEPORT_ALLOW_NO_SECOND_FACTOR=true
    volumes:
      - /root/teleport/config:/etc/teleport
      - /root/teleport/data:/var/lib/teleport
      - /root/teleport/certs:/etc/teleport/certs
    ports:
      - "3080:3080"
      - "3023:3023"
      - "3025:3025"
      - "3024:3024"
```

### Create teleport.yaml file
```bash
vi /root/teleport/config/teleport.yaml
```

```bash
version: v3

teleport:
  data_dir: /var/lib/teleport
  log:
    output: stderr
    severity: INFO

auth_service:
  enabled: true
  cluster_name: teleport.lab.local
  listen_addr: 0.0.0.0:3025
  authentication:
    type: local
    second_factor: off    # <–– TURN OFF MFA

proxy_service:
  enabled: true
  web_listen_addr: 0.0.0.0:3080
  tunnel_listen_addr: 0.0.0.0:3024
  public_addr: ["teleport.lab.local:3080"]
  https_keypairs:
    - key_file: /etc/teleport/certs/teleport.key
      cert_file: /etc/teleport/certs/teleport-fullchain.crt

ssh_service:
  enabled: false
```

### Run docker
```bash
docker compose up -d
```

### Create admin account
```bash
docker exec teleport tctl users add admin3 --roles=editor,access,auditor
```
<img width="997" height="128" alt="image" src="https://github.com/user-attachments/assets/81ab56dd-c935-40b1-9cb2-9bd848032e8e" />

### Log in to generate link and set a password
<img width="897" height="588" alt="image" src="https://github.com/user-attachments/assets/c758e32b-eaf2-4efd-9256-09728a604b39" />
<img width="1892" height="83" alt="image" src="https://github.com/user-attachments/assets/f8f65c23-4bfa-42ad-99db-21f64306713c" />

### Troubleshooting
```bash
docker logs -f teleport
```
<img width="1892" height="83" alt="image" src="https://github.com/user-attachments/assets/09072dbe-a065-4be7-b6ca-3f9cfce015f0" />

### Generate token to connect server SSH to teleport
```bash
docker exec -it teleport tctl nodes add --roles=node --ttl=10000h
```

### Create admin access role acount
```bash
docker exec -i teleport sh -c 'cat >/etc/teleport/role-access-logins.yaml <<EOF
kind: role
version: v6
metadata:
  name: access
spec:
  allow:
     logins: ["root","super_admin"]
EOF
tctl create -f /etc/teleport/role-access-logins.yaml --force
```

### Download agent on server from portal
```bash
https://goteleport.com/download/
```

### For example, download and install od Debian
```bash
sudo tar xzf teleport-v18.2.3-linux-amd64-bin.tar.gz
cd teleport
sudo install -m 0755 teleport tctl tsh /usr/local/bin/
teleport version
sudo adduser --system --group --home /var/lib/teleport --shell /usr/sbin/nologin teleport
sudo mkdir -p /etc/teleport /var/lib/teleport /var/log/teleport
sudo chown -R teleport:teleport /etc/teleport /var/lib/teleport /var/log/teleport
```
### Create file teleport.yaml
```bash
vi /etc/teleport/teleport.yaml
```

```bash
version: v3

teleport:
  nodename: proxmox-mail-gateway-1
  data_dir: /var/lib/teleport
  log:
    output: stderr
    severity: INFO

  proxy_server: teleport.lab.local:3024     # <—  'tctl nodes add'
#   ca_pin: "sha256:eea13353bc39ac21f1943d8f907054d489b60668ea815554531d54e29fb04ce4"  # <— paste generated ca_pin
  auth_token: "22585e81ea418bbdb4b6a9945ac30a24" # <-- paste generated token

auth_service:
  enabled: false

proxy_service:
  enabled: false

ssh_service:
  enabled: true
  labels:
    env: mgmt
```

### Create service teleport
```bash
[Unit]
Description=Teleport Service
After=network-online.target
Wants=network-online.target

[Service]
User=root
Group=root
ExecStart=/usr/local/bin/teleport start --config=/etc/teleport/teleport.yaml
Restart=on-failure
RestartSec=3
LimitNOFILE=16384

[Install]
WantedBy=multi-user.target
```
### Start service and add to autostart
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now teleport
sudo systemctl status teleport -f
```
### Troubleshooting
### Add RootCA.crt to trust certificate
### Add to etc/hosts

<img width="331" height="65" alt="image" src="https://github.com/user-attachments/assets/4c5d5667-b2f0-45ac-a469-94bd5957552b" />

### Set timedate to this same on teleport
```bash
systemctl stop chronyd
timedatectl set-timezone Europe/Warsaw 
timedatectl set-time '2025-10-01 09:37:00'
date -u
```
<img width="956" height="878" alt="image" src="https://github.com/user-attachments/assets/2e5dc522-541f-495e-b239-abc7b66ca36a" />
