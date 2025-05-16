# Deploy Web Sederhana dengan repo github ke ArgoCD [Bahasa Indonesia]

## Buat File YAML dan Halaman web dengan struktur direktori sebagai berikut:

- Direktori pada repo Github
```
repo-github/
├─ manifest/
│  ├─ app/
│  │  ├─ index.html
│  ├─ deployment.yaml
│  ├─ kustomization.yaml
```
- Direktori pada local PC (File yaml ini untuk dideploy di openshift, best practice untuk TIDAK di upload ke repository)
```
local-direktory/
├─ application.yaml
├─ secret.yaml # Apabila repository bersifat private
```
## Isi tiap file yaml
### deployment.yaml
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-website #nama aplikasi bebeas
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-website  #nama aplikasi bebeas
  template:
    metadata:
      labels:
        app: my-website  #nama aplikasi bebeas
    spec:
      containers:
      - name: nginx
        image: nginxinc/nginx-unprivileged:alpine #menggunakan web service nginx, di sesuaikan kebutuhan
        ports:
        - containerPort: 8080  # nginx-unprivileged listen di port 8080
        volumeMounts:
        - name: web-content
          mountPath: /usr/share/nginx/html
      volumes:
      - name: web-content
        configMap:
          name: my-website-html
---
apiVersion: v1
kind: Service
metadata:
  name: my-website
spec:
  selector:
    app: my-website
  ports:
  - port: 80
    targetPort: 8080  # arahkan ke port 8080 container
```

### kustomization.yaml
Untuk resource website, usahakan terletak pada satu direktori dengan deployment.yaml dan kustomization.yaml
apabila ingin menambah banyak file dan direktori maka di define satu persatu untuk mempercepat bisa menggunakan command `find /`
atau bisa push ke container akan dijelaskan pada next tutorial
```
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- deployment.yaml # resource deployment
configMapGenerator:
- name: my-website-html
  files: # resource website
  - app/index.html
```

### application.yaml
```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-website # nama aplikasi bebas
  namespace: openshift-gitops # namespace argoCD
spec:
  source:
    directory:
      recurse: false
    repoURL: https://github.com/earthbendergara/simple-web #Repo githubmu
    path: site  # Path ke manifest aplikasi
    targetRevision: main
  project: default
  destination:
    namespace: demo # namespace aplikasi pada openshift
    name: openshift-prod # Cluster ArgoCD
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

### secret.yaml
ini optional, apabila repo mu bersifat private hukumnya wajib

```
apiVersion: v1
kind: Secret
metadata:
  name: my-private-repo-secret # nama secret bebas
  namespace: openshift-gitops # namespace ArgoCD
  labels:
    argocd.argoproj.io/secret-type: repository
stringData:
  url: https://github.com/earthbendergara/simple-web #Repo githubmu
  username: <username-github>
  password: <personal-access-token>
type: Opaque
```

### index.html
sesuaikan kebutuhan webmu 
```
<html>
<body>
<h1>It Works</h1>
</body>
</html>
```

## Langkah deployment
### Deploy application.yaml

```
oc apply -f application.yaml
```

### Expose service
```
oc expose service my-website -n demo
```
cek apakah URL dari route sudah dibuat
```
oc get route my-website -n demo
```
output seharusnya seperti ini
```
NAME          HOST/PORT                                      PATH   SERVICES      PORT   TERMINATION   WILDCARD
my-website    my-website-demo.apps.cluster.example.com              my-website    8080                    None
```
### Buka web pada browser

```
http://my-website-demo.apps.cluster.example.com
```

## Apabila route tidak otomatis muncul
### Buat file route.yaml
file ini TIDAK untuk diupload di Repositori, terletak satu direktori dengan application.yaml dan secret.yaml
```
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: my-website
  namespace: demo
spec:
  to:
    kind: Service
    name: my-website
  port:
    targetPort: 80
  tls:
    termination: edge
```

### Deploy route.yaml
```
oc apply -f route.yaml
```
### Cek apakah sudah keluar route

```
oc get route my-website -n demo
```

## Apabila Openshift terletak pada jaringan local jangan lupa menambahkan ke /etc/hosts
### tambahkan ke file /etc/hosts

```
sudo nano /etc/hosts
```

IP address sesuaikan dengan IP Route Openshift
```
...
10.10.10.10  my-website-demo.apps.cluster.example.com
...
```

## Maintanence website
Setiap ada perubahan pada repositori websitemu, akan otomatis dilakukan synchronize oleh ArgoCD tiap 3 Menit


