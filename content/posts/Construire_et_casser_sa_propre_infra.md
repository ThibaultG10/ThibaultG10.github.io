+++
title = Construire et casser sa propre infrastructure : IaC, Podman et audit de sécurité'
date = 2026-11-01T21:17:37+01:00
draft = false
+++
# Construire et casser sa propre infrastructure : IaC, Podman et audit de sécurité

Il y a deux semaines, je ne connaissais ni Terraform ni les subtilités de Podman. Dans le cadre d'un TP de cybersécurité, j'ai dû construire une infrastructure conteneurisée de A à Z — réseau segmenté, passerelle, backends, base de données — puis la mettre à l'épreuve avec une machine d'attaque. Pas un exercice théorique : une vraie construction, suivie d'une vraie tentative de la faire plier.

Cet article retrace ce parcours : les choix techniques, les erreurs qui m'ont forcé à comprendre plutôt qu'à copier des commandes, et la méthodologie qui m'a permis, à la fin, de prouver — pas seulement d'affirmer — que l'infrastructure tenait la route.

## Le décor : deux VM, un réseau isolé

Le lab repose sur deux machines virtuelles VirtualBox : une Ubuntu Desktop qui héberge l'infrastructure, une Kali Linux qui joue le rôle de l'attaquant. Entre les deux, un réseau interne VirtualBox dédié — aucune route vers l'extérieur, aucun accès depuis l'hôte physique. Juste un bac à sable fermé où on peut casser des choses sans risque.

Côté choix techniques, le premier a été de préférer **Podman** à Docker pour héberger les conteneurs. L'argument : Podman fonctionne sans démon central tournant en root, contrairement à Docker — un détail qui compte dès qu'on pense sécurité par défaut. Le compromis, c'est que l'écosystème Terraform est historiquement pensé pour Docker. La solution tient en une ligne : Podman expose une API compatible avec celle de Docker via un socket Unix, et le provider Terraform `kreuzwerker/docker` n'y voit que du feu.

```hcl
provider "docker" {
  host = "unix:///run/user/1000/podman/podman.sock"
}
```

Terraform croit parler à Docker. C'est Podman qui répond.

## Terraform : trois concepts que les erreurs m'ont appris mieux que la doc

Terraform fonctionne sur un principe simple à énoncer, plus subtil à intérioriser : on décrit un **état voulu**, et l'outil calcule lui-même les actions nécessaires pour l'atteindre. Pas de script qui enchaîne des commandes — un fichier qui dit « voici ce que je veux », et un cycle `init` → `plan` → `apply` qui s'occupe du reste.

Le concept qui m'a le plus servi, c'est l'**idempotence** : relancer `apply` sans rien changer ne doit rien faire. `0 added, 0 changed, 0 destroyed`. C'est ce qui permet de rejouer un déploiement sans craindre de tout casser — exactement ce qu'on attend d'un pipeline automatisé.

Mais la vraie leçon est venue d'un problème que je n'avais pas anticipé : le **drift**. À chaque `plan`, Terraform recréait mes conteneurs sans que j'aie touché au code. La cause : le provider `kreuzwerker/docker` est pensé pour un vrai démon Docker, et Podman ne représente pas certains attributs exactement de la même façon. Là où mon code disait `cap_drop = ["ALL"]`, Podman répondait avec la liste détaillée des capacités réellement bloquées. Terraform comparait un mot-clé à une liste — jamais égal, donc toujours une recréation.

La solution n'était pas de corriger mon code, qui était juste, mais de dire à Terraform d'arrêter de comparer ces attributs précis :

```hcl
lifecycle {
  ignore_changes = [ulimit, pid_mode, security_opts, capabilities]
}
```

Ce drift n'était pas une erreur de ma part. C'était la limite réelle d'un outil « compatible » qui ne l'est jamais à 100 %. Une leçon qui vaut largement au-delà de ce TP : la compatibilité annoncée entre deux outils est un point de départ pour le debug, pas une garantie.

## Une architecture pensée pour résister, pas juste pour fonctionner

L'infrastructure finale tient sur un principe simple : un seul point d'entrée, tout le reste caché. Deux réseaux Podman distincts — `lab_front`, accessible depuis l'extérieur, et `lab_back`, marqué `internal = true`, totalement cloisonné, sans la moindre sortie possible même vers la machine hôte. Une passerelle nginx, seule ressource avec un pied sur les deux réseaux, fait le pont. Derrière elle, des backends générés dynamiquement avec `for_each` — passer de deux à quatre instances n'a recréé que les deux nouvelles, sans toucher aux existantes, grâce à des clés nommées stables plutôt qu'un simple compteur. Et une base PostgreSQL, invisible de bout en bout, sans le moindre port publié.

Le durcissement des conteneurs a donné lieu à la leçon la plus concrète du projet. Le principe de moindre privilège dit : retirer toutes les capacités Linux, n'en garder que le strict nécessaire.

```hcl
capabilities {
  drop = ["ALL"]
}
```

Appliqué tel quel, ce bloc a fait planter tous mes conteneurs nginx au démarrage — `container exited immediately`, sans autre explication. En creusant, la cause tenait en une ligne : nginx démarre en root, doit changer le propriétaire de certains fichiers, puis abandonne ses privilèges vers un utilisateur non-root pour tourner. Sans la capacité `CHOWN`, il ne peut tout simplement pas franchir cette étape.

```hcl
capabilities {
  drop = ["ALL"]
  add  = ["NET_BIND_SERVICE", "CHOWN", "SETUID", "SETGID"]
}
```

