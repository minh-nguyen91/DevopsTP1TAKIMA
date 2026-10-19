# TP 01 – Discover Docker

Application 3-tiers entièrement conteneurisée :

```
Navigateur ──▶ HTTP server (Apache) ──▶ Backend API (Spring Boot) ──▶ Database (PostgreSQL)
```

## Lancer le projet

```bash
cp .env.example .env        # puis renseigner les valeurs
docker compose up -d --build
```

L'API est accessible sur http://localhost (ex. http://localhost/departments/IRC/students).

## Structure du dépôt

Un dossier par image :

```
.
├── docker-compose.yml          # orchestration des 3 services
├── .env.example                # modèle des variables d'environnement (le .env n'est pas versionné)
├── Database/
│   ├── Dockerfile
│   └── sql/                    # scripts d'initialisation
├── Backend/
│   ├── hello-world/            # étape « Basics » : Java compilé sur la machine
│   ├── multistage/             # étape « Multistage build » : Java compilé par Docker
│   └── simple-api/             # API Spring Boot
└── HttpServer/
    ├── Dockerfile
    ├── proxy.conf              # configuration du reverse proxy
    └── index.html
```

---

## 1. Database

### Dockerfile

```dockerfile
FROM postgres:17.2-alpine
COPY sql/ /docker-entrypoint-initdb.d/
```

- Image `postgres:17.2-alpine` : image officielle, version fixée, base Alpine légère.
- Les scripts `sql/01-CreateScheme.sql` (création des tables) et `sql/02-InsertData.sql` (données initiales) sont copiés dans `/docker-entrypoint-initdb.d/`. Postgres les exécute **par ordre alphabétique** au **premier démarrage**, quand le dossier de données est vide. D'où la numérotation : les tables doivent exister avant les `INSERT`.
- Le Dockerfile du TP contenait les identifiants en `ENV`. Je les ai retirés : ils sont passés au lancement (voir 1-1).

### Commandes

```bash
# Réseau partagé entre la base, adminer et le backend
docker network create app-network

# Build de l'image
docker build -t tp1-database ./Database

# Lancement : identifiants avec -e, données dans un volume nommé
docker run -d --name tp1-database --network app-network \
  -e POSTGRES_DB=db -e POSTGRES_USER=usr -e POSTGRES_PASSWORD=pwd \
  -v tp1-db-data:/var/lib/postgresql/data tp1-database

# Adminer, accessible sur http://localhost:8090 (serveur : tp1-database)
docker run -d -p 8090:8080 --network app-network --name adminer adminer

# Vérifications
docker ps
docker logs tp1-database
docker exec -it tp1-database psql -U usr -d db -c "SELECT * FROM students;"
```

| Option | Rôle |
|---|---|
| `-d` | Lance le conteneur en arrière-plan |
| `--name` | Nomme le conteneur ; ce nom sert aussi de nom DNS sur le réseau |
| `--network app-network` | Branche le conteneur sur le réseau partagé. On utilise `--network` plutôt que `--link`, qui est déprécié. |
| `-e` | Variable d'environnement |
| `-v tp1-db-data:/var/lib/postgresql/data` | Volume nommé qui contient les données de Postgres |
| `-p 8090:8080` | Publie le port 8080 d'adminer sur le port 8090 de la machine |

La base n'a **pas** de `-p` : elle n'est pas accessible depuis la machine hôte, seulement depuis le réseau Docker.

**Test de persistance :** ajouter un étudiant, `docker rm -f tp1-database`, relancer le même `docker run` avec le même `-v`. L'étudiant est toujours présent.

### 1-1 Pourquoi passer les variables d'environnement avec `-e` plutôt que dans le Dockerfile ?

- **Sécurité** : un mot de passe écrit dans le Dockerfile est enregistré dans l'image. N'importe qui ayant accès à l'image peut le lire avec `docker inspect` ou `docker history`. Il finit aussi dans git et sur Docker Hub si on publie l'image. Docker le signale d'ailleurs avec le warning `SecretsUsedInArgOrEnv`.
- **Réutilisabilité** : la même image peut servir en dev, en test et en prod avec des identifiants différents, sans avoir à la reconstruire.
- **Séparation code / configuration** : c'est le principe de la *12-factor app*. La configuration dépend de l'environnement, pas de l'image.

### 1-2 Pourquoi attacher un volume au conteneur Postgres ?

