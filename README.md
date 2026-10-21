# OCI CI/CD Docker Deployment

[![CI/CD](https://github.com/Fahdkrachel/oci-cicd-docker-deployment/actions/workflows/deploy.yml/badge.svg)](https://github.com/Fahdkrachel/oci-cicd-docker-deployment/actions/workflows/deploy.yml)
![Docker](https://img.shields.io/badge/Docker-nginx%3Aalpine-2496ED?logo=docker&logoColor=white)
![GitHub Actions](https://img.shields.io/badge/GitHub%20Actions-self--hosted%20runner-2088FF?logo=githubactions&logoColor=white)
![Ubuntu](https://img.shields.io/badge/Ubuntu-24.04-E95420?logo=ubuntu&logoColor=white)

Déploiement automatisé d'une application web conteneurisée sur une machine virtuelle cloud. À chaque `git push` sur la branche `main`, une image Docker est construite, publiée dans un registre local, puis déployée sans intervention manuelle.

Projet réalisé dans le cadre du TP3 du module *Ingénierie des Infrastructures Big Data et Cloud* (UH2C / ENSET, 2026/2027).

## Architecture

![Architecture globale](docs/architecture.png)

```
Développeur ──git push──▶ GitHub ──déclenche──▶ Runner (sur la VM)
                                                    │
                                          build ◀───┘
                                            │
                                  Registre local (localhost:5000)
                                            │
                                  Conteneur webapp (nginx, port 80) ──▶ Utilisateur
```

| Composant | Rôle |
|---|---|
| GitHub | Code source, Dockerfile et workflow |
| Runner GitHub Actions | Service installé sur la VM, exécute les jobs `build` et `deploy` |
| Docker | Construction de l'image et exécution du conteneur |
| Registre local (`registry:2`) | Stockage des images, accessible uniquement depuis la VM |
| Conteneur `webapp` | Application servie par nginx sur le port 80 |

## Fonctionnalités

- Construction d'une image Docker à partir du `Dockerfile`
- Publication de l'image dans un registre local avec deux tags : le hash du commit et `latest`
- Déploiement automatique : remplacement de l'ancien conteneur par le nouveau
- Déclenchement à chaque push sur `main`, ou manuellement depuis l'onglet Actions
- Redémarrage automatique du conteneur et du registre avec la machine
- Supervision des ressources avec `docker stats` et `htop`

## Équivalences avec le sujet Oracle Cloud

Le sujet du TP prévoit des services Oracle Cloud Infrastructure. La machine utilisée est fournie par une plateforme pédagogique, sans accès à la console OCI : chaque étape a donc été réalisée avec un outil équivalent installé sur la VM.

| Sujet (OCI) | Réalisation |
|---|---|
| Instance Compute Ubuntu | VM Ubuntu 24.04 fournie par l'école |
| Ouverture des ports 80 et 443 | Gestion des ports depuis la plateforme de l'école |
| OCI DevOps (build et déploiement) | Workflow GitHub Actions avec runner auto-hébergé |
| Oracle Container Registry | Registre Docker local |
| Déclencheur Git | Événement `push` sur `main` |
| OCI Monitoring | `docker stats` et `htop` |

## Structure du dépôt

```
.
├── .github/
│   └── workflows/
│       └── deploy.yml     # Pipeline CI/CD
├── docs/
│   └── architecture.png   # Schéma d'architecture
├── Dockerfile             # Image nginx servant la page
├── index.html             # Page web de l'application
└── README.md
```

## Prérequis

- Une machine virtuelle Ubuntu accessible en SSH, avec le port 80 ouvert
- Un compte GitHub et un dépôt
- Docker installé sur la VM

## Installation

### 1. Se connecter à la VM

```bash
ssh-keygen -t ed25519            # si aucune clé n'existe sur le poste
ssh ubuntu@<ADRESSE_IP>
```

La clé publique (`~/.ssh/id_ed25519.pub`) doit être enregistrée sur la plateforme de la machine.

### 2. Installer Docker

```bash
sudo apt update
sudo apt install -y docker.io
sudo systemctl enable --now docker
sudo usermod -aG docker $USER    # puis se reconnecter
docker --version
```

### 3. Lancer le registre local

```bash
docker run -d --restart=always --name registry -p 127.0.0.1:5000:5000 registry:2
curl http://localhost:5000/v2/_catalog
```

Le registre n'écoute que sur `127.0.0.1` : il n'est pas exposé sur Internet.

### 4. Installer le runner GitHub Actions

Dans le dépôt : **Settings > Actions > Runners > New self-hosted runner** (Linux, x64). Exécuter sur la VM les commandes affichées par GitHub, puis installer le runner comme service :

```bash
cd ~/actions-runner
sudo ./svc.sh install ubuntu
sudo ./svc.sh start
sudo ./svc.sh status
```

Le runner doit apparaître en **Idle** dans les paramètres du dépôt.

## Pipeline CI/CD

Le workflow `.github/workflows/deploy.yml` comporte deux jobs exécutés sur le runner de la VM.

| Job | Étapes |
|---|---|
| `build` | Récupération du code, `docker build`, `docker push` vers `localhost:5000` |
| `deploy` | `docker pull`, suppression de l'ancien conteneur, `docker run -p 80:80` |

```yaml
on:
  push:
    branches: [main]
  workflow_dispatch:
```

## Utilisation

1. Modifier un fichier du dépôt, par exemple `index.html`.
2. Valider et pousser sur `main` :
   ```bash
   git add .
   git commit -m "Mise à jour de la page"
   git push
   ```
3. Suivre l'exécution dans l'onglet **Actions**. Quand les deux jobs sont verts, l'application est à jour.
4. Ouvrir `http://<ADRESSE_IP>` (recharger avec Ctrl + F5 pour éviter le cache).

Le workflow peut aussi être lancé à la main : **Actions > TP3-FAHD-KRACHEL > Run workflow**.

## Vérification et monitoring

```bash
docker ps                                  # conteneurs webapp et registry
curl http://localhost:5000/v2/_catalog     # images du registre
curl http://localhost                      # réponse de l'application
docker stats                               # CPU, mémoire et réseau du conteneur
```

## Dépannage

| Symptôme | Piste |
|---|---|
| `Permission denied (publickey)` | La clé publique enregistrée sur la plateforme ne correspond pas à la clé du poste |
| `permission denied` sur `docker.sock` | Se déconnecter puis se reconnecter après `usermod -aG docker` |
| Exécution bloquée sur « Queued » | Le runner est hors ligne : `sudo ./svc.sh status` puis `start` |
| `Dockerfile: no such file or directory` | Le `Dockerfile` doit être à la racine du dépôt |
| La page ne change pas après un déploiement | Recharger avec Ctrl + F5 et vérifier `docker ps` |

## Limites

- Le pipeline ne s'exécute que si la VM est en marche : le runner est hors ligne quand la machine est arrêtée.
- Une régénération de la machine efface le runner, le registre et les images.
- Le registre local n'est pas authentifié : il convient à un usage pédagogique, pas à la production.
- Sur un dépôt public, éviter d'exécuter sur le runner les workflows de pull requests de tiers.

## Pistes d'amélioration

- Tests automatisés dans le pipeline avant le déploiement
- HTTPS avec un certificat sur le port 443
- Authentification du registre d'images
- Supervision graphique avec Netdata, ou Prometheus et Grafana
- Description de l'infrastructure avec Terraform

## Auteur

**Fahd Krachel**, étudiant à l'ENSET (UH2C).
Encadrant : Pr. Kamal EL GUEMMAT.
