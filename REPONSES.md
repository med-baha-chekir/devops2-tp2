# REPONSES : TP2 Docker Compose

**Module :** DevOps 2 (ECUE733)

---

## Partie 1 : Découvrir les deux applications

**Q1. Variables d'environnement attendues par Strapi pour PostgreSQL, valeur de `DATABASE_HOST`**

Dans `backend/config/database.js` (section `postgres`), Strapi lit :

- `DATABASE_CLIENT` : type de base (`postgres` ici, `sqlite` par défaut) ;
- `DATABASE_HOST` : adresse du serveur (défaut `localhost`) ;
- `DATABASE_PORT` : port (défaut `5432`) ;
- `DATABASE_NAME` : nom de la base (défaut `strapi`) ;
- `DATABASE_USERNAME` : utilisateur (défaut `strapi`) ;
- `DATABASE_PASSWORD` : mot de passe (défaut `strapi`) ;
- optionnelles : `DATABASE_URL`, `DATABASE_SSL` (et ses variantes), `DATABASE_SCHEMA` (défaut `public`), `DATABASE_POOL_MIN` / `DATABASE_POOL_MAX`, `DATABASE_CONNECTION_TIMEOUT`.

`DATABASE_HOST` doit valoir **`postgres`**, le nom du service dans `docker-compose.yml`.

Pourquoi pas `localhost` ? Dans un conteneur, `localhost` désigne le conteneur lui-même, pas la machine hôte ni les autres conteneurs. Strapi chercherait PostgreSQL sur son propre port 5432, où rien n'écoute, et la connexion échouerait. Docker Compose fournit un DNS interne qui résout le nom du service (`postgres`) vers l'adresse IP du conteneur PostgreSQL.

**Q2. Rôle de `backend/.env.example` et pourquoi ne pas commiter `.env`**

`.env.example` est un fichier modèle qui liste les variables d'environnement nécessaires à l'application (accès à la base, clés secrètes comme `APP_KEYS`, `JWT_SECRET`, etc.) avec des valeurs d'exemple. Il est commité pour documenter la configuration : un nouveau développeur le copie (`cp .env.example .env`) puis adapte les valeurs.

Le vrai `.env` contient les vrais secrets (mots de passe, clés de signature). Il ne doit jamais être commité car :

- les secrets seraient exposés à toute personne ayant accès au dépôt, surtout s'il est public ;
- ils resteraient dans l'historique Git même après suppression du fichier, ce qui obligerait à tout régénérer ;
- chaque environnement (dev, test, production) a ses propres valeurs.

C'est pourquoi `.env` est dans le `.gitignore`, alors que `.env.example` (sans secrets réels) est versionné.

---

## Partie 2 : PostgreSQL

**Q3. Différence entre `docker compose down` et `docker compose down -v`**

`docker compose down` arrête et supprime les conteneurs et les réseaux de la stack, mais **conserve les volumes** : les données de PostgreSQL (volume `pgdata`) survivent et sont retrouvées au prochain `up`. `docker compose down -v` supprime en plus les **volumes nommés** déclarés dans le fichier : toutes les données sont perdues et la base repart de zéro. Test effectué avec `docker volume ls` : après `down`, `tp2_pgdata` est toujours présent ; après `down -v`, il a disparu.

**Q4. Pourquoi `$${POSTGRES_USER}` avec deux `$` dans le healthcheck**

Compose interprète lui-même `${...}` et le remplace par la valeur du `.env` à la lecture du fichier. Avec `$$`, le `$` est échappé : Compose écrit un seul `$` dans la commande, et c'est le **shell du conteneur** (`CMD-SHELL`) qui évalue `${POSTGRES_USER}` à partir des variables d'environnement du conteneur. Avec un seul `$`, la substitution serait faite par Compose sur la machine hôte au lieu d'être laissée au conteneur. On le vérifie avec `docker compose config` : les autres variables sont remplacées par leurs valeurs, alors que celle du healthcheck garde sa forme `$${POSTGRES_USER}`.

---

## Partie 3 : pgAdmin

**Q5. `"Host": "postgres"`, `:ro`, `${PGADMIN_PORT:-5050}`**

