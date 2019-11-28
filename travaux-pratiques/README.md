# Contexte
L'objectif de ce TP est de comprendre l'interet des ressources Kubernetes de type Pod, Deployment, Service, et Ingress.
* Vous avez un cluster à votre disposition. Accès derrière les IPs Richemont exclusivement (me faire signe pour ouvrir plus large)
* Liens utiles :
* Dashboard : https://dashboard.k8s.lab.smartwavesa.com
* Terminal de la workstation : https://workstation-term.k8s.lab.smartwavesa.com ou en ssh : prenom:prenom@workstation.k8s.lab.smartwavesa.com
* Vous pouvez également récupérer le fichier de config de kubectl (~/.kube/config) et l'utiliser en local. Remplacer l'URL de l'APIServer par "master.k8s.lab.smartwavesa.com"

# Pod
1. Instancier ce pod dans votre namespace avec la commande : `k apply -f votrefichier.yaml`
2. En binome :
    * Récupérer L'IP de votre pod :
        * Via le dashboard pour les petits joueurs
        * Via le CLI + jq pour les vrais : `k get pods mon-premier-pod -o json | jq -r '.status.podIP'
    * Se placer dans le pod : `k exec -ti mon-premier-prod /bin/sh`
    * Installer curl (apk curl) puis récupérer le contenu exposé par le pod de votre voisin
   On vient de montrer que par défaut, les pods du namespace de votre binome ont le droit d'accéder aux pods dans votre namespace, et que les pods munis d'un terminal (non distroless) sont pratiques pour un éventuel pirate !
3. Supprimer votre pod pour simuler une erreur : il ne revient pas ! Un pod est immuable et éphémère.

**Pour ceux qui sont en avance...**
4. Utiliser l'image sdenel/hello-world :
  * Retenter `k exec -ti mon-premier-prod /bin/sh`
  * Comme vous le remarquez, ce n'est pas possible ! l'image ne contient que l'executable nginx et ses dépendances (sans bash ni sh ni aucun gestionnaire de paquet). Elle est distroless. Voir les sources : https://github.com/sdenel/docker-nginx-file-listing
5. Ajouter une livenessProbe au pod (qui expose sur le pod 80)

# Deployment
1. Instancier la stack du deployment fourni.
2. Via le dashboard ou via le CLI :
    * Trouver le deployment
    * Trouver le replicaset
    * Trouver le pod
3. Tenter de supprimer :
    * Le pod. Que se passe t'il ?
    * Le replicaset : que se passe t'il ?
4. Mettre à jour l'image du pod et redéployer. Observer la rotation des pods : `k get pods`
**Pour ceux qui sont en avance...**
5. Débrouillez vous.

# Service
1. Instancier la stack du service fournie
2. Récupérer l'IP du service, ainsi que son nom de domaine
   Noter qu'un serveur DNS interne permet de fournir aux services un nom de domaine, qui peut donc etre utilisé dans les autres applications (ex microservices)
3. Y accéder depuis un autre pod du cluster : `k run -ti --image=alpine alpine /bin/sh`

**Pour ceux qui sont en avance...**
5. Aller voir les iptables sur un des noeuds du cluster (https://worker-1-term.k8s.lab.smartwavesa.com)
   Avec le plugin Calico (idem avec Flannel) les services ne sont in fine que des règles de routage IPTables.
   A noter que avec Flannel (et Calico également ?) les communications entre pods qui ne sont pas sur un meme node passent par VxLan :
   * Pas de chiffrement pas défaut
   * Communications en UDP. Les paquets encapsulés sont potentiellement déjà en TCP si nécessaire.
   
# Ingress
1. **Customiser** puis déployer la ressource de type ingress. Attention à utiliser un nom qui vous est propre pour ne pas créer une guerre civile dans la salle...

**Pour ceux qui sont en avance...**
2. Sécuriser l'accès avec du basic auth, via une annotation spécifique à Traefik


# Et pour ceux qui s'ennuient...
* Créer une ConfigMap, et l'injecter une des valeurs qu'il contient dans un pod sous forme de variable d'environnement
* Enlever proprement un noeud du cluster (avec un drain)
