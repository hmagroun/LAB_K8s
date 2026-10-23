
# Lab Pratique : Gestion des ConfigMaps, Secrets et Déploiement Nginx

## Objectifs du Lab

A la fin de ce lab, vous serez capable de :

1. Creer et manipuler des Secrets Kubernetes et comprendre leur encodage Base64.
2. Creer des ConfigMaps a partir de fichiers d'environnement et de fichiers de configuration.
3. Injecter des variables et monter des fichiers de configuration dans un Deployment.
4. Regrouper l'ensemble de la stack dans un fichier manifeste unique.
5. Gérer la mise à jour dynamique des ConfigMaps et le rechargement de configuration dans Kubernetes.

---

## Etape 1 : Créer et analyser un Secret Kubernetes

1. Generer le fichier manifeste pour un secret generique nomme `secret-1` contenant un utilisateur (`ali`) et un mot de passe (`123`) :

```bash
kubectl create secret generic secret-1 --from-literal=user=ali --from-literal=password=123 --dry-run=client -o yaml > secret-1.yaml

```

2. Appliquer le secret sur le cluster :

```bash
kubectl create -f secret-1.yaml

```

3. Verifier la creation et inspecter le secret :

```bash
kubectl get secrets
kubectl describe secret secret-1

```

4. Afficher le secret au format YAML pour observer l'encodage Base64 :

```bash
kubectl get secrets secret-1 -o yaml

```

5. Decoder les valeurs en ligne de commande pour retrouver les donnees en clair :

```bash
echo -n MTIz | base64 -d
echo -n YWxp | base64 -d

```

---

## Etape 2 : Créer un ConfigMap à partir d'un fichier d'environnement

1. Creer un fichier local contenant des parametres sous forme de paires cle-valeur :

```bash
nano my-env-file

```

Contenu du fichier `my-env-file` :

```env
param1=valeur1
param2=valeur2

```

2. Generer le fichier manifeste du ConfigMap `cm-param` a partir de ce fichier :

```bash
kubectl create cm cm-param --from-env-file=my-env-file --dry-run=client -o yaml > cm-param.yaml

```

3. Creer le ConfigMap dans le cluster et verifier :

```bash
kubectl create -f cm-param.yaml
kubectl get cm
kubectl describe cm cm-param

```

---

## Etape 3 : Créer un ConfigMap pour la configuration Nginx

1. Creer un fichier de configuration Nginx personnalise :

```bash
nano my-nginx-config

```

Contenu du fichier `my-nginx-config` :

```nginx
user  nginx;
worker_processes  auto;

error_log  /var/log/nginx/error.log notice;
pid        /run/nginx.pid;

events {
    worker_connections  1030;
}

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile        on;
    keepalive_timeout  65;
    gzip  on;

    include /etc/nginx/conf.d/*.conf;
}

```

2. Generer le manifeste du ConfigMap `cm-nginx-config` associant le fichier a la cle `nginx.conf` :

```bash
kubectl create cm cm-nginx-config --from-file=nginx.conf=./my-nginx-config --dry-run=client -o yaml > cm-nginx-config.yaml

```

3. Appliquer et verifier :

```bash
kubectl create -f cm-nginx-config.yaml
kubectl describe cm cm-nginx-config

```

---

## Etape 4 : Créer et exécuter le Déploiement

1. Generer le squelette du deploiement :

```bash
kubectl create deploy my-app --image=nginx --dry-run=client -o yaml > my-app.yaml

```

2. Editer le fichier `my-app.yaml` pour associer les variables d'environnement et le volume de configuration :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: my-app
  name: my-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - image: nginx
        name: nginx
        env:
        - name: MY_VAR
          value: "xyz"
        - name: PARAM1
          valueFrom:
            configMapKeyRef:
              name: cm-param
              key: param1
        - name: USER
          valueFrom:
            secretKeyRef:
              name: secret-1
              key: user
        - name: PASSWORD
          valueFrom:
            secretKeyRef:
              name: secret-1
              key: password
        volumeMounts:
        - name: volume-config
          mountPath: /etc/nginx/nginx.conf
          subPath: nginx.conf
      volumes:
      - name: volume-config
        configMap:
          name: cm-nginx-config

