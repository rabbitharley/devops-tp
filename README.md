# Rapport DevOps

Ce rapport présente ce que j'ai appris et compris au cours des trois TP, ainsi que mes réponses aux questions posées. Le fil conducteur est une application 3-tiers (base de données PostgreSQL, API Spring Boot, serveur HTTP Apache), que j'ai successivement :

1. **conteneurisée** avec Docker et orchestrée avec docker-compose (TP 1) ;
2. **testée et publiée automatiquement** avec GitHub Actions (TP 2) ;
3. **déployée automatiquement** sur un serveur distant avec Ansible, avec un front-end et des secrets chiffrés (TP 3).

**Architecture finale :**

```
                                    ┌──► front    (Vue.js + nginx)
Navigateur ──► httpd (Apache, 80) ──┤
                                    └──► backend  (Spring Boot, 8080) ──► database (PostgreSQL, 5432)

   /      → front
   /api/  → backend
```

Seul Apache est accessible depuis l'extérieur. Un push sur `main` déclenche toute la chaîne : **tests → publication des images sur Docker Hub → déploiement sur le serveur**.

**Organisation du dépôt** (un dossier par image) :

```
devops-tp/
├── .github/workflows/main.yml   # pipeline CI/CD
├── ansible/                     # inventaire, rôles, playbook, secrets chiffrés
├── backend/simpleapi/           # API Spring Boot
├── database/                    # PostgreSQL et scripts SQL d'initialisation
├── front/                       # front-end Vue.js
├── http-server/                 # Apache, reverse proxy
├── docker-compose.yml           # exécution en local
└── .env.example                 # modèle de configuration (le vrai .env n'est pas versionné)
```

---

# TP 1 – Docker

## Ce que j'ai compris de Docker

**Image et conteneur.** Une image est un modèle figé, en lecture seule, construit à partir d'un `Dockerfile`. Un conteneur est une instance de cette image en cours d'exécution. Une même image peut donner plusieurs conteneurs. Contrairement à une machine virtuelle, un conteneur ne contient pas de système d'exploitation complet : il partage le noyau de la machine hôte, ce qui le rend léger et rapide à démarrer.

**Le conteneur est éphémère.** Un conteneur vit tant que sa commande principale tourne : `docker run alpine /bin/sh` s'arrête immédiatement, car le shell n'a rien à exécuter sans terminal (`-it`). Et quand on supprime un conteneur, tout ce qu'il a écrit disparaît. Cette idée explique à elle seule l'usage des volumes, des variables d'environnement et des images reconstruites.

**Les couches et le cache.** Chaque instruction d'un Dockerfile crée une couche, mise en cache. C'est pourquoi on copie la liste des dépendances (`requirements.txt`, `pom.xml`) **avant** le code : tant qu'elle ne change pas, les dépendances ne sont pas réinstallées à chaque modification du code.

**Build et exécution sont deux moments distincts.** `RUN` s'exécute pendant la construction de l'image, `CMD` ou `ENTRYPOINT` au démarrage du conteneur. Conséquence concrète : quand j'ai modifié la configuration de l'API, il a fallu **reconstruire l'image**, car le fichier était copié dans l'image au moment du build.

**Quelques pièges rencontrés :**
- les noms d'images doivent être en minuscules : `Bunny/myfirstapp` est interprété comme l'adresse d'un registre ;
- `docker run` crée un **nouveau** conteneur, alors que `docker start` relance un conteneur existant ;
- `EXPOSE` ne fait que documenter un port : c'est `-p hôte:conteneur` qui le rend accessible ;
- quand un build échouait faute d'accès à Internet, j'ai appris à diagnostiquer couche par couche : tester depuis Windows, puis la résolution DNS depuis un conteneur, puis la connexion elle-même.

## 1. Base de données

```dockerfile
FROM postgres:17.2-alpine

ENV POSTGRES_DB=db \
    POSTGRES_USER=usr

COPY sql/ /docker-entrypoint-initdb.d/
```

L'image officielle PostgreSQL exécute automatiquement, au **premier** démarrage, les scripts placés dans `/docker-entrypoint-initdb.d`, par ordre alphabétique. Les préfixes `01-CreateScheme.sql` et `02-InsertData.sql` garantissent que les tables existent avant l'insertion des données. Ces scripts ne sont rejoués que si le volume de données est vide.

