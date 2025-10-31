+++
title = "Gérer ses certificats Java avec KeyStore Explorer"
description = "Ajouter facilement une autorité de certification dans une JVM à l’aide de KeyStore Explorer."
date = 2025-10-30
draft = false
tags =  ["java", "keystore", "keystore explorer"]
+++

Récemment, j’ai dû faire face à un petit casse-tête que beaucoup d’équipes connaissent : la mise à jour des autorités de certification (CA) dans un environnement Java un peu ancien.
Au travail, nous sommes passés de Sectigo à Hurica pour la signature de nos certificats internes. Et comme notre application repose sur une vieille JVM, celle-ci ne reconnaissait pas la nouvelle autorité racine par défaut.

Plutôt que de plonger dans les commandes keytool à rallonge, j’ai redécouvert un outil graphique aussi pratique qu’efficace : **KeyStore Explorer**.

<!--more-->

## Pourquoi KeyStore Explorer ?

`keytool`, l’outil en ligne de commande fourni avec Java, est complet mais souvent… peu convivial.
KeyStore Explorer, lui, offre une interface graphique intuitive pour :
* Créer, ouvrir et modifier des fichiers de type JKS, PKCS12, JCEKS, etc.
* Importer ou exporter des certificats.
* Gérer les clés privées et les chaînes de certification.
* Signer des CSR (Certificate Signing Requests).

En clair, il s’agit d’une surcouche graphique à keytool, idéale pour manipuler des certificats sans erreur de syntaxe.

## Le problème rencontré

Après le passage à Hurica, notre JVM refusait certaines connexions HTTPS internes.
Dans les logs, on trouvait des erreurs du type :

```text
javax.net.ssl.SSLHandshakeException: 
sun.security.validator.ValidatorException: PKIX path building failed: 
unable to find valid certification path to requested target
```
Traduction : Java ne fait pas confiance à l’autorité de certification qui a signé notre certificat serveur.

## La solution : ajouter la racine Hurica dans le truststore

1. Télécharger le certificat racine Hurica
La première étape a été de récupérer le certificat racine de l’autorité Hurica.

Plutôt que de chercher un lien de téléchargement sur leur site, j’ai simplement exporté le certificat directement depuis le navigateur :
* J’ai ouvert une page HTTPS signée par Hurica (par exemple notre portail interne).
* Puis j’ai cliqué sur le cadenas dans la barre d’adresse → Afficher le certificat.
* Dans les détails, j’ai navigué jusqu’à l’autorité racine Hurica, puis choisi Exporter ou Enregistrer le certificat.

Cela m’a permis d’obtenir un fichier .crt que j’ai ensuite importé dans le truststore Java via KeyStore Explorer.

2. Ouvrir le truststore Java avec KeyStore Explorer

Le truststore par défaut de la JVM se trouve généralement ici : 
```text
$JAVA_HOME/lib/security/cacerts
```

Dans KeyStore Explorer :
* Fichier → Ouvrir → sélection du fichier cacerts
* Mot de passe par défaut : `changeit`

3. Importer le certificat racine
* Clic droit → Importer une entrée de confiance
* Sélectionner le certificat Hurica
* Donner un alias explicite (ex. `hurica-root-ca`)
* Sauvegarder le keystore

## Conclusion

KeyStore Explorer m’a une fois de plus sauvé du temps.
Au lieu de manipuler des commandes obscures, j’ai pu visualiser clairement la chaîne de confiance, importer la nouvelle CA et relancer le service en toute sérénité.

Si vous gérez encore des environnements Java anciens, je vous recommande chaudement cet outil : https://keystore-explorer.org

