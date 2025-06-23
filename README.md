# 2025

## Logging & dashboards

*23/05/2025*

Encore une semaine riche !

Notamment dans les autres aspects de ma vie où j'ai dû allouer pas mal de temps, sachant que j'avais aussi mes responsabilités d'ingénieur système et cybersécurité. Mais quand j'avais du temps libre, j'ai continué à faire avancer mon projet DevOps bootstrap et j'ai attaqué (et fini !) le bloc logging et dashboards.

**Observabilité : logging vs monitoring**

Une distinction importante que j'ai mieux comprise cette semaine : le logging et les dashboards sont différents du monitoring et de l'alerting, bien que tout ça s'inscri dans le domaine de "l'observabilité". Avant d'avoir réellement mis les mains dans le cambouis, ce n'était pas évident pour moi.

J'ai donc mis en place la stack classique :

- **Promtail** (collecte des logs)
- **Loki** (stockage et indexation des logs)
- **Grafana** (visualisation et dashboards)

**Debugging : le piège du curl**

Un point qui m'a marqué : quand j'essayais de tester certains endpoints comme `http://<EC2_PUBLIC_IP>/monitoring/grafana`, bien que dans le browser l'endpoint était fonctionnel, en ligne de commande curl me renvoyait :

```http
405 Method Not Allowed
```

La cause : curl envoie par défaut une requête de type GET, mais certains endpoints n'acceptent que des méthodes spécifiques.

Solution : spécifier explicitement la méthode HTTP, par exemple :

```bash
curl -X POST ...
```

***
## Prise de tête DNS

*15/06/2025*

Cette semaine, le DNS a volé mon temps ! 

Je me suis creusé la tête sur un problème bien spécifique : comment faire en sorte que lorsque Ansible lance docker-compose, mon nom de domaine soit bien propagé dans la plupart des resolvers DNS (en particulier celui de Traefik), pour qu'il puisse me générer un certificat HTTPS à coup sûr.

Le contexte : j'ai pris un nom de domaine sur Dynadot, et j'ai réussi via l'API Dynadot à automatiquement associer une nouvelle IP à mon domaine. Cette nouvelle IP est celle de ma VM EC2 qui est générée à chaque `terraform destroy` + `apply` (ce que je fais constamment durant mes tests).

Je n'ai pas encore résolu à 100% mon problème, mais voici ce que j'ai appris :

1. **La propagation DNS prend du temps**

Une modification d'enregistrement DNS ne la rend pas visible immédiatement. Il faut attendre que le DNS se "propage" à travers les différents resolvers.

D'ailleurs, petite astuce pour récupérer son IP publique :

```bash
curl -s ifconfig.me
```

2. **Let's Encrypt a des limites**

Let's Encrypt limite le nombre de certificats pour le même nom de domaine dans un laps de temps donné.

Pendant le développement, utiliser absolument l'environnement de staging ! : https://letsencrypt.org/docs/staging-environment/

3. **Piège avec les variables Bash**

En Bash, `$(...)` écrase les sauts de ligne, ce qui peut rendre `grep` inefficace :

```bash
LOGS=$(docker logs traefik)
echo $LOGS | grep "ERR.*Unable to obtain"
# → Rien ne s'affiche
```

Alors que directement :

```bash
docker logs traefik | grep "ERR.*Unable to obtain"
# → Fonctionne correctement
```

**Ma solution actuelle**

Pour contourner cette histoire de DNS, Ansible lance un script bash qui inspecte les logs du container Traefik. S'il détecte une erreur de certificat, il relance automatiquement toute la stack.

***

## Réinstall RHEL : LDAP, NFS & audit

*08/06/2025*

Cette semaine, le "highlight" a été la réinitialisation d'un serveur Rocky vers une version spécifique avec son noyau.

La post-installation du serveur : il était là le vrai challenge !

- J'avais fait un backup des fichiers des interfaces réseau `/etc/sysconfig/network-scripts/ifcfg-*`. Je me suis rendu compte qu'avec SELinux activé, il fallait corriger les étiquettes avec `restorecon`. Restaurer les contextes SELinux lors de copie de fichiers, on y pense pas directement ! (D'ailleurs, faire gaffe avec la méthode de remise des fichiers d'interfaces réseau, car l'UID des connexions NMCLI change)

- En parlant de backup, je me suis même fait une espèce de petit script qui fait un audit de la machine avant réinitialisation pour rapidement la remettre comme il faut :

```bash
DEST="preinstall_audit_$(date +%Y_%m_%d)"
mkdir -p "$DEST" "$DEST/network-scripts"

echo "[+] Collecte des infos dans $DEST"

lsblk > "$DEST/01_lsblk.txt"
blkid > "$DEST/02_blkid.txt"
vgs   > "$DEST/03_vgs.txt"
lvs   > "$DEST/04_lvs.txt"
pvs   > "$DEST/05_pvs.txt"
cat /etc/fstab                        > "$DEST/06_fstab.txt"
cat /etc/exports                      > "$DEST/07_exports.txt"
mount | grep nfs                     > "$DEST/08_nfs_mounts.txt"
ip r                                 > "$DEST/09_ip_route.txt"
rpm -qa                              > "$DEST/10_all_rpms.txt"
rpm -qa | grep -Ei 'sssd|ldap|nslcd|krb5|realm' > "$DEST/11_auth_related_rpms.txt"
cat /etc/sssd/sssd.conf              > "$DEST/12_sssd_conf.txt"
mount                                > "$DEST/13_all_mounts.txt"
df -h                                > "$DEST/14_df_h.txt"
cp /etc/sysconfig/network-scripts/ifcfg-* "$DEST/network-scripts/"

ls -l "$DEST"
```

- Pareil, pour remettre le serveur NFS et activer les bons services via `firewall-cmd`, ou bien remettre le LDAP en faisant attention au fichiers PAM du style `/etc/authselect/password-auth` fait toute la différence !

***

## DevOps Bootstrap Stack

*01/06/2025*

Cette semaine j'ai pu faire tellement de trucs et apprendre énormément sur mon projet DevOps-Bootstrap !

D'ailleurs, j'ai affiné ma réflexion par rapport à ce projet. Si je devais le présenter, je dirais que c'est un projet proche de la production qui provisionne l'infrastructure, automatise la configuration et déploie des services web sécurisés et évolutifs en utilisant des conteneurs.

L'objectif c'est de construire une stack conçue pour les développeurs backend et les ingénieurs infrastructure.

### Ce que j'ai réalisé cette semaine

J'ai énormément avancé et j'ai pu :

**1. Mettre en place un reverse proxy avec Traefik** : J'ai intégré Traefik dans ma stack (web/back/db) qui route dynamiquement :

- `/` vers le service `web`
- `/api` vers le service `back`

--> [doc](https://github.com/bilalgu/devops-bootstrap/blob/main/docs/06-architecture-v2.md)

**2. Introduire le monitoring en temps réel** : J'ai ajouté des métriques pour surveiller les performances au niveau des conteneurs :

- `cadvisor` — expose les métriques des conteneurs et de l'hôte
- `prometheus` — collecte ces métriques

--> [doc](https://github.com/bilalgu/devops-bootstrap/blob/main/docs/07-monitoring.md)

Avec ce bloc terminé, la [v2.0.0](https://github.com/bilalgu/devops-bootstrap/releases/tag/v2.0.0) est terminée !

**3. Use Case : Backend API as a Service** : J'ai fini la semaine en montrant comment un développeur backend peut déployer une API (Node.js + PostgreSQL) en quelques minutes avec la Stack.

--> [doc](https://github.com/bilalgu/devops-bootstrap/blob/main/docs/08-use-case-api.md)

### En cours : HTTPS et domain routing

Actuellement, je travaille sur l'ajout d'un accès sécurisé à tous les services via HTTPS, en utilisant un vrai nom de domaine et la génération automatique de certificats avec Let's Encrypt (via Traefik).

J'ai déjà acheté le nom de domaine, mais il reste un point à automatiser : à chaque fois qu'on crée automatiquement une VM avec une nouvelle IP via Terraform, il faut encore changer manuellement le DNS dans le provider du nom de domaine.

En bref, on continue d'avancer, de progresser et d'apprendre !

***

## Full stack et docker-compose

*25/05/2025*

Cette semaine, j'ai super bien avancé dans mon projet (et le résultat me fait kiffer !)

J'ai fait évoluer l'architecture que je déploie automatiquement via Ansible vers quelque chose de beaucoup plus robuste : une stack complète orchestrée par docker-compose. L'approche est maintenant bien plus professionnelle et maintenable.

La nouvelle architecture se compose de trois services :

- **`web`** - Frontend statique (HTML/CSS/JS servi par Nginx)
- **`back`** - Backend/API (Node.js avec Express)
- **`db`** - Base de données (PostgreSQL)

Cette stack se déploie entièrement via docker-compose, lui-même lancé par Ansible. 

Un aspect qui m'a marqué, c'est les **health checks**. J'ai configuré un health check sur le service `db` pour que le backend ne démarre qu'une fois que PostgreSQL est prêt à accepter des connexions.

```yaml
depends_on:
  db:
    condition: service_healthy
```

Ça évite les erreurs de connexion au démarrage (que j'ai eues).

Pour les détails techniques, j'ai documenté cette évolution dans [docs/06-architecture-v2](https://github.com/bilalgu/devops-bootstrap/blob/main/docs/06-architecture-v2.md)

Et un petit aperçu du docker-compose pour finir :

```yaml
services:
  web:
    build: ./web
    ports:
      - "80:80"
    depends_on:
        back:
            condition: service_started
    networks:
      - app-net
  back:
    build: ./backend
    ports:
      - "8080:8080"
    depends_on:
      db:
        condition: service_healthy
    environment:
      DB_HOST: db
      DB_USER: appuser
      DB_PASSWORD: mysecretpassword
      DB_NAME: appdb
    networks:
      - app-net
  db:
    image: "postgres:alpine"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U appuser -d appdb"]
      interval: 5s
      timeout: 5s
      retries: 5
    restart: always
    environment:
      POSTGRES_DB: appdb
      POSTGRES_USER: appuser
      POSTGRES_PASSWORD: mysecretpassword
    volumes:
      - pgdata:/var/lib/postgresql/data
    networks:
      - app-net
volumes:
  pgdata:
networks:
  app-net:
```

***
## Documentation et nftables

*18/05/2025*

De retour de mon voyage, j'ai dû m'attaquer à un bon nombre de tickets qui se sont accumulés pendant la semaine et demie où j'étais absent. Mais j'ai quand même pu avancer sur mon projet :

J'ai créé un dossier complet de documentation ([devops-bootstrap/docs](https://github.com/bilalgu/devops-bootstrap/tree/main/docs)) où j'ai détaillé chaque étape de mon projet jusqu'à présent :

  - Provisioning
  - Configuration
  - Security hardening

J'ai réussi à "ansibiliser" mes configurations fail2ban de la semaine dernière.

J'ai également implémenté des règles de firewall avec nftables qui s'activent automatiquement au démarrage de la VM EC2 (toujours via Ansible). Pour ça j'ai :

  - Créé un service systemd qui applique ces règles à chaque redémarrage
  - Fait particulièrement attention à ne pas écraser les règles de firewall utilisées par SSH

***

## Regex fail2ban et aventures !

*11/05/2025*

Une semaine assez calme car j'étais en voyage en Arabie Saoudite à l'aventure ! J'ai tout de même profité des quelques heures libres que j'avais pour continuer à avancer sur mon projet.

Toujours sur la partie sécurité, j'ai enfin pu mettre en place ma jail qui détecte les logs d'erreur SSH d'Amazon EC2. Voici à quoi ressemble le filtre que fail2ban va utiliser pour les connexions SSH :

```
[Definition]

prefregex = ^.*sshd\[<F-MLFID>\d+</F-MLFID>]: <F-CONTENT>.+</F-CONTENT>$
failregex =
			^<F-NOFAIL>AuthorizedKeysCommand</F-NOFAIL> .+ failed, status \d+$
			^Connection closed by .* <HOST> port \d+ \[preauth\]$
ignoreregex =
```

Le `prefregex` fait en sorte de capturer le contenu de plusieurs lignes qui ont le même `sshd[PID]`. Ensuite, on analyse ce contenu dans `failregex` : s'il y a un enchaînement d'un `AuthorizedKeysCommand` avec un "failed status" suivi d'un "connection closed" avec une IP, alors celle-ci est détectée. En fonction du `maxretry` configuré dans ma jail, l'adresse IP sera bannie ou non.

La balise `<F-NOFAIL>` est vraiment importante ici car, sans elle, comme la ligne "AuthorizedKeysCommand" ne contient pas d'adresse IP, fail2ban générerait un message d'erreur.

***

## Première release et fail2ban

*04/05/2025*

J'ai pu pas mal avancer dans mon projet ! J'ai mis en place une pipeline CI/CD avec GitHub Actions (d'ailleurs j'ai dû créer une branche parallèle `ci/test-pipeline`, que j'ai ensuite mergée dans le main tellement j'ai dû faire de tests avec les `git push`).

J'ai restructuré l'ensemble, refait le README et finalisé la stack complète que j'avais imaginée pour une V1 de ce projet. J'en ai donc fait une première [release](https://github.com/bilalgu/devops-bootstrap/releases/tag/v1.0.0).

Maintenant, j'ai imaginé les axes d'amélioration pour une V2 que j'ai créés dans une [roadmap](https://github.com/bilalgu/devops-bootstrap/blob/main/ROADMAP.md). Je travaille actuellement sur le partie sécurité. J'essaie de durcir la configuration SSH de la VM EC2 qui est créée automatiquement dans ma stack et aussi d'ajouter fail2ban dessus.

Sauf que... je suis en train de plonger dans les regex de fail2ban en essayant de mettre en place ma propre jail, car les logs d'échecs SSH d'Amazon EC2 ne sont pas les mêmes que sur SSH classique, et ça change tout !

***

## Ansible et provisionnement automatique

*27/04/2025*

Au-delà de mon travail d'ingénieur système et cybersécurité, où j'ai dû m'engouffrer dans un code Python à plus de 1300 lignes qui interagit avec d'autres gros scripts pour déboguer un service, j'ai pu avancer sur mon projet cette semaine !

J'ai franchi plusieurs étapes :

1. Mise en ligne du projet sur GitHub
2. Provisionnement automatique d'une VM via Terraform
3. Configuration de la VM avec Ansible pour installer Docker
4. Déploiement d'un conteneur à partir d'un Dockerfile qui lance un site web statique Nginx (toujours avec Ansible)

Un simple `terraform init`, `plan`, `apply` suivi d'un `ansible-playbook -i inventory.ini playbook.yml` et boum ! Tout se déploie automatiquement.

À chaque étape, j'ai essayé de documenter proprement le projet avec un README. Pour voir l'avancée actuelle du projet, voici le [dernier commit](https://github.com/bilalgu/devops-bootstrap/tree/0e430004dda4da47ca23e1e58c62cbcd006a6d24).

Mon prochain challenge : mettre en place une pipeline CI/CD avec GitHub Actions. L'objectif est que chaque push sur la branche main qui modifie certains fichiers dans l'arborescence déclenche automatiquement le déploiement sur ma VM.

***

## Début de projet DevOps

*20/04/2025*

Pour rebondir sur la semaine précédente, après avoir passé plusieurs heures d'affilée à la lecture du livre sur les réseaux (qui est super intéressant et qui explique simplement des notions "complexes") - d'ailleurs, leur explication du modèle OSI est excellente ! - j'ai senti qu'il me manquait quand même de la pratique dans ma journée.

J'ai donc finalement franchi le cap et structuré la mise en place d'un projet centré sur l'apprentissage d'un domaine qui m'intéresse de plus en plus : le DevOps. Ce projet intègre des technologies d'Infrastructure as Code, de Configuration Management, de Conteneurisation et de Déploiement Continu (CI/CD).

Malheureusement, mes autres contraintes et responsabilités font que ce projet passe en dernière priorité :'(. Je n'ai pu avancer dessus que quelques soirs, mais au stade où j'en suis, j'ai réussi avec Terraform à créer une instance EC2 AWS avec un security group me permettant de me connecter en SSH avec une clé - et tout ça juste avec un `terraform apply` !

***

## Médicat, Fortinet et Réseaux

*13/04/2025*

Cette semaine a été "un peu moins dense", car je me suis principalement concentré sur la rédaction d'un long rapport de plus d'une trentaine de pages.

Au travail, j'ai réinstallé et configuré plusieurs machines. J'ai même pu récupérer des ordinateurs destinés à la réforme qui présentaient "seulement" des problèmes de BIOS ou de batterie. Un super outil qui m'a aidé et que je recommande vivement est [Medicat](https://medicatusb.com/) - une véritable boîte à outils pour le diagnostic et la réparation de systèmes.

Par curiosité, la semaine dernière j'ai commencé à me former via la plateforme Fortinet, notamment sur la certification [Fortinet Certified Fundamentals Cybersecurity](https://training.fortinet.com/local/staticpage/view.php?page=fcf_cybersecurity). Mais honnêtement, après quelques heures, mon enthousiasme était limité. J'ai trouvé que cela "apportait peu de valeur" au vu de ma situation actuelle, comparé à d'autres actions que je pouvais entreprendre.

J'ai donc pivoté vers la lecture d'un nouveau livre (pour changer, haha) : [Réseaux informatiques - Notions fondamentales](https://www.editions-eni.fr/livre/reseaux-informatiques-notions-fondamentales-10e-edition-protocoles-architectures-reseaux-sans-fil-9782409048265). J'ai tout de suite accroché à ce contenu. Mais, la question de commencer à réaliser des projets commence vraiment à me trotter dans la tête - peut-être est-il temps de passer de la théorie à plus de pratique ?

***

## NAT et iptables

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

## Capabilites et firewall

*30/03/2025*

Cette semaine je n'ai pas eu l'occasion de me former car j'étais à fond dans le travail, mais j'ai appris et pratiqué plein de choses intéressantes.

J'ai notamment vu les "capabilities" Linux, un concept intéressant qui décompose les droits root en plusieurs "petits droits" qu'on peut ensuite attribuer à des binaires ou à des processus spécifiques.

Par exemple, si je souhaite manipuler des règles iptables via un script Python, au lieu de le lancer avec les droits root (ce qui risquerait de créer des conflits avec l'arborescence de fichiers créée par Python), je peux simplement donner la capability `CAP_NET_ADMIN` au binaire Python. Même si ça crée potentiellement un trou de sécurité dans le sens où n'importe qui exécutant Python pourrait interférer avec les règles du firewall.

J'ai aussi vu l'importance de la règle nft/iptables `ct state ESTABLISHED,RELATED accept`. Sans ça, les requêtes DNS ne fonctionnent pas correctement !

***

## Fin de lecture

*23/03/2025*

100% de la lecture du livre de préparation à la certification LPIC-2 terminé (~900 pages) ! Tout en mettant en pratique chaque chapitre, les derniers modules qui m'ont marqué :

- **Gestion des clients réseau** : (OpenLDAP, DHCP)
- **Serveurs de fichiers** : (Samba, NFS)
- **Services e-mail** : Configuration de Postfix

La prochaine étape : déconstruire mes notes et préparer l'examen 201. 

***

## AC Adapter

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

## Préparation LPIC-2

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

## Hardware et petite sœur en détresse

*02/03/2025*

Après avoir décroché la certification LPIC-1, j'ai entamé la lecture de [LINUX - Préparation à la certification LPIC-2](https://www.editions-eni.fr/livre/linux-preparation-a-la-certification-lpic-2-examens-lpi-201-et-lpi-202-5e-edition-9782409038303). Pour l'instant, j'ai vu tout ce qui concerne le démarrage d'un système et les différents périphériques de stockage (NVMe, RAID, etc.). C'est marrant comme à chaque relecture, je comprends ces concepts de mieux en mieux !

Au travail, on m'a confié une nouvelle mission : démonter un ordinateur pour y installer une nouvelle carte graphique, puis le configurer dans notre DNS afin de l'utiliser comme serveur hébergeant un micro-service. Le côté hardware, c'est un domaine où j'ai peu d'expérience : savoir où placer la carte graphique et comment la connecter avec les nappes, c'était une grande première pour moi. Encore une occasion d'apprendre !

Et pour finir la semaine, ma petite sœur m'a appelé à la rescousse ce weekend. Son mini-PC [Blackview MP60](https://www.blackview.fr/products/blackview-mp60) refusait de démarrer sous Windows, restant bloqué sur un écran de dépannage après le BIOS. J'ai réussi à booter l'ordinateur avec un SystemRescue, puis à me connecter dessus en SSH pour analyser le disque contenant l'OS Windows. J'ai utilisé des commandes comme `lsblk`, `blkid`, `fdisk` et `badblocks`. Et en les lançant dans un `tmux`, j'ai pu me déconnecter de ma session SSH sans interrompre les interrompre.

Ce cas m'a fait prendre conscience des progrès que j'ai réalisés. Il y a quelques mois à je n'aurais jamais été capable de faire ça !

***

## Fin certification LPIC-1 

*23/02/2025*

Cette semaine j'ai eu l'opportunité de consacré pas mal de temps à la préparation de l'examen LPI 102, la deuxième partie de la certification LPIC-1. 

Pour me préparer, je me suis entraîné sur des TPs et des QCMs que j'ai notamment générer via l'IA (Claude). Voici un exemple de [prompt](Ressources/prompt-claude-lpi102.md).

Lors de mes entraînements, je me suis attardé sur le concept de métrique dans les tables de routage. Ce [cas d'étude](Ressources/metric-route.md) m'a permis d'en apprendre plus sur le sujet.

En fin de semaine, j'ai décidé de passer l'examen LPI 102. Comme pour le 101, je me suis rendu dans un centre d'examen à Paris. Et bonne nouvelle : j'ai réussi l'épreuve avec un score de 620/800, soit 15,5/20 ! [(La certification)](https://www.credly.com/badges/14a172f1-0f1e-479b-bd35-54e9106316a4/public_url)

En parallèle, j'ai également progressé sur le sujet iptables/nftables abordé la semaine dernière. J'ai fait une [documentation](Ressources/nftables-doc.md) détaillant les configurations que j'ai mises en place. L'objectif est que mon collègue puisse facilement comprendre et prendre le relais.

***

## Python, iptables et LPI 101

*16/02/2025*

Une semaine riche !

Dans le cadre de mon travail, j’ai eu pour mission de sécuriser un microservice. Première étape : analyser son fonctionnement, comprendre son code et sa structure. J'ai donc fait pas mal de python ! 

J’ai notamment vu comment gérer des conteneurs Docker via des modules Python, mais aussi comment appliquer des règles iptables directement depuis un script.

En parlant d’iptables, j’ai approfondi mes connaissances sur son sujet. Une ressource qui m'a aidé : [iptables : the two variants and their relationship with nftables](https://developers.redhat.com/blog/2020/08/18/iptables-the-two-variants-and-their-relationship-with-nftables).

Et enfin la semaine ce clôture samedi matin, où j’ai passé et validé l’examen LPI 101 : 580/800 (soit 14,5/20)

***

## Formation et certification LPIC-1

*09/02/2025*

Cette semaine j'ai eu l'occasion de pas mal me former. J’ai terminé le livre [Linux - Préparation à la certification LPIC-1 (examens LPI 101 et LPI 102)](https://www.editions-eni.fr/livre/linux-preparation-a-la-certification-lpic-1-examens-lpi-101-et-lpi-102-7e-edition-9782409043109) et mis en pratique les connaissances acquises. Prochaine étape : **passer l’examen LPI 101** (inscription déjà faite !).

En attendant, j’ai attaqué une autre lecture : [UNIX and Linux System Administration Handbook](https://www.amazon.com/UNIX-Linux-System-Administration-Handbook/dp/0134277554). 

Petite découverte : le parcours de **Evi Nemeth**, pionnière dans le monde UNIX/Linux, qui a marqué plusieurs générations d’administrateurs système.

***

## En production : CentOS 7 --> Rocky 8

*02/02/2025*

Après les tests effectués sur VMs la semaine dernière, j’ai enfin appliqué la migration en **production**. Notre cluster tournant sous **CentOS 7** est maintenant en **Rocky Linux 8** !

Mais tout ne s’est pas passé comme prévu…

L’un des problèmes les plus inattendus a été l’échec de `dnf distrosync`, qui affichait une erreur indiquant un manque d’espace sur le **système de fichiers racine**. Après quelques vérifications avec `du`, j’ai identifié le coupable : **/var/cache/dnf**.

Solution trouvée :

- J’ai déplacé **/var/cache/dnf** vers un stockage **NFS**.
- Et j’ai créé un **lien symbolique** avec `ln -s` pour pointer vers ce nouvel emplacement.

***

## Migration CentOS 7 vers Rocky Linux 8

*26/01/2025*

Toujours dans mon activité d’ingénieur système, j’ai reçu comme mission de migrer nos serveurs (actuellement en CentOS 7) en Rocky Linux 8. Pour éviter tout impact en production, j’ai d’abord effectué ces tests sur une machine virtuelle qui se rapproche le plus des conditions en production.

Les ressources qui m’ont particulièrement aidées :

- [Guide officiel Rocky Linux : migrate2rocky](https://docs.rockylinux.org/guides/migrate2rocky/)
- [Guide de migration CentOS vers Rocky Linux](https://gist.github.com/Trogvars/d93f8e370e9d01d4afc6e2a7e8c69ab2)
- [Migrate CentOS 7/8 to Rocky Linux 8](https://www.golinuxcloud.com/migrate-centos-to-rocky-linux/)

La plus grande difficulté dans cette migration c'est les **problèmes de dépendances** des différents paquets… (et il y en a un paquet). Il faut aussi s'assurer de bien **changer les dépôts** correctement.

***
## Suite du DNS : dnsmasq et systemd-resolved

*19/01/2025*

Pour faire suite à la semaine dernière, on a réalisé qu'on utiliser bind9 alors que dans notre configuration actuelle, **NetworkManager** utilise déjà dnsmasq dans un de ses sous-processus. Après réflexion, on a donc décidé de configurer notre DNS interne avec **dnsmasq**.

Une ressource qui m’a particulièrement aidé dans cette tâche : [linuxtricks](https://www.linuxtricks.fr/wiki/dnsmasq-le-serveur-dns-et-dhcp-facile-sous-linux).

Points importants rencontrés :

- **Conflit avec systemd-resolved** : j’ai remarqué que **systemd-resolved** et **dnsmasq** entraient en conflit car ils écoutaient tous les deux sur le port 53.

- **Solution** : j’ai configuré dnsmasq pour écouter sur le port **5353**, et redirigé les requêtes DNS spécifiques via **systemd-resolved** vers l’interface où je souhaitais qu’il y ait le DNS de dnsmasq.

***
## DNS interne avec BIND9 et conflits dnsmasq

*12/01/2025*

Dans le cadre de mon contrat actuel en tant qu’ingénieur système, j’ai mis en place un DNS interne en utilisant **BIND9**. Une ressource particulièrement utile qui m’a aidé dans cette tâche : [tutos.eu](https://www.tutos.eu/3446).

Pendant cette configuration, j’ai également rencontré un problème avec **NetworkManager**. En investiguant, j’ai découvert qu’il lançait une instance de **dnsmasq**. En inspectant le service et le binaire de **dnsmasq**, j’ai constaté, grâce à l’outil `ldd`, que certaines bibliothèques partagées n’étaient pas accessibles, car elles se trouvaient dans un répertoire géré par **Snap**. Pour résoudre ce problème, j’ai utilisé **ldconfig** afin de rendre ces répertoires accessibles au système.

***
## Préparation LPIC-1 : Pratique, dépannage et scripting

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

# 2024

## Serveur Puppet et sécurité Linux

*26/12/2024*

Après avoir lu le livre [Linux - De la ligne de commande à l'administration système](https://www.editions-eni.fr/livre/linux-de-la-ligne-de-commande-a-l-administration-systeme-9782409045929), j'ai mis en place un serveur et un client Puppet avec un manifest appliquant une configuration minimaliste. Ce livre m’a permis m'a permis de réviser des principes fondamentaux de la sécurité Linux (en plus des droits d'accès) notamment :

- SELinux
- AppArmor
- Les ACL (Access Control Lists)
- Les Capabilities


J'ai également approfondi mes connaissances sur la conteneurisation avec Docker et l'orchestration avec Kubernetes. Chaque fois qu'un concept me semblait intéressant, je prenais le temps de l'expérimenter concrètement sur une ou plusieurs machines virtuelles, afin de solidifier mes acquis.


