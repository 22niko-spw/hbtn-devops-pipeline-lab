# Déploiement staging sur Render

**État : NON ENCORE CONFIGURÉ.** Le workflow est préparé ; aucun déploiement
Render ni contrôle HTTP sur un staging réel n'a encore été validé. L'URL réelle,
les secrets et les preuves de déploiement restent à renseigner après configuration.

## Préparation

1. Laisser le job `build` publier l'image
   `ghcr.io/<compte>/hbtn-devops-pipeline-lab:<SHA complet>` et son tag `latest`.
   L'image de production est construite pour `linux/amd64`.
2. Un nouveau package GHCR est privé par défaut. Le rendre public pour ce lab,
   ou fournir à Render un credential GitHub avec un PAT autorisé à lire le package
   et doté de `read:packages`. Le `GITHUB_TOKEN` du workflow publie l'image ; il ne
   remplace pas le credential utilisé par Render pour lire une image privée.
3. Dans Render, créer une base PostgreSQL et un Web Service dans la même région.
   Pour le Web Service, choisir **Existing Image** et renseigner l'image GHCR.
   Le service doit utiliser cette image préconstruite, et non construire le dépôt
   Git. Le chemin de l'image configurée doit correspondre à celui du workflow.
4. Configurer le Web Service :
   - `DATABASE_URL` : URL interne de la base PostgreSQL Render ;
   - `PORT=3000`, pour correspondre au `HEALTHCHECK` du Dockerfile ;
   - `NODE_ENV=production` ;
   - Health Check Path : `/health`.
   Garder la commande de démarrage du Dockerfile : les migrations idempotentes
   sont déjà lancées au démarrage. Ne jamais publier `DATABASE_URL` dans le dépôt.
5. Dans GitHub, ouvrir **Settings → Secrets and variables → Actions** et ajouter :
   - secret `RENDER_API_KEY` : clé API créée dans les paramètres du compte Render ;
   - secret `RENDER_SERVICE_ID` : identifiant du Web Service, commençant par `srv-` ;
   - variable `STAGING_URL` : origine HTTPS réelle, sans chemin ni paramètres,
     par exemple `https://nom-du-service.onrender.com`.
6. Relancer le workflow après cette configuration. Lors du premier passage,
   l'image peut être publiée avant que Render et ses secrets soient prêts :
   le job `deploy` échoue alors explicitement sur la configuration manquante.

Les ressources Render et leurs éventuels coûts dépendent du plan choisi.

## Fonctionnement du pipeline

Un push sur `main` exécute `test`, puis `build`, puis `deploy`. Les pull requests
ne publient et ne déploient aucune image. Le job `deploy` reçoit seul les secrets
Render, utilise des permissions GitHub vides et possède une limite de 25 minutes.

Le workflow envoie un POST à
`https://api.render.com/v1/services/<serviceId>/deploys`, avec `imageUrl` fixé
au SHA complet du commit et `clearCache` à `do_not_clear`. `latest` reste un repère ;
le déploiement et le retour arrière utilisent un tag SHA précis.

Le job suit l'ID renvoyé, exige le statut `live` et vérifie que `image.ref`
correspond à l'image demandée. Il effectue au maximum 60 consultations de statut,
espacées de 5 secondes, avec un délai maximal de 10 secondes par requête.
Les échecs de construction, de pré-déploiement ou de mise à jour arrêtent le job.

Le POST n'est pas réessayé automatiquement : une réponse perdue peut masquer un
déploiement déjà créé. Render peut aussi répondre HTTP 202 avec un déploiement
en attente, sans ID exploitable. Dans ce cas, le job échoue explicitement ; vérifier
les déploiements dans Render avant de relancer pour éviter les doublons.

Après `live`, `/health` et `/items` doivent chacun répondre exactement HTTP 200.
Chaque endpoint bénéficie de 12 tentatives, espacées de 5 secondes, avec un délai
maximal de 10 secondes par requête. Les deux endpoints sont contrôlés même si
l'un échoue. `/health` vérifie l'application ; `/items` vérifie aussi PostgreSQL.
Un ancien service encore accessible ne suffit donc pas à valider le nouveau
déploiement. Les journaux n'affichent ni clé API ni corps des réponses Render.

## Vérification indépendante et preuves

Après le succès du job, contrôler les codes HTTP depuis une autre session :

```bash
export STAGING_URL='https://nom-du-service.onrender.com'
for endpoint in health items; do
  code=$(curl --silent --show-error --output /dev/null \
    --write-out '%{http_code}' --connect-timeout 5 --max-time 30 \
    "${STAGING_URL%/}/$endpoint") || exit 1
  printf '/%s: HTTP %s\n' "$endpoint" "$code"
  test "$code" = 200 || exit 1
done
```

Conserver les preuves réelles : URL du run Actions réussi, SHA du commit, tags
GHCR publiés, ID du déploiement Render et réponses HTTP observées. À ce stade,
ces preuves de staging ne sont pas encore disponibles.

## Retour arrière

Choisir le SHA d'une version précédemment validée et encore disponible dans GHCR.
Définir `RENDER_API_KEY` et `RENDER_SERVICE_ID` dans la session locale sans les
écrire dans un fichier suivi par Git, puis envoyer :

```bash
IMAGE='ghcr.io/<compte>/hbtn-devops-pipeline-lab:<SHA précédent>'
payload=$(jq -nc --arg image "$IMAGE" \
  '{imageUrl: $image, clearCache: "do_not_clear"}')
response=$(curl --fail --silent --show-error \
  --connect-timeout 5 --max-time 30 --request POST \
  --header "Authorization: Bearer $RENDER_API_KEY" \
  --header 'Content-Type: application/json' \
  --data "$payload" \
  "https://api.render.com/v1/services/$RENDER_SERVICE_ID/deploys") || exit 1
DEPLOY_ID=$(printf '%s' "$response" | jq -er '.id') || exit 1
printf 'Déploiement de retour arrière : %s\n' "$DEPLOY_ID"
```

Dans Render, suivre **cet ID** jusqu'à `live` et confirmer l'image SHA choisie,
puis refaire la vérification HTTP indépendante des deux endpoints. Si le POST
échoue ou ne renvoie aucun ID, inspecter Render avant de réessayer. Ce retour
arrière restaure l'application, pas les données PostgreSQL. Un prochain push
sur `main` déclenche de nouveau la livraison du commit courant.

## Nettoyage après validation

- Arrêter la stack locale avec `docker compose down -v` ; cela supprime aussi
  les données PostgreSQL locales du lab.
- Sauvegarder les données nécessaires avant de supprimer les ressources Render
  de démonstration devenues inutiles.
- Retirer les secrets et la variable GitHub du lab, puis révoquer la clé API
  dédiée et, le cas échéant, le credential de registre devenu inutile.
- Conserver les images SHA actives ou utiles au retour arrière : Render doit
  pouvoir les récupérer de nouveau lors d'un déploiement ou d'un redémarrage.

## Références

- [Déclencher un déploiement Render](https://api-docs.render.com/reference/create-deploy)
- [Consulter un déploiement précis](https://api-docs.render.com/reference/retrieve-deploy)
- [Schéma OpenAPI et statuts Render](https://api-docs.render.com/openapi/render-public-api-1.json)
- [Déployer une image préconstruite sur Render](https://render.com/docs/deploying-an-image)
- [Utiliser GitHub Container Registry](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry)