```bash
docker network create app-network
docker build -t <utilisateur>/database .
docker run -d --name database --network app-network \
  -e POSTGRES_PASSWORD=pwd \
  -v pgdata:/var/lib/postgresql/data \
  <utilisateur>/database
docker run -d -p 8090:8080 --network app-network --name adminer adminer
```

Sur un réseau Docker, les conteneurs se joignent **par leur nom** : Adminer se connecte au serveur `database`. La base n'a pas d'option `-p` : elle n'a besoin d'être accessible que depuis le réseau Docker.

### Question 1-1 : Pourquoi passer les variables d'environnement avec `-e` plutôt que dans le Dockerfile ?

Le Dockerfile est versionné dans git, et l'image peut être publiée. Une valeur écrite dans le Dockerfile devient donc lisible par n'importe qui ayant accès au dépôt ou à l'image, par exemple avec `docker inspect` ou `docker history`. En passant le mot de passe avec `-e` au lancement, le secret n'est stocké ni dans l'image ni dans le code. Cela permet aussi d'utiliser **la même image** avec des valeurs différentes selon l'environnement (développement, test, production).

### Question 1-2 : Pourquoi attacher un volume au conteneur PostgreSQL ?

Un conteneur est éphémère : s'il est supprimé, les données de la base disparaissent avec lui. Le volume stocke le dossier `/var/lib/postgresql/data` **en dehors du conteneur**. Je l'ai vérifié en ajoutant une ligne dans la base, en supprimant le conteneur puis en le recréant : la ligne était toujours là.

### Question 1-3 : Éléments essentiels du conteneur de base de données

- **Image** : `postgres:17.2-alpine`, officielle et légère ;
- **Initialisation** : scripts SQL copiés dans `/docker-entrypoint-initdb.d` ;
- **Configuration** : nom de la base et utilisateur dans le Dockerfile, mot de passe passé au lancement (`-e`) ;
- **Persistance** : volume nommé monté sur `/var/lib/postgresql/data` ;
- **Réseau** : `app-network`, sur lequel le conteneur est joignable sous le nom `database` ;
- **Exposition** : aucun port publié vers l'hôte.

## 2. Backend : le build multistage

Pour exécuter du Java, un JRE suffit ; pour le compiler, il faut un JDK et Maven. Je l'ai d'abord vérifié avec un simple Hello World compilé dans un conteneur temporaire, puis exécuté avec une image JRE. J'ai ensuite confié la compilation à Docker lui-même :

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

L'API fournie par l'enseignant se connecte à la base. Dans `application.yml`, l'adresse de la base est `database` et non `localhost` : à l'intérieur d'un conteneur, `localhost` désigne le conteneur lui-même. Toutes les valeurs de connexion sont lues dans des variables d'environnement (`${DB_HOST}`, `${DB_PASSWORD}`…), pour ne rien écrire en dur.

### Question 1-4 : Pourquoi un build multistage ? Explication de chaque étape

**Pourquoi :** compiler demande un JDK et Maven, exécuter ne demande qu'un JRE. Le multistage compile dans une première image, puis ne garde que le résultat dans une seconde. L'image finale est ainsi :
- **plus légère** : ni JDK, ni Maven, ni code source, ni dépendances de compilation ;
- **plus sûre** : moins d'outils en production, donc moins de failles possibles ;
- **reproductible** : il n'est pas nécessaire d'installer Java sur sa machine, Docker suffit.

