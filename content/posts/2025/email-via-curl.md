+++
title = "Envoyer un email simple avec cURL"
description = "Comment utiliser la commande curl pour envoyer un email"
date = 2025-12-09
draft = false
tags =  ["curl", "smtp", "email"]
+++

`curl` est principalement connu pour les requêtes HTTP, mais c'est un outil très polyvalent qui prend en charge de nombreux protocoles, y compris le SMTP (Simple Mail Transfer Protocol). Apprendre à l'utiliser pour envoyer des emails permet de se passer de clients mail comme `sendmail` ou `mutt` dans des environnements contraints (comme des conteneurs Docker ou des serveurs sans configuration locale).

<!--more-->

## Les prérequis SMTP

Pour envoyer un mail, vous avez besoin de connaître trois informations essentielles :
* L'adresse du MTA (Mail Transfer Agent) : L'hôte et le port de votre serveur SMTP (ex. : `smtp://mon-serveur.mon-domaine.fr:25`).
* L'adresse de l'expéditeur (MAIL FROM) : L'adresse affichée comme envoyeur (ex. : `robot@mon-domaine.fr`).
* L'adresse du destinataire (RCPT TO) : L'adresse qui recevra l'email.

## Le défi : Structurer le message

Le protocole SMTP nécessite que le message soit envoyé en deux parties distinctes :
* Les En-têtes (Headers) : From, To, Subject, Content-Type, suivis d'une ligne vide obligatoire.
* Le Corps (Body) : Le contenu réel de l'email.

En Bash, nous allons utiliser les commandes `echo` pour générer ces lignes, puis les "piper" (|) directement à `curl`.

## La commande magique

La clé pour que `curl` lise le contenu du message depuis le pipe est l'option --upload-file - (où - signifie lire depuis l'entrée standard).

Voici la structure complète de la commande que vous pouvez lancer dans un terminal :
```shell
# Configuration rapide
SMTP_SERVER="smtp://mon-serveur.mon-domaine.fr"
EXPEDITEUR="support@mon-domaine.fr"
DESTINATAIRE="utilisateur@mon-domaine.fr"
SUJET="Test d'envoi via CURL"
CORPS_MESSAGE="Ceci est le corps de mon email. Il gère les retours à la ligne si j'utilise \n dans un script."

# La commande complète en une seule ligne
{ 
    echo "From: $EXPEDITEUR"; 
    echo "To: $DESTINATAIRE"; 
    echo "Subject: $SUJET"; 
    echo "Content-Type: text/plain; charset=UTF-8";
    echo ""; 
    echo -e "$CORPS_MESSAGE"
} | curl --url "$SMTP_SERVER" \
         --mail-from "$EXPEDITEUR" \
         --mail-rcpt "$DESTINATAIRE" \
         --upload-file - \
         --verbose
```

**Explication des options curl** :
* `--url`: Spécifie l'hôte et le protocole (smtp://).
* `--mail-from`: Définit l'expéditeur SMTP réel (commande MAIL FROM:).
* `--mail-rcpt`: Définit le destinataire SMTP réel (commande RCPT TO:).
* `--upload-file -`: Indique à curl de lire le contenu du pipe (-) et de l'envoyer via la commande DATA du protocole SMTP.
* `--verbose`: (Optionnel pour le test) Affiche la conversation complète avec le serveur SMTP, indispensable pour le débogage.

## Astuce pour le "Silence"

Une fois que vous êtes sûr que l'envoi fonctionne, vous voudrez que votre script soit silencieux. Le serveur SMTP renvoie souvent un code de succès (ex. : `250 OK`), ce qui pollue la sortie standard.

Pour rendre l'envoi silencieux et masquer les messages de succès, modifiez la fin de la commande :

```shell
... | curl --url "$SMTP_SERVER" ... --upload-file - > /dev/null 2>&1
```
* `> /dev/null` : Redirige la sortie standard vers le néant.
* `2>&1`: Redirige les erreurs (sortie 2) vers la même destination que la sortie standard (1).

## Intégration dans les scripts Bash

Dans un script, il est préférable d'encapsuler la logique dans une fonction pour faciliter la gestion des erreurs.

La fonction `send_email`ci-dessous repose sur l'utilisation de variables globales que vous devez définir au début de votre script.

**Variables d'environnement:**

* `$SMTP_MTA`: L'adresse complète de votre serveur SMTP. Ex: `smtp://smtp.mon-domaine.fr`
* `$SMTP_FROM`: L'adresse utilisée dans l'en-tête From. Ex: `robot@mon-domaine.fr`
* `$ENV`: Définit l'environnement d'exécution. En TEST, le sujet est préfixé par `[TEST]`et l'email est envoyé à une adresse de substitution.
*  `$EMAIL_TEST`: L'adresse de substitution à utiliser en mode TEST.

**Fonction de journalisation:**

La fonction `log` sert uniquement à enregistrer l'envoi dans un fichier. Elle est définie par :
```shell
log() {
    echo "$(date '+%Y-%m-%d %H:%M:%S') - $1" >> "/var/log/email_script.log"
}
```

**Fonction d'envoi des emails:**

```shell
#
# Envoi d'un email
# $1: destinataire
# $2: sujet
# $3: contenu
#
send_email() {
    local to="$1"
    local subject="$2"
    local content="$3"

    # Logique optionnel pour ne pas envoyer les emails aux destinataires réels pendant les phases de tests
    if [[ "$ENV" == "TEST" ]]
    then
        subject="[TEST] $subject"
        to="$EMAIL_TEST"
    fi

    log "Envoi email '$subject' à '$to'"

    if ! {
        echo "From: $SMTP_FROM"
        echo "To: $to"
        echo "Subject: $subject"
        echo "Content-Type: text/plain; charset=UTF-8"
        echo ""
        echo -e "$content"
    } | curl --silent --mail-from "$SMTP_FROM" --mail-rcpt "$to" --url "$SMTP_MTA" --upload-file - > /dev/null 2>&1
    then
        log "Echec lors de l'envoi de l'email '$subject' à '$to'"
        return 1
    fi
}
```

## Conclusion

L'utilisation de `curl` pour l'envoi d'emails en SMTP est une technique puissante qui démontre la polyvalence de cet outil.

En maîtrisant la structure MIME des en-têtes et en utilisant l'option `--upload-file -`, on obtient une solution d'envoi d'emails légère, sans dépendance externe (hormis curl et le serveur SMTP), et parfaitement intégrable dans n'importe quel script Bash.


Cette méthode est particulièrement précieuse pour :
* Les systèmes embarqués ou les conteneurs Docker où l'installation de clients mail complets n'est pas souhaitée.
* Le diagnostic rapide des problèmes de connectivité SMTP.
* Les notifications simples et robustes dans vos automatisations quotidiennes.

