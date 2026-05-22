# Analyse complète du codebase — `llm/`

> Sous-arborescence `llm/` du tree plan9port, dédiée aux assets LLM-facing
> (skills Claude Code, hooks, slash commands, prompts).
> Périmètre : ce répertoire seul, en complément du
> `/usr/local/plan9/codebase_analysis.md` (vue racine) et du
> `/usr/local/plan9/src/cmd/acme/codebase_analysis.md` (vue acme).

## 1. Vue d'ensemble du projet

### Nature

`llm/` n'est **pas** une application au sens classique : c'est un
ensemble cohérent de cinq familles d'artefacts destinés à
**configurer un agent Claude Code** pour qu'il sache piloter `acme(1)`
et écrire correctement du `rc(1)` plan9port :

| Famille | Rôle |
|---|---|
| Skills (`acme/`, `rc/`) | Documentation de référence chargée par Claude Code quand l'agent rencontre un contexte acme ou rc. |
| Hook PostToolUse (`bin/acme-update`) | Synchronise les fenêtres acme ouvertes après chaque `Edit`/`Write`/`MultiEdit`/`NotebookEdit` de Claude. |
| Driver manuel (`bin/acme-sync-recent`) | Resynchronise toutes les fenêtres à partir de `git diff --name-only HEAD`. |
| Slash command (`commands/update-acme.md`) | Expose ce driver comme `/update-acme` dans Claude Code. |
| Templates de prompt (`plan`, `implement`, `verify`, `TODO`) | Consommés à la main (copier-coller dans une session). |

L'installeur (`mkfile` + `install-hook.rc`) déploie les quatre
premières familles dans `~/.claude/`. La cinquième est laissée libre.

### Stack technique

- **Langage principal :** Plan 9 `rc` (shell de Plan 9), pas `sh`/`bash`.
- **Build :** `mk` (équivalent Plan 9 de `make`), `mkfile` autonome
  non balayé par le `./INSTALL` racine.
- **Documentation :** Markdown avec frontmatter YAML pour les skills
  (`name`, `description`).
- **Dépendances runtime :** `jq` (fusion `settings.json`), `9p` (client
  9P de plan9port), `awk`/`sed`/`grep` (flavour Plan 9), `expr`
  (BSD-style ; sans page de man en p9p mais le binaire existe).
- **Interopérabilité :** lit le 9P d'acme à `/mnt/acme`, fusionne du
  JSON dans `~/.claude/settings.json`, est invoqué par Claude Code via
  un hook PostToolUse.

### Architecture

Pattern : **pipeline d'observabilité événementiel sans état**.

- Le hook est invoqué *après* chaque outil de modification de Claude.
- Il lit du JSON sur `stdin`, en extrait `tool_name`, `file_path` et
  `new_string`, interroge le 9P d'acme pour repérer la fenêtre du
  fichier touché, puis :
  - si fenêtre propre → `get` (rechargement disque) + écriture de la
    range dans `<id>/addr` + `dot=addr` + `show` (4 writes 9P) ;
  - si fenêtre sale → message dans `+Errors`, jamais d'écriture ;
  - si pas de fenêtre → en crée une (pour les outils supportés).
- Aucun état persistant entre invocations ; chaque appel est
  idempotent et silencieux en cas d'absence d'acme.

### Langage et versions

- **Plan 9 rc** (rc(1) version plan9port, pas dialecte sh). Tous les
  scripts portent `#!/usr/local/plan9/bin/rc`.
- **Markdown CommonMark** pour SKILL.md et commande slash.
- **JSON** pour la fusion `settings.json` (via `jq`).
- Pas de C, pas de Python, pas de JavaScript.

## 2. Analyse détaillée de la structure de répertoires

```
llm/
|-- README.md                ← table des matières + idiomes rc clés
|-- CLAUDE.md                ← guide d'entretien (quand mettre à jour quoi)
|-- mkfile                   ← installeur (cibles install*/uninstall*)
|-- install-hook.rc          ← fusion jq dans ~/.claude/settings.json
|-- .ai-jail                 ← config sandbox (orthogonal, non installé)
|-- TODO                     ← travail en cours (conversion Emph* → scripts)
|-- plan                     ← prompt template : produit ./PLAN.md depuis ./TODO
|-- implement                ← prompt template : implémente une étape de PLAN.md
|-- verify                   ← prompt template : vérifie la dernière étape faite
|-- acme/
|   `-- SKILL.md             ← 9P d'acme, ctl/nctl, event, dirty rules (658 l.)
|-- rc/
|   `-- SKILL.md             ← langage rc + outils texte plan9port (900 l.)
|-- bin/
|   |-- acme-update          ← hook PostToolUse + helper mono-fichier
|   `-- acme-sync-recent     ← driver manuel (corps de /update-acme)
`-- commands/
    `-- update-acme.md       ← définition de la slash command
