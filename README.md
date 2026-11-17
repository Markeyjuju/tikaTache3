# Tâche 3 IFT3913

[Répertoire original](https://github.com/umontreal-diro/tika)

## Workflow Github Actions

J’ai commencé par me baser sur un fichier de configuration obtenu sur le [site de Semgrep](https://docs.semgrep.dev/semgrep-ci/sample-ci-configs#sample-github-actions-configuration-file) :

```
# Name of this GitHub Actions workflow.
name: Semgrep
on:
  # Scan changed files in PRs (diff-aware scanning):
  pull_request: {}
  # Scan on-demand through GitHub Actions interface:
  workflow_dispatch: {}
  # Scan mainline branches if there are changes to .github/workflows/semgrep.yml:
  push:
    branches:
      - main
      - master
    paths:
      - .github/workflows/semgrep.yml
  # Schedule the CI job (this method uses cron syntax):
  schedule:
    - cron: '20 17 * * *' # Sets Semgrep to scan every day at 17:20 UTC.
    # It is recommended to change the schedule to a random time.
permissions:
  contents: read
jobs:
  semgrep:
    # User definable name of this GitHub Actions job.
    name: semgrep/ci
    # If you are self-hosting, change the following `runs-on` value:
    runs-on: ubuntu-latest
    container:
      # A Docker image with Semgrep installed. Do not change this.
      image: semgrep/semgrep
    # Skip any PR created by dependabot to avoid permission issues:
    if: (github.actor != 'dependabot[bot]')
    steps:
      # Fetch project source with GitHub Actions Checkout. Use either v3 or v4.
      - uses: actions/checkout@v6
      # Run the "semgrep ci" command on the command line of the docker image.
      - run: semgrep ci
        env:
          # Connect to Semgrep AppSec Platform through your SEMGREP_APP_TOKEN.
          # Generate a token from Semgrep AppSec Platform > Settings
          # and add it to your GitHub secrets.
          SEMGREP_APP_TOKEN: ${{ secrets.SEMGREP_APP_TOKEN }}
```

J'ai commencé par enlever les "Triggers" d'action pour les pull requests, l'activation manuelle et à intervalle de temps. Je crois que l'analyse statique du code n'est nécessaire que lorsqu'on ajoute ou modifie le code et que ce serait suffisant de verifier les modifications à chaque push.

J'ai changé :
```
	paths:
      - .github/workflows/semgrep.yml
```
À :
```
    paths-ignore:
      - 'docs/**'
```
Car je ne veux pas que l'action se trigger seulement quand le fichier du github action change. 
Je veux trigger l'action lorsque n'importe quoi dans le répertoire change mis à part le dossier docs car les autres fichiers Github actions ignorent aussi ce dossier. Ce qui fait du sens aussi étant donné que ce dossier est le résultat du build des autres fichiers, il est plus pertinent d'évaluer la source.

J'ai enlevé : 
```
    branches:
      - main
      - master
```
Car je veux que l'action s'exécute peut importe la branche.

J'ai enlevé ce morceau de code :
```

    # Skip any PR created by dependabot to avoid permission issues:
    if: (github.actor != 'dependabot[bot]')

```
Car étant donné que l'action ne se trigger seulement lorsque qu'on push, il est inutile de verifier si l'action provient de Dependabot car il agit lors des pull requests (PR).

J'ai remplacé : 
```
    - run: semgrep ci
```
Par :
```
	- run: semgrep scan
```
Car la commande ci nécessite un compte et de la configuration mais je voulais quelque chose de plus simple et indépendant d'une plateforme web. La commande scan s'exécute directement avec la ligne de commande et est explicite dans le fichier de l'action.

La commande scan s'exécute plus en détail comme ceci :
```
        with: 
            fetch-depth: 0

      - name: Nouvelle Branche
        if: github.event.created == true
        run: >
          semgrep scan --error --metrics off
          --config p/java --config p/owasp-top-ten
          --exclude '**/src/test/**'
          --baseline-commit origin/main
    
      - name: Branche deja cree sur github
        if: github.event.created == false
        run: >
          semgrep scan --error --metrics off
          --config p/java --config p/owasp-top-ten
          --exclude '**/src/test/**'
          --baseline-commit "${{ github.event.before }}"
```
J'utilise fetch-depth: 0 car je veux que Semgrep ait accès à l'historique complet pour pouvoir possiblement comparer les changements avec un commit sur une autre branche.

Si on fait un push vers une branche qui n'existe pas dans le répertoire Github, on ne pourra pas comparer avec un ancien commit de cette branche, j'ai décidé dans ce cas de comparer avec le dernier commit de la branche main sur Github. 
Pour ce faire, j'utilise le fait que la valeur de github.event.creted sera égal à "true". 

Si cette valeur est true le "--baseline-commit" sera :
```
origin/main 
```
Sinon :
```
"${{ github.event.before }}"
```

Le reste de la commande est pareil, --error pour arrêter l'action en cas de détection d'erreur, --metrics off pour ne pas envoyer de données à Semgrep, les --configs pour le language Java et les top 10 catégories de vulnérabilités OWASP et exclure le dossier test car il contient du texte erroné par exprès.

Une fois que j'ai push ces changements j'ai eu une vulnérabilité détectée par Semgrep concernant la version actions/checkout@v6. 
```
┌────────────────┐

│ 1 Code Finding │

└────────────────┘

.github/workflows/semgrep.yml

❯❱ yaml.github-actions.security.github-actions-mutable-action-tag.github-actions-mutable-action-tag

❰❰ Blocking ❱❱

GitHub Actions step uses a mutable tag or branch reference. Tags and branch names can be silently

repointed by the action owner, enabling supply-chain attacks — as seen in the trivy-action and kics-

github-action compromises. Pin the reference to a full 40-character commit SHA instead, e.g. `uses:

actions/checkout@8ade135a41bc03ea155e62e844d188df1ea18608`.

Details: [https://sg.run/2LgAL](https://sg.run/2LgAL)

28┆ - uses: actions/checkout@v6
```
J'ai donc changé la version de v6 à la chaîne de charactères :
d23441a48e516b6c34aea4fa41551a30e30af803
qui corresponds à la version 6.1.0, la dernière version 6 disponible et la release officielle de v6.

---

### ***Reflexions***

Je crois que mon implémentation inspecte bien les changements mais il faudrait quand meme verrifier ce qu'il y a de déjà écrit avec Semgrep... Je ne crois pas que ça devrait être fait via des workflows mais bien à la ligne de commande localement! 

Je crois qu'ensuite une autre action qui verrifie le code du répertoire au complet périodiquement à chaque mois disons pourrait être pertinent car les configurations des règles de Semgrep/Java/OWASP pourraient changer avec le temps.