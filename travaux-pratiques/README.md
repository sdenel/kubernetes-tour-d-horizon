# Contexte
L'objectif de ce TP est de comprendre l'interet des ressources Kubernetes de type Pod, Deployment, Service, et Ingress.
* Vous avez un cluster à votre disposition. Accès derrière les IPs Richemont exclusivement (me faire signe pour ouvrir plus large)
* Liens utiles :
    * Dashboard Kubernetes : https://dashboard.k8s.lab.smartwavesa.com
    * Terminal de la workstation : https://workstation-term.k8s.lab.smartwavesa.com ou en ssh : prenom:prenom@workstation.k8s.lab.smartwavesa.com
    * Vous pouvez également récupérer le fichier de config de kubectl (~/.kube/config) et l'utiliser en local. Remplacer l'URL de l'APIServer par "master.k8s.lab.smartwavesa.com"

# I. Pod
1. Instancier un pod dans votre namespace avec la commande : `k apply -f votrefichier.yaml`
   Le fichier "1-pods.yaml" peut etre utilisé tel quel. Vous pouvez également ajouter à ce pod des labels / annotations.
2. En binome :
    * **Tata** :
        * récupérer L'IP de votre pod :
            * Via le dashboard pour les petits joueurs
            * Via le CLI + jq pour les vrais : `k get pods mon-premier-pod -o json | jq -r '.status.podIP'
        * La fournir à votre camarade. Normalement, vous exposez en HTTP sur le port 80.
    * **Toto** :
        * Se placer dans votre pod : `k exec -ti mon-premier-prod /bin/sh`
        * Installer curl (apk curl) puis récupérer le contenu exposé par le pod de votre voisin
   On vient de montrer que par défaut, les pods du namespace de votre binome ont le droit d'accéder aux pods dans votre namespace, et que les pods munis d'un terminal (non distroless) sont pratiques pour un éventuel pirate !
3. Supprimer votre pod pour simuler une erreur : il ne revient pas ! Un pod doit etre vu comme un objet immuable et éphémère.

**Pour ceux qui sont en avance (ou qui veulent rouler des mécaniques)...**
1. Utiliser l'image sdenel/hello-world :
    * Retenter `k exec -ti mon-premier-prod /bin/sh`
    * Comme vous le remarquez, ce n'est pas possible ! l'image ne contient que l'executable nginx et ses dépendances (sans bash ni sh ni aucun gestionnaire de paquet). Elle est distroless. Voir les sources : https://github.com/sdenel/docker-nginx-file-listing
2. Ajouter une livenessProbe au pod (qui expose sur le pod 80)

# II. Deployment
Comme on a vu, un pod est éphemere. Pour assurer que N pods ayant la meme définition soient toujours up, on a besoin d'instancier une ressource de type Deployment.

1. Instancier la stack du deployment fourni.
2. Via le dashboard ou via le CLI :
    * Trouver le deployment
    * Trouver le replicaset
    * Trouver le pod
3. Tenter de supprimer :
    * Le pod. Spoiler : il doit revenir.
    * Le replicaset. Spoiler : il doit revenir.
4. Mettre à jour l'image du pod et redéployer.
    * Observer la rotation des pods : `k get pods`
    * Observer les replicasets : un nouveau replciaset a été crée. Un replicaset correspond à une version donnée du template du Deployment.
**Pour ceux qui sont en avance...**
5. Trouver comment faire un rollback (car vous avez déployez un bug).
6. Faire un deployment progressif

# III. Service
Comme on a vu (fallait suivre !) du haut d'un Deployment, vous obtenez un tas de pods ephemeres, sans moyen de les joindre facilement. Une ressource de type Service est nécessaire pour ça.

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
   
# IV. Ingress
1. **Customiser** puis déployer la ressource de type ingress. Attention à utiliser un nom qui vous est propre pour ne pas créer une guerre civile dans la salle...

**Pour ceux qui sont en avance...**
La ressource Ingress est, dans ce cluster, gerée par Traefik. Comme tout reverse proxy, il dispose de nombreux paramètres de configuration.
* Sécuriser l'accès avec du basic auth, via une annotation spécifique à Traefik
* Ou (plus simple) : bloquer l'accès à certaines IPs seulement (demander à Simon pour ouvrir temporairement le security group de manière plus large pour valider)
## Un peu de doc
### Sécurisation avec user/password (basic auth) :

```bash
htpasswd -c unfichierquivacontenirusrpwdhashe unutilisateur
kubectl create secret generic unsecret --from-file=unfichierquivacontenirusrpwdhashe
```

Puis ajouter comme annotations à l'ingress :

```yaml
annotations:
  ingress.kubernetes.io/auth-type: basic
  ingress.kubernetes.io/auth-secret: unsecret
  traefik.frontend.rule.type: PathPrefixStrip
```

Voir également : <https://docs.traefik.io/configuration/entrypoints/#basic-authentication>

### Filtrage IP :

Ajouter les annotations suivantes à l'Ingress:

```yaml
annotations:
  traefik.ingress.kubernetes.io/whitelist-source-range: "1.2.3.4/32"
  ingress.kubernetes.io/whitelist-x-forwarded-for: "true"
```




# V. Et pour ceux qui s'ennuient...
* Créer une ConfigMap, et l'injecter une des valeurs qu'il contient dans un pod sous forme de variable d'environnement et/ou de fichier.
* Déployer une DB (ex postgres) et appliquer une NetworkPolicy autour pour empecher les pods qui ne sont pas dans votre deployment d'y accéder. Valider :-) Exemple de NetworkPolicy ici : https://github.com/sdenel/blog/blob/master/deployment/k8s-init.yaml 
* Enlever proprement puis remettre le worker node "worker-linux-3.k8s.lab.smartwavesa.com" (terminal : https://worker-3-term.k8s.lab.smartwavesa.com/) un noeud du cluster : cordon / drain / reboot / uncordon