Un conteneur est éphémère : tout ce qu'il écrit est stocké dans sa couche modifiable, qui est supprimée avec lui (`docker rm`, recréation après une nouvelle version de l'image, crash suivi d'une recréation…). Une base de données doit garder ses données durablement.

Le volume stocke `/var/lib/postgresql/data` en dehors du conteneur, sur l'hôte. On peut donc détruire, recréer ou mettre à jour le conteneur sans perdre les données.

Effet de bord : comme le dossier de données n'est plus vide au redémarrage, les scripts de `/docker-entrypoint-initdb.d` ne sont pas rejoués. C'est ce qu'on veut, sinon les données seraient insérées en double. Pour les rejouer, il faut supprimer le volume (`docker volume rm tp1-db-data`).

### 1-3 Documenter le conteneur de la base

Voir le Dockerfile et les commandes ci-dessus.

---

## 2. Backend API

### 2.1 Hello World (`Backend/hello-world`)

On n'a besoin que d'**exécuter** du Java : l'image contient donc seulement un **JRE**, pas de JDK.

```dockerfile
FROM eclipse-temurin:21-jre-alpine
WORKDIR /usr/src
COPY Main.class .
CMD ["java", "Main"]
```

```bash
cd Backend/hello-world
javac --release 21 Main.java      # compilation sur la machine hôte
docker build -t tp1-hello-world .
docker run --rm tp1-hello-world   # affiche "Hello World!"
```

`--release 21` garantit un bytecode compatible avec le JRE 21 de l'image, même si le JDK local est plus récent. `--rm` supprime le conteneur à la fin de l'exécution.

Limite de cette approche : il faut un JDK installé sur la machine pour compiler.

### 2.2 Multistage build (`Backend/multistage`)

La compilation est faite **par Docker** : aucun JDK n'est nécessaire sur la machine.

```dockerfile
FROM eclipse-temurin:21-jdk-alpine     # stage 0 : contient javac
WORKDIR /usr/src
COPY Main.java .
RUN javac Main.java

FROM eclipse-temurin:21-jre-alpine     # stage 1 : image finale, JRE seul
WORKDIR /usr/src
COPY --from=0 /usr/src/Main.class .
CMD ["java", "Main"]
```

```bash
cd Backend/multistage
docker build -t tp1-hello-multistage .
docker run --rm tp1-hello-multistage
```

### 2.3 Spring Boot simple-api (`Backend/simple-api`)

Projet généré avec Spring Initializr : Maven, Java 21, Spring Boot 3.4.5, Jar, dépendance Spring Web. Spring Initializr ne proposant plus la version 3.4.5, le projet a été généré avec une version plus récente, puis la version du parent a été remise à `3.4.5` dans le `pom.xml` (elle est toujours disponible sur Maven Central).

```dockerfile
# Build stage
FROM eclipse-temurin:21-jdk-alpine AS myapp-build
ENV MYAPP_HOME=/opt/myapp
WORKDIR $MYAPP_HOME

RUN apk add --no-cache maven

COPY pom.xml .
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
cd Backend/simple-api
docker build -t tp1-simple-api .
docker run -d --name tp1-simple-api -p 8080:8080 tp1-simple-api
# http://localhost:8080/?name=Minh  →  {"id":1,"content":"Hello, Minh!"}
```

### 1-4 Pourquoi un multistage build ? Explication de chaque étape

**Pourquoi :** compiler un projet Maven demande un JDK, Maven et des centaines de Mo de dépendances, alors que l'exécuter ne demande qu'un JRE et le `.jar`. Le multistage sépare les deux :

- **image finale plus légère** : elle ne contient que le JRE et le `.jar` ;
- **plus sûre** : ni compilateur, ni Maven, ni code source en production, donc une surface d'attaque réduite ;
- **build reproductible** : aucun outil à installer sur la machine, Docker fait tout.

**Étapes :**

