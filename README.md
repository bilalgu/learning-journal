# Début de projet DevOps

*20/04/2025*

Pour rebondir sur la semaine précédente, après avoir passé plusieurs heures d'affilée à la lecture du livre sur les réseaux (qui est super intéressant et qui explique simplement des notions "complexes") - d'ailleurs, leur explication du modèle OSI est excellente ! - j'ai senti qu'il me manquait quand même de la pratique dans ma journée.

J'ai donc finalement franchi le cap et structuré la mise en place d'un projet centré sur l'apprentissage d'un domaine qui m'intéresse de plus en plus : le DevOps. Ce projet intègre des technologies d'Infrastructure as Code, de Configuration Management, de Conteneurisation et de Déploiement Continu (CI/CD).

Malheureusement, mes autres contraintes et responsabilités font que ce projet passe en dernière priorité :'(. Je n'ai pu avancer dessus que quelques soirs, mais au stade où j'en suis, j'ai réussi avec Terraform à créer une instance EC2 AWS avec un security group me permettant de me connecter en SSH avec une clé - et tout ça juste avec un `terraform apply` !

***

# Médicat, Fortinet et Réseaux

*13/04/2025*

Cette semaine a été "un peu moins dense", car je me suis principalement concentré sur la rédaction d'un long rapport de plus d'une trentaine de pages.

Au travail, j'ai réinstallé et configuré plusieurs machines. J'ai même pu récupérer des ordinateurs destinés à la réforme qui présentaient "seulement" des problèmes de BIOS ou de batterie. Un super outil qui m'a aidé et que je recommande vivement est [Medicat](https://medicatusb.com/) - une véritable boîte à outils pour le diagnostic et la réparation de systèmes.

Par curiosité, la semaine dernière j'ai commencé à me former via la plateforme Fortinet, notamment sur la certification [Fortinet Certified Fundamentals Cybersecurity](https://training.fortinet.com/local/staticpage/view.php?page=fcf_cybersecurity). Mais honnêtement, après quelques heures, mon enthousiasme était limité. J'ai trouvé que cela "apportait peu de valeur" au vu de ma situation actuelle, comparé à d'autres actions que je pouvais entreprendre.

