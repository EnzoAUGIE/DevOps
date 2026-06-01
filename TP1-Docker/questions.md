# TP1 Docker - Réponses aux questions

## Q1-1 For which reason is it better to run the container with a flag -e to give the environment variables rather than put them directly in the Dockerfile?

Il est préférable d'utiliser le flag `-e` lors du `docker run` plutôt que de définir les variables sensibles (comme `POSTGRES_PASSWORD`) directement dans le Dockerfile. En effet, le Dockerfile est un fichier versionné et déposé sur GitHub,
 ce qui signifie que les mots de passe seraient visibles publiquement. En passant les variables au moment du lancement du container avec `-e`, les secrets restent uniquement sur la machine qui exécute le container et ne sont jamais exposés
 dans le code source.

## Q1-2  Why do we need a volume to be attached to our postgres container?

Sans volume, toutes les données stockées dans la base de données sont perdues dès que le container est détruit, car elles sont enregistrées dans le système de fichiers éphémère du container. En attachant un volume, les données sont persistées
 sur le disque de la machine hôte, ce qui permet de les retrouver intactes même après la destruction et la recréation du container.

## Q1-3 Document your database container essentials: commands and Dockerfile.

Dockerfile :

FROM postgres:17.2-alpine

ENV POSTGRES_DB=db \
    POSTGRES_USER=usr \
    POSTGRES_PASSWORD=pwd

COPY sql-scripts/ /docker-entrypoint-initdb.d/

Commandes :

# Créer le network
docker network create app-network

# Builder l'image
docker build -t enzoaugie/database .

# Lancer le container avec volume
docker run --name database \
    -e POSTGRES_DB=db \
    -e POSTGRES_USER=usr \
    -e POSTGRES_PASSWORD=pwd \
    --network app-network \
    -v /Users/enzoaugie/data/postgres:/var/lib/postgresql/data \
    -d enzoaugie/database

# Lancer Adminer pour visualiser la DB
docker run \
    -p "8090:8080" \
    --net=app-network \
    --name=adminer \
    -d \
    adminer

## Q1-4 Why do we need a multistage build? And explain each step of this dockerfile.

Un multistage build permet de séparer la phase de compilation de la phase d'exécution.Sans multistage, l'image finale contiendrait le JDK entier (~200MB) alors qu'on n'a besoin que du JRE (~50MB) pour exécuter l'application.
 Le multistage permet donc d'avoir une imagefinale plus légère et plus sécurisée car elle ne contient pas les outils de compilation.

Explication de chaque étape :

# Stage 1 - Build
FROM eclipse-temurin:21-jdk-alpine AS myapp-build
 On utilise le JDK (Java Development Kit) qui contient les outils de compilation
 On nomme ce stage "myapp-build" pour y faire référence ensuite

ENV MYAPP_HOME=/opt/myapp
WORKDIR $MYAPP_HOME
On définit et on se place dans le dossier de travail /opt/myapp

RUN apk add --no-cache maven
On installe Maven, l'outil qui gère les dépendances et compile le projet Java

COPY pom.xml .
COPY src ./src
On copie les fichiers source du projet dans le container

