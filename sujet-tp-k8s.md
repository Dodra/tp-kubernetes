# TP - Kubernetes avec Minikube
## Déploiement, résilience, mise à l’échelle et diagnostic


**Outils :** Docker, Minikube, kubectl, navigateur ou curl

---

## 1. Contexte professionnel

Vous êtes ingénieur DevOps dans une entreprise qui souhaite déployer une application web sur Kubernetes. L’équipe veut pouvoir déclarer l’état souhaité de l’application, rétablir automatiquement certaines pannes, faire évoluer le nombre d’instances et mettre à jour l’application sans gérer manuellement chaque conteneur.

Votre mission consiste à déployer une application web dans un cluster Kubernetes local, à expérimenter ses mécanismes d’orchestration et à documenter vos observations.

### Objectifs pédagogiques

À la fin du TP, vous etes capable de :

- expliquer le rôle des principaux composants Kubernetes ;
- créer et inspecter des ressources Kubernetes à partir de manifestes YAML ;
- distinguer Pod, ReplicaSet, Deployment et Service ;
- configurer une application web et l’exposer localement ;
- expérimenter la réconciliation et le remplacement d’un Pod ;
- configurer des probes de santé et des ressources CPU/mémoire ;
- mettre à l’échelle un Deployment ;
- effectuer une mise à jour et un retour arrière ;
- diagnostiquer des erreurs courantes à l’aide de `kubectl` ;
- discuter des limites d’un cluster local par rapport à un cluster de production.

---

## 2. Environnement de travail

### 2.1 Choisir une solution locale

Utilisez l’une des solutions suivantes :

- **Minikube avec Docker comme driver** — solution recommandée pour ce TP ;

Les commandes de ce sujet sont principalement prévues pour Minikube.

### 2.2 Prérequis conseillés

- Docker installé et démarré ;
- Minikube installé ;
- `kubectl` installé ;
- une connexion Internet pour télécharger les images nécessaires.

Documentation officielle :

- Minikube : https://minikube.sigs.k8s.io/docs/start/
- kubectl : https://kubernetes.io/docs/tasks/tools/
- Kubernetes dans Docker Desktop : https://docs.docker.com/desktop/features/kubernetes/

### 2.3 Vérifier les outils

```bash
docker --version
kubectl version --client
minikube version
```

### 2.4 Démarrer le cluster

```bash
minikube start --driver=docker --cpus=2 --memory=4096
```

Vérifiez son état :

```bash
kubectl cluster-info
kubectl get nodes
kubectl get pods -A
```

Le nœud doit être à l’état `Ready`.

**Si le démarrage échoue :** vérifiez que Docker fonctionne, que le poste dispose de suffisamment de mémoire et que la virtualisation requise par votre environnement est opérationnelle.

---

## 3. Partie 1 — Explorer l’architecture Kubernetes (30 min)

### Travail à réaliser

Exécutez les commandes suivantes et interprétez leurs résultats :

```bash
kubectl get nodes -o wide
kubectl get namespaces
kubectl get pods -A
kubectl get deployments -A
kubectl describe node
```

Pour consulter les composants du système :

```bash
kubectl get pods -n kube-system
kubectl get events -A --sort-by=.metadata.creationTimestamp
```

La commande `kubectl get componentstatuses` est obsolète dans les versions récentes de Kubernetes. Utilisez plutôt les informations sur les Pods du système et les mécanismes de santé adaptés à la version du cluster.

### Questions

1. Quel est le rôle du control plane ? Le control plane est le cerveau du cluster Kubernetes, il va gérer et superviser l'ensemble du cluster.
2. Quelle différence existe-t-il entre un nœud Kubernetes et un Pod ? Un noeud est une machine physique ou virtuelle qui exécute les applications du cluster, tandis que le pod est le plus petite unité deployable de Kubernetes, il va contenir un ou plusieurs conteneurs.
3. Quel contrôleur maintient le nombre de réplicas demandé par un Deployment ? C'est ReplicaSet
4. Quelle différence existe-t-il entre un conteneur et un Pod ? Un conteneur est un environnement où on exécute une application, un pod lui peut regrouper plusieurs conteneurs qui partagent la même adresse IP
5. Quelles limites présente un cluster local à un seul nœud par rapport à un cluster de production ? Ressources limitées, ne reproduit pas les vraies conditions d'un environnement distribué, ne possède pas les mécanismes de sécurité d'un cluster de production, ne permet pas de tester la tolérance à la panne d'un noeud.

