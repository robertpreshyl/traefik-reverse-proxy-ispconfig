# Traefik Plugins Configuration Guide

This document provides detailed configuration examples for all installed Traefik plugins.

---

## 📦 Installed Plugins

- **Fail2Ban** (v0.8.7) - Brute force protection
- **Traefik OIDC** (v0.7.8) - Enterprise SSO/OAuth2
- **GeoBlock** (v0.3.3) - Geographic access control

---

## 🚫 Fail2Ban Plugin

### Overview
Automatically bans IP addresses after multiple failed authentication attempts, protecting against brute force attacks.

### Configuration

Located at: `/opt/traefik/config/dynamic/fail2ban-config.yml`

```yaml
http:
  middlewares:
    fail2ban:
      plugin:
        fail2ban:
          # Whitelist - IPs that will never be banned
          allowlist:
            ip: "::1,127.0.0.1,100.66.0.0/16"
          
          # Blacklist - Manually banned IPs (optional)
          denylist:
            ip: "192.168.1.100,203.0.113.0/24"
          
          # Ban rules
          rules:
            bantime: "3h"           # How long to ban (3 hours)
            enabled: "true"         # Enable the rule
            findtime: "10m"         # Time window for counting attempts
            maxretry: "4"           # Max failed attempts before ban
            statuscode: "400,401,403-499"  # Status codes that count as failures
```

### Usage

Apply to a router:

```yaml
labels:
  - "traefik.http.routers.myapp.middlewares=fail2ban@file"
```

### Examples

**Strict Protection** (Admin Panel):
```yaml
rules:
  bantime: "24h"
  findtime: "5m"
  maxretry: "3"
  statuscode: "401,403"
```

**Relaxed Protection** (Public API):
```yaml
rules:
  bantime: "1h"
  findtime: "15m"
  maxretry: "10"
  statuscode: "401"
```

---

## 🔑 Traefik OIDC Plugin

### Overview
Enables enterprise Single Sign-On (SSO) using OAuth2/OIDC providers like Google, Azure AD, Keycloak, Okta.

### Setup Steps

#### 1. Create OAuth Application

