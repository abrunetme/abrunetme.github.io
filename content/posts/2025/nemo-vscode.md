+++
title = "Ajouter \"Ouvrir dans VS Code\" au menu contextuel de Nemo"
date = 2025-01-07
draft = false
tags =  ["linux", "mint", "nemo", "vscode"]
+++

Si vous utilisez Visual Studio Code et le gestionnaire de fichiers Nemo (par défaut sur Linux Mint Cinnamon), il peut être très pratique d’ouvrir directement un dossier dans VS Code depuis le clic droit.

<!--more-->

Nemo permet facilement d’ajouter ce genre de raccourci via ses actions personnalisées. Elles sont stockées dans le dossier `~/.local/share/nemo/actions/`. 

Pour ajouter une action permettant d'ouvrir VS Code dans le menu contextuel, il faut créer un fichier dans ce dossier avec le contenu suivant :

```ini
[Nemo Action]
Name=Ouvrir dans VS Code
Comment=Ouvrir ce dossier avec Visual Studio Code
Exec=code %F
Icon-Name=com.visualstudio.code
Selection=Any
Extensions=dir;
Quote=double
Dependencies=code;
```

On peut bien sûr adapter pour gérer d'autres actions contextuelles.
