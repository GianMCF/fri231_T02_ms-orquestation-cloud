# fri231_T02_ms-orquestation-cloud

# Kubernetes Cloud + Seguridad Profesional

## Descripción

Este proyecto implementa un cluster Kubernetes en AWS utilizando instancias EC2 Ubuntu 24.04, aplicando conceptos de:

- Alta disponibilidad
- Escalabilidad
- Seguridad avanzada
- RBAC
- Network Policies
- Escaneo de imágenes
- Rolling Updates
- Backups

La práctica fue desarrollada colaborativamente entre dos estudiantes:

| Rol | Responsable |
|---|---|
| Infraestructura | Estudiante A - Gianmarco Castillo Flores|
| Seguridad | Estudiante B - Alejandro O. Soto Cardenas|

---

# Arquitectura

La implementación se realizó utilizando dos instancias EC2:

| Instancia | Función | Responsable |
|---|---|---|
| EC2 MASTER | Control Plane Kubernetes | Estudiante A |
| EC2 WORKER | Nodo Worker Kubernetes | Estudiante A |
| Seguridad | RBAC + Policies + Trivy | Estudiante B |

---

# Diagrama

```text
                    ┌──────────────────────────────┐
                    │          AWS CLOUD           │
                    │                              │
                    │    ┌──────────────────┐      │
                    │    │   EC2 MASTER     │      │
                    │    │  Control Plane   │      │
                    │    │                  │      │
                    │    │ kube-apiserver   │      │
                    │    │ scheduler        │      │
                    │    │ etcd             │      │
                    │    └────────┬─────────┘      │
                    │             │                │
                    │             │ Kubernetes     │
                    │             │ Cluster        │
                    │             ▼                │
                    │    ┌──────────────────┐      │
                    │    │   EC2 WORKER     │      │
                    │    │    Worker Node   │      │
                    │    │                  │      │
                    │    │ nginx pods       │      │
                    │    │ busybox pods     │      │
                    │    └──────────────────┘      │
                    │                              │
                    └──────────────────────────────┘
```

---

# Tecnologías Utilizadas

- Kubernetes v1.30
- Ubuntu 24.04
- AWS EC2
- Calico CNI
- Trivy
- containerd
- kubeadm
- kubectl

---

# FASE 1 — CREACIÓN DEL CLUSTER

## Creación de Instancias EC2

Se crearon dos instancias EC2 tipo `t3.medium`:

| Nombre | Tipo | Sistema Operativo |
|---|---|---|
| master-node | t3.medium | Ubuntu 24.04 |
| worker-node | t3.medium | Ubuntu 24.04 |

---

## Configuración del Security Group

Se habilitaron los siguientes puertos:

| Tipo | Puerto |
|---|---|
| SSH | 22 |
| Kubernetes API | 6443 |
| All Traffic | Security Group Interno |

Esto permitió:

- Comunicación entre nodos
- Funcionamiento de Calico
- Comunicación kubelet
- DNS interno
- Tráfico entre pods

---

## Instalación de Kubernetes

En ambas instancias se realizaron los siguientes pasos:

```
sudo apt update && sudo apt upgrade -y

sudo swapoff -a

sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

## Configuración de módulos
```
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay

sudo modprobe br_netfilter
```

## Configuración sysctl

```
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
net.ipv4.ip_forward=1
EOF

sudo sysctl --system
``` 

## Instalación de containerd

``` 
sudo apt install -y containerd

sudo mkdir -p /etc/containerd

containerd config default | sudo tee /etc/containerd/config.toml
``` 

Hallar y modificar lo siguiente:

``` 
SystemdCgroup = false
``` 

por el valor `true`:

``` 
SystemdCgroup = true
``` 

## Instalación de Kubernetes
``` 
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
sudo mkdir -p /etc/apt/keyrings

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | \
sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /" | \

sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt update

sudo apt install -y kubelet kubeadm kubectl

sudo apt-mark hold kubelet kubeadm kubectl
``` 

## Inicialización del Cluster

SÓLO EN LA INSTANCIA MASTER

```
sudo kubeadm init --pod-network-cidr=192.168.0.0/16

Configuración kubectl

mkdir -p $HOME/.kube

sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config

