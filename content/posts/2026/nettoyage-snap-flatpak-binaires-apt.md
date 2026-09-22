---
title: "Un seul gestionnaire de paquets : adieu Snap, Flatpak et binaires à la main"
description: "Comment j'ai recentré mon système Linux sur un unique gestionnaire de paquets, APT, en remplaçant mes installations Snap, Flatpak et manuelles par des paquets natifs."
date: 2026-09-22
draft: false
tags: ["apt", "debian", "linux-mint", "flatpak", "snap"]
summary: "Fini les gestionnaires de paquets en parallèle. J'ai nettoyé mon système des Snap, Flatpak et binaires manuels pour revenir à un Linux 100% APT : inventaire des paquets hors dépôt, remplacements de Rocket.Chat, DBeaver et Remmina, et désactivation des services liés."
---

Il fut un temps où mes outils s'installaient chacun à sa façon : un Snap par-ci, un Flatpak par-là, un script `curl | bash` quand il n'y avait rien de mieux. Résultat : des mises à jour dispersées entre trois gestionnaires, des versions différentes selon les applications et un système dont je ne savais plus exactement ce qu'il contenait.

Récemment, j'ai tout nettoyé. Objectif : **un seul gestionnaire de paquets, APT**, avec un `apt upgrade` qui met à jour l'ensemble du système.

<!--more-->

## Faire l'état des lieux

Avant de nettoyer, il faut savoir ce qui traîne. Deux commandes m'ont permis de repérer les paquets installés hors dépôt et les binaires manuels.

**1. Les paquets installés mais sans provenance de dépôt**

Un paquet installé qu'aucun dépôt ne référence est presque toujours un `.deb` téléchargé à la main — qui ne se mettra jamais à jour avec le reste du système :

```bash
dpkg-query -W -f='${Package}\n' | while read -r pkg; do
  if ! apt-cache policy "$pkg" 2>/dev/null | grep -E -q '500|1000|100 '; then
    echo "Hors dépôt : $pkg"
  fi
done
```

**2. Les exécutables manuels dans `/usr/local/bin`**

Cette commande sépare ce qui appartient à un paquet APT de ce qui a été recopié à la main :

```bash
for f in /usr/local/bin/*; do
  [ -e "$f" ] || continue
  pkg=$(dpkg -S "$f" 2>/dev/null | cut -d: -f1)
  if [ -n "$pkg" ]; then
    echo "APT ($pkg) : $f"
  else
    echo "MANUEL / HORS-APT : $f"
  fi
done
```

L'inventaire établi, place aux remplacements.

## Rocket.Chat : du Snap au dépôt APT personnel

Rocket.Chat était le seul paquet Snap de mon système — un second gestionnaire de paquets pour une seule application. Problème supplémentaire : **Rocket.Chat ne publie qu'un `.deb` par version, sans mise à jour automatique**.

