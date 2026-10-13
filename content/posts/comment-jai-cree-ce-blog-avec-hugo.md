---
title: "Comment j'ai créé ce blog avec Hugo"
date: 2026-10-13
draft: false
ShowToc: true
TocOpen: true
---

Il y a quelques mois, dans une démarche de dégafamisation, j'ai voulu monter mon propre Nextcloud pour remplacer Google Drive, accessible à distance via Cloudflare Tunnel, avec un nom de domaine loué pour l'occasion. L'idée me plaisait : reprendre la main sur mes données plutôt que de les laisser chez un GAFAM.

Sauf qu'en creusant, je me suis rendu compte qu'un NAS classique, une copie sur mon PC perso et un disque dur externe couvraient déjà largement mes besoins réels de stockage et de sauvegarde, sans la complexité de maintenir un Nextcloud à jour, sécurisé et accessible depuis l'extérieur. Le tunnel Cloudflare et l'accès distant, en pratique, ne me servaient presque jamais. J'ai donc arrêté le projet Nextcloud, gardé une architecture de stockage plus simple, et me suis retrouvé avec un nom de domaine loué pour 3 ans, mais plus vraiment d'utilité pour lui.

Plutôt que de le laisser à l'abandon, je me suis dit qu'il y avait mieux à en faire. Vu que je suis en admin sys & réseau, un blog technique pour documenter mes labs et ma montée en compétences avait beaucoup plus de sens qu'un nouveau projet self-hosted que j'allais sous-utiliser.

Cet article raconte comment je suis passé de ce nom de domaine sans emploi à un vrai blog en ligne, avec la table des matières ci-dessus pour naviguer directement vers ce qui t'intéresse.

## Pourquoi Hugo

Ce que je cherchais était simple : un site pour documenter mes labs et suivre ma montée en compétences, sans prise de tête à déployer ni à maintenir.

Hugo, un générateur de site statique, correspondait bien à ça. Le principe : on écrit le contenu en Markdown, et Hugo génère des fichiers HTML statiques, sans base de données ni CMS à patcher. Ça m'apportait ce que je cherchais :

- **Rien à maintenir niveau sécurité** : pas de service exposé à surveiller, contrairement à mon expérience avec Nextcloud
- **Hébergement gratuit** via GitHub Pages, sans serveur à gérer
- **Versionné avec Git**, cohérent avec mes habitudes d'admin sys
- **Rapide à prendre en main** : un thème existant (PaperMod), et un blog fonctionnel en moins d'une heure

L'alternative la plus proche aurait été Jekyll ou Astro, mais Hugo m'a semblé le meilleur compromis entre simplicité et rapidité pour ce que je voulais faire.

## Comment fonctionne Hugo

Le fonctionnement tient en quelques idées simples.

**Le contenu, c'est du Markdown.** Chaque page ou article est un fichier `.md` rangé dans le dossier `content/`. En haut du fichier, un petit en-tête (le "front matter") donne le titre, la date, et un champ `draft` qui décide si la page est publiée (`false`) ou reste un brouillon (`true`).

**Le thème gère l'apparence.** Il vit dans `themes/` et définit la mise en page de toutes les pages. Je n'écris jamais de HTML moi-même : Hugo applique le thème à mes fichiers Markdown. Si je veux modifier un détail du thème, comme le pied de page, je recrée le fichier concerné dans mon propre dossier `layouts/`, et il prend le dessus sans toucher au thème.

**Un seul fichier de configuration.** `hugo.yaml` regroupe le titre du site, le menu, les icônes sociales et les réglages du thème.

**Le build transforme tout en site statique.** La commande `hugo` assemble le contenu, le thème et la configuration, puis génère le site complet en HTML dans un dossier `public/`. C'est ce dossier qui est servi aux visiteurs. Je ne le versionne pas dans Git, car GitHub Actions le régénère à chaque push.

**Un serveur local pour prévisualiser.** `hugo server -D` lance le site sur `localhost:1313` et le recharge à chaque modification. Le `-D` affiche aussi les brouillons, ce qui permet de rédiger tranquillement avant de publier.

En résumé, j'écris en Markdown, je vérifie en local, je pousse sur Git, et le reste est automatique.

## La mise en place technique

L'installation de Hugo est en théorie très simple :

```bash
sudo apt install git hugo
```

Sauf que sur Linux Mint (et Ubuntu/Debian en général), la version de Hugo dans les dépôts officiels traîne souvent en retard. Dans mon cas, la version installée via `apt` était incompatible avec la dernière version du thème PaperMod, avec une erreur explicite au démarrage du serveur local :

```
ERROR => hugo v0.146.0 or greater is required for hugo-PaperMod to build
```

La solution : installer la dernière version directement depuis les releases officielles de Hugo sur GitHub, plutôt que de passer par `apt`.

Une fois Hugo à jour, la création du site elle-même est rapide :

```bash
hugo new site blogThibault --format yaml
cd blogThibault
git init
git submodule add --depth=1 https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod
```

Sur la structure du contenu, j'ai choisi de séparer deux types de pages :

- **Blog** (`content/posts/`) : des articles chronologiques et narratifs, comme celui-ci
- **Projets** (`content/projects/`) : des fiches statiques décrivant l'état actuel d'un lab, mises à jour au fil du temps plutôt que datées

Cette séparation évite de mélanger le récit ("comment j'ai fait X") avec la documentation de référence ("voici l'état actuel de X").

## Le déploiement

Pour héberger le site, direction **GitHub Pages**, couplé à une **GitHub Action** pour que tout se déploie automatiquement à chaque `git push`, sans avoir à lancer de build manuellement.