```

### `acme/` — skill « driver acme »

- **Un seul fichier** : `SKILL.md`, 658 lignes.
- Frontmatter : `name: acme`, `description:` mentionnant le 9P à
  `/mnt/acme` et les deux règles dures (`dirty`/`clean` interdits,
  pas de mutation d'une fenêtre sale).
- Sections : règles dures, vérification de l'état sale, modèle
  mental, catalogue de fichiers (par fenêtre + globaux), verbes
  `ctl` et `nctl`, idiomes courants, protocole d'événements,
  plumber, écriture de scripts rc pour acme, auto-sync avec Claude
  Code, conventions.
- **Source de vérité** pour tout agent qui touche `/mnt/acme`.
- Liens internes vers : `$PLAN9/scripts/`, `$PLAN9/man/`, le
  CODE_CHANGES.md racine, le codebase_analysis.md d'acme.

### `rc/` — skill « langage rc »

- **Un seul fichier** : `SKILL.md`, 900 lignes — le plus volumineux.
- Frontmatter : `name: rc`, `description:` couvrant syntaxe rc,
  outils p9p, shebang portable, idiomes.
- Sections : règles dures (`$status` est une chaîne, `~` mute
  `$status`, variables = listes, pas de `local`, heredocs cassés
  dans `{}`, `[ ]` pas test, double-quote inerte), cheat-sheet
  bash/POSIX vs rc, langage (variables, carets libres, quoting,
  `~`, contrôle de flux, fonctions, substitution, redirection,
  built-ins, variables notables, globs), outils plan9port
  (`regexp(7)`, `test`, `awk`, `sed`, `grep`, `tr`, `wc`, `cat`,
  `read`, `mc`, `seq`, `expr`, `9p`, `mk`, wrapper `9`), idiomes
  communs, **pitfalls** (longue liste de surprises empiriques),
  plumber/9P, écriture de scripts qui pilotent acme, helpers de
  haut niveau, conventions, pointeurs.
- **Companion** d'`acme/SKILL.md` : si un script touche
  `/mnt/acme`, on lit les deux.

### `bin/` — exécutables installés

| Fichier | Lignes | Rôle |
|---|---|---|
| `acme-update` | 141 | Bi-mode (hook stdin JSON, ou `FILE [TOOL]`). Trois branches : fenêtre propre → reload+dot, fenêtre sale → warning, pas de fenêtre → création. |
| `acme-sync-recent` | 46 | Itère `git diff --name-only HEAD` + untracked, appelle `acme-update` avec `Edit` ou `Write` selon le statut git. |

Les deux sont copiés vers `~/.claude/skills/acme/bin/` (les hooks
résident sous le répertoire du skill `acme`, par convention de
plan9port).

### `commands/` — slash commands Claude

- `update-acme.md` : frontmatter YAML avec `description` et
  `allowed-tools: Bash(*acme-sync-recent*)`. Le corps Markdown
  explique le rôle ; la ligne finale `!~/.claude/skills/acme/bin/acme-sync-recent`
  est la commande exécutée à l'appel de `/update-acme`.

### Fichiers racine

- **`README.md`** : tables des fichiers + section « Implementation
  notes » qui catalogue trois constructions rc cruciales (capture
  stdin sans squash, itération newline-safe, awk standard pour
  fixed-string).
- **`CLAUDE.md`** : guide *opérationnel* du mainteneur. Quatre
  sections « When to update X » : quand rééditer `acme/SKILL.md`,
  `rc/SKILL.md`, le hook. Plus une section « Plan 9 rc quirks
  worth knowing » qui duplique en plus court ce qui est dans
  `rc/SKILL.md` (heredocs cassés, `for` sur ifs par défaut, pas de
  `return` builtin, carets, `~`, ``` `{} ```).
- **`mkfile`** : cibles `install`, `install-skill`,
  `install-acme-skill`, `install-rc-skill`, `install-bin`,
  `install-command`, `register-hook` ; `uninstall*` symétriques.
  Granularité fine. Variable `ACMESKILLDIR=$HOME/.claude/skills/acme`,
  `RCSKILLDIR=$HOME/.claude/skills/rc`, `BINDIR=$ACMESKILLDIR/bin`.
- **`install-hook.rc`** : 80 lignes, fusion jq idempotente avec
  copie `.bak`, contrôle de validité JSON, fallback manuel si `jq`
  manquant. **Exemple de référence** pour le contournement
  heredoc-dans-`{}` (fonction pleine de `echo`).
- **`TODO`** : 17 lignes d'instructions pour la prochaine
  itération (convertir les commandes intégrées `Emph*` d'acme en
  scripts rc autonomes dans `$PLAN9/scripts/`).
- **`plan`, `implement`, `verify`** : trois prompt templates très
  courts (5-10 lignes chacun) consommés en mode copier-coller
  dans des sessions Claude Code, qui forment un mini-workflow
  *plan → implement → verify* basé sur les fichiers
  `./PLAN.md`/`./DONE` à la racine du tree de travail.
