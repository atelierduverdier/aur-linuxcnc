# rs274 natif — paquet AUR `linuxcnc`, corrigé pour Arch

Ce dossier existe pour une seule raison : **le visualiseur de parcours G-code a
besoin de `rs274`**, l'interpréteur de LinuxCNC, et rien d'autre de LinuxCNC.

Il tournait dans un conteneur distrobox (Debian + `linuxcnc-uspace`), supprimé
le 12/08/2026. Le natif est **douze fois plus rapide** : 16 ms par appel contre
190, et la suite de tests du visualiseur passe de 14 s à 5 s. Chaque appel
payait un `distrobox-enter`, et le visualiseur en fait un par fichier chargé.

| | |
|---|---|
| `linuxcnc-2.9.10-1-x86_64.pkg.tar.zst` | le paquet compilé, prêt à installer |
| `PKGBUILD.corrige` | la recette AUR **avec** le correctif |
| `correctif-aur.diff` | le correctif seul, à réappliquer |

---

## Installer

```bash
sudo pacman -U ~/Projets/machine/aur-linuxcnc/linuxcnc-2.9.10-1-x86_64.pkg.tar.zst
```

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
