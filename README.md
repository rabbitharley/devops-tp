# TP DevOps – Partie 01 : Docker

## Structure du projet

Un dossier par image Docker :

```
devops-tp/
├── flask-app/        # TD : application Flask de découverte
├── database/         # Base de données PostgreSQL
│   ├── Dockerfile
│   └── sql/
│       ├── 01-CreateScheme.sql
│       └── 02-InsertData.sql
├── backend/          # API Java / Spring Boot
├── http-server/      # Serveur HTTP Apache (reverse proxy)
└── README.md         # Ce rapport
```

---

## 0. TD – Découverte de Docker (application Flask)

Application Python Flask affichant un GIF de chat aléatoire, conteneurisée à partir de l'image `alpine:3.21.0`.

```bash
docker build -t bunny/myfirstapp .
docker run -p 8888:5000 --name myfirstapp bunny/myfirstapp
```

L'application est accessible sur http://localhost:8888.

**Remarques :**
- Les noms d'images Docker doivent être en **minuscules** : `Bunny/myfirstapp` est interprété comme un nom de serveur de registre et provoque une erreur.
- `docker run` crée un nouveau conteneur, `docker start` relance un conteneur existant.
- Problème rencontré : les conteneurs n'avaient pas accès à Internet pendant le build (`apk add` échouait). <!-- À COMPLÉTER : expliquer ce qui a résolu le problème (DNS, VPN, antivirus...) -->

---

## 1. Base de données

### Dockerfile

```dockerfile
FROM postgres:17.2-alpine

ENV POSTGRES_DB=db \
    POSTGRES_USER=usr

COPY sql/ /docker-entrypoint-initdb.d/
```

- `FROM postgres:17.2-alpine` : image officielle PostgreSQL basée sur Alpine (légère).
- `ENV` : nom de la base et de l'utilisateur. Le mot de passe n'est volontairement **pas** défini ici (voir question 1-1).
- `COPY sql/ /docker-entrypoint-initdb.d/` : les scripts SQL de ce dossier sont exécutés automatiquement, par ordre alphabétique, au premier démarrage du conteneur. Les préfixes `01-` et `02-` garantissent que les tables sont créées avant l'insertion des données.

### Commandes

```bash
# Réseau privé permettant aux conteneurs de communiquer par leur nom
docker network create app-network

# Construction de l'image
docker build -t bunny/database .

# Lancement de la base
docker run -d --name database --network app-network \
  -e POSTGRES_PASSWORD=pwd \
  -v pgdata:/var/lib/postgresql/data \
  bunny/database

# Interface web Adminer pour consulter la base
docker run -d -p 8090:8080 --network app-network --name adminer adminer
```

| Option | Rôle |
|---|---|
| `-d` | Lance le conteneur en arrière-plan |
| `--name database` | Nomme le conteneur ; ce nom sert aussi d'adresse sur le réseau |
| `--network app-network` | Connecte le conteneur au réseau privé partagé avec Adminer (et plus tard le backend) |
| `-e POSTGRES_PASSWORD=pwd` | Passe le mot de passe au lancement, hors de l'image |
| `-v pgdata:/var/lib/postgresql/data` | Stocke les données dans un volume nommé, indépendant du conteneur |
| `-p 8090:8080` | Publie le port d'Adminer sur la machine hôte |

La base n'a **pas** de `-p` : elle n'a pas besoin d'être accessible depuis l'hôte, seulement depuis le réseau Docker.

