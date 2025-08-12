══════════════════════════════════════════════════════════════════════════════
 STEP 1 — INITIAL HOST INFO & IP IDENTIFICATION
══════════════════════════════════════════════════════════════════════════════
# Purpose: Know the host’s system info and IP for later testing.

# Show hostname (computer name)
hostname

# Show kernel + OS details
uname -a

# Show distro info (Ubuntu/CentOS/etc.)
cat /etc/os-release

# Show IPv4 addresses of host
ip -4 addr show

# Quick, clean list of IPs
hostname -I

# Optional: Public IP (Internet-facing)
curl -s https://ifconfig.co
══════════════════════════════════════════════════════════════════════════════


══════════════════════════════════════════════════════════════════════════════
 STEP 2 — CHECK PORT USAGE (5678)
══════════════════════════════════════════════════════════════════════════════
# Purpose: See if any process is already using port 5678.

# Method 1 — Using ss
sudo ss -tulnp | grep ':5678'

# Method 2 — Using lsof
sudo lsof -nP -iTCP:5678 -sTCP:LISTEN

# Method 3 — Using netstat (if installed)
sudo netstat -tulnp | grep 5678

# OUTPUT:
# tcp   LISTEN   0   128   0.0.0.0:5678   0.0.0.0:*   users:(("docker-proxy",pid=1234,fd=4))
# → Means port is in use by docker-proxy (PID 1234)
══════════════════════════════════════════════════════════════════════════════


══════════════════════════════════════════════════════════════════════════════
 STEP 3 — KILL EXISTING PROCESSES ON PORT 5678 (IF ANY)
══════════════════════════════════════════════════════════════════════════════
# Purpose: Free up port 5678 before running n8n.

# Find PID of process using port
sudo lsof -t -i:5678

# Show details of that PID
ps -p <PID> -o pid,user,cmd

# If it’s a Docker container, stop it:
docker ps --format "table {{.ID}}\t{{.Names}}\t{{.Ports}}"
docker stop <container_id_or_name>
docker rm <container_id_or_name>

# If it’s a system service:
sudo systemctl stop <service-name>

# If must kill manually:
sudo kill <PID>         # try graceful
sudo kill -15 <PID>     # force terminate
sudo kill -9 <PID>      # last resort
══════════════════════════════════════════════════════════════════════════════


══════════════════════════════════════════════════════════════════════════════
 STEP 4 — PREPARE HOST RESOURCES
══════════════════════════════════════════════════════════════════════════════
# Purpose: Ensure Docker, disk, RAM, firewall are ready.

# Check Docker installed
docker --version

# Start Docker service
sudo systemctl enable --now docker

# Disk space
df -h

# Memory
free -h

# CPU cores
nproc

# Firewall (Ubuntu ufw)
sudo ufw status
sudo ufw allow 5678/tcp
sudo ufw reload

# Firewall (CentOS/RHEL firewalld)
sudo firewall-cmd --permanent --add-port=5678/tcp
sudo firewall-cmd --reload

# Prepare volume for n8n data
mkdir -p ~/.n8n
sudo chown 1000:1000 ~/.n8n
══════════════════════════════════════════════════════════════════════════════


══════════════════════════════════════════════════════════════════════════════
 STEP 5 — START n8n DOCKER CONTAINER
══════════════════════════════════════════════════════════════════════════════
# Quick method with docker run:
docker run -d \
  --name n8n \
  --restart unless-stopped \
  -p 5678:5678 \
  -v ~/.n8n:/home/node/.n8n \
  -e N8N_BASIC_AUTH_ACTIVE=true \
  -e N8N_BASIC_AUTH_USER=admin \
  -e N8N_BASIC_AUTH_PASSWORD='StrongPassHere' \
  -e TZ='Asia/Kolkata' \
  n8nio/n8n:latest

# Recommended method with docker-compose:
nano docker-compose.yml
---
version: "3.8"
services:
  n8n:
    image: n8nio/n8n:latest
    container_name: n8n
    restart: unless-stopped
    ports:
      - "5678:5678"
    volumes:
      - ~/.n8n:/home/node/.n8n
    environment:
      - N8N_BASIC_AUTH_ACTIVE=true
      - N8N_BASIC_AUTH_USER=admin
      - N8N_BASIC_AUTH_PASSWORD=StrongPassHere
      - TZ=Asia/Kolkata
---
# Save & exit, then:
docker-compose up -d
══════════════════════════════════════════════════════════════════════════════


══════════════════════════════════════════════════════════════════════════════
 STEP 6 — VERIFY PORT BINDING
══════════════════════════════════════════════════════════════════════════════
# See container is running
docker ps --filter name=n8n

# See if host is listening on port 5678
sudo ss -tulnp | grep ':5678'

# Or using docker inspect
docker port n8n 5678
══════════════════════════════════════════════════════════════════════════════


══════════════════════════════════════════════════════════════════════════════
 STEP 7 — TEST n8n ACCESSIBILITY
══════════════════════════════════════════════════════════════════════════════
# From host machine
curl -I http://localhost:5678

# From remote machine (replace with server IP)
curl -I http://<server-ip>:5678

# In browser
http://<server-ip>:5678

# EXPECTED:
# - Browser shows n8n UI (with login prompt if basic auth enabled)
# - curl shows "HTTP/1.1 200 OK" or auth request
══════════════════════════════════════════════════════════════════════════════


══════════════════════════════════════════════════════════════════════════════
 STEP 8 — FINAL CLEANUP & CONFIRMATION
══════════════════════════════════════════════════════════════════════════════
# Stop container
docker stop n8n

# Remove container
docker rm n8n

# Remove unused images/volumes
docker system prune -f

# Check if port is free
sudo ss -tulnp | grep ':5678' || echo "Port 5678 free"
══════════════════════════════════════════════════════════════════════════════


══════════════════════════════════════════════════════════════════════════════
 STEP 9 — NOTES / SECURITY / DOUBTS
══════════════════════════════════════════════════════════════════════════════
- Always map volume (~/.n8n) for persistence.
- Use strong BASIC auth in production.
- Better: put n8n behind reverse proxy with HTTPS.
- For production DB: Use Postgres instead of SQLite.
- Backup both DB + ~/.n8n regularly.
- Keep Docker & n8n image updated: 
  docker pull n8nio/n8n && docker-compose up -d
══════════════════════════════════════════════════════════════════════════════
