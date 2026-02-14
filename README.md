# Setup Server - Infrastructure Ansible

Infrastructure as Code pour le provisionnement automatique d'un serveur VPS avec Traefik, CrowdSec et stack de monitoring complète.

## Architecture

```
Internet
   |
   v
[Traefik v3] --- Let's Encrypt (HTTPS automatique)
   |
   +-- CrowdSec Bouncer (protection WAF)
   |
   +-- Grafana    (dashboards)
   +-- Prometheus (métriques)
   +-- Loki       (logs)
   +-- Alertmanager (alertes Telegram + Email)
```

## Stack technique

| Composant | Version | Description |
|-----------|---------|-------------|
| Traefik | 3.1 | Reverse proxy + Let's Encrypt + TLS 1.2+ |
| CrowdSec | 1.6 | IPS collaboratif + bouncer Traefik |
| Prometheus | 2.53 | Collecte de métriques |
| Grafana | 11.0 | Dashboards et visualisation |
| Loki | 3.1 | Agrégation de logs |
| Promtail | 3.1 | Collecte de logs Docker + syslog |
| Node Exporter | 1.8 | Métriques système |
| cAdvisor | 0.49 | Métriques containers Docker |
| Alertmanager | 0.27 | Routage des alertes |

## Prérequis

- **Machine locale** : Python 3.10+, Ansible 2.15+
- **Serveur cible** : Debian 12 / Ubuntu 22.04+, accès SSH root ou sudo

### Installation des dépendances

```bash
pip install ansible ansible-lint yamllint
ansible-galaxy install -r requirements.yml
```

## Structure du projet

```
setup-server/
├── ansible.cfg                     # Configuration Ansible
├── site.yml                        # Playbook principal
├── requirements.yml                # Collections Ansible
│
├── inventories/
│   ├── production/hosts.yml        # Inventaire production
│   └── staging/hosts.yml           # Inventaire staging
│
├── group_vars/all/
│   ├── main.yml                    # Variables globales
│   └── vault.yml                   # Secrets (chiffré)
│
├── playbooks/
│   ├── setup-base.yml              # Setup système + Docker
│   ├── deploy-stack.yml            # Déploiement stack complète
│   └── update-monitoring.yml       # MAJ monitoring seul
│
└── roles/
    ├── common/                     # Hardening système
    ├── docker/                     # Docker CE + Compose
    ├── traefik/                    # Reverse proxy
    ├── crowdsec/                   # IPS + bouncer
    ├── monitoring/                 # Prometheus + Grafana + Loki
    │   └── files/dashboards/       # 4 dashboards JSON
    └── alerting/                   # Alertmanager + notifications
```

## Configuration

### 1. Configurer les secrets

Modifier `group_vars/all/vault.yml` avec vos valeurs :

```yaml
# Serveur
vault_server_ip: "203.0.113.10"
vault_base_domain: "mondomaine.fr"
vault_acme_email: "admin@mondomaine.fr"

# Connexion initiale (premier run)
vault_initial_user: "root"
vault_initial_ssh_port: 22

# Config cible (après hardening)
vault_new_user: "deploy"
vault_new_ssh_port: 2222

# Services
vault_traefik_dashboard_password: "mot_de_passe_fort"
vault_crowdsec_enroll_key: "cle_crowdsec"
vault_crowdsec_bouncer_key: "cle_bouncer"
vault_grafana_admin_password: "mot_de_passe_grafana"
vault_telegram_bot_token: "123456:ABC..."
vault_telegram_chat_id: "-1001234567890"
vault_smtp_host: "smtp.example.com"
vault_smtp_password: "mot_de_passe_smtp"
```

Puis chiffrer le fichier :

```bash
ansible-vault encrypt group_vars/all/vault.yml
```

### 2. Configurer l'inventaire

Modifier `inventories/production/hosts.yml` avec l'IP et le port SSH de votre serveur.

### 3. Personnaliser les variables

Ajuster `group_vars/all/main.yml` selon vos besoins (timezone, versions, domaine, etc.).

## Utilisation

### Premier run (serveur vierge)

Le vault est pré-configuré pour se connecter en `root` sur le port `22`. Le playbook va :

1. Créer l'utilisateur défini dans `vault_new_user` (ex: `deploy`)
2. Changer le port SSH vers `vault_new_ssh_port` (ex: `2222`)
3. Désactiver la connexion root et l'auth par mot de passe