Connexion dans Adminer (http://localhost:8090) : système PostgreSQL, serveur `database`, utilisateur `usr`, base `db`.

### Vérifications

- `docker logs database` montre l'exécution des scripts `01-CreateScheme.sql` et `02-InsertData.sql`.
- Les tables `departments` et `students` et leurs données sont visibles dans Adminer.
- **Persistance** : après avoir ajouté une ligne, supprimé le conteneur (`docker rm -f database`) puis relancé la même commande `docker run`, la ligne est toujours présente.

**Point d'attention :** les scripts d'initialisation ne s'exécutent que si le volume est vide. Pour les rejouer, il faut supprimer le volume (`docker volume rm pgdata`).

### Questions

**1-1. Pourquoi est-il préférable de passer les variables d'environnement avec `-e` plutôt que de les écrire dans le Dockerfile ?**

Le Dockerfile est versionné dans git et l'image peut être publiée : toute valeur écrite dedans devient lisible par n'importe qui ayant accès au dépôt ou à l'image (par exemple via `docker inspect` ou `docker history`). En passant le mot de passe avec `-e` au lancement, le secret n'est jamais stocké dans l'image ni dans le code source. Cela permet aussi d'utiliser la même image avec des valeurs différentes selon l'environnement (développement, test, production).

**1-2. Pourquoi faut-il attacher un volume au conteneur PostgreSQL ?**

Un conteneur est éphémère : lorsqu'il est supprimé, tout ce qui a été écrit dans son système de fichiers disparaît avec lui. Sans volume, chaque recréation du conteneur repartirait d'une base vide. Le volume stocke les fichiers de la base (`/var/lib/postgresql/data`) en dehors du conteneur, sur l'hôte : les données survivent à la suppression et à la recréation du conteneur, ainsi qu'aux mises à jour de l'image.

**1-3. Éléments essentiels du conteneur de base de données**

Voir le Dockerfile et les commandes ci-dessus.

---

## 2. Backend API

### 2.1 Hello World Java

Un simple programme Java (`Main.java`) affichant « Hello World! ».

Pour exécuter du code Java déjà compilé, un **JRE** (environnement d'exécution) suffit : l'image `eclipse-temurin:21-jre-alpine` a été choisie car elle est légère et ne contient pas les outils de compilation.

**Première version : compilation sur la machine hôte**

Plutôt que d'installer un JDK sur le PC, la compilation a été faite dans un conteneur temporaire :

```bash
docker run --rm -v "${PWD}:/app" -w /app eclipse-temurin:21-jdk-alpine javac Main.java
```

- `-v "${PWD}:/app"` : partage le dossier courant avec le conteneur.
- `-w /app` : définit ce dossier comme répertoire de travail.
- `--rm` : supprime le conteneur à la fin de la compilation.

Le fichier `Main.class` produit est ensuite copié dans l'image :

```dockerfile
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY Main.class .
CMD ["java", "Main"]
```

```bash
docker build -t bunny/hello-java .
docker run --rm bunny/hello-java
# Hello World!
```

### 2.2 Build multistage

La compilation est maintenant réalisée par Docker lui-même, en deux étapes :

```dockerfile
# Étape 1 : compilation avec le JDK
FROM eclipse-temurin:21-jdk-alpine AS build
WORKDIR /usr/src
COPY Main.java .
RUN javac Main.java

# Étape 2 : exécution avec le JRE seulement
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY --from=build /usr/src/Main.class .
CMD ["java", "Main"]
```

- La première étape, nommée `build`, utilise le JDK pour compiler `Main.java`.
- La seconde repart d'une image JRE vierge et ne récupère que `Main.class` grâce à `COPY --from=build`.
- Seule la dernière étape constitue l'image finale : le JDK n'y est pas inclus.

Vérification : après suppression de `Main.class` sur la machine hôte, le build réussit et le conteneur affiche toujours « Hello World! », ce qui prouve que la compilation a bien eu lieu dans Docker.

### 2.3 API Spring Boot

Projet généré sur https://start.spring.io : Maven, Java 21, packaging Jar, dépendance **Spring Web**, groupe `fr.takima.training`, artefact `simpleapi`.

**Remarque :** la version 3.4.5 de Spring Boot demandée par l'énoncé n'est plus proposée par Spring Initializr. La version stable **4.1.1** a été utilisée à la place (les versions `SNAPSHOT` et `M2` sont des préversions, à éviter).

Un contrôleur `GreetingController` expose un endpoint `/` qui renvoie un message de bienvenue et un compteur d'appels.

```dockerfile
# Build stage
FROM eclipse-temurin:21-jdk-alpine AS myapp-build
ENV MYAPP_HOME=/opt/myapp
WORKDIR $MYAPP_HOME

RUN apk add --no-cache maven

COPY pom.xml .
RUN mvn dependency:go-offline

COPY src ./src
RUN mvn package -DskipTests

# Run stage
FROM eclipse-temurin:21-jre-alpine
ENV MYAPP_HOME=/opt/myapp
WORKDIR $MYAPP_HOME
COPY --from=myapp-build $MYAPP_HOME/target/*.jar $MYAPP_HOME/myapp.jar

ENTRYPOINT ["java", "-jar", "myapp.jar"]
```

```bash
docker build -t bunny/simple-api .
docker run -d -p 8080:8080 --name simple-api bunny/simple-api
```

- http://localhost:8080/ renvoie `{"id":1,"content":"Hello, World!"}`
- http://localhost:8080/?name=Bunny renvoie `{"id":2,"content":"Hello, Bunny!"}`

### Question 1-4 : Pourquoi un build multistage ? Explication de chaque étape du Dockerfile

**Pourquoi un build multistage ?**

Compiler une application Java nécessite un JDK et Maven, mais l'exécuter ne nécessite qu'un JRE. Le build multistage sépare ces deux besoins :

- **Image finale plus légère** : elle ne contient que le JRE et le `.jar`. Le JDK, Maven, le code source et les bibliothèques téléchargées pour la compilation restent dans l'étape de build et ne sont pas inclus.
- **Plus sécurisée** : moins d'outils présents en production, donc une surface d'attaque réduite.
- **Build reproductible** : il n'est pas nécessaire d'installer Java ou Maven sur sa machine. Docker suffit, et tout le monde compile avec les mêmes versions.
- **Un seul fichier** décrit à la fois la compilation et l'exécution.

**Explication de chaque étape :**

*Étape de build :*
- `FROM eclipse-temurin:21-jdk-alpine AS myapp-build` : part d'une image contenant le JDK 21, et nomme cette étape `myapp-build` pour pouvoir y faire référence plus tard.
- `ENV MYAPP_HOME=/opt/myapp` : définit une variable contenant le chemin de l'application.
- `WORKDIR $MYAPP_HOME` : se place dans ce dossier (créé s'il n'existe pas).
- `RUN apk add --no-cache maven` : installe Maven, l'outil de build Java.
- `COPY pom.xml .` : copie le fichier de configuration Maven, qui liste les dépendances.
- `RUN mvn dependency:go-offline` *(ajouté par rapport à l'énoncé)* : télécharge toutes les dépendances. Comme cette instruction ne dépend que du `pom.xml`, Docker la garde en cache tant que ce fichier ne change pas : modifier le code ne relance pas le téléchargement.
- `COPY src ./src` : copie le code source.
- `RUN mvn package -DskipTests` : compile le code et produit le fichier `.jar` dans `target/`, sans exécuter les tests.

*Étape d'exécution :*
- `FROM eclipse-temurin:21-jre-alpine` : repart d'une image neuve ne contenant que le JRE. Tout ce qui précède est abandonné.
- `ENV` et `WORKDIR` : recréent le dossier de l'application dans cette nouvelle image.
- `COPY --from=myapp-build $MYAPP_HOME/target/*.jar $MYAPP_HOME/myapp.jar` : récupère uniquement le `.jar` produit à l'étape de build et le renomme `myapp.jar`.
- `ENTRYPOINT ["java", "-jar", "myapp.jar"]` : commande lancée au démarrage du conteneur, qui exécute l'application.

---

## 3. Serveur HTTP

<!-- À COMPLÉTER -->

### Question 1-5 : Pourquoi a-t-on besoin d'un reverse proxy ?

<!-- À COMPLÉTER -->

---

## 4. Docker Compose

### Question 1-6 : Pourquoi docker-compose est-il si important ?

<!-- À COMPLÉTER -->

### Question 1-7 : Principales commandes docker-compose

<!-- À COMPLÉTER -->

### Question 1-8 : Documentation du fichier docker-compose.yml

<!-- À COMPLÉTER -->

---

## 5. Publication

### Question 1-9 : Commandes de publication et images publiées

<!-- À COMPLÉTER -->

### Question 1-10 : Pourquoi publier nos images dans un dépôt en ligne ?

<!-- À COMPLÉTER -->