# VS Code Server mit Entra ID OIDC Setup

Dieses Dokument erklärt, wie Sie den VS Code Server (code-server) mit Microsoft Entra ID (ehemals Azure AD) Single Sign-On (SSO) einrichten.

## Übersicht

Das Setup besteht aus drei Komponenten:
1. **code-server**: VS Code im Browser
2. **oauth2-proxy**: OIDC-Authentifizierung vor code-server
3. **Traefik**: Reverse Proxy mit SSL/TLS

```
Internet → Traefik (SSL) → oauth2-proxy (Entra ID Auth) → code-server
```

## 1. Entra ID App Registration erstellen

### Schritt 1: Azure Portal öffnen
1. Gehe zu [Azure Portal](https://portal.azure.com)
2. Navigiere zu **Microsoft Entra ID** (früher Azure Active Directory)
3. Klicke auf **App registrations** → **New registration**

### Schritt 2: App registrieren
- **Name**: `Babsy Code Server`
- **Supported account types**:
  - Wähle "Accounts in this organizational directory only" für Single-Tenant
  - Oder "Accounts in any organizational directory" für Multi-Tenant
- **Redirect URI**:
  - Type: `Web`
  - URL: `https://code.test.babsy.ch/oauth2/callback`

Klicke auf **Register**.

### Schritt 3: Client Secret erstellen
1. Gehe zu **Certificates & secrets**
2. Klicke auf **New client secret**
3. Beschreibung: `OAuth2 Proxy Secret`
4. Expires: Wähle eine passende Laufzeit (z.B. 24 Monate)
5. Klicke auf **Add**
6. **WICHTIG**: Kopiere den Secret-Wert sofort! Er wird nur einmal angezeigt.

### Schritt 4: API Permissions konfigurieren
1. Gehe zu **API permissions**
2. Klicke auf **Add a permission**
3. Wähle **Microsoft Graph** → **Delegated permissions**
4. Füge folgende Permissions hinzu:
   - `openid`
   - `profile`
   - `email`
   - `User.Read`
5. Klicke auf **Grant admin consent for [Your Organization]**

### Schritt 5: Notiere die Werte
Im **Overview** Tab findest du:
- **Application (client) ID**: `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx`
- **Directory (tenant) ID**: `yyyyyyyy-yyyy-yyyy-yyyy-yyyyyyyyyyyy`

## 2. Environment Variables konfigurieren

Erstelle oder erweitere die `.env` Datei im Projekt-Root:

```bash
# VS Code Server / OAuth2 Proxy Settings
# ========================================

# Entra ID Client Credentials
ENTRA_CLIENT_ID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
ENTRA_CLIENT_SECRET=your-client-secret-value

# Entra ID OIDC Issuer URL
# Format: https://login.microsoftonline.com/{tenant-id}/v2.0
ENTRA_ISSUER_URL=https://login.microsoftonline.com/yyyyyyyy-yyyy-yyyy-yyyy-yyyyyyyyyyyy/v2.0

# OAuth2 Redirect URL (muss mit App Registration übereinstimmen)
ENTRA_REDIRECT_URL=https://code.test.babsy.ch/oauth2/callback

# OAuth2 Cookie Secret (generiere einen zufälligen String)
# Generieren mit: python3 -c 'import os,base64; print(base64.urlsafe_b64encode(os.urandom(32)).decode())'
OAUTH2_COOKIE_SECRET=your-random-32-byte-base64-encoded-string

# Cookie Domain für OAuth2
OAUTH2_COOKIE_DOMAIN=.test.babsy.ch

# Optional: Code Server User
CODE_SERVER_USER=coder
```

### Cookie Secret generieren

**Linux/macOS:**
```bash
python3 -c 'import os,base64; print(base64.urlsafe_b64encode(os.urandom(32)).decode())'
```

**Oder mit openssl:**
```bash
openssl rand -base64 32 | tr -d "=+/" | cut -c1-32
```

## 3. DNS / Traefik Konfiguration

Stelle sicher, dass der DNS-Eintrag für `code.test.babsy.ch` auf deinen Server zeigt:

```
code.test.babsy.ch.  IN  A  <SERVER-IP>
```

Die Traefik-Konfiguration ist bereits im `docker-compose.yml` enthalten mit:
- SSL/TLS via Let's Encrypt
- Routing zu oauth2-proxy
- Port 4180 für oauth2-proxy

## 4. Services starten

### Initial Setup
```bash
# Docker Images bauen
docker compose build code-server oauth2-proxy

# Services starten
docker compose up -d code-server oauth2-proxy
```

### Logs überprüfen
```bash
# OAuth2 Proxy Logs
docker compose logs -f oauth2-proxy

# Code Server Logs
docker compose logs -f code-server
```

## 5. Zugriff testen

1. Öffne https://code.test.babsy.ch
2. Du wirst zu Microsoft Entra ID Login weitergeleitet
3. Melde dich mit deinem Organisations-Account an
4. Nach erfolgreicher Authentifizierung wirst du zu VS Code weitergeleitet

## 6. Benutzer-Verwaltung

### Zugriff einschränken
Per default können sich alle Benutzer der Organisation anmelden. Um den Zugriff einzuschränken:

#### Option 1: Über Azure Portal (empfohlen)
1. Gehe zu **Enterprise Applications**
2. Suche und öffne "Babsy Code Server"
3. Gehe zu **Users and groups**
4. Klicke auf **Add user/group**
5. Wähle die Benutzer/Gruppen aus, die Zugriff haben sollen
6. Gehe zu **Properties**
7. Setze **Assignment required?** auf **Yes**

#### Option 2: Via Email Domains (oauth2-proxy config)
Bearbeite `docker/oauth2-proxy/oauth2-proxy.cfg`:
```conf
# Nur bestimmte Domains erlauben
email_domains = ["babsy.ch", "example.com"]

# Oder bestimmte E-Mails
authenticated_emails_file = "/etc/oauth2-proxy/emails.txt"
```

Erstelle `docker/oauth2-proxy/emails.txt`:
```
user1@babsy.ch
user2@babsy.ch
```

Dann in `docker-compose.yml` Volume hinzufügen:
```yaml
volumes:
  - ./docker/oauth2-proxy/oauth2-proxy.cfg:/etc/oauth2-proxy/oauth2-proxy.cfg:ro
  - ./docker/oauth2-proxy/emails.txt:/etc/oauth2-proxy/emails.txt:ro
```

## 7. VS Code Extensions

Die folgenden Extensions sind bereits vorinstalliert:
- ESLint
- Prettier
- Tailwind CSS IntelliSense
- TypeScript Next
- DotENV
- GitLens

### Weitere Extensions installieren
1. Öffne VS Code im Browser
2. Klicke auf Extensions (Strg+Shift+X)
3. Suche und installiere gewünschte Extensions

Oder füge sie im Dockerfile hinzu:
```dockerfile
RUN code-server --install-extension <extension-id>
```

## 8. Workspace Persistence

Der Workspace wird in einem Docker Volume gespeichert:
- Volume Name: `code-data`
- Mount Path: `/home/coder/workspace`

Das Projekt-Verzeichnis wird automatisch unter `/home/coder/workspace/project` gemountet.

### Backup erstellen
```bash
docker run --rm \
  -v babsy_code-data:/source:ro \
  -v $(pwd)/backups:/backup \
  alpine tar czf /backup/code-workspace-$(date +%Y%m%d-%H%M%S).tar.gz -C /source .
```

### Backup wiederherstellen
```bash
docker run --rm \
  -v babsy_code-data:/target \
  -v $(pwd)/backups:/backup \
  alpine sh -c "cd /target && tar xzf /backup/code-workspace-YYYYMMDD-HHMMSS.tar.gz"
```

## 9. Troubleshooting

### OAuth2 Proxy startet nicht
```bash
# Logs prüfen
docker compose logs oauth2-proxy

# Häufige Fehler:
# - ENTRA_CLIENT_SECRET falsch oder abgelaufen
# - ENTRA_ISSUER_URL Format falsch
# - OAUTH2_COOKIE_SECRET zu kurz oder ungültig
```

### Infinite Redirect Loop
- Prüfe, ob Cookie Domain korrekt ist (`.test.babsy.ch`)
- Prüfe, ob Redirect URL in Entra ID App Registration korrekt ist
- Lösche Browser Cookies für `code.test.babsy.ch`

### 403 Forbidden nach Login
- Prüfe `email_domains` in oauth2-proxy.cfg
- Prüfe User Assignment in Azure Enterprise Applications

### Code Server nicht erreichbar
```bash
# Container Status prüfen
docker compose ps code-server

# Logs prüfen
docker compose logs code-server

# Netzwerk testen
docker compose exec oauth2-proxy ping code-server
```

### Extensions funktionieren nicht
- Prüfe Dateiberechtigungen im Volume
- Neustart von code-server: `docker compose restart code-server`

## 10. Sicherheits-Best Practices

1. **Cookie Secret regelmäßig rotieren** (z.B. alle 6 Monate)
2. **Client Secret in Entra ID regelmäßig erneuern**
3. **Audit Logs in Azure überwachen**
4. **Multi-Factor Authentication (MFA) für Benutzer aktivieren**
5. **Conditional Access Policies in Entra ID konfigurieren**
6. **IP-Whitelisting via Traefik erwägen** (für zusätzliche Sicherheit)

### IP-Whitelisting (optional)
Füge in `docker-compose.yml` bei oauth2-proxy Labels hinzu:
```yaml
labels:
  - "traefik.http.middlewares.code-ipwhitelist.ipwhitelist.sourcerange=192.168.1.0/24,10.0.0.0/8"
  - "traefik.http.routers.babsy-code.middlewares=code-ipwhitelist"
```

## 11. Updates

### OAuth2 Proxy aktualisieren
```bash
docker compose pull oauth2-proxy
docker compose up -d oauth2-proxy
```

### Code Server aktualisieren
```bash
docker compose build --no-cache code-server
docker compose up -d code-server
```

## Support

Bei Problemen:
1. Prüfe die Logs: `docker compose logs oauth2-proxy code-server`
2. Prüfe die Entra ID Sign-in Logs im Azure Portal
3. Erstelle ein Issue im GitHub Repository
