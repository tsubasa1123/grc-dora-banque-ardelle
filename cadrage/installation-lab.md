# Installation du lab CISO Assistant

## Environnement

| Élément | Choix |
| --- | --- |
| Hyperviseur | VMware Workstation |
| Système | Ubuntu 24.04 LTS |
| Ressources | 2 vCPU, 4 Go de RAM (6 Go recommandés), disque de 60 Go |
| Outil | [CISO Assistant Community Edition](https://github.com/intuitem/ciso-assistant-community) (AGPL v3), images Docker préconstruites |

## Installation

```bash
# Docker
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker $USER      # puis se déconnecter et se reconnecter

# CISO Assistant
git clone --single-branch -b main https://github.com/intuitem/ciso-assistant-community.git
cd ciso-assistant-community
./docker-compose.sh                # crée la base et le compte administrateur
```

Accès : `https://localhost:8443`

## Exploitation courante

```bash
cd ~/ciso-assistant-community
docker compose up -d --pull never  # démarrer sans retélécharger les images
docker compose ps                  # attendre que le backend soit « healthy »

# Sauvegarde en fin de séance
docker compose stop
sudo cp -a db ~/sauvegarde-db-$(date +%F)
docker compose up -d --pull never
```
