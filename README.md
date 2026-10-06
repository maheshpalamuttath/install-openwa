# Installing OpenWA with Docker and Apache Reverse Proxy

[OpenWA](https://github.com/rmyndharis/OpenWA) provides a self-hosted WhatsApp API and dashboard. In this guide, we will install OpenWA using Docker and expose it through Apache as a reverse proxy.

## 1. Install Docker

Update the system and install the required packages:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y uidmap curl
```

Install Docker:

```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
```

Add your user to the Docker group:

```bash
sudo usermod -aG docker $USER
```

Log out and log back in for the group change to take effect.

## 2. Download and Run OpenWA

Clone the OpenWA repository:

```bash
git clone https://github.com/rmyndharis/OpenWA.git
cd OpenWA
```

Start OpenWA:

```bash
docker compose -f docker-compose.dev.yml up -d
```

OpenWA's dashboard and API are available on port **2785**.

You can test it locally with:

```text
http://localhost:2785
```

Or from another computer:

```text
http://SERVER-IP:2785
```

## 3. Install Apache

Install Apache:

```bash
sudo apt install apache2 -y
```

Enable the required proxy modules:

```bash
sudo a2enmod proxy
sudo a2enmod proxy_http
sudo a2enmod proxy_wstunnel
sudo a2enmod rewrite
```

Restart Apache:

```bash
sudo systemctl restart apache2
```

## 4. Configure Apache Reverse Proxy

Create a new Apache configuration:

```bash
sudo nano /etc/apache2/sites-available/openwa-proxy.conf
```

Add:

```apache
<VirtualHost *:2786>

    ProxyPreserveHost On

    ProxyPass        / http://127.0.0.1:2785/
    ProxyPassReverse / http://127.0.0.1:2785/

    # WebSocket support
    RewriteEngine On
    RewriteCond %{HTTP:Upgrade} =websocket [NC]
    RewriteCond %{HTTP:Connection} upgrade [NC]
    RewriteRule /(.*) ws://127.0.0.1:2785/$1 [P,L]

</VirtualHost>
```

Here, OpenWA continues to run on port **2785**, while Apache listens on **2786** and forwards requests to OpenWA.

## 5. Configure Apache Port

Edit:

```bash
sudo nano /etc/apache2/ports.conf
```

Add:

```apache
Listen 2786
```

Enable the site:

```bash
sudo a2ensite openwa-proxy.conf
```

Check the Apache configuration:

```bash
sudo apache2ctl configtest
```

You should see:

```text
Syntax OK
```

Restart Apache:

```bash
sudo systemctl restart apache2
```

## 6. Access OpenWA

OpenWA can now be accessed through Apache:

```text
http://SERVER-IP:2786
```

For example:

```text
http://192.168.29.2:2786
```

The request flow is:

```text
Browser
   ↓
Apache :2786
   ↓
OpenWA :2785
   ↓
Docker Container
```

You can still access OpenWA directly on port 2785, but port 2786 is the Apache reverse-proxy endpoint.

## 7. Test the Services

Check the OpenWA container:

```bash
docker ps
```

Test OpenWA directly:

```bash
curl http://127.0.0.1:2785
```

Test the Apache proxy:

```bash
curl http://127.0.0.1:2786
```

Check Apache status:

```bash
sudo systemctl status apache2
```

## Using a Domain Name

If you have a domain and a reverse-proxy manager such as Nginx Proxy Manager, you can also expose OpenWA using a subdomain such as:

```text
https://openwa.example.com
```

In that case, the proxy can forward the domain directly to:

```text
http://SERVER-IP:2785
```
Get API Key

```text
docker exec openwa-api cat /app/data/.api-key
```

This avoids exposing OpenWA's port 2785 directly to the Internet and gives you a cleaner URL.

## Conclusion

Running OpenWA with Docker makes installation and maintenance simple, while Apache provides a convenient reverse-proxy layer.

The important ports in this setup are:

| Service | Port |
|---|---:|
| OpenWA | 2785 |
| Apache Reverse Proxy | 2786 |

For a local network, you can use:

```text
http://SERVER-IP:2786
```

For production use, I recommend putting OpenWA behind a domain with HTTPS and restricting direct access to the OpenWA port.
