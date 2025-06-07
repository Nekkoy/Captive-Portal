[![en](https://img.shields.io/badge/lang-en-red.svg)](README_EN.md)
[![ua](https://img.shields.io/badge/lang-ua-yellow.svg)](README.md)
[![ru](https://img.shields.io/badge/lang-ru-blue.svg)](README_RU.md)

# Captive Portal for Internet Service Providers

**A simple implementation of a Captive Portal**, whose purpose is to display a stub page to clients with limited internet access.

<img src="https://raw.githubusercontent.com/Nekkoy/Captive-Portal/main/img_en.jpg" width="560" height="380">

---

## 🔧 How does this work

1. When connecting to the network, the client device checks for Internet access via DNS requests to special domains (e.g., `www.msftconnecttest.com`, `captive.apple.com`, etc.).
2. We intercept these requests and respond to them from a local DNS server (Unbound), pointing to the IP Captive Portal.
3. The client makes an HTTP request to the received IP. If the server returns `HTTP 302 Found` with the required URL, the client browser automatically opens the page with this URL.

---

## 📦 Installation and configuration

### 1. Installing nginx and unbound

#### Debian/Ubuntu:
```bash
apt-get install nginx unbound
```

#### RHEL/CentOS:
```bash
yum install nginx unbound
```

### 2. Configuring nginx
#### a. Stub page
Copy the `nginx_captive_portal_page.conf` file to `/etc/nginx/conf.d/`:
```bash
cp -f ./nginx_captive_portal_page.conf /etc/nginx/conf.d/
```
Change the domain name `captiveportal.example.com` to your domain and install the SSL certificate.

Create a directory for statics:
```bash
mkdir -p /var/www/captive_portal
```

And put the HTML file there:
```bash
cp -f ./index.html /var/www/captive_portal/index.html
```

#### b. Redirect
Copy the `nginx_captive_portal_redirect.conf` file to the same folder:
```bash
cp -f ./nginx_captive_portal_redirect.conf /etc/nginx/conf.d/
```
Replace the domain `captiveportal.example.com` with yours in the line:
```bash
 set $portal_domain captiveportal.example.com;
```

Restart `nginx`:
```bash
systemctl restart nginx
```

### 3. Setting up unbound
Copy the configuration file:
```bash
cp -f ./unbound.conf /etc/unbound/
```

Restart `Unbound`:
```bash
systemctl restart unbound
```

## 🔐 Intercepting DNS requests via iptables
#### a. Redirecting DNS clients to local DNS (192.168.1.254) with address substitution for the gateway address (192.168.1.1)
```bash
iptables -t nat -A PREROUTING -s 192.168.1.0/24 -p udp --dport 53 -j DNAT --to-destination 192.168.1.254:53
iptables -t nat -A PREROUTING -s 192.168.1.0/24 -p tcp --dport 53 -j DNAT --to-destination 192.168.1.254:53
```
#### b. SNAT for outgoing DNS queries
```bash
iptables -t nat -A POSTROUTING -p udp -d 192.168.1.254 --dport 53 -j SNAT --to-source 192.168.1.1
iptables -t nat -A POSTROUTING -p tcp -d 192.168.1.254 --dport 53 -j SNAT --to-source 192.168.1.1
```

#### c. SNAT for responses from local DNS
```bash
iptables -t nat -A POSTROUTING -p udp -s 192.168.1.254 --sport 53 -j SNAT --to-source 192.168.1.1
iptables -t nat -A POSTROUTING -p tcp -s 192.168.1.254 --sport 53 -j SNAT --to-source 192.168.1.1
```

## 🔒 Client restrictions
For Captive Portal to work correctly, clients must be restricted as follows:

1. Allowed:
  - DNS (UDP/TCP 53)
  - HTTP (80) and HTTPS (443) access only to:
    - Captive Portal
    - Personal account
    - Payment systems and other "white" sites

2. Prohibited:
  - HTTP(S) access to other hosts on the Internet
