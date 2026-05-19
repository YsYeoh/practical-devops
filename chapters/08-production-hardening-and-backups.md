# Chapter 8: Production Hardening & Backups

In this chapter, you will secure your server with HTTPS and a firewall, and set up a MongoDB backup and restore strategy.

---

## Part 1: Enable HTTPS with Certbot

### Why HTTPS Matters

| Without HTTPS | With HTTPS |
|---|---|
| Data sent in plain text | Data encrypted with TLS |
| Browser shows "Not Secure" | Browser shows padlock icon |
| No trust for users | Users see a verified connection |
| Some browser APIs blocked | All browser features available |

### Step 1.1 — Prerequisites

Before setting up HTTPS, you need:

- A domain name pointing to your server's IP address (e.g., `app.yourdomain.com`)
- DNS A record configured: `app.yourdomain.com → YOUR_SERVER_IP`
- Port 80 reachable (Certbot validates domain ownership via HTTP)

### Step 1.2 — Install Certbot and the Nginx Plugin

```bash
sudo apt install certbot python3-certbot-nginx -y
```

### Step 1.3 — Obtain and Install SSL Certificate

```bash
sudo certbot --nginx
```

**What this does:**

1. Certbot asks for your domain name
2. It connects to Let's Encrypt to validate domain ownership
3. It automatically modifies your Nginx configuration to use HTTPS
4. It sets up automatic certificate renewal

Certbot will ask:
- **Which domain(s)?** Enter your domain (e.g., `app.yourdomain.com`)
- **Redirect HTTP to HTTPS?** Choose **2** (redirect) — this ensures all traffic uses HTTPS

### Step 1.4 — Verify HTTPS

1. Visit `https://YOUR_DOMAIN` in your browser
2. You should see a padlock icon next to the URL
3. Test the redirect: visit `http://YOUR_DOMAIN` — it should redirect to `https://`

### Step 1.5 — Test Auto-Renewal

Let's Encrypt certificates expire after 90 days. Certbot installs a systemd timer that automatically renews them.

Test the renewal process:

```bash
sudo certbot renew --dry-run
```

If this succeeds, you do not need to worry about manual renewal. Certbot's timer runs twice daily and renews certificates that are within 30 days of expiry.

---

## Part 2: Set Up UFW Firewall

### Step 2.1 — Install UFW (if not already installed)

UFW (Uncomplicated Firewall) usually comes pre-installed on Ubuntu 22.04. If not:

```bash
sudo apt install ufw -y
```

### Step 2.2 — Set Default Policies

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
```

This blocks all incoming traffic by default and allows all outgoing traffic.

### Step 2.3 — Allow Essential Services

Allow SSH, HTTP, and HTTPS:

```bash
sudo ufw allow 22
sudo ufw allow 80
sudo ufw allow 443
```

If you still need Jenkins access for this training:

```bash
sudo ufw allow 8080
```

> In production, you would restrict Jenkins access to your office IP only:
> ```bash
> sudo ufw allow from YOUR_OFFICE_IP to any port 8080
> ```

### Step 2.4 — Enable the Firewall

```bash
sudo ufw enable
```

Confirm with `y` when prompted.

### Step 2.5 — Verify the Firewall

```bash
sudo ufw status verbose
```

Expected output:

```
Status: active
Logging: on (low)
Default: deny (incoming), allow (outgoing)
New profiles: skip

To                         Action      From
--                         ------      ----
22/tcp                     ALLOW IN    Anywhere
80/tcp                     ALLOW IN    Anywhere
443/tcp                    ALLOW IN    Anywhere
8080/tcp                   ALLOW IN    Anywhere
```

---

## Part 3: MongoDB Backup Strategy

Database backups are critical. Docker makes MongoDB backup and restore straightforward.

### Step 3.1 — Create a Backup Directory

```bash
mkdir -p ~/backups
```

### Step 3.2 — Create a Backup Script

Create a script to automate backups:

```bash
nano ~/backups/backup-mongodb.sh
```

Paste the following:

```bash
#!/bin/bash

# Configuration
BACKUP_DIR=~/backups
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="$BACKUP_DIR/mongodb_backup_$TIMESTAMP.gz"