### Livrable

Schéma simplifié du cluster local :

```text
                 CONTROL PLANE
        +-----------------------------+
        | API Server                  |
        | Scheduler                   |
        | Controller Manager          |
        | etcd                        |
        +--------------+--------------+
                       |
                       v
              NŒUD MINIKUBE (Docker)
        +-----------------------------+
        | Kubelet                     |
        |                             |
        |  +---------+  +---------+   |
        |  | Pod 1   |  | Pod 2   |   |
        |  | Nginx   |  | Nginx   |   |
        |  +----+----+  +----+----+   |
        |       +----+------+         |
        |          Service            |
        +-------------+---------------+
                      |
                port-forward
                      |
                      v
              Navigateur localhost:8080
```

Le Control Plane gère le cluster. L'API Server reçoit les commandes, le Scheduler choisit où placer les Pods, le Controller Manager veille au respect de l'état souhaité et etcd stocke les informations du cluster. Le nœud Minikube héberge les Pods et le Kubelet surveille leur fonctionnement. Le Service permet d'accéder aux Pods avec une adresse stable et le port-forward permet d'ouvrir l'application depuis le navigateur.

---

## 4. Partie 2 — Déployer une application web (45 min)

### Objectif

Déployer Nginx avec deux réplicas, une page HTML personnalisée et un Service Kubernetes.

### 4.1 Créer l’arborescence

Créez le répertoire suivant :

```text
tp-kubernetes/
└── k8s/
    ├── 01-namespace.yaml
    ├── 02-configmap.yaml
    ├── 03-deployment.yaml
    └── 04-service.yaml
```

Placez-vous dans `tp-kubernetes` pour exécuter les commandes suivantes.

### 4.2 Créer un namespace

Créez `k8s/01-namespace.yaml` :

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: tp-k8s
```

Appliquez le manifeste :

```bash
kubectl apply -f k8s/01-namespace.yaml
kubectl get namespaces
```

**Question 2.** Quel intérêt présente un namespace dans un environnement partagé entre plusieurs équipes ? 
Un namespace permet de séparer les ressources Kubernetes. C'est à dire que chaque équipe peut disposer de ses propres pods et services  sans mélanger ses ressources avec celles des autres équipes. Il facilite également la gestion des droits d'accès, des quotas et des limites de ressources.

### 4.3 Créer une page web avec une ConfigMap

Créez `k8s/02-configmap.yaml` :

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: page-web
  namespace: tp-k8s
data:
  index.html: |
    <!DOCTYPE html>
    <html lang="fr">
    <head>
      <meta charset="UTF-8">
      <title>TP Kubernetes</title>
    </head>
    <body>
      <h1>Application déployée sur Kubernetes</h1>
      <p> Cours sur le DevOps .</p>
    </body>
    </html>
```

Appliquez la configuration :

```bash
kubectl apply -f k8s/02-configmap.yaml
kubectl get configmap -n tp-k8s
kubectl describe configmap page-web -n tp-k8s
```

**Question 3.** Pourquoi utiliser une ConfigMap plutôt que d’intégrer directement la page dans le manifeste du Deployment ?
Une ConfigMap permet de séparer la configuration du code de l'application et du coup on peut modifier la page HTML sans devoir modifier directement le manifeste du Deployment. Ca va aussi faciliter la maintenance et la réutilisation des configs.

### 4.4 Créer le Deployment

