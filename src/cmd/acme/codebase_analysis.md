# Analyse du code source — `acme` (plan9port)

> Périmètre : `/usr/local/plan9/src/cmd/acme/` uniquement. Le sous-projet
> `mail/` (mkfile séparé) est exclu. La couche `libframe`
> (`src/libframe/`) est hors périmètre mais référencée : la fonctionnalité
> d'emphase s'appuie sur une extension de cette bibliothèque. Les en-têtes
> plan9port consommés au build (`u.h`, `libc.h`, `frame.h`, `draw.h`,
> `thread.h`, `9p.h`, …) viennent de `$PLAN9/include/` et ne sont pas
> copiés ici.

## 1. Vue d'ensemble du projet

- **Type** : éditeur de texte graphique multi-fenêtres, à la souris, issu de Plan 9 et porté sur Unix par plan9port.
- **Langage** : C ANSI (style Plan 9, en-têtes `u.h`/`libc.h`/`draw.h`/`thread.h` fournis par plan9port).
- **Modèle de concurrence** : multi-thread coopératif via `libthread` (channels CSP, `threadmain`).
- **Interface système** : sert un système de fichiers 9P via `9pserve` (`fsys.c`). Les fenêtres et leurs commandes sont contrôlables par lecture/écriture de fichiers virtuels (`ctl`, `body`, `tag`, `addr`, `data`, `event`, etc.).
- **Architecture** : boucle d'événements + serveur 9P. Chaque sous-système (clavier, souris, plumb, commandes, xfid) tourne dans son propre thread et communique par channels.
- **Taille** (au 2026-05-22) : **~17 000 lignes** de C sur 23 fichiers `.c` + 3 fichiers `.h`. Plus grands : `text.c` (2123), `exec.c` (2022), `ecmd.c` (1396), `xfid.c` (1337), `acme.c` (1197), `look.c` (944), `rows.c` (913), `regx.c` (843), `wind.c` (736).
- **Build** : `mkfile` Plan 9 (`mk`), via `$PLAN9/src/mkone` et `$PLAN9/src/mkdirs`.

## 2. Structure du répertoire

```
src/cmd/acme/
├── *.c, *.h            — sources et en-têtes (23 .c + 3 .h)
├── mkfile              — règles de build mk
├── CLAUDE.md           — instructions pour assistants IA
├── codebase_analysis.md — ce fichier
├── .ai-jail            — garde-fou pour assistants IA (script de sandbox)
├── .gitignore          — ignore `likeplan9/`
├── .claude/            — configuration locale Claude Code
├── tests/              — suite de tests (créée pour Emph)
│   ├── emph_test.c     — logique pure des ranges (groupes A-F)
│   ├── g_test.c        — intégration regex rxcompile/rxexecute (G1-G4)
│   ├── test_ctl.rc     — tests d'interface 9P (incl. verbes `nctl`)
│   └── mkfile
└── (mail/ — sous-projet séparé, exclu de cette analyse)
```

> Notes :
> - Il n'existe **pas** de fichiers `ACME-SPEC` / `ACME-USER` dans ce
>   répertoire (d'anciennes versions du `CLAUDE.md` y faisaient référence).
> - Les fichiers d'historique de développement (`NOTES`, `TASK`,
>   `STAGE0_SUMMARY.md`, `CODE_CHANGES.md`) qui documentaient les phases
>   d'implémentation de la fonctionnalité Emph ont été retirés du
>   répertoire ; le contenu équivalent (changements vs amont, historique
>   du design) est résumé dans `$PLAN9/CODE_CHANGES.md` (à la racine du
>   tree) et dans `$PLAN9/llm/` (skills `acme/` et `rc/`).
> - La documentation de la fonctionnalité Emph vit dans les manpages
>   plan9port : `man/man1/acme.1` (CLI, commandes), `man/man4/acme.4`
>   (verbes `nctl`/`ctl`) et `man/man3/frame.3` (extensions libframe).
> - Le répertoire `include/` mentionné dans une version précédente de ce
>   document était une copie de travail locale ; il n'est plus présent ici
>   et n'est plus nécessaire (les en-têtes plan9port sont consommés
>   directement depuis `$PLAN9/include/` au build).

## 3. Décomposition par fichier

### Cœur applicatif