sudo chown $(id -u):$(id -g) $HOME/.kube/config
``` 

## Instalación de Calico

``` 
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/calico.yaml
``` 

## Unión del Worker

EN LA INSTANCIA MASTER

``` 
kubeadm token create --print-join-command
``` 

EN LA INSTANCIA WORKER
``` 
sudo kubeadm join kubeadm join 172.31.39.208:6443 --token ic89oi.8uitzqfks12a49u0 --discovery-token-ca-cert-hash sha256:71372e48cd88e71f15084d06b5fd135de371a25be7ab23f848cab87d4534e815
``` 

### Validación

``` 
kubectl get nodes
``` 

Resultado esperado:
``` 
master-node   Ready   control-plane
worker-node   Ready   <none>
``` 

``` 
kubectl cluster-info
``` 

# FASE 2 — ESCALABILIDAD Y BACKUPS

### Namespace

``` 
kubectl create namespace t02-cloud-security
```

### Deployment
```
kubectl create deployment web-app \
--image=nginx:1.25 \
-n t02-cloud-security
```
### Escalamiento
``` 
kubectl scale deployment web-app \
--replicas=2 \
-n t02-cloud-security
``` 

### Service
``` 
kubectl expose deployment web-app \
--port=80 \
--target-port=80 \
--name=web-app-service \
-n t02-cloud-security
``` 

### Validación
```
kubectl get pods -n t02-cloud-security -o wide
kubectl get svc -n t02-cloud-security
```
### Rolling Update
```
kubectl set image deployment/web-app nginx=nginx:1.26 -n t02-cloud-security
```
### Validar Update
```
kubectl rollout status deployment/web-app -n t02-cloud-security
```
### Resultado:
```
deployment "web-app" successfully rolled out
```
### Backup

Se realizó un snapshot del volumen EBS desde AWS Console.

### Ruta:

EC2 → Volumes → Create Snapshot
# FASE 3 — RBAC

### Archivo: rbac-role.yaml
```
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: t02-cloud-security
  name: pod-reader-role
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
```
### Archivo: rbac-rolebinding.yaml
```
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pod-reader-binding
  namespace: t02-cloud-security

subjects:
  - kind: User
    name: usuario
    apiGroup: rbac.authorization.k8s.io

roleRef:
  kind: Role
  name: pod-reader-role
  apiGroup: rbac.authorization.k8s.io
```
## Aplicar RBAC
```
kubectl apply -f rbac-role.yaml
```
```
kubectl apply -f rbac-rolebinding.yaml
```
### Validar
```
kubectl auth can-i get pods --as=usuario -n t02-cloud-security
```
### Resultado:
```
yes
```
```
kubectl auth can-i delete pods --as=usuario -n t02-cloud-security
```
### Resultado:
```
no
```

# FASE 4 — NETWORK POLICIES

### Archivo: networkpolicy-deny-all.yaml
```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy

metadata:
  name: deny-all-ingress
  namespace: t02-cloud-security

spec:
  podSelector:
    matchLabels:
      app: web-app

  policyTypes:
    - Ingress
```
### Archivo: networkpolicy-allow-authorized.yaml
```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy

metadata:
  name: allow-authorized-pods
  namespace: t02-cloud-security

spec:
  podSelector:
    matchLabels:
      app: web-app

  policyTypes:
    - Ingress

  ingress:
    - from:
        - podSelector:
            matchLabels:
              role: authorized

      ports:
        - protocol: TCP
          port: 80
```

Aplicar Policies
```
kubectl apply -f networkpolicy-deny-all.yaml
```
```
kubectl apply -f networkpolicy-allow-authorized.yaml
```
## Crear Pods de prueba

### Pod denied
```
kubectl run denied \
--image=busybox \
-n t02-cloud-security \
--labels="role=denied" \
--restart=Never \
-- sleep 3600
```

### Pod allowed
```
kubectl run allowed \
--image=busybox \
-n t02-cloud-security \
--labels="role=authorized" \
--restart=Never \
-- sleep 3600
```

### Validación

Debe funcionar
```
kubectl exec -it allowed -n t02-cloud-security -- wget -qO- http://IP_SERVICE
```
Resultado:
```
Welcome to nginx!
```
Debe fallar
```
kubectl exec -it denied -n t02-cloud-security -- wget -qO- --timeout=5 http://IP_SERVICE
```
Resultado:
```
wget: download timed out
```

# FASE 5 — TRIVY
Instalación
```
sudo apt-get install wget apt-transport-https gnupg -y

wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | \
gpg --dearmor | sudo tee /usr/share/keyrings/trivy.gpg > /dev/null
echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] \
https://aquasecurity.github.io/trivy-repo/deb generic main" | \

sudo tee -a /etc/apt/sources.list.d/trivy.list

sudo apt update

sudo apt install trivy -y
```

### Escaneo
```
trivy image nginx:1.25
```
### Guardar Reporte
```
trivy image nginx:1.25 > trivy-report.txt
```
# Problemas y Soluciones

### Problema 1 — Permisos PEM

## Causa

Permisos inseguros en la llave privada.

## Solución
```
chmod 400 ubnt.pem
```
### Problema 2 — Worker sin acceso kubectl

## Causa

admin.conf pertenece a root.

## Solución
```
sudo cp /etc/kubernetes/admin.conf /home/ubuntu/admin.conf

sudo chown ubuntu:ubuntu /home/ubuntu/admin.conf
```
### Problema 3 — Policies no funcionaban
## Causa

No existía CNI compatible.

## Solución

Instalación de Calico.

### Problema 4 — Error DNS
## Causa

Calico aún no iniciaba correctamente.

## Solución

Esperar inicialización de pods kube-system.

### Problema 5 — Vulnerabilidades
## Causa

Uso de imágenes antiguas.

## Solución

Escaneo y validación con Trivy.

# Conclusión

Se implementó correctamente un cluster Kubernetes funcional sobre AWS EC2 aplicando:

Alta disponibilidad
Escalabilidad
Rolling Updates
RBAC
Network Policies
Seguridad de imágenes
Backups

El proyecto permitió comprender la implementación real de Kubernetes en cloud y la aplicación de controles profesionales de seguridad.