Le moindre privilège, ce n'est pas tout bloquer. C'est une liste blanche minimale et fonctionnelle — pas une interdiction totale qui casse le service qu'on essaie justement de protéger.

## Durcir le système hôte : de 61 à 81

Sécuriser les conteneurs ne sert à rien si la machine qui les héberge reste une passoire. C'est là qu'intervient **Lynis**, un outil d'audit système qui attribue un score de durcissement sur 100 et liste, pour chaque point perdu, une suggestion concrète.

Premier passage : 61/100. Le diagnostic était clair — aucun pare-feu actif, des paquets de sécurité de base absents (`fail2ban`, `auditd`, `rkhunter`), une configuration SSH permissive, des paramètres noyau laissés par défaut.

Quelques heures de travail méthodique plus tard :

- Un pare-feu `ufw` configuré en liste blanche stricte (tout bloqué en entrée, sauf SSH et le port de la passerelle)
- SSH durci : forwarding désactivé, tentatives limitées, logs en mode verbeux
- Une dizaine de paramètres noyau alignés sur le profil de sécurité recommandé
- Des outils de détection installés et activés (`fail2ban`, `rkhunter`, `auditd`)

Score final : 81/100. Ce qui reste non corrigé est documenté et assumé — repartitionner le disque pour isoler `/home`, `/tmp` et `/var` n'avait pas de sens sur une VM de lab, et deux paramètres noyau auraient cassé Podman. Un score de sécurité n'est pas un objectif absolu : c'est un arbitrage, et le vrai travail consiste à savoir justifier chaque écart.

## Prouver la sécurité, pas l'affirmer

Voici la distinction qui a structuré toute la seconde moitié du projet : un **pentest** cherche à exploiter une faille jusqu'au bout. Un **assessment de sécurité** cartographie, détecte, évalue — et s'arrête à la preuve. Depuis Kali, en suivant une version simplifiée du standard PTES (reconnaissance, énumération, analyse de vulnérabilités, rapport), l'objectif n'était jamais d'obtenir un accès, mais de documenter précisément ce qu'un attaquant externe pourrait voir et faire.

Le résultat de la reconnaissance a validé le travail des semaines précédentes : sur 65535 ports scannés, seuls deux répondaient — SSH pour l'administration, et le port de la passerelle. Un test de segmentation ciblé sur les ports internes (la base de données, la plage des backends) a confirmé qu'ils étaient tous `filtered` — invisibles, pas juste fermés. Et pour ne pas confondre une segmentation qui fonctionne avec un système qui serait simplement éteint, j'ai vérifié l'autre face : depuis l'intérieur, la passerelle continuait de dialoguer normalement avec les backends et la base.

Le moment le plus parlant du projet reste une capture Wireshark. L'audit avait révélé, par simple lecture des en-têtes HTTP, que la passerelle ne chiffrait rien — pas de TLS, pas de `Strict-Transport-Security`. Une affirmation reste abstraite tant qu'elle n'est pas montrée. J'ai donc lancé une capture réseau pendant qu'une requête transitait vers la passerelle, puis ouvert le flux capturé dans Wireshark. Le contenu de la requête — en clair, lisible — confirmait en une image ce que les en-têtes laissaient seulement supposer. C'est la différence entre dire « le trafic n'est pas chiffré » et le montrer noir sur blanc.

## Transformer des commandes en rapport

Un audit qui reste dans un terminal ne sert à personne. La dernière étape a consisté à synthétiser toutes les trouvailles — des deux dernières semaines, côté Kali comme côté Ubuntu — en un rapport hiérarchisé par criticité, où chaque ligne suit la même structure : la trouvaille, sa preuve, sa recommandation.

L'absence de TLS est ressortie comme la priorité haute, avec la capture Wireshark en pièce à conviction. Une image PostgreSQL référencée par tag plutôt que par hash exact, des en-têtes de sécurité HTTP absents, ont atterri en criticité moyenne — des vraies faiblesses, mais qui nécessitent des conditions supplémentaires pour être exploitées. Et puis des écarts mineurs, assumés et expliqués plutôt que corrigés à l'aveugle : une capacité Linux en trop sur les conteneurs web, justifiée par un test empirique qui avait montré qu'enlever cette capacité faisait planter le service.

Ce format — criticité, preuve, recommandation — n'est pas un exercice académique. C'est la structure qu'attend n'importe qui doit lire un rapport de sécurité sans avoir soi-même lancé les commandes : savoir quoi corriger en premier, et pourquoi.

## Ce que je retiens

Aucune des commandes de ce projet n'était compliquée en soi. Ce qui a pris du temps, c'est de comprendre *pourquoi* elles se comportaient parfois autrement que prévu — pourquoi Terraform recréait des ressources sans raison apparente, pourquoi un conteneur durci refusait de démarrer, pourquoi un score de sécurité ne se résume jamais à une seule note.

Deux idées me suivront au-delà de ce TP. La première : un outil « compatible » avec un autre ne l'est jamais totalement — la friction fait partie du travail, pas un accident à éviter. La seconde : la sécurité ne se décrète pas, elle se prouve. Affirmer qu'un réseau est isolé n'a aucune valeur tant qu'on ne l'a pas vérifié depuis l'extérieur, avec les mêmes outils qu'un attaquant utiliserait.

L'infrastructure tient debout aujourd'hui, avec ses forces documentées et ses limites assumées. C'est, je crois, le seul type de conclusion honnête qu'un audit puisse offrir.
