# rs274 natif — paquet AUR `linuxcnc`, corrigé pour Arch

Ce dossier existe pour une seule raison : **le visualiseur de parcours G-code a
besoin de `rs274`**, l'interpréteur de LinuxCNC, et rien d'autre de LinuxCNC.

Il tournait dans un conteneur distrobox (Debian + `linuxcnc-uspace`), supprimé
le 12/08/2026. Le natif est **douze fois plus rapide** : 16 ms par appel contre
190, et la suite de tests du visualiseur passe de 14 s à 5 s. Chaque appel
payait un `distrobox-enter`, et le visualiseur en fait un par fichier chargé.

Depuis, l'**interface** a suivi : `qtdragon_hd`, l'écran de la machine, tourne
aussi sur ce poste. Il a fallu quatre dépendances manquantes et un paquet
entier à écrire — voir « Ce que le paquet AUR oublie » plus bas.

| | |
|---|---|
| `PKGBUILD.corrige` | la recette AUR **avec** les correctifs |
| `correctif-aur.diff` | les correctifs seuls, à réappliquer sur l'original |
| `python-qscintilla-qt5/` | un paquet à part, sans lequel aucune interface Qt de LinuxCNC ne démarre sur Arch |

**Ce dépôt ne contient que des recettes.** Les paquets compilés, les archives
sources et les dossiers de travail de `makepkg` en sont exclus : 17 Ko suivis
pour 65 Mo sur le disque. Tout se refabrique, et chaque `PKGBUILD` porte
l'URL et la somme de contrôle de ses sources.

---

## Installer

```bash
makepkg -si
```

Depuis ce dossier, avec `PKGBUILD.corrige` renommé en `PKGBUILD` — ou en
réappliquant `correctif-aur.diff` sur la recette fraîche de l'AUR, ce qui est
préférable après une mise à jour amont. La section « Reconstruire » plus bas
donne la marche à suivre.

Vérifier tout de suite :

```bash
printf 'G21 G90\nG0 X10 Y10\nG1 X20 F100\nM2\n' > /tmp/t.ngc && : > /tmp/t.var && rs274 -g -v /tmp/t.var /tmp/t.ngc | head -3
```

Des lignes `USE_LENGTH_UNITS(...)` : c'est bon. Sinon, revenir en arrière avec
`sudo pacman -R linuxcnc`.

---

## Le correctif, et pourquoi il existe

Le `configure` de LinuxCNC 2.9.10 cherche Python en descendant : 3.13, puis
3.12, puis 3.11… **Sa liste s'arrête à 3.13**, écrite avant qu'Arch ne passe à
3.14. Il ne trouve donc pas le Python du système, descend d'un cran, et tombe
sur le `python3.12` installé ici pour tout autre chose — où ni `gi` ni
`boost_python314` n'existent. Il s'arrête alors sur :

```
checking for python pango module... ModuleNotFoundError: No module named 'gi'
configure: error: Python pango module not found!
install with "sudo apt-get install python3-gi"
```

Le conseil `apt-get` sur une machine Arch a de quoi égarer : le paquet est bien
installé, c'est l'interpréteur qui n'est pas le bon.

### Avant

```bash
  ./autogen.sh

  ./configure \
    --prefix=/usr \
```

### Après

```bash
  ./autogen.sh

  PYTHON=/usr/bin/python3 \
  ./configure \
    --prefix=/usr \
```

Ce n'est pas une commande de plus : la barre oblique colle les deux lignes, et
un `NOM=valeur` posé devant une commande ne vaut **que pour elle** — rien n'est
modifié dans l'environnement. Les deux contrôles qui échouaient passent alors :

```
checking whether boost_python314 is the correct library... yes
checking for python pango module... found
```

---

## « can't find package Linuxcnc » — ce n'est PAS un correctif manquant

Symptôme, au lancement de l'**interface** (`rs274` seul n'est pas concerné,
il n'utilise pas Tcl) :

```
can't find package Linuxcnc
    while executing "package require Linuxcnc "
    (file "/usr/share/linuxcnc/hallib/check_config.tcl" line 224)
check_config validation failed
```

Le paquet AUR **pose déjà** le correctif : il installe
`/etc/profile.d/linuxcnc.sh`, qui exporte
`TCLLIBPATH=/usr/lib/tcltk/linuxcnc`. C'est nécessaire parce que Tcl, sur Arch,
ne cherche que dans `/usr/lib/tcl8.6` et `/usr/lib`, et ne descend que d'**un**
niveau : `/usr/lib/tcltk/linuxcnc` ne serait jamais vu. (Debian s'en passe, son
Tcl ayant `/usr/lib/tcltk` dans son chemin par défaut.)

**Le piège est ailleurs** : `/etc/profile.d/` n'est lu que par les shells de
**connexion**. Un terminal ordinaire n'en ouvre pas — la variable reste vide, et
tout a l'air cassé alors que tout est correctement installé.

Vérifier d'abord :

```bash
echo "${TCLLIBPATH:-vide}"
```

Vide ⇒ ce n'est pas le paquet, c'est le shell. Trois façons de s'en sortir,
de la plus ponctuelle à la plus durable :

```bash
# 1. pour cette fois
TCLLIBPATH=/usr/lib/tcltk/linuxcnc linuxcnc <config.ini>

# 2. pour ce terminal
. /etc/profile.d/linuxcnc.sh

# 3. pour toujours — c'est ce qui est en place depuis le 12/08/2026
#    (dans ~/.bashrc, géré par chezmoi)
[ -r /etc/profile.d/linuxcnc.sh ] && . /etc/profile.d/linuxcnc.sh
```

La ligne 3 **source** le fichier du paquet au lieu d'y recopier le chemin : si
le paquet change un jour d'emplacement, on suit sans rien corriger.

