
Toutes les informations ci-dessus ont été tirées par les vidéos YouTube de **HackerSploit**.

Vidéos consultées :

- [Docker Security Essentials | How to Secure Docker Containers](https://www.youtube.com/watch?v=KINjI1tlo2w&list=PLBf0hzazHTGNv0-GVWZoveC49pIDHEHbn&index=1)

- [Auditing Docker Security](https://www.youtube.com/watch?v=mQkVB6KMHCg&list=PLBf0hzazHTGNv0-GVWZoveC49pIDHEHbn&index=2)
# Avoir les derniers packages installés

- Permet d'installer des mises à jour de sécurité automatiquement : 

```sh
apt-get install unattended-upgrades # (normalement déjà installé sur Ubuntu)
```

  - Permet de se focaliser uniquement sur les patchs de sécurité :

```sh	
dpkg-reconfigure --priority=low unattended-upgrades	
```
	
  - Cela va créer 2 fichiers de configuration dans le dossier /etc/apt/apt.conf.d :
  
      - 20auto-upgrades : qui va ordonner l'exécution des scripts de mise à jour
      - 50unattended-upgrades : configurations pour la mise à jour automatique
	   	
	   - Possibilité de demander le redémarrage du serveur après mise à jour
	   - Être notifié par mail en cas de mise à jour	
	   - Pour vérifier l'exécution de la configuration appliquée : unattended-upgrade --dry-run --debug		 		
	

# L'accès SSH

À l'origine, SSH est censé être un protocole sécurisé, mais tend à avoir des failles en raison de mauvaises pratiques.

- Créer un utilisateur d'administration **non-root** :

```sh
useradd admin -m -s /bin/bash -c "admin user" # -m : création d'un HOME pour l'utilisateur admin
usermod -aG sudo,adm,docker admin # être intégré au groupe sudo + avoir la possibilité de lire des fichiers de log + ajouté au groupe docker"
passwd admin
```

- Créer une paire de clés (préférer Ed25519, mais attention aux incompatibilités : dans ce cas, préférer RSA), avec une **passphrase**. Ne **jamais partager la clé privée !**.

- Mettre le bon propriétaire pour la config SSH de l'utilisateur : `chown admin:admin /home/admin/.ssh`

- Modifier le fichier `/etc/ssh/sshd_config`
  - Mettre `PermitRootLogin` à `No`
  - Mettre `PasswordAuthentication` à `No`

## Passer par un proxy

SSH peut être suffisant s'il n'y a qu'une seule personne qui gère le serveur.
Mais si **l'accès doit être partagé**, il est toujours utile de passer par un **proxy pour l'authentification**.

Un outil intéressant : **Telebot**, qui est un proxy 2FA, et est open-source et gratuit.

# Exposition des services

Les services sont des processus qui sont continuellement actifs, et qui sont attachés sur le réseau.
Pour lister ceux qui sont actifs : `ss -ltpn`. **Toujours exposer les services nécessaires !**

Des services pertinents :

- Le serveur SSH (port 22 ou le changer s'il le faut)
- Un serveur HTTP (port 80, pour une application -> **préférer SSL/TLS (port 443)**)

# Configuration d'un firewall (avec UFW)

Uncomplicated Firewall est un outil qui permet de créer un firewall sans trop de complications (par rapport à `iptables`).

- Pour l'installer : `apt-get install ufw`
- Pour autoriser un port (ici le port de SSH) : `ufw allow 22`
- Activer le firewall ufw : `ufw enable` **(l'activation va uniqumement ouvrir les ports autorisés ! Donc bloquer les autres ports qui ne le sont pas !)**

**ATTENTION : CELA NE PREND PAS EN COMPTE LES CONTENEURS DOCKER PAR DÉFAUT !**

# Utiliser un reverse-proxy

Un reverse-proxy peut-être utile si on souhaite exposer une application non-sécurisée (qui tourne sur du HTTP).
Il s'agit d'un serveur entre le client et l'application web. Il faut quand-même ajouter une couche de sécurité en HTTPS pour le reverse-proxy afin que le trafic soit sécurisée (parce que l'application ne possède pas de certificat TLS actif)

Des reverse-proxy intéressants :

- Le reverse-proxy du serveur NGINX
- Le gestionnaire de reverse-proxy de NGINX (NGINX reverse-proxy manager), qui est une interface intuitive

# VPN et DMZ

Il est important de **priver** les interfaces d'administration (NGINX reverse-proxy manager par exemple), via un DMZ ou un VPN.

Des exemples de VPN : WireGuard, tailscale, OpenVPN 

# Isoler ses applications

Éviter que ses applications n'intéragissent trop avec le système.
Des solutions :

- AppArmor, qui est un MAC permettant de définir des règles de sécurité autorisant ou refusant l'accès à des ressources
- Les conteneurs Docker

# Docker

Docker est en soit une plateforme pas vraiment non-sécurisée, au contraire.
Les vulnérabilités peuvent provenir d'une mauvaise configuration (hôte, conteneurs...), on d'une mauvaise compréhension sur les ressources qu'il faudrait sécuriser.

Toutes les ressources de Docker se trouvent sur le chemin du système suivant : `/var/lib/docker`

Pour pouvoir utiliser Docker de manière sécurisé sur son environnement de production, il est nécessaire de se focaliser sur 3 points :

1. L'hôte de Docker
2. Le démon Docker
3. Les conteneurs

Des ressources intéressantes qui présentent les vulnérabilités de Docker :

- CVE (Common Vulnerabilities and Exposure), qui précise les vulnérabilités dans le code, des expositions de données : https://www.cvedetails.com/product/28125/Docker-Docker.html?vendor_id=13534

**Attention : malgré le fait qu'on voit des vulnérabilités en baisse, cela ne veut pas dire qu'il faut partir du postulat qu'il n'y a rien à renforcer / sécuriser davantage !**

- CIS (Center for Internet Security), qui présente un benchmark : https://www.cisecurity.org/benchmark/docker

### Créer un audit de sécurité

Il très recommandé de lancer un audit de sécurité sur les conteneurs Docker qu'on souhaite déployer en production.

Des outils d'audit : 

- **[docker-security-bench](https://github.com/docker-security-bench)**

- Chef InSpec : qui est un outil permettant de réaliser un audit sur **le système**.

Les outils d'audit se basent en général sur les données remontées par le CIS.

### 1. L'hôte de Docker

Il faut savoir que **la sécurité de l'hôte** tournant les conteneurs, joue naturellement sur la sécurité des conteneurs Docker.

Plusieurs recommendations : 

1. L'OS de l'hôte doit subir des audits de sécurité pour ensuite prendre les mesures nécessaires

2. L'OS doit est au configuré **au minimum**. Et devrait donc être utilisé **pour Docker exclusivement**. De telle sorte à réduire les risques d'attaque sur d'autres applications.

3. Il est préférable de se focaliser sur des OS minimaux en cloud-natives, car ils sont déjà configurés pour exécuter des conteneurs, et donc cela réduit les configurations à apporter.

Des exemples de OS minimaux (qui sont cloud-natives) :

- Ubuntu Core
- CoreOS (déprécié, mais a des patchs de sécurité)
- RancherOS


### 3. Les conteneurs Docker

- Lorsqu'on utilise des conteneurs Docker sur son système, il est toujours important de connaître **la source des images**.
Certaines d'entre-elles peuvent être **obsolètes** par exemple.

L'outil **watchtower** permet d'exécuter automatiquement des mises à jour sur les conteneurs.

**Attention :** les conteneurs sont **immutables**, ils doivent donc être au préalable, **détruits** avant de faire une mise à jour.

#### **Prévenir l'escalade en root**

- Créer et utiliser un utilisateur **non-privilégié** : 

```dockerfile
RUN groupadd -r user && useradd -r -g user user
```

- L'utilisateur pourra toujours changer en root avec`su`, il faut donc **désactiver l'utilisateur root dans le système :**

```dockerfile
RUN chsh -s /usr/sbin/nologin root
```

Même si un password pour le root du conteneur a été appliqué, il ne sera pas possible de changer d'utilisateur.

#### **Réduire les capacités d'un conteneur docker**

- Ne jamais lancer un conteneur **en mode privilégié (avec l'option `--privileged`) ! Cela permet sinon au conteneur d'utiliser toutes les _capacités / capabilities_ du noyau Linux !**
  1. Ajouter le flag `--security-opt=no-new-privileges` au docker run pour empêcher l'escalade de privilèges.
  2. Retirer tous les capacitiés que le conteneur peut exécuter avec le flag `--cap-drop all`
  3. Préciser la capacité que le conteneur doit avoir avec le flag `--cap-add`.
  
     Voir la page de `man capabilities` pour regarder la liste des capacités existantes.
     Une des capacités qu'il est intéressant d'ajouter est **CAP_NET_ADMIN** --> `--cap-add NET_ADMIN` (pour gérer les tâches liées au réseau, comme le démarrage d'un service qui s'attachera à un port par exemple).
     
#### **Restreindre l'accès au système de fichiers (si nécessaire)**

- Empêcher toute écriture à l'intérieur du conteneur, ajouter le flag `--read-only` au docker run
- OU créer un **accès temporaire à un endroit spécifique :** `--tmpfs <chemin désiré à l'intérieur du conteneur>`

#### **Empêcher des inter-communications**

Cela consiste à créer un `network` spécifique pour chaque groupe de conteneur.
Avec le network bridge, des conteneurs peuvent communiquer entre eux.
Mais il est possible de les isoler complètement à l'aide de l'option `"com.docker.network.bridge.enable_icc=false"`.

`docker network create -o "com.docker.network.bridge.enable_icc=false" <network_name>`
# Mise en pratique

Pour appliquer quelques mesures ci-dessus, notamment pour la sécurisation des conteneurs Docker, une VM sous Vagrant va être utilisée.

## Manipulations

- Installation de **docker-security-bench**

  - Via : `git clone git clone https://github.com/docker/docker-bench-security`

- Exécution de `docker-security-bench.sh`

  - Ce script explore chaque couche de Docker (hôte, conteneurs), et établit un rapport de sécurité, expliquant les points à être améliorés.

- Utilisation de **Chef InSpec**.

   - Installation via `curl https://omnitruck.chef.io/install.sh | sudo bash -s -- -P inspec` (cf. [documentation](https://docs.chef.io/inspec/install/))

   - Téléchargement du profil CIS de Docker : `git clone https://github.com/dev-sec/cis-docker-benchmark.git`

   Ce profil de sécurité contient toutes les best-practices pour sécuriser sa plateforme. Le télécharger, permettra à l'outil InSpec de se focaliser sur l'analyse de la plateforme Docker.

   Exécution de l'analyse de InSpec via cette commande :
   ```
   inspec exec cis-docker-benchmark
   ```

   Le même travail est réalisé par **docker-security-bench**. Ces deux utilitaires se basent sur le CIS. À la différence que InSpec fournit un rapport plus détaillé sur les actions à entreprendre.