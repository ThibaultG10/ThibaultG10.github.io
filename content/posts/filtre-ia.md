---
title: "Filtre IA : un outil local pour ne plus envoyer de données sensibles aux IA"
date: 2026-11-08T12:00:00+01:00
draft: false
ShowToc: true
TocOpen: true
description: "Pourquoi et comment j'ai construit Filtre IA, un anonymiseur 100 % local qui masque les données sensibles d'un prompt avant de l'envoyer à ChatGPT, Claude ou Gemini."
tags: ["IA", "confidentialité", "open source", "RGPD"]
---

> **À lire avant d'aller plus loin.** Je ne suis pas développeur de formation. Cet outil est un projet que j'ai construit en m'appuyant sur une IA, et en le testant au maximum. Il peut donc contenir des erreurs, des oublis ou des cas que je n'ai pas prévus. **Il réduit le risque, il ne le supprime pas** : relisez toujours le texte avant de l'envoyer, et ne le considérez pas comme une garantie de conformité. Le code est ouvert précisément pour que des personnes plus expérimentées puissent le relire, le critiquer et l'améliorer.

## D'où ça part

L'utilisation de l'IA en entreprise est devenue inévitable. On colle un mail à reformuler, un extrait de contrat à résumer, un message d'erreur à comprendre, et ça fait gagner un temps réel.

Le problème, c'est ce qu'on envoie sans y penser : un numéro de téléphone, une adresse IP de serveur, un IBAN, le nom d'un client, une clé d'accès oubliée dans un extrait de code. Une fois le texte parti chez un service tiers, on ne le reprend plus.

Les interdire complètement ne marche pas : les gens les utilisent quand même, en dehors de tout cadre. Il me semblait plus réaliste de leur donner un moyen simple de **garder l'avantage de l'IA sans lui envoyer les données sensibles**.

## L'idée

Le principe tient en une phrase : *avant d'envoyer un texte à une IA, on masque ce qui ne doit pas partir, puis on remet les vraies valeurs dans la réponse.*

Concrètement :

1. **Vous collez votre texte** dans l'outil.
2. **Il repère les données sensibles** : téléphones, e-mails, IBAN, cartes bancaires, adresses IP, clés d'API, numéros de sécurité sociale… ainsi que tous les termes que vous avez listés vous-même dans un annuaire (noms de clients, de projets, de serveurs, de personnes).
3. **Vous vérifiez** : chaque élément repéré est surligné, vous pouvez en retirer un qui n'en est pas un, ou en ajouter un qui a été oublié.
4. **Il remplace chaque valeur par un jeton** : « Appelez Dupont SA au 06 12 34 56 78 » devient « Appelez [CLIENT_1] au [TEL_1] ».
5. **Vous envoyez le texte nettoyé** à l'IA.
6. **Vous collez sa réponse** dans l'outil, qui remet les vraies valeurs à la place des jetons.

L'IA travaille sur un texte qui garde son sens, sans jamais avoir vu les données réelles.

## Tout se passe dans votre navigateur

C'était pour moi la condition de départ. Un outil qui envoie votre texte sur un serveur pour l'anonymiser ne résout rien : il déplace le problème.

Filtre IA fonctionne donc **entièrement en local** :

- aucune donnée n'est envoyée sur Internet, il n'y a ni serveur, ni compte, ni statistiques ;
- la page interdit elle-même toute connexion réseau (une politique de sécurité du navigateur, vérifiable) ;
- l'outil tient dans un seul fichier HTML que l'on peut ouvrir hors ligne.

Si vous voulez le vérifier plutôt que de me croire, c'est possible : le code est public.

## Ce qu'il sait faire aujourd'hui

- Détecter des formats français et internationaux (téléphones, IBAN, SIRET, numéros de sécurité sociale français, américains et britanniques, cartes, IP, clés d'API, URL internes), avec des contrôles de validité pour limiter les fausses alertes.
- Gérer un annuaire de termes interdits, insensible à la casse et aux accents, importable et exportable en JSON ou CSV.
- Restaurer les vraies valeurs dans la réponse de l'IA, même si elle a écrit un jeton en minuscules ou avec des espaces.
- Chiffrer la session avec une phrase de passe si vous voulez la retrouver plus tard.
- S'utiliser en français ou en anglais.

## Ce qu'il ne fait pas

Je préfère être clair sur les limites :

- **Il ne détecte pas les noms propres qu'il ne connaît pas.** Un nom absent de votre annuaire ne sera pas masqué.
- Un contexte peut permettre d'identifier quelqu'un même quand chaque donnée a été masquée.
- L'IA peut reformuler ou oublier un jeton : la restauration n'est pas garantie à 100 %.
- Il ne traite ni les images, ni les PDF, ni les pièces jointes.
- Il peut se tromper, dans les deux sens : masquer quelque chose d'anodin, ou laisser passer une donnée écrite de façon inhabituelle.

C'est pour cela que l'étape de vérification existe : l'outil propose, **c'est vous qui décidez**.

## Comment il a été fabriqué

Je ne code pas au quotidien. J'ai défini le besoin et les règles, puis j'ai travaillé avec une IA pour écrire le code, les tests et la documentation, en insistant sur les vérifications (plus d'un millier de tests automatiques, des contrôles de sécurité sur le comportement du navigateur). Cela ne remplace pas la relecture d'un développeur ou d'un spécialiste de la sécurité, et je ne prétends pas que ce soit le cas.

Si vous repérez une erreur, un format qui n'est pas détecté ou une idée d'amélioration, **dites-le-moi** : c'est exactement pour cela que le projet est ouvert.

## Essayer et contribuer

- Le code est disponible sous licence MIT : [github.com/ThibaultG10/AI-Filter](https://github.com/ThibaultG10/AI-Filter)
- Un problème, une suggestion ? Ouvrez une « issue » sur GitHub, ou écrivez-moi : [contact@thibault-guillonneau.fr](mailto:contact@thibault-guillonneau.fr)

Si vous testez l'outil sur un vrai cas d'usage, votre retour m'intéresse beaucoup, y compris et surtout les critiques.

*Merci de ne jamais utiliser cet outil comme seule protection pour des données réellement critiques.*