Se déconnecter puis se reconnecter marche aussi, mais ne règle rien pour les
terminaux non-connexion des jours suivants.

---

## Ce que le paquet AUR oublie

Quatre dépendances, ajoutées le 13/08/2026. Chacune s'est signalée par une
panne, et aucune ne se devine à la lecture de la recette.

**`python-pyqt5`** — sans lui `qtvcp` ne démarre pas. La panne est trompeuse :
les modules temps réel, HAL, l'IO et `milltask` se chargent d'abord, tout a
l'air normal, et seul l'affichage tombe sur un `ModuleNotFoundError`.

**`python-xlib`** — `gmoccapy` refuse de démarrer sans, en conseillant un
`apt install`. `qtvcp`, lui, s'en contente d'un avertissement : la dépendance
est donc obligatoire ou non **selon l'interface**, ce qui invite à la ranger
en optionnelle. C'est une erreur.

**`tclx`, déplacé de `makedepends` vers `depends`.** `latency-histogram` fait
un `package require Tclx` **sec** à l'exécution — `hal-histogram`, lui,
l'entoure d'un `catch` et s'en passe. En dépendance de compilation, `tclx`
ressortait **orphelin** après le build ; or il épingle `tcl` à une version
exacte. Le jour où CachyOS a republié `tcl` avec son suffixe `znver4`, cet
épinglage a bloqué **toute mise à jour du système** :

```
:: l'installation de tcl (8.6.16-1.1) casse la dépendance « tcl=8.6.16-1 »
   requise par tclx
```

Déclaré en dépendance de fonctionnement, il ne peut plus devenir orphelin.

---

## `python-qscintilla-qt5` — le paquet qui n'existe plus

Arch a suivi Qt6 et ne fournit plus que `python-qscintilla-qt6`. Or l'éditeur
de G-code de `qtvcp` est en PyQt5 :

```python
try:
    from PyQt5.Qsci import QsciScintilla, QsciLexerCustom, QsciLexerPython
except ImportError as e:
    LOG.critical("Can't import QsciScintilla ...")
    sys.exit(1)
```

`sys.exit(1)` : aucune dégradation possible, **tout écran `qtvcp` meurt avec
lui** — `qtdragon`, `qtdragon_hd`, `qtaxis`. Rien dans l'AUR ne comble le trou.

La bibliothèque C++ est pourtant là : `qscintilla-qt5` est dans `extra`. Seules
les liaisons Python manquent, et c'est tout ce que ce paquet fabrique.

```bash
cd python-qscintilla-qt5 && makepkg -si
```

Deux pièges y sont documentés en tête, tous deux vérifiés après construction :

* **l'archive PyPI embarque le C++.** `project.py` décide de son mode par
  `qsci_external_lib = not os.path.isdir('src')` : construite telle quelle,
  elle recompilerait sa propre `libqscintilla2_qt5.so` et l'installerait
  par-dessus celle du système. D'où le retrait de `src` et `scintilla` dans
  `prepare()`.
* **le choix Qt5/Qt6 se fait d'après ce que rend `qmake`.** `/usr/bin/qmake`
  est en Qt5 aujourd'hui ; rien ne le garantit demain. D'où
  `--qmake /usr/bin/qmake-qt5` explicite — sans lui, une mise à jour d'Arch
  produirait un jour des liaisons Qt6 **sans un mot**, et l'échec ressemblerait
  au précédent.

Vérifier après coup que c'est bien la bibliothèque du système qui est employée :

```bash
ldd /usr/lib/python3*/site-packages/PyQt5/Qsci.abi3.so | grep -E "Qt5|qscintilla"
```

`libQt5*` et `libqscintilla2_qt5.so.15` : c'est juste. Un `libQt6` quelque part
signifierait que `qmake` a changé sous les pieds du paquet.

**À rejouer quand `qscintilla-qt5` changera de version** : les liaisons Python
doivent suivre la bibliothèque C++ au numéro près.

---

## Reconstruire après une mise à jour du paquet

Un `git pull` de l'AUR **écrasera** le PKGBUILD corrigé et la compilation
échouera de nouveau, exactement de la même façon.

```bash
git clone https://aur.archlinux.org/linuxcnc.git
cd linuxcnc
patch PKGBUILD < ~/Projets/machine/aur-linuxcnc/correctif-aur.diff
makepkg -si
```

Si le correctif ne s'applique plus, c'est que le `prepare()` a changé : rouvrir
ce fichier, la section « Avant / Après » dit quoi remettre et où.

Le vrai remède serait que le mainteneur AUR l'intègre — cinq lignes à lui
envoyer, et ce dossier deviendrait inutile.

---

## Revenir au conteneur

Le natif dépend d'un correctif à réappliquer ; le conteneur, lui, était
**gelé** — insensible à une montée de Python ou de Boost, et il permettait de
figer exactement la version qui tourne sur la machine. Si le natif devient
pénible :

```bash
distrobox create --name lcnc --image debian:bookworm
distrobox enter lcnc -- sudo apt install linuxcnc-uspace
distrobox enter lcnc -- distrobox-export --bin /usr/bin/rs274
```

**Prendre la version du dépôt LinuxCNC, pas celle de Debian** : bookworm livre
`2.9.0~pre1`, dont le `rs274` multiplie par 25,4 tous les décalages de la table
d'outils, `[TRAJ] LINEAR_UNITS = mm` ou non. Voir le README du visualiseur.

Rien à changer dans le code du visualiseur dans un sens comme dans l'autre :
`rs274` est cherché dans le `PATH`, et `/usr/bin` passe avant `~/.local/bin` —
donc le natif gagne dès qu'il est installé, et l'export du conteneur reprend la
main dès qu'on le retire.
