---
{"dg-publish":true,"permalink":"/1-projets/elasticlabs-cours/2026-ho-l-sdi-avec-docker-ensg/j2-conteneurisation-and-reverse-proxy/lab-01-mise-en-route/","tags":["gardenEntry"],"noteIcon":""}
---


Bienvenue dans cette série de cours sur la conception d'Infrastructures de Données Géospatiales sous forme de micro services. 

L'architecture finale visée est la suivante : 

![Archi cours ENSG 2025.png](/img/user/1.%20Projets/Elasticlabs%20-%20Cours/2025_SDI-with-Docker-2025-FR/1.%20Fondations%20et%20gestion%20des%20donnees/Labs/Archi%20cours%20ENSG%202025.png)

Listez les COTS employés par catégorie : 
 - Coeur de métier
 - Services de socle
 - Service de support
 - Services de données

## Configuration du lab

Sur votre machine, assurez-vous d'avoir installé les applications suivantes : 
- Chromium
- VSCode

### Dépôt github

Créez un dépôt github dédié à l'application. 

Créez un token applicatif et conservez-le précieusement, il vous sera très utile pour la gestion de vos fichiers par la suite! RDV à l'URL [GitHub tokens](https://github.com/settings/tokens) de votre compte Github. 

Créez un repository public (par exemple `ensg-sdi-2025`) contenant un fichier README.md et un fichier de licence MIT. 

### Création du répertoire racine de vos applications

Sur votre machine, naviguez à la racine de votre espace de fichiers personnel, et créez le dossier ``~/Apps`` destiné à contenir les clones de vos dépôts GIT. 

### Configuration de VSCode

L'éditeur de code VSCode est l'outil de référence, gratuit, pour la majorité des développeurs du monde entier depuis de nombreuses années. 

Configurez l'outil VSCode simplement, en intégrant GitHub : https://code.visualstudio.com/docs/sourcecontrol/intro-to-git

Clonez votre repository github avec la commande VSCode "CTRL + P suivi de git : clone", et ajoutez son répertoire de stockage à l'espace de travail VSCode. 

- [i] Après avoir ajouté votre dépôt initial, effectuez une modification du fichier ``README.md`` et faites un ``commit`` puis un ``push`` pour valider la configuration.


## Mise en oeuvre de Docker

**Docker** est un ensemble de produits PaaS (plateforme en tant que service) qui utilisent la virtualisation au niveau du système d'exploitation pour fournir des logiciels dans des packages appelés conteneurs. Les conteneurs sont isolés les uns des autres et regroupent leurs propres logiciels, bibliothèques et fichiers de configuration ; ils peuvent communiquer entre eux via des canaux bien définis. Tous les conteneurs sont gérés par un seul noyau de système d'exploitation et utilisent donc moins de ressources que les machines virtuelles — [Source](https://en.wikipedia.org/wiki/Docker_(software))


La conteneurisation présente plusieurs avantages, notamment les suivants :
- Plusieurs conteneurs peuvent s'exécuter sur la même machine et chaque conteneur dans un processus isolé.
- Les conteneurs prennent moins de place que les VM.
- Vous pouvez exécuter des conteneurs en quelques minutes, avec moins de mémoire car ils n'ont pas besoin du système d'exploitation complet.
- Nous pouvons déployer et tester des conteneurs partout.

### Installation 

L'installation de docker est très intuitive pour tout utilisateur de systèmes UNIX / Linux, et très bien documentée. 

RDV sur le site de l'éditeur : https://docs.docker.com/engine/install/ubuntu/
- Section  [Install using the `apt` repository](https://docs.docker.com/engine/install/ubuntu/#install-using-the-repository)
- Très important, ne pas oublier de dérouler les étapes de post-installation de docker! 
	- https://docs.docker.com/engine/install/linux-postinstall/

### Docker workshop

En guise d'introduction pratique à docker, le workshop officiel fait efficacement écho au contenu vu en cours. RDV sur https://docs.docker.com/get-started/workshop/ et suivez le jusqu'à la partie 8 pour prendre en main en douceur votre installation de docker. 