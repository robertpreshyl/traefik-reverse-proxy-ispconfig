# Traefik Deployment on ISPConfig VPS - Production Implementation GuideYou are a highly experienced DevOps/Networking AI assistant. Your task is to **install and configure Traefik on my Oracle ARM VPS** as a reverse proxy, while ensuring it does **not conflict** with any existing services.  



You are a highly experienced DevOps/Networking/Containerization AI assistant with expertise in reverse proxies, Docker orchestration, and production-grade infrastructure. Your task is to **install and configure Traefik v3 on an ISPConfig production VPS** as a reverse proxy for Docker containers, while ensuring **ZERO disruption** to existing services.**Current VPS setup:**

- NetBird (self-hosted, Docker) using **Caddy** for management/dashboard.

---- WireGuard VPN (host-level)

- OpenVPN server (host-level)

## 🎯 **Deployment Context**- Other Docker containers running various apps



### **Target Server:****Requirements:**

- **Primary VPS**: ISPConfig production server (web hosting platform)

- **Purpose**: Deploy Traefik to manage containerized applications separately from ISPConfig-managed websites1. **Pre-Installation Analysis**

- **Architecture**: Phased approach for zero-downtime deployment   - Thoroughly inspect the VPS network stack, currently used ports, and container services.

   - Ensure Traefik will not conflict with Caddy, VPNs, or existing Docker containers.

### **Why ISPConfig VPS (Not NetBird VPS):**   - Review relevant Traefik documentation for ARM deployment and security best practices.

- NetBird VPS runs critical VPN infrastructure (WireGuard, OpenVPN, NetBird mesh) with Caddy on ports 80/443

- Deploying Traefik there would create port conflicts and risk disrupting VPN services2. **Installation**

- ISPConfig VPS is the logical location for application containers and has the necessary resources   - Install Traefik via Docker (or Docker Compose) compatible with ARM.

   - Assign custom ports if necessary to avoid conflict with Caddy (used by NetBird) and VPN ports.

### **Current ISPConfig VPS Setup (UNKNOWN - MUST BE DISCOVERED):**   - Configure Traefik for **HTTPS with Let’s Encrypt** automatically.

- ISPConfig 3.x web hosting control panel

- Apache2 OR nginx web server (occupying ports 80/443)3. **Configuration**

- MySQL/MariaDB database server   - Enable the Traefik dashboard (internal, secured with strong credentials) on a safe port.

- PHP-FPM, Postfix, Dovecot (email services)   - Enable auto-discovery for Docker containers.

- Pure-FTPd or ProFTPD   - Prepare routing for future apps and potential routing to ISPConfig VPS websites.

- BIND DNS server (optional)   - Ensure routing rules do not interfere with VPN traffic (WireGuard or OpenVPN) or NetBird’s Caddy service.

- Multiple client websites hosted via ISPConfig   - Apply security best practices from the official Traefik docs.

- Potentially existing Docker containers

- Unknown resource utilization (CPU, RAM, disk)4. **Validation & Testing**

   - Test that existing VPN connections (NetBird, WireGuard, OpenVPN) remain fully functional.

---   - Confirm existing Docker containers and NetBird services are unaffected.

   - Validate Traefik is routing traffic correctly and HTTPS certificates are issued without issues.

## 📋 **CRITICAL PRE-IMPLEMENTATION REQUIREMENTS**   - Verify dashboard access is secure and only reachable from authorized clients.



### **Phase 0: Comprehensive System Discovery & Safety Checks**5. **Deliverables**

   - Fully functional `docker-compose.yml` or container setup.

Before ANY installation or configuration changes, you MUST:   - Sample Traefik dynamic routing config with comments.

   - Documentation on adding future services safely.

#### **1. System Identification & Resource Assessment**   - Confirmation of no conflicts with NetBird, Caddy, VPNs, or Docker apps.