**Les étapes :**
- `FROM ...jdk-alpine AS myapp-build` : première étape, avec le JDK 21, nommée pour pouvoir y faire référence ;
- `ENV` et `WORKDIR` : définissent et créent le dossier de travail `/opt/myapp` ;
- `RUN apk add maven` : installe l'outil de build ;
- `COPY pom.xml` puis `RUN mvn dependency:go-offline` : télécharge les dépendances dans une couche qui reste en cache tant que le `pom.xml` ne change pas (ajout par rapport à l'énoncé) ;
- `COPY src` puis `RUN mvn package -DskipTests` : compile et produit le `.jar` ;
- `FROM ...jre-alpine` : seconde étape, qui repart d'une image vierge avec seulement le JRE ;
- `COPY --from=myapp-build` : récupère uniquement le `.jar` de la première étape ;
- `ENTRYPOINT` : lance l'application au démarrage du conteneur.

## 3. Serveur HTTP et reverse proxy

J'ai d'abord servi une page statique avec l'image `httpd:2.4-alpine`, puis récupéré sa configuration par défaut avec `docker cp`. Pour en faire un reverse proxy, j'ai activé les modules `proxy` et `proxy_http`, puis ajouté une redirection vers le backend :

```apache
<VirtualHost *:80>
ProxyPreserveHost On
ProxyPass / http://backend:8080/
ProxyPassReverse / http://backend:8080/
</VirtualHost>
```

`ProxyPass` transmet les requêtes au backend ; `ProxyPassReverse` réécrit les en-têtes des réponses pour qu'elles pointent vers le proxy. J'ai aussi utilisé `docker stats`, `docker inspect` et `docker logs` pour observer les conteneurs : les logs d'Apache montrent bien chaque requête qui passe par lui.

### Question 1-5 : Pourquoi a-t-on besoin d'un reverse proxy ?

Le reverse proxy est un **point d'entrée unique** placé devant l'application :
- seul Apache est exposé ; le backend et la base restent cachés dans le réseau Docker, ce qui réduit la surface d'attaque ;
- le HTTPS, les journaux et les règles d'accès se gèrent à un seul endroit ;
- il peut **répartir la charge** entre plusieurs instances du backend ;
- il permet de servir plusieurs services sous une même adresse. C'est ce que j'ai fait au TP 3 : `/` pour le front, `/api/` pour l'API.

## 4. docker-compose

docker-compose décrit tous les services dans un seul fichier. J'ai centralisé toute la configuration (noms d'hôtes, ports, mot de passe) dans un fichier `.env`, lu automatiquement par compose et exclu de git. Un modèle `.env.example`, sans le vrai mot de passe, est versionné.

```yaml
services:
  database:
    build: ./database
    environment:
      POSTGRES_DB: ${POSTGRES_DB}
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      PGPORT: ${DB_PORT}
    volumes:
      - db-data:/var/lib/postgresql/data
    networks:
      - back-network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U $${POSTGRES_USER} -d $${POSTGRES_DB}"]
      interval: 5s
      timeout: 5s
      retries: 10
    restart: unless-stopped

  backend:
    build: ./backend/simpleapi
    environment:
      DB_HOST: ${DB_HOST}
      DB_PORT: ${DB_PORT}
      DB_NAME: ${POSTGRES_DB}
      DB_USER: ${POSTGRES_USER}
      DB_PASSWORD: ${POSTGRES_PASSWORD}
      BACKEND_PORT: ${BACKEND_PORT}
    networks:
      - back-network
      - front-network
    depends_on:
      database:
        condition: service_healthy
    restart: unless-stopped

  front:
    build: ./front
    networks:
      - front-network
    restart: unless-stopped

  httpd:
    build: ./http-server
    environment:
      BACKEND_HOST: ${BACKEND_HOST}
      BACKEND_PORT: ${BACKEND_PORT}
      FRONT_HOST: ${FRONT_HOST}
      HTTPD_PORT: ${HTTPD_PORT}
    ports:
      - "${HTTP_PORT}:${HTTPD_PORT}"
    networks:
      - front-network
    depends_on:
      - backend
      - front
    restart: unless-stopped

networks:
  back-network:
  front-network:

volumes:
  db-data:
```

### Question 1-6 : Pourquoi docker-compose est-il si important ?

Sans compose, il faut lancer chaque conteneur à la main, avec de longues commandes `docker run`, dans le bon ordre, et tout recommencer à chaque modification. C'est long et source d'erreurs. Avec compose :
- toute l'application est décrite dans **un seul fichier versionné** : l'infrastructure devient du code ;
- **une seule commande** construit, démarre ou arrête l'ensemble ;
- l'ordre de démarrage et les dépendances sont gérés (`depends_on`) ;
- les réseaux et volumes sont créés automatiquement ;
- l'environnement est **reproductible** à l'identique sur n'importe quelle machine.

### Question 1-7 : Les commandes docker-compose les plus importantes

| Commande | Rôle |
|---|---|
| `docker compose up -d --build` | construit les images et démarre tous les services en arrière-plan |
| `docker compose ps` | affiche l'état des services |
| `docker compose logs -f <service>` | suit les logs d'un service |
| `docker compose restart <service>` | redémarre un service |
| `docker compose stop` / `start` | arrête ou relance sans supprimer |
| `docker compose down` | supprime conteneurs et réseaux, en gardant les volumes |
| `docker compose down -v` | supprime aussi les volumes, donc les données |
| `docker compose config` | affiche le fichier final, avec les variables du `.env` remplacées |