Le principe du workflow (`.github/workflows/hugo.yml`) : à chaque push sur `main`, une machine virtuelle temporaire chez GitHub installe Hugo, build le site, puis publie le résultat sur GitHub Pages.

Quelques pièges rencontrés en chemin, qui valent la peine d'être notés :

**Le nom du dépôt.** Pour que GitHub Pages publie automatiquement à la racine (`https://pseudo.github.io/`), le dépôt doit être nommé exactement `pseudo.github.io`. Un autre nom fonctionne aussi, mais l'URL finale inclut alors le nom du dépôt en plus.

**La source de déploiement.** Par défaut, GitHub Pages essaie de builder le site avec Jekyll, pas avec Hugo. Il faut explicitement changer la source vers "GitHub Actions" dans Settings → Pages, sinon le workflow tourne mais son résultat n'est jamais réellement publié.

**Les sous-modules.** J'ai ajouté le thème PaperMod comme sous-module Git : mon dépôt ne contient qu'un pointeur vers le dépôt du thème, pas ses fichiers. Ça permet de le mettre à jour facilement, mais GitHub Actions ne récupère par défaut que ce pointeur. Sans l'option `submodules: recursive` dans le workflow, le dossier du thème reste vide et le build échoue.

Une fois ces pièges évités, le workflow tourne en 30 à 60 secondes à chaque modification, et le site se met à jour tout seul.

## Brancher le nom de domaine

Une fois le site en ligne sur `pseudo.github.io`, il restait à y faire pointer mon nom de domaine, acheté chez OVH. Côté GitHub, il suffit de renseigner le domaine dans Settings → Pages. Côté DNS, j'ai ajouté deux types d'enregistrements :

- **Un CNAME pour `www`**, qui pointe vers `pseudo.github.io`. Un CNAME est un alias vers un autre nom de domaine, ce qui évite d'avoir à suivre les changements d'IP de GitHub.
- **Quatre enregistrements A pour la racine du domaine**, qui pointent vers les IP fournies par GitHub. On ne peut pas mettre de CNAME à la racine d'un domaine, car il doit cohabiter avec d'autres enregistrements comme ceux des mails.

Premier piège : OVH refusait mon CNAME avec l'erreur `CNAME and other data`. Un sous-domaine ne peut pas avoir un CNAME en même temps qu'un autre enregistrement, et il restait de vieilles entrées de mon ancien Nextcloud (un A et un TXT sur `www`). Il a fallu les supprimer avant de pouvoir continuer.

Deuxième piège, plus vicieux : OVH affichait un bandeau "cette zone DNS n'est pas autoritaire pour votre nom de domaine". Mon domaine utilisait encore les serveurs DNS de Cloudflare, restés là depuis le tunnel Nextcloud. Tant que c'était le cas, tout ce que je configurais côté OVH n'avait aucun effet. La solution : repasser sur les serveurs DNS d'OVH dans l'onglet "Serveurs DNS" du domaine, puis nettoyer Cloudflare (suppression du tunnel et de la zone devenus inutiles).

Après ça, le DNS se propage en quelques heures, GitHub valide le domaine, puis génère tout seul un certificat HTTPS.

## Les finitions

Une fois le site en ligne sur mon domaine, j'ai ajouté trois petites briques pour le rendre plus complet, en gardant la même philosophie : rien à héberger ni à maintenir de mon côté.

**Les commentaires avec Giscus.** Un site statique n'a pas de base de données, donc pas de commentaires natifs. Giscus s'appuie sur les Discussions GitHub du dépôt : les lecteurs se connectent avec leur compte GitHub, et leurs commentaires sont stockés dans les Discussions. C'est gratuit, sans publicité, sans tracking, et ça filtre naturellement le spam. Côté PaperMod, il suffit de créer un fichier `comments.html` contenant le script fourni par giscus.app, puis d'activer `comments: true` dans la config. Les pages comme Contact ou Qui suis-je n'ont pas besoin de commentaires, je les désactive avec `comments: false` dans leur en-tête.

**Les statistiques avec GoatCounter.** Pour savoir si quelqu'un lit le blog, j'ai choisi GoatCounter : gratuit pour un usage personnel, sans cookies, respectueux de la vie privée. Un simple script ajouté dans le `<head>` du site suffit (via le fichier `extend_head.html` prévu par PaperMod pour ça).

**Une adresse mail sans l'exposer.** Plutôt que d'afficher mon adresse personnelle sur la page Contact, j'ai créé une redirection chez OVH : `contact@mon-domaine.fr` renvoie vers ma vraie boîte. Si l'adresse se fait spammer un jour, je peux la couper ou la changer sans toucher à ma boîte principale.

## Pour finir

Au final, je suis parti d'un nom de domaine sans emploi et j'ai un blog complet : Hugo pour la génération, GitHub Pages pour l'hébergement, un déploiement automatique à chaque push, mon propre domaine en HTTPS, des commentaires et des statistiques. Rien à maintenir côté serveur, ce qui était exactement le but.

J'y ai surtout appris beaucoup sur Git, le DNS et les pipelines de déploiement, avec quelques galères qui m'auront fait comprendre les choses bien mieux qu'un tutoriel sans accroc. La suite, c'est de remplir ce blog : mes labs OPNsense, mes tests d'outils de cybersécurité, et tout ce que je découvrirai en route.

Si tu as déjà bataillé avec des problèmes similaires, ou si quelque chose n'est pas clair dans cet article, les commentaires juste en dessous sont là pour ça.
