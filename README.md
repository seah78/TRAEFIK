# TRAEFIK

name: CI/CD Deployment for Traefik  # Nom du workflow GitHub Actions, il indique ce que fait ce workflow (déployer Traefik via CI/CD)

on:  # Section qui définit les déclencheurs d'exécution du workflow
  push:  # Déclenche le workflow à chaque fois qu'un "push" est effectué
    branches:  # Spécifie les branches pour lesquelles le workflow est déclenché
      - main  # Le workflow se déclenchera uniquement lorsqu'un push est effectué sur la branche "main"

jobs:  # Section contenant les différents jobs du workflow
  deploy:  # Nom du job (ici, le déploiement de Traefik)
    runs-on: ubuntu-latest  # Indique que le job sera exécuté sur une machine virtuelle Ubuntu avec la dernière version disponible

    steps:  # Liste des étapes à suivre dans ce job
      - name: Checkout code  # Nom de l'étape pour cloner le dépôt
        uses: actions/checkout@v3  # Utilise l'action officielle de GitHub pour récupérer le code du dépôt

      - name: Deploy Traefik via SSH on VPS  # Nom de l'étape de déploiement de Traefik via SSH sur le VPS
        uses: appleboy/ssh-action@v0.1.8  # Utilise l'action SSH de GitHub pour exécuter des commandes sur le VPS
        with:  # Les paramètres nécessaires pour l'action SSH
          host: ${{ secrets.HOST }}  # L'adresse IP ou le nom d'hôte du VPS, récupéré des secrets GitHub
          port: ${{ secrets.SSH_PORT }}  # Le port SSH du VPS (ex. 22 ou un port personnalisé)
          username: ${{ secrets.SSH_USERNAME }}  # Nom d'utilisateur pour se connecter au VPS, récupéré des secrets GitHub
          key: ${{ secrets.SSH_PRIVATE_KEY }}  # Clé privée SSH utilisée pour l'authentification, récupérée des secrets GitHub
          envs: ACME_EMAIL=${{ secrets.ACME_EMAIL }}  # Injecte la variable d'environnement `ACME_EMAIL` provenant des secrets GitHub
          script: |  # Les commandes à exécuter sur le VPS via SSH
            sudo docker volume create letsencrypt || true  # Crée un volume Docker nommé "letsencrypt" pour stocker les certificats SSL ; ignore l'erreur si le volume existe déjà
            sudo docker stop seah-traefik || true  # Arrête le conteneur "seah-traefik" s'il est en cours d'exécution, ignore l'erreur si le conteneur n'existe pas
            sudo docker rm seah-traefik || true  # Supprime le conteneur "seah-traefik" s'il existe, ignore l'erreur si le conteneur n'existe pas
            sudo docker run -d \  # Lance le conteneur Traefik en mode détaché (en arrière-plan)
              -p 80:80 -p 443:443 \  # Mappe le port 80 (HTTP) et le port 443 (HTTPS) du VPS aux ports 80 et 443 du conteneur Traefik
              --restart unless-stopped \  # Configure le conteneur pour redémarrer automatiquement sauf s'il est explicitement arrêté
              --name seah-traefik \  # Attribue le nom "seah-traefik" au conteneur
              --env ACME_EMAIL=${{ secrets.ACME_EMAIL }} \  # Passe la variable d'environnement `ACME_EMAIL` au conteneur
              --volume /var/run/docker.sock:/var/run/docker.sock:ro \  # Monte le socket Docker pour permettre à Traefik de surveiller les conteneurs Docker
              --volume letsencrypt:/letsencrypt \  # Monte le volume "letsencrypt" pour stocker les certificats SSL
              traefik:v2.9 \  # Spécifie l'image Traefik officielle version 2.9
              --api.insecure=true \  # Active l'API Traefik en mode non sécurisé (utile pour les tests, mais à désactiver en production)
              --providers.docker=true \  # Indique à Traefik d'utiliser Docker comme fournisseur de services
              --entrypoints.web.address=:80 \  # Définit le point d'entrée HTTP de Traefik sur le port 80
              --entrypoints.websecure.address=:443 \  # Définit le point d'entrée HTTPS de Traefik sur le port 443
              --certificatesresolvers.myresolver.acme.tlschallenge=true \  # Active le challenge TLS pour la validation des certificats Let's Encrypt
              --certificatesresolvers.myresolver.acme.email=${ACME_EMAIL} \  # Spécifie l'adresse e-mail utilisée pour l'enregistrement des certificats Let's Encrypt
              --certificatesresolvers.myresolver.acme.storage=/letsencrypt/acme.json  # Indique à Traefik l'emplacement où stocker les informations des certificats Let's Encrypt dans le volume "letsencrypt"

