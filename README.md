# cloud-playbook

Collection de playbooks Ansible pour le déploiement automatisé de VMs étudiantes sur un hyperviseur Proxmox, avec accès distant via Apache Guacamole.

## Vue d'ensemble

Chaque étudiant dispose d'une VM Linux et/ou Windows clonée depuis un template, configurée avec une IP statique, et accessible depuis un navigateur via SSH ou RDP grâce à Guacamole.

```
Ansible ──► Proxmox (nimbus14)
              ├── VM Linux  : 200<vmid>  (172.16.205.<vmid>)
              └── VM Windows: 300<vmid>  (172.16.205.<vmid>)
                                │
                          Guacamole (SSH / RDP)
```

## Prérequis

**Système de contrôle Ansible**

```bash
pip install -r requirements.txt
ansible-galaxy collection install -r collections/requirements.yml
```

**Collections requises**

| Collection | Usage |
|---|---|
| `community.proxmox` | Gestion des VMs Proxmox |
| `scicore.guacamole` | Gestion des connexions Guacamole |
| `community.docker` | Déploiement de conteneurs (Linux) |
| `chocolatey.chocolatey` | Installation de packages (Windows) |
| `ansible.windows` | Modules PowerShell/WinRM |

## Templates disponibles

| Clé `choix_os` | ID Template | VMID créé |
|---|---|---|
| `UbuntuServ` | 103 | `200<vmid>` |
| `Debian` | 101 | `200<vmid>` |
| `Win10` | 104 | `300<vmid>` |

## Playbooks

### `deploy.yml` — Déploiement complet

Clone le template, injecte l'IP via Cloud-Init, démarre la VM, et crée le compte + la connexion Guacamole.

```bash
ansible-playbook deploy.yml \
  -e vmid_etudiant=42 \
  -e choix_os=UbuntuServ \
  -e pve_api_host=<IP_PROXMOX> \
  -e pve_api_token_id=<TOKEN_ID> \
  -e pve_api_token_secret=<TOKEN_SECRET>
```

### `services.yml` — Installation de services

Installe Apache et/ou MariaDB sur une VM déjà démarrée. Sur Linux, les services tournent dans Docker ; sur Windows, ils sont installés via Chocolatey.

```bash
ansible-playbook services.yml \
  -e vmid_etudiant=42 \
  -e choix_os=UbuntuServ \
  -e "choix_services=['Apache','MariaDB']"
```

| Service | Linux | Windows | Port |
|---|---|---|---|
| Apache | Docker (`httpd:latest`) | Chocolatey (`apache-httpd`) | 80 |
| MariaDB | Docker (`mariadb:latest`) | Chocolatey (`mariadb`) | 3306 |

### `powerOn.yml` / `powerOff.yml` — Self-service

Démarrage ou arrêt d'une VM sans la supprimer.

```bash
ansible-playbook powerOn.yml \
  -e vmid_etudiant=42 \
  -e choix_os=UbuntuServ \
  -e pve_api_host=<IP_PROXMOX> \
  -e pve_api_token_id=<TOKEN_ID> \
  -e pve_api_token_secret=<TOKEN_SECRET>
```

### `exam_mode.yml` — Mode examen / Proxy tinyproxy

Active ou désactive le proxy tinyproxy sur la VM proxy selon le mode examen.

- **Mode examen activé** (`exam_mode=true`) : tinyproxy est arrêté, aucun accès internet depuis les VMs étudiantes.
- **Mode normal** (`exam_mode=false`) : tinyproxy est démarré et applique la whitelist définie dans `proxy_whitelist.txt`.

Le sous-réseau interne `172.16.0.0/16` est toujours joignable sans passer par le proxy (`NO_PROXY`), les VMs étudiantes peuvent communiquer entre elles dans les deux modes.

