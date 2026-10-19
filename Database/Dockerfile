FROM postgres:17.2-alpine

# Les identifiants ne sont plus dans l'image : ils sont passés au lancement avec -e
# Les scripts SQL de ce dossier sont exécutés par ordre alphabétique au premier démarrage
COPY sql/ /docker-entrypoint-initdb.d/