```bash**Security & Safety Focus:**  

# Operating System & Architecture- Avoid any changes that could break existing VPN tunnels or NetBird service.

lsb_release -a- Follow official documentation for secure configuration.

uname -m- Consider port conflicts, container network isolation, and proper firewall rules.

hostnamectl

# Resource Availability
free -h                          # RAM (need at least 512MB free for Traefik)
df -h                            # Disk space (need at least 5GB free)
nproc                            # CPU cores
uptime                           # System load average

# Check if this is actually the ISPConfig server
ls -la /usr/local/ispconfig/
cat /usr/local/ispconfig/interface/lib/config.inc.php | grep -i version
```

#### **2. Critical Port Discovery & Conflict Analysis**

```bash
# COMPREHENSIVE port scan - identify ALL listeners
sudo ss -tulnp | sort -k5

# Specifically check critical ports
sudo ss -tulnp | grep -E ':(80|443|8080|8443|3306|25|110|143|587|993|995|53) '

# Check which web server is running
systemctl status apache2 2>/dev/null || echo "Apache not active"
systemctl status nginx 2>/dev/null || echo "Nginx not active"

# If Apache is running, check virtual hosts
apachectl -S 2>/dev/null || apache2ctl -S 2>/dev/null

# If nginx is running, check configuration
nginx -T 2>/dev/null | grep -E "listen|server_name" | head -50
```

#### **3. Docker Environment Assessment**

```bash
# Check if Docker is installed
docker --version
docker compose version  # or docker-compose --version

# If Docker exists, check current containers
docker ps -a
docker network ls
docker volume ls

# Check Docker daemon configuration
cat /etc/docker/daemon.json 2>/dev/null || echo "No custom Docker config"

# Check if any containers use ports 8080 or 8443
docker ps --format "table {{.Names}}\t{{.Ports}}" | grep -E "8080|8443"
```

#### **4. Firewall & Security Rules Analysis**

```bash
# Check firewall status (iptables/ufw/firewalld)
sudo iptables -L -n -v | head -30
sudo ufw status verbose 2>/dev/null || echo "UFW not active"
sudo firewall-cmd --list-all 2>/dev/null || echo "Firewalld not active"

# Check if fail2ban is running (important for security)
systemctl status fail2ban 2>/dev/null || echo "Fail2ban not installed"
```

#### **5. ISPConfig Service Health Check**

```bash
# Verify ISPConfig services are healthy BEFORE we start
systemctl status ispconfig
systemctl status apache2 || systemctl status nginx
systemctl status mysql || systemctl status mariadb
systemctl status postfix
systemctl status dovecot
systemctl status pure-ftpd-mysql || systemctl status proftpd

# Check if ISPConfig panel is accessible
curl -I https://localhost:8080 2>/dev/null || echo "ISPConfig panel not responding on 8080"
```

#### **6. DNS & Domain Configuration Discovery**

```bash
# Check server's primary domain/hostname
hostname -f

# Check /etc/hosts configuration
cat /etc/hosts

# If BIND DNS is running
named-checkconf 2>/dev/null && echo "BIND DNS is configured"
```

#### **7. SSL/TLS Certificate Analysis**

```bash
# Check for existing Let's Encrypt certificates
ls -la /etc/letsencrypt/live/ 2>/dev/null
certbot certificates 2>/dev/null || echo "Certbot not installed or no certs"

# Check ISPConfig's SSL certificate location
ls -la /usr/local/ispconfig/interface/ssl/
```

---

## 🚀 **Implementation Strategy - Phased Approach**

Based on expert DevOps recommendations, we will use a **two-phase deployment**:

### **Phase 1: Traefik on Alternative Ports (INITIAL DEPLOYMENT)**

**Objective**: Deploy Traefik safely without ANY changes to ISPConfig/Apache/nginx

**Configuration**:
- Traefik HTTP: **Port 8080** (Docker container port 80)
- Traefik HTTPS: **Port 8443** (Docker container port 443)
- Traefik Dashboard: **Port 8081** (secured with authentication)
- ISPConfig continues on ports 80/443 (UNCHANGED)

