<div align="center">

# Inception

**Infrastructure conteneurisée multi-services orchestrée par Docker Compose**

*Projet du cursus 42 — System Administration*

![Docker](https://img.shields.io/badge/Docker-2496ED?style=flat&logo=docker&logoColor=white)
![Alpine](https://img.shields.io/badge/Alpine_Linux-0D597F?style=flat&logo=alpinelinux&logoColor=white)
![NGINX](https://img.shields.io/badge/NGINX-009639?style=flat&logo=nginx&logoColor=white)
![WordPress](https://img.shields.io/badge/WordPress-21759B?style=flat&logo=wordpress&logoColor=white)
![MariaDB](https://img.shields.io/badge/MariaDB-003545?style=flat&logo=mariadb&logoColor=white)
![Redis](https://img.shields.io/badge/Redis-DC382D?style=flat&logo=redis&logoColor=white)
![License](https://img.shields.io/badge/License-42_School-000000?style=flat)

</div>

---

## Sommaire

1. [À propos](#à-propos)
2. [Architecture](#architecture)
3. [Stack technique](#stack-technique)
4. [Arborescence](#arborescence)
5. [Prérequis](#prérequis)
6. [Installation](#installation)
7. [Utilisation](#utilisation)
8. [Accès aux services](#accès-aux-services)
9. [Configuration](#configuration)
10. [Sécurité](#sécurité)
11. [Cibles Makefile](#cibles-makefile)
12. [Bonnes pratiques respectées](#bonnes-pratiques-respectées)
13. [Auteur](#auteur)

---

## À propos

**Inception** est un projet du cursus 42 dont l'objectif est de mettre en place une infrastructure web complète, **entièrement conteneurisée**, en respectant un ensemble strict de contraintes système :

- Chaque service tourne dans son **propre conteneur dédié**.
- Tous les conteneurs sont construits à partir de **Dockerfiles personnalisés** (aucune image préfabriquée du Docker Hub, hormis les images de base Alpine).
- Les conteneurs sont **orchestrés via Docker Compose**.
- L'accès au site se fait **exclusivement via TLS (HTTPS)** sur le port `443`.
- Les **mots de passe et identifiants** sont gérés via le mécanisme **Docker Secrets** et un fichier `.env`, jamais codés en dur.
- Aucune commande de type `tail -f`, `sleep infinity` ou hack contournant le PID 1 n'est utilisée : chaque conteneur tourne sur son **processus principal en avant-plan**.

Le projet inclut la partie obligatoire (NGINX, WordPress, MariaDB) ainsi que **l'intégralité des bonus**.

---

## Architecture

```
                                  Host (Linux)
                                       │
                                       │  ports : 443, 8080, 8025, 3000, 21, 21000-21010
                                       ▼
              ┌────────────────────────────────────────────────────┐
              │                       NGINX                        │
              │           Reverse-proxy TLS 1.2 / 1.3              │
              └─────┬──────────────┬──────────────┬───────────────┘
                    │              │              │
       ┌────────────┴───┐  ┌───────┴────┐  ┌──────┴──────┐
       │   WordPress    │  │  Adminer   │  │   Mailpit   │
       │   (php-fpm)    │  │            │  │             │
       └────────┬───────┘  └─────┬──────┘  └─────────────┘
                │                │
       ┌────────┴────────────────┴─────────┐
       │             MariaDB               │
       └───────────────────────────────────┘

       ┌──────────┐     ┌──────────┐     ┌────────────┐
       │  Redis   │     │   FTP    │     │  Static    │
       │ (cache)  │     │ (vsftpd) │     │  Website   │
       └──────────┘     └──────────┘     └────────────┘

       Réseaux Docker :
         • frontend_net  → nginx, wordpress, adminer, mailpit, ftp, static
         • backend_net   → mariadb, wordpress, redis, adminer

       Volumes (bind mounts vers ~/data) :
         • mariadb-data   → /var/lib/mysql
         • wordpress-data → /var/www/html
```

---

## Stack technique

| Service       | Rôle                                                | Image de base | Port exposé      | Réseau                       |
|---------------|-----------------------------------------------------|---------------|------------------|------------------------------|
| **nginx**     | Reverse-proxy TLS unique point d'entrée HTTPS       | `alpine:3.22` | `443`            | `frontend_net`               |
| **wordpress** | CMS PHP-FPM 8.3 + WP-CLI                            | `alpine:3.22` | `9000` *(int.)*  | `frontend_net`, `backend_net`|
| **mariadb**   | Base de données relationnelle                       | `alpine:3.22` | `3306` *(int.)*  | `backend_net`                |
| **redis**     | *(bonus)* Cache objet pour WordPress                | `alpine:3.22` | `6379` *(int.)*  | `backend_net`                |
| **ftp**       | *(bonus)* Serveur FTP (vsftpd) pointant sur WP      | `alpine:3.22` | `21`, `21000-21010` | `frontend_net`            |
| **adminer**   | *(bonus)* Interface web d'administration MariaDB    | `alpine:3.22` | `8080`           | `frontend_net`, `backend_net`|
| **mailpit**   | *(bonus)* Catcher SMTP pour WordPress               | `alpine:3.22` | `8025`, `1025`   | `frontend_net`               |
| **static**    | *(bonus)* Site statique de présentation             | `alpine:3.22` | `3000`           | `frontend_net`               |

---

## Arborescence

```
Inception_42/
├── Makefile                       # Orchestration : build / up / down / fclean / re
├── README.md                      # Ce fichier
├── secrets/                       # Mots de passe (montés via Docker Secrets)
│   ├── credentials.txt
│   ├── db_password.txt
│   ├── db_root_password.txt
│   ├── ftp_password.txt
│   └── second_password.txt
└── srcs/
    ├── .env                       # Variables d'environnement non-sensibles
    ├── docker-compose.yml         # Définition de la stack
    └── requirements/
        ├── mariadb/
        │   ├── Dockerfile
        │   ├── conf/50-server.cnf
        │   └── tools/mariadb-init.sh
        ├── nginx/
        │   ├── Dockerfile
        │   ├── conf/default.conf
        │   └── tools/generate_cert.sh
        ├── wordpress/
        │   ├── Dockerfile
        │   └── tools/wp-setup.sh
        └── bonus/
            ├── adminer/Dockerfile
            ├── ftp/
            │   ├── Dockerfile
            │   ├── conf/vsftpd.conf
            │   └── tools/ftp-entrypoint.sh
            ├── mailpit/Dockerfile
            ├── redis/
            │   ├── Dockerfile
            │   ├── conf/redis.conf
            │   └── tools/redis-entrypoint.sh
            └── static/
                ├── Dockerfile
                ├── default.conf
                └── website/{index.html,style.css}
```

---

## Prérequis

| Outil           | Version recommandée |
|-----------------|---------------------|
| Docker Engine   | ≥ `24.0`            |
| Docker Compose  | ≥ `v2.20` (plugin)  |
| GNU Make        | ≥ `4.0`             |
| Système hôte    | Linux x86_64        |

Avant la première exécution, ajouter le domaine personnalisé au fichier `/etc/hosts` :

```bash
echo "127.0.0.1 throbert.42.fr" | sudo tee -a /etc/hosts
```

> **Note :** adapter le chemin des bind mounts dans `srcs/docker-compose.yml`
> (clé `device:` des volumes `mariadb-data` et `wordpress-data`) pour qu'il
> corresponde au répertoire `~/data` de votre utilisateur.

---

## Installation

```bash
git clone <url-du-dépôt> Inception
cd Inception
make
```

La cible par défaut enchaîne :

1. **`dirs`** — création des points de montage `~/data/mariadb` et `~/data/wordpress` sur l'hôte.
2. **`build`** — construction de toutes les images à partir des Dockerfiles locaux.
3. **`up`** — démarrage de la stack en avant-plan (logs visibles).

---

## Utilisation

| Action                                  | Commande           |
|-----------------------------------------|--------------------|
| Démarrer la stack                       | `make`             |
| Reconstruire les images                 | `make build`       |
| Démarrer les conteneurs                 | `make up`          |
| Arrêter et supprimer les conteneurs     | `make down`        |
| Nettoyage complet (images + volumes)    | `make fclean`      |
| Reset total puis redémarrage            | `make re`          |

---

## Accès aux services

Après démarrage, les services sont accessibles aux URL suivantes :

| Service           | URL                                  | Identifiants                                           |
|-------------------|--------------------------------------|--------------------------------------------------------|
| **WordPress**     | <https://throbert.42.fr>             | Admin : `superboss` / *(cf. `secrets/credentials.txt`)*|
| **Adminer**       | <https://throbert.42.fr:8080>        | Serveur : `mariadb`, user : `wp_user`                  |
| **Mailpit (UI)**  | <https://throbert.42.fr:8025>        | —                                                      |
| **Site statique** | <https://throbert.42.fr:3000>        | —                                                      |
| **FTP**           | `ftp://throbert.42.fr:21`            | `ftpuser` / *(cf. `secrets/ftp_password.txt`)*         |

> Le certificat TLS étant **auto-signé**, le navigateur affichera un avertissement à la première connexion : c'est attendu.

---

## Configuration

### Variables d'environnement (`srcs/.env`)

Toutes les variables **non-sensibles** sont centralisées dans `srcs/.env` :

```env
DOMAIN_NAME=throbert.42.fr

MYSQL_DATABASE=wordpress
MYSQL_USER=wp_user
MYSQL_PASSWORD_FILE=/run/secrets/db_password
MYSQL_ROOT_PASSWORD_FILE=/run/secrets/db_root_password

WP_ADMIN_USER=superboss
WP_ADMIN_PASSWORD_FILE=/run/secrets/credentials
WP_ADMIN_EMAIL=admin@throbert.42.fr

WP_SECOND_USER=visitor
WP_SECOND_EMAIL=visitor@throbert.42.fr
WP_SECOND_PASSWORD_FILE=/run/secrets/second_password

FTP_USER=ftpuser
FTP_PASSWORD_FILE=/run/secrets/ftp_password
```

### Secrets Docker (`secrets/*.txt`)

Les mots de passe sont **isolés** dans le répertoire `secrets/` et injectés dans les conteneurs sous `/run/secrets/<nom>` au runtime, sans jamais être stockés dans les couches d'image.

---

## Sécurité

- **TLS uniquement** (TLSv1.2 / TLSv1.3) — aucune autre version autorisée. Pas de port `80`, aucune redirection `HTTP`.
- **Mots de passe externalisés** via Docker Secrets, jamais en clair dans les Dockerfiles ni dans `docker-compose.yml`.
- **Isolation réseau** entre `frontend_net` (services exposés) et `backend_net` (base de données / cache).
- **Aucun conteneur root inutile** : MariaDB tourne sous `mysql`, php-fpm sous `www-data`.
- **Volumes nommés bind-mountés** sur l'hôte pour la persistance et la lisibilité.

> ### Avant mise en production
> 1. Retirer `srcs/.env` et le répertoire `secrets/` du dépôt distant.
> 2. Vérifier la présence de `.gitignore` (cf. modèle fourni dans le projet).
> 3. Régénérer **tous** les mots de passe.

---

## Cibles Makefile

| Cible       | Description                                                                            |
|-------------|----------------------------------------------------------------------------------------|
| `all`       | (défaut) `build` + `up`.                                                               |
| `dirs`      | Crée les points de montage `~/data/mariadb` et `~/data/wordpress`.                     |
| `build`     | Construit toutes les images définies dans `docker-compose.yml`.                        |
| `up`        | Démarre la stack en avant-plan.                                                        |
| `down`      | Arrête les conteneurs et supprime les volumes nommés.                                  |
| `fclean`    | `down` + `docker system prune -af` + suppression des bind mounts hôte.                 |
| `re`        | `fclean` + `build` + `up`.                                                             |

---

## Bonnes pratiques respectées

- [x] Un seul service par conteneur (un seul processus en PID 1).
- [x] Dockerfiles personnalisés pour **chaque** service, basés sur Alpine.
- [x] Aucune image préfabriquée du Docker Hub (hormis l'image de base).
- [x] Aucune commande de type `tail -f`, `sleep infinity`, `bash` infini.
- [x] Politique de redémarrage automatique (`restart: always`) sur chaque service.
- [x] Communication inter-conteneurs via les noms de service Docker (pas d'IP en dur).
- [x] Certificat TLS auto-généré à la construction de l'image NGINX.
- [x] Initialisation WordPress idempotente via WP-CLI.
- [x] Variables d'environnement et secrets séparés.

---

## Auteur

**throbert** — étudiant à l'École 42

> Projet réalisé dans le cadre du cursus *Common Core* — branche *System Administration*.
