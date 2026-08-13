# LinuxCNC sur Arch — deux recettes

Le paquet AUR `linuxcnc` ne compile pas sur Arch, et une fois compilé il lui
manque quatre dépendances plus un paquet qu'Arch a retiré. Voici les
correctifs, et pourquoi chacun existe.

Point de départ : un [visualiseur de parcours
G-code](https://github.com/atelierduverdier/visualiseur-gcode) qui a besoin de
`rs274`, l'interpréteur de LinuxCNC. En natif il est **douze fois plus rapide**
que dans un conteneur distrobox — 16 ms par appel contre 190. Depuis,
`qtdragon_hd`, l'interface de la machine, tourne aussi sur ce poste.

| | |
|---|---|
| `PKGBUILD.corrige` | la recette AUR avec les correctifs |
| `correctif-aur.diff` | les correctifs seuls, à réappliquer sur l'original |
| `python-qscintilla-qt5/` | un paquet à part, sans lequel aucune interface Qt de LinuxCNC ne démarre |

Ce dépôt ne contient **que des recettes** : ni paquets compilés, ni archives
sources. Chaque `PKGBUILD` porte l'URL et la somme de contrôle de ses sources.

```bash
makepkg -si                          # après avoir renommé PKGBUILD.corrige
cd python-qscintilla-qt5 && makepkg -si
```

Vérifier :

```bash
printf 'G21 G90\nG0 X10 Y10\nM2\n' > /tmp/t.ngc && : > /tmp/t.var && rs274 -g -v /tmp/t.var /tmp/t.ngc | head -3
```

---

## 1. La compilation échoue sur Python

Le `configure` de LinuxCNC 2.9.10 cherche Python en descendant depuis 3.13.
Arch est en **3.14** : il ne trouve pas le Python du système, descend d'un cran
et tombe sur un `python3.12` installé pour autre chose, où ni `gi` ni
`boost_python314` n'existent.

```
checking for python pango module... ModuleNotFoundError: No module named 'gi'
configure: error: Python pango module not found!
install with "sudo apt-get install python3-gi"
```

Le conseil `apt-get` égare : le paquet **est** installé, c'est l'interpréteur
qui n'est pas le bon. Une ligne devant `configure` suffit :

```diff
   ./autogen.sh
 
+  PYTHON=/usr/bin/python3 \
   ./configure \
     --prefix=/usr \
```

Un `NOM=valeur` posé devant une commande ne vaut que pour elle — rien n'est
modifié dans l'environnement.

## 2. Quatre dépendances manquantes

Chacune s'est signalée par une panne ; aucune ne se devine à la lecture de la
recette.

| paquet | sans lui |
|---|---|
| `python-pyqt5` | `qtvcp` ne démarre pas. Trompeur : le temps réel, HAL et `milltask` se chargent d'abord, seul l'affichage tombe. |
| `python-xlib` | `gmoccapy` refuse de démarrer. `qtvcp`, lui, n'émet qu'un avertissement — la dépendance est obligatoire **selon l'interface**. |
| `tclx` | `latency-histogram` ne démarre pas : il fait un `package require Tclx` **sec**, sans `catch`. |

`tclx` était en `makedepends`, ce qui est faux et coûte cher : il en ressortait
**orphelin**, or il épingle `tcl` à une version exacte. Le jour où CachyOS a
republié `tcl` avec son suffixe `znver4`, cet épinglage a bloqué **toute mise à
jour du système**.

```
:: l'installation de tcl (8.6.16-1.1) casse la dépendance « tcl=8.6.16-1 » requise par tclx
```

## 3. `python-qscintilla-qt5` — le paquet qui n'existe plus

Arch a suivi Qt6. Or l'éditeur de G-code de `qtvcp` est en PyQt5 :

```python
try:
    from PyQt5.Qsci import QsciScintilla, QsciLexerCustom, QsciLexerPython
except ImportError as e:
    LOG.critical("Can't import QsciScintilla ...")
    sys.exit(1)
```

`sys.exit(1)` : aucune dégradation, **tout écran `qtvcp` meurt avec lui** —
`qtdragon`, `qtdragon_hd`, `qtaxis`. Rien dans l'AUR ne comble le trou. La
bibliothèque C++ est pourtant là, `qscintilla-qt5` est dans `extra` ; seules
les liaisons Python manquent.

Deux pièges, documentés en tête du PKGBUILD :

* **l'archive PyPI embarque le C++.** `project.py` décide de son mode par
  `qsci_external_lib = not os.path.isdir('src')` — construite telle quelle,
  elle écraserait la `libqscintilla2_qt5.so` du système. D'où le retrait de
  `src` et `scintilla` dans `prepare()`.
* **le choix Qt5/Qt6 vient de `qmake`.** `/usr/bin/qmake` est en Qt5
  aujourd'hui, rien ne le garantit demain. D'où `--qmake /usr/bin/qmake-qt5`
  explicite : sans lui, une mise à jour produirait un jour des liaisons Qt6
  **sans un mot**.

Vérifier après construction :

```bash
ldd /usr/lib/python3*/site-packages/PyQt5/Qsci.abi3.so | grep -E "Qt5|qscintilla"
```

`libQt5*` et `libqscintilla2_qt5.so.15` : c'est juste. **À rejouer quand
`qscintilla-qt5` changera de version** — les liaisons suivent la bibliothèque
au numéro près.

## 4. « can't find package Linuxcnc » — ce n'est pas un correctif manquant

Au lancement de l'**interface** seulement ; `rs274` n'utilise pas Tcl.

Le paquet AUR pose déjà ce qu'il faut : `/etc/profile.d/linuxcnc.sh`, qui
exporte `TCLLIBPATH=/usr/lib/tcltk/linuxcnc`. Nécessaire parce que Tcl, sur
Arch, ne cherche que dans `/usr/lib/tcl8.6` et `/usr/lib`, et ne descend que
d'**un** niveau.

**Le piège est ailleurs** : `/etc/profile.d/` n'est lu que par les shells de
**connexion**. Un terminal ordinaire n'en ouvre pas, la variable reste vide, et
tout a l'air cassé alors que tout est bien installé.

```bash
echo "${TCLLIBPATH:-vide}"                              # vide ⇒ c'est le shell
[ -r /etc/profile.d/linuxcnc.sh ] && . /etc/profile.d/linuxcnc.sh   # dans ~/.bashrc
```

On **source** le fichier du paquet au lieu d'y recopier le chemin : s'il change
d'emplacement, on suit sans rien corriger.

---

## Reconstruire après une mise à jour de l'AUR

Un `git pull` écrasera le PKGBUILD corrigé et la compilation échouera de
nouveau, à l'identique.

```bash
git clone https://aur.archlinux.org/linuxcnc.git && cd linuxcnc
patch PKGBUILD < ../correctif-aur.diff
makepkg -si
```

Si le correctif ne s'applique plus, c'est que `prepare()` a changé : la
section 1 dit quoi remettre et où.

Le vrai remède serait que le mainteneur AUR intègre tout ça — et ce dépôt
deviendrait inutile.

## Revenir au conteneur

Le natif dépend de correctifs à réappliquer ; un conteneur, lui, est **gelé**
— insensible à une montée de Python ou de Boost, et il fige la version qui
tourne sur la machine.

```bash
distrobox create --name lcnc --image debian:bookworm
distrobox enter lcnc -- sudo apt install linuxcnc-uspace
distrobox enter lcnc -- distrobox-export --bin /usr/bin/rs274
```

**Prendre la version du dépôt LinuxCNC, pas celle de Debian** : bookworm livre
`2.9.0~pre1`, dont le `rs274` multiplie par 25,4 tous les décalages de la table
d'outils, `[TRAJ] LINEAR_UNITS = mm` ou non.

Rien à changer côté visualiseur : `rs274` est cherché dans le `PATH`, et
`/usr/bin` passe avant `~/.local/bin`.
