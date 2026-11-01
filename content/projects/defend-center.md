---
title: "Lab Defend Center"
date: 2026-11-01T21:00:00+01:00
draft: false
description: "Un laboratoire VirtualBox qui reproduit l'architecture d'un Defend Center : pare-feu, détection, supervision et attaques de test."
summary: "Je monte dans VirtualBox une version réduite d'un Defend Center, un boîtier de sécurité open source pensé pour les petites entreprises, afin de valider son architecture et son temps de reconstruction."
tags: ["cybersécurité", "lab", "virtualbox", "opnsense", "wazuh", "suricata"]
categories: ["Projets"]
comments: false
---

Un Defend Center, c'est l'idée d'un boîtier de sécurité à destination des petites entreprises : on l'ajoute à leur système d'information existant, et il apporte d'un coup un pare-feu, une détection d'intrusion, un SIEM et des tableaux de bord, le tout en open source. Le service est complété par de l'humain : un RSSI à temps partagé, un consultant, un analyste SOC. L'argument commercial tient en une phrase : si le boîtier tombe en panne ou brûle, on peut le recréer en une journée.

Avant d'investir dans du matériel, je veux savoir si cette architecture tient debout et si cette promesse est réaliste. Ce projet est donc un laboratoire : une version réduite du Defend Center, montée dans VirtualBox sur un seul ordinateur.

*Projet en cours : la conception est faite, la réalisation et les résultats seront ajoutés au fil des étapes.*

## L'architecture en bref

Le lab est découpé en trois zones reliées par un pare-feu OPNsense : une zone d'attaque, une zone cliente à protéger et une zone d'administration qui supervise.

![Schéma du lab : OPNsense relie Internet et trois zones (attaque, cliente, administration)](schema.svg)

Tout le trafic entre les zones traverse OPNsense, qui bloque par défaut et n'ouvre que le strict nécessaire. Kali Linux joue l'attaquant extérieur et ne voit que le port 80 de l'application cible. Les machines de la zone cliente envoient leurs événements à Wazuh, dans la zone d'administration. Wazuh reçoit aussi les alertes de Suricata, la détection d'intrusion installée sur le pare-feu. Pour intervenir en SSH sur la zone cliente, il n'existe qu'un seul chemin : Guacamole, qui sert de bastion.

## Pourquoi ces choix

**Un pare-feu qui bloque par défaut.** Chaque flux autorisé correspond à un besoin précis. Les agents Wazuh ont accès aux ports 1514 et 1515 de la machine de supervision, les mises à jour passent par les ports 53, 80 et 443, et l'attaquant n'atteint que le port 80 de la cible. Le reste est bloqué et journalisé, ce qui alimente aussi la détection.

**Une cible volontairement vulnérable.** DVWA, une application web faite pour être attaquée, joue le service exposé d'une petite entreprise. Elle permet de vérifier que la supervision voit vraiment quelque chose quand on l'attaque.

**Un bastion plutôt que des accès directs.** Seule la machine Guacamole a le droit d'ouvrir une session SSH vers la zone cliente. Si l'accès d'administration est compromis, il n'y a au moins qu'un seul point à surveiller.

**Tout décrit en code.** Les machines Linux sont décrites avec Vagrant et configurées avec Ansible, et la configuration d'OPNsense est exportée en fichier. C'est ce qui rend le test de reconstruction possible.

## Les machines virtuelles

Le lab tient sur un ordinateur de 16 Go de RAM, dont 12 Go alloués aux machines virtuelles.

| Machine | Rôle | RAM |
|---|---|---:|
| `opnsense` | Pare-feu, routeur, Suricata, VPN WireGuard | 2 Go |
| `wazuh` | SIEM : collecte, corrélation, alertes, tableau de bord | 4 Go |
| `outils` | Grafana et Guacamole (bastion web) | 2 Go |
| `web-cible` | Application vulnérable DVWA, agent Wazuh | 1 Go |
| `poste-client` | Poste utilisateur simulé, agent Wazuh | 1 Go |
| `kali` | Outils d'attaque (allumée seulement pendant les tests) | 2 Go |

Ces valeurs sont indicatives. Wazuh à 4 Go est un minimum serré, sa documentation en recommande davantage dès que le nombre d'agents augmente.

Les réseaux utilisent les plages `10.10.10.0/24` pour la zone cliente, `10.10.20.0/24` pour l'administration et `10.10.99.0/24` pour l'attaque. OPNsense a une carte sur chacun, plus une carte en NAT pour sortir sur Internet.

## Déroulement

Le projet avance en quatre étapes, chacune validée par un test avant de passer à la suivante.

1. **Le réseau.** Création des réseaux internes, installation d'OPNsense et de ses règles. Le test : la cible sort sur Internet, Kali atteint le port 80 de la cible et ne voit rien de la zone d'administration.
2. **La supervision.** Installation de Wazuh, des agents sur les machines clientes, de Suricata sur le pare-feu, puis de Grafana et Guacamole. Le test : les agents apparaissent actifs dans Wazuh et les journaux du pare-feu s'y affichent.
3. **Les attaques.** Depuis Kali, un scan de ports avec nmap, une attaque par force brute et une injection SQL avec sqlmap sur DVWA, puis la modification d'un fichier surveillé. Le test : chaque attaque doit produire au moins une alerte. Celles qui passent inaperçues indiquent ce qu'il faut améliorer.
4. **La reconstruction.** Destruction complète du lab puis recréation à partir du code, en chronométrant. C'est le test de la promesse « recréé en une journée ».

## Ce que ce lab permet de conclure

Il valide le découpage en zones, la logique des règles de filtrage, la chaîne complète détection, alerte et tableau de bord, ainsi que le principe de reconstruction par le code.

Il ne dit rien des pannes matérielles, des performances réelles, de Ceph et de la haute disponibilité de Proxmox, ni de la charge d'un vrai SOC. Ces points demanderont une seconde phase sur une machine dédiée avec Proxmox, où l'on pourra aussi ajouter Security Onion et un annuaire Samba AD.

## Outils

OPNsense, Suricata, WireGuard, Wazuh, Grafana, Apache Guacamole, DVWA, Kali Linux, nmap, sqlmap, Vagrant, Ansible, VirtualBox.

## Résultats

*À compléter : captures du tableau de bord Wazuh, alertes obtenues pour chaque attaque, temps de reconstruction mesuré et lien vers le dépôt Git du Vagrantfile et des playbooks.*
