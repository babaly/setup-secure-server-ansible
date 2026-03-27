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
| Traefik | 3.3 | Reverse proxy + Let's Encrypt + docker-socket-proxy |
| CrowdSec | latest | IPS collaboratif (engine uniquement) |
| Prometheus | 2.53.0 | Collecte de métriques |
| Grafana | 11.0.0 | Dashboards et visualisation |
| Loki | 3.1.0 | Agrégation de logs (schema tsdb) |
| Promtail | 3.1.0 | Collecte de logs Docker + syslog |
| Node Exporter | 1.8.0 | Métriques système |
| cAdvisor | latest | Métriques containers Docker |
| Alertmanager | 0.27.0 | Routage des alertes |
| Docker CE | 27.5.1 | **Pinné** — Docker 29.x incompatible avec Traefik |
| CrowdSec Bouncer | plugin v1.4.2 | Plugin natif Traefik pour blocage actif des IPs |

## Prérequis

- **Machine locale** : Ansible 2.15+ (via WSL sur Windows)
- **Serveur cible** : Ubuntu 22.04 / 24.04 LTS, accès SSH root ou sudo
- **Compte Slack** : workspace avec droits de création d'application (pour les alertes)
- **Compte CrowdSec** : compte gratuit sur [app.crowdsec.net](https://app.crowdsec.net) (pour la console et le bouncer)

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

## Obtenir les clés et tokens externes

### Webhook Slack

Les alertes (Alertmanager + CrowdSec) sont envoyées via un **Incoming Webhook Slack**.

**Étapes pour créer le webhook :**

1. Aller sur [api.slack.com/apps](https://api.slack.com/apps) et cliquer **Create New App**
2. Choisir **From scratch**, donner un nom (ex: `Server Alerts`) et sélectionner votre workspace
3. Dans le menu de gauche, cliquer sur **Incoming Webhooks**
4. Activer le toggle **Activate Incoming Webhooks**
5. Cliquer **Add New Webhook to Workspace**
6. Sélectionner le channel Slack qui recevra les alertes (ex: `#alerts`) et cliquer **Allow**
7. Copier l'URL générée — elle ressemble à :

   ```text
   https://hooks.slack.com/services/<WORKSPACE_ID>/<CHANNEL_ID>/<TOKEN>
   ```

8. Coller cette URL dans le vault sous `vault_slack_webhook_url`

> Le même webhook est utilisé à la fois par Alertmanager (alertes système) et par CrowdSec (bans d'IP).

---

### Clé d'enrôlement CrowdSec (console)

La **clé d'enrôlement** permet de connecter votre instance CrowdSec à la console cloud [app.crowdsec.net](https://app.crowdsec.net) pour visualiser les événements et bénéficier de la threat intelligence collaborative.

**Étapes :**

1. Créer un compte gratuit sur [app.crowdsec.net](https://app.crowdsec.net)
2. Une fois connecté, aller dans **Instances** > **Add Instance**
3. La console affiche une commande du type :

   ```bash
   cscli console enroll <VOTRE_CLÉ>
   ```

4. Copier uniquement la clé (la partie après `enroll`)
5. La coller dans le vault sous `vault_crowdsec_enroll_key`

> Cette étape est optionnelle. Si vous ne souhaitez pas utiliser la console, laissez `vault_crowdsec_enroll_key` vide.

---

### Clé bouncer CrowdSec (plugin Traefik)

Le **bouncer** est le composant qui permet à Traefik de bloquer activement les IPs identifiées par CrowdSec. Il communique avec le moteur CrowdSec via une clé API locale.

**Étapes :**

1. Une fois CrowdSec déployé sur le serveur, se connecter en SSH :

   ```bash
   ssh -p 2222 deploy@votre-serveur.fr
   ```

2. Générer la clé bouncer :

   ```bash
   docker exec crowdsec cscli bouncers add traefik-bouncer
   ```

3. La commande retourne une clé API, par exemple :

   ```text
   Api key for 'traefik-bouncer':
   abc123def456ghi789jkl012mno345pqr678stu901
   ```

4. Copier cette clé et la coller dans le vault sous `vault_crowdsec_bouncer_key`

5. Mettre à jour le vault chiffré et relancer le déploiement Traefik :

   ```bash
   ansible-vault edit group_vars/all/vault.yml
   ansible-playbook site.yml --tags traefik --ask-vault-pass
   ```

> **Important** : cette clé doit être générée **après** le premier déploiement de CrowdSec. Elle ne peut pas être connue à l'avance.

---

### Mot de passe bcrypt (Traefik dashboard et Grafana)

Traefik et Grafana utilisent des mots de passe hashés en **bcrypt**. Pour générer le hash :

```bash
# Installer htpasswd si nécessaire
sudo apt install apache2-utils

# Générer le hash (remplacer "votre_mot_de_passe")
htpasswd -nbB admin "votre_mot_de_passe"
# Exemple de sortie : admin:$2y$05$abc123...
```

Copier uniquement la partie après `admin:` dans le vault sous `vault_traefik_dashboard_password` et `vault_grafana_admin_password`.

---

## Configuration

### 1. Configurer les secrets

Modifier `group_vars/all/vault.yml` avec vos valeurs :

```yaml
# Serveur
vault_server_ip: "X.X.X.X"
vault_base_domain: "votre-domaine.fr"
vault_acme_email: "admin@votre-domaine.fr"

# Connexion initiale (premier run)
vault_initial_user: "root"
vault_initial_ssh_port: 22

# Config cible (après hardening)
vault_new_user: "deploy"
vault_new_ssh_port: 2222

# Services
vault_traefik_dashboard_user: "admin"
vault_traefik_dashboard_password: "<hash_bcrypt>"
vault_crowdsec_enroll_key: "<cle_crowdsec_console>"
vault_crowdsec_bouncer_key: "<cle_bouncer>"
vault_grafana_admin_user: "admin"
vault_grafana_admin_password: "<hash_bcrypt>"
vault_slack_webhook_url: "https://hooks.slack.com/services/XXX/YYY/ZZZ"
vault_slack_channel: "#alerts"
vault_smtp_host: "smtp.votre-domaine.fr"
vault_smtp_port: 587
vault_smtp_user: "user@votre-domaine.fr"
vault_smtp_password: "<mot_de_passe_smtp>"
vault_alert_email_from: "noreply@votre-domaine.fr"
vault_alert_email_to: "admin@votre-domaine.fr"
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
- Plugin natif Traefik (`crowdsec-bouncer-traefik-plugin v1.4.2`) pour blocage actif
- Collections : traefik, http-cve, linux
- Enrôlement console CrowdSec (optionnel)
- Notifications Slack lors des bans d'IP (webhook configurable)

### monitoring
- **Prometheus** : scrape node-exporter, cAdvisor, Traefik (15s)
- **Grafana** : provisionnement automatique datasources + dashboards
- **Loki** : agrégation logs avec schema v13
- **Promtail** : collecte logs Docker + syslog
- **Node Exporter** : métriques CPU, RAM, disque, réseau
- **cAdvisor** : métriques par container

### alerting
- Alertmanager avec routage par sévérité
- **Critical** -> Slack + Email
- **Warning** -> Slack uniquement
- Templates HTML pour les notifications
- Alertes configurées : CPU >80%, RAM >85%, disque <15%, container down, latence élevée, erreurs 5xx, certificat SSL expirant
- **CrowdSec** -> notifications Slack dédiées lors des bans d'IP

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
- CrowdSec IPS collaboratif + plugin Traefik natif (blocage actif)
- Notifications Slack pour les bans d'IP (CrowdSec + Alertmanager)
- Headers de sécurité sur toutes les réponses
- Rate limiting sur Traefik
- Secrets chiffrés avec Ansible Vault
- Docker socket exposé via proxy read-only (docker-socket-proxy)
