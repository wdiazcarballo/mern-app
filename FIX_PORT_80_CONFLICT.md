# Fix Port 80 Conflict on EC2

## Problem
Docker cannot start nginx-proxy container because port 80 is already in use on the EC2 instance.

Error message:
```
Error response from daemon: failed to set up container networking:
failed to bind host port for 0.0.0.0:80:172.18.0.5:80/tcp: address already in use
```

## Solution Steps

Run these commands **on your EC2 instance** (SSH into the server first):

### Step 1: Check what's using port 80

```bash
sudo lsof -i :80
```

Alternative command if above doesn't work:
```bash
sudo ss -tulpn | grep :80
```

This will show you which process/service is using port 80 (commonly Apache or Nginx).

### Step 2: Stop the conflicting service

**If it's Apache:**
```bash
sudo systemctl stop apache2
sudo systemctl disable apache2
sudo systemctl status apache2
```

**If it's Nginx:**
```bash
sudo systemctl stop nginx
sudo systemctl disable nginx
sudo systemctl status nginx
```

The `disable` command prevents the service from starting automatically on server reboot.

### Step 3: Verify port 80 is now free

```bash
sudo lsof -i :80
```

This should return **nothing** (empty output), meaning port 80 is now available.

### Step 4: Stop any running Docker containers

```bash
cd ~/mern-app
docker compose down
```

### Step 5: Start Docker Compose

```bash
docker compose up -d
```

The `-d` flag runs containers in detached mode (background).

### Step 6: Verify all containers are running

```bash
docker compose ps
```

Expected output - all containers should show "Up" status:
```
NAME            SERVICE    STATUS    PORTS
mongodb         mongo      Up        27017/tcp
backend-api     backend    Up        5000/tcp
frontend-app    frontend   Up        80/tcp
nginx-proxy     nginx      Up        0.0.0.0:80->80/tcp
```

### Step 7: Check logs

```bash
docker compose logs -f
```

Press `Ctrl+C` to exit logs.

### Step 8: Test the application

```bash
curl http://localhost/api/health
```

Expected response:
```json
{
  "status": "OK",
  "message": "Backend is running!",
  "timestamp": "..."
}
```

Access from browser: `http://<EC2-PUBLIC-IP>`

## Alternative Solution: Use Different Port

If you need to keep the existing web server running on port 80, you can use a different port for your MERN app.

Edit `docker-compose.yml` line 54, change:
```yaml
ports:
  - "8080:80"  # Changed from "80:80"
```

Then:
```bash
docker compose down
docker compose up -d
```

Access application at: `http://<EC2-PUBLIC-IP>:8080`

**Note:** You must add port 8080 to your EC2 Security Group inbound rules.

## Troubleshooting

### If containers still won't start

Check individual container logs:
```bash
docker compose logs mongodb
docker compose logs backend
docker compose logs frontend
docker compose logs nginx
```

### If MongoDB is not healthy

```bash
docker exec -it mongodb mongosh
db.runCommand({ping: 1})
```

### Reset everything

```bash
docker compose down -v  # Remove volumes (deletes database data)
docker compose up -d
```

## Security Group Configuration

Ensure your EC2 Security Group allows inbound traffic:
- Type: HTTP
- Port: 80 (or 8080 if using alternative port)
- Source: 0.0.0.0/0 (for public access)