Créez `k8s/03-deployment.yaml` :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  namespace: tp-k8s
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web
  strategy:
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: nginx
          image: nginx:stable-alpine
          ports:
            - name: http
              containerPort: 80
          resources:
            requests:
              cpu: 50m
              memory: 32Mi
            limits:
              cpu: 250m
              memory: 128Mi
          readinessProbe:
            httpGet:
              path: /
              port: http
            initialDelaySeconds: 2
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /
              port: http
            initialDelaySeconds: 5
            periodSeconds: 10
          volumeMounts:
            - name: page-web
              mountPath: /usr/share/nginx/html/index.html
              subPath: index.html
      volumes:
        - name: page-web
          configMap:
            name: page-web
```

Appliquez le manifeste et observez le résultat :

```bash
kubectl apply -f k8s/03-deployment.yaml
kubectl get deployments -n tp-k8s
kubectl get replicasets -n tp-k8s
kubectl get pods -n tp-k8s -o wide
```

Attendez que les deux Pods soient prêts.

**À retenir :** `replicas: 2` indique l’état souhaité. Le contrôleur du Deployment s’appuie sur un ReplicaSet pour maintenir le nombre de Pods demandé.

### 4.5 Créer le Service

Créez `k8s/04-service.yaml` :

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web
  namespace: tp-k8s
spec:
  type: ClusterIP
  selector:
    app: web
  ports:
    - name: http
      port: 80
      targetPort: http
```

Appliquez et inspectez le Service :

```bash
kubectl apply -f k8s/04-service.yaml
kubectl get services -n tp-k8s
kubectl get endpointslices -n tp-k8s
```

Le Service sélectionne les Pods qui possèdent le label `app: web`. Il fournit une adresse stable à l’intérieur du cluster.

### 4.6 Accéder à l’application

Dans un terminal, lancez :

```bash
kubectl port-forward -n tp-k8s service/web 8080:80
```

Laissez ce terminal ouvert et accédez à :

http://localhost:8080

Vous pouvez aussi tester depuis un second terminal :

```bash
curl -i http://localhost:8080
```

**Résultat attendu :** la page HTML personnalisée s’affiche.

### Questions

1. Quel est le rôle du Deployment par rapport au ReplicaSet ? Le Deployment crée et supervise un ReplicaSet, qui est chargé de maintenir le nombre de Pods demandé.
2. Pourquoi ne pas accéder directement aux adresses IP des Pods ? Les Pods peuvent être supprimés et recréés automatiquement donc leur adresse IP peut changer.
3. Quelle différence existe-t-il entre `ClusterIP`, `NodePort` et `LoadBalancer` ? ClusterIP expose le Service uniquement à l'intérieur du cluster, NodePort quant à lui expose le Service sur un port de chaque nœud du cluster et LoadBalancer expose le Service à l'extérieur du cluster grâce à un répartisseur de charge.
4. Quelle différence existe-t-il entre une readiness probe et une liveness probe ? Readiness probe vérifie si le conteneur est prêt à recevoir du trafic et Liveness probe vérifie si le conteneur fonctionne correctement.
5. Que peut-il se passer si une limite mémoire est trop basse ? Le conteneur peut dépasser sa limite mémoire et être arrêté par Kubernetes avec le statut OOMKilled, ce qui peut provoquer des redémarrages répétés et rendre l'application indisponible.

---

## 5. Partie 3 — Résilience et auto-rétablissement (40 min)

### 5.1 Supprimer un Pod

Listez les Pods :

```bash
kubectl get pods -n tp-k8s
```

Supprimez un Pod en remplaçant `<nom-du-pod>` par son nom réel :

```bash
kubectl delete pod <nom-du-pod> -n tp-k8s
```

Dans un autre terminal, observez les changements :

```bash
kubectl get pods -n tp-k8s -w
```

Arrêtez l’observation avec `Ctrl+C`.

**Résultat attendu :** le Deployment demande le maintien de deux réplicas. Un Pod de remplacement est créé.

### 5.2 Examiner les journaux et les événements