- **`acme.c`** (1197 lignes) : `threadmain`, parsing CLI (`-f` police variable, `-F` chasse fixe, `-e`/`-E` polices d'emphase, `-C colorspec` couleur d'emphase globale), `fontnames[4]` global, démarrage des threads, boucle principale d'événements. Fournit `rfget(int fix, int save, int setfont, char *name)` (~ligne 908) — cache de `Reffont` par nom, avec comptage de références. `iconinit` (~ligne 1063) initialise `tagcols[NCOL]` et `textcols[NCOL]`, et alloue `cols[EMPH]` selon `emphcolorspec` (sinon défaut `0x0000AAFF`, bleu foncé).
- **`fsys.c`** (751 lignes) : serveur 9P d'acme. Définit le `Dirtab` (table des fichiers virtuels : `cons`, `index`, `log`, `new`, et par fenêtre `addr`/`body`/`ctl`/`nctl`/`data`/`event`/`tag`/`xdata`/...). Énumération `Q…`/`QW…` au sommet de `dat.h`.
- **`xfid.c`** (1337 lignes) : implémentation des opérations 9P côté serveur.
  - **Crochet `/ctl` par fenêtre** : `xfidctlwrite` gère les verbes `clean`, `dirty`, `show`, `name X`, `font X`, `del`, `delete`, `get`, `put`, `dot=addr`, `addr=dot`, `limit=addr`, `nomark`, `mark`, `cleartag`, `lock`, `unlock`, `menu`, `nomenu`, `dumpdir`, `dump`. Tout `xfidctlwrite` tourne sous `winlock(w, 'F')`.
  - **Crochet `/nctl` par fenêtre** : `xfidnctlwrite` gère les verbes d'emphase `emph=REGEX`, `noemph`, `emphfont <path>`, `emphfont` (reset), `emphcolor <hex>`, `emphcolor` (reset). La lecture de `nctl` renvoie deux champs : `<emphfont> <emphcolor 6 hex>` (cf. § 4).
  - **Crochet `/ctl` global** : `xfidglobalctlread` retourne deux lignes (`font` puis `varfont`) ; `xfidglobalctlwrite` accepte `font <path>` et `varfont <path>` pour changer les polices par défaut et publier dans l'environnement.
- **`exec.c`** (2022 lignes) : table `exectab[]` des commandes acme (Cut, Paste, Del, Edit, Font, Get, Put, Look, Send, Sort, Tab, Undo, Zerox, ...). Dispatch `execute()` qui mappe un nom de commande vers un handler `void f(Text *et, Text *t, Text *argt, int flag1, int flag2, Rune *arg, int narg)`. Commandes d'emphase : `LEmph`/`emph`, `LEmphFont`/`emphfontx`, `LEmphMe`/`emphme`, `LEmphAll`/`emphall`, `LEmphNone`/`emphnone`, `LAutoEmph`/`autoemphx`. Le handler `get` rejoue `emphrecompute`+`emphapply`+`emphauto` ; le handler `fontx` (commande `Font`) appelle `emphfontupdate`.
- **`wind.c`** (736 lignes) : cycle de vie d'une `Window`. `wininit` initialise les champs Emph à zéro/`nil` (incluant `emphcolor = nil`) ; `winclose` appelle `emphfree` qui libère regex, plages, police *et* image de couleur. `winundo` appelle `emphrefresh` sur chaque fenêtre du fichier après undo/redo pour que l'emphase suive.
- **`text.c`** (2123 lignes) : couche `Text` (Frame libframe + buffer) — rendu, sélection, insertion (`textinsert`), suppression (`textdelete`), double-clic, fill, redraw. Contient tout le **pipeline de politique d'emphase** (cf. § 6), `emphsetmetrics` (réglage `lineheight`/`ascent` du `Frame`), `winsetemphcolor` / `winresetemphcolor` (couleur d'emphase par fenêtre), `winensureemphfont`, `emphpattern` (lecture de `lib/emph.regexp`), `emphbyext`, `emphauto`.
- **`cols.c`** (592 lignes), **`rows.c`** (913 lignes) : géométrie (colonnes / lignes de fenêtres) — drag, resize, drop. Les calculs de hauteur de corps utilisent désormais `body.fr.lineheight` (et non `body.fr.font->height`) pour gérer la hauteur de ligne variable quand une police d'emphase plus haute est chargée. `rowdump1` (rows.c:317) sérialise l'état d'emphase par fenêtre (`m <emphon>\n<path>\n<pat>\n`) ; `rowload` le restaure ; `A <autoemph>` global.
- **`scrl.c`** (159 lignes) : barre de défilement.

### Données et buffers