J'en ai profité pour l'intégrer à mon dépôt APT personnel [apt-overlay](https://apt.abrunet.me/), présenté dans l'article [Mon premier dépôt APT avec apt-overlay]({{< ref "depot-apt-personnel" >}}). Le `.deb` publié sur GitHub est désormais reconstruit chaque nuit et indexé : les mises à jour arrivent par un simple `apt upgrade`, comme pour n'importe quel paquet natif.

**Inventaire et désinstallation Snap** :

```bash
snap list                                        # lister les applications snap installées
sudo snap services rocketchat-server             # voir les services de l'application
sudo snap stop rocketchat-server                 # les arrêter
sudo snap remove rocketchat-server --purge       # désinstaller + purger les données
```

**Désactivation des services système liés à snapd**

Contrairement à Flatpak, Snap installe un vrai démon (`snapd`) ainsi que plusieurs unités systemd qui tournent en arrière-plan. Une fois plus aucun Snap utilisé, autant les désactiver :

```bash
systemctl list-units --all | grep -E 'snap|snapd'            # lister les unités liées à Snap
systemctl status snapd snapd.socket snapd.seeded.service snapd.apparmor.service
sudo systemctl stop snapd snapd.socket snapd.seeded.service snapd.apparmor.service
sudo systemctl disable snapd snapd.socket snapd.seeded.service snapd.apparmor.service
sudo systemctl mask snapd                                     # empêcher tout redémarrage
```

**Nettoyage final** : suppression du paquet `snapd` et des fichiers restants :

```bash
sudo apt remove --purge snapd
sudo rm -rf /var/snap /snap /var/lib/snapd
```

### Les profils AppArmor résiduels

Snap s'appuie sur AppArmor pour confiner ses applications et y dépose des profils dans `/etc/apparmor.d/`. Même après la purge du démon, il peut rester des fichiers `snap.*` orphelins. Vérifier :

```bash
ls /etc/apparmor.d/ | grep -i snap
```

Si des profils traînent, les supprimer puis recharger AppArmor :

```bash
sudo rm -f /etc/apparmor.d/snap.* /etc/apparmor.d/*snap*
sudo systemctl reload apparmor
```

## DBeaver : du Flatpak au dépôt officiel

DBeaver me servait en Flatpak, mais l'éditeur fournit aussi un **dépôt APT officiel**. Récupérer la clé GPG, ajouter la source, installer comme un paquet natif — et voilà les mises à jour gérées par le système :

```bash
sudo apt install dbeaver-ce
```

## Remmina : retour au dépôt de la distribution

Remmina, le client RDP/VNC, était lui aussi installé en Flatpak. La version fournie par **le dépôt de la distribution** me suffit largement, sans le surcoût d'un sandbox complet :

```bash
sudo apt install remmina
```

## Le grand ménage Flatpak

Une fois les remplacements effectués, il reste à purger les applications Flatpak. Chaque application embarque en plus des *runtimes* partagés, parfois volumineux :

```bash
flatpak list                                    # lister les applications installées
flatpak uninstall --user --delete-data <App_ID> # désinstaller + effacer ses données
flatpak uninstall --unused                      # retirer les runtimes devenus inutiles
```

Le `--delete-data` est important : sans lui, les données de l'application restent dans `~/.var/app`. Flatpak étant un gestionnaire *par utilisateur*, il ne laisse aucun service système à désactiver : pas de démon, rien qui tourne en continu une fois tout désinstallé.

**Le ménage des données résiduelles**

Même après désinstallation complète, flatpak laisse des traces qui peuvent peser lourd : les données applicatives, le cache et surtout le dépôt système `/var/lib/flatpak`. Sur ma machine, ils représentaient encore plus de 100 Mo. Dernière passe de nettoyage :

```bash
rm -rf ~/.var/app ~/.local/share/flatpak ~/.cache/flatpak # résidus utilisateur
sudo rm -rf /var/lib/flatpak                              # dépôt système (112 Mo chez moi)
```

Pas d'inquiétude : ces répertoires sont de simples données, pas des binaires. Flatpak les recrée automatiquement à la prochaine installation — il faudra simplement re-télécharger la clé de confiance de flathub et le cache appstream.

## Conclusion

Mon système ne connaît plus qu'un seul gestionnaire de paquets. Les bénéfices se font sentir immédiatement :

* **Mises à jour uniformes** : un seul `apt upgrade` suffit, tout est cohérent ;
* **Moins d'espace disque** : fini les runtimes dupliqués et les snap volumineux ;
* **Un système auditable** : tout est répertorié par `dpkg`, rien ne traîne hors de contrôle ;
* **Moins de surface d'attaque** : plus de daemons cachés (snapd) qui tournent pour rien.

Cette démarche prolonge naturellement mon dépôt APT personnel [apt-overlay](https://apt.abrunet.me/) : désormais, chaque outil s'installe comme un vrai paquet natif et se met à jour avec le reste du système.