```bash
kubectl describe pod <nom-du-pod> -n tp-k8s
kubectl logs <nom-du-pod> -n tp-k8s
kubectl get events -n tp-k8s --sort-by=.metadata.creationTimestamp
```

Si le Pod supprimé n’existe plus, sélectionnez un Pod encore présent.

### Questions d’analyse

1. Pourquoi le Pod supprimé est-il remplacé ? Le Deployment doit maintenir 2 replicas donc quand ReplicaSet constate qu'il n'en reste qu'un il en crée un nouveau.
2. Quelle différence existe-t-il entre la suppression d’un Pod et l’arrêt d’un processus dans un conteneur ? La suppression d'un Pod entraîne sa disparition du cluster alors que l'arrêt d'un processus dans un conteneur peut entraîner le redémarrage du conteneur dans le même Pod.
3. Pourquoi un redémarrage de conteneur ne signifie-t-il pas nécessairement que le cluster entier est en panne ? Parce que Kubernetes isole les applications dans différents Pods et conteneurs.
4. Que se passerait-il si le nœud unique de Minikube devenait indisponible ? Tous les Pods exécutés sur ce noeud deviendraient indisponibles.
5. Pourquoi un cluster local à un seul nœud ne permet-il pas de démontrer la tolérance à la panne d’un nœud physique ? Parce qu'il ne possède aucun autre noeud capable de prendre le relais, donc une panne du seul nœud entraîne l'indisponibilité de toutes les applications du cluster.

---

## 6. Partie 4 — Mise à l’échelle et découverte de services (35 min)

### 6.1 Augmenter le nombre de réplicas

Passez de deux à quatre instances :

```bash
kubectl scale deployment/web --replicas=4 -n tp-k8s
kubectl get deployment web -n tp-k8s
kubectl get pods -n tp-k8s -o wide
```

Réduisez ensuite le nombre d’instances à une :

```bash
kubectl scale deployment/web --replicas=1 -n tp-k8s
kubectl get pods -n tp-k8s
```

### 6.2 Examiner les points de terminaison du Service

Repassez à quatre réplicas :

```bash
kubectl scale deployment/web --replicas=4 -n tp-k8s
kubectl get endpointslices -n tp-k8s
```

Le Service peut distribuer les connexions vers les Pods prêts. La page étant identique sur toutes les instances, cette expérience ne permet pas à elle seule de mesurer la répartition du trafic.

### Questions

1. Quelle différence existe-t-il entre la mise à l’échelle manuelle et l’autoscaling ? La mise à l'échelle manuelle consiste à modifier soi-même le nombre de réplicas à l'aide d'une commande, alors que l'autoscaling ajuste automatiquement le nombre de réplicas en fonction de critères comme l'utilisation du processeur ou de la mémoire.
2. Quel est le rôle d’un Horizontal Pod Autoscaler (HPA) ? Le HPA ajuste automatiquement le nombre de Pods d'un Deployment ou d'un autre contrôleur compatible en fonction de métriques définies.
3. Pourquoi un HPA basé sur le CPU nécessite-t-il des métriques et des ressources correctement configurées ? Le HPA a besoin de connaître l'utilisation du processeur pour savoir s'il doit augmenter ou réduire le nombre de Pods. Il faut donc que les métriques soient disponibles et que les ressources CPU soient correctement définies.
4. Pourquoi quatre réplicas sur un seul nœud ne garantissent-ils pas la disponibilité en cas de panne de ce nœud ? Parce que les quatre Pods sont hébergés sur le même nœud. Si celui-ci tombe en panne, les quatre deviennent indisponibles en même temps.

---

## 7. Partie 5 — Mise à jour et retour arrière (35 min)

### 7.1 Modifier la page web

Modifiez le titre dans `k8s/02-configmap.yaml` :

```html
<h1>Version 2 — Application Kubernetes</h1>
```

Réappliquez la ConfigMap :

```bash
kubectl apply -f k8s/02-configmap.yaml
```

Le fichier est monté avec `subPath`. Dans cette configuration, les modifications de la ConfigMap ne sont pas automatiquement répercutées dans le fichier monté. Redémarrez donc le Deployment :

