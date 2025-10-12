+++
title = "Utiliser une clé SSH spécifique pour un dépôt Git"
description = "Comment utiliser une clé SSH spécifique pour un dépôt Git lorsque vous gérez plusieurs comptes GitHub. Découvrez la commande GIT_SSH_COMMAND"
date = 2025-10-12
draft = false
tags =  ["git", "ssh", "github", "key"]
+++

Quand on gère plusieurs comptes GitHub, il peut arriver que la clé SSH par défaut utilisée par git ne corresponde pas au bon compte.
C’est par exemple le cas quand un dépôt est associé à une autre identité GitHub que celle configurée globalement sur votre machine.

<!--more-->

## Le problème

Par défaut, Git utilise la première clé SSH disponible (souvent ~/.ssh/id_rsa).
Si cette clé correspond à un autre compte GitHub, la commande git push échouera avec un message du type :

```text
ERROR: Permission to user/repo.git denied to other-user.
fatal: Impossible de lire le dépôt distant.

Veuillez vérifier que vous avez les droits d'accès
et que le dépôt existe.
```

## Solution

Vous pouvez forcer Git à utiliser une clé SSH précise en définissant la variable d’environnement GIT_SSH_COMMAND :

```sh
GIT_SSH_COMMAND="ssh -i ~/.ssh/autre-cle -o IdentitiesOnly=yes" git push
```

Explication :
* -i ~/.ssh/autre-cle → indique la clé privée à utiliser
* -o IdentitiesOnly=yes → empêche SSH d’essayer d’autres clés ou agents
* GIT_SSH_COMMAND="..." → fait en sorte que Git appelle SSH avec ces options

