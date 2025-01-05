# 05/01/2025

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

# 26/12/2024

Après avoir lu le livre [Linux - De la ligne de commande à l'administration système](https://www.editions-eni.fr/livre/linux-de-la-ligne-de-commande-a-l-administration-systeme-9782409045929), j'ai mis en place un serveur et un client Puppet avec un manifest appliquant une configuration minimaliste. Ce livre m’a permis m'a permis de réviser des principes fondamentaux de la sécurité Linux (en plus des droits d'accès) notamment :

- SELinux
- AppArmor
- Les ACL (Access Control Lists)
- Les Capabilities


J'ai également approfondi mes connaissances sur la conteneurisation avec Docker et l'orchestration avec Kubernetes. Chaque fois qu'un concept me semblait intéressant, je prenais le temps de l'expérimenter concrètement sur une ou plusieurs machines virtuelles, afin de solidifier mes acquis.


