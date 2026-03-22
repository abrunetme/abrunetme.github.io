---
title: "Activer l'autologin avec systemd et démarrer X automatiquement"
description: "Guide pour configurer systemd-automatiquement l'utilisateur au démarrage et lancer une session graphique via ~/.zlogin."
date: 2026-01-01
tags: [systemd, autologin, startx, zlogin, cinnamon]
---

Sur mon ordinateur, je suis le seul utilisateur et j'aimerai ne pas avoir à saisir mon login et mon identifiant. J'ai cherché une solution pour avoir un login automatique et lancer ma session graphique automatiquement.

<!--more-->

## 0. Désactiver les gestionnaires de session existants

Si vous utilisez par défaut un gestionnaire de session (GDM, SDDM, LightDM), vous devez les désactiver pour éviter les conflits avec l'autologin personnalisé.

### Identifier le gestionnaire actif

```bash
systemctl list-units --type=service | grep manager
```

### Désactiver le service

Remplacez le nom du service par le vôtre (ex: `gdm`, `sddm`, `lightdm`) :

```bash
sudo systemctl disable --now gdm
```

Pour vérifier qu'il est bien désactivé :

```bash
systemctl is-active gdm
# Doit retourner "inactive"
```

## 1. Configurer systemd pour l'autologin

L'objectif est de modifier le service `getty@tty1.service` pour qu'il ne demande pas de mot de passe à l'utilisateur `arnaud`.

### Créer le répertoire d'override

```bash
sudo mkdir -p /etc/systemd/system/getty@tty1.service.d/
```

### Créer le fichier de configuration

On crée un fichier d'override nommé `override.conf`. Ce fichier permet de surcharger les paramètres par défaut du service.

```bash
sudo vim /etc/systemd/system/getty@tty1.service.d/override.conf
```

```ini
[Service]
ExecStart=
ExecStart=-/usr/bin/agetty --autologin arnaud --noclear %I $TERM
```

*   `--autologin arnaud` : Lance automatiquement la session pour l'utilisateur `arnaud`.
*   `--noclear` : Empêche l'écran de se vider à chaque rechargement.
*   `%I` : Repère le terminal (par défaut `tty1` sur cette configuration).
*   `$TERM` : Utilise le terme de couleur approprié (ex: `xterm-256color`).

### Activer les modifications

Après avoir sauvegardé le fichier, rechargiez la configuration systemd :

```bash
sudo systemctl daemon-reload
```

Puis redémarrez le serveur pour tester :

```bash
sudo reboot
```

Après le reboot vous devez arriver sur le `tty1` sans à avoir à vous identifier. La prochaine étape est de lancer automatiquement le server X.

## 2. Lancer X automatiquement

Une fois l'autologin effectué sur le `TTY` (par `systemd`), le **shell** (ex: `zsh`, `bash`) est lancé. C'est dans ce contexte qu'il faut intervenir pour lancer la session graphique.

Pour ma part, j'utilise `zsh`. Le script suivant doit être placé dans le fichier `~/.zlogin`.
Si vous utilisez `bash`, le fichier est `~/.bash_profile`.

Le script vérifie trois conditions avant de lancer `startx` :
* **Pas d'interface graphique active** : `[[ -z $DISPLAY ]]`
* **L'utilisateur n'est pas root** : `(( $EUID != 0 ))`
* **Nous sommes sur le terminal tty1** : `[[ ${TTY} == '/dev/tty1' ]]`.

Puis le lancement du serveur X : `startx >~/.xsession-errors 2>&1 &` :
* Redirige les erreurs de démarrage de X vers `~/.xsession-errors` pour le débogage.
* Le `&` lance le processus en arrière-plan pour ne pas bloquer le login.

On fait une redirection des sorties dans `~/.xession-errors` mais le serveur X va aussi écrire ses logs dans `~/.local/share/xorg/Xorg.0.log`.

```bash
$ cat ~/.zlogin 
# Auto startx on TTY1
if [[ -z $DISPLAY ]] && (( $EUID != 0 )) {
    [[ ${TTY} == '/dev/tty1' ]] &&
        startx > ~/.xsession-errors 2>&1 &
}
```

## 3. Lancer l'environnement de bureau

Le serveur X va utiliser le fichier `~/.xinitrc` pour savoir quoi faire après son initialisation. C'est dans ce fichier qu'on indique le lancement de notre environnement de bureau (GNOME, KDE, Cinnamon ou autre).

Pour ma part, j'utilise Cinnamon :
```bash
$ cat ~/.xinitrc 
exec cinnamon-session
$
```

## Résumé de la configuration

Cette configuration permet à un utilisateur de se connecter directement à sa session graphique sans passer par l'écran de mot de passe de login, tout en conservant un contrôle manuel via `~/.zlogin` ou `~/.bash_profile` pour gérer l'initialisation de l'environnement graphique.

**Architecture du démarrage :**
1.  **Systemd** lance `agetty --autologin user --noclear`.
2.  **Login Shell** (`.zlogin`/`.bash_profile`) s'exécute.
    *   Vérifie : Pas de `$DISPLAY`, pas root, TTY1.
    *   Si OK : Lance `startx`.
3.  **Serveur X** démarre et écrit ses logs dans `~/.xsession-errors` et `~/.local/share/xorg/Xorg.0.log`.
4.  **`.xinitrc`** est exécuté par X.
5.  **Session graphique** (ex: Cinnamon) démarre et prend le contrôle.
6.  **Shell initial** est remplacé par la session.