- **`"Host": "postgres"`** : pgAdmin tourne dans son propre conteneur, donc `localhost` y désignerait pgAdmin lui-même. `postgres` est le nom du service Compose, que le DNS interne de Docker résout vers l'IP du conteneur PostgreSQL.
- **`:ro`** : *read-only*. Le fichier `servers.json` est monté en lecture seule : pgAdmin peut le lire mais pas le modifier, ce qui protège le fichier de l'hôte.
- **`${PGADMIN_PORT:-5050}`** : syntaxe de valeur par défaut. Compose utilise la valeur de `PGADMIN_PORT` si elle est définie et non vide, sinon `5050`. Le port 5050 de l'hôte est redirigé vers le port 80 du conteneur.

---

## Partie 4 : Strapi

**Q6. Pourquoi copier `package*.json` et lancer `npm ci` avant le reste du code**

Docker construit l'image couche par couche et met en cache chaque instruction tant que ses entrées n'ont pas changé. En copiant d'abord seulement `package.json` et `package-lock.json`, la couche `npm ci` (longue) n'est invalidée que si les dépendances changent. Si on copiait tout le code avant, la moindre modification d'un fichier source invaliderait le cache et réinstallerait toutes les dépendances à chaque build.

Après modification d'un fichier de `backend/config/` : les étapes jusqu'à `RUN npm ci` restent en `CACHED`, et sont rejouées `COPY . .` puis `RUN npm run build` (ainsi que la copie vers le stage final).

**Q7. Rôle de `USER node` et `--chown=node:node`**

- **`USER node`** : le processus s'exécute avec l'utilisateur non-root `node` au lieu de `root`. Si l'application est compromise, l'attaquant a des droits limités, ce qui réduit l'impact (principe du moindre privilège).
- **`--chown=node:node`** : les fichiers copiés appartiennent par défaut à `root`. Avec `--chown`, ils appartiennent à `node`, qui peut donc lire et écrire dedans (cache, `.tmp`, uploads). Sans cela, Strapi tournant en `node` aurait des erreurs de permissions.

**Q8. Rôle de `depends_on: condition: service_healthy`**

Compose attend que le `healthcheck` de `postgres` passe à `healthy` avant de démarrer `strapi`. Sans cette condition, Compose démarre `strapi` dès que le conteneur `postgres` est *lancé*, même si PostgreSQL n'est pas encore prêt à accepter des connexions. Strapi tenterait alors de se connecter trop tôt, échouerait et planterait (ou redémarrerait en boucle) au démarrage.

**Q9. Lignes de la table `articles` pour un article publié**

Un article publié occupe **2 lignes** dans la table `articles`. Strapi 5 utilise le système *Draft & Publish* : chaque document a une version brouillon (`published_at` NULL) et une version publiée (`published_at` renseigné). Les deux lignes partagent le même `document_id` mais ont des `id` différents. Dans mon cas, j'ai créé 2 articles (`test1` et `test2`), publiés tous les deux, d'où 4 lignes au total (voir la capture `captures/capture-pgadmin.png`).

---

## Partie 5 : Next.js

**Q10. Pourquoi `STRAPI_URL=http://localhost:1337` ne fonctionnerait pas dans le conteneur**

Dans le conteneur `frontend`, `localhost` désigne le conteneur lui-même, pas Strapi. Le serveur Next.js chercherait Strapi sur son propre port 1337, où rien n'écoute : la connexion serait refusée. Il faut utiliser le nom du service, `http://strapi:1337`, que le DNS interne de Compose résout vers l'IP du conteneur Strapi.

**Q11. Étapes du Dockerfile, contenu de l'image finale, sans `standalone`, taille**

