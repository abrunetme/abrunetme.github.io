---
title: "Mon premier dépôt APT avec apt-overlay"
description: "Pourquoi et comment j'ai monté mon propre dépôt APT : des binaires GitHub transformés en vrais paquets .deb natifs, servis statiquement via GitHub Pages avec une signature GPG."
date: 2026-09-19
draft: false
tags: ["apt", "debian", "ubuntu", "deb", "github-actions"]
summary: "Plutôt que d'installer mes outils avec des scripts curl | bash, j'ai créé mon propre dépôt APT. Découverte des .deb, de l'index Packages, de la signature GPG et de la CI qui régénère tout."
---

Jusqu'à présent, l'installation de mes outils « à la mode » passait toujours par un script douteux ou un gestionnaire lourd : `curl | bash`, Homebrew, Snap, Flatpak... Autant dire le contraire d'une solution propre et maintenable.

J'ai donc décidé de monter **mon propre dépôt APT**, servi gratuitement sur https://apt.abrunet.me/, où mes outils préférés s'installent comme de vrais paquets natifs :

<!--more-->

```bash
$ sudo apt install opencode sshm talosctl
```

Et surtout, comme tout paquet natif, ils se mettent à jour avec le reste du système :

```bash
$ sudo apt upgrade
```

C'était la première fois que je générais des paquets `.deb`. Voici le pourquoi et le comment.

## Pourquoi un dépôt APT personnel ?

J'utilise Linux Mint (dérivé d'Ubuntu/Debian). APT gère très bien les paquets officiels, mais les outils qui ne publient que sur GitHub échappent au gestionnaire de paquets : chacun impose sa propre méthode d'installation.

L'idée : **récupérer les binaires publiés sur GitHub et les empaqueter dans des `.deb` natifs**, servis statiquement, sans aucun démon local. Inspiré du concept d'overlay Gentoo, j'ai écrit un petit moteur nommé [apt-overlay](https://github.com/abrunetme/apt-overlay) qui automatise tout ça, en totalité dans GitHub Actions et servi via GitHub Pages.

## Le principe : une recette déclarative par paquet

Chaque application est décrite par un simple fichier `.conf` dans `packages/`. Exemple réel pour `opencode` :

```bash
# packages/opencode.conf
REPO_PATH="anomalyco/opencode"
ASSET_NAME="opencode-linux-x64.tar.gz"
DESCRIPTION="The open source coding agent."
```

Trois variables suffisent :
* `REPO_PATH` : le dépôt GitHub (`Owner/Repository`) d'où vient le binaire,
* `ASSET_NAME` : le nom exact de l'asset de la dernière release (`.tar.gz` ou binaire seul),
* `DESCRIPTION` : la description embarquée dans les métadonnées du paquet.

Le nom du paquet correspond au nom du fichier sans l'extension (avec possibilité de le surcharger via `PKG_NAME`). Il doit respecter les règles de nommage Debian : `[a-z0-9][a-z0-9+.-]*`.

## Comment est fabriqué un `.deb`

C'est la partie que je découvrais. Un paquet Debian est, en substance, une archive contenant des métadonnées et les fichiers à installer.

Pour chaque paquet, le script `build-repo.sh` prépare cette arborescence :

```
build/<paquet>_<version>_amd64/
├── DEBIAN/control        # les métadonnées du paquet
└── usr/local/bin/<paquet> # le binaire
```

Le fichier `control` est minimaliste mais essentiel — c'est lui qui décrit le paquet à APT :

```text
Package: opencode
Version: 0.1.23
Architecture: amd64
Maintainer: APT Overlay System
Description: The open source coding agent.
```

Ensuite, quelques étapes :

1. **Récupération de la version** : le script appelle l'API GitHub pour obtenir le dernier tag de release, puis le transforme en version Debian (suppression du `v` préfixe, et obligation de démarrer par un chiffre).
2. **Récupération de l'asset** : soit une extraction du `.tar.gz` (avec détection intelligente du binaire — le plus gros fichier exécutable si le nom n'est pas trouvé), soit un simple binaire autonome.
3. **Assemblage** : on copie le binaire dans `usr/local/bin/`, on le rend exécutable, puis on génère l'archive avec `dpkg-deb` :