| Instruction | Rôle |
|---|---|
| `FROM eclipse-temurin:21-jdk-alpine AS myapp-build` | Stage de build, à partir d'une image avec le JDK 21. Il est nommé pour pouvoir y faire référence plus tard. |
| `ENV MYAPP_HOME=/opt/myapp` | Définit une variable contenant le dossier de l'application |
| `WORKDIR $MYAPP_HOME` | Crée ce dossier et s'y place pour les instructions suivantes |
| `RUN apk add --no-cache maven` | Installe Maven avec le gestionnaire de paquets d'Alpine, sans garder le cache de l'index |
| `COPY pom.xml .` | Copie la description du projet et de ses dépendances |
| `COPY src ./src` | Copie le code source |
| `RUN mvn package -DskipTests` | Télécharge les dépendances, compile et produit `target/*.jar`, sans lancer les tests |
| `FROM eclipse-temurin:21-jre-alpine` | Stage d'exécution : repart d'une image vierge avec seulement le JRE. Rien du stage précédent n'est conservé. |
| `ENV` / `WORKDIR` | Redéfinis, car chaque stage est indépendant |
| `COPY --from=myapp-build ... myapp.jar` | Récupère **uniquement** le `.jar` du stage de build et lui donne un nom fixe |
| `ENTRYPOINT ["java", "-jar", "myapp.jar"]` | Commande lancée au démarrage du conteneur |

**Amélioration possible :** à chaque modification du code, Maven retélécharge toutes les dépendances. En ajoutant `RUN mvn dependency:go-offline` entre `COPY pom.xml .` et `COPY src ./src`, les dépendances sont placées dans une couche qui reste en cache tant que le `pom.xml` ne change pas.

### 2.4 Backend API connectée à la base

