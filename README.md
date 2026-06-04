# DevOps - Docker, GitHub Actions & Ansible

> TP/TD DevOps - Docker, GitHub Actions, Ansible  
> Auteur : EnzoAUGIE  
> École : EPF  

---

## 📋 Table des matières

- [Architecture](#architecture)
- [TP1 - Docker](#tp1---docker)
- [TP2 - GitHub Actions](#tp2---github-actions)
- [TP3 - Ansible](#tp3---ansible)
- [Structure du projet](#structure-du-projet)
- [Prérequis](#prérequis)
- [Déploiement](#déploiement)

---

## Architecture

```
Internet
    │
    ▼ port 80
┌─────────────────────────────────┐
│ container "httpd" (Apache)      │
│ /api/ → backend:8080            │
│ /     → front:80                │
└─────────────────────────────────┘
    │ front-network
    ▼
┌─────────────────────────────────┐
│ container "front" (Vue.js/Nginx)│
│ Interface utilisateur           │
└─────────────────────────────────┘
    │ front-network
    ▼
┌─────────────────────────────────┐
│ container "backend" (Spring Boot│
│ API REST - port 8080            │
└─────────────────────────────────┘
    │ back-network
    ▼
┌─────────────────────────────────┐
│ container "database" (PostgreSQL│
│ port 5432                       │
└─────────────────────────────────┘
```

---

## TP1 - Docker

### Objectif
Containeriser une application 3-tiers (PostgreSQL + Spring Boot + Apache) avec Docker et Docker Compose.

### Contenu
- **`TP1-Docker/database/`** : Image PostgreSQL avec scripts d'initialisation SQL
- **`TP1-Docker/backend/simpleapi/`** : Application Spring Boot (API REST)
- **`TP1-Docker/httpd/`** : Reverse proxy Apache httpd
- **`TP1-Docker/docker-compose.yml`** : Orchestration des 3 services

### Réseaux Docker
- `front-network` : httpd ↔ backend
- `back-network` : backend ↔ database

### Variables d'environnement
Créer un fichier `.env` dans `TP1-Docker/` :
```env
POSTGRES_DB=db
POSTGRES_USER=usr
POSTGRES_PASSWORD=pwd
```

### Lancer l'application localement
```bash
cd TP1-Docker
docker-compose up -d
```

L'API sera accessible sur `http://localhost/departments/IRC/students`

### Images Docker Hub
| Service   | Image                              |
|-----------|------------------------------------|
| Backend   | `enzoaugie/tp-devops-backend:latest`  |
| Database  | `enzoaugie/tp-devops-database:latest` |
| Httpd     | `enzoaugie/tp-devops-httpd:latest`    |
| Frontend  | `enzoaugie/tp-devops-front:latest`    |

---

## TP2 - GitHub Actions

### Objectif
Mettre en place une pipeline CI/CD complète avec tests automatiques, analyse de qualité et déploiement des images Docker.

### Pipelines

#### `test.yml` — Tests Backend
**Déclenchement** : Push sur `main` et `develop`, Pull Requests

```
Push/PR
   │
   ▼
test-backend (mvn clean verify)
   │
   ▼
sonarcloud (analyse qualité)
```

#### `deploy.yml` — Build & Push Docker Images
**Déclenchement** : Automatiquement après `test.yml` vert sur `main`

```
test.yml 
   │
   ▼
build-and-push-backend  ──┐
build-and-push-database ──┤── Docker Hub
build-and-push-httpd    ──┤
build-and-push-front    ──┘
```

#### `ansible.yml` — Déploiement Ansible
**Déclenchement** : Automatiquement après `deploy.yml` vert sur `main` OU manuellement

```
deploy.yml 
   │
   ▼
deploy (Ansible playbook)
   │
   ▼
Serveur enzo.augie.takima.school
```

#### `main.yml` — Désactivé
Ancien pipeline conservé pour référence, déclenché uniquement manuellement.

### Secrets GitHub requis
| Secret              | Description                          |
|---------------------|--------------------------------------|
| `DOCKERHUB_USERNAME`| Nom d'utilisateur Docker Hub         |
| `DOCKERHUB_TOKEN`   | Token Docker Hub                     |
| `SONAR_TOKEN`       | Token SonarCloud                     |
| `SSH_PRIVATE_KEY`   | Clé SSH privée (format RSA)          |
| `SSH_HOST`          | Hostname du serveur distant          |

### Protection de la branche `main`
-  Pull Request obligatoire avant merge
-  `test-backend` requis avant merge
-  Force push bloqué

### SonarCloud
- Organisation : `enzoaugie`
- Project Key : `EnzoAUGIE_DevOps`
- URL : https://sonarcloud.io

### Versioning des images
Chaque image est taguée avec :
- `latest` → dernière version
- `<SHA_commit>` → version spécifique

---

## TP3 - Ansible

### Objectif
Automatiser le déploiement de l'application sur un serveur distant AWS avec Ansible et des roles.

### Prérequis
```bash
brew install ansible
```

### Inventaire
Le fichier `ansible/inventories/setup.yml` contient la configuration du serveur :
```yaml
all:
  vars:
    ansible_user: admin
    ansible_ssh_private_key_file: ~/.ssh/id_rsa_takima
    postgres_db: db
    postgres_user: usr
    postgres_password: pwd
  children:
    prod:
      hosts: enzo.augie.takima.school
```

### Roles

| Role       | Description                              |
|------------|------------------------------------------|
| `docker`   | Installation de Docker sur le serveur    |
| `network`  | Création des réseaux Docker              |
| `database` | Lancement du container PostgreSQL        |
| `app`      | Lancement du container Spring Boot       |
| `front`    | Lancement du container Vue.js            |
| `proxy`    | Lancement du container Apache httpd      |

### Structure des roles
```
ansible/
├── inventories/
│   └── setup.yml
├── playbook.yml
└── roles/
    ├── docker/tasks/main.yml
    ├── network/tasks/main.yml
    ├── database/tasks/main.yml
    ├── app/tasks/main.yml
    ├── front/tasks/main.yml
    └── proxy/tasks/main.yml
```

### Lancer le playbook manuellement
```bash
cd ansible
ansible-playbook -i inventories/setup.yml playbook.yml
```

### Optimisation future
Dans chaque role, décommenter `pull: true` pour forcer le téléchargement de la dernière image Docker à chaque déploiement :
```yaml
# pull: true
# Décommenter pour forcer le téléchargement de la dernière image depuis Docker Hub
# à chaque déploiement, évitant ainsi de supprimer manuellement l'ancienne image
```

---

## Structure du projet

```
DevOps/
├── .github/
│   └── workflows/
│       ├── test.yml          # CI - Tests backend + SonarCloud
│       ├── deploy.yml        # CD - Build et push images Docker
│       ├── ansible.yml       # CD - Déploiement Ansible
│       └── main.yml          # Désactivé (référence)
├── TD1-Docker/
│   └── flask-app/            # App Flask TD1
├── TP1-Docker/
│   ├── backend/simpleapi/    # Spring Boot API
│   ├── database/             # PostgreSQL + scripts SQL
│   ├── httpd/                # Apache reverse proxy
│   ├── docker-compose.yml
│   └── questions.md
├── ansible/
│   ├── inventories/
│   │   └── setup.yml
│   ├── playbook.yml
│   └── roles/
│       ├── docker/
│       ├── network/
│       ├── database/
│       ├── app/
│       ├── front/
│       └── proxy/
└── README.md
```

---

## Prérequis

- Docker Desktop
- Docker Compose
- Ansible (`brew install ansible`)
- Clé SSH pour le serveur (`~/.ssh/id_rsa_takima`)

---

## Déploiement

### Local
```bash
cd TP1-Docker
docker-compose up -d
```

### Production (via GitHub Actions)
1. Push sur `develop`
2. Créer une Pull Request vers `main`
3. Les tests se lancent automatiquement
4. Merger la PR une fois les tests verts
5. Les images Docker sont buildées et poussées automatiquement
6. Ansible déploie automatiquement sur le serveur

### Production (manuel)
```bash
# Lancer le playbook Ansible
cd ansible
ansible-playbook -i inventories/setup.yml playbook.yml --private-key=~/.ssh/id_rsa_takima -u admin
```

### URLs de production
| Service  | URL                                                      |
|----------|----------------------------------------------------------|
| Frontend | http://enzo.augie.takima.school                          |
| API      | http://enzo.augie.takima.school/api/departments          |
| Étudiants IRC | http://enzo.augie.takima.school/api/departments/IRC/students |

---

## Liens utiles

- [Docker Hub](https://hub.docker.com/u/enzoaugie)
- [GitHub Repository](https://github.com/EnzoAUGIE/DevOps)
- [SonarCloud](https://sonarcloud.io/project/overview?id=EnzoAUGIE_DevOps)
- [Serveur de production](http://enzo.augie.takima.school)
