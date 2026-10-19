Pourquoi passer les variables d'environnement avec -e plutôt que de les écrire dans le Dockerfile ?

Sécurité : un mot de passe écrit dans le Dockerfile est enregistré dans l'image. N'importe qui ayant accès à l'image peut le lire avec docker inspect ou docker history. Il finit aussi dans git et sur Docker Hub si on publie l'image. Docker le signale d'ailleurs avec le warning SecretsUsedInArgOrEnv.
Réutilisabilité : la même image peut servir en dev, en test et en prod avec des identifiants différents, sans avoir à la reconstruire.
Séparation code / configuration : c'est le principe de la 12-factor app. La configuration dépend de l'environnement, pas de l'image.

Pourquoi attacher un volume au conteneur Postgres ?

Un conteneur est éphémère : tout ce qu'il écrit est stocké dans sa couche modifiable, qui est supprimée avec lui (docker rm, recréation après une nouvelle version de l'image, crash suivi d'une recréation…). Une base de données doit garder ses données durablement.

Le volume stocke /var/lib/postgresql/data en dehors du conteneur, sur l'hôte. On peut donc détruire, recréer ou mettre à jour le conteneur sans perdre les données.

Effet de bord : comme le dossier de données n'est plus vide au redémarrage, les scripts de /docker-entrypoint-initdb.d ne sont pas rejoués. C'est ce qu'on veut, sinon les données seraient insérées en double.


## Database

### Dockerfile
```dockerfile
FROM postgres:17.2-alpine
COPY sql/ /docker-entrypoint-initdb.d/
```
Les scripts `sql/01-CreateScheme.sql` (création des tables) et `sql/02-InsertData.sql`
(données initiales) sont exécutés par ordre alphabétique au premier démarrage.
Les identifiants ne sont pas dans l'image : ils sont passés avec `-e` au lancement.

### Commandes
```bash
# Réseau partagé entre la base, adminer et (plus tard) le backend
docker network create app-network

# Build de l'image
docker build -t tp1-database ./Database

# Lancement de la base : identifiants avec -e, données dans un volume nommé
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
