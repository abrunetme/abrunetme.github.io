+++
title = "Retrouver l’exécutable associé à une extension de fichier sous Windows"
date = 2025-02-12
draft = false
+++

Il arrive qu’on veuille savoir **quel programme est associé à une extension de fichier** sous Windows — par exemple, quel exécutable ouvre les fichiers `.jnlp`. Windows fournit deux commandes pratiques pour cela : `assoc` et `ftype`.

<!--more-->

`assoc` affiche l’association entre une **extension** et un **type de fichier** (FileType).
```cmd
assoc .jnlp
```
donne
```text
.jnlp=JNLPFile
```

`ftype` affiche la **commande complète** (donc l’exécutable) utilisée pour ouvrir ce type de fichier.
```cmd
ftype JNLPFile
```
donne
```text
JNLPFile="C:\Program Files\Java\jre1.8.0_281\bin\jp2launcher.exe" -securejws "%1"
```

On peut bien sûr combiner les deux commandes en un seul appel pour avoir directement l'exécutable :
```cmd
for /f "delims== tokens=2" %a in ('assoc .jnlp') do @ftype %a
```
donne
```text
JNLPFile="C:\Program Files\Java\jre1.8.0_281\bin\jp2launcher.exe" -securejws "%1"
```


