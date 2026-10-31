# TP2 — Réponses

**Nom Prénom : Nour El Houda Ben Slimene**

## Partie 1 — Découvrir les deux applications

**Q1.** `DATABASE_CLIENT` choisit le moteur (`postgres`). Pour PostgreSQL, Strapi attend `DATABASE_HOST`, `DATABASE_PORT`, `DATABASE_NAME`, `DATABASE_USERNAME` et `DATABASE_PASSWORD` (ou `DATABASE_URL` si elle est définie). Dans Compose, `DATABASE_HOST` doit valoir `postgres`, le nom du service, que Docker résout automatiquement sur le réseau du projet. `localhost` ne marcherait pas : dans un conteneur, il désigne le conteneur lui-même (ici Strapi), où PostgreSQL ne tourne pas.

**Q2.** `backend/.env.example` est un modèle versionné : il liste les variables nécessaires à la configuration de Strapi, sans secret réel, pour qu'on puisse créer son `.env` local. Le vrai `.env` contient des mots de passe et des secrets cryptographiques (`APP_KEYS`, `JWT_SECRET`…). Le commiter les exposerait à toute personne ayant accès au dépôt, et ils resteraient dans l'historique Git même après suppression. On l'ignore donc avec `.gitignore`.

## Partie 2 — PostgreSQL

**Q3.** `docker compose down` arrête et supprime les conteneurs et les réseaux du projet, mais conserve les volumes nommés (dont `pgdata`) : les données PostgreSQL sont retrouvées au prochain `up`. `docker compose down -v` supprime aussi les volumes nommés déclarés par le projet : `pgdata` disparaît et la base repart de zéro. *Remarque : `down -v` n'a pas été exécuté sur ma stack pour préserver mes données ; cette réponse décrit le comportement documenté de Docker.*

**Q4.** Compose interprète les `${...}` du fichier YAML avant de lancer les conteneurs. Le double `$$` échappe cette substitution et passe un seul `$` au shell du conteneur. Ainsi `pg_isready` lit `POSTGRES_USER` et `POSTGRES_DB` dans l'environnement du conteneur PostgreSQL, au moment où le healthcheck s'exécute.

## Partie 3 — pgAdmin

**Q5.** `"Host": "postgres"` désigne PostgreSQL par le nom de son service Compose : pgAdmin le joint sur le réseau Docker `backend` (`localhost` désignerait le conteneur pgAdmin lui-même). `:ro` signifie *read-only* : `servers.json` est monté en lecture seule, le conteneur ne peut pas le modifier. `${PGADMIN_PORT:-5050}` prend la valeur de `PGADMIN_PORT` si elle est définie, sinon `5050` ; ce port de l'hôte est redirigé vers le port 80 de pgAdmin.

## Partie 4 — Strapi

**Q6.** On copie d'abord `package.json` et `package-lock.json`, puis on lance `npm ci`, pour que Docker mette cette étape en cache : tant que ces fichiers ne changent pas, les dépendances ne sont pas réinstallées, quelle que soit la modification du code. Après avoir modifié un fichier de `backend/config/`, les couches jusqu'à `RUN npm ci` sont réutilisées (`CACHED`), puis `COPY . .`, `RUN npm run build` et les étapes suivantes sont rejouées. Si les fichiers de dépendances changent, `npm ci` et tout ce qui suit sont aussi rejoués.

**Q7.** `USER node` fait tourner Strapi avec un utilisateur non privilégié au lieu de `root` : si l'application est compromise, l'attaquant a moins de droits dans le conteneur. `--chown=node:node` donne à cet utilisateur la propriété des fichiers copiés, pour que l'application puisse les lire (et écrire là où c'est nécessaire) sans être `root`.

**Q8.** `depends_on` avec `condition: service_healthy` demande à Compose d'attendre que PostgreSQL passe son healthcheck (`pg_isready`) avant de démarrer Strapi. Sans cela, Compose attendrait seulement que le conteneur soit *démarré* : Strapi pourrait se lancer alors que la base s'initialise encore, échouer à s'y connecter et s'arrêter au démarrage.

**Q9.** La table `articles` contient **4 lignes pour 2 articles publiés** (*Table* et *Souris*), soit deux lignes par article. Elles partagent le même `document_id` : l'une est la version brouillon (`published_at` vide), l'autre la version publiée (`published_at` renseigné). C'est le fonctionnement de Strapi 5 avec *Draft & Publish*.

## Partie 5 — Next.js

**Q10.** Dans le conteneur `frontend`, `localhost` désigne ce conteneur et non Strapi : la connexion échouerait (`ECONNREFUSED`). Il faut utiliser `http://strapi:1337`, où `strapi` est le nom du service Compose, résolu par le DNS du réseau `frontend` partagé avec Strapi. L'appel est fait côté serveur (par Next.js), pas par le navigateur.

**Q11.** Le Dockerfile compte **3 étapes** `FROM` : `deps` (installation des dépendances), `builder` (build Next.js) et `runner` (exécution). L'image finale vient de `runner` : elle contient Node.js, le serveur minimal `.next/standalone`, les fichiers statiques `.next/static` et `public`, mais ni les `node_modules` complets ni les outils de build. Sans `output: "standalone"`, Next.js ne génère pas `.next/standalone` : l'instruction `COPY --from=builder /app/.next/standalone` échoue et le build s'arrête. Taille mesurée (`docker images tp2/frontend`) : **334 Mo** sur disque, **85,6 Mo** compressés.

## Partie 6 — Réseaux et sécurité

**Q12.** `strapi` est dans les deux réseaux car il fait le pont : il parle à PostgreSQL via `backend` et au front via `frontend`. Le front, lui, n'est que dans `frontend` et ne peut donc pas joindre la base (vérifié : `bad address` pour `postgres`). Ne publier aucun port pour `postgres` évite de l'exposer à la machine hôte et au réseau extérieur ; seuls les services du réseau `backend` peuvent y accéder.

**Q13.** Deux mesures pour la production :
1. Remplacer les mots de passe et secrets d'exemple par des valeurs uniques et robustes (`openssl rand -base64 16`), fournis par un gestionnaire de secrets (Docker secrets, Vault, secrets du CI) plutôt que par un fichier.
2. Placer les services accessibles derrière un reverse proxy HTTPS, limiter les ports publiés (par exemple ne pas exposer 1337 et 5050 publiquement) et restreindre l'accès à pgAdmin (VPN, liste d'adresses autorisées).

## Partie 7 — Cycle de vie

**Q14.** `restart: unless-stopped` demande à Docker de redémarrer automatiquement le conteneur s'il s'arrête de façon inattendue (plantage, erreur, redémarrage du démon ou de la machine), sauf s'il a été arrêté explicitement. Après `docker compose stop strapi` ou `kill strapi`, l'arrêt est volontaire : Docker ne le relance pas. Il le relance si le processus plante ou sort en erreur, tant que personne ne l'a arrêté à la main.