```bash
kubectl rollout restart deployment/web -n tp-k8s
kubectl rollout status deployment/web -n tp-k8s
```

Rechargez http://localhost:8080.

### 7.2 Examiner l’historique du Deployment

```bash
kubectl rollout history deployment/web -n tp-k8s
kubectl describe deployment web -n tp-k8s
```

### 7.3 Expérimenter une mise à jour d’image

Modifiez l’image du conteneur :

```bash
kubectl set image deployment/web nginx=nginx:alpine -n tp-k8s
kubectl rollout status deployment/web -n tp-k8s
```

Observez le remplacement progressif des Pods :

```bash
kubectl get pods -n tp-k8s -w
```

Pour revenir à la révision précédente du Deployment :

```bash
kubectl rollout undo deployment/web -n tp-k8s
kubectl rollout status deployment/web -n tp-k8s
```

**Attention :** les tags d’images tels que `alpine` sont mutables. En production, il faut contrôler les versions utilisées et envisager l’identification des images par digest pour renforcer la reproductibilité.

### Questions

1. Que signifie la stratégie `RollingUpdate` ? C'est une stratégie qui permet de remplacer progressivement les anciens Pods par les nouveaux sans arrêter toute l'application en même temps.
2. Pourquoi une mise à jour progressive ne garantit-elle pas, à elle seule, une disponibilité parfaite ? Parce que les nouveaux Pods peuvent rencontrer des problèmes au démarrage ou ne pas fonctionner correctement. Il peut donc quand même y avoir une interruption du service.
3. Dans quelles conditions `rollout undo` fonctionne-t-il ? Il faut qu'une ancienne version du Deployment soit encore disponible dans son historique. La commande permet alors de revenir à cette version précédente.
4. Pourquoi faut-il versionner les configurations et planifier les migrations de base de données séparément ? Cela permet de retrouver une ancienne configuration en cas de problème. Pour les bases de données, un retour arrière de l'application ne restaure pas automatiquement l'ancienne structure de la base, ce qui peut provoquer des problèmes de compatibilité.

---

## 8. Partie 6 — Diagnostic et dépannage (30 min)

Pour chaque incident, suivez une démarche structurée : **symptôme → hypothèse → diagnostic → correction → vérification**.

### Incident A — Image introuvable

Modifiez temporairement l’image du Deployment pour utiliser un nom d’image inexistant.

Inspectez les Pods :

```bash
kubectl get pods -n tp-k8s
kubectl describe pod <nom-du-pod> -n tp-k8s
kubectl get events -n tp-k8s --sort-by=.metadata.creationTimestamp
```

**Questions :**

- Quel état observez-vous ? Le Pod peut être en état `ErrImagePull` ou `ImagePullBackOff`, ce qui signifie que Kubernetes n'arrive pas à télécharger l'image.
- Quel événement indique la cause probable ? L'événement `Failed to pull image` indique que Kubernetes n'a pas réussi à récupérer l'image demandée.
- Comment corriger le manifeste et vérifier le retour à la normale ? Il faut corriger le nom de l'image dans le fichier YAML puis réappliquer le Deployment avec `kubectl apply`. On peut ensuite vérifier que les Pods sont bien en état `Running`.

### Incident B — Service inaccessible

Modifiez temporairement le sélecteur du Service pour qu’il ne corresponde plus aux labels des Pods.

Inspectez les points de terminaison :

```bash
kubectl get services -n tp-k8s
kubectl get endpointslices -n tp-k8s
```

**Questions :**

- Pourquoi le Service n’a-t-il plus de destination exploitable ? Parce que le sélecteur du Service ne correspond plus aux labels des Pods, il ne trouve donc plus de Pods vers lesquels envoyer les requêtes.
- Pourquoi les Pods peuvent-ils être opérationnels alors que l’application est inaccessible via le Service ? Parce que les Pods fonctionnent toujours normalement, mais le Service ne sait plus vers lesquels rediriger les connexions.
- Quelle correction permet de rétablir le routage ? Il faut remettre le bon sélecteur dans le Service pour qu'il corresponde aux labels des Pods, puis réappliquer le manifeste.