### Question 1-8 : Documentation du fichier docker-compose

- **`database`** : construite depuis `./database`, configurée par `.env`, avec ses données dans le volume `db-data`. Le **healthcheck** (`pg_isready`) vérifie que PostgreSQL accepte réellement les connexions ; le double `$$` laisse la variable au conteneur au lieu que compose la remplace.
- **`backend`** : reçoit de quoi joindre la base. `condition: service_healthy` le fait attendre que la base soit **prête**, et pas seulement démarrée : c'est la limite de `depends_on` seul.
- **`front`** : le front-end, sans port publié.
- **`httpd`** : le **seul service avec un port publié**, point d'entrée unique de l'application.
- **Deux réseaux** : `front-network` (Apache, front, backend) et `back-network` (backend, base). Apache ne peut pas joindre la base directement.
- **`restart: unless-stopped`** : un service qui plante redémarre automatiquement, sauf s'il a été arrêté volontairement.
- **Aucune valeur en dur** : hôtes, ports et mot de passe viennent du `.env`, et chaque application les lit avec la syntaxe `${...}` (Spring dans `application.yml`, Apache dans `httpd.conf`).

**Problème rencontré :** au premier lancement, Apache renvoyait une erreur 503, car le backend n'avait pas fini de démarrer. J'ai compris qu'après un échec, Apache « met de côté » le backend pendant un moment : un `docker compose restart httpd` a suffi.

## 5. Publication

### Question 1-9 : Commandes de publication et images publiées

```bash
docker login
docker tag devops-tp-database <utilisateur>/tp-database:1.0
docker push <utilisateur>/tp-database:1.0
# idem pour le backend et httpd
```

`docker tag` ajoute un second nom à une image existante, sous la forme `utilisateur/image:version` attendue par Docker Hub. Un numéro de version (`1.0`) identifie précisément une version, contrairement à `latest`. Les images publiées sont visibles sur `https://hub.docker.com/u/<utilisateur>`. Par la suite, c'est le pipeline du TP 2 qui publie automatiquement `tp-devops-database`, `tp-devops-simple-api`, `tp-devops-httpd` et `tp-devops-front`.

### Question 1-10 : Pourquoi mettre nos images dans un dépôt en ligne ?

Une image locale n'existe que sur ma machine. Un registre en ligne permet de :
- **partager** les images avec l'équipe, sans qu'elle ait à les reconstruire ;
- **déployer** sur un serveur qui télécharge l'image avec `docker pull`, sans code source ni outils de build ;
- **garantir** que tout le monde exécute exactement la même version ;
- **versionner** grâce aux tags, et revenir à une version précédente ;
- **automatiser** : c'est la base de la CI/CD.

## Ce que je retiens du TP 1

- Un conteneur est jetable : ce qui doit durer va dans un **volume**, ce qui varie va dans des **variables d'environnement**.
- Les conteneurs se trouvent **par leur nom** sur un réseau Docker ; `localhost` désigne le conteneur lui-même.
- Le multistage sépare ce qui sert à **construire** de ce qui sert à **exécuter**.
- docker-compose transforme une suite de commandes en **un fichier reproductible**.

---

# TP 2 – GitHub et GitHub Actions

## Ce que j'ai appris sur Git et GitHub

**Les clés SSH.** Une paire de clés remplace le mot de passe : la clé **privée** reste sur ma machine et ne doit jamais être partagée ; la clé **publique** (`.pub`) est donnée à GitHub. Elle permet de cloner et de publier sans s'authentifier à chaque fois.

**Fork et clone.** Le fork copie un dépôt dans **mon compte GitHub** ; le clone le copie **sur ma machine**. J'ai forké le projet de l'enseignant, cloné le fork en SSH, puis publié une modification avec le cycle `git status` → `git add` → `git commit` → `git push`. `git status` montre en rouge ce qui est modifié mais pas encore ajouté, et en vert ce qui sera commité.

**La protection de branche.** Une règle sur `main` empêche de supprimer la branche ou de réécrire son historique avec `push --force`. En équipe, elle peut aussi imposer une relecture par *pull request*.

