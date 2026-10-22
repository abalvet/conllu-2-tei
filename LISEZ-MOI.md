# conllu2tei v3: CoNLL-U vers XML TEI (GT XMLisation, Corli)

Une seule applet (`index.html`), deux modes de déploiement.

## Architecture

- **`index.html`** : applet autonome, sans dépendance externe. Contient tout le convertisseur
  CoNLL-U → TEI (lecture, validation, construction de l'arbre XML, indentation).
- **`server/app.py`** (facultatif) : mini-serveur FastAPI qui ne fait que de l'analyse
  (texte → CoNLL-U) avec Stanza, UDPipe local, ou en relayant LINDAT et l'API Corli.
  La conversion TEI n'est jamais faite côté serveur : une seule implémentation, deux contextes.

Au chargement, la page cherche `api/capabilities` :

1. `?api=https://…` dans l'URL, sinon `<meta name="conllu2tei-api" content="…">` dans `index.html` ;
2. sinon, la même origine (sauf sur `*.github.io`) ;
3. si un serveur répond, **mode serveur** : les moteurs annoncés par le serveur sont proposés ;
4. sinon, **mode autonome** : conversion CoNLL-U → TEI, plus UDPipe LINDAT en appel direct si le
   navigateur y est autorisé (CORS). S'il ne l'est pas, l'onglet d'analyse explique la marche à suivre.

## Déploiements

| Cas | Fichiers | Résultat |
|---|---|---|
| Site Corli / GitHub Pages | `index.html` seul | mode autonome |
| Serveur (Huma-Num…) | tout le dossier | mode serveur, applet servie sur `/` |
| Page statique + serveur distinct | `index.html` ici, serveur là | `?api=https://serveur` ou balise `<meta>`, et `ALLOW_ORIGINS=https://site` côté serveur |

## Lancer le serveur

```bash
cd server
pip install -r requirements.txt
# moteurs locaux, selon les besoins
pip install stanza && python -c "import stanza; stanza.download('fr', package='gsd')"
pip install ufal.udpipe          # + modèles *.udpipe dans UDPIPE_MODELS_DIR
uvicorn app:app --host 0.0.0.0 --port 8000
```

Variables d'environnement : voir l'en-tête de `server/app.py`. Un `Dockerfile` est fourni
(construction depuis la racine : `docker build -f server/Dockerfile -t conllu2tei .`).
Derrière un proxy avec préfixe d'URL, l'applet utilise des URL relatives (`api/parse`) :
l'adresse de la page doit se terminer par `/`.

## Profils de sortie

- **TEI P5** (par défaut) : `<w>` / `<pc>`, `@lemma`, `@pos` (UPOS), `@msd` (`UPosTag=…|FEATS`),
  `@join="right"` pour `SpaceAfter=No`, dépendances dans un `<linkGrp type="UD-SYN">` avec des
  `<link ana="ud-syn:nsubj" target="#tête #dépendant"/>` (conventions proches de ParlaMint).
  Mots multi-lexicaux : `<seg type="mwt" n="du">` enveloppant les mots syntaxiques.
  Dépendances enrichies (colonne DEPS) : second `<linkGrp type="UD-ENHANCED">`, en option.
- **TXM / XML-w** : attributs à plat sur `<w>` (`lemma`, `upos`, `xpos`, `feats` ou `feat_*`,
  `deprel`, `head` = `xml:id` de la tête, MISC séparé ou non).

## Enrichir un XML existant

Dans l'onglet « Analyser », un XML/TEI est lu, le contenu des éléments choisis (`p,u,ab` par
défaut) est analysé puis remplacé par des `<s>`/`<w>` ; l'en-tête est complété
(`appInfo`, `listPrefixDef`, `change`). Limite : le balisage interne de ces éléments
(`<hi>`, `<anchor>`, etc.) est aplati, car l'analyse porte sur le texte seul.

## Points à vérifier avant diffusion

- Le TEI produit n'a pas été validé contre `tei_all` ni contre le schéma ParlaMint (option de
  déclaration `xml-model` fournie pour le faire dans oXygen).
- Le relais Corli, Stanza local et UDPipe local n'ont pas été exécutés avec de vrais modèles.
- L'accès direct à LINDAT depuis le navigateur dépend de la politique CORS du service.
