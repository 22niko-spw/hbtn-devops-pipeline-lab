# Preuves du pipeline

Observations du 23 septembre 2026. Les résultats ci-dessous proviennent des
commandes et runs exécutés, pas d'une simulation de déploiement.

## Base locale

- `local_verify.txt` : 8/8 tests unitaires, 3/3 tests d'intégration, `/health`
  HTTP 200, puis suppression des conteneurs et volumes du lab.
- `image_verify.txt` : pull et smoke-test de l'image réellement publiée dans
  GHCR, `/health` HTTP 200, puis suppression du conteneur de smoke-test.

## Diagnostic progressif

1. [Run initial](https://github.com/22niko-spw/hbtn-devops-pipeline-lab/actions/runs/35832549884) :
   workflow rejeté avant démarrage, annotation YAML ligne 27. Correction
   `62e873e` : alignement de `steps`.
2. [Run après correction YAML](https://github.com/22niko-spw/hbtn-devops-pipeline-lab/actions/runs/35832605714) :
   `Unknown command: "install-deps"`. Correction `5e45285` : `npm ci`.
3. [Run après correction npm](https://github.com/22niko-spw/hbtn-devops-pipeline-lab/actions/runs/35832705141) :
   tests unitaires verts, intégration rouge : `DATABASE_URL is not set`.
   Correction `baddd8d` : connexion à PostgreSQL via `localhost:5432` sur le runner.
4. [Run des trois corrections](https://github.com/22niko-spw/hbtn-devops-pipeline-lab/actions/runs/35832819146) :
   installation et deux suites réussies.

## Cache, rapports et image

- [Premier run avec cache/JUnit](https://github.com/22niko-spw/hbtn-devops-pipeline-lab/actions/runs/35832930785) :
  lint, 11 tests et upload réussis.
- [Deuxième run, puis publication GHCR](https://github.com/22niko-spw/hbtn-devops-pipeline-lab/actions/runs/35833018448) :
  cache exact restauré, 11 tests réussis, rapports conservés, build et push réussis.
- Clé npm observée :
  `Linux-X64-node20-npm-a60c392c8377c32196f267a9993b9dc2900ee73afaf7d23c3393e86f4ab2fcba`.
- SHA publié : `ecd74ae7585f6fa65f1d301055f37e958df41454`.
- Tags `ghcr.io/22niko-spw/hbtn-devops-pipeline-lab:<SHA>` et `:latest`
  associés au même digest
  `sha256:ae98a77fb7a7439b252a82360a6b8bd9fef05819f1b74f5be82efcd03dccb4db`.

## Audit

`actionlint` 1.7.12 passe. Les sources, tests originaux, Dockerfile et lock file
restent identiques au starter. Les valeurs `pipeline` sont celles de la base
jetable de CI. Le token de publication est automatique ; seul `build` reçoit
`packages: write`. Les valeurs Render sont référencées par secrets et variable.

Le scan des formats de credentials demandé par le sujet ne remplace pas une
revue des permissions ni une rotation lorsqu'un secret réel a été exposé.

## Staging restant à valider

Render n'est pas encore connecté/configuré. Le job `deploy` est implémenté et
échoue explicitement si `RENDER_API_KEY`, `RENDER_SERVICE_ID` ou `STAGING_URL`
manquent. Voir `DEPLOY.md`.

Il reste à obtenir un run complet `test → build → deploy` vert et les deux
réponses HTTP réelles, puis réaliser l'expérience complète du safety gate avec
observation de l'image en staging avant/après. Aucune preuve de staging n'est
revendiquée à ce stade.