**Ne jamais publier de secret.** Un dépôt est public par défaut. Les mots de passe ont leur place dans les **secrets GitHub**, chiffrés.

## Intégration continue (CI)

Le pipeline est défini dans `.github/workflows/main.yml`. Il se déclenche à chaque push sur `main` ou `develop`, et à chaque *pull request*.

```yaml
  test-backend:
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v4
      - name: Set up JDK 21
        uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '21'
          cache: maven
      - name: Build and test with Maven
        run: mvn clean verify --file ./backend/simpleapi/pom.xml
```

J'ai compris la différence entre `uses`, qui appelle une **action** toute faite (récupérer le code, installer Java), et `run`, qui exécute une **commande**. `mvn clean verify` nettoie les anciens builds, compile, puis lance les tests unitaires (une classe isolée) et les tests d'intégration (plusieurs composants avec une vraie base).

**Ce que les erreurs m'ont appris :**
- le pipeline ciblait `main`, mais ma branche s'appelait `master` : rien ne se déclenchait. Je l'ai renommée, puis définie comme branche par défaut ;
- la compilation échouait sur GitHub mais pas chez moi : git connaissait le fichier sous le nom `SimpleapiApplication.java` alors que la classe s'appelait `SimpleApiApplication`. **Windows ignore la casse, Linux non.** C'est exactement ce que la CI sert à détecter : ce qui « marche sur ma machine » mais pas ailleurs.

### Question 2-1 : Que sont les Testcontainers ?

Testcontainers est une bibliothèque Java qui démarre de **vrais conteneurs Docker pendant les tests**. Ici, elle lance une base PostgreSQL au début des tests d'intégration, puis la supprime à la fin. On teste ainsi l'application contre une vraie base, identique à la production, sans rien installer : il suffit que Docker soit disponible, ce qui est le cas sur les machines de GitHub. L'URL `jdbc:tc:postgresql://...` du fichier de configuration des tests déclenche ce mécanisme.

## Livraison continue (CD)

Un second job construit les images et les publie sur Docker Hub :

```yaml
  build-and-push-docker-image:
    needs: test-backend
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v4
      - name: Login to DockerHub
        run: echo "${{ secrets.DOCKERHUB_TOKEN }}" | docker login --username ${{ secrets.DOCKERHUB_USERNAME }} --password-stdin
      - name: Build image and push backend
        uses: docker/build-push-action@v6
        with:
          context: ./backend/simpleapi
          tags: ${{ secrets.DOCKERHUB_USERNAME }}/tp-devops-simple-api:latest
          push: ${{ github.ref == 'refs/heads/main' }}
      # idem pour database, httpd et front
```

- La connexion utilise un **token** Docker Hub plutôt que le mot de passe du compte ;
- `push: ${{ github.ref == 'refs/heads/main' }}` : les images sont **construites** sur toutes les branches, ce qui vérifie les Dockerfiles, mais **publiées** uniquement depuis `main`.

J'ai aussi appris à lire les logs : quand la connexion échouait, la commande affichait `echo ""` et un `--username` vide. Un secret existant aurait été masqué par `***` : le secret n'était donc pas trouvé. Il avait été créé au mauvais endroit.

### Question 2-2 : Pourquoi utiliser des variables sécurisées ?

Le fichier `main.yml` est dans le dépôt, qui peut être public : un mot de passe écrit dedans serait lisible par tous. Les secrets GitHub sont **chiffrés**, ne peuvent plus être relus une fois enregistrés, et sont **masqués (`***`) dans les logs**. Le pipeline peut donc s'authentifier sans que les identifiants apparaissent ni dans le code, ni dans les journaux d'exécution.

### Question 2-3 : Pourquoi `needs: test-backend` sur le job Docker ?

Les jobs d'un workflow s'exécutent **en parallèle** par défaut. Sans `needs`, les images seraient publiées en même temps que les tests, donc **même si les tests échouent**. Avec `needs: test-backend`, le job attend la fin des tests et ne s'exécute que s'ils réussissent : on ne publie jamais une version cassée. Sur GitHub, la dépendance se voit sous forme de flèche entre les deux jobs.

### Question 2-4 : Pourquoi pousser les images Docker ?

