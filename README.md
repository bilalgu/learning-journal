# Formation et certification LPIC-1

*09/02/2025*

Cette semaine j'ai eu l'occasion de pas mal me former. J’ai terminé le livre **[Linux - Préparation à la certification LPIC-1 (examens LPI 101 et LPI 102)](https://www.editions-eni.fr/livre/linux-preparation-a-la-certification-lpic-1-examens-lpi-101-et-lpi-102-7e-edition-9782409043109)** et mis en pratique les connaissances acquises. Prochaine étape : **passer l’examen LPI 101** (inscription déjà faite !).

En attendant, j’ai attaqué une autre lecture : **[UNIX and Linux System Administration Handbook](https://www.amazon.com/UNIX-Linux-System-Administration-Handbook/dp/0134277554)**. 

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