- **`.ai-jail`** : configuration orthogonale du sandbox
  https://github.com/akitaonrails/ai-jail. Pointe vers Gemini, pas
  vers Claude — résidu d'un autre workflow ; non installé par le
  `mkfile`.

## 3. Décomposition fichier par fichier

### 3.1 Fichiers cœur (application logic)

#### `bin/acme-update` — le hook lui-même

Structure (de haut en bas) :

1. **Garde-fous silencieux** (lignes 26-29) : retourne 0 si `jq`
   ou `9p` indisponibles. Un hook qui sort non-zéro remonte une
   erreur à Claude — proscrit ici.
2. **Initialisation** : `tool`, `file`, `needle`, `nl` (séparateur
   newline littéral).
3. **Branchement mode** :
   - **Hook** (`$#* == 0`) : capture `stdin` avec
     `payload=`''{cat}` (séparateur vide pour préserver newlines
     et espaces du JSON), extrait via `jq -r` :
     `tool_name`, `file_path`/`notebook_path`, et `new_string` selon
     l'outil (cas particulier `MultiEdit` → `edits | last.new_string`,
     `NotebookEdit` → `new_source // new_string`).
   - **Manuel** (`acme-update FILE [TOOL]`) : assigne `$file=$1`,
     `$tool=$2`, défaut `Edit`.
4. **Extraction `first`/`last`** (lignes 70-73) : premières et
   dernières lignes non-blanches de `$needle` via `sed`/`awk`,
   pour cibler la sélection. Coercition explicite vers `''` si
   capture vide (sinon `set_dot` reçoit moins d'arguments —
   bitten in the wild).
5. **`fn find_window`** (lignes 79-81) : un seul `awk` qui scanne
   `acme/index` et imprime `$1 $5` du premier record dont `$6`
   (chemin de fichier dans le tag) matche le path. Le commentaire
   note explicitement le choix d'un seul awk plutôt qu'une boucle
   rc — car *pas de `return`* en rc plan9port.
6. **`fn set_dot`** (lignes 90-109) : best-effort. Cherche la
   première ligne contenant `$sd_first` ; si `$sd_last` diffère,
   cherche en avant à partir de cette ligne. Écrit
   `start,end` (ou `start` seul, ou `1` à défaut) dans
   `acme/$wid/addr`, puis `dot=addr` et `show` dans `ctl`.
   **Commentaire critique** : tous les noms de variables sont
   préfixés `sd_` car `path` global est synchronisé avec `$PATH`
   et tout assignement `path=$3` casserait `exec` en aval
   silencieusement.
7. **Branchement principal** (lignes 111-141) :
   - Fenêtre absente + outil supporté + fichier sur disque →
     `9p read acme/new/ctl` → rename → `get` → `set_dot`.
   - Fenêtre sale → message dans `acme/cons`, exit.
   - Fenêtre propre → `get` + `set_dot`.

**Forces** : robustesse aux modes dégradés (acme absent, jq
absent, fenêtre supprimée entre deux appels), respect strict
des règles dures, code court (141 l. très commentées), zéro
dépendance non-standard.

