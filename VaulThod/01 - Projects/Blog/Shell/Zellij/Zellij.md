# Zellij, le multiplexeur de terminal

![zellij](https://blog.stephane-robert.info/assets/images/zellij-ae0d645e41c0468f888bff08ea553623.gif)

**Je vais commencer par vous présenter Zellij, un outil qui a su se faire une place de choix dans mes outils préférés. Zellij est un multiplexeur de terminal, une catégorie d'outils conçue pour améliorer la gestion des terminaux dans les environnements de développement et d'administration système.**

**Zellij** se distingue par sa facilité d'utilisation et sa riche palette de fonctionnalités :

- Gestion des onglets
- Gestion des fenêtres flottantes
- Gestion de sessions
- Gestion de plugins
- Configuration via des fichiers écrit en KDL
- ...

## Installation de Zellij[​](https://blog.stephane-robert.info/docs/outils/systeme/zellij/#installation-de-zellij "Lien direct vers Installation de Zellij")

Passons maintenant à l'installation et la configuration initiale de **Zellij**. L'installation est un processus simple et direct, quelle que soit la plateforme que vous utilisez.

### Sur Linux[​](https://blog.stephane-robert.info/docs/outils/systeme/zellij/#sur-linux "Lien direct vers Sur Linux")

La plupart des distributions Linux peuvent installer **Zellij** via leur gestionnaire de paquets. Comme toujours, je préfère utiliser [asdf-vm](https://blog.stephane-robert.info/docs/outils/systeme/asdf-vm/) :

```
asdf plugin add zellij
asdf install zellij latest
asdf global zellij latest
```

### Sur macOS[​](https://blog.stephane-robert.info/docs/outils/systeme/zellij/#sur-macos "Lien direct vers Sur macOS")

Pour les utilisateurs de macOS, **Zellij** peut être installé via Homebrew, un gestionnaire de paquets populaire. La commande d'installation est la suivante :

```
brew install zellij
```

### Sur Windows[​](https://blog.stephane-robert.info/docs/outils/systeme/zellij/#sur-windows "Lien direct vers Sur Windows")

Les utilisateurs de Windows peuvent installer **Zellij** en utilisant le Sous-système Windows pour Linux (WSL). Une fois WSL configuré, suivez les instructions d'installation pour Linux.

## Utilisation de Zellij[​](https://blog.stephane-robert.info/docs/outils/systeme/zellij/#utilisation-de-zellij "Lien direct vers Utilisation de Zellij")

Nous allons voir en détail comment utiliser **Zellij**.

### Utilisation basique[​](https://blog.stephane-robert.info/docs/outils/systeme/zellij/#utilisation-basique "Lien direct vers Utilisation basique")

Pour lancer **Zellij**, il suffit d'utiliser la commande sans option :

```
zellij
```

Ce qui donnera affichage avec une seule section avec en bas la liste des principaux raccourcis que voici :

|Raccourcis|Actions|
|---|---|
|Alt+[n]|Ouvrir une fenêtre|
|Alt+Flèches|Se déplacer dans le volet|
|Alt+[+]|Augmenter la taille d'une fenêtre|
|Alt+[-]|Diminuer la taille d'une fenêtre|
|Alt+[=]|Attribuer une taille égale à toutes les fenêtre|
|Ctrl+[g]|Interface de verrouillage (utile lors de l'utilisation de raccourcis système)|
|Ctrl+[p] x|Pour fermer la fenêtre active|
|Ctrl+[p] c|Renommer la fenêtre active|
|Ctrl+[p] f|Pour ouvrir la fenêtre active en plein écran (idem pour taille initiale)|
|Ctrl+[p] w|Pour ouvrir une fenêtre flottante|
|Ctrl+[p] e|Pour rendre une fenêtre flottante (idem pour la remettre)|
|Ctrl+[q]|Quitter|

Il est possible d'ouvrir plusieurs onglets avec la séquence de touches suivantes : [ctrl]+[t] [n]. Pour passer d'un onglet à un autre, on peut utiliser la souris ou utiliser la séquence de touche [ctrl]+[t] [tab].

### Utilisation de layouts[​](https://blog.stephane-robert.info/docs/outils/systeme/zellij/#utilisation-de-layouts "Lien direct vers Utilisation de layouts")

Les **layouts** dans **Zellij** sont des configurations prédéfinies de l'interface du terminal qui organisent l'affichage des panneaux et des fenêtres : leur taille et parfois même les applications ou commandes qui s'exécutent dans chaque panneau.

Par exemple **Zellij** est livré avec un layout nommé `strider` qui propose un navigateur de fichier intégré.

```
zellij --layout strider
```

Ce qui donne :

![zellij strider](https://blog.stephane-robert.info/assets/images/zellij-strider-b29ef87a9c515c6410b43d354964773d.webp)

Nous verrons plus tard comment en créer.

### Gestion des sessions[​](https://blog.stephane-robert.info/docs/outils/systeme/zellij/#gestion-des-sessions "Lien direct vers Gestion des sessions")

Une session dans **Zellij** est un environnement de travail qui conserve l'état de votre interface de terminal, y compris les panneaux ouverts, leur disposition et les processus en cours d'exécution. Cela permet de sauvegarder votre contexte de travail et de le reprendre plus tard, même après une déconnexion ou un redémarrage.

Pour lister les sessions actives :

```
zellij ls

stellar-zebra [Created 0s ago] (EXITED - attach to resurrect)
verdant-lemur [Created 0s ago]
vitreous-jellyfish [Created 0s ago] (EXITED - attach to resurrect)
implacable-mountain [Created 0s ago]
sincere-peach [Created 0s ago]
```

On retrouve ici toutes les sessions qui ont été automatiquement créé par `Zellij`

Pour démarrer une session avec un nom prédéfini :

```
zellij -s <nom de la session>
```

Pour quitter en conservant une session en mémoire, il faut utiliser les raccourcis suivants : [ctrl]+[o] [d].

Pour reprendre une session, il faut l'attacher :

```
zellij a <nom de la session>
```

Pour détruire une session :

```
zellij d <nom de la session>
```

### Pilotage par la ligne de commande[​](https://blog.stephane-robert.info/docs/outils/systeme/zellij/#pilotage-par-la-ligne-de-commande "Lien direct vers Pilotage par la ligne de commande")

Il est possible de piloter les sessions de **Zellij** depuis la ligne de commande. Pour cela, on utilise la commande `action`.

```
zellij acction <sous-commande>
```

Voici une liste de quelques commandes :

- `edit <fichier>` : Ouvre un pane avec l'éditeur définit par défaut sur le fichier de votre choix.
- `dump-screen /tmp/screen-dump.txt` : copie le contenu d'une pane dans le fichier spécifié.
- `close-pane` : ferme le pane actif
- `close-tab` : ferme l'oglent actif
- ...

## Configuration de Zellij[​](https://blog.stephane-robert.info/docs/outils/systeme/zellij/#configuration-de-zellij "Lien direct vers Configuration de Zellij")

**Zellij** utilise le langage KDL pour sa configuration. Pour commencer, il est nécessaire de créer un répertoire de configuration et d'y générer un fichier de configuration par défaut en KDL :

```
mkdir ~/.config/zellij
zellij setup --dump-config > ~/.config/zellij/config.kdl
```

Ces commandes créent le dossier de configuration de **Zellij** dans votre répertoire personnel et y déposent un fichier de configuration par défaut.

En termes d'emplacement, **Zellij** cherche le fichier `config.kdl` dans plusieurs endroits, en suivant un ordre précis. Il commence par vérifier si un répertoire de configuration a été spécifié via le drapeau `--config-dir` ou la variable d'environnement `ZELLIJ_CONFIG_DIR`. Si aucun de ces éléments n'est spécifié, **Zellij** utilisera le chemin par défaut, qui varie selon le système d'exploitation. Sur Linux, le chemin par défaut est `/home/[username]/.config/zellij`, tandis que sur macOS, il s'agit de `/Users/[username]/Library/Application Support/org.Zellij-Contributors.Zellij`. Il existe également une option pour une configuration au niveau du système, située dans `/etc/zellij`.

### Changer de polices de caractères[​](https://blog.stephane-robert.info/docs/outils/systeme/zellij/#changer-de-polices-de-caract%C3%A8res "Lien direct vers Changer de polices de caractères")

Par défaut, zellij utilise des [polices nerd](https://www.nerdfonts.com/). Si de telles polices ne sont pas installées sur votre système, il est possible de configurer zellij pour qu'il utilise des polices plus simples. Dans le fichier de configuration, dé-commentez cette ligne :

```
simplified_ui true
```

Pour ceux qui veulent en installer :

```
git clone --depth 1 https://github.com/ryanoasis/nerd-fonts.git    # warning: takes a while

cd nerd-fonts/
./install.sh FiraCode
```

### Raccourcis Personnalisés[​](https://blog.stephane-robert.info/docs/outils/systeme/zellij/#raccourcis-personnalis%C3%A9s "Lien direct vers Raccourcis Personnalisés")

**Zellij** permet aux utilisateurs de définir leurs propres raccourcis clavier pour une navigation et une gestion plus efficaces des fenêtres et des panneaux. Pour personnaliser les raccourcis, vous devez modifier le fichier `config.kdl`. Voici un exemple de configuration pour un raccourci personnalisé :

```
keybinds {
    normal {
        bind "Alt c" { Copy; }
    }
}
```

Dans cet exemple, `Ctrl-c` est configuré activer la copie automatique sur sélection.

Pour désactiver un raccourci, on utilise cette syntaxe :

```
keybinds {
    unbind "Ctrl g" // unbind in all modes
}
```

### Changement de Thème[​](https://blog.stephane-robert.info/docs/outils/systeme/zellij/#changement-de-th%C3%A8me "Lien direct vers Changement de Thème")

**Zellij** offre également la possibilité de personnaliser l'apparence de votre terminal avec [différents thèmes](https://zellij.dev/documentation/theme-gallery). Pour changer le thème, vous devez modifier votre fichier `config.kdl` et y spécifier le thème de votre choix. Par exemple :

```
theme "dracula"
```

### Les plugins[​](https://blog.stephane-robert.info/docs/outils/systeme/zellij/#les-plugins "Lien direct vers Les plugins")

Sachez qu'il est possible d'écrire ses propres plugins en Webassembly/WASI. Il en existe quelques-uns [ici](https://zellij.dev/documentation/plugin-examples).

### Création de layouts[​](https://blog.stephane-robert.info/docs/outils/systeme/zellij/#cr%C3%A9ation-de-layouts "Lien direct vers Création de layouts")

Les layouts sont définis aussi dans des fichiers définis au format KDL. Pour créer un premier layout, on peut utiliser la commande suivante :

```
zellij setup --dump-layout default > ~/my-layout-file.kdl
```

Ce qui donne :

```
layout {
    pane size=1 borderless=true {
        plugin location="zellij:tab-bar"
    }
    pane
    pane size=2 borderless=true {
        plugin location="zellij:status-bar"
    }
}
```

Les layouts sont composés de :

- `pane`: les éléments de base de la mise en page, peuvent représenter des shells, des commandes, des plugins ou des conteneurs logiques pour d'autres panes.
- `tab` : représente un onglet de navigation Zellij et peut contenir des pane

Il est possible de lancer des commandes dans les pane :

Un exemple :

````
layout {
    pane // panes can be bare
    pane command="htop" // panes can have arguments on the same line
    pane {
        // panes can have arguments inside child-braces
        command "exa"
        cwd "/"
    }
    pane command="ls" { // or a mixture of same-line and child-braces arguments
        cwd "/"
    }
}

On peut divier les pane horizontalement ou verticalement :

```bash
layout {
    pane split_direction="vertical" {
        pane
        pane
    }
    pane {
        // value omitted, will be layed out horizontally
        pane size="80%"
        pane size="20%"
    }
}
````

Utilisation des onglets :

```
layout {
    tab // a tab with a single pane
    tab {
        // a tab with three horizontal panes
        pane
        pane
        pane
    }
    tab name="my third tab" split_direction="vertical" {
        // a tab with a name and two vertical panes
        pane
        pane
    }
}
```

## Conclusion[​](https://blog.stephane-robert.info/docs/outils/systeme/zellij/#conclusion "Lien direct vers Conclusion")

En conclusion, **Zellij** se révèle être un outil extrêmement puissant. En maîtrisant les sessions, les layouts et les diverses personnalisations, vous pouvez transformer votre terminal en un espace de travail dynamique et efficace. Que ce soit pour la gestion multitâche, l'organisation de workflows complexes, ou simplement pour augmenter votre productivité au quotidien, **Zellij** offre une gamme de fonctionnalités adaptées à vos besoins. En intégrant cet outil dans votre routine, vous vous dotez d'une capacité à gérer et à naviguer dans des environnements de développement et d'administration système avec une aisance et une efficacité accrues.

D'ailleurs, c'est pour cette raison que **Zellij** a rejoint [ma liste d'outils DevOps Indispensables](https://blog.stephane-robert.info/docs/outils/indispensables/) !

## Plus d'infos[​](https://blog.stephane-robert.info/docs/outils/systeme/zellij/#plus-dinfos "Lien direct vers Plus d'infos")

- **site Officiel** : [https://zellij.dev/](https://zellij.dev/)
- **Documentation Officielle** : [https://zellij.dev/documentation](https://zellij.dev/documentation)
- **Projet** : [https://github.com/zellij-org/zellij](https://github.com/zellij-org/zellij)