# Server Setup Reference

## First-time server setup

### Linux (Ubuntu/Debian)

```bash
# --- 1. Install Docker Engine ---
# Remove old versions
sudo apt-get remove -y docker docker-engine docker.io containerd runc 2>/dev/null

# Install prerequisites
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg

# Add Docker's official GPG key and repo
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Verify
docker --version
docker compose version

# --- 2. Create deploy user ---
sudo useradd -m -s /bin/bash -G docker deploy

# --- 3. Set up SSH key auth for deploy user ---
sudo mkdir -p /home/deploy/.ssh
sudo chmod 700 /home/deploy/.ssh
# Paste your CI/CD public key here:
echo "ssh-ed25519 AAAA... github-actions" | sudo tee /home/deploy/.ssh/authorized_keys
sudo chmod 600 /home/deploy/.ssh/authorized_keys
sudo chown -R deploy:deploy /home/deploy/.ssh

# --- 4. Create application directory ---
sudo mkdir -p /opt/app
sudo chown deploy:deploy /opt/app

# --- 5. Create .env file (fill in real values) ---
sudo -u deploy cp /opt/app/.env.example /opt/app/.env
# Edit with real values:
sudo -u deploy nano /opt/app/.env

# --- 6. Firewall ---
sudo ufw allow OpenSSH
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw --force enable
sudo ufw status

# --- 7. (Optional) Harden SSH ---
# Disable password auth — only key-based
sudo sed -i 's/#PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config
sudo sed -i 's/PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config
sudo systemctl restart sshd
```

### Windows Server

```powershell
# --- 1. Install Docker Engine (Docker Desktop or Docker EE) ---
# Option A: Docker Desktop (dev/small prod)
# Download from https://docs.docker.com/desktop/install/windows-install/

# Option B: Docker Engine via Windows feature
Enable-WindowsOptionalFeature -Online -FeatureName containers -All
Enable-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V -All
# Restart required, then install Docker CLI

# Verify
docker --version
docker compose version

# --- 2. Create application directory ---
New-Item -ItemType Directory -Force -Path C:\app

# --- 3. Configure firewall ---
New-NetFirewallRule -DisplayName "HTTP" -Direction Inbound -Protocol TCP -LocalPort 80 -Action Allow
New-NetFirewallRule -DisplayName "HTTPS" -Direction Inbound -Protocol TCP -LocalPort 443 -Action Allow

# --- 4. Set up SSH (OpenSSH Server) ---
Add-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0
Start-Service sshd
Set-Service -Name sshd -StartupType Automatic

# Configure key-based auth
$authorizedKeysPath = "C:\ProgramData\ssh\administrators_authorized_keys"
# Add your CI/CD public key to this file
```

## SSH key generation (run on your local machine or in CI)

```bash
# Generate a dedicated key pair for deployments
ssh-keygen -t ed25519 -C "github-actions-deploy" -f ~/.ssh/deploy_key -N ""

# The private key (~/.ssh/deploy_key) goes into GitHub Secrets as SSH_PRIVATE_KEY
# The public key (~/.ssh/deploy_key.pub) goes into the server's authorized_keys
```

## Manual deployment (without CI/CD)

For first deploy or debugging, run from the project root:

```bash
# Build and start all services
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d --build

# Check status
docker compose ps

# View logs
docker compose logs -f

# Restart a single service
docker compose restart app

# Full teardown (preserves volumes)
docker compose down

# Full teardown including volumes (DESTRUCTIVE)
docker compose down -v
```