**Subtilités à retenir** :
- Choix `$nl=` (newline littéral en quoting fort) comme
  séparateur de `` `{...} `` pour préserver les espaces des
  chemins.
- `awk -v s=$first 'index($0, s) { print NR; exit }'` —
  fixed-string et early-exit, car plan9port `grep` n'a ni `-F`
  ni `-m`.
- Le `set_dot` ne lit jamais `acme/<id>/addr` pour vérifier :
  ouvrir ce fichier reset l'adresse à `0,0` (cf. `xfid.c:110`),
  c'est documenté dans `acme/SKILL.md`.

#### `bin/acme-sync-recent` — driver manuel

Plus simple (46 l.) :

1. Vérifie que `acme-update` est installé et exécutable.
2. `git rev-parse --show-toplevel` ; sort si pas dans un repo.
3. `git diff --name-only HEAD` + `git ls-files --others --exclude-standard` → liste fusionnée.
4. Pour chaque fichier : `git ls-files --error-unmatch` détecte
   les nouveaux fichiers (untracked) → passe `Write` pour
   provoquer l'ouverture d'une fenêtre ; sinon `Edit`.
5. Compteur final `synced N file(s)`.

#### `install-hook.rc` — fusion idempotente settings.json

Pattern remarquable :

- `print_snippet` est une **fonction pleine de `echo`** au lieu
  d'un heredoc — workaround documenté pour la limitation rc
  « heredocs cassés dans `{}` et `fn` ».
- Si `jq` absent : imprime le snippet à fusionner à la main et
  sort avec `exit nojq` (label informatif, pas `exit 1`).
- Si fichier `settings.json` absent : `echo '{}' > $settings`.
- Copie `.bak` *avant* toute modification.
- `add` : fusion jq qui **supprime** d'abord toute entrée
  référençant le même `command`, puis re-ajoute — d'où
  l'idempotence stricte.
- `remove` : retire le command du tableau `hooks`, puis filtre
  les blocs vides.
- Validation finale : `jq . settings.new` ; si invalide → garde
  le `.bak`, supprime le `.new`, sort avec `exit invalid`.

### 3.2 Fichiers de configuration

| Fichier | Format | Rôle |
|---|---|---|
| `mkfile` | mkfile Plan 9 | Cibles d'installation granulaires. |
| `commands/update-acme.md` | Markdown + frontmatter YAML | Définition de la slash command (`description`, `allowed-tools`). |
| `.ai-jail` | TOML | Sandbox pour Gemini — non lié à Claude. |

### 3.3 Data layer

**Aucune** : pas de DB, pas de modèles, pas de migrations. Toute
l'état applicatif vit dans le 9P d'acme (lu/écrit à chaud) ou dans
`~/.claude/settings.json` (modifié par l'installeur).

### 3.4 Frontend / UI

**Aucune** au sens classique. L'« UI » est :
- les fenêtres acme elles-mêmes (que le hook synchronise) ;
- le chat Claude Code (où apparaissent les slash commands et les
  descriptions de skills).

### 3.5 Tests

**Aucun test automatisé.** Cohérent avec la convention plan9port
racine (`CLAUDE.md` du tree : « There is no global automated test
suite. Verification is mostly by running the program »).
Vérification = installation manuelle + utilisation en session.

### 3.6 Documentation

Trois niveaux :

1. **README.md** : front-door (qui lit quoi, comment installer).
2. **CLAUDE.md** : guide d'entretien (quand modifier quoi, quels
   patterns rc surveiller).
3. **SKILL.md × 2** : source de vérité technique, chargée *dans
   le contexte de Claude* à la demande.

Et trois prompts (`plan`, `implement`, `verify`) qui forment un
mini-workflow pour itérer sur des tâches plus larges.

### 3.7 DevOps / CI

- **Pas de CI** (pas de `.github/`, `.gitlab-ci.yml`, etc.).
- **Pas de Docker.**
- Déploiement = `cd llm && mk install` sur la machine cible.
- Versionning = celui du repo parent plan9port (tag local
  `gitbutler/workspace`, branche principale `master`).

## 4. Analyse des endpoints API

**Sans objet** : ce module n'expose ni HTTP, ni gRPC, ni MQ.

Il **consomme** trois surfaces :

| Surface | Protocole | Usage |
|---|---|---|
| `/mnt/acme/*` | 9P (via `9p read`/`write`) | **Lectures :** `acme/index` (id, dirty, tag), `acme/new/ctl` (ouverture = création de fenêtre). **Écritures :** `acme/<id>/ctl` (verbes `name`, `get`, `dot=addr`, `show`), `acme/<id>/addr` (range sam-style sans `\n` final), `acme/cons` (message warning pour fenêtre sale). **Jamais touchés par le hook :** `<id>/body`, `<id>/data`, `<id>/xdata`, `<id>/tag`, `<id>/event`. |
| `~/.claude/settings.json` | JSON sur disque | Fusion jq pour ajouter/retirer le hook. |
| Claude Code PostToolUse | JSON sur stdin | Champ `tool_name`, `tool_input.file_path`, `tool_input.new_string` (etc.). |

## 5. Architecture deep dive

### 5.1 Architecture globale

```
+--------------------+        +------------------+
|   Utilisateur      |        |   Claude Code    |
|   (acme ouvert)    |        |   (CLI ou IDE)   |
+----------+---------+        +---------+--------+
           |                            |
           | actions clavier/souris     | Edit/Write/MultiEdit
           v                            v
+--------------------+        +------------------+
|     acme(1)        |<-------|  Hook PostToolUse |
|  /mnt/acme (9P)    |        |   acme-update     |
+--------------------+        +---------+--------+
           ^                            |
           |                            | JSON sur stdin
           | 9p read/write              | jq parse
           |                            |
           +----------------------------+
                        |
                +-------v--------+
                |  Disque        |
                |  (fichier      |
                |   modifié)     |
                +----------------+
```

### 5.2 Cycle de vie d'une modification

1. **Claude appelle `Edit`** sur un fichier `/foo/bar.c`.
2. Le harness Claude Code (extérieur à ce repo) **émet un
   événement PostToolUse** une fois l'Edit terminée, avec un JSON
   contenant `tool_name`, `tool_input.file_path`,
   `tool_input.new_string`, etc.
3. **Le matcher** `Edit|Write|MultiEdit|NotebookEdit` (configuré
   par `install-hook.rc`) **déclenche `acme-update`**.
4. `acme-update` :
   - capture stdin en `payload=`''{cat}`,
   - extrait via `jq` : `tool=Edit`, `file=/foo/bar.c`, `needle=<contenu inséré>`,
   - extrait `first`/`last` lignes non-blanches de `needle`,
   - `find_window /foo/bar.c` → renvoie `42 0` (id=42, clean),
   - écrit `get` dans `acme/42/ctl` → acme recharge depuis disque,
   - `set_dot 42 first last /foo/bar.c` → écrit `start,end` dans
     `acme/42/addr`, `dot=addr` puis `show` dans `acme/42/ctl`,
   - sort 0.
5. **L'utilisateur voit** la fenêtre rechargée avec la sélection
   sur le bloc modifié, prête à être inspectée ou éditée.

### 5.3 Cas dégradés (tous gérés silencieusement)

| Cas | Comportement |
|---|---|
| Acme pas lancé | `9p read acme/index` échoue → exit 0 silent. |
| jq absent | exit 0 silent (le hook n'aurait rien à fusionner). |
| Fichier inconnu d'acme + outil supporté + sur disque | Ouvre une nouvelle fenêtre via `acme/new/ctl`. |
| Fichier inconnu d'acme + pas sur disque | exit 0 silent. |
| Fenêtre sale | Message dans `+Errors` via `acme/cons`. Pas de mutation. |
| `new_string` vide (cas `Write`) | `first`/`last` vides → `set_dot` retombe sur ligne 1. |
| Fenêtre fermée entre deux étapes | `>[2]/dev/null` sur les writes best-effort → erreur ignorée. |

### 5.4 Patterns de design

- **Defensive scripting** : tout échec récupérable → `exit 0`.
  Aucune exception, aucune trace bruyante.
- **Single-pass parsing** : `find_window` fait *tout* en un seul
  `awk` (économie de forks + travail autour de l'absence de
  `return` en rc).
- **Préfixage de variables locales** : `sd_*` dans `set_dot` pour
  ne pas écraser `$path`/`$home`/etc. (cf. footgun documenté).
- **Compositions courtes** : chaque fonction tient sur un écran ;
  composition avec awk/sed/grep/expr.
- **Idempotence** : `install-hook.rc add` retire toute entrée
  préexistante avant de réinsérer.
- **Backup-before-mutate** : `.bak` du `settings.json` avant
  chaque fusion, vérification de validité JSON avant
  remplacement.

### 5.5 Dépendances entre modules

```
mkfile  (orchestrateur)
   |
   |---> install-hook.rc        (fusion JSON via jq, avec .bak + validation)
   |        |
   |        `--jq->  ~/.claude/settings.json
   |
   |---> acme/SKILL.md  ------> ~/.claude/skills/acme/SKILL.md
   |---> rc/SKILL.md    ------> ~/.claude/skills/rc/SKILL.md
   |---> bin/acme-update ----> ~/.claude/skills/acme/bin/
   |---> bin/acme-sync-recent ↑
   `---> commands/update-acme.md → ~/.claude/commands/

À l'exécution :
   PostToolUse  ----JSON stdin----> bin/acme-update --9p--> /mnt/acme
                                              |
                                              `---disque----> fichier édité

   /update-acme ---bash---> bin/acme-sync-recent ---fork---> bin/acme-update
                                              |
                                              `---git---> diff HEAD + untracked
```

## 6. Environnement et installation

### Pré-requis

| Outil | Pour quoi | Si absent |
|---|---|---|
| `jq` | Fusion `settings.json` | Installeur : imprime snippet à fusionner manuellement. Hook : exit 0 silent. |
| `9p` (plan9port) | Tout dialogue avec acme | Hook : exit 0 silent. |
| `awk`, `sed`, `grep`, `expr` (Plan 9) | Parsing | Fournis par plan9port. |
| `rc` (Plan 9) | Exécution scripts | Idem. |
| `acme` running | Cible des updates | Hook : exit 0 silent. |
| `git` | `acme-sync-recent` | Exit `notgit`. |

### Variables d'environnement

Aucune variable d'application propre. Le hook hérite de
l'environnement Claude Code (variables propagées par le harness)
et accède à `$HOME` pour résoudre les paths. Le script
`acme-update` n'utilise *ni* `$winid` *ni* `$samfile` (variables
acme-spawned) : il opère depuis l'extérieur d'acme.

### Workflow de développement

1. **Modifier** un fichier sous `llm/`.
2. **`cd llm && mk install`** pour redéployer (idempotent).
3. **Tester** dans une session Claude Code (Edit un fichier
   suivi par acme, observer la fenêtre).
4. **Mettre à jour les SKILL/CLAUDE** appropriés si l'on a
   découvert un footgun ou un nouveau verbe.

### Workflow de release

Trivial : ce module n'est pas releasé indépendamment. Il vit
dans le repo plan9port et bénéficie de son git. Le `CLAUDE.md`
mentionne explicitement que `acme/SKILL.md` doit rester
project-relative pour permettre une future extraction en repo
séparé.

### Désinstallation

```sh
cd llm && mk uninstall
```

Retire les fichiers déployés, dé-registre le hook (avec `.bak`).

## 7. Stack technologique détaillée

### Runtime

- **`rc(1)` plan9port** — interpréteur principal. Différences vs
  bash documentées exhaustivement dans `rc/SKILL.md`.
- **Brian Kernighan's `awk`** — pas gawk, donc pas de `gensub`.
- **Plan 9 `sed`/`grep`/`tr`** — pas de GNU extensions, regex
  flavour `regexp(7)` (pas POSIX ERE/BRE).

### Outils externes

- **`jq`** (1.x) — fusion JSON dans `install-hook.rc`.
- **`git`** (≥ 2.0) — `rev-parse`, `diff --name-only`, `ls-files`
  dans `acme-sync-recent`.
- **`9p`** (plan9port) — client 9P, le seul protocole de
  dialogue avec acme.

### Framework cible

- **Claude Code** : harness conteneur. Le module produit un
  hook *consommé* par ce harness via la convention
  `~/.claude/settings.json::hooks.PostToolUse`.

### Build tools

- **`mk(1)` plan9port** — pas `make`. Cibles `:V:` (virtuelles),
  `:V:` chains pour orchestration.

### Pas de :

- Bundler, transpiler, linter, formatter automatisé.
- Test runner.
- Container runtime.
- Reverse proxy, queue, cache.
- ORM, schema migration.

Cette absence est *delibérée* et cohérente avec l'esprit
plan9port : composer des outils Unix existants plutôt que
embarquer du nouveau.

## 8. Diagramme d'architecture visuel

### 8.1 Hiérarchie d'installation

```
SOURCE (llm/)                     DÉPLOYÉ (~/.claude/)
=============                     =====================

llm/
|-- acme/                         skills/acme/
|   `-- SKILL.md         --->     |-- SKILL.md
|                                 `-- bin/
|-- bin/                              |-- acme-update         (chmod +x)
|   |-- acme-update      --->         `-- acme-sync-recent    (chmod +x)
|   `-- acme-sync-recent --->
|
|-- rc/                          skills/rc/
|   `-- SKILL.md         --->     `-- SKILL.md
|
|-- commands/                    commands/
|   `-- update-acme.md   --->     `-- update-acme.md
|
|-- install-hook.rc      --->    settings.json (fusion jq + .bak)
|-- mkfile                       (lui-même non installé)
|-- README.md                    (idem)
|-- CLAUDE.md                    (idem)
|-- plan, implement, verify      (templates, jamais installés)
|-- TODO                         (état de travail, non installé)
`-- .ai-jail                     (config externe, non installé)
```

### 8.2 Flux d'exécution du hook

```
   ____________
  / Claude     \    Edit /foo/bar.c
  \   Code     /---------------+
   ------------                |
                               v
                    +----------+-----------+
                    | PostToolUse matcher  |
                    | "Edit|Write|         |
                    |  MultiEdit|          |
                    |  NotebookEdit"       |
                    +----------+-----------+
                               | JSON stdin
                               v
                    +----------+-----------+
                    |   acme-update        |
                    |                      |
                    | 1. jq absent? exit 0 |
                    | 2. acme absent?      |
                    |    exit 0            |
                    | 3. parse JSON ------>+--+ jq -r .tool_name
                    | 4. extract needle    |  | jq -r .tool_input.file_path
                    | 5. sed first line    |  | jq -r .tool_input.new_string
                    | 6. awk last line     |  |
                    | 7. find_window ----->+--+ 9p read acme/index | awk
                    |                      |
                    +----------+-----------+
                               |
                +--------------+--------------+
                v              v              v
       +--------+--+   +-------+----+  +------+-------+
       | No window |   | Dirty win  |  | Clean win    |
       |           |   |            |  |              |
       | new/ctl   |   | acme/cons  |  | get          |
       | name PATH |   | warning    |  | addr=<range> |
       | get       |   | (no mut.)  |  | dot=addr     |
       | set_dot   |   |            |  | show         |
       +-----------+   +------------+  +--------------+
                               |
                               v
                    +----------+-----------+
                    |   exit 0             |
                    +----------------------+
```

### 8.3 Flux du driver manuel

```
   /update-acme
        |
        v
   bash command (frontmatter allowed-tools)
        |
        v
   acme-sync-recent
        |
        +--> git rev-parse --show-toplevel
        |        |
        |        v
        +--> git diff --name-only HEAD
        |        |
        +--> git ls-files --others --exclude-standard
        |        |
        |        v
        |    files = (diff ∪ untracked)
        |
        +--> pour chaque f:
                 git ls-files --error-unmatch -- $f
                       |
                +------+-------+
                |              |
            untracked         tracked
                |              |
                v              v
            acme-update    acme-update
              $f Write       $f Edit
                |              |
                +------+-------+
                       |
                       v
            (cf. flux hook ci-dessus)
                       |
                       v
                  n = n + 1
                       |
                       v
              echo synced $n file(s)
```

### 8.4 Surfaces 9P consommées

```
/mnt/acme/                          opération
-----                               ---------
index                               9p read (find_window : awk '$6==t {print $1,$5; exit}')
                                              -- $1=id, $5=dirty, $6=premier mot du tag
new/ctl                             9p read (ouverture = création; awk '{print $1}' du
                                              ctl renvoyé donne le nouvel id)
<id>/ctl                            9p write des verbes :
                                              "name PATH"  (rename, branche new-window)
                                              "get"        (recharger depuis disque)
                                              "dot=addr"   (poser la sélection sur l'addr
                                                            qu'on vient d'écrire)
                                              "show"       (scroller pour rendre dot visible)
                                            -- jamais lu par le hook
                                            -- jamais "addr=dot" (qui ferait l'inverse :
                                                                   copier dot dans addr)
                                            -- jamais "clean" ni "dirty" (règle dure)
<id>/addr                           9p write "start,end" (range sam-style,
                                              SANS newline final ! cf. parser isaddrc)
cons                                9p write (message d'avertissement vers +Errors
                                              lorsqu'une fenêtre sale aurait dû être
                                              rechargée)

Jamais touchés par le hook :
<id>/body  <id>/data  <id>/xdata  <id>/tag  <id>/event  <id>/errors  <id>/editout
acme/ctl   acme/log   acme/cons (en lecture)
```

## 9. Insights et recommandations

### 9.1 Qualité du code

**Points forts :**

- **Documentation exhaustive** : 1500+ lignes de SKILL.md
  croisées avec un guide d'entretien (`CLAUDE.md`) qui dit
  *quand* mettre à jour quoi. Rare et précieux.
- **Tolérance aux pannes** : `exit 0 silent` partout, pas
  d'exception bruyante, hook qui ne casse jamais Claude.
- **Idempotence** : installeur testable par re-exécution sans
  effet secondaire.
- **Backup-before-mutate** : `.bak` systématique avant fusion
  JSON, validation post-fusion.
- **Commentaires denses *là où il faut*** : `set_dot` explique
  *pourquoi* tous les noms sont préfixés `sd_`, `find_window`
  explique *pourquoi* un seul `awk` plutôt qu'une boucle rc. Ces
  commentaires sont *non-évidents depuis le code* — exactement ce
  que la convention demande.
- **Tests indirects par documentation** : chaque footgun rc
  rencontré a été immortalisé dans `rc/SKILL.md` (« bitten in the
  wild by acme-update's set_dot »).

**Points à surveiller :**

- **Pas de tests automatisés** — cohérent avec plan9port mais
  signifie qu'une régression du hook ne sera détectée qu'à
  l'usage. Mitigation : la documentation des cas est si riche
  qu'un test unitaire serait redondant pour les patterns simples.
- **Duplication partielle** entre `CLAUDE.md::Plan 9 rc quirks` et
  `rc/SKILL.md::Pitfalls`. Le premier est *opérateur*, le second
  *référence* — la duplication semble volontaire mais mérite
  d'être maintenue en synchronisation. Risque de divergence si
  l'un est mis à jour sans l'autre.
- **`acme/SKILL.md` mélange driver-d'acme et écriture-de-scripts-acme** —
  les deux sont liés mais distincts. Pour un futur split, la
  section « Writing rc scripts for acme » pourrait migrer vers
  `rc/SKILL.md` ou un troisième skill.

### 9.2 Améliorations potentielles

| Priorité | Suggestion |
|---|---|
| Haute | Ajouter un `mk test` qui exerce `acme-update` en mode `FILE [TOOL]` avec une acme jetable lancée dans un namespace dédié (`NAMESPACE=/tmp/test.ns.$pid 9 acme &` puis `NAMESPACE=/tmp/test.ns.$pid 9p ls acme` ; cf. `9 man 4 namespace` et `bin/namespace`). |
| Moyenne | Extraire une fonction `read_json_field` paramétrée pour DRY-ifier les trois `jq -r '...'` consécutifs au début de `acme-update`. |
| Moyenne | Logger les sauts dans `+Errors` pour les modes « pas de fenêtre + fichier absent du disque » — actuellement silent, ce qui peut surprendre l'utilisateur (le hook a-t-il tourné ?). |
| Basse | Compléter `acme-sync-recent` avec un filtre `-- $PATHSPEC` optionnel (déjà commenté dans `commands/update-acme.md` ?). |
| Basse | Supprimer ou clarifier le rôle de `.ai-jail` (configuration Gemini orthogonale à ce sous-module). |
| Basse | Ajouter dans `README.md` un schéma d'installation visuel similaire à 8.1 ci-dessus. |

### 9.3 Considérations de sécurité

**Surface d'attaque limitée et bien circonscrite :**

- Le hook lit du JSON sur `stdin` provenant de Claude Code (le
  harness), pas d'une source non fiable. `jq -r` extrait des
  champs typés (`tool_name`, `file_path`, `new_string`) sans
  exécution de code.
- Le hook ne réalise pas `eval` ni d'interpolation directe :
  toutes les valeurs extraites sont passées via `awk -v` ou
  écrites comme données dans `acme/<id>/addr`/`data`/`ctl`.
- Les écritures `9p` ne peuvent affecter que les fenêtres acme
  de l'utilisateur courant (`/mnt/acme` est privé au namespace).
- Pas de réseau, pas de DB, pas d'auth, pas de PII.

**Risques résiduels :**

- Un `file_path` malicieusement crafté avec des espaces et des
  metacaractères rc pourrait théoriquement perturber le
  parsing — mais (1) `file_path` vient de Claude Code lui-même,
  (2) `file` est capturé via ``file=`$nl{...}`` (séparateur =
  newline littéral), donc `rc` ne le splitte pas sur les espaces
  internes et `awk -v t=$file` reçoit un seul argument même si le
  chemin contient des blancs, (3) `awk -v` interprète la valeur
  comme une chaîne littérale (pas comme un programme awk), et
  (4) en cas d'absurdité, `awk` ne matche aucune ligne et le hook
  bascule sur la branche « no window ».
- Le `cp $settings $settings.bak` dans `install-hook.rc` pourrait
  écraser un `.bak` précédent volontairement conservé.
  Mitigation actuelle : il n'y a qu'un seul slot de backup ; la
  validation post-fusion permet le rollback manuel
  (`mv ~/.claude/settings.json.bak ~/.claude/settings.json`).

### 9.4 Performance

**Coût par invocation du hook : O(1) modeste.**

- 1 × `jq --version` (test).
- 1 × `9p read acme/index` (test + parsing 1 seul `awk`).
- 1 × `cat` pour capturer `stdin`.
- 3 × `jq -r '...'` pour extraction.
- 2 × `sed`/`awk` pour `first`/`last`.
- 1 × `find_window` (1 `9p read` + 1 `awk`).
- Branche heureuse : 3 × `9p write` (get, dot=addr, show) + 1 ×
  `9p write addr`.

Pas de boucle sur les fenêtres autres que celle ciblée. Un seul
fork par `awk`/`sed`/`jq` invoqué — le commentaire dans
`find_window` indique explicitement le choix « un seul awk
plutôt qu'une boucle » pour économiser les forks.

**Latence end-to-end estimée :** ~50-150 ms sur du matériel
moderne, dominée par les forks shell et les writes 9P sur
socket Unix. Imperceptible pour l'utilisateur.

### 9.5 Maintenabilité

**Excellente** pour la taille du projet :

- Chaque fichier exécutable < 150 lignes.
- Chaque concept a son `SKILL.md` (acme vs rc) ou sa section.
- Le `CLAUDE.md` explicite les conditions de mise à jour de
  chaque artefact — un mainteneur nouveau sait *où* regarder
  quand un verbe change.
- Conventions de tree (français pour l'utilisateur, anglais
  dans le code, pas d'emoji, fonctions courtes) appliquées
  systématiquement.

**Risques de drift :**

- Les exemples dans `acme/SKILL.md` référencent des scripts de
  `$PLAN9/scripts/` (`A`, `F`, `Bold`, `Color`, `EmphAs`). Si un
  script est renommé sans mise à jour du SKILL, l'agent
  Claude reçoit une référence morte.
- Le `TODO` mentionne la conversion en cours de `Emph*` en
  scripts rc. À la fin de ce travail, la section « Built-in
  commands worth remembering » d'`acme/SKILL.md` devra être
  réécrite (les `Emph*` ne seront plus des commandes intégrées).
- Les chemins absolus `/usr/local/plan9` apparaissent dans
  `bin/acme-update` (shebang) et dans `install-hook.rc`. Un
  port vers un préfixe différent demanderait un sed sur les
  scripts livrés ou un substituer au moment de l'installation.

## 10. Conclusion

`llm/` est un module **petit, dense et soigné** qui transforme
Claude Code en un compagnon de travail conscient d'acme :

- **3 scripts rc exécutables** (~270 lignes au total :
  `bin/acme-update` 141 l., `install-hook.rc` 80 l.,
  `bin/acme-sync-recent` 46 l.) qui font tout le travail visible.
- **2 skills SKILL.md** (~1550 lignes Markdown) qui forment la
  référence technique chargée *dans* Claude.
- **Un installeur** qui sait fusionner du JSON sans casser
  l'existant et qui se désinstalle proprement.
- **Une documentation d'entretien** qui dit *quand* mettre à
  jour chaque artefact — la partie la plus rare dans la plupart
  des projets et la plus précieuse pour la longévité.

Le module reflète l'esprit plan9port : peu de fichiers, composer
des outils existants, documenter les surprises *là où l'agent
suivant les cherchera*. Aucune dépendance lourde, aucune
infrastructure, juste du shell qui parle 9P à un éditeur lancé
par l'utilisateur — et qui sait disparaître silencieusement si
quelque chose manque.
