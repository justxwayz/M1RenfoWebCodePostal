FROM mysql:8.4

ENV TZ=Europe/Paris

# Le dump est exécuté automatiquement à la 1re création du volume
COPY cp.sql /docker-entrypoint-initdb.d/