**Benefits**:
- ✅ **ZERO risk** to existing websites and ISPConfig panel
- ✅ Complete isolation between ISPConfig and Traefik
- ✅ Easy rollback (just stop Traefik container)
- ✅ Can run indefinitely in this configuration

**Trade-offs**:
- ⚠️ Docker apps accessible via non-standard ports (e.g., https://domain.com:8443/app)
- ⚠️ Requires firewall rules for ports 8080/8443

---

### **Phase 2: Traefik as Primary Frontend (OPTIONAL - FUTURE MIGRATION)**

**Objective**: Make Traefik the main reverse proxy on ports 80/443 (only after Phase 1 is stable)

**Configuration**:
- Traefik: Ports 80/443
- Apache/nginx: Moved to ports 8080/8081 (backend only)
- Traefik routes:
  - `*.yourdomain.com` → ISPConfig backend (Apache/nginx on 8080)
  - `app1.yourdomain.com` → Docker container
  - `app2.yourdomain.com` → Docker container

**Benefits**:
- ✅ Unified SSL certificate management
- ✅ Central logging and monitoring
- ✅ Advanced routing capabilities
- ✅ Better performance and caching options

**Complexity**:
- ⚠️ Requires reconfiguring ISPConfig web server
- ⚠️ Potential 5-10 minute downtime during migration
- ⚠️ Must test thoroughly before implementing

**Note**: Phase 2 should ONLY be implemented after Phase 1 runs successfully for at least 1-2 weeks.

---

## 🔧 **Requirements for Phase 1 Implementation**

### **1. Pre-Installation Analysis (MANDATORY)**

You MUST complete ALL checks from Phase 0 and provide:

- [ ] Confirmation that this is an ISPConfig server
- [ ] Web server type (Apache or nginx) and version
- [ ] List of ALL ports currently in use
- [ ] Available system resources (RAM, CPU, disk)
- [ ] Docker installation status
- [ ] Firewall configuration type (iptables/ufw/firewalld)
- [ ] Existing SSL certificates (if any)
- [ ] List of active ISPConfig services
- [ ] Current system load and stability confirmation

**DO NOT PROCEED** until ALL safety checks pass.

---

### **2. Docker Installation & Configuration**

If Docker is not installed or outdated:

```bash
# Install Docker (Ubuntu/Debian)
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Install Docker Compose V2
sudo apt-get update
sudo apt-get install -y docker-compose-plugin

# Verify installation
docker --version
docker compose version

# Add non-root user to docker group (if needed)
sudo usermod -aG docker $USER
newgrp docker
```

**Security considerations**:
- Verify Docker daemon runs with default security settings
- Ensure Docker socket is NOT exposed to network
- Configure Docker logging to prevent disk space issues

---

### **3. Traefik Installation (Phase 1 - Alternative Ports)**

#### **Directory Structure**

```bash
# Create Traefik directory structure
sudo mkdir -p /opt/traefik/{config,logs,acme}
sudo chmod -R 755 /opt/traefik
```

#### **docker-compose.yml Configuration**

Create `/opt/traefik/docker-compose.yml`:

```yaml
version: '3.8'

services:
  traefik:
    image: traefik:v3.0
    container_name: traefik
    restart: unless-stopped
    security_opt:
      - no-new-privileges:true
    networks:
      - traefik-public
    ports:
      # Phase 1: Alternative ports to avoid ISPConfig conflicts
      - "8080:80"      # HTTP (external 8080 → container 80)
      - "8443:443"     # HTTPS (external 8443 → container 443)
      - "8081:8080"    # Traefik Dashboard (secured)
    environment:
      - TZ=UTC
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./config/traefik.yml:/traefik.yml:ro
      - ./config/dynamic:/etc/traefik/dynamic:ro
      - ./acme:/acme
      - ./logs:/var/log/traefik
    labels:
      - "traefik.enable=true"
      
      # Dashboard configuration (secured with BasicAuth)
      - "traefik.http.routers.dashboard.rule=Host(`traefik.yourdomain.com`) || PathPrefix(`/dashboard`) || PathPrefix(`/api`)"
      - "traefik.http.routers.dashboard.service=api@internal"
      - "traefik.http.routers.dashboard.entrypoints=websecure"
      - "traefik.http.routers.dashboard.tls=true"
      - "traefik.http.routers.dashboard.tls.certresolver=letsencrypt"
      - "traefik.http.routers.dashboard.middlewares=dashboard-auth"
      
      # BasicAuth middleware (replace with your credentials)
      # Generate with: htpasswd -nb admin yourpassword | sed 's/\$/\$\$/g'
      - "traefik.http.middlewares.dashboard-auth.basicauth.users=admin:$$apr1$$REPLACE_WITH_HASHED_PASSWORD"

networks:
  traefik-public:
    name: traefik-public
    driver: bridge
```

#### **traefik.yml Configuration**

Create `/opt/traefik/config/traefik.yml`:

```yaml
# Traefik Static Configuration
global:
  checkNewVersion: true
  sendAnonymousUsage: false

# API and Dashboard
api:
  dashboard: true
  debug: false
  insecure: false  # Dashboard requires auth

# Entry Points (Phase 1: Alternative ports)
entryPoints:
  web:
    address: ":80"  # Container port 80 (mapped to host 8080)
    http:
      redirections:
        entryPoint:
          to: websecure
          scheme: https
          permanent: true
  
  websecure:
    address: ":443"  # Container port 443 (mapped to host 8443)
    http:
      tls:
        certResolver: letsencrypt
        domains:
          - main: yourdomain.com
            sans:
              - "*.yourdomain.com"

# Docker Provider
providers:
  docker:
    endpoint: "unix:///var/run/docker.sock"
    exposedByDefault: false  # Only containers with traefik.enable=true
    network: traefik-public
    watch: true
  
  file:
    directory: "/etc/traefik/dynamic"
    watch: true

# Let's Encrypt Certificate Resolver
certificatesResolvers:
  letsencrypt:
    acme:
      email: your-email@example.com  # REPLACE WITH YOUR EMAIL
      storage: /acme/acme.json
      keyType: EC384
      httpChallenge:
        entryPoint: web
      # For wildcard certificates, use dnsChallenge instead:
      # dnsChallenge:
      #   provider: cloudflare  # or your DNS provider
      #   delayBeforeCheck: 0

# Logging
log:
  level: INFO  # DEBUG, INFO, WARN, ERROR
  filePath: /var/log/traefik/traefik.log
  format: json

accessLog:
  filePath: /var/log/traefik/access.log
  format: json
  bufferingSize: 100

# Metrics (optional)
metrics:
  prometheus:
    addEntryPointsLabels: true
    addRoutersLabels: true
    addServicesLabels: true

# Pilot/Observability (optional)
# pilot:
#   token: "your-pilot-token"
```

#### **Dynamic Configuration (Optional Security Headers)**

Create `/opt/traefik/config/dynamic/security-headers.yml`:

```yaml
# Security headers middleware
http:
  middlewares:
    security-headers:
      headers:
        frameDeny: true
        sslRedirect: true
        browserXssFilter: true
        contentTypeNosniff: true
        forceSTSHeader: true
        stsIncludeSubdomains: true
        stsPreload: true
        stsSeconds: 31536000
        customFrameOptionsValue: "SAMEORIGIN"
        customResponseHeaders:
          X-Robots-Tag: "none,noarchive,nosnippet,notranslate,noimageindex"
          server: ""  # Remove server header
    
    rate-limit:
      rateLimit:
        average: 100
        burst: 50
```

---

### **4. Security Hardening (Industry Standards)**

#### **A. Generate Secure Dashboard Credentials**

```bash
# Install apache2-utils for htpasswd
sudo apt-get install -y apache2-utils

# Generate hashed password (replace 'yourpassword' with strong password)
htpasswd -nb admin yourpassword | sed 's/\$/\$\$/g'

# Copy output and paste into docker-compose.yml dashboard-auth label
```

#### **B. Secure acme.json File**

```bash
# Create and secure Let's Encrypt storage
sudo touch /opt/traefik/acme/acme.json
sudo chmod 600 /opt/traefik/acme/acme.json
```

#### **C. Firewall Configuration**

```bash
# If using UFW
sudo ufw allow 8080/tcp comment 'Traefik HTTP'
sudo ufw allow 8443/tcp comment 'Traefik HTTPS'
sudo ufw allow 8081/tcp comment 'Traefik Dashboard'

# If using iptables directly
sudo iptables -I INPUT -p tcp --dport 8080 -j ACCEPT -m comment --comment "Traefik HTTP"
sudo iptables -I INPUT -p tcp --dport 8443 -j ACCEPT -m comment --comment "Traefik HTTPS"
sudo iptables -I INPUT -p tcp --dport 8081 -j ACCEPT -m comment --comment "Traefik Dashboard"

# Save iptables rules
sudo iptables-save | sudo tee /etc/iptables/rules.v4
```

#### **D. Docker Socket Security**

```bash
# Verify Docker socket permissions
ls -la /var/run/docker.sock

# Should be owned by root:docker with 660 permissions
# If not, fix it:
sudo chown root:docker /var/run/docker.sock
sudo chmod 660 /var/run/docker.sock
```

---

### **5. Deployment & Validation**

#### **Step 1: Pre-Deployment Backup**

```bash
# Backup ISPConfig configuration
sudo tar -czf /root/ispconfig-backup-$(date +%F).tar.gz \
  /usr/local/ispconfig \
  /etc/apache2 \
  /etc/nginx \
  2>/dev/null

# Backup current iptables rules
sudo iptables-save > /root/iptables-backup-$(date +%F).txt
```

#### **Step 2: Deploy Traefik**

```bash
cd /opt/traefik

# Validate docker-compose.yml syntax
docker compose config

# Start Traefik
docker compose up -d

# Check logs for errors
docker compose logs -f traefik
```

#### **Step 3: Comprehensive Validation**

```bash
# 1. Verify Traefik container is running
docker ps | grep traefik

# 2. Check Traefik is listening on correct ports
sudo ss -tulnp | grep -E ':(8080|8443|8081) '

# 3. Verify ISPConfig services remain healthy
systemctl status apache2 || systemctl status nginx
systemctl status mysql || systemctl status mariadb
curl -I http://localhost  # Should return ISPConfig/Apache response

# 4. Test Traefik dashboard access (replace with your domain)
curl -k https://localhost:8081/dashboard/

# 5. Check Traefik logs for issues
docker compose logs traefik | grep -i error

# 6. Verify Docker network
docker network inspect traefik-public
```

#### **Step 4: Deploy Test Container**

```bash
# Create test application
cat > /opt/traefik/docker-compose-test.yml <<EOF
version: '3.8'

services:
  whoami:
    image: traefik/whoami
    container_name: whoami-test
    restart: unless-stopped
    networks:
      - traefik-public
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.whoami.rule=Host(\`test.yourdomain.com\`)"
      - "traefik.http.routers.whoami.entrypoints=websecure"
      - "traefik.http.routers.whoami.tls=true"
      - "traefik.http.routers.whoami.tls.certresolver=letsencrypt"
      - "traefik.http.services.whoami.loadbalancer.server.port=80"

networks:
  traefik-public:
    external: true
EOF

# Deploy test container
docker compose -f /opt/traefik/docker-compose-test.yml up -d

# Test access (replace with your domain - remember port 8443!)
curl -k https://test.yourdomain.com:8443
```

---

### **6. Monitoring & Maintenance**

```bash
# Check Traefik logs
docker compose -f /opt/traefik/docker-compose.yml logs -f

# Monitor resource usage
docker stats traefik

# View active routes
curl -s http://localhost:8081/api/http/routers | jq

# Check certificate status
docker compose exec traefik cat /acme/acme.json | jq
```

---

## 📋 **Deliverables**

Upon successful completion, provide:

1. **System Discovery Report**:
   - All pre-installation checks completed
   - Port usage analysis
   - Resource availability confirmation
   - ISPConfig service health status

2. **Deployment Artifacts**:
   - Complete `/opt/traefik/` directory structure
   - `docker-compose.yml` with proper configurations
   - `traefik.yml` static configuration
   - Dynamic configuration files
   - Firewall rules applied

3. **Validation Report**:
   - Traefik container status
   - Port binding confirmation
   - ISPConfig services health check (post-deployment)
   - Test application deployment success
   - SSL certificate issuance confirmation

4. **Documentation**:
   - Step-by-step commands used
   - Any issues encountered and resolutions
   - Rollback procedure (if needed)
   - Instructions for adding new Docker services
   - Phase 2 migration guide (for future reference)

5. **Security Audit**:
   - Dashboard authentication verified
   - Firewall rules documented
   - Docker socket permissions confirmed
   - SSL/TLS configuration validated
   - Security headers implementation confirmed

---

## 🚨 **Critical Safety Rules**

### **NEVER DO THESE WITHOUT EXPLICIT CONFIRMATION:**

- ❌ **DO NOT** modify Apache/nginx configuration files
- ❌ **DO NOT** change ISPConfig settings or database
- ❌ **DO NOT** stop or restart ISPConfig services
- ❌ **DO NOT** bind Traefik to ports 80/443 in Phase 1
- ❌ **DO NOT** modify existing iptables rules (only ADD new rules)
- ❌ **DO NOT** change Docker daemon configuration
- ❌ **DO NOT** proceed if any pre-checks fail

### **ALWAYS DO THESE:**

- ✅ **CREATE** backups before ANY changes
- ✅ **VERIFY** ISPConfig remains functional after each step
- ✅ **TEST** with non-production domains first
- ✅ **MONITOR** logs continuously during deployment
- ✅ **DOCUMENT** every command executed
- ✅ **PROVIDE** rollback steps if issues occur

---

## 🔄 **Rollback Procedure**

If anything goes wrong:

```bash
# 1. Stop Traefik immediately
cd /opt/traefik
docker compose down

# 2. Remove firewall rules
sudo ufw delete allow 8080/tcp
sudo ufw delete allow 8443/tcp
sudo ufw delete allow 8081/tcp

# 3. Verify ISPConfig is healthy
systemctl status apache2 || systemctl status nginx
curl -I http://localhost

# 4. Restore iptables if modified
sudo iptables-restore < /root/iptables-backup-*.txt

# 5. Clean up (optional)
docker network rm traefik-public
rm -rf /opt/traefik
```

---

## 📚 **References & Best Practices**

- [Traefik Official Documentation](https://doc.traefik.io/traefik/)
- [Traefik Docker Provider](https://doc.traefik.io/traefik/providers/docker/)
- [Let's Encrypt Rate Limits](https://letsencrypt.org/docs/rate-limits/)
- [Docker Security Best Practices](https://docs.docker.com/engine/security/)
- [ISPConfig Documentation](https://www.ispconfig.org/documentation/)

---

## ❓ **Information Required Before Implementation**

Please provide:

1. **Server Access**: SSH credentials or access method
2. **Domain Name**: Primary domain for Traefik dashboard
3. **Email Address**: For Let's Encrypt certificate notifications
4. **Subdomain Availability**: Can you create DNS A records? (e.g., traefik.yourdomain.com, test.yourdomain.com)
5. **Preferred Authentication**: Dashboard username/password
6. **Monitoring Preference**: Do you want Prometheus metrics enabled?
7. **Backup Confirmation**: Acknowledge that you have ISPConfig backups

---

**REMEMBER**: This is a production server. Safety, stability, and zero-downtime are the top priorities. Every step must be validated before proceeding to the next.
