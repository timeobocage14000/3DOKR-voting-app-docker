# Application de vote avec Docker

Application de vote (Cats vs Dogs) mise dans des conteneurs avec Docker Compose.
Chaque module a son Dockerfile et tout se lance avec une seule commande.

## Architecture

- `vote` : page pour voter (Python / Flask), port 8080
- `result` : page des résultats (Node.js), port 8081
- `worker` : prend les votes dans Redis et les écrit dans PostgreSQL (.NET)
- `redis` : file d'attente des votes
- `db` : base PostgreSQL

Trois réseaux :

- `frontend` : `vote` et `result` (les seuls ports publiés)
- `redis_net` (interne) : `vote`, `redis`, `worker`
- `db_net` (interne) : `worker`, `db`, `result`

Comme ça `vote` ne peut pas joindre la base, et `result` ne peut pas joindre Redis.

## Prérequis

Docker avec Docker Compose v2.

## Lancer le projet

Créer le mot de passe de la base (le fichier n'est pas versionné) :

```bash
mkdir -p secrets
openssl rand -base64 24 | tr -d '\n' > secrets/db_password.txt
chmod 644 secrets/db_password.txt
```

Puis lancer :

```bash
docker compose up -d --build --wait
```

- Vote : http://localhost:8080
- Résultats : http://localhost:8081

Pour arrêter : `docker compose down` (les votes sont gardés) ou `docker compose down -v` (supprime aussi les votes).

Attention : PostgreSQL lit le mot de passe seulement à la première création du volume. Si on change `secrets/db_password.txt` après, il faut faire `docker compose down -v` avant de relancer.

## Choix techniques

- Images officielles avec la version complète et la variante Alpine (plus légère)
- Multi-stage pour `vote`, `result` et `worker` : les dépendances ou la compilation sont faites dans la première étape
- Utilisateur non-root dans chaque image avec un UID numérique (vote 10001, result 1000, worker 1654), `USER` placé avant les `COPY` qui utilisent `--chown`
- Un `.dockerignore` par module
- `CMD` et `ENTRYPOINT` en forme exec
- Un healthcheck par service, et `depends_on` avec `service_healthy`
- Volumes nommés `db_data` et `redis_data`
- Le mot de passe de la base est un secret Docker (`POSTGRES_PASSWORD_FILE`), il n'est pas dans le code
- `restart: unless-stopped` et des limites de CPU et de mémoire pour chaque service

## Versions

Les versions d'origine ne sont plus maintenues, je les ai changées (vérifié sur endoflife.date le 11/11/2026) :

| Composant | Avant | Maintenant | Pourquoi |
|-----------|-------|------------|----------|
| Python | 3.11 | 3.14.10 | supporté jusqu'en 2030 |
| Node.js | 18 | 26.13.0 | version LTS |
| .NET | 7 | 10.0 | version LTS |
| PostgreSQL | latest | 18.7 | version figée |
| Redis | latest | 8.10.3 | version figée |

Les paquets NuGet et les dépendances Python sont aussi à jour et figés.
Pour mettre à jour : changer le tag dans le Dockerfile ou le `compose.yaml`, puis `docker compose build --pull` et `docker compose up -d --wait`.

## Modifications du code

- `vote/app.py` : l'adresse de Redis vient de `REDIS_HOST` et le mode debug est désactivé
- `result/server.js` : la connexion PostgreSQL vient de variables d'environnement, le mot de passe est lu dans un fichier
- `result/package-lock.json` : dépendances mises à jour avec `npm update` (failles de sécurité)
- `worker/Program.cs` : même chose pour PostgreSQL et Redis, et chiffrement GSS désactivé (la bibliothèque n'est pas dans Alpine)
- `worker/Worker.csproj` : `net7.0` devient `net10.0`, paquets mis à jour
- `run/*.bash` : supprimés, remplacés par `docker compose`

## Tests faits

- les 5 conteneurs sont `healthy` après `docker compose up -d --build --wait`
- un vote arrive bien dans PostgreSQL et la page des résultats répond
- les votes sont toujours là après `docker compose down` puis `up`
- depuis `vote` on ne peut pas joindre `db`, depuis `result` on ne peut pas joindre `redis`
