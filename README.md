# conllu2tei v3: CoNLL-U to TEI XML (XMLisation working group, Corli)

A single applet (`index.html`), two deployment modes.

## Architecture

- **`index.html`**: standalone applet with no external dependencies. It contains the whole CoNLL-U → TEI converter (parsing, validation, XML tree construction, indentation).
- **`server/app.py`** (optional): a small FastAPI server that only does linguistic analysis (text → CoNLL-U), using Stanza, a local UDPipe, or by relaying to LINDAT and the Corli API. TEI conversion is never done server-side: one implementation, two contexts.

On load, the page looks for `api/capabilities`:

1. `?api=https://…` in the URL, otherwise `<meta name="conllu2tei-api" content="…">` in `index.html`;
2. otherwise, the same origin (except on `*.github.io`);
3. if a server responds, **server mode**: the engines advertised by the server are offered;
4. otherwise, **standalone mode**: CoNLL-U → TEI conversion, plus direct calls to UDPipe at LINDAT if the browser allows it (CORS). If it does not, the analysis tab explains what to do instead.

## Deployments

| Case | Files | Result |
|---|---|---|
| Corli site / GitHub Pages | `index.html` only | standalone mode |
| Server (Huma-Num…) | the whole folder | server mode, applet served at `/` |
| Static page + separate server | `index.html` on one side, server on the other | `?api=https://server` or the `<meta>` tag, and `ALLOW_ORIGINS=https://site` on the server |

## Running the server

```bash
cd server
pip install -r requirements.txt
# local engines, as needed
pip install stanza && python -c "import stanza; stanza.download('fr', package='gsd')"
pip install ufal.udpipe          # + *.udpipe models in UDPIPE_MODELS_DIR
uvicorn app:app --host 0.0.0.0 --port 8000
```

Environment variables: see the header of `server/app.py`. A `Dockerfile` is provided (build from the project root: `docker build -f server/Dockerfile -t conllu2tei .`). Behind a proxy with a URL prefix, the applet uses relative URLs (`api/parse`): the page address must end with `/`.

## Output profiles

- **TEI P5** (default): `<w>` / `<pc>`, `@lemma`, `@pos` (UPOS), `@msd` (`UPosTag=…|FEATS`), `@join="right"` for `SpaceAfter=No`, and dependencies in a `<linkGrp type="UD-SYN">` made of `<link ana="ud-syn:nsubj" target="#head #dependent"/>` (conventions close to ParlaMint). Multiword tokens: `<seg type="mwt" n="du">` wrapping the syntactic words. Enhanced dependencies (DEPS column): an optional second `<linkGrp type="UD-ENHANCED">`.
- **TXM / XML-w**: flat attributes on `<w>` (`lemma`, `upos`, `xpos`, `feats` or `feat_*`, `deprel`, `head` = the head's `xml:id`, MISC split or not).

## Enriching an existing XML file

In the "Analyse" tab, an XML/TEI file is read, the content of the chosen elements (`p,u,ab` by default) is analysed and replaced by `<s>`/`<w>`; the header is completed (`appInfo`, `listPrefixDef`, `change`). Limitation: the internal markup of these elements (`<hi>`, `<anchor>`, etc.) is flattened, since the analysis operates on plain text only.

## Points to check before release

- The TEI output has not been validated against `tei_all` or the ParlaMint schema (an `xml-model` declaration option is provided so you can do so in oXygen).
- The Corli relay, local Stanza and local UDPipe have not been run with real models.
- Direct access to LINDAT from the browser depends on the service's CORS policy.
