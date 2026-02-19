# Code-Server

VS Code im Browser für das Babsy IT Team, abgesichert via Microsoft Entra ID (Azure AD) OIDC.

## Stack

- **[code-server](https://github.com/coder/code-server)** — VS Code im Browser
- **[oauth2-proxy](https://github.com/oauth2-proxy/oauth2-proxy)** — Entra ID OIDC Authentifizierung
- **[Traefik](https://traefik.io/)** — Reverse Proxy mit Let's Encrypt TLS

## URL

`https://code.test.babsy.ch` (Login via Microsoft Entra ID)

## Setup

1. `.env.example` → `.env` kopieren und ausfüllen
2. `docker compose up -d`

Detaillierte Anleitung: [docs/VSCODE-SETUP.md](docs/VSCODE-SETUP.md)

## Workspace

Das Workspace-Verzeichnis ist konfigurierbar via `HOST_WORKSPACE` in der `.env`.  
Standard: `./workspace` (lokaler Ordner im Repo-Verzeichnis)  
Empfohlen: `/home/stefan/projects` (Zugriff auf alle Repos)