### Incident C — Sonde de santé incorrecte

Configurez temporairement une route HTTP inexistante dans la readiness probe, puis observez :

```bash
kubectl describe pod <nom-du-pod> -n tp-k8s
kubectl get pods -n tp-k8s
kubectl get endpointslices -n tp-k8s
```

**Questions :**

- Quel est l’effet d’une readiness probe en échec ? Le Pod est considéré comme non prêt et ne reçoit donc plus de trafic via le Service, même si son conteneur continue de fonctionner.
- En quoi son effet diffère-t-il de celui d’une liveness probe en échec ? La readiness probe empêche simplement le Pod de recevoir du trafic, alors que la liveness probe peut provoquer le redémarrage du conteneur.
- Pourquoi une sonde mal configurée peut-elle provoquer une indisponibilité ? Parce qu'elle peut considérer un Pod fonctionnel comme défaillant. Si tous les Pods sont concernés, aucun ne pourra recevoir de trafic et l'application deviendra inaccessible.

**Consigne :** restaurez tous les manifestes à un état fonctionnel après les expériences.

---

## 9. Mini-projet de synthèse

Faites évoluer le déploiement afin de satisfaire les exigences suivantes :

- [ ] Déployer une application avec au moins deux réplicas.
- [ ] Créer un namespace et un Service dédiés.
- [ ] Configurer des probes de santé.
- [ ] Définir des requests et des limits CPU/mémoire.
- [ ] Démontrer le remplacement automatique d’un Pod supprimé.
- [ ] Réaliser une mise à l’échelle manuelle.
- [ ] Effectuer une mise à jour puis un retour arrière.
- [ ] Provoquer et diagnostiquer un incident.
- [ ] Documenter les commandes et les observations.
- [ ] Justifier les choix d’architecture et leurs limites.

### Livrables attendus

1. Le répertoire `k8s/` contenant les manifestes YAML.
2. Une analyse des limites d’un cluster local par rapport à un cluster cloud de production.

Un cluster local comme Minikube permet de tester Kubernetes facilement, mais il possède certaines limites. Il utilise généralement un seul nœud et dispose de ressources limitées. Il ne permet donc pas de tester correctement la tolérance aux pannes ou la répartition des applications sur plusieurs machines.

Un cluster cloud de production possède généralement plusieurs nœuds, davantage de ressources et des mécanismes de haute disponibilité. Il offre aussi des possibilités supplémentaires pour gérer la sécurité, le stockage et le trafic réseau.

---

## 12. Conclusion attendue

Répondez à la question suivante :

 > En quoi Kubernetes apporte-t-il des mécanismes d’orchestration qui dépassent la simple exécution de conteneurs avec Docker, et quelles précautions restent nécessaires pour exploiter une application de manière fiable en production ?

Docker permet de créer et d'exécuter des conteneurs, tandis que Kubernetes permet de les gérer automatiquement. Il permet notamment de remplacer les Pods en cas de problème, de gérer le nombre de réplicas, de faire des mises à jour et de répartir les connexions entre les différentes instances.

Pour une utilisation en production, il faut quand même faire attention à la sécurité, aux ressources disponibles, aux sauvegardes et à la surveillance des applications. Il faut également prévoir plusieurs nœuds pour éviter qu'une seule panne rende toute l'application indisponible.


## 13. Nettoyage de l’environnement

Supprimez les ressources du TP :

```bash
kubectl delete namespace tp-k8s
```

Si vous utilisez Minikube et souhaitez simplement arrêter le cluster :

```bash
minikube stop
```

Pour supprimer entièrement le cluster local et ses données :

```bash
minikube delete
```

**Attention :** `minikube delete` supprime le cluster concerné. Ne l’exécutez pas si vous y avez conservé d’autres travaux.