- **`dat.h`** (620 lignes) : structures centrales — `Buffer`, `Block`, `File`, `Elog`, `Text`, `Window`, `Column`, `Row`, `Reffont`, `Range`, `Rangeset`, `Dirtab`, `Fid`, `Xfid`, `Command`, `Mntdir`, `Timer`, `Expand`. Déclare `fontnames[4]`, les channels globaux, et les prototypes des fonctions d'emphase. **`Window`** porte 10 champs d'emphase : `emphon`, `emphpat`/`nemphpat`, `emphmatch`/`nemphmatch`/`aemphmatch`, `emphfont` (`Reffont *`), `emphfontpath` (`char *`), `emphcolor` (`Image *`), `emphcolorrgb` (`ulong`). Globales (dat.c) : `autoemph` (int) — auto-emphase à l'ouverture ; `emphcolorspec` (char*) — spec brute de `-C` ; `emphglobalcolorrgb` (ulong) — couleur d'emphase par défaut (init `0x0000AAFF`).
- **`dat.c`** (65 lignes) : définitions/instances des variables globales (incluant les couleurs d'emphase et `autoemph`).
- **`buff.c`** (325 lignes) : `Buffer` (insertion/suppression de runes, pagination disque via `Block`).
- **`disk.c`** (133 lignes) : stockage temporaire des blocs sur disque.
- **`file.c`** (311 lignes) : `File` (un `Buffer` + historique undo/redo via `delta`/`epsilon`, nom, mtime, sha1).
- **`elog.c`** (354 lignes) : journal des éditions en attente (commandes `Edit`).

### Édition et adresses

- **`edit.h`, `edit.c`** (686 lignes), **`ecmd.c`** (1396 lignes) : langage `Edit` (style `sam` : `a/`, `c/`, `d`, `s///`, `x/`, `g/`, ...). Parseur (yacc-ish hand-written) + AST (`Cmd`, `Addr`) + exécuteur par commande.
- **`addr.c`** (297 lignes) : évaluation des adresses (`#n`, lignes, regex, `,`, `;`, `+`, `-`).
- **`regx.c`** (843 lignes) : moteur regex d'acme (compile/exécute) sur runes. Fonctions clés : `rxcompile`, `rxexecute`, `rxbexecute`. **Stateful global** : `lastregexp` — une seule regex compilée à la fois ; `rxcompile` recompile seulement si le pattern change.
- **`logf.c`** (199 lignes) : fichier `log` (flux d'événements de fenêtres).
- **`look.c`** (944 lignes) : recherche, expansion B3, navigation (`Look`, plumb). `plumbshow` et `openfile` appellent `emphauto` après chargement du contenu.

### Emphase (ajout récent)

- **`emphranges.c`** (118 lignes) : logique **pure** de manipulation des tableaux de `Range`, sans dépendance à acme ni à l'affichage :
  - `emphpush` — append d'une plage, capacité doublée au besoin.
  - `rangeshift` — décale les plages après une insertion/suppression ; **abandonne** les plages qui chevauchent l'édition.
  - `rangemerge` — fusionne deux tableaux triés en gardant l'ordre.
  - `emphfreearr` — libère le tableau et le remet à zéro.
  Testée hors GUI ; couverte par 44 tests (groupes A-F de `tests/emph_test.c`).

### Service / utilitaires

- **`fns.h`** (114 lignes) : prototypes inter-fichiers (la plupart des fonctions d'emphase sont déclarées dans `dat.h` ; `emphauto` est dans `fns.h`).
- **`util.c`** (528 lignes) : `emalloc`, `erealloc`, `estrdup`, `runemalloc`, `warning`, `error`, gestion de processus, utilitaires runes/octets, et `parsecolor(char *spec, ulong *rgb)` (parse `82f`/`8822ff`). Les allocs abortent à l'échec — pas de null-check après.
- **`time.c`** (124 lignes) : timers via channels.
- **`mkfile`** : build Plan 9 mk ; `emphranges.$O` dans `OFILES` ; cibles `test` (recurse dans `tests/`), `likeplan9` et `diffplan9`.

### Tests

- **`tests/emph_test.c`** : runner C autonome, sans framework. Groupes A-F (logique de ranges) — **44 PASS** : A (`emphpush`), B/C (`rangeshift` insertion/suppression), D (compaction multi-ranges), E (`rangemerge`), F (`emphfreearr`).
- **`tests/g_test.c`** : intégration regex (G1-G4, 11 tests) — appelle `rxcompile`/`rxexecute` directement (sans dépendance `textreadc`), couvre notamment les matches vides (`^`, `$`) sans boucle infinie.
- **`tests/test_ctl.rc`** : tests d'interface 9P (T01-T04, T06). SKIP propre (exit 77, TAP) si `/mnt/acme` non monté.
- **`tests/mkfile`** : build des runners.

### Documentation / état

- **`CLAUDE.md`** : guide pour assistants IA (résumé architectural + règles d'écriture du code).
- **`codebase_analysis.md`** : ce fichier.
- **`$PLAN9/CODE_CHANGES.md`** *(à la racine du tree)* : explication détaillée des changements apportés vs l'amont plan9port par la fonctionnalité Emph (couches libframe + acme).
- **`$PLAN9/OBSERVATIONS.md`** : cross-check des skills `llm/acme/SKILL.md` vs le code (lacunes de manpages, verbes `ctl` non documentés, fragilités de scripts).
- **`$PLAN9/llm/acme/SKILL.md`** : catalogue exhaustif des verbes `ctl`/`nctl`, protocole `event`, idiomes de scripting via `/mnt/acme`.

## 4. Endpoints — fichiers virtuels 9P

Acme n'expose pas d'API HTTP mais un système de fichiers 9P. Résumé des
fichiers globaux et par fenêtre sous `/mnt/acme` :

| Fichier      | Sémantique principale |
|--------------|-----------------------|
| `cons`       | sortie standard partagée des commandes |
| `consctl`    | stub de compatibilité |
| `index`      | une ligne par fenêtre (id, tailles, flags, tag) |
| `label`      | stub |
| `log`        | flux des événements de fenêtres (new/zerox/get/put/del) |
| `new/`       | accéder à un fichier ici crée une nouvelle fenêtre |
| `ctl` (racine) | lecture : deux lignes (`font`, `varfont`) ; écriture : `font <path>` / `varfont <path>` pour changer les polices par défaut (et `$font` / `$varfont`) |
| `<id>/addr`  | définit/lit l'adresse courante (pour `data`/`xdata`) |
| `<id>/body`  | corps de la fenêtre |
| `<id>/tag`   | barre de tag |
| `<id>/data`  | accès aléatoire au corps, indexé par `addr` |
| `<id>/xdata` | comme `data`, lecture bornée à fin d'`addr` |
| `<id>/ctl`   | lecture : 10 entiers (largeurs fixes 11 char) + nom de police `%q`-quoté ; écriture : verbes texte (`dot=addr`, `addr=dot`, `limit=addr`, `clean`, `dirty`, `name X`, `font X`, `del`, `delete`, `get`, `put`, `nomark`, `mark`, `cleartag`, `lock`, `unlock`, `menu`, `nomenu`, `show`, `dump`, `dumpdir`) |
| `<id>/nctl`  | emphase par fenêtre. Lecture : une ligne `<emphfont> <emphcolor 6 hex>` (police non quotée, couleur en `%06lx`). Écriture : `emph=REGEX`, `noemph`, `emphfont <path>`, `emphfont` (toggle/reset), `emphcolor <hex>` (3 ou 6 chiffres), `emphcolor` (reset défaut global) |
| `<id>/event` | rapporte/intercepte les actions souris B2/B3 |
| `<id>/errors`| ajoute au `+Errors` du dossier |
| `<id>/editout` | sortie des commandes `Edit` |

## 5. Architecture interne — flux des données

```
              +------------------+
              |  threadmain()    |  acme.c
              +---------+--------+
                        |
        +---------------+---------------+----------------+
        |               |               |                |
        v               v               v                v
  +-----------+   +-----------+   +-----------+   +-------------+
  | keyboard  |   | mouse     |   | plumb     |   | 9P server   |
  | thread    |   | thread    |   | thread    |   | (fsys.c)    |
  +-----+-----+   +-----+-----+   +-----+-----+   +------+------+
        |               |               |                |
        |   channels (cwait, ckill, ccommand, ...)        |
        +---------------+---------------+----------------+
                        |
                        v
                 +-------------+
                 |  Window     |  wind.c  (locked by Window.lk)
                 |  +--------+ |
                 |  |  Text  | |  text.c  -> Frame (libframe)
                 |  | tag    | |
                 |  +--------+ |
                 |  |  Text  | |
                 |  | body   | |  -> File -> Buffer -> Block*  (disque)
                 |  +--------+ |
                 +------+------+
                        |
                        v
                 +-------------+
                 |  Disk       |  disk.c (stockage paginé)
                 +-------------+

  Edit (sam-style)  :  ecmd.c  ->  edit.c  ->  elog.c  ->  buffer
  Regex             :  regx.c  (rxcompile / rxexecute / rxbexecute)
  Adresses          :  addr.c  (#n, /re/, ., $, , ; + -)
  Exécution cmds    :  exec.c  (exectab[] -> handlers)
  Contrôle 9P       :  fsys.c (Dirtab) -> xfid.c (xfidctlwrite, ...)
  Emphase regex     :  text.c (setemph, emphrecompute, emphapply)
                       + emphranges.c (logique pure des ranges)
                       + libframe (frsetboxfont, police par boîte)
```

### Pile de données (bas vers haut)

```
Block (disk.c)   blocs runes de taille fixe, pageable disque
  ↑
Buffer (buff.c)  séquence de runes avec bloc courant en cache
  ↑
File (file.c)    Buffer + delta/epsilon undo + nom/mtime/sha1
  ↑
Text (text.c)    File + libframe Frame + Reffont + sélection + géométrie
  ↑
Window (wind.c)  tag Text + body Text + état 9P + état d'emphase
  ↑
Column / Row     géométrie (cols.c, rows.c)
```

### Cycle d'une écriture `ctl` ou `nctl` (ex. `dot=addr`, `emph=foo`)

1. Client écrit dans `/mnt/acme/<id>/ctl` (verbes généraux) ou `/mnt/acme/<id>/nctl` (verbes d'emphase).
2. `fsys.c` route le message 9P vers un `Xfid` ; `xfidwrite` détecte `QWctl` ou `QWnctl`.
3. `xfidctlwrite` / `xfidnctlwrite` (xfid.c) lit ligne par ligne, compare le préfixe contre une liste de mots-clés.
4. Le handler manipule la `Window`/`Text` correspondante sous `winlock(w, 'F')` (pris en amont par le dispatcher xfid).
5. Réponse 9P renvoyée au client.

### Cycle d'une commande utilisateur (ex. `Edit`, `Emph`)

1. L'utilisateur clique B2 sur le mot dans un tag/corps.
2. `execute()` (exec.c) cherche dans `exectab[]`.
3. Handler trouvé (ou commande shell sinon) sur le thread editor.
4. Le handler `emph` ne prend **pas** de verrou interne : la sérialisation
   est garantie par l'appelant (voir § 11, piège du deadlock).

## 6. Polices et emphase — état au 2026-05-22

La fonctionnalité d'emphase (`Emph`) est **implémentée et opérationnelle**.
Elle met en évidence toutes les occurrences d'un regex dans le corps d'une
fenêtre, en les rendant dans une **police différente**, une **couleur
différente** et avec une **hauteur de ligne pouvant différer**.

### Polices

Tableau `fontnames[4]` (acme.c, dat.h) :
- `[0]` police variable principale (`-f`)
- `[1]` police à chasse fixe principale (`-F`)
- `[2]` police variable d'emphase (`-e`) — exportée via `$emphfont`
- `[3]` police à chasse fixe d'emphase (`-E`) — exportée via `$varemphfont`

Chargement via `rfget(int fix, int save, int setfont, char *name)`
(acme.c:908). La police d'emphase d'une fenêtre est chargée **à la
demande** lors du premier `Emph` et conservée dans `Window.emphfont`
(un `Reffont*`). `emphfontname` (text.c:1786) choisit `[2]` ou `[3]`
selon le mode (variable/fixe) du corps de la fenêtre ; `emphfontupdate`
(text.c:2009) la recharge quand la commande `Font` bascule la police du
corps. `winensureemphfont` (text.c:1828) centralise le chargement et
déclenche un `winresize` si la police change.

**Hauteur de ligne variable.** libframe gère désormais une hauteur de
ligne `Frame.lineheight` et un ascender `Frame.ascent` distincts de
ceux de `Frame.font`. `emphsetmetrics` (text.c:26) règle ces champs à
`max(body font, emph font)` après chaque chargement de police d'emphase
et appelle `frinittick`. Les calculs de géométrie (`cols.c`, `rows.c`,
`wind.c`) utilisent désormais `body.fr.lineheight`, **pas**
`body.fr.font->height` — sans cela une police d'emphase plus haute fait
déborder le corps sur la tagline (cf. § 11).

### Couleur d'emphase

Deux niveaux :

1. **Couleur globale** : `-C colorspec` (acme.c) parse vers
   `emphglobalcolorrgb` (dat.c, défaut `0x0000AAFF` — bleu foncé).
   `iconinit` (acme.c:1063) alloue `tagcols[EMPH]` et `textcols[EMPH]`
   à partir de cette valeur ; ces images servent de couleur de défaut
   pour le frame de toute nouvelle fenêtre.
2. **Couleur par fenêtre** : `Window.emphcolor` (Image*) +
   `Window.emphcolorrgb` (ulong). `nil` => utilise la globale.
   `winsetemphcolor` (text.c:1795) alloue une nouvelle image,
   l'affecte à `w->body.fr.cols[EMPH]`, et stocke RGB.
   `winresetemphcolor` (text.c:1815) libère l'image et restaure le
   défaut global.

`parsecolor(char *spec, ulong *rgb)` (util.c:500) accepte 3 ou 6
chiffres hexadécimaux RGB :
- 3 chiffres : chaque chiffre est doublé (ex: `82f` → `0x8822ffff`)
- 6 chiffres : utilisé tel quel (ex: `8822ff` → `0x8822ffff`)

libframe a été étendu avec un slot `EMPH` dans l'enum des couleurs
(frame.h, `NCOL` passe de 5 à 6). Les fonctions de dessin
`_frdrawtext()` et `frdrawsel0()` (frdraw.c) utilisent `f->cols[EMPH]`
au lieu de `f->cols[TEXT]` lorsqu'une boîte a une police personnalisée
(`b->font != nil`).

### Auto-emphase

`autoemph` (global int, dat.c) — quand actif, toute ouverture de fichier
applique automatiquement la regex de `lib/emph.regexp` correspondant à
l'extension du fichier. Workflow :

- `emphpattern(char *ext)` (text.c:2039) cherche d'abord
  `$PLAN9/lib/emph.regexp`, puis `$HOME/lib/emph.regexp`, retourne une
  rune-string contenant le pattern pour `ext`.
- `emphbyext(Window *w)` (text.c:2085) extrait l'extension via
  `fileext`, appelle `emphpattern`, et lance `setemph` avec le pattern.
- `emphauto(Window *w)` (text.c:2115) garde-fou : ne fait rien si
  `autoemph` est désactivé ou si la fenêtre a déjà une emphase active.
- Crochets d'appel : `get()` (exec.c:692), `readfile()` (acme.c),
  `plumbshow()` et `openfile()` (look.c:311, 877).
- Commande utilisateur : `AutoEmph` (`autoemphx`, exec.c:1291) — toggle
  global, affiche un warning.
- Persistance : `rowdump1` sérialise `A <autoemph>` dans le dump global ;
  `rowload` le restaure.

Commandes utilisateur connexes :

- `Emph [regex]` — pose ou togglette l'emphase de la fenêtre courante.
- `EmphMe` (exec.c:1225) — applique `emphbyext` sur la fenêtre courante.
- `EmphAll` (exec.c:1241) — applique `emphbyext` à toutes les fenêtres
  non-scratch.
- `EmphNone` (exec.c:1267) — coupe l'emphase de toutes les fenêtres.
- `EmphFont [path]` — fixe / réinitialise la police d'emphase courante.

### Architecture en deux couches

```
   COUCHE ACME (la politique : quoi emphaser, quelle couleur, quelle police)
   ---------------------------------------------------------------
   commande Emph / nctl emph=        ->  setemph()
   emphmatch[] : liste triée des plages qui matchent le regex
   emphapply() : projette emphmatch[] sur les boîtes visibles
   emphsetmetrics() : règle Frame.lineheight / .ascent
   winsetemphcolor() : affecte Frame.cols[EMPH] à l'image par-fenêtre
                          |
                          v   appelle frsetboxfont(), frrelayout()
   ---------------------------------------------------------------
   COUCHE LIBFRAME (le mécanisme : rendre boîte par boîte)
   ---------------------------------------------------------------
   Frbox.font : police propre à la boîte (nil => police du frame)
   Frame.cols[EMPH] : couleur de premier plan pour texte emphasé
   Frame.lineheight / .ascent : métriques de ligne
   FRBOXFONT(f,b) : macro qui choisit la bonne police
   frsetboxfont() : affecte une police à une plage de boîtes
   _frdrawtext() / frdrawsel0() : utilisent cols[EMPH] si b->font != nil
```

### État de l'emphase (couche acme)

- **`/nctl` verbes** : `emph=REGEX`, `noemph`, `emphfont <path>`, `emphfont` (reset/toggle), `emphcolor <hex>`, `emphcolor` (reset). Implémentés dans `xfidnctlwrite` (xfid.c, sous `winlock('F')`).
- **Commandes utilisateur** : `Emph regex` (B2), `Emph` seul (toggle on/off du dernier pattern), chord 2-1, plus `EmphMe`, `EmphAll`, `EmphNone`, `EmphFont`, `AutoEmph`.
- **Point d'entrée unique** : `setemph` (text.c, ~ligne 1870) — branche « off » : éteint et redessine ; branche « on » : compile le regex, charge la police à la demande via `winensureemphfont`, recalcule `emphmatch[]`, applique, redessine.
- **Source de vérité** : `Window.emphmatch[]` — tableau dynamique trié, sans chevauchement, de `Range` ; stocke des **offsets dans le fichier** (survivent au défilement).
- **Calcul des matches** : `emphrecompute` (text.c:1893) balaye tout le fichier via `rxexecute` ; `emphrefreshlocal` (text.c:1927) ne réexamine qu'une fenêtre locale autour d'une édition. `emphrefresh` (text.c:1915) = `emphrecompute` + `emphapply` + redraw (utilisé après undo/redo).
- **Application au rendu** : `emphclear` (text.c:1734) remet toutes les boîtes en police standard ; `emphapply` (text.c:1742) efface tout puis applique la police d'emphase aux boîtes des plages *visibles*, en convertissant les offsets fichier en offsets frame via `body.org`, puis rejoue la mise en page via `frrelayout` (libframe) + `textfill`. `emphapplylocal` (text.c:1777) fait suivre d'un `frredraw`.
- **Hooks d'édition** : `textinsert` et `textdelete` appellent `emphshift` + `emphrefreshlocal` + `emphapplylocal`. **Chemin clavier-cache** (`tofile=FALSE`) : la re-recherche est différée à `textcommit` (le fichier est encore stale). **Chemin undo/redo** : également `tofile=FALSE` mais le fichier *est* à jour — `winundo` (wind.c:380) appelle `emphrefresh` (recompute+apply complet) pour toutes les fenêtres du fichier. `textredraw` / `textsetorigin` rejouent `emphapply` après remplissage du frame — sinon l'emphase disparaît au scroll.
- **Couplage avec `Font`** : `fontx` (exec.c) appelle `emphfontupdate` après avoir changé la police du corps — un seul `Font` bascule corps *et* emphase (variable <-> fixe).
- **Cycle de vie** : `wininit` initialise tous les champs Emph à zéro/`nil` ; `winclose` -> `emphfree` libère regex, plages, police (`rfclose`) *et* image de couleur (`freeimage`).
- **Persistance** (rows.c:317) : `rowdump1` émet pour chaque fenêtre emphasée `m <emphon>\n<path>\n<pat>\n` ; `rowload` les restaure. La couleur par fenêtre n'est *pas* persistée actuellement (à confirmer si on étend le format dump).
- **Tests** : 44 PASS pour la logique de ranges (A-F), 11 pour l'intégration regex (G1-G4), tests rc pour `nctl` (T01-T04, T06).
- **Documentation** : manpages `man/man1/acme.1` (commandes), `man/man4/acme.4` (`nctl`), `man/man3/frame.3` (lineheight/ascent/EMPH).

## 7. Environnement et build

- **Pré-requis** : installation plan9port (`$PLAN9` défini, binaires `mk`, `9c`, etc. sur le `PATH`).
- **Construction** :
  ```
  mk             # build acme + mail/
  mk install     # installe dans $PLAN9/bin
  mk clean
  mk test        # recurse dans tests/, lance o.emph_test, o.g_test, test_ctl.rc
  mk likeplan9   # vue "comme Plan 9" via sed (rewrites de field access)
  mk diffplan9   # diff vs ../plan9 (sibling upstream) pour voir la dérive
  ```
- **Variables d'environnement notables** : `$PLAN9`, `$font`, `$emphfont`, `$acmeshell`, `$home`.
- **Lancement** :
  ```
  acme [-a] [-c ncol] [-C colorspec] [-f varfont] [-F fixfont] [-e emphfont] [-E emphfixfont]
       [-l loadfile] [-W winsize] [file ...]
  ```
- **Pas de CI/CD** : ni `.github/workflows`, ni Dockerfile, ni `.gitlab-ci.yml`. Build et test sont locaux.

## 8. Pile technologique

| Couche        | Choix |
|---------------|-------|
| Langage       | C (style Plan 9, ANSI) |
| Concurrence   | `libthread` (CSP : threads coopératifs + channels) |
| Graphique     | `libdraw`, `libframe` (frames de texte runes, police par boîte) |
| Texte         | Runes Unicode (UTF-8 dehors, rune-32 dedans) — convertir uniquement aux frontières 9P/disque (`bytetorune`, `runetobyte`, `cvttorunes`) |
| Regex         | Moteur interne `regx.c` sur runes (`rxcompile`, `rxexecute`, `rxbexecute`) |
| Stockage      | Buffer paginé en blocs, swap disque (`disk.c`) |
| IPC externe   | 9P via `9pserve` |
| Build         | `mk` (plan9port) |
| Plumbing      | `libplumb` (routage de messages inter-apps) |
| Tests         | Runner C autonome (`emph_test.c`, `g_test.c`) + script `rc` (`test_ctl.rc`). Aucun framework externe. |

## 9. Schéma visuel

```
       Utilisateur (souris, clavier)
                 |
                 v
   +--------------------------+         +----------------------+
   |  acme (process)          |<------->|  9pserve (process)   |
   |  threads CSP             |  9P     |  -> /mnt/acme        |
   |  - mouse, kbd, plumb     |         +----------------------+
   |  - command dispatcher    |                  ^
   |  - 9P server (fsys.c)    |                  |
   |  - per-Xfid worker       |                  |
   +--------------------------+         +----------------------+
                 |                      | Programmes externes  |
                 v                      | (sed, awk, win, ...) |
   +--------------------------+         | echo emph=foo > nctl |
   |  Window (lk-protected)   |         | echo font X > ctl    |
   |  - tag Text              |         +----------------------+
   |  - body Text             |
   |  - File -> Buffer        |
   |  - emphmatch[] (regex)   |
   |  - emphcolor (Image*)    |
   +--------------------------+
                 |
                 v
   +--------------------------+        +----------------------+
   |  libframe (Frame)        |<------>|  libdraw (Image)     |
   |  - box[] layout          |        |  -> X11/devdraw      |
   |  - Frbox.font par boîte  |        +----------------------+
   |  - FRBOXFONT / frsetbox  |
   |  - lineheight / ascent   |
   |  - cols[EMPH] (par-w)    |
   +--------------------------+
```

## 10. Qualité, observations

- **Style** : très Plan 9 — fonctions courtes, peu de commentaires, identifiants concis. Pas de tests automatisés historiquement ; les tests présents (`tests/`) ont été ajoutés *pour* la fonctionnalité Emph.
- **Sécurité / robustesse** : code mature, usage intensif de pointeurs bruts et `emalloc` (qui aborte au lieu de retourner nil). Rester rigoureux sur la libération des `Rune*` lorsqu'on ajoute des champs `Window`.
- **Concurrence** : tout passage par `xfidctlwrite` est sérialisé par `winlock` — bonne pratique pour tout nouveau verbe `/ctl`. Le handler `emph` ne verrouille **pas** : la sérialisation vient de l'appelant. Reprendre `winlock` y provoquerait un deadlock (`qlock` non réentrant — cf. § 11).
- **Compatibilité Plan 9 amont** : la cible `likeplan9` du `mkfile` aligne ce fork via des `sed`. Toute extension locale (fontnames[2..3], `emphranges.c`, `Frbox.font`) divergera — surveiller avec `mk diffplan9`. Si on renomme un champ ciblé par les `sed` (`fcall`, `lk`, `b`, `fr`, `ref`, `m`, `u`, `u1`), mettre la règle à jour.
- **Rétrocompatibilité libframe** : une boîte dont `font == nil` se comporte exactement comme avant ; `sam`, `samterm`, `9term` n'appellent jamais `frsetboxfont` et ne voient aucun changement. Ces programmes initialisent `cols[EMPH]` à `display->black` pour compatibilité avec l'extension NCOL=6.
- **Documentation** : les manpages `man/man1/acme.1` et `man/man4/acme.4` doivent être mises à jour à chaque ajout de commande utilisateur ou de verbe `/ctl`.
- **Dépendance Dirtab** : ne pas casser l'ordre du `Dirtab` ni l'énumération `Q…`/`QW…` — les macros `QID` et `FILE` en dépendent.

## 11. La fonctionnalité Emph — historique et conception

### Le problème central

libframe découpe le texte visible en **boîtes** (`Frbox`) et utilisait
*toujours* une unique police (`frame->font`). Pour emphaser, il faut que
certaines boîtes soient rendues dans une autre police : libframe était par
construction « mono-police ».

### Approche abandonnée : l'overlay

Une première implémentation **superposait** un second passage de dessin
(`textemphdraw` / `emphpaint`) par-dessus le rendu standard. Cette approche
a produit cinq bugs structurels (sélection souris incohérente, artefacts de
pixels, emphase fantôme après changement de regex, crashs) — **insolubles**
tant que libframe restait mono-police. Elle a été entièrement supprimée.

### Approche retenue : police par boîte

libframe a été étendu pour qu'une boîte porte sa *propre* police. La
politique (« quelles plages emphaser ») reste dans acme ; libframe ne gagne
que la *capacité* technique de rendre boîte par boîte. Modifications :

- `include/frame.h` : champ `Frbox.font`, macro `FRBOXFONT(f,b)`, prototype `frsetboxfont`.
- `src/libframe/frboxfont.c` *(nouveau)* : `frsetboxfont` — découpe les boîtes aux bornes de la plage puis affecte la police, recalcule `b->wid`.
- `src/libframe/frdraw.c`, `frutil.c`, `frbox.c`, `frptofchar.c`, `frinsert.c` : remplacement de `f->font` par `FRBOXFONT(f,b)` aux points de dessin, de mesure et de conversion pixel↔caractère ; `_frclean` interdit de fusionner deux boîtes de polices différentes ; `bxscan` initialise `b->font = nil`.

### Pièges résolus (récapitulatif `CODE_CHANGES.md`)

| Problème | Cause | Solution |
|---|---|---|
| libframe mono-police | `f->font` partout | champ `Frbox.font` + macro `FRBOXFONT` |
| Approche overlay buggée | second passage de dessin fragile | abandonnée au profit de la police par boîte |
| Crash au clic souris | `frptofchar` marchait en police standard sur des boîtes plus larges | `FRBOXFONT` dans `frptofchar.c` |
| Emphase perdue au scroll / `Get` | boîtes reconstruites avec `font = nil` | rejouer `emphapply` dans `textsetorigin`, `textredraw`, `get` |
| Fusion de boîtes incohérente | `_frclean` fusionnait des polices différentes | garde `b[0].font == b[1].font` |
| Deadlock de la commande `Emph` | `winlock` non réentrant, déjà tenu par l'appelant | suppression du verrou interne dans le handler |
| Emphase fantôme après collage | `emphapply` corrige la donnée mais ne repeint pas | `emphapplylocal` fait suivre d'un `frredraw` |
| Débordement / crash après `Emph` (police plus large) | `frsetboxfont` élargit les boîtes sans refaire le repli des lignes | `frrelayout` rejoue `_frdraw` ; `emphapply` l'appelle puis `textfill` |
| Police d'emphase fixe (`-E`) jamais utilisée | `setemph` chargeait toujours `fontnames[2]` | `emphfontname` choisit `[2]`/`[3]` selon le mode du corps |
| `Font` ne basculait pas l'emphase | `fontx` ignorait `w->emphfont` | `emphfontupdate` appelé après le changement de police |
| Corps déborde sur la tagline avec une police d'emphase plus haute | géométrie utilisait `font->height` | `Frame.lineheight`/`Frame.ascent` ajoutés ; `emphsetmetrics` les règle ; `cols.c`/`rows.c`/`wind.c` migrés vers `lineheight` |
| Crash au cut/paste avec emphfont plus large | `textinsert`/`textdelete` avec `tofile=FALSE` re-calcule sur un fichier stale | différer le refresh à `textcommit` quand `tofile=FALSE` |
| Emphase fausse après undo/redo | chemin `tofile=FALSE` mais fichier déjà courant | `winundo` appelle `emphrefresh` (full recompute+apply) pour toutes les fenêtres du fichier |

Détail des matchs : après chaque match on avance `p = q1` ; mais si le
match est *vide* (`q0 == q1`, ancres `^`/`$`), on avance `p = q0 + 1` pour
éviter une boucle infinie.

### Ajouts ultérieurs (post-refonte par-boîte)

Une fois la mécanique de base stable, plusieurs extensions ont été
greffées sans toucher au cœur libframe :

- **Couleur par fenêtre** (`b1dbf31f`, `f484ab5d`) : `Window.emphcolor`
  + verbes `emphcolor` dans `nctl`.
- **Auto-emphase** (`1e8ea952`) : flag global `autoemph`, lookup
  `lib/emph.regexp`, hooks dans `get`/`readfile`/`plumbshow`/`openfile`.
- **Persistance** (`8763ef88`, `9156e7e5`) : `rowdump1` / `rowload`.
- **Contrôle global des polices via `/ctl`** (`279661d2`) : `font <path>`
  / `varfont <path>` au niveau racine (`xfidglobalctlwrite`).
- **Couplage des polices** (`c9ddda2d`) : la commande `EmphFont` sans
  argument togglette entre « suivre la police du corps » et police
  distincte.

---

Fichier mis à jour le 2026-05-22. Reflète l'état du code **après** :
refonte par-boîte-police, ajout de la hauteur de ligne variable
(`Frame.lineheight`/`Frame.ascent`), couleur d'emphase par fenêtre
(`Window.emphcolor`), auto-emphase (`autoemph` + `lib/emph.regexp`),
persistance dans les dumps de fenêtre, et contrôle global des polices
via le fichier `/ctl` racine.