Publier les images transforme chaque version validée par les tests en un **livrable prêt à être déployé**. N'importe quel serveur peut le récupérer avec `docker pull`, sans recompiler, et tout le monde exécute la même version. C'est le principe de la livraison continue, et ce sont précisément ces images qu'Ansible déploie au TP 3.

## Ce que je retiens du TP 2

- **CI** : chaque push est compilé et testé automatiquement, sur une machine neutre.
- **Continuous Delivery** : chaque version validée produit un livrable prêt à déployer. **Continuous Deployment** : ce livrable est en plus déployé automatiquement (TP 3).
- Les jobs sont parallèles, sauf si `needs` crée une dépendance.
- Les secrets ne se mettent jamais dans le code.
- La partie SonarCloud (quality gate) n'a pas été réalisée. J'en retiens le principe : une analyse statique du code (bugs, vulnérabilités, couverture de tests, duplications) qui peut bloquer un code ne respectant pas des seuils de qualité.

---

# TP 3 – Ansible

## Ce que j'ai compris d'Ansible

Ansible est un outil de **gestion de configuration et de déploiement** : il installe, configure et déploie des applications sur des serveurs, à partir de fichiers YAML.

- **Sans agent** : il se connecte aux serveurs **par SSH** et n'a besoin que de Python côté serveur. Il tourne sur ma machine (ici dans WSL, car Ansible ne fonctionne pas sous Windows) et pilote le serveur à distance.
- **Déclaratif** : on décrit l'**état voulu** (`state: present`, `state: started`), pas les commandes à taper.
- **Idempotent** : relancer une tâche ne change rien si l'état est déjà atteint. La première exécution affiche `changed`, les suivantes `ok`.

**La connexion SSH.** La clé privée fournie doit avoir des droits restreints (`chmod 400`) : SSH refuse une clé lisible par d'autres utilisateurs. J'ai aussi appris qu'un fichier téléchargé sous Windows peut contenir des fins de ligne `\r` qui le rendent illisible pour SSH, et que `ssh-keygen -y -f <clé>` permet de vérifier qu'une clé est valide avant d'accuser le serveur.

**Le module `ping`** ne se contente pas de tester le réseau : sa réponse `pong` prouve que le serveur est joignable, que l'utilisateur existe et que l'authentification fonctionne. Sans `-u admin` ni la clé, il échoue. Sans `--become`, l'installation d'Apache échoue aussi : il faut les droits administrateur, obtenus via `sudo`.

### Question 3-1 : Documenter l'inventaire et les commandes de base

```yaml
all:
  vars:
    ansible_user: admin
    ansible_ssh_private_key_file: ~/.ssh/takima_key
  children:
    prod:
      hosts: <serveur>
```

- `vars` : variables appliquées à tous les serveurs, ici l'utilisateur et le **chemin** de la clé. La clé elle-même n'est pas dans le dépôt ;
- `children` : les **groupes** de serveurs. Le groupe `prod` contient mon serveur ; on pourrait en ajouter d'autres et les cibler séparément.

| Commande | Rôle |
|---|---|
| `ansible all -i inventories/setup.yml -m ping` | vérifie la connexion et l'authentification |
| `ansible all -i inventories/setup.yml -m setup -a "filter=ansible_distribution*"` | lit des *facts* : des informations découvertes sur le serveur (ici, Debian et sa version) |
| `ansible all -i inventories/setup.yml -m apt -a "name=apache2 state=absent" --become` | désinstalle Apache ; relancée, la commande ne change plus rien |

## Playbooks et rôles

Un **playbook** regroupe des tâches dans un fichier que l'on peut rejouer. Il se vérifie avec `--syntax-check` avant d'être exécuté. Un **rôle**, créé avec `ansible-galaxy init`, range des tâches réutilisables dans une structure standard ; je n'ai gardé que les dossiers `tasks` et `handlers`.

### Question 3-2 : Documenter le playbook

Le playbook final comporte deux parties :

```yaml
# Installer Docker, avec le Python du système
- hosts: all
  gather_facts: true
  become: true
  roles:
    - docker

# Déployer l'application, avec le Python qui contient le SDK Docker
- hosts: all
  gather_facts: false
  become: true
  vars:
    ansible_python_interpreter: /opt/docker_venv/bin/python
  roles:
    - network
    - database
    - app
    - front
    - proxy
```