# Run mongodump inside the container
echo "Starting MongoDB backup..."
docker exec mongodb mongodump \
  --username admin \
  --password password123 \
  --archive="$BACKUP_FILE" \
  --gzip

# Copy backup from container to host
docker cp mongodb:"$BACKUP_FILE" "$BACKUP_FILE"

# Remove the backup file from inside the container
docker exec mongodb rm "$BACKUP_FILE"

echo "Backup completed: $BACKUP_FILE"
```

Make the script executable:

```bash
chmod +x ~/backups/backup-mongodb.sh
```

### Step 3.3 — Run a Manual Backup

```bash
~/backups/backup-mongodb.sh
```

Verify the backup file exists:

```bash
ls -lh ~/backups/
```

### Step 3.4 — Create a Restore Script

```bash
nano ~/backups/restore-mongodb.sh
```

Paste:

```bash
#!/bin/bash

# Usage: ./restore-mongodb.sh <backup-file>

if [ -z "$1" ]; then
  echo "Usage: $0 <backup-file>"
  echo "Example: $0 ~/backups/mongodb_backup_20240101_120000.gz"
  exit 1
fi

BACKUP_FILE=$1

echo "Starting MongoDB restore from: $BACKUP_FILE"

# Copy backup file into the container
docker cp "$BACKUP_FILE" mongodb:/dump.gz

# Run mongorestore inside the container
docker exec mongodb mongorestore \
  --username admin \
  --password password123 \
  --archive=/dump.gz \
  --gzip

# Clean up the backup file from inside the container
docker exec mongodb rm /dump.gz

echo "Restore completed"
```

Make it executable:

```bash
chmod +x ~/backups/restore-mongodb.sh
```

### Step 3.5 — Test the Restore

```bash
# Find your latest backup
ls -t ~/backups/*.gz | head -1

# Restore from it
~/backups/restore-mongodb.sh ~/backups/mongodb_backup_20240101_120000.gz
```

### Step 3.6 — Automate Daily Backups (Optional)

Edit the crontab:

```bash
crontab -e
```

Add this line to back up daily at 2 AM:

```cron
0 2 * * * /home/yourusername/backups/backup-mongodb.sh >> /home/yourusername/backups/backup.log 2>&1
```

Replace `yourusername` with your actual username.

> **Important:** In production, you should also copy backups off-server (to S3, another server, or a backup service). A backup on the same server is not a real backup — if the server fails, the backup is lost too.

---

## Part 4: Update Nginx for HTTPS (If Not Using Certbot)

If you skipped the Certbot setup in Part 1, you can manually update your Nginx config. Otherwise, Certbot already handled this.

Manually, your `nginx/default.conf` should look like:

```nginx
server {
    listen 80;
    server_name yourdomain.com;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl;
    server_name yourdomain.com;

    ssl_certificate     /etc/letsencrypt/live/yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/yourdomain.com/privkey.pem;

    location / {
        proxy_pass http://nextjs:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}
```

---

## Verification Checklist

Before moving on, confirm:

- [x] HTTPS is working: `https://YOUR_DOMAIN` loads with a padlock
- [x] HTTP redirects to HTTPS automatically
- [x] UFW is active and shows the correct rules
- [x] SSH still works after enabling UFW
- [x] MongoDB backup script completes without errors
- [x] MongoDB restore script can restore from a backup
- [x] Backup file exists in `~/backups/`

---

## Troubleshooting

| Problem | Solution |
|---|---|
| `certbot: command not found` | Certbot was not installed. Run `sudo apt install certbot python3-certbot-nginx` |
| Certbot says "No domain found" | Your domain's DNS must point to your server's IP. Verify with `dig yourdomain.com` |
| UFW blocked my SSH connection | Use your cloud provider's console (VNC/Serial) to run `sudo ufw allow 22` |
| `docker exec mongodb mongodump` fails | Container name might differ. Check with `docker ps` and use the correct container name |
| `Connection refused` on port 443 | Nginx config might not have SSL. Run `sudo certbot --nginx` again, or check `nginx/default.conf` |

---

**Next:** [Chapter 9 — Full Ecosystem Migration](09-full-ecosystem-migration.md)