```bash
ansible-playbook site.yml --ask-vault-pass
```

### Runs suivants

Après le premier run, mettre à jour le vault pour utiliser le nouvel utilisateur et port :

```yaml
vault_initial_user: "deploy"       # était "root"
vault_initial_ssh_port: 2222       # était 22
```

Puis relancer normalement :

```bash
ansible-playbook site.yml --ask-vault-pass
```

### Déploiement partiel par tags

```bash
# Système de base uniquement
ansible-playbook site.yml --tags common,docker --ask-vault-pass

# Traefik + CrowdSec
ansible-playbook site.yml --tags traefik,crowdsec --ask-vault-pass

# Monitoring uniquement
ansible-playbook site.yml --tags monitoring,alerting --ask-vault-pass
```

### Playbooks spécialisés

```bash
# Setup base (common + docker)
ansible-playbook playbooks/setup-base.yml --ask-vault-pass

# Déployer/mettre à jour la stack Docker
ansible-playbook playbooks/deploy-stack.yml --ask-vault-pass

# Mettre à jour le monitoring
ansible-playbook playbooks/update-monitoring.yml --ask-vault-pass
```

### Avec un fichier de mot de passe vault

```bash
echo "votre_mot_de_passe" > .vault_pass
chmod 600 .vault_pass
ansible-playbook site.yml --vault-password-file .vault_pass
```

## Rôles

### common
Hardening du serveur :
- Mise à jour système et installation des paquets essentiels
- Création utilisateur `deploy` avec sudo NOPASSWD
- Durcissement SSH (changement de port, désactivation root/password)
- Pare-feu UFW (ports 80, 443, SSH uniquement)
- Configuration swap

### docker
- Installation Docker CE depuis le dépôt officiel
- Configuration daemon (logging, address pools)
- Création du réseau `traefik-public`

### traefik
- Traefik v3 avec Let's Encrypt automatique
- Redirection HTTP -> HTTPS
- TLS 1.2 minimum avec cipher suites sécurisées
- Headers de sécurité (HSTS, XSS, CSP)
- Rate limiting (100 req/s, burst 50)
- Dashboard protégé par Basic Auth
- Métriques Prometheus sur :8082

### crowdsec
- CrowdSec engine avec acquisition logs Traefik
- Bouncer Traefik (forwardAuth middleware)
- Collections : traefik, http-cve, linux
- Enrôlement console CrowdSec (optionnel)

### monitoring
- **Prometheus** : scrape node-exporter, cAdvisor, Traefik (15s)
- **Grafana** : provisionnement automatique datasources + dashboards
- **Loki** : agrégation logs avec schema v13
- **Promtail** : collecte logs Docker + syslog
- **Node Exporter** : métriques CPU, RAM, disque, réseau
- **cAdvisor** : métriques par container

### alerting
- Alertmanager avec routage par sévérité
- **Critical** -> Telegram + Email
- **Warning** -> Email uniquement
- Templates HTML pour les notifications
- Alertes configurées : CPU >80%, RAM >85%, disque <15%, container down, latence élevée, erreurs 5xx, certificat SSL expirant

## Dashboards Grafana

4 dashboards pré-configurés et provisionnés automatiquement :

| Dashboard | Panels | Description |
|-----------|--------|-------------|
| System - Node Exporter | 17 | CPU, RAM, disque, réseau, load average |
| Docker - cAdvisor | 11 | CPU/RAM/IO par container, tableau de statut |
| Traefik - Reverse Proxy | 15 | Requêtes/s, latence p50/p90/p99, codes HTTP |
| Logs - Loki | 8 | Volume de logs, recherche, filtres par container |

## Accès après déploiement

| Service | URL |
|---------|-----|
| Traefik Dashboard | `https://traefik.votre-domaine.fr` |
| Grafana | `https://grafana.votre-domaine.fr` |

## Sécurité

- SSH durci (port custom, clé uniquement, pas de root)
- UFW activé (whitelist ports)
- TLS 1.2+ avec HSTS
- CrowdSec IPS collaboratif
- Headers de sécurité sur toutes les réponses
- Rate limiting sur Traefik
- Secrets chiffrés avec Ansible Vault
- Socket Docker monté en lecture seule