- **Nombre d'étapes :** 3 `FROM` (`deps`, `builder`, `runner`).
- **Contenu de l'image finale :** uniquement le stage `runner` : le dossier `public`, le serveur minimal généré dans `.next/standalone` (avec `server.js` et seulement les dépendances nécessaires) et `.next/static`. Les `node_modules` complets, le code source et les outils de build restent dans les stages précédents, qui sont abandonnés.
- **Sans `output: "standalone"` :** le build échoue à l'étape `COPY --from=builder --chown=node:node /app/.next/standalone ./` avec l'erreur `"/app/.next/standalone": not found`. Le `npm run build` réussit, mais sans cette option Next.js ne génère pas le dossier `.next/standalone` (serveur Node minimal avec `server.js`), donc il n'y a rien à copier dans l'image finale.
- **Taille de l'image finale :** `docker images tp2/frontend` affiche 334 MB d'espace disque (`DISK USAGE`) et 85,6 MB de contenu (`CONTENT SIZE`). L'image est légère grâce au multi-stage et au mode standalone.

---

## Partie 6 : Réseaux et sécurité

**Q12. Pourquoi `strapi` est dans les deux réseaux, intérêt de ne publier aucun port pour `postgres`**

`strapi` fait le pont : il doit joindre `postgres` (réseau `backend`) et être joignable par `frontend` (réseau `frontend`). Le front n'accède aux données que via l'API Strapi, jamais directement à la base. Vérification : depuis `frontend`, `nc -zv -w2 postgres 5432` renvoie `nc: bad address 'postgres'` (isolation effective), alors que `wget -qO- http://strapi:1337/_health` répond `OK`. `docker network ls` montre bien `tp2_backend` et `tp2_frontend`.

Ne publier aucun port pour `postgres` le rend inaccessible depuis la machine hôte et depuis l'extérieur : seuls les conteneurs du réseau `backend` peuvent le joindre. Cela réduit la surface d'attaque (pas de scan ni de brute-force du mot de passe depuis l'extérieur).

**Q13. Deux autres mesures de durcissement en production**

1. **Gérer les secrets proprement :** régénérer tous les secrets (`openssl rand -base64 16`) et utiliser Docker secrets ou un gestionnaire de secrets plutôt qu'un fichier `.env` en clair ; ne pas garder les mots de passe d'exemple du TP.
2. **Réduire l'exposition :** ne pas publier pgAdmin (ou le retirer en production) et placer un reverse proxy (Nginx, Traefik) avec HTTPS devant le front et Strapi au lieu d'exposer directement les ports.

On peut aussi ajouter `read_only: true`, `cap_drop: [ALL]`, des limites de ressources (`mem_limit`, `cpus`), épingler précisément les versions d'images et scanner les images (Trivy).

---

## Partie 7 : Cycle de vie

**Q14. Rôle de `restart: unless-stopped`**

`restart: unless-stopped` demande à Docker de redémarrer automatiquement le conteneur lorsqu'il s'arrête (crash, code de sortie non nul, redémarrage du démon Docker ou de la machine), **sauf** s'il a été arrêté manuellement. Après `docker compose stop strapi` (ou `kill`), Docker considère que c'est une décision explicite de l'utilisateur et ne le relance pas (`docker compose ps -a` le montre en état `Exited`). Il le redémarre automatiquement en cas de crash du processus ou de redémarrage du démon Docker ou de la machine, tant qu'il n'a pas été arrêté manuellement.

**Observations des tests de la partie 7**

- **Panne :** après `docker compose stop strapi`, le front affiche « Impossible de joindre Strapi : TypeError: fetch failed ». Le serveur Next.js appelle Strapi à chaque requête (rendu à la demande) et le conteneur n'existe plus à l'adresse `strapi:1337`. Le front reste disponible avec un message d'erreur propre. Après `docker compose start strapi`, les articles réapparaissent sans redémarrer le front.
- **Persistance :** après `docker compose down` puis `docker compose up -d`, les articles sont conservés (volume `pgdata`). Après `docker compose down -v`, la base est vide. <!-- À CONFIRMER avec ton test réel -->
- **Sauvegarde :** `docker compose exec postgres pg_dump -U strapi strapi > backup.sql` produit un dump SQL de la base.
- **Rebuild ciblé :** `docker compose up -d --build frontend` ne reconstruit que le service `frontend`.

---

## Captures d'écran

- `captures/capture-front.png` : le front affichant les articles ;
- `captures/capture-pgadmin.png` : pgAdmin montrant la table `articles`.