```

3. Deployer sur le cluster et verifier le statut :

```bash
kubectl create -f my-app.yaml
kubectl get pods

```

4. Executer les commandes de validation a l'interieur du conteneur :

```bash
kubectl exec -it deployment/my-app -- bash

```

Une fois dans le conteneur :

```bash
env | grep PARAM
env | grep USER
grep worker_connections /etc/nginx/nginx.conf
grep gzip /etc/nginx/nginx.conf
exit

```

---

## Etape 5 : Automatiser le déploiement via un fichier unique

Au lieu de gerer plusieurs fichiers manifestes indepedants, il est possible de regrouper l'ensemble des objets Kubernetes dans un seul fichier YAML en se servant du separateur `---`.

1. Creer un fichier nomme `all-in-one.yaml` :

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: secret-1
type: Opaque
stringData:
  user: ali
  password: "123"
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: cm-param
data:
  param1: valeur1
  param2: valeur2
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: cm-nginx-config
data:
  nginx.conf: |
    user  nginx;
    worker_processes  auto;
    error_log  /var/log/nginx/error.log notice;
    pid        /run/nginx.pid;
    events {
        worker_connections  1030;
    }
    http {
        include       /etc/nginx/mime.types;
        default_type  application/octet-stream;
        log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                          '$status $body_bytes_sent "$http_referer" '
                          '"$http_user_agent" "$http_x_forwarded_for"';
        access_log  /var/log/nginx/access.log  main;
        sendfile        on;
        keepalive_timeout  65;
        gzip  on;
        include /etc/nginx/conf.d/*.conf;
    }
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: my-app
  name: my-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - image: nginx
        name: nginx
        env:
        - name: MY_VAR
          value: "xyz"
        - name: PARAM1
          valueFrom:
            configMapKeyRef:
              name: cm-param
              key: param1
        - name: USER
          valueFrom:
            secretKeyRef:
              name: secret-1
              key: user
        - name: PASSWORD
          valueFrom:
            secretKeyRef:
              name: secret-1
              key: password
        volumeMounts:
        - name: volume-config
          mountPath: /etc/nginx/nginx.conf
          subPath: nginx.conf
      volumes:
      - name: volume-config
        configMap:
          name: cm-nginx-config

```

2. Nettoyer les ressources precedentes puis tout redeployer en une seule commande :

```bash
kubectl delete deploy my-app
kubectl delete cm cm-param cm-nginx-config
kubectl delete secret secret-1

kubectl apply -f all-in-one.yaml

```

---

## Etape 6 : Gérer la mise à jour dynamique d'un ConfigMap

La modification d'un ConfigMap existant necessite de comprendre comment Kubernetes et les applications reagissent aux changements.

1. Modifier le ConfigMap `cm-param` pour changer la valeur de `param1` :

```bash
kubectl edit cm cm-param

```

Remplacez `valeur1` par `nouvelle-valeur`.

2. Verifier l'impact sur le Pod en cours d'execution :

```bash
kubectl exec -it deployment/my-app -- env | grep PARAM

```

Constat : La variable d'environnement `PARAM1` contient toujours `valeur1`. Les variables d'environnement ne sont lues qu'au lancement du conteneur.

3. Modifier le ConfigMap `cm-nginx-config` :

```bash
kubectl edit cm cm-nginx-config

```

Modifiez la ligne `worker_connections 1030;` par `worker_connections 2048;`.

4. Comprendre le comportement du volume :
Lorsqu'un ConfigMap est monte en volume, le fichier dans le pod est mis a jour de maniere asynchrone par le kubelet. Cependant, le processus Nginx ne recharge pas automatiquement son fichier de configuration en memoire.
5. Appliquer les changements aux processus :

Pour recharger la configuration Nginx sans interrompre le pod :

```bash
kubectl exec -it deployment/my-app -- nginx -s reload

```

Pour recharger l'ensemble des variables d'environnement et de la configuration via un redemarrage progressif (Rolling Restart) du Deployment :

```bash
kubectl rollout restart deployment/my-app

```

6. Verifier que le nouveau pod a pris en compte les modifications :

```bash
kubectl exec -it deployment/my-app -- env | grep PARAM

```
