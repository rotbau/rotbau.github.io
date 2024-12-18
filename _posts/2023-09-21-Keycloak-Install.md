---
layout: archive
title: Deploying Keycloak on Linux for Pinniped and OIDC
---

Several of the VMware Tanzu products leverage Pinniped for OIDC authentication. Tanzu Kubernetes Grid, Tanzu Mission Control Self-Managed and Tanzu Application Platform to name a few can all leverage Pinniped for OIDC authentication.  Pinniped can work with a number of OIDC providers like Okta, Ping, Worksapce One and Keycloak.  For lab or isolated environments, Keycloak is an ideal solution.

This blog covers a quick and dirty install of Keycloak on Linux You can also install Keycloak on docker if you prefer.  Setup with be different but the client config for TKG and TMC should be the same.  Thanks to my collegue [Navneet](https://github.com/papivot/install-keycloak) for documenting the install steps listed below and helping with Keycloak Client configuration.

## Pre-Requisites

1. I used Ubuntu 22.04 because it supports openjdk-1.19-jdk which appears to be required by the lated 22.0.1 version of Keycloak.
2. Install openjdk and create keycloak group

```
sudo apt install openjdk-19-jdk
java --version
sudo groupadd keycloak
sudo useradd -r -g keycloak -d /opt/keycloak -s /sbin/nologin keycloak
```

## Download and Unpack Keycloak

```
sudo wget https://github.com/keycloak/keycloak/releases/download/22.0.1/keycloak-22.0.1.zip -O /opt/keycloak-22.0.1.zip
sudo unzip /opt/keycloak-22.0.1.zip -d /opt
sudo mv /opt/keycloak-22.0.1 /opt/keycloak
sudo chmod o+x /opt/keycloak/bin
sudo chown -R keycloak /opt/keycloak
```

## Install Keycloak

### Generate Certificate for Keycloak Server

1. If you have an established CA you can request a certificate for Keycloak server.  Be sure to include subject alternative for at least DNS but would recommend IP as well.
2. Alternatively if you want to just generate a quick self-signed CA Cert 

```
# Configure keycloak
sudo -u keycloak openssl req -newkey rsa:2048 -nodes -keyout /opt/keycloak/conf/key.pem -subj "/C=US/ST=ST/L=CityO=VMware/OU=Tanzu/CN=keycloak.example.com" -addext 'subjectAltName = DNS:keycloak.example.com, IP:10.197.107.63, IP:192.168.10.1' -x509 -days 365 -out /opt/keycloak/conf/certificate.pem
```

### Configure and Build Keycloak

1. Edit keycloak.conf

```
sudo -u keycloak cp /opt/keycloak/conf/keycloak.conf /opt/keycloak/conf/keycloak.conf.bak
sudo -u keycloak vi /opt/keycloak/conf/keycloak.conf
```

2. Modify keycloak.conf as needed.  For our use we are only editing the hostname variable to match what we used in the certificate (ex keycloak vtechk8s.com)

```
# Basic settings for running in production. Change accordingly before deploying the server.
# Database
# The database vendor.
#db=postgres
# The username of the database user.
#db-username=keycloak
# The password of the database user.
#db-password=password
# The full database JDBC URL. If not provided, a default URL is set based on the selected database vendor.
#db-url=jdbc:postgresql://localhost/keycloak
# Observability
# If the server should expose healthcheck endpoints.
#health-enabled=true
# If the server should expose metrics endpoints.
#metrics-enabled=true
# HTTP
# The file path to a server certificate or certificate chain in PEM format.
https-certificate-file=/opt/keycloak/conf/certificate.pem
# The file path to a private key in PEM format.
https-certificate-key-file=/opt/keycloak/conf/key.pem
# The proxy address forwarding mode if the server is behind a reverse proxy.
#proxy=reencrypt
# Do not attach route to cookies and rely on the session affinity capabilities from reverse proxy
#spi-sticky-session-encoder-infinispan-should-attach-route=false
# Hostname for the Keycloak server.
hostname=keycloak.vtechk8s.com
```

3. Build keycloak

```
sudo -u keycloak /opt/keycloak/bin/kc.sh build
sudo -u keycloak /opt/keycloak/bin/kc.sh show-config
```

### Set up Keycloak Service

1. Set up keycloak.service file

```
cat > /tmp/keycloak.service <<-EOF
[Unit]
Description=Keycloak Application Server
After=syslog.target network.target

[Service]
Type=idle
User=keycloak
Group=keycloak
RemainAfterExit=yes
LimitNOFILE=102642
ExecStart=/opt/keycloak/bin/kc.sh start --optimized
StandardOutput=null

[Install]
WantedBy=multi-user.target
EOF
```

2. Install keycloak.service

```
sudo mv /tmp/keycloak.service  /etc/systemd/system/keycloak.service 
sudo chown root:root /etc/systemd/system/keycloak.service
sudo systemctl daemon-reload
sudo systemctl enable keycloak
sudo systemctl start keycloak
sudo systemctl status keycloak
```

### Initial Login

Best way to set the initial admin user and password IMHO is to have a GUI available on the machine Keycloak was installed on.  Then you can just use a browser to visit http://localhost:8443 and set the credentials.

You can also set environmental variables for the admin user and password and then run the /opt/keycloak/bin/kc.bat start 

```
export KEYCLOAK_ADMIN=<username>
export KEYCLOAK_ADMIN_PASSWORD=<password>

bin/kc.[sh|bat] start
```

You are now ready to configure clients (applications) in Keycloak for your application.  I will have follow up blogs documenting setting up a client for Tanzu Mission Control Self-Managed and Tanzu Kubernetes Grid.

**Disclaimer:** All posts, contents and examples are for educational purposes only and does not constitute professional advice. No warranty and user excepts All information, contents, opinions are my own and do not reflect the opinions of my employer. Most likely you shouldn’t listen to what I’m saying and should close this browser window immediately