```bash
chmod +x "${PKG_BUILD_DIR}/usr/local/bin/${PKG_NAME}"
dpkg-deb --build "${PKG_BUILD_DIR}" "${REPO_DIR}/"
```

Le résultat est un `.deb` natif que je peux installer et désinstaller comme n'importe quel autre paquet Debian. Le script ne garde que la dernière version de chaque paquet et refuse de reconstruire une version déjà présente.

## L'index et la signature : ce que voit APT

Un dépôt APT, ce n'est pas juste des `.deb` dans un dossier. APT doit connaître la liste des paquets disponibles et pouvoir vérifier qu'ils n'ont pas été falsifiés.

* **`Packages`** : l'index généré par `apt-ftparchive packages`, qui liste chaque paquet, sa version, son fichier et ses checksums. Une version compressée `Packages.gz` est aussi fournie.
* **`Release`** : les métadonnées du dépôt générées par `apt-ftparchive release`.
* **`InRelease`** et **`Release.gpg`** : la signature GPG du dépôt.

Les `.deb` ne sont pas signés individuellement : APT authentifie l'ensemble via les checksums contenus dans `Packages`, lui-même couvert par la signature `Release`. Une seule clé GPG est donc nécessaire pour tout signer :

```bash
gpg --batch --yes --default-key "$KEY_ID" --clearsign -o InRelease Release
gpg --batch --yes --default-key "$KEY_ID" -abs -o Release.gpg Release
```

La clé privée est injectée dans l'environnement de la CI via le secret `GPG_PRIVATE_KEY`. La clé publique, elle, est publiée à la racine du dépôt sous le nom `Release.key`.

## L'intégration continue

Le workflow `update-repo.yml` déclenche la reconstruction :
* à chaque `push` sur `main`,
* automatiquement chaque nuit (`cron`), pour suivre les nouvelles releases GitHub,
* manuellement via `workflow_dispatch`.

Point astucieux : le build part de l'**état précédent du dépôt** restauré depuis GitHub Pages (il re-télécharge les `.deb` déjà publiés). Combiné à l'idempotence du script, cela lui permet de ne reconstruire que les paquets dont la version a changé, sans jamais perdre les versions déjà en ligne. Puis `upload-pages-artifact` et `deploy-pages` publient le dossier `public/`, servi en HTTPS sur apt.abrunet.me.

## Utilisation côté client

Côté machine cliente, il suffit d'ajouter le dépôt puis d'installer :

```bash
curl -fsSL https://apt.abrunet.me/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/abrunet-overlay.gpg

echo "deb [signed-by=/etc/apt/keyrings/abrunet-overlay.gpg] https://apt.abrunet.me dist/" | sudo tee /etc/apt/sources.list.d/apt-overlay.list

sudo apt update
```

Puis les outils s'installent comme n'importe quel logiciel natif :

```bash
sudo apt install opencode sshm talosctl
```

## Ajouter un paquet

Pour ajouter un outil au dépôt, il suffit de créer un fichier `packages/<nom>.conf` avec le bon `REPO_PATH` et `ASSET_NAME`, de pousser, et la CI se charge du reste : téléchargement de la dernière release, génération du `.deb`, indexation, signature et publication.

## Conclusion

Disposer de mon propre dépôt APT change vraiment le quotidien : plus de scripts d'installation jetables, plus de binaires à recopier à la main, mais un `apt install` / `apt upgrade` uniforme pour mes outils préférés. Côté technique, cette petite aventure m'a permis de comprendre en profondeur l'anatomie d'un paquet Debian : fichier `control`, `dpkg-deb`, index `Packages` et signature `Release`.

Le projet est open-source : https://github.com/abrunetme/apt-overlay — vous pouvez forker, ajouter vos propres recettes et servir le dépôt sur votre domaine ou un sous-chemin GitHub Pages.