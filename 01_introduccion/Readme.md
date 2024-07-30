# Guia helm

# Pre Requisitos
## Instalar Docker
```sh
# https://docs.docker.com/engine/install/ubuntu/
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
```

## Docker Post Install
```sh
# https://docs.docker.com/engine/install/linux-postinstall/
sudo groupadd docker
sudo usermod -aG docker $USER
```

## Kubectl
```sh
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
chmod +x kubectl
mkdir -p ~/.local/bin
mv ./kubectl ~/.local/bin/kubectl
```

## Minikube: 
```sh
# https://minikube.sigs.k8s.io/docs/start/?arch=%2Flinux%2Fx86-64%2Fstable%2Fbinary+download
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube && rm minikube-linux-amd64
```

# Instalar helm
```sh
# https://helm.sh/docs/intro/install/
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
sudo apt-get install apt-transport-https --yes
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm
```

# Buscar charts
Repositorio de charts: 
- https://artifacthub.io/ 
- https://artifacthub.io/packages/search

# Desplegar chart
Vamos a desplegar kuberentes-dashboard https://artifacthub.io/packages/helm/k8s-dashboard/kubernetes-dashboard
```sh
helm repo add kubernetes-dashboard https://kubernetes.github.io/dashboard/

helm upgrade --install kubernetes-dashboard \
  kubernetes-dashboard/kubernetes-dashboard \
  --create-namespace \
  --namespace kubernetes-dashboard
```

Configurar Kubernetes Dashboard
```yml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
```

Ingresar a Kuberentes Dashboard
```sh
kubectl apply -f cfg.yml
kubectl -n kubernetes-dashboard create token admin-user
kubectl -n kubernetes-dashboard port-forward svc/kubernetes-dashboard-kong-proxy 8080:443
```
Abrir en el navegador https://localhost:8080

# Crear chart
```sh
helm create hellochart
```
# Estructura
Esto generará un directorio llamado hellochart con la siguiente estructura:
```sh
hellochart/
  charts/             # Dependencias del chart
  templates/          # Plantillas de Kubernetes que se expanden con los valores
  Chart.yaml          # Información sobre el chart
  values.yaml         # Valores por defecto para los templates
```

## Metadatos
El archivo `Chart.yaml` contiene **metadatos del chart**
```yml
apiVersion: v2
type: application

name: hellochart
description: A Helm chart for Kubernetes

version: 0.1.0
appVersion: "1.0.0"
```

## Valores por defecto
El archivo `values.yaml` contiene **valores por defecto del chart**:
```yml
replicaCount: 1

ingress:
  host: hello-app.example.com
```

## Template
En el directorio `templates`, agregamos los **manifiestos de kubernetes con variables**

`deployment.yml`
```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}
  namespace: {{ .Release.Namespace }}
  labels:
    app: {{ .Release.Name }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Release.Name }}
  template:
    metadata:
      labels:
        app: {{ .Release.Name }}
    spec:
      containers:
        - name: hello-app
          image: gcr.io/google-samples/hello-app:1.0
          ports:
            - containerPort: 8080
```

`service.yml`
```yml
apiVersion: v1
kind: Service
metadata:
  name: {{.Release.Name}}-svc
  namespace: {{ .Release.Namespace }}
spec:
  selector:
    app: {{.Release.Name}}
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
```

`ingress.yml`
```yml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ .Release.Name }}-ingress
  namespace: {{ .Release.Namespace }}
spec:
  rules:
    - host: {{ .Values.ingress.host }}
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: {{ .Release.Name }}-svc
                port:
                  number: 8080
```

## Probar
Para probar el chart localmente
```sh
helm template hellochart
```
Para instalar el chart en tu cluster de Kubernetes:
```sh
helm install my-first-chart ./hellochart
```

# Publicar chart
Para publicarlo, podemos usar un repositorio OCI (Open Container Initiative) como DockerHub https://helm.sh/docs/topics/registries/

```sh
helm package hellochart
docker login
helm push hellochart-0.1.0.tgz oci://registry-1.docker.io/caosbinario
```