- `hosts: all` : tous les serveurs de l'inventaire ;
- `gather_facts: true` : collecte les facts, dont la version de Debian, utilisée pour choisir le bon dépôt Docker ;
- `become: true` : exécution en administrateur ;
- `ansible_python_interpreter` : les modules Docker d'Ansible ont besoin du SDK Python installé dans `/opt/docker_venv`. Ce SDK n'existe qu'après l'installation de Docker, d'où les deux parties.

Le rôle `docker` suit la documentation officielle : paquets prérequis (`apt`), clé GPG de Docker (`get_url`), dépôt officiel adapté à la version de Debian (`apt_repository`), installation de `docker-ce`, SDK Docker pour Python dans un environnement virtuel (`pip`), puis démarrage du service (`service`). Par rapport au modèle de l'énoncé, j'ai remplacé `apt_key`, déprécié sous Debian 12, et rendu l'installation du SDK idempotente avec le module `pip`. Relancer le playbook n'affiche alors que des tâches `ok`.

## Déploiement de l'application

Chaque partie de l'application a son rôle : `network` crée le réseau (`docker_network`), puis `database`, `app`, `front` et `proxy` lancent chacun un conteneur à partir des images publiées par le pipeline. Les valeurs communes sont dans `group_vars/all/vars.yml` :

```yaml
docker_user: <utilisateur>
docker_network: app-network
db_host: database
db_port: "5432"
db_name: db
db_user: usr
backend_host: backend
backend_port: "8080"
front_host: front
httpd_port: "80"
db_password: "{{ vault_db_password }}"
```

### Question 3-3 : Documenter la configuration des tâches `docker_container`

Exemple du rôle `app` :

```yaml
- name: Launch backend API
  community.docker.docker_container:
    name: "{{ backend_host }}"
    image: "{{ docker_user }}/tp-devops-simple-api:latest"
    pull: true
    state: started
    restart_policy: unless-stopped
    networks:
      - name: "{{ docker_network }}"
    env:
      DB_HOST: "{{ db_host }}"
      DB_PORT: "{{ db_port }}"
      DB_NAME: "{{ db_name }}"
      DB_USER: "{{ db_user }}"
      DB_PASSWORD: "{{ db_password }}"
      BACKEND_PORT: "{{ backend_port }}"
```

- `name` : le nom du conteneur, qui sert aussi d'adresse sur le réseau (`database`, `backend`, `front`, `httpd`) ;
- `image` : l'image publiée sur Docker Hub par le pipeline ;
- `pull: true` : télécharge la dernière version de l'image ; le conteneur est recréé si elle a changé ;
- `state: started` : l'état voulu, un conteneur qui tourne ;
- `restart_policy: unless-stopped` : redémarrage automatique en cas de plantage ou de redémarrage du serveur ;
- `networks` : rattache le conteneur au réseau `app-network` ;
- `env` : les variables d'environnement, qui remplissent les `${...}` de `application.yml`. La base reçoit son nom, son utilisateur et son mot de passe ; le proxy reçoit l'adresse du backend et du front ;
- `volumes` (base uniquement) : un volume nommé, pour la persistance ;
- `ports` (proxy uniquement) : seul Apache publie le port 80.

On retrouve exactement les notions du TP 1 (nom, réseau, variables, volume, port), mais décrites comme un état voulu et appliquées à distance.

## Déploiement continu

Un troisième job du pipeline lance le playbook après la publication des images, uniquement depuis `main` :

```yaml
  deploy:
    needs: build-and-push-docker-image
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v4
      - name: Install Ansible
        run: pipx install --force --include-deps ansible
      - name: Set up SSH key
        run: |
          mkdir -p ~/.ssh
          echo "${{ secrets.SSH_PRIVATE_KEY }}" | tr -d '\r' > ~/.ssh/takima_key
          chmod 400 ~/.ssh/takima_key
      - name: Deploy with Ansible
        working-directory: ./ansible
        env:
          ANSIBLE_HOST_KEY_CHECKING: "False"
        run: |
          echo "${{ secrets.ANSIBLE_VAULT_PASSWORD }}" > .vault_pass
          trap 'rm -f .vault_pass' EXIT
          ansible-playbook -i inventories/setup.yml playbook.yml --vault-password-file .vault_pass
```

La clé SSH du serveur est un secret GitHub, écrite au chemin attendu par l'inventaire. Je l'ai vérifié en modifiant un texte de l'API : après le push, la nouvelle version était en ligne sans aucune action manuelle.