```bash
# Activer le mode examen (couper internet)
ansible-playbook exam_mode.yml \
  -e exam_mode=true \
  -e vmid_proxy=99

# Revenir en mode normal (proxy + whitelist)
ansible-playbook exam_mode.yml \
  -e exam_mode=false \
  -e vmid_proxy=99

# Mode normal + configurer le proxy sur les VMs étudiantes 01 à 03
ansible-playbook exam_mode.yml \
  -e exam_mode=false \
  -e vmid_proxy=99 \
  -e '{"vmid_list":["01","02","03"]}'
```

**Modifier la whitelist** : éditer `proxy_whitelist.txt` (une regex par ligne, `#` pour commenter), puis relancer le playbook en mode normal.

| Variable | Description | Exemple |
|---|---|---|
| `exam_mode` | `true` = proxy arrêté, `false` = proxy actif | `true` |
| `vmid_proxy` | VMID de la VM proxy | `99` |
| `vmid_list` | (optionnel) Liste des vmid étudiants à configurer | `["01","02"]` |
| `proxy_port` | Port tinyproxy (défaut : 8888) | `8888` |

### `suppression.yml` — Suppression complète

Éteint et supprime la VM, puis retire la connexion Guacamole associée.

```bash
ansible-playbook suppression.yml \
  -e vmid_etudiant=42 \
  -e choix_os=UbuntuServ \
  -e pve_api_host=<IP_PROXMOX> \
  -e pve_api_token_id=<TOKEN_ID> \
  -e pve_api_token_secret=<TOKEN_SECRET> \
  -e guac_host=<IP_GUACAMOLE> \
  -e guac_admin=guacadmin \
  -e guac_pass=guacadmin
```

## Variables

| Variable | Description | Exemple |
|---|---|---|
| `vmid_etudiant` | Identifiant étudiant (2 chiffres) | `42` |
| `choix_os` | OS à déployer | `UbuntuServ`, `Debian`, `Win10` |
| `choix_services` | Services à installer | `['Apache', 'MariaDB']` |
| `pve_api_host` | Adresse IP de l'API Proxmox | `192.168.1.10` |
| `pve_api_token_id` | Nom du token API Proxmox | `ansible` |
| `pve_api_token_secret` | Secret du token API Proxmox | `xxxxxxxx-...` |
| `guac_host` | Adresse IP de Guacamole | `172.16.184.235` |
| `guac_admin` | Compte admin Guacamole | `guacadmin` |
| `guac_pass` | Mot de passe admin Guacamole | `guacadmin` |

## Structure du projet

```
cloud-playbook/
├── deploy.yml              # Orchestrateur principal
├── exam_mode.yml           # Mode examen — activation/désactivation du proxy
├── services.yml            # Installation des services applicatifs
├── powerOn.yml             # Démarrage self-service
├── powerOff.yml            # Arrêt self-service
├── suppression.yml         # Suppression complète
├── proxy_whitelist.txt     # Whitelist tinyproxy (bloc-notes des sites autorisés)
├── requirements.txt        # Dépendances Python (proxmoxer)
├── collections/
│   └── requirements.yml    # Collections Ansible requises
├── tasks/
│   ├── create_linux.yml    # Clone + démarrage VM Linux
│   ├── create_windows.yml  # Clone + démarrage VM Windows
│   ├── guac_linux.yml      # Connexion SSH dans Guacamole
│   └── guac_windows.yml    # Connexion RDP dans Guacamole
└── templates/
    ├── tinyproxy.conf.j2   # Config tinyproxy (whitelist + Allow sous-réseau)
    └── tinyproxy_filter.j2 # Fichier de filtres généré depuis proxy_whitelist.txt
```

## Accès étudiant

Après déploiement, l'étudiant `<vmid>` peut se connecter sur Guacamole avec :

- **URL** : `http://<IP_GUACAMOLE>:8080/guacamole`
- **Login** : `etu_<vmid>`
- **Mot de passe** : `etu`

Les connexions disponibles sont `SSH_Linux_<vmid>` (Linux) et `RDP_Windows_<vmid>` (Windows) selon l'OS déployé.