Le dossier `src` et le `pom.xml` ont été remplacés par ceux du dépôt [simple-api-student](https://github.com/takima-training/simple-api-student). Le Dockerfile est inchangé.

Configuration de la connexion dans `src/main/resources/application.yml` :

```yaml
  datasource:
    url: jdbc:postgresql://${DB_HOST:database}:${DB_PORT:5432}/${DB_NAME:db}
    username: ${DB_USER}
    password: ${DB_PASSWORD}
```

- `jdbc:postgresql://hôte:port/base` est le format d'URL du driver PostgreSQL JDBC.
- **L'hôte est le nom du conteneur ou du service de la base** : les conteneurs d'un même réseau Docker se trouvent par leur nom grâce au DNS interne. Ce n'est pas `localhost`, qui désignerait le conteneur de l'API lui-même.
- `${VARIABLE:défaut}` est la syntaxe Spring : la valeur est lue dans une variable d'environnement, avec une valeur par défaut. L'utilisateur et le mot de passe n'ont **pas** de valeur par défaut : aucun secret n'est écrit dans le code ni dans l'image (même logique que 1-1). Sans eux, l'API refuse de démarrer.

```bash
docker run -d --name tp1-simple-api --network app-network \
  -e DB_HOST=tp1-database -e DB_USER=usr -e DB_PASSWORD=pwd \
  -p 8080:8080 tp1-simple-api
# http://localhost:8080/departments/IRC/students
```

Principaux endpoints : `GET /departments`, `GET /departments/{name}`, `GET /departments/{name}/students`, `GET /departments/{name}/count`, `GET|POST /students`, `GET|PUT|DELETE /students/{id}`.

---

## 3. HTTP server

### 3.1 Basics

Image officielle Apache : `httpd:2.4-alpine`. Le fichier `index.html` est copié dans `/usr/local/apache2/htdocs/`, le dossier servi par défaut.

```bash
cd HttpServer
docker build -t tp1-httpd .
docker run -d --name tp1-httpd -p 80:80 tp1-httpd

docker logs tp1-httpd               # démarrage d'Apache et requêtes reçues
docker stats --no-stream tp1-httpd  # consommation CPU / mémoire
docker inspect tp1-httpd            # configuration complète (ports, réseaux, commande)
```

### 3.2 Configuration

Récupération de la configuration par défaut depuis le conteneur en cours d'exécution :

```bash
docker cp tp1-httpd:/usr/local/apache2/conf/httpd.conf ./httpd.conf
# équivalent avec docker exec :
docker exec tp1-httpd cat /usr/local/apache2/conf/httpd.conf > httpd.conf
```

### 3.3 Reverse proxy

Plutôt que de versionner une copie complète de `httpd.conf` (plus de 550 lignes, dont 7 seulement modifiées), la configuration du proxy est isolée dans **`proxy.conf`**. Ce fichier est inclus à la fin du `httpd.conf` par défaut de l'image. Avantage : on ne versionne que ce qu'on a écrit, et une mise à jour de l'image ne réinstalle pas une ancienne configuration.

`proxy.conf` :

```apache
LoadModule proxy_module modules/mod_proxy.so
LoadModule proxy_http_module modules/mod_proxy_http.so

ServerName localhost

<VirtualHost *:80>
ProxyPreserveHost On
ProxyPass / http://${BACKEND_HOST}:${BACKEND_PORT}/
ProxyPassReverse / http://${BACKEND_HOST}:${BACKEND_PORT}/
</VirtualHost>
```

| Directive | Rôle |
|---|---|
| `LoadModule proxy_module` / `proxy_http_module` | Active le moteur de proxy et le relais HTTP |
| `ServerName localhost` | Évite l'avertissement `AH00558` au démarrage |
| `ProxyPreserveHost On` | Transmet au backend l'en-tête `Host` d'origine |
| `ProxyPass / http://backend:8080/` | Toute requête reçue est transmise au backend |
| `ProxyPassReverse` | Réécrit les en-têtes de redirection renvoyés par le backend (ex. `Location`) pour qu'ils pointent vers le proxy |
| `${BACKEND_HOST}` / `${BACKEND_PORT}` | Variables d'environnement lues par Apache, avec des valeurs par défaut définies dans le Dockerfile |

`Dockerfile` :

```dockerfile
FROM httpd:2.4-alpine

ENV BACKEND_HOST=backend \
    BACKEND_PORT=8080

COPY index.html /usr/local/apache2/htdocs/

COPY proxy.conf /usr/local/apache2/conf/extra/proxy.conf
RUN echo "Include conf/extra/proxy.conf" >> /usr/local/apache2/conf/httpd.conf
```

Comme `ProxyPass /` transmet toutes les URL au backend, `index.html` n'est plus servi. Il correspond à l'étape « Basics ».

Vérifications utiles :

```bash
docker exec tp1-httpd httpd -t   # "Syntax OK" : configuration valide
docker exec tp1-httpd httpd -M   # liste des modules : proxy_module, proxy_http_module
```

### 1-5 Pourquoi un reverse proxy ?

- **Point d'entrée unique** : le client ne connaît qu'une adresse. Le backend et la base ne sont pas exposés directement, ce qui réduit la surface d'attaque.
- **Masque l'architecture interne** : on peut déplacer, remplacer ou multiplier les backends sans impact pour le client.
- **Load balancing** : répartition des requêtes entre plusieurs instances du backend.
- **HTTPS / SSL** : les certificats sont gérés à un seul endroit, le backend restant en HTTP.
- **Plusieurs applications derrière une même adresse** : front-end sur `/`, API sur `/api`, etc.
- **Fonctions transverses** : cache, compression, logs centralisés, limitation de débit.

Un reverse proxy ne protège que s'il est le **seul** accès : le backend ne doit pas publier son port (`-p`) sur la machine hôte.

---

## 4. Docker-compose

### Variables d'environnement

Les valeurs sont dans un fichier `.env`, lu automatiquement par docker compose. Ce fichier est dans le `.gitignore` et n'est jamais versionné ; le modèle `.env.example` l'est.

```env
POSTGRES_DB=db
POSTGRES_USER=usr
POSTGRES_PASSWORD=pwd
HTTP_PORT=80
```

### docker-compose.yml

```yaml
services:
    backend:
        build: ./Backend/simple-api
        image: tp1-simple-api
        environment:
            DB_HOST: database
            DB_NAME: ${POSTGRES_DB}
            DB_USER: ${POSTGRES_USER}
            DB_PASSWORD: ${POSTGRES_PASSWORD}
        networks:
            - backend-network
            - frontend-network
        depends_on:
            database:
                condition: service_healthy
        healthcheck:
            test: ["CMD-SHELL", "wget -qO- http://localhost:8080/actuator/health || exit 1"]
            interval: 5s
            timeout: 5s
            retries: 20
        restart: unless-stopped

    database:
        build: ./Database
        image: tp1-database
        environment:
            POSTGRES_DB: ${POSTGRES_DB}
            POSTGRES_USER: ${POSTGRES_USER}
            POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
        volumes:
            - db-data:/var/lib/postgresql/data
        networks:
            - backend-network
        healthcheck:
            test: ["CMD-SHELL", "pg_isready -U $${POSTGRES_USER} -d $${POSTGRES_DB}"]
            interval: 5s
            timeout: 5s
            retries: 10
        restart: unless-stopped

    httpd:
        build: ./HttpServer
        image: tp1-httpd
        environment:
            BACKEND_HOST: backend
            BACKEND_PORT: 8080
        ports:
            - "${HTTP_PORT:-80}:80"
        networks:
            - frontend-network
        depends_on:
            backend:
                condition: service_healthy
        restart: unless-stopped

networks:
    backend-network:
    frontend-network:

volumes:
    db-data:
```

### 1-8 Documentation du fichier docker-compose

| Élément | Choix et justification |
|---|---|
| `build` | Chaque service est construit à partir de son dossier (un dossier par image) |
| `image` | Nomme l'image construite, ce qui sera utile pour la publication |
| `environment` | Toutes les valeurs viennent du `.env` : aucun secret dans le fichier. L'API reçoit les mêmes identifiants que la base, sous ses propres noms de variables (`DB_USER`, `DB_PASSWORD`). |
| `DB_HOST: database` / `BACKEND_HOST: backend` | Avec compose, chaque service est joignable par son **nom de service** |
| `volumes: db-data` | Volume nommé pour la persistance des données de la base |
| **Ports** | **Seul `httpd` publie un port.** Le backend et la base ne sont pas accessibles depuis la machine hôte, comme demandé. |
| **Réseaux** | Ségrégation en deux réseaux : `frontend-network` (httpd ↔ backend) et `backend-network` (backend ↔ database). Seul le backend est sur les deux : **httpd ne peut pas joindre la base**. C'est le principe du moindre privilège : si le service exposé est compromis, la base n'est pas directement atteignable. |
| `healthcheck` | `pg_isready` pour la base, `/actuator/health` (Spring Actuator) pour le backend. Un conteneur est `healthy` quand le service est réellement **prêt**, pas seulement démarré. |
| `depends_on` + `condition: service_healthy` | Ordre de démarrage garanti : database prête → backend prêt → httpd. Sans cette condition, le backend pourrait démarrer avant que Postgres accepte les connexions, et Apache répondrait par des erreurs 503 pendant le démarrage de Spring. |
| `$$` dans le healthcheck | Échappe le `$` pour que la variable soit lue **dans le conteneur** et non remplacée par compose |
| `restart: unless-stopped` | Redémarrage automatique en cas de crash ou de redémarrage de Docker, sauf après un arrêt manuel |

```
                ┌──── frontend-network ────┐┌──── backend-network ────┐
Hôte ──:80────▶ │  httpd  ──────────▶  backend  ──────────▶  database │──▶ volume db-data
                └──────────────────────────┘└─────────────────────────┘
```

### 1-6 Pourquoi docker-compose est-il si important ?

- **L'application entière est décrite dans un seul fichier**, versionné avec le code, au lieu de plusieurs longues commandes `docker run` à retenir et à exécuter dans le bon ordre.
- **Une seule commande** construit, démarre, arrête ou recrée tous les services.
- **Ordre de démarrage et dépendances** gérés automatiquement (`depends_on`, healthchecks).
- **Réseaux, volumes et variables d'environnement** sont créés et reliés automatiquement.
- **Reproductible** : n'importe qui clone le dépôt, crée son `.env` et obtient exactement le même environnement.
- Seuls les services dont la configuration a changé sont recréés.

### 1-7 Commandes docker-compose principales

| Commande | Rôle |
|---|---|
| `docker compose up -d` | Crée et démarre tous les services en arrière-plan |
| `docker compose up -d --build` | Idem, en reconstruisant les images |
| `docker compose down` | Arrête et supprime les conteneurs et les réseaux (les volumes sont conservés) |
| `docker compose down -v` | Idem, en supprimant aussi les volumes (perte des données) |
| `docker compose ps` | État des services (dont `healthy`) |
| `docker compose logs -f <service>` | Logs d'un service en continu |
| `docker compose build` | Construit les images sans démarrer les services |
| `docker compose restart <service>` | Redémarre un service |
| `docker compose stop` / `start` | Arrête / redémarre sans supprimer les conteneurs |
| `docker compose exec <service> <cmd>` | Exécute une commande dans un service (ex. `docker compose exec database psql -U usr -d db`) |
| `docker compose config` | Valide le fichier et affiche la configuration finale, variables remplacées |

### Vérifications

```bash
docker compose ps                                        # 3 services, database et backend "healthy"
curl http://localhost/departments/IRC/students           # OK via le reverse proxy
curl http://localhost:8080/departments                   # échec : backend non exposé
docker compose exec httpd sh -c "nc -z database 5432"    # échec : httpd ne voit pas la base
docker compose down && docker compose up -d              # les données sont conservées (volume)
```

---

## 5. Publish

*À compléter.*

### 1-9 Commandes de publication et images publiées

### 1-10 Pourquoi mettre ses images dans un dépôt en ligne ?
