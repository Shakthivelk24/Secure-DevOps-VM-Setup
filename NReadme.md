# Nginx and Certbot Setup – Secure DevOps VM

Configure **Nginx as a reverse proxy** and **Certbot with Let's Encrypt** to enable HTTPS for applications deployed on an Azure Ubuntu Virtual Machine.

This setup uses a DuckDNS domain to expose the application securely over HTTPS.

## 🏗️ Architecture

```text
                  User / Browser
                         |
                         v
          https://shakthivelassistant.duckdns.org
                         |
                         v
                  Nginx (Port 443)
                         |
                  SSL Termination
                         |
              +----------+----------+
              |                     |
              v                     v
          Frontend               Backend
       localhost:3000         localhost:5000
              |                     |
              v                     v
          React App              Node.js API

                  Certbot
                     |
                     v
             Let's Encrypt SSL
```

### Technologies

| Component | Purpose |
|---|---|
| Nginx | Reverse proxy and web server |
| DuckDNS | Free dynamic DNS domain |
| Certbot | SSL certificate management |
| Let's Encrypt | Free SSL/TLS certificates |
| Azure VM | Application hosting |
| Ubuntu 24.04 LTS | Operating system |

## 1. Configure DuckDNS Domain

1. Visit [DuckDNS](https://www.duckdns.org/).
2. Sign in to your account.
3. Create a subdomain.

Example:

```text
shakthivelassistant.duckdns.org
```

4. Configure the domain to point to your Azure VM's public IP address.
5. Verify that the domain resolves to the correct IP.

Run:

```bash
nslookup shakthivelassistant.duckdns.org
```

The returned IP address should match your Azure VM public IP.

**Important:** If your VM public IP changes, update the DuckDNS record before renewing or requesting a certificate.

## 2. Configure Azure Network Security Group

Open the following inbound ports in your Azure VM's Network Security Group:

| Port | Protocol | Purpose |
|---|---|---|
| 22 | TCP | SSH access |
| 80 | TCP | HTTP and Let's Encrypt validation |
| 443 | TCP | HTTPS |

Restrict SSH access to your trusted IP address. Ports 80 and 443 should be accessible to users if the application is publicly hosted.

Do not expose internal application ports such as 3000 and 5000 publicly when Nginx is acting as the reverse proxy.

## 3. Install Nginx

Update the Ubuntu packages:

```bash
sudo apt update
sudo apt upgrade -y
```

Install Nginx:

```bash
sudo apt install -y nginx
```

Enable and start the service:

```bash
sudo systemctl enable nginx
sudo systemctl start nginx
```

Check Nginx status:

```bash
sudo systemctl status nginx
```

Verify the installation:

```bash
nginx -v
```

Test Nginx using your domain:

```text
http://shakthivelassistant.duckdns.org
```

## 4. Configure Nginx Reverse Proxy

Create a new Nginx configuration file:

```bash
sudo nano /etc/nginx/sites-available/shakthivelassistant
```

Add the following configuration:

### Nginx Configuration

```nginx
server {
    listen 80;
    listen [::]:80;

    server_name shakthivelassistant.duckdns.org;

    client_max_body_size 50M;

    # Frontend - React Application
    location / {
        proxy_pass http://127.0.0.1:3000;

        proxy_http_version 1.1;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # Backend - Node.js API
    location /api/ {
        proxy_pass http://127.0.0.1:5000;

        proxy_http_version 1.1;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

**Note:** This example assumes the frontend listens on port `3000` and the backend listens on port `5000`. Change these ports to match your actual application configuration. The backend example preserves the `/api/` path.

Save the file:

- Press `CTRL + O`.
- Press `Enter`.
- Press `CTRL + X`.

## 5. Enable the Nginx Configuration

Create a symbolic link:

```bash
sudo ln -s /etc/nginx/sites-available/shakthivelassistant \
/etc/nginx/sites-enabled/
```

Remove the default Nginx site if it is not required:

```bash
sudo rm -f /etc/nginx/sites-enabled/default
```

Test the Nginx configuration:

```bash
sudo nginx -t
```

If the configuration test is successful, reload Nginx:

```bash
sudo systemctl reload nginx
```

Check Nginx status:

```bash
sudo systemctl status nginx
```

## 6. Configure Docker Application Ports

If your frontend and backend run in Docker containers, bind their ports to localhost so external users access them through Nginx.

Example Docker Compose configuration:

```yaml
services:
  frontend:
    ports:
      - "127.0.0.1:3000:80"

  backend:
    ports:
      - "127.0.0.1:5000:5000"
```

**Important:** Adjust the internal container ports to match your Docker images. For example, a backend listening on port 5000 inside the container should use `5000:5000`.

Restart the application:

```bash
sudo docker compose up -d
```

Check the containers:

```bash
sudo docker compose ps
```

Test the frontend and backend locally:

```bash
curl -I http://127.0.0.1:3000
curl -I http://127.0.0.1:5000
```

The backend root endpoint may return an application-specific response or 404; test the actual API health endpoint if available.

## 7. Install Certbot

Certbot automatically obtains and configures SSL certificates from Let's Encrypt.

Install Snap:

```bash
sudo apt update
sudo apt install -y snapd
```

Ensure Snap is up to date:

```bash
sudo snap install core
sudo snap refresh core
```

Remove an older apt-installed Certbot if present:

```bash
sudo apt remove -y certbot
```

Install Certbot:

```bash
sudo snap install --classic certbot
```

Create a symbolic link:

```bash
sudo ln -sf /snap/bin/certbot /usr/bin/certbot
```

Verify installation:

```bash
certbot --version
```

## 8. Generate SSL Certificate Using Certbot

Run the following command:

```bash
sudo certbot --nginx -d shakthivelassistant.duckdns.org
```

Certbot will:

1. Verify domain ownership through Let's Encrypt.
2. Generate an SSL certificate.
3. Configure the Nginx HTTPS server.
4. Configure HTTP-to-HTTPS redirection if selected.
5. Enable HTTPS for your domain.

Follow the interactive instructions and provide a valid email address.

After successful installation, access:

```text
https://shakthivelassistant.duckdns.org
```

Your application should now be accessible over HTTPS.

## 9. Verify SSL Certificate

List the installed certificates:

```bash
sudo certbot certificates
```

Check the certificate details:

```bash
sudo openssl x509 \
-in /etc/letsencrypt/live/shakthivelassistant.duckdns.org/fullchain.pem \
-noout -dates -issuer -subject
```

Test HTTPS:

```bash
curl -I https://shakthivelassistant.duckdns.org
```

Test HTTP redirection:

```bash
curl -I http://shakthivelassistant.duckdns.org
```

If HTTP-to-HTTPS redirection was configured, the HTTP response should return a redirect to HTTPS.

## 10. Configure Automatic SSL Certificate Renewal

Let's Encrypt certificates are valid for 90 days. Certbot installs an automatic renewal mechanism when installed through Snap.

Check the renewal timer:

```bash
sudo systemctl status snap.certbot.renew.timer
```

Test certificate renewal:

```bash
sudo certbot renew --dry-run
```

If the dry run succeeds, automatic renewal should be working.

Check available Certbot timers:

```bash
systemctl list-timers | grep certbot
```

## 11. Useful Nginx Commands

| Operation | Command |
|---|---|
| Check Nginx version | `nginx -v` |
| Start Nginx | `sudo systemctl start nginx` |
| Stop Nginx | `sudo systemctl stop nginx` |
| Restart Nginx | `sudo systemctl restart nginx` |
| Reload Nginx | `sudo systemctl reload nginx` |
| Check status | `sudo systemctl status nginx` |
| Test configuration | `sudo nginx -t` |
| View access logs | `sudo tail -f /var/log/nginx/access.log` |
| View error logs | `sudo tail -f /var/log/nginx/error.log` |

## 12. Useful Certbot Commands

| Operation | Command |
|---|---|
| Check version | `certbot --version` |
| Request certificate | `sudo certbot --nginx -d DOMAIN` |
| List certificates | `sudo certbot certificates` |
| Renew certificates | `sudo certbot renew` |
| Test renewal | `sudo certbot renew --dry-run` |
| Delete certificate | `sudo certbot delete` |

Replace `DOMAIN` with your actual domain name.

## 13. Troubleshooting

| Issue | Possible Cause | Solution |
|---|---|---|
| Domain not resolving | Incorrect DuckDNS IP | Update DuckDNS record |
| Certbot validation failed | Port 80 blocked | Check Azure NSG and firewall |
| Nginx configuration failed | Invalid syntax | Run `sudo nginx -t` |
| 502 Bad Gateway | Backend or frontend unavailable | Check application ports and Docker logs |
| HTTPS not working | Certificate or Nginx configuration issue | Check Certbot certificates and Nginx logs |
| Certificate renewal failed | DNS or port 80 issue | Check domain resolution and inbound rules |

### Check Nginx Logs

```bash
sudo tail -f /var/log/nginx/error.log
```

### Check Docker Containers

```bash
sudo docker ps -a
```

### Check Application Logs

```bash
sudo docker compose logs -f
```

### Check Listening Ports

```bash
sudo ss -tulpn
```

## 14. Security Best Practices

- Use HTTPS for all public application traffic.
- Keep ports 3000 and 5000 private behind Nginx.
- Restrict SSH access using Azure Network Security Groups.
- Keep Nginx, Certbot, and Ubuntu updated.
- Never expose private keys or DuckDNS tokens in GitHub.
- Use secure HTTP headers and appropriate application security policies.
- Configure application trust-proxy settings correctly when running behind Nginx.
- Test SSL renewal periodically.
- Use HTTPS-only secure cookies for authentication in production.

<!-- ## 🎯 Final Result

After completing the configuration:

| Service | URL |
|---|---|
| Application | `https://shakthivelassistant.duckdns.org` |
| HTTP | `http://shakthivelassistant.duckdns.org` |
| HTTPS | Enabled |
| SSL Provider | Let's Encrypt |
| Reverse Proxy | Nginx |
| Certificate Management | Certbot |
| Hosting | Azure Ubuntu VM | -->

## 👨‍💻 Author

**Shakthi Vel K**

Computer Science and Engineering

GitHub: [Your GitHub Profile](https://github.com/Shakthivelk24)

---

**Project:** Secure DevOps VM Setup

**Domain:** DevOps | DevSecOps | Cloud Computing | Nginx | SSL/TLS | HTTPS

**Cloud Platform:** Microsoft Azure

**Operating System:** Ubuntu 24.04 LTS