# 🏗️ CI/CD Lab: Jenkins + Kubernetes (K3s) en Proxmox

Pipeline automatizada que despliega una web en Kubernetes cada vez que se lanza una build en Jenkins. Todo corriendo sobre Proxmox en un miniPC.

---

## 🧅 Arquitectura

```
Proxmox VE (Hipervisor)
└── VM Ubuntu ──────────── Clúster K3s
    └── LXC Ubuntu
        └── Docker
            └── Jenkins
```

---

## ⚙️ Stack

| Tecnología | Versión | Rol |
|---|---|---|
| Proxmox VE | 8.x | Hipervisor |
| K3s | latest | Kubernetes ligero |
| Jenkins | LTS | Motor CI/CD |
| Docker | latest | Runtime de Jenkins |
| Nginx | latest | Servidor web desplegado |

---

## 🚀 Puesta en marcha

### 1. Instalar K3s (en la VM)

```bash
curl -sfL https://get.k3s.io | sh -
```

Extraer el kubeconfig y cambiar `127.0.0.1` por la IP real del servidor:

```bash
cat /etc/rancher/k3s/k3s.yaml
```

### 2. Arrancar Jenkins (en el LXC)

```bash
docker exec jenkins-master cat /var/jenkins_home/secrets/initialAdminPassword
```

Accede en `http://<IP-LXC>:8080` e instala los plugins sugeridos.

### 3. Conectar Jenkins con K3s

Desde la consola del LXC (fuera de Docker):

```bash
docker cp /usr/local/bin/kubectl jenkins-master:/usr/local/bin/kubectl
docker exec -u root jenkins-master chmod +x /usr/local/bin/kubectl
docker exec -u root jenkins-master mkdir -p /var/jenkins_home/.kube
docker cp /root/.kube/config jenkins-master:/var/jenkins_home/.kube/config
docker exec -u root jenkins-master chown -R jenkins:jenkins /var/jenkins_home/.kube
```

### 4. Crear la Pipeline en Jenkins

Nueva tarea → **Pipeline** → pega el contenido de [`Jenkinsfile`](./Jenkinsfile).

---

## 📄 Jenkinsfile

El pipeline aplica tres manifiestos de Kubernetes en un solo paso:

- **ConfigMap** — inyecta el HTML personalizado
- **Deployment** — 4 réplicas de Nginx sirviendo ese HTML
- **Service** — NodePort expuesto en el puerto `32000`

```groovy
pipeline {
    agent any
    stages {
        stage('Deploy') {
            steps {
                sh '''
                cat <<EOF | kubectl apply --kubeconfig=/var/jenkins_home/.kube/config -f -
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: mi-web-html
data:
  index.html: |
    <!DOCTYPE html>
    <html>
    <body style="background-color:#2b303b;color:#42b983;text-align:center;padding:100px;font-family:sans-serif;">
      <h1>🚀 Desplegado con Jenkins + K3s</h1>
    </body>
    </html>
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mi-web-pro
spec:
  replicas: 4
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
        volumeMounts:
        - name: html
          mountPath: /usr/share/nginx/html/index.html
          subPath: index.html
      volumes:
      - name: html
        configMap:
          name: mi-web-html
---
apiVersion: v1
kind: Service
metadata:
  name: mi-web-pro-svc
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
      nodePort: 32000
EOF
                '''
            }
        }
    }
}
```

---

## ✅ Verificación

Desde la VM de K3s, tras ejecutar la build:

```bash
sudo kubectl get pods
# NAME                          READY   STATUS    RESTARTS   AGE
# mi-web-pro-xxxxxxxxx-xxxxx    1/1     Running   0          30s
# (x4 réplicas)
```

La web queda disponible en:

```
http://<IP-VM-K3S>:32000
```

---

## 📁 Estructura del repositorio

```
.
├── Jenkinsfile          # Pipeline declarativa
├── manifests/
│   ├── configmap.yaml
│   ├── deployment.yaml
│   └── service.yaml
└── README.md
```

---

## 📖 Referencias

- [K3s Docs](https://docs.k3s.io/)
- [Jenkins Pipeline Syntax](https://www.jenkins.io/doc/book/pipeline/syntax/)
- [Proxmox VE](https://www.proxmox.com/en/proxmox-virtual-environment/overview)