RUN mvn package -DskipTests
Maven compile le code et génère un fichier .jar (l'application empaquetée)
-DskipTests permet de sauter les tests pour aller plus vite

# Stage 2 - Run
FROM eclipse-temurin:21-jre-alpine
On repart d'une image propre avec uniquement le JRE (Java Runtime Environment)
Le JRE est plus léger car il ne contient que ce qui est nécessaire pour exécuter du Java

ENV MYAPP_HOME=/opt/myapp
WORKDIR $MYAPP_HOME

COPY --from=myapp-build $MYAPP_HOME/target/*.jar $MYAPP_HOME/myapp.jar
On récupère uniquement le .jar compilé depuis le stage 1
L'image finale ne contient pas le JDK ni Maven, juste le JRE et le .jar

ENTRYPOINT ["java", "-jar", "myapp.jar"]
Commande exécutée au démarrage du container pour lancer l'application

## Q1-5 Why do we need a reverse proxy?

Le reverse proxy permet de cacher l'architecture interne de l'application. L'utilisateur n'accède qu'au port 80 du serveur Apache, sans connaître l'existence du backend sur le port 8080. Il permet aussi de gérer le SSL, le load balancing
et de servir plusieurs applications derrière une seule adresse.

## Q1-6 Why is docker-compose so important?

Docker Compose est important car il permet d'orchestrer plusieurs containers en une seule commande. Sans lui, il faudrait lancer chaque container manuellement avec tous ses paramètres (network, volumes,ports, variables d'environnement...).
Il gère aussi automatiquement l'ordre de démarrage grâce à "depends_on" (ex: le backend démarre après la database). Il permet également de tout arrêter proprement en une seule commande avec "docker compose down".

## Q1-7  Document docker-compose most important commands.

docker compose up --build  → Lance tous les containers en rebuilant les images
docker compose up -d       → Lance tous les containers en arrière-plan (detached)
docker compose down        → Stoppe et supprime tous les containers et le network
docker compose ps          → Liste l'état de tous les containers
docker compose logs        → Affiche les logs de tous les containers
docker compose logs -f     → Affiche les logs en temps réel
docker compose build       → Rebuild les images sans lancer les containers
docker compose restart     → Redémarre tous les containers

## Q1-8 Document your docker-compose file.

services:
    backend:
        build: ./backend/simpleapi  → Chemin vers le Dockerfile du backend
        networks:
            - app-network           → Connecté au réseau virtuel
        depends_on:
            - database              → Démarre après la database

    database:
        build: ./database           → Chemin vers le Dockerfile de la database
        networks:
            - app-network           → Connecté au réseau virtuel
        environment:
            POSTGRES_DB: db         → Nom de la base de données
            POSTGRES_USER: usr      → Utilisateur PostgreSQL
            POSTGRES_PASSWORD: pwd  → Mot de passe PostgreSQL
        volumes:
            - db-data:/var/lib/postgresql/data  → Volume pour persister les données

    httpd:
        build: ./httpd              → Chemin vers le Dockerfile Apache
        ports:
            - "80:80"               → Expose le port 80 sur la machine hôte
        networks:
            - app-network           → Connecté au réseau virtuel
        depends_on:
            - backend               → Démarre après le backend

networks:
    app-network:                    → Définit le réseau virtuel partagé entre les containers

volumes:
    db-data:                        → Définit le volume nommé pour persister les données PostgreSQL

## Q1-9 Document your publication commands and published images in dockerhub.

# Connexion à Docker Hub
docker login

# Tagger les images avec une version
docker tag tp1-docker-database:latest enzoaugie/tp1-docker-database:1.0
docker tag tp1-docker-backend:latest enzoaugie/tp1-docker-backend:1.0
docker tag tp1-docker-httpd:latest enzoaugie/tp1-docker-httpd:1.0

# Pousser les images sur Docker Hub
docker push enzoaugie/tp1-docker-database:1.0
docker push enzoaugie/tp1-docker-backend:1.0
docker push enzoaugie/tp1-docker-httpd:1.0

Images publiées sur : https://hub.docker.com/u/enzoaugie
- enzoaugie/tp1-docker-database:1.0
- enzoaugie/tp1-docker-backend:1.0
- enzoaugie/tp1-docker-httpd:1.0

## Q1-10 Why do we put our images into an online repo?

Mettre les images sur Docker Hub permet de :
1. Partager les images avec d'autres membres de l'équipe sans avoir à rebuilder
2. Déployer sur n'importe quelle machine avec un simple "docker pull"
3. Versionner les images (tag 1.0, 2.0...) pour revenir à une version précédente
4. Avoir une sauvegarde des images en cas de perte de la machine locale
5. Permettre à d'autres de réutiliser nos images comme base pour leurs projets