J'ai donc pivoté vers la lecture d'un nouveau livre (pour changer, haha) : [Réseaux informatiques - Notions fondamentales](https://www.editions-eni.fr/livre/reseaux-informatiques-notions-fondamentales-10e-edition-protocoles-architectures-reseaux-sans-fil-9782409048265). J'ai tout de suite accroché à ce contenu. Mais, la question de commencer à réaliser des projets commence vraiment à me trotter dans la tête - peut-être est-il temps de passer de la théorie à plus de pratique ?

***

# NAT et iptables

*06/04/2025*

Toujours dans la continuité de la semaine dernière : on reste dans le thème des firewalls — mais cette fois-ci, avec un cas concret de NAT.

J'ai voulu expérimenter la mise en place d’un petit réseau virtuel entre deux machines, pour permettre à une VM Rocky Linux d’accéder à Internet en passant par une VM Debian agissant comme routeur NAT.

- VM Debian a deux interfaces réseau :
    - une en NAT via VirtualBox
    - une en réseau privé hôte (connexion avec Rocky)
        
- VM Rocky Linux a une seule interface, en réseau privé hôte

Il faut d'abord activer le forwarding pour que le noyau Linux autorise le passage de paquets entre interfaces réseau.

```bash
# Activer temporairement
echo 1 > /proc/sys/net/ipv4/ip_forward

# Activer de manière permanente
vi /etc/sysctl.conf
# → décommenter ou ajouter : net.ipv4.ip_forward = 1
```

On indique à Rocky que Debian est sa passerelle par défaut :

```bash
ip route add default via 192.168.56.8 dev enp0s3
```

Après cette configuration, Debian pouvait bien faire un ping vers l’extérieur (`ping 8.8.8.8`), mais Rocky… non :

```bash
[root@rocky ~]# ping 8.8.8.8
PING 8.8.8.8 (8.8.8.8) 56(84) octets de données.
^C
--- statistiques ping 8.8.8.8 ---
2 paquets transmis, 0 reçus, 100% packet loss
```

Rocky envoie bien la requête vers 8.8.8.8 → Debian la relaie vers Internet, mais… le paquet garde comme IP source l’adresse privée de Rocky (192.168.56.9). Résultat : le serveur tente de répondre à une adresse qui n’existe pas sur Internet. D’où l’absence de réponse.

**Solution : MASQUERADE**

Cette règle a tout débloqué ! :

```bash
iptables -t nat -A POSTROUTING -o enp0s3 -j MASQUERADE
```

De  cette façon, Debian remplace l’adresse source des paquets sortants par sa propre IP sur l’interface NAT (`10.0.2.15`). Ainsi, les réponses peuvent revenir sans problème, et Debian les retransmet ensuite à Rocky.

```bash
[root@rocky ~]# ping -c 3 8.8.8.8
64 octets de 8.8.8.8 : icmp_seq=1 ttl=61 temps=9.39 ms
64 octets de 8.8.8.8 : icmp_seq=2 ttl=61 temps=6.27 ms
64 octets de 8.8.8.8 : icmp_seq=3 ttl=61 temps=11.5 ms
```

On peut le voir de manière plus concrète avec tcpdump (je fais un `ping 8.8.8.8` via Rocky).

Avant la règle NAT :

```bash
root@debian:~# tcpdump -i enp0s3 -n host 8.8.8.8
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on enp0s3, link-type EN10MB (Ethernet), snapshot length 262144 bytes
15:56:58.962831 IP 192.168.56.9 > 8.8.8.8: ICMP echo request, id 9, seq 1, length 64
15:56:59.971352 IP 192.168.56.9 > 8.8.8.8: ICMP echo request, id 9, seq 2, length 64
15:57:00.995193 IP 192.168.56.9 > 8.8.8.8: ICMP echo request, id 9, seq 3, length 64
```

Après la règle NAT :

```bash
root@debian:~# tcpdump -i enp0s3 -n host 8.8.8.8
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on enp0s3, link-type EN10MB (Ethernet), snapshot length 262144 bytes
15:56:08.551268 IP 10.0.2.15 > 8.8.8.8: ICMP echo request, id 8, seq 1, length 64
15:56:08.556382 IP 8.8.8.8 > 10.0.2.15: ICMP echo reply, id 8, seq 1, length 64
15:56:09.553435 IP 10.0.2.15 > 8.8.8.8: ICMP echo request, id 8, seq 2, length 64
15:56:09.562202 IP 8.8.8.8 > 10.0.2.15: ICMP echo reply, id 8, seq 2, length 64
15:56:10.555232 IP 10.0.2.15 > 8.8.8.8: ICMP echo request, id 8, seq 3, length 64
15:56:10.569560 IP 8.8.8.8 > 10.0.2.15: ICMP echo reply, id 8, seq 3, length 64
```

***

# Capabilites et firewall

*30/03/2025*

Cette semaine je n'ai pas eu l'occasion de me former car j'étais à fond dans le travail, mais j'ai appris et pratiqué plein de choses intéressantes.

J'ai notamment vu les "capabilities" Linux, un concept intéressant qui décompose les droits root en plusieurs "petits droits" qu'on peut ensuite attribuer à des binaires ou à des processus spécifiques.

Par exemple, si je souhaite manipuler des règles iptables via un script Python, au lieu de le lancer avec les droits root (ce qui risquerait de créer des conflits avec l'arborescence de fichiers créée par Python), je peux simplement donner la capability `CAP_NET_ADMIN` au binaire Python. Même si ça crée potentiellement un trou de sécurité dans le sens où n'importe qui exécutant Python pourrait interférer avec les règles du firewall.

J'ai aussi vu l'importance de la règle nft/iptables `ct state ESTABLISHED,RELATED accept`. Sans ça, les requêtes DNS ne fonctionnent pas correctement !

***

# Fin de lecture

*23/03/2025*

100% de la lecture du livre de préparation à la certification LPIC-2 terminé (~900 pages) ! Tout en mettant en pratique chaque chapitre, les derniers modules qui m'ont marqué :

- **Gestion des clients réseau** : (OpenLDAP, DHCP)
- **Serveurs de fichiers** : (Samba, NFS)
- **Services e-mail** : Configuration de Postfix

La prochaine étape : déconstruire mes notes et préparer l'examen 201. 

***

# AC Adapter

*16/03/2025* 

Une semaine où j'ai jonglé entre formation, résolution de tickets et missions confiées. 

Un cas intéressant cette semaine : un utilisateur est venu car son laptop ne se chargeait plus malgré le branchement de l'adaptateur secteur. En analysant la situation, on s'est rendu compte que l'adaptateur n'était pas assez puissant (130W) alors que le laptop nécessitait 180W. 

Pour résoudre le problème on a :

1. Ouvert le PC et démonté la batterie
2. Remis en place la batterie
3. Branché l'adaptateur secteur 

Et là, surprise ! La batterie a été détectée et a commencé à se charger. 

Avant ce cas, je ne pensais pas que la puissance d'un adaptateur jouait un rôle aussi crucial pour un bon fonctionnement de PC portable !

***

# Préparation LPIC-2

*09/03/2025*

Une semaine plus tranquille que les précédentes, ce qui m'a permis de me concentrer sur la lecture du livre de préparation à la certification LPIC-2. J'en suis à 60% de cet ouvrage de 882 pages.

Parmi les principaux sujets que j'ai vu cette semaine :

- Configuration réseau
- Maintenance système
- Noyau Linux
- Planification des ressources
- Serveurs de noms de domaine (DNS)

J'essaie de mettre en pratique au fur et à mesure les connaissances acquises. Actuellement, je suis sur la mise en place d'un serveur DNS de cache (appelé aussi "resolver" ou "caching name server").

***

# Hardware et petite sœur en détresse

*02/03/2025*

Après avoir décroché la certification LPIC-1, j'ai entamé la lecture de [LINUX - Préparation à la certification LPIC-2](https://www.editions-eni.fr/livre/linux-preparation-a-la-certification-lpic-2-examens-lpi-201-et-lpi-202-5e-edition-9782409038303). Pour l'instant, j'ai vu tout ce qui concerne le démarrage d'un système et les différents périphériques de stockage (NVMe, RAID, etc.). C'est marrant comme à chaque relecture, je comprends ces concepts de mieux en mieux !

Au travail, on m'a confié une nouvelle mission : démonter un ordinateur pour y installer une nouvelle carte graphique, puis le configurer dans notre DNS afin de l'utiliser comme serveur hébergeant un micro-service. Le côté hardware, c'est un domaine où j'ai peu d'expérience : savoir où placer la carte graphique et comment la connecter avec les nappes, c'était une grande première pour moi. Encore une occasion d'apprendre !

Et pour finir la semaine, ma petite sœur m'a appelé à la rescousse ce weekend. Son mini-PC [Blackview MP60](https://www.blackview.fr/products/blackview-mp60) refusait de démarrer sous Windows, restant bloqué sur un écran de dépannage après le BIOS. J'ai réussi à booter l'ordinateur avec un SystemRescue, puis à me connecter dessus en SSH pour analyser le disque contenant l'OS Windows. J'ai utilisé des commandes comme `lsblk`, `blkid`, `fdisk` et `badblocks`. Et en les lançant dans un `tmux`, j'ai pu me déconnecter de ma session SSH sans interrompre les interrompre.

Ce cas m'a fait prendre conscience des progrès que j'ai réalisés. Il y a quelques mois à je n'aurais jamais été capable de faire ça !

***

# Fin certification LPIC-1 

*23/02/2025*

Cette semaine j'ai eu l'opportunité de consacré pas mal de temps à la préparation de l'examen LPI 102, la deuxième partie de la certification LPIC-1. 

Pour me préparer, je me suis entraîné sur des TPs et des QCMs que j'ai notamment générer via l'IA (Claude). Voici un exemple de [prompt](Ressources/prompt-claude-lpi102.md).

Lors de mes entraînements, je me suis attardé sur le concept de métrique dans les tables de routage. Ce [cas d'étude](Ressources/metric-route.md) m'a permis d'en apprendre plus sur le sujet.

En fin de semaine, j'ai décidé de passer l'examen LPI 102. Comme pour le 101, je me suis rendu dans un centre d'examen à Paris. Et bonne nouvelle : j'ai réussi l'épreuve avec un score de 620/800, soit 15,5/20 ! [(La certification)](https://www.credly.com/badges/14a172f1-0f1e-479b-bd35-54e9106316a4/public_url)

En parallèle, j'ai également progressé sur le sujet iptables/nftables abordé la semaine dernière. J'ai fait une [documentation](Ressources/nftables-doc.md) détaillant les configurations que j'ai mises en place. L'objectif est que mon collègue puisse facilement comprendre et prendre le relais.

***

# Python, iptables et LPI 101

*16/02/2025*

Une semaine riche !

Dans le cadre de mon travail, j’ai eu pour mission de sécuriser un microservice. Première étape : analyser son fonctionnement, comprendre son code et sa structure. J'ai donc fait pas mal de python ! 

J’ai notamment vu comment gérer des conteneurs Docker via des modules Python, mais aussi comment appliquer des règles iptables directement depuis un script.

En parlant d’iptables, j’ai approfondi mes connaissances sur son sujet. Une ressource qui m'a aidé : [iptables : the two variants and their relationship with nftables](https://developers.redhat.com/blog/2020/08/18/iptables-the-two-variants-and-their-relationship-with-nftables).

Et enfin la semaine ce clôture samedi matin, où j’ai passé et validé l’examen LPI 101 : 580/800 (soit 14,5/20)

***

# Formation et certification LPIC-1

*09/02/2025*

Cette semaine j'ai eu l'occasion de pas mal me former. J’ai terminé le livre [Linux - Préparation à la certification LPIC-1 (examens LPI 101 et LPI 102)](https://www.editions-eni.fr/livre/linux-preparation-a-la-certification-lpic-1-examens-lpi-101-et-lpi-102-7e-edition-9782409043109) et mis en pratique les connaissances acquises. Prochaine étape : **passer l’examen LPI 101** (inscription déjà faite !).

En attendant, j’ai attaqué une autre lecture : [UNIX and Linux System Administration Handbook](https://www.amazon.com/UNIX-Linux-System-Administration-Handbook/dp/0134277554). 

Petite découverte : le parcours de **Evi Nemeth**, pionnière dans le monde UNIX/Linux, qui a marqué plusieurs générations d’administrateurs système.

***

# En production : CentOS 7 --> Rocky 8

*02/02/2025*

Après les tests effectués sur VMs la semaine dernière, j’ai enfin appliqué la migration en **production**. Notre cluster tournant sous **CentOS 7** est maintenant en **Rocky Linux 8** !

Mais tout ne s’est pas passé comme prévu…

L’un des problèmes les plus inattendus a été l’échec de `dnf distrosync`, qui affichait une erreur indiquant un manque d’espace sur le **système de fichiers racine**. Après quelques vérifications avec `du`, j’ai identifié le coupable : **/var/cache/dnf**.

Solution trouvée :

- J’ai déplacé **/var/cache/dnf** vers un stockage **NFS**.
- Et j’ai créé un **lien symbolique** avec `ln -s` pour pointer vers ce nouvel emplacement.

***

# Migration CentOS 7 vers Rocky Linux 8

*26/01/2025*

Toujours dans mon activité d’ingénieur système, j’ai reçu comme mission de migrer nos serveurs (actuellement en CentOS 7) en Rocky Linux 8. Pour éviter tout impact en production, j’ai d’abord effectué ces tests sur une machine virtuelle qui se rapproche le plus des conditions en production.

Les ressources qui m’ont particulièrement aidées :

- [Guide officiel Rocky Linux : migrate2rocky](https://docs.rockylinux.org/guides/migrate2rocky/)
- [Guide de migration CentOS vers Rocky Linux](https://gist.github.com/Trogvars/d93f8e370e9d01d4afc6e2a7e8c69ab2)
- [Migrate CentOS 7/8 to Rocky Linux 8](https://www.golinuxcloud.com/migrate-centos-to-rocky-linux/)

La plus grande difficulté dans cette migration c'est les **problèmes de dépendances** des différents paquets… (et il y en a un paquet). Il faut aussi s'assurer de bien **changer les dépôts** correctement.

***
# Suite du DNS : dnsmasq et systemd-resolved

*19/01/2025*

Pour faire suite à la semaine dernière, on a réalisé qu'on utiliser bind9 alors que dans notre configuration actuelle, **NetworkManager** utilise déjà dnsmasq dans un de ses sous-processus. Après réflexion, on a donc décidé de configurer notre DNS interne avec **dnsmasq**.

Une ressource qui m’a particulièrement aidé dans cette tâche : [linuxtricks](https://www.linuxtricks.fr/wiki/dnsmasq-le-serveur-dns-et-dhcp-facile-sous-linux).

Points importants rencontrés :

- **Conflit avec systemd-resolved** : j’ai remarqué que **systemd-resolved** et **dnsmasq** entraient en conflit car ils écoutaient tous les deux sur le port 53.

- **Solution** : j’ai configuré dnsmasq pour écouter sur le port **5353**, et redirigé les requêtes DNS spécifiques via **systemd-resolved** vers l’interface où je souhaitais qu’il y ait le DNS de dnsmasq.

***
# DNS interne avec BIND9 et conflits dnsmasq

*12/01/2025*

Dans le cadre de mon contrat actuel en tant qu’ingénieur système, j’ai mis en place un DNS interne en utilisant **BIND9**. Une ressource particulièrement utile qui m’a aidé dans cette tâche : [tutos.eu](https://www.tutos.eu/3446).

Pendant cette configuration, j’ai également rencontré un problème avec **NetworkManager**. En investiguant, j’ai découvert qu’il lançait une instance de **dnsmasq**. En inspectant le service et le binaire de **dnsmasq**, j’ai constaté, grâce à l’outil `ldd`, que certaines bibliothèques partagées n’étaient pas accessibles, car elles se trouvaient dans un répertoire géré par **Snap**. Pour résoudre ce problème, j’ai utilisé **ldconfig** afin de rendre ces répertoires accessibles au système.

***
# Préparation LPIC-1 : Pratique, dépannage et scripting

*05/01/2025*

Après avoir terminé la lecture du précédent livre, qui s'inscrit dans une série d'ouvrages sur l'administration Linux, je me suis fixé comme objectif : obtenir la certification LPIC-1. 

Je me suis donc procuré le livre [Linux - Préparation à la certification LPIC-1 (examens LPI 101 et LPI 102)](https://www.editions-eni.fr/livre/linux-preparation-a-la-certification-lpic-1-examens-lpi-101-et-lpi-102-7e-edition-9782409043109), toujours de la même édition (que je trouve super d'ailleurs).

À la fin de cette semaine, j'ai progressé jusqu'à 69 % du livre. J'ai pris soin de pratiquer chaque fois qu'un concept intéressant apparaissait. Le livre propose à la fin de chaque chapitre un QCM "Validation des connaissances", qui comporte généralement une cinquantaine de questions, ainsi qu'environ quatre travaux pratiques pour mettre en application ce qui a été appris.

Deux expériences m'ont particulièrement marqué cette semaine :

1. **Réparation d'une VM suite à une erreur critique**  
    En pratiquant, j'ai accidentellement déplacé une bibliothèque système essentielle sur ma VM. Cela a rendu le système complètement inutilisable (même les commandes de base comme `cd`, `ls` ou `mv` ne fonctionnaient plus). J'ai réussi à résoudre le problème en démarrant la VM avec une ISO de SystemRescue, en montant le système de fichiers et en le modifiant pour le "réparer". Ce qui était encore plus gratifiant, c'est que j'ai vraiment compris ce que je faisais pendant cette intervention !
    
2. **Création d'un script shell**  
    J'ai écrit un script shell qui prend en argument un fichier texte, lit deux nombres ligne par ligne, puis affiche leur somme. Voici le script :

```
[ $# -eq 1 -a -f $1 ] || exit 1
typeset -i var=0
while read a b
do
        var=$a+$b
        echo $a + $b = $var
done <$1
```

***

# Serveur Puppet et sécurité Linux

*26/12/2024*

Après avoir lu le livre [Linux - De la ligne de commande à l'administration système](https://www.editions-eni.fr/livre/linux-de-la-ligne-de-commande-a-l-administration-systeme-9782409045929), j'ai mis en place un serveur et un client Puppet avec un manifest appliquant une configuration minimaliste. Ce livre m’a permis m'a permis de réviser des principes fondamentaux de la sécurité Linux (en plus des droits d'accès) notamment :

- SELinux
- AppArmor
- Les ACL (Access Control Lists)
- Les Capabilities


J'ai également approfondi mes connaissances sur la conteneurisation avec Docker et l'orchestration avec Kubernetes. Chaque fois qu'un concept me semblait intéressant, je prenais le temps de l'expérimenter concrètement sur une ou plusieurs machines virtuelles, afin de solidifier mes acquis.


