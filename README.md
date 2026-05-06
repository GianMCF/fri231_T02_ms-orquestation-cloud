# fri231_T02_ms-orquestation-cloud

## Proyecto: Kubernetes en Plataforma Cloud AWS con Seguridad Profesional

## Arquitectura

Se implementó un cluster Kubernetes sobre una instancia EC2 utilizando **Kind (Kubernetes in Docker)** para simular un entorno multi-nodo.

### Componentes

* 1 instancia EC2 (Ubuntu 22.04)
* Cluster Kubernetes con:

  * 1 nodo control-plane
  * 2 nodos worker
* Aplicación de prueba (nginx)
* Pods cliente para pruebas de red
* Configuraciones de seguridad:

  * RBAC
  * Network Policies
---
### Diagrama

```
                ┌────────────────────────────────┐
                │        EC2 Instance            │
                │                                │
                │   ┌────────────────────────┐   │
                │   │      Kubernetes        │   │
                │   │     Cluster (Kind)     │   │
                │   │                        │   │
                │   │    [Control Plane]     │   │
                │   │           │            │   │
                │   │     ┌─────┴─────┐      │   │
                │   │     │           │      │   │
                │   │ [Worker1]   [Worker2]  │   │
                │   │     │           │      │   │
                │   │   Pods         Pods    │   │
                │   │     │           │      │   │
                │   │   Service (NodePort)   │   │
                │   │           │            │   │
                │   │     Acceso (curl)      │   │
                │   └────────────────────────┘   │
                └────────────────────────────────┘
```

---

## Proceso

### 1. Creación del cluster

Se utilizó Kind para crear un cluster multi-nodo:

```bash
kind create cluster --config cluster.yaml
kubectl get nodes
kubectl cluster-info
```

Se validó la existencia de 3 nodos (alta disponibilidad lógica).

---

### 2. Despliegue de aplicación

Se desplegó una aplicación nginx con múltiples réplicas:

```bash
kubectl apply -f deployment.yaml
kubectl scale deployment web-app --replicas=3
```

Se comprobó distribución de pods en distintos nodos.

---

### 3. Exposición del servicio

Se creó un Service tipo NodePort:

```bash
kubectl apply -f service.yaml
curl localhost:30007
```

Se validó acceso externo a la aplicación.

---

### 4. Rolling Update

Se actualizó la imagen sin caída del servicio:

```bash
kubectl set image deployment/web-app nginx=nginx:1.26
kubectl rollout status deployment/web-app
```
 Se evidenció actualización progresiva sin downtime.

---

### 5. Backup

Se realizó backup del estado actual del cluster:

```bash
kubectl get all -o yaml > backup.yaml
```

Se almacenó configuración completa del sistema.

---

### 6. RBAC

Se creó un Role y RoleBinding restringiendo acceso a pods.

Validación:

```bash
kubectl auth can-i delete pods --as=usuario-demo
```

 Resultado: acceso denegado.

---

### 7. Network Policies

Se configuraron políticas para:

* Bloquear tráfico por defecto
* Permitir acceso solo desde pods autorizados

Pruebas:

* Pod no autorizado → acceso denegado 
* Pod autorizado → acceso permitido 

---

### 8. Seguridad de imágenes

Se utilizó Trivy para escaneo:

```bash
trivy image nginx:1.25
```

 Se detectaron vulnerabilidades HIGH y CRITICAL.

Se definió política:

* Bloquear imágenes inseguras
* Usar versiones actualizadas

---

## Problemas y Soluciones

### Problema 1: No acceso desde otra instancia

**Causa:** API server en 127.0.0.1
**Solución:** Configurar `apiServerAddress: 0.0.0.0`

---

### Problema 2: NetworkPolicy no funcionaba

**Causa:** Falta de CNI compatible
**Solución:** Uso de configuración adecuada en Kind

---

### Problema 3: Error en RBAC

**Causa:** Role mal definido
**Solución:** Ajuste de permisos (verbs/resources)

---

### Problema 4: Pod no accede al servicio

**Causa:** Selector incorrecto
**Solución:** Corregir labels en deployment y service

---

### Problema 5: Vulnerabilidades en imágenes

**Causa:** Uso de versiones desactualizadas
**Solución:** Actualización de imagen y validación con Trivy

---

## Conclusión

Se logró implementar un entorno Kubernetes funcional que simula un cluster en la nube, aplicando:

* Alta disponibilidad (multi-nodo)
* Escalabilidad
* Seguridad (RBAC + Network Policies)
* Control de imágenes
* Backup del sistema

El proyecto demuestra una comprensión integral de orquestación de contenedores y prácticas de seguridad en Kubernetes.

---