**Google Example:**
1. Go to [Google Cloud Console](https://console.cloud.google.com/)
2. Create project → APIs & Services → Credentials
3. Create OAuth 2.0 Client ID
4. Add authorized redirect URI: `https://prxy.allyshipglobal.com:8443/oauth2/callback`
5. Save Client ID and Client Secret

**Azure AD Example:**
1. Azure Portal → Azure Active Directory → App registrations
2. New registration
3. Add redirect URI: `https://prxy.allyshipglobal.com:8443/oauth2/callback`
4. Certificates & secrets → New client secret
5. Save Application ID and Secret

#### 2. Generate Session Key

```bash
openssl rand -base64 32
```

#### 3. Configuration

Create `/opt/traefik/config/dynamic/oidc-production.yml`:

```yaml
http:
  middlewares:
    oidc-google:
      plugin:
        traefikoidc:
          # Provider Configuration
          providerURL: "https://accounts.google.com"
          clientID: "123456789.apps.googleusercontent.com"
          clientSecret: "your-client-secret-here"
          
          # URLs
          callbackURL: "/oauth2/callback"
          logoutURL: "/oauth2/logout"
          postLogoutRedirectURI: "/"
          
          # Session Security
          sessionEncryptionKey: "your-32-byte-random-key-here"
          enablePKCE: "true"
          forceHTTPS: "true"
          
          # Access Control
          allowedUserDomains:
            - "allyshipglobal.com"
          
          # Exclude public paths
          excludedURLs:
            - "/health"
            - "/metrics"
            - "/public"
          
          # User Info Headers
          headers:
            - name: "X-User-Email"
              value: "{{.Claims.email}}"
            - name: "X-User-Name"
              value: "{{.Claims.name}}"
            - name: "X-User-ID"
              value: "{{.Claims.sub}}"
          
          # Settings
          logLevel: "info"
          rateLimit: "100"
```

### Usage

```yaml
labels:
  - "traefik.http.routers.myapp.middlewares=oidc-google@file"
```

### Provider Examples

#### Azure AD

```yaml
providerURL: "https://login.microsoftonline.com/{tenant-id}/v2.0"
clientID: "your-application-id"
clientSecret: "your-secret"
scopes:
  - "openid"
  - "profile"
  - "email"
```

#### Keycloak

```yaml
providerURL: "https://keycloak.example.com/realms/myrealm"
clientID: "traefik-client"
clientSecret: "your-secret"
```

#### Auth0

```yaml
providerURL: "https://your-tenant.auth0.com"
clientID: "your-client-id"
clientSecret: "your-secret"
```

---

## 🌍 GeoBlock Plugin

### Overview
Restricts access based on visitor's geographic location using IP geolocation.

### Configuration

Located at: `/opt/traefik/config/dynamic/geoblock-config.yml`

```yaml
http:
  middlewares:
    geoblock-allowed:
      plugin:
        geoblock:
          allowLocalRequests: "true"
          allowUnknownCountries: "false"
          
          # GeoIP API
          api: "https://get.geojs.io/v1/ip/country/{ip}"
          apiTimeoutMs: "150"
          cacheSize: "15"
          
          # Allowed Countries (ISO 2-letter codes)
          countries:
            - "US"  # United States
            - "CA"  # Canada
            - "GB"  # United Kingdom
            - "DE"  # Germany
            - "FR"  # France
            - "NG"  # Nigeria
          
          # Logging
          forceMonthlyUpdate: "true"
          logAllowedRequests: "false"
          logApiRequests: "true"
          logLocalRequests: "false"
          silentStartUp: "false"
```

### Usage

```yaml
labels:
  - "traefik.http.routers.myapp.middlewares=geoblock-allowed@file"
```

### Examples

**Block Specific Countries:**
```yaml
# Use a blocklist approach - allow all except:
allowUnknownCountries: "true"
# Then use fail2ban to block specific countries
denylist:
  countries:
    - "CN"
    - "RU"
    - "KP"
```

**Strict Whitelist:**
```yaml
allowUnknownCountries: "false"
countries:
  - "US"
  - "CA"
```

---

## 🔗 Combining Middlewares

You can chain multiple middlewares together:

```yaml
labels:
  - "traefik.http.routers.admin.middlewares=security-headers@file,fail2ban@file,oidc-google@file"
```

**Order matters:**
1. Security headers (first)
2. Fail2ban (early filtering)
3. OIDC (authentication)
4. Your app-specific middlewares

---

## 🔄 Applying Changes

After modifying plugin configurations:

```bash
# Traefik watches for file changes automatically
# Or manually restart:
cd /opt/traefik
sudo docker compose restart
```

Check logs for errors:

```bash
sudo docker logs traefik | grep -i error
sudo tail -f /opt/traefik/logs/traefik.log
```

---

## 📚 Additional Resources

- **Fail2Ban Plugin**: https://plugins.traefik.io/plugins/628c9ebcffc0cd18356a979f/fail2-ban
- **Traefik OIDC Plugin**: https://plugins.traefik.io/plugins/6613338ea28c508f411a44d5/traefik-oidc
- **GeoBlock Plugin**: https://plugins.traefik.io/plugins/65c37fd5e4c5c70685de8f7c/geoblock
- **Traefik Middleware Docs**: https://doc.traefik.io/traefik/middlewares/overview/

---

## 🛡️ Security Best Practices

1. **Always use HTTPS** - Set `forceHTTPS: "true"` for OIDC
2. **Rotate secrets** - Change session keys and client secrets regularly
3. **Monitor logs** - Watch for suspicious patterns
4. **Whitelist IPs** - Add trusted IPs to fail2ban allowlist
5. **Test in staging** - Verify configurations before production
6. **Keep updated** - Regularly update plugin versions

---

## 🐛 Troubleshooting

### Plugin Not Loading

```bash
# Check Traefik logs
sudo docker logs traefik | grep plugin

# Common issues:
# 1. Wrong version number
# 2. Network connectivity (plugins download at startup)
# 3. Syntax errors in YAML
```

### OIDC Redirect Loop

1. Verify callback URL matches OAuth app configuration
2. Check sessionEncryptionKey is set
3. Ensure forceHTTPS matches your setup

### GeoBlock Blocking Legitimate Users

1. Check IP geolocation: `curl https://get.geojs.io/v1/ip/country/YOUR_IP`
2. Add country to whitelist
3. Set `allowUnknownCountries: "true"` temporarily for testing

---

**For more help**, see the [main README](./README.md) or [deployment guide](./DEPLOYMENT_COMPLETE.md).
