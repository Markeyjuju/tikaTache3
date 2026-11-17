# Tâche 3 IFT3913

[Répertoire original](https://github.com/umontreal-diro/tika)

## Workflow Github Actions

J’ai commencé par me baser sur un [fichier de configuration](https://docs.semgrep.dev/semgrep-ci/sample-ci-configs#sample-github-actions-configuration-file) obtenu sur le site de Semgrep pour faire le fichier [semgrepPush](https://github.com/Markeyjuju/tikaTache3/blob/main/.github/workflows/semgrepPush.yml)

J'ai commencé par enlever les "Triggers" d'action pour les pull requests, l'activation manuelle et à intervalle de temps. Je crois que l'analyse statique du code qui ***diffère*** n'est nécessaire que lorsqu'on ajoute ou modifie le code et que ce serait suffisant de verifier les modifications à chaque push.

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

### ***Ajout deuxième fichier d'action***

Je crois qu'une autre action qui verrifie le code du répertoire au complet périodiquement à chaque mois disons pourrait être pertinente car les configurations des règles de Semgrep/Java/OWASP pourraient changer avec le temps. 
Je pense aussi qu'il serait pertinent d'ajouter une action lors des PR car lorsqu'on fait une branche l'action pour les Push compare avec la dernière version de main seulement. Elle ne compare pas avec le commit du quel on a créé la branche originalement ni si on a fait une branche à partir d'une autre branche que main.

J'ai donc fait un nouveau [fichier](https://github.com/Markeyjuju/tikaTache3/blob/main/.github/workflows/semgrepPR_Cron_Man.yml) de workflow pour une analyse complète du répertoire lors de pull requests, mensuellement et manuellement.

J'ai mis :
```
on:
    pull_request:
        branches:
            - main
        paths-ignore:
            - 'docs/**'
    schedule:
        - cron: '0 12 1 * *'
    workflow_dispatch:
```
Pour déclancher l'action lors des pull requests vers la branche main, chaque premier du mois à midi et la possibilité de déclancher manuellement. 

Puisque je n'analyse pas un diff mais bien le répertoire au complet je n'ai pas besoin d'ajouter de baseline. J'ai aussi ajouté une config pour security-audit à la demande de mon coéquipier pour mieux détecter les vulnérabilités. Et je n'ai pas mis --error car on veut principalement juste soulever des erreurs quand il y a nouvelle vulnérabilité et non resignaler toujours les mêmes lors des scans complets.
```
      - run: >
          semgrep scan --metrics off
          --config p/java --config p/owasp-top-ten --config p/security-audit
          --exclude '**/src/test/**'
```

---
Vulnérabilités

1. ExternalParser.java

J’ai choisi ce fichier, car le parsing est un mécanisme frais dans ma mémoire grâce à mes choix de cours actuels. Le choix plus précis de ExternalParser.java se résume au fait qu’elle lance déjà des processus avec ProcessBuilder. L’injection de commande se fait donc relativement facilement. La vulnérabilité présente est de type “CWE-78: Improper Neutralization of Special Elements used in an OS Command ('OS Command Injection')”. La vulnérabilité a lieu, car resourceName, une variable qui est peut être manipulée par un utilisateur, peut contenir des commandes avec accès direct à l’OS. Le rapport Semgrep passe de 43 vulnérabilité dans notre baseline à 44, la seule nouvelle étant la suivante: java.lang.security.audit.command-injection-process-builder.command-injection-process-builder A formatted or concatenated string was detected as input to a ProcessBuilder call. This is dangerous if a variable is controlled by user input and could result in a command injection. Ligne : new ProcessBuilder("sh", "-c", shellLine).

2. NetworkParser.java

J’ai choisi ce fichier, car c’est une classe qui s’occupe d’ouvrir des connexions réseau vers des URI externes. Si on veut s’introduire de manière malveillante dans un réseau, c’est par cette classe. J’y vois également des liens reliés au cours de technologies de l’internet que j’ai pris cet été, un cours focusant sur la connexion client serveur d’API. La vulnérabilité ajoutée est de type “validation de certificat désactivée (CWE-295)”. Elle a lieu parce que le X509TrustManager implémenter n’effectue tout simplement pas de vérifications dans checkServerTrusted. Ceci veut dire que le Trust Manager du code ne Manage pas le Trust des serveurs avec lequel il communique. Il prétend faire son travail sans vraiment le faire. Tout serveur est accepté, même si il est sur la top liste du FBI. L’analyse Semgrep passe de 43 vulnérabilité dans notre nouvelle baseline, à 45 vulnérabilités. java.lang.security.audit.weak-ssl-context.weak-ssl-context (ligne 91\) java.lang.security.audit.crypto.ssl.insecure-trust-manager.insecure-trust-manager (lignes 93-94).

3. CryptoParser.java

J’ai choisi ce fichier, car je m'intéressais sur les générateurs d’aléatoires dans le code. Cette classe en est responsable. Elle le fait par SecureRandom. C’est la cible à simplifier. La vulnérabilité est de type “CWE-330: Use of Insufficiently Random Values”. Elle arrive lorsque le contexte ne définit pas de contexte. Le get va donc échouer et va se mettre par défaut sur un “tika-default-seed”. C’est une seed très prévisible pour n’importe qui examinant le code ou simplement connaissant tika. Pour le rapport Semgrep, une erreur importante m’empêche de relever cette faille dans le Semgrep. Malgré plusieurs heures de débogage, il semble que ma compréhension de cette règle ou de son implémentation me fasse défaut. Je n’ai pas réussi à la faire flag par le Semgrep. Du peu que je comprenne, l’erreur pourrait seulement être levé avec scala, mais pas en java. Voici le lien utilisé pour cette vulnérabilité: https\://semgrep.dev/r?q=scala.lang.security.audit.insecure-random.insecure-random

4. XmlRootExtractor.java

J’ai choisi ce fichier, car l’extraction de XML sonnait dangereux. Ce sont deux mots que j’associe avec des failles de sécurité, alors de les voir ensemble m’a donné des étincelles aux yeux. La classe analyse du XML non fiable pour extraire certains de ses éléments. Par là, il est évident qu’on peut introduire du XML malveillant. La vulnérabilité est de type “CWE-611: Improper Restriction of XML External Entity Reference”. Elle occurre, car nous contournons le mécanisme renforcé de Tika contre ce type de vulnérabilité. On construit directement notre propre SAXParserFactory, sans désactiver les DOCTYPE externes. C’est de cette façon que des XML malicieux peuvent déclarer des entités externes. Par là, il est ensuite possible d’accéder à l’OS. Le rapport Semgrep la lève: java.lang.security.audit.xxe.saxparserfactory-disallow-doctype-decl-missing.saxparserfactory-disallow-doctype-decl-missing DOCTYPE declarations are enabled for this SAXParserFactory. This is vulnerable to XML external entity attacks

5. InputStreamDigester.java

J’ai choisi ce fichier, car je ne comprenais honnêtement pas ce qu’y se passait à l'intérieur. L'algorithme utilisé est toujours hors de ma connaissance, mais j’ai compris pourquoi il est important. La vulnérabilité est de type “Use of a broken or risky cryptographic algorithm CWE-327”. Si l’algorithm demandé n’est pas disponible, on va utiliser MD5 à la place de lancer une erreure. Si la configuration e base veut utiliser un autre algorithm, il risque quand même de se retrouver avec des traces de MD5. Le rapport Semgrep la lève: java.lang.security.audit.crypto.use-of-md5.use-of-md5 Detected MD5 hash algorithm which is considered insecure. MD5 is not collision resistant and is therefore not suitable as a cryptographic signature. Use HMAC instead. Ligne : return MessageDigest.getInstance("MD5"); 

6. ConfigDeserializer.java

J’ai choisi ce fichier, car la désérialisation est un mécanisme qui m’intrigue. Je suis fasciné par la capacité d’encoder de l’information sous une telle forme. Ici, on désérialise du JSON externe. La vulnérabilité est de type “CWE-502: Deserialization of Untrusted Data”. Elle occur, car un LaissezFaire introduit accepte sans discrimination. On peut forcer toutes sortes d’injections depuis le JSON initial. Pour le rapport Semgrep, encore une fois, l’erreur n’a pas été levée. Cette fois-ci, mon erreur semble être causée par ma compréhension de Semgrep. Mon débogage intensif avec Claude n’a pas été mécaniquement fructueux, mais je semble comprendre que Semgrep analyse seulement de manière statique le code et certains éléments de mon implémentation lui passe par dessus la tête. Voici le lien utilisé pour cette vulnérabilité: https\://semgrep.dev/r?q=go.lang.security.deserialization.unsafe-deserialization-interface.go-unsafe-deserialization-interface.

7. Écriture de fichier arbitraire dans DigestHelper.java

J’ai choisi ce fichier, car il crée un fichier temporaire. Il est commode lors de débogage de créer une copie du fichier temporaire portant un nom similaire. Ce fichier peut être manipulé et inclure de la malice. La vulnérabilité est de type “CWE-22: Improper Limitation of a Pathname to a Restricted Directory ('Path Traversal')”. Cette faille occur puisqu’on ne valide pas resourceName. C’est une donnée qui vient du nom du fichier et peut-être contrôlée. Si le nom brise certaines conventions, le fichier risque d’être supprimé sur le système. Pour le rapport Semgrep, encore une fois, l’erreur m’est apparue trop tard. Il semblerait que cette erreur soit seulement détectable lorsque provenant d’un servlet. Ce concept ne m’est pas familier et m’a totalement pris de recours. Notre code ne lit pas le path depuis une requête HTTP et donc ne lève pas l’erreur. Voici le lien utilisé pour cette vulnérabilité: https\://semgrep.dev/r?q=java.lang.security.httpservlet-path-traversal.httpservlet-path-traversal
