# brightbean-studio-coolify

Déploie [BrightBean Studio](https://github.com/brightbeanxyz/brightbean-studio) (gestion des réseaux sociaux open source : planification, publication, inbox et statistiques pour Facebook, Instagram, Threads, LinkedIn, TikTok, YouTube, Pinterest, Bluesky, Mastodon, Google Business Profile et DEV.to) sur une instance [Coolify](https://coolify.io) auto-hébergée, en liant directement ce repo public (build pack Docker Compose).

L'amont ne publie pas d'image Docker : les services Django sont construits sur le serveur à partir du `Dockerfile` amont, sur un commit épinglé (`BRIGHTBEAN_REF`). Aucun compte externe n'est nécessaire pour démarrer.

## Architecture

| Service | Rôle |
|---------|------|
| caddy | Point d'entrée public (port 80) : sert les médias publics depuis le volume (requêtes Range, vidéos lisibles dans Safari) et proxifie le reste vers `app` |
| app | Django + gunicorn (port 8000), commande du `Dockerfile` amont |
| worker | `process_tasks` : publication programmée, synchronisation de l'inbox, relances, rappels, nettoyages |
| migrate | Job one-shot à chaque déploiement : migrations, puis enregistrement des tâches récurrentes exécutées par le worker |
| postgres | PostgreSQL 16 |

C'est la topologie de production recommandée par l'amont (`docker-compose.prod.yml`), le TLS en moins : il est terminé par le proxy Coolify.

`SECRET_KEY`, `ENCRYPTION_KEY_SALT` et le mot de passe Postgres sont générés par Coolify (magic vars). `APP_URL` et `ALLOWED_HOSTS` suivent le domaine posé sur le service `caddy`.

## Prérequis

- Une instance [Coolify](https://coolify.io) v4.x (déployé et vérifié sur la 4.3.23)
- Un serveur amd64 ou arm64 avec `git` installé (Docker va chercher le dépôt amont au moment du build)

## Déploiement (1 clic)

### Depuis l'UI

1. UI Coolify > + New > Public Repository
2. URL du repo : `https://github.com/welcoMattic/brightbean-studio-coolify`, branche `main`
3. Build pack : **Docker Compose** (le compose est à `/docker-compose.yaml`, l'emplacement par défaut)
4. Domaine du service `caddy` : par exemple `https://brightbean.example.com`. Laissé vide, Coolify en génère un sur le domaine wildcard du serveur.
5. Deploy. Le premier déploiement (pip, npm, Tailwind) prend environ 5 minutes, les suivants profitent du cache.

### Depuis l'API

Création avec le domaine, puis déploiement :

```bash
curl -X POST https://<votre-coolify>/api/v1/applications/public \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "project_uuid": "<uuid-projet>",
    "server_uuid": "<uuid-serveur>",
    "environment_name": "production",
    "name": "brightbean",
    "git_repository": "https://github.com/welcoMattic/brightbean-studio-coolify",
    "git_branch": "main",
    "build_pack": "dockercompose",
    "ports_exposes": "80",
    "docker_compose_domains": [{"name": "caddy", "domain": "https://brightbean.example.com"}]
  }'

curl -X POST "https://<votre-coolify>/api/v1/deploy?uuid=<uuid-retourné>" \
  -H "Authorization: Bearer <token>"
```

Sans déploiement immédiat, Coolify charge d'abord le compose et génère les secrets : on peut vérifier les variables avant le premier build. Si le serveur a plusieurs destinations Docker, ajouter `destination_uuid`.

Pour changer de domaine ensuite : `PATCH /api/v1/applications/<uuid>` avec le même champ `docker_compose_domains`, puis redéployer.

## Premier compte et administration

1. Ouvrir le domaine et créer un compte : il devient propriétaire de sa propre organisation.
2. Pour l'admin Django (`/admin/`, nécessaire pour saisir des identifiants de plateformes par organisation), créer un superutilisateur depuis Coolify > Terminal, conteneur `app` :

```bash
python manage.py createsuperuser
```

3. Fermer les inscriptions une fois votre compte créé : Coolify > Environment Variables, `DISABLE_SIGNUP=true`, puis Redeploy.

## Variables

Générées par Coolify, rien à fournir :

| Variable | Rôle |
|----------|------|
| `SERVICE_PASSWORD_64_SECRETKEY` | `SECRET_KEY` de Django |
| `SERVICE_PASSWORD_64_ENCRYPTIONSALT` | `ENCRYPTION_KEY_SALT` (dérivation de la clé de chiffrement) |
| `SERVICE_PASSWORD_POSTGRES` | Mot de passe PostgreSQL |
| `SERVICE_URL_CADDY` / `SERVICE_FQDN_CADDY` | URL publique (`APP_URL`) et hôte (`ALLOWED_HOSTS`), dérivés du domaine du service `caddy` |

`SERVICE_FQDN_CADDY_80`, dans le compose, n'est pas une variable : c'est la déclaration qui route le proxy Coolify vers le port 80 de `caddy`.

Toutes les autres sont optionnelles et documentées dans [`.env.example`](./.env.example) : fermeture des inscriptions (`DISABLE_SIGNUP`), version et dépôt construits (`BRIGHTBEAN_REF`, `BRIGHTBEAN_GIT`), SMTP, identifiants des plateformes (`PLATFORM_*`), connexion Google, webhooks, stockage S3, Unsplash, Sentry. Elles se renseignent dans Coolify > Environment Variables, puis Redeploy.

URL de callback OAuth à déclarer chez chaque plateforme : `https://<domaine>/social-accounts/callback/<plateforme>/` (pour TikTok, `social1` au lieu de `tiktok`). Détails par plateforme, webhooks compris, dans la section [Platform Credentials](https://github.com/brightbeanxyz/brightbean-studio#platform-credentials) du README amont.

## Mise à jour

1. Choisir un commit amont sur [brightbeanxyz/brightbean-studio](https://github.com/brightbeanxyz/brightbean-studio/commits/main), SHA complet (40 caractères). L'amont ne publie ni release ni tag.
2. Coolify > Environment Variables : ajouter `BRIGHTBEAN_REF=<sha>` (elle n'apparaît pas d'office, le compose ne l'utilise que dans le contexte de build), ou modifier la valeur par défaut dans le compose.
3. Redeploy : les images sont reconstruites sur ce commit et `migrate` applique les migrations.

Pour tester un correctif pas encore fusionné en amont, construire depuis un fork : `BRIGHTBEAN_GIT=https://github.com/<vous>/brightbean-studio.git` et `BRIGHTBEAN_REF` sur le commit du fork, puis Redeploy. Retirer les deux variables pour revenir à l'amont.

## Pièges connus

- **Ne jamais régénérer ni supprimer `SERVICE_PASSWORD_64_SECRETKEY` et `SERVICE_PASSWORD_64_ENCRYPTIONSALT`** : les identifiants des plateformes et les jetons OAuth sont chiffrés avec une clé dérivée des deux. Les perdre oblige à reconnecter tous les comptes.
- **Inscriptions ouvertes par défaut** : l'amont n'a pas d'interrupteur, quiconque atteint le domaine peut créer un compte et sa propre organisation. `DISABLE_SIGNUP=true` les coupe au niveau de Caddy (403 sur l'inscription par formulaire et par compte tiers). Revers : une invitation envoyée à une adresse sans compte n'aboutit pas tant que c'est fermé, l'invité passant par la même page d'inscription. Rouvrir le temps de son inscription, ou créer son compte depuis `/admin/` (une organisation par défaut est créée avec chaque compte), puis l'inviter. Si la connexion Google est configurée (`GOOGLE_AUTH_*`), une première connexion Google crée encore un compte.
- **Emails écrits dans les logs** tant que `EMAIL_BACKEND_TYPE` ne vaut pas `smtp` : les liens de réinitialisation de mot de passe apparaissent dans les logs du service `app`. En SMTP, seul STARTTLS (port 587) est pris en charge.
- **Changement de domaine** : `APP_URL` et `ALLOWED_HOSTS` se réconcilient au déploiement suivant, il faut donc redéployer. Seul le premier domaine est pris en compte. Mettre aussi à jour les URL de callback chez chaque plateforme.
- **Le conteneur `migrate` s'affiche "exited"** : normal, c'est un job one-shot, exclu du statut Coolify (`exclude_from_hc`).
- **Statut "running (unknown)"** : `caddy` et `worker` n'ont pas de healthcheck (le worker n'expose rien à sonder), c'est attendu.
- **Médias publics** : Caddy sert sans authentification `media_library/`, `avatars/` et `workspaces/icons/`, comme Django (les plateformes récupèrent les médias côté serveur au moment de publier). Les pièces jointes de commentaires restent derrière Django. Caddy refuse aussi tout chemin contenant un segment `..`.
- **Ne retirez pas `SERVICE_FQDN_CADDY_80`** du compose : c'est ce qui déclare le routage du proxy Coolify.
- **Passage au stockage S3** (`STORAGE_BACKEND=s3`) : renseigner les `S3_*` et `CADDY_MEDIA_ROOT=/var/empty`. Les fichiers déjà présents dans `media_data` ne sont pas migrés.

## Volumes persistants

| Volume | Contenu |
|--------|---------|
| `<uuid>_postgres-data` | Base PostgreSQL |
| `<uuid>_media-data` | Médias téléversés (bibliothèque, avatars, pièces jointes) |

Coolify les préfixe avec l'UUID de l'application et remplace les `_` des noms du compose par des `-`. Ils survivent aux redéploiements : à sauvegarder avant toute migration ou suppression.

## Ressources

- Dépôt amont : [brightbeanxyz/brightbean-studio](https://github.com/brightbeanxyz/brightbean-studio) (AGPL-3.0)
- Magic vars et Docker Compose dans Coolify : [coolify.io/docs](https://coolify.io/docs/knowledge-base/docker/compose)
- Licence de ce repo (configuration uniquement) : [MIT](./LICENSE)