### Question : est-il vraiment sûr de déployer automatiquement chaque nouvelle image ?

Non. Une image boguée ou compromise (compte Docker Hub piraté, dépendance malveillante) partirait directement en production. Pour sécuriser :
- utiliser des **tags versionnés** plutôt que `latest`, pour savoir exactement ce qui est déployé et pouvoir revenir en arrière ;
- ne déployer qu'**après les tests et une quality gate**, ce que `needs` garantit déjà en partie ;
- passer par un environnement de **staging** avant la production ;
- exiger une **validation manuelle** avant la mise en production ;
- prévoir un **rollback** vers la version précédente ;
- **scanner les images** pour détecter les vulnérabilités connues.

## Front-end

Le front-end fourni est une application Vue.js, construite puis servie par nginx. Son adresse d'API est fixée **au moment du build** (`VUE_APP_API_URL` dans `.env.production`). Comme le front et l'API utilisent les mêmes chemins (`/departments`), le proxy les sépare par préfixe :

```apache
<VirtualHost *:${HTTPD_PORT}>
ProxyPreserveHost On
ProxyPass /api/ http://${BACKEND_HOST}:${BACKEND_PORT}/
ProxyPassReverse /api/ http://${BACKEND_HOST}:${BACKEND_PORT}/
ProxyPass / http://${FRONT_HOST}:80/
ProxyPassReverse / http://${FRONT_HOST}:80/
</VirtualHost>
```

J'ai retenu qu'Apache applique la **première** règle qui correspond : `/api/` doit précéder `/`. Le front et l'API étant servis à la même adresse, il n'y a pas de problème de CORS. Quand une URL `/api/...` a renvoyé une erreur 404 de Spring Boot, j'ai compris que la requête atteignait bien le backend, mais avec le préfixe `/api` : le serveur tournait encore avec l'ancienne configuration, en attendant la fin du pipeline.

## Ansible Vault

Pour ne plus passer le mot de passe de la base en clair, je l'ai chiffré avec **Ansible Vault** :

```bash
ansible-vault encrypt group_vars/all/vault.yml
```

- `vault.yml` contient `vault_db_password`, chiffré en AES-256 : le fichier peut être versionné sans risque ;
- `vars.yml` y fait référence avec `db_password: "{{ vault_db_password }}"` ;
- en local, le playbook se lance avec `--ask-vault-pass` ; dans le pipeline, le mot de passe du vault est un secret GitHub, écrit dans un fichier temporaire supprimé à la fin de l'étape ;
- un fichier `.gitattributes` (`*.yml text eol=lf`) empêche git de convertir les fins de ligne sous Windows, ce qui corromprait le fichier chiffré.

Au final, aucun secret n'apparaît en clair, ni dans le dépôt, ni dans les logs.

## Ce que je retiens du TP 3

- Ansible décrit un **état voulu** et l'applique à distance par SSH, de façon idempotente.
- L'inventaire dit **où** et **comment** se connecter ; le playbook dit **quoi** faire ; les rôles rendent ce « quoi » réutilisable.
- Associé à GitHub Actions, il ferme la boucle : du code poussé au serveur en production, sans intervention manuelle.
- Les secrets ont leur place dans un coffre : secrets GitHub pour le pipeline, Ansible Vault pour les fichiers versionnés.

---

# Bilan

Ces trois TP m'ont fait parcourir toute la chaîne DevOps sur une même application : la **conteneuriser** pour qu'elle fonctionne partout de la même façon, la **tester et la publier** automatiquement à chaque modification, puis la **déployer** automatiquement sur un vrai serveur.

Trois idées traversent l'ensemble :
- **tout décrire dans des fichiers versionnés** (Dockerfile, docker-compose, workflow, playbook), pour que l'infrastructure soit reproductible ;
- **séparer la configuration et les secrets du code** (variables d'environnement, `.env`, secrets GitHub, Vault) ;
- **automatiser avec des garde-fous** : on ne publie que si les tests passent, et on ne déploie que ce qui a été publié.

Enfin, les erreurs rencontrées m'ont appris une méthode : lire attentivement les logs, comprendre **où** se situe le problème (ma machine, le conteneur, le réseau, GitHub, le serveur), et tester chaque couche séparément.