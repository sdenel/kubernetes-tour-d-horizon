# Contexte
L'objectif de ce TP est de comprendre l'interet des ressources Kubernetes de type Pod, Deployment, Service, et Ingress.
* Vous avez un cluster à votre disposition. Accès derrière les IPs whitelistées exclusivement (me faire signe pour ouvrir plus large)
* Liens utiles :
    * https://k8s.denel.io/
    * Vous pouvez également récupérer le fichier de config de kubectl (~/.kube/config) et l'utiliser en local. Remplacer l'URL de l'APIServer par "master.k8s.lab.smartwavesa.com"

# I. Pod
1. Instancier un pod avec un serveur http en écoute sur le port 80 dans votre namespace avec la commande : `k apply -f votrefichier.yaml`
   Le fichier "1-pods.yaml" est une très bonne base. Vous pouvez également ajouter à ce pod des labels / annotations.
2. En binome :
    * **Tata** :
        * récupérer L'IP de votre pod :
            * Via le dashboard pour les petits joueurs
            * Via le CLI + jq pour les vrais : `k get pods mon-premier-pod -o json | jq -r '.status.podIP'
        * La fournir à votre camarade. Normalement, vous exposez en HTTP sur le port 80.
    * **Toto** :
        * Ouvrir un terminal dans votre pod : `k exec -ti mon-premier-prod /bin/sh`
        * Installer curl (apk curl) puis récupérer le contenu exposé par le pod de votre voisin
   On vient de montrer que par défaut, les pods du namespace de votre binome ont le droit d'accéder aux pods dans votre namespace, et que les pods munis d'un terminal (non distroless) peuvnet etre pratiques... pour un attaquant !
3. Supprimer votre pod pour simuler une erreur : il ne revient pas ! Un pod doit etre vu comme un objet immuable et éphémère.

**Pour ceux qui sont en avance (ou qui veulent rouler des mécaniques)...**
1. Utiliser l'image sdenel/hello-world :
    * Retenter `k exec -ti mon-premier-prod /bin/sh`
    * Comme vous le remarquez, ce n'est pas possible ! l'image ne contient que l'executable nginx et ses dépendances (sans bash ni sh ni aucun gestionnaire de paquet). Elle est distroless. Voir les sources : https://github.com/sdenel/docker-nginx-file-listing
2. Ajouter une livenessProbe au pod (qui expose actuellement sur le pod 80)

# II. Deployment
Comme on a vu, un pod est éphémère. Pour assurer que N pods ayant la meme définition soient toujours up, on a besoin d'instancier une ressource de type Deployment.

1. Instancier la stack du deployment fourni.
2. Via le dashboard ou via le CLI :
    * Trouver le deployment
    * Trouver le replicaset
    * Trouver le pod
3. Tenter de supprimer :
    * Le pod. Spoiler : il doit revenir (enfin, un clone).
    * Le replicaset. Spoiler : il doit revenir (enfin, un clone).
4. Mettre à jour l'image du pod (par exemple en allant chercher un tag différent pour votre image nginx) et redéployer.
    * Observer la rotation des pods : `k get pods`
    * Observer les replicasets : un nouveau replciaset a été crée. Un replicaset correspond à une version donnée du template du Deployment.
**Pour ceux qui sont en avance...**
5. Trouver comment faire un rollback (car vous avez déployez un bug).
6. Faire un deployment sans interruption de service (idéalement, valider avec une boucle sur un curl que tout se passe comme prévu)

# III. Service
Comme on a vu dans les slides, avec un Deployment, vous obtenez un tas de pods éphémères. Une ressource de type Service est nécessaire pour assurer un accès durable (à travers une IP fixe). On peut voir un Service comme un Load Balancer interne au cluster.

1. Instancier la stack du service fournie
2. Récupérer l'IP du service, ainsi que son nom de domaine. Noter qu'un serveur DNS interne (voir /etc/resolv.conf dans un Pod) permet de fournir aux services un nom de domaine, qui peut donc etre utilisé dans les autres applications (ex microservices).
3. Y accéder depuis un autre pod dans le cluster : `k run -ti --image=alpine alpine /bin/sh` puis curl sur le nom de domaine du Service

**Pour ceux qui sont en avance...**
5. Aller voir les iptables sur un des noeuds du cluster (Errata: vrai si on utilisait Calico, non étudié avec Flannel)
   Avec le plugin Flannel les services ne sont in fine que des règles de routage IPTables.
   A noter que avec Flannel (et Calico également ?) les communications entre pods qui ne sont pas sur un meme node passent par VxLan :
   * Pas de chiffrement pas défaut
   * Communications en UDP. Les paquets encapsulés sont potentiellement déjà en TCP si nécessaire.
   
# IV. Ingress
1. **Customiser** puis déployer la ressource de type ingress. Attention à utiliser un nom qui vous est propre pour ne pas créer une guerre civile dans la salle...

Astuce : vous pouvez personnaliser le fichier "/usr/share/nginx/html/index.html" dans le pod courant pour valider que c'est bien votre pod qui répond

**Pour ceux qui sont en avance...**
La ressource Ingress est, dans ce cluster, gerée par Traefik. Comme tout reverse proxy, il dispose de nombreux paramètres de configuration. Cette configuration peut etre injectée sous forme de "Middleware" (voir la doc en ligne de Traefik 2).
* Sécuriser l'accès avec du basic auth, via une annotation spécifique à Traefik
* Ajouter un middleware de compression

# V. Et pour ceux qui s'ennuient...
* Créer une ConfigMap, et injecter une des valeurs qu'il contient dans un pod sous forme de variable d'environnement et/ou de fichier.
* Déployer une DB (ex postgres) et appliquer une NetworkPolicy autour pour empecher les pods qui ne sont pas dans votre deployment d'y accéder. Valider :-) Exemple de NetworkPolicy ici : https://github.com/sdenel/blog/blob/master/deployment/k8s-init.yaml 
* Enlever proprement puis remettre le worker node "worker-linux-2.k8s.lab.smartwavesa.com" : cordon / drain / reboot / uncordon
