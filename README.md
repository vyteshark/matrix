<div align="center">

# hi there! this is my Matrix [Synapse](https://github.com/matrix-org/synapse) and [nginx](https://github.com/nginx/nginx) configuration files!
**[Matrix](https://matrix.org) is an open, decentralized communication protocol for real-time messaging, VoIP, and IoT.**
</div>

- [getting ready](#getting-ready)
- [getting started](#getting-started)
- [install synapse](#installing-synapse)
- [NGINX + HTTPS](#nginx--https-configuration)
- [federation](#federation-configuration)
- [TURN server](#turn-server-for-voip-and-video-calls)
- [systemd service](#launching-synapse-as-a-systemd-service)
- [first user](#a-final-thing-enable-synapse-and-create-your-first-user)

## getting ready

what i used:
- distro: Debian 12 (bookworm)
- CPU: AMD Ryzen 9 3900 (10) @ 3.099GHz 
- RAM: 24GB
- SSD NVMe: 230GB
---
what you'll need (at least):
- 512MB/2GB RAM\* (depends on your distro! if you have <4GB, do **not** use heavy distros like Ubuntu. use lightweight ones, like Debian.)
- CPU: 1 vCPU
- storage: 2gb\*\*
- network: a public IP, a domain (get your own for free on duckdns.org!), open ports 8008 and 8448 <br>

\* i don't recommend using a server with just that specs actually... it will just crash (but ofc you can try! this is just my experience) <br>
\** with disabled attachments

\--- <br>

actual recommended specs (from my experience):
- distro: Debian 12/Ubuntu Server 22.04 LTS
- CPU: 2 vCPU
- RAM: 4GB (on Ubuntu) / 2GB (on Debian/Arch)
- storage: 20GB SSD
- network: a public IP, a domain (get your own for free on duckdns.org!), open ports 8008, 443, 20 and 8448

## getting started

1. install everything needed for what we're doin' right now:
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y python3-pip virtualenv build-essential libffi-dev python3-dev libssl-dev libjpeg-dev libxslt1-dev zlib1g-dev libpq-dev git curl gnupg apt-transport-https lsb-release
```

2. install and get into PostgreSQL (database):
```bash
sudo apt install -y postgresql
sudo -u postgres psql
```

3. in pSQL:
```sql
CREATE USER synapse WITH PASSWORD 'your-password';
CREATE DATABASE synapse ENCODING 'UTF8' LC_COLLATE='C' LC_CTYPE='C' template=template0 OWNER synapse;
\q
```

4. install Synapse:
```bash
sudo adduser --system --home /var/lib/synapse --group synapse
sudo -u synapse -i
```
- you can use another path for synapse's homedir, doesn't really matter, just don't mess up the permissions like i did in my first try :D

5. create your homedir and setup venv:
```bash
sudo mkdir -p /var/lib/synapse
sudo chown -R synapse:synapse /var/lib/synapse
cd /var/lib/synapse
virtualenv -p python3 pyenv
source pyenv/bin/activate
pip install --upgrade pip
pip install matrix-synapse[postgres]
```

## installing synapse

1. generate a config:
```bash
cd /var/lib/synapse
python -m synapse.app.homeserver
```
- and copy the generated keys from here, we'll be using them (or generate another ones with ```openssl rand -hex 32``` if ya want)

2. edit the config:
- clone my config from this repo: 
```bash
git clone https://codeberg.org/nightlyfoxx/matrix.git
cp /var/lib/synapse/matrix/homeserver.yaml /var/lib/synapse
```

- edit it: <br>
2.1  create the .env file - ```touch .env``` (we'll talk about how to apply it later, dw twin ;D) <br>
2.2  add the following lines (and edit them, ofc :P):
```
SYNAPSE_DB_PASSWORD="your-psql-db-password"
SYNAPSE_REG_SHARED_SECRET="your-secret-key"
SYNAPSE_MACAROON_SECRET_KEY="your-secret-key"
SYNAPSE_FORM_SECRET="your-secret-key"
SYNAPSE_TURN_SHARED_SECRET="your-turn-secret-key"
SYNAPSE_SIGNING_KEY="/var/lib/synapse/yourdomain.com.signing.key"
SYNAPSE_SRVNAME="yourdomain.com"
SYNAPSE_PID_FILE="/var/lib/synapse/homeserver.pid"
SYNAPSE_MEDIA_DIR="/var/lib/synapse/media_store"
SYNAPSE_LOGFILE="/var/log/synapse/homeserver.log.config"
PYTHONUNBUFFERED=1
```

2.3  edit homeserver.yaml:
- i left a lot of hints inside of it to make sure you'll figure everything out!
- if something is wrong in there or you don't understand something, feel free to [contact me](https://nightfox.neocities.org) ;D

## nginx + https configuration

1. install everything needed (yep, again):
```bash
sudo apt install -y nginx certbot python3-certbot-nginx
```

2. setup nginx:
```bash
sudo cp /var/lib/synapse/matrix/nginx/sites-enabled/default /etc/nginx/sites-enabled/default
sudo nano /etc/nginx/sites-enabled/default
sudo ufw allow 3478,5349/udp
sudo ufw allow 80,443/tcp
```
- adjust everything inside of this file! (domains, paths etc.)
- after you did:
```bash
sudo nginx -t # check if there is any errors in your config, let's hope there isn't :D
sudo systemctl restart nginx
sudo systemctl enable nginx
sudo systemctl status nginx
```
- if it works, continue configuring everything else in there!
- if not, try searching for the answer first, but if you still can't figure something out, read troubleshooting.md (currently WIP) or [contact me](https://nightfox.neocities.org)! :)

3. place something into ```/var/www/matrix/index.html```. it can be anything - an actual website, a stub, a... \<h1>hello world\</h1>?..

4. get an SSL cert:
```bash
sudo certbot --nginx -d yourdomain.com
```
- replace yourdomain.com with your actual domain!

## federation configuration
**federation is a feature, that allows users of another homeservers (e.g. matrix.org) connect to rooms from your homeserver!**
**skip this step if you have a potato server - it will take a lot of resources :P**

1. uncomment the following lines in ```/etc/nginx/sites-enabled/default```:
```
#    location /.well-known/matrix/server {
#        default_type application/json;
#        return 200 '{"m.server": "matrix-nextdragon.duckdns.org:443"}'; # replace with your domain here too!
#    }
#
#    location /.well-known/matrix/client {
#        default_type application/json;
#        return 200 '{"m.homeserver": {"base_url": "https://matrix-nextdragon.duckdns.org"}}'; # and here!
#    }
```

*...why did i dedicate a whole section to this...*

## TURN server (for VoIP and video calls)

**again, if you have a potato server - skip this step! :)**

1. install coturn:
```bash
sudo apt install coturn
```
2. configure it:
```bash
sudo nano /etc/turnserver.conf
```

example configuration:
```conf
listening-port=3478
tls-listening-port=5349
use-auth-secret
static-auth-secret=your-secret-key
realm=yourdomain.com
lt-cred-mech # required for Synapse to authenticate!
fingerprint
no-multicast-peers
no-cli
# if behind NAT or multiple IPs:
# listening-ip=<your-private-ip-here>
# external-ip=<your-public-ip-here>
```



3. enable coTURN:
```bash
sudo systemctl enable --now coturn
```


## launching Synapse as a systemd service

1. create a unit: ```sudo nano /etc/systemd/system/matrix-synapse.service```
2. install gettext (it's used to get things from the .env file, which is kinda important yea): ```sudo apt install -y gettext```
3. configure it:
```
# /etc/systemd/system/matrix-synapse.service
[Unit]
Description=Matrix Synapse Server
After=network.target

[Service]
Type=simple
User=synapse
WorkingDirectory=/var/lib/synapse
EnvironmentFile=/var/lib/synapse/.env
ExecStart=/bin/bash -c '/var/lib/synapse/pyenv/bin/python -m synapse.app.homeserver --config-path <(envsubst < /var/lib/synapse/homeserver.yaml)'
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

3. reload systemd daemon and synapse:
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now matrix-synapse
```
- reloading the systemd daemon is needed for it to apply changes in units files!

# done! now you have a fully working matrix server! :D

## a final thing, enable synapse and create your first user:

```bash
sudo -u synapse -i
source pyenv/bin/activate
register_new_matrix_user -c /var/lib/synapse/homeserver.yaml http://localhost:8008
```

```bash
synapse@kukuruzaparty:~$ register_new_matrix_user -c /var/lib/synapse/homeserver.yaml http://localhost:8008
New user localpart [synapse]: testuser   # username!
Password: 
Confirm password: 
Make admin [no]: yes    # if you want it to be an administrator!
```

**to create a user with admin perms, you'll have to register it through CLI!** <br>
**regular users can register through an app like element or nheko, if you enabled registrations in the config! (enabled by default in the one you cloned earlier)**

### i guess that's it! have a good day ;))

<br> <br>

<div align="center">
<small>built for debian by vshark with <3 and ~4 hours of work</small> <br>
<small>you can join <a href="https://m.vshark.xyz">my</a> server btw! use m.vshark.xyz in homeserver URL field</small>
</div>
