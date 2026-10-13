# TP Authentification Full Stack — Réponses aux questions


## Partie 1 — Prisma et SQLite

**Q1. Pourquoi l'attribut `@unique` sur `email` est-il indispensable ?**

Il garantit, au niveau de la base de données, qu'un email ne correspond qu'à un seul compte : c'est indispensable pour retrouver l'utilisateur de façon fiable à la connexion. Il protège aussi contre les doublons créés par deux requêtes d'inscription simultanées avec le même email (ce qui provoquerait sinon une erreur Prisma `P2002` au moment de la contrainte, détectée au bon endroit).

**Q2. Que contient le dossier `prisma/migrations` et pourquoi doit-il être versionné ?**

Il contient l'historique SQL de tous les changements apportés au schéma (création de tables, ajout de colonnes, etc.), un dossier par migration avec un fichier `migration.sql`. Il doit être versionné dans Git car c'est ce qui permet à toute l'équipe, à la CI et à l'environnement de production de rejouer exactement les mêmes étapes et d'obtenir le même schéma de base de données, de façon reproductible.

**Q3. Quelle limite de SQLite faut-il connaître avant la production avec plusieurs serveurs ?**

SQLite est un simple fichier local : il ne supporte qu'un seul processus écrivain à la fois et ne permet pas un partage fiable entre plusieurs serveurs ou conteneurs (surtout des conteneurs éphémères, où le fichier ne persiste pas). Pour une production avec plusieurs instances, il faut migrer vers un vrai SGBD serveur comme PostgreSQL ou MySQL (il suffit de changer le `provider` dans `schema.prisma`).

## Partie 2 — Validation et session

**Q1. Quel est l'intérêt des options `httpOnly`, `secure` et `sameSite` ?**

- `httpOnly` : rend le cookie illisible depuis JavaScript côté client, ce qui limite fortement le vol de session en cas de faille XSS.
- `secure` : le cookie n'est envoyé que sur une connexion HTTPS, ce qui empêche son interception en clair sur le réseau.
- `sameSite: lax` : le cookie n'est pas envoyé sur la plupart des requêtes initiées depuis un autre site, ce qui atténue les attaques CSRF.

**Q2. Pourquoi ne jamais mettre le mot de passe (même haché) dans le payload du JWT ?**

Le payload d'un JWT n'est qu'encodé en base64, pas chiffré : n'importe qui en possession du token peut le décoder et le lire. La signature garantit seulement l'intégrité du contenu (qu'il n'a pas été modifié), pas sa confidentialité. C'est pourquoi on n'y place qu'un identifiant non sensible, comme `userId`.

**Q3. Que se passe-t-il si `JWT_SECRET` est modifié alors que des utilisateurs sont connectés ?**

Tous les tokens émis avec l'ancien secret deviennent invalides, car leur signature ne peut plus être vérifiée avec le nouveau secret : tous les utilisateurs connectés sont donc déconnectés automatiquement. C'est en pratique un moyen radical de révoquer toutes les sessions actives d'un coup (par exemple en cas de compromission).

## Partie 3 — API d'authentification

**Q1. Pourquoi renvoyer le même message pour un email inconnu et un mauvais mot de passe ?**

Pour éviter l'énumération d'utilisateurs : si les messages étaient différents, un attaquant pourrait tester des emails un par un et déduire lesquels possèdent un compte sur le site, ce qui est une fuite d'information exploitable.

**Q2. Que signifie le paramètre `10` dans `bcrypt.hash` ? Quel effet a son augmentation ?**

C'est le facteur de coût (cost factor) de bcrypt : il correspond à 2^10 itérations de l'algorithme de hachage. Chaque incrément de +1 double le temps de calcul nécessaire. Augmenter cette valeur ralentit donc les attaques par force brute, mais ralentit aussi les connexions légitimes des utilisateurs — une valeur entre 10 et 12 est un bon compromis courant.

**Q3. Quels codes HTTP utilise-t-on (201, 400, 401, 404, 409) et quand ?**

- `201 Created` : l'inscription a réussi, l'utilisateur a été créé.
- `400 Bad Request` : les données envoyées sont invalides (échec de la validation Zod).
- `401 Unauthorized` : l'utilisateur n'est pas authentifié, ou les identifiants fournis sont incorrects.
- `404 Not Found` : l'utilisateur n'a pas été trouvé (par exemple sur `/api/auth/me` si le compte a été supprimé entre-temps).
- `409 Conflict` : l'email utilisé à l'inscription existe déjà en base.

## Partie 4 — Protection des routes (middleware)

**Q1. Pourquoi ne peut-on pas utiliser Prisma directement dans le middleware ?**

Le middleware Next.js s'exécute dans l'Edge Runtime, un environnement restreint qui ne supporte pas toutes les API Node.js utilisées par le moteur natif de Prisma. C'est pourquoi on se limite, dans le middleware, à vérifier la validité du JWT avec la librairie `jose`, qui est compatible avec cet environnement, sans accéder directement à la base de données.

**Q2. Qu'est-ce que la défense en profondeur ? Pourquoi ne pas se fier au seul middleware ?**

La défense en profondeur consiste à empiler plusieurs barrières de sécurité indépendantes plutôt que de reposer sur un seul point de contrôle. Le middleware peut être mal configuré (un `matcher` oublié sur une route sensible) ou, dans certains cas, contourné. C'est pourquoi chaque page sensible (`/dashboard`) et chaque route API sensible (`/api/auth/me`) revérifie elle-même la session côté serveur, au plus près des données, indépendamment du middleware.
