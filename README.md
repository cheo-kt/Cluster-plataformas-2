# Cluster-plataformas-2



# Proyecto: Deployments y Ingress en Minikube

**Autor:** Sergio Fernando Florez Sanabria
**Código:** A00396046
**Curso:** Plataformas 2

---

## Resumen

Este README documenta paso a paso la práctica realizada con **Minikube**: instalación/arranque del clúster local, habilitación de `metrics-server` e `ingress`, creación de **4 Deployments** (cada uno con su Pod y Service tipo `ClusterIP`), exposición mediante **Ingress** y pruebas. También incluye los comandos de diagnóstico y solución de problemas usados.

> Nota: ya contaba con el `deployment myapp` (nginx) como uno de los cuatro.

---

## Prerrequisitos

* Máquina con Docker y Minikube instalados (driver `docker` en este lab).
* `kubectl` instalado y configurado para usar el clúster de Minikube.
* Permiso para editar `/etc/hosts` en tu máquina (para mapear dominios locales a la IP de Minikube).

---

## Resumen de las acciones principales (comandos reales usados)

(Se listan aquí en orden lógico para reproducir el laboratorio)

### 1. Iniciar Minikube

```bash
minikube start --driver=docker
minikube ip                # obtener IP del nodo (ej. 192.168.49.2)
minikube status
```

### 2. Habilitar addons útiles

```bash
minikube addons enable metrics-server
minikube addons enable ingress
```

> Si el addon `ingress` queda atascado en `ContainerCreating`, usar la sección *Troubleshooting* abajo.

### 3. Crear los 4 Deployments (el primero ya estaba creado: `myapp`)

```bash
# Deployment 1 (ya creado)
kubectl create deployment myapp --image=nginx
kubectl scale deployment myapp --replicas=1
kubectl expose deployment myapp --port=80 --target-port=80 --type=ClusterIP

# Deployment 2 (Apache)
kubectl create deployment apacheapp --image=httpd
kubectl scale deployment apacheapp --replicas=1
kubectl expose deployment apacheapp --port=80 --target-port=80 --type=ClusterIP

# Deployment 3 (Redis)
kubectl create deployment redisapp --image=redis
kubectl scale deployment redisapp --replicas=1
kubectl expose deployment redisapp --port=6379 --target-port=6379 --type=ClusterIP

# Deployment 4 (whoami, sirve para probar respuestas HTTP)
kubectl create deployment whoamiapp --image=traefik/whoami
kubectl scale deployment whoamiapp --replicas=1
kubectl expose deployment whoamiapp --port=80 --target-port=80 --type=ClusterIP
```

### 4. Verificar pods y servicios

```bash
kubectl get pods
kubectl get deployments
kubectl get svc
kubectl get endpoints
```

### 5. Crear `ingress.yaml` (archivo) y aplicarlo

Archivo `ingress.yaml` (ejemplo usado):

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: myapp.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: myapp
            port:
              number: 80
  - host: apache.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: apacheapp
            port:
              number: 80
  - host: redis.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: redisapp
            port:
              number: 6379
  - host: whoami.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: whoamiapp
            port:
              number: 80
```

Aplicar el Ingress:

```bash
kubectl apply -f ingress.yaml
kubectl get ingress
```

### 6. Mapear dominios locales al IP de Minikube

Obtener IP:

```bash
minikube ip   # ej. 192.168.49.2
```

Agregar al `/etc/hosts` (Linux/macOS) o al archivo de hosts de Windows:

```
192.168.49.2 myapp.local apache.local redis.local whoami.local
```

### 7. Probar desde la máquina local (navegador o curl)

```bash
curl http://myapp.local
curl http://apache.local
curl http://whoami.local
curl http://redis.local   # -> 502 Bad Gateway esperado (ver explicación)
```

---

## Qué significan los resultados (explicaciones)

* `kubectl get pods` → muestra que cada deployment tiene su Pod en `Running`.
* `kubectl get svc` → muestra servicios `ClusterIP` creados para cada deployment.
* `kubectl get ingress` → muestra `ADDRESS` = `minikube ip` y hosts configurados.
* `curl http://redis.local` → **502 Bad Gateway**: normal, porque Redis es un servicio TCP (puerto 6379) que **no habla HTTP**; el Ingress (L7/nginx) intentó enrutar HTTP a un backend que no responde HTTP. Esto indica que el Ingress está funcionando (hace el ruteo), pero el backend no es compatible con HTTP.

---

## Opciones para Redis (si quieres acceder desde fuera)

* **Con cliente Redis dentro del clúster**:

  ```bash
  kubectl run -it redis-client --image=redis -- bash
  # dentro del pod:
  redis-cli -h redisapp
  ```
* **Exponer Redis por NodePort (o LoadBalancer + minikube tunnel)**:

  ```bash
  # NodePort (acceso desde host en puerto alto)
  kubectl expose deployment redisapp --type=NodePort --port=6379 --target-port=6379

  # O LoadBalancer (minikube requiere `minikube tunnel` para external IP)
  kubectl expose deployment redisapp --type=LoadBalancer --port=6379 --target-port=6379
  # en otra terminal:
  minikube tunnel
  ```

---

## Metrics (kubectl top)

* Habilitar metrics-server:

  ```bash
  minikube addons enable metrics-server
  ```
* Probar:

  ```bash
  kubectl top nodes
  kubectl top pods   # en default mostrará los pods que existan; si no hay, "No resources found"
  ```

---

## Troubleshooting (problemas comunes y comandos de diagnóstico)

### Ingress atascado en `ContainerCreating` o `ImagePull` lento

1. Ver pods del namespace:

```bash
kubectl get pods -n ingress-nginx -o wide
kubectl describe pod -n ingress-nginx <ingress-pod-name>
kubectl get events -n ingress-nginx --sort-by=.metadata.creationTimestamp
```

2. Forzar descarga de imágenes dentro de Minikube:

```bash
minikube image pull registry.k8s.io/ingress-nginx/controller:v1.13.2
minikube image pull registry.k8s.io/ingress-nginx/kube-webhook-certgen:v1.6.2
```

3. Si las jobs de admission no crearon el secret a tiempo (sincronización), recrear el pod controller:

```bash
# Verificar jobs y secret
kubectl get jobs -n ingress-nginx
kubectl get secret -n ingress-nginx

# Si jobs completaron y secret existe, recrear controller para que monte el secret:
kubectl delete pod -n ingress-nginx -l app.kubernetes.io/component=controller
kubectl get pods -n ingress-nginx -w
```

4. Reset del addon:

```bash
minikube addons disable ingress
minikube addons enable ingress
```

5. Si la imagen no puede bajarse desde `registry.k8s.io`, usar mirror o `minikube image load` con una imagen descargada localmente:

```bash
docker pull k8s.gcr.io/ingress-nginx/controller:v1.13.2
minikube image load k8s.gcr.io/ingress-nginx/controller:v1.13.2
```

### `kube-apiserver` no healthy (fallo inicial)

* Asegurarse de desactivar swap (si se presentó el error en `kubeadm`):

```bash
sudo swapoff -a
# Para hacerlo permanente, comentar la línea swap en /etc/fstab
```

### Ver logs generales de Minikube

```bash
minikube logs --file=minikube-logs.txt
```

---

## Comandos útiles adicionales

```bash
kubectl describe pod <pod-name>
kubectl logs <pod-name>               # logs de un pod
kubectl logs -n kube-system -l k8s-app=metrics-server
kubectl get svc -o wide
kubectl get ingress -o wide
minikube dashboard
minikube service list
minikube service myapp               # abre servicio en el navegador usando Minikube
```

---

## Limpieza (quitar todo cuando termines)

```bash
kubectl delete -f ingress.yaml
kubectl delete deployment myapp apacheapp redisapp whoamiapp
kubectl delete svc myapp apacheapp redisapp whoamiapp
minikube addons disable ingress
minikube stop
minikube delete
```

---

## Notas finales

* El Ingress (ingress-nginx) gestiona tráfico HTTP/HTTPS (L7). No es la herramienta correcta para exponer servicios no-HTTP (como Redis) a navegadores; para eso usar NodePort o LoadBalancer y/o clientes específicos.
* En Minikube, `LoadBalancer` a menudo requiere `minikube tunnel` para asignar IP externa.
* El comportamiento lento en la activación/descarga de imágenes suele ser por la velocidad/latencia de la red y el pull del registry; `minikube image pull` o `minikube image load` ayudan a mitigar.




##Imagenes/evidencia.

<img width="1600" height="759" alt="image" src="https://github.com/user-attachments/assets/5388f2aa-328e-4db4-9a2d-2eba67e23b4e" />

---
<img width="1600" height="72" alt="image" src="https://github.com/user-attachments/assets/18e6f063-d92d-4964-962b-82716c407701" />

---
<img width="1600" height="746" alt="image" src="https://github.com/user-attachments/assets/ee82ba60-9f7e-454e-b961-66f56bdf28ea" />

---
<img width="1384" height="629" alt="image" src="https://github.com/user-attachments/assets/1ba848f8-3490-49e0-b6cf-30148e7a8fb9" />

---
<img width="1198" height="532" alt="image" src="https://github.com/user-attachments/assets/197562b4-2847-432b-b610-4dcbfa45f618" />

---
<img width="878" height="1051" alt="image" src="https://github.com/user-attachments/assets/cc597fd8-b865-4ffc-acbc-b4a14f2361c6" />

---
<img width="772" height="101" alt="image" src="https://github.com/user-attachments/assets/4620043d-fe8f-43d7-8cdf-0bdac2648341" />

---
<img width="805" height="546" alt="image" src="https://github.com/user-attachments/assets/40c4b1f5-6984-4997-b1c9-fb9a7ce61e62" />

---
<img width="1198" height="532" alt="image" src="https://github.com/user-attachments/assets/279e26a5-aa82-45cc-b524-bad2d38d1fb8" />

---
<img width="1600" height="554" alt="image" src="https://github.com/user-attachments/assets/9c31ef10-e77b-4392-ad8f-dcd8f0fc0e45" />

---
<img width="1144" height="590" alt="image" src="https://github.com/user-attachments/assets/ab7e3777-ed31-4360-97e6-c5da1c8509e2" />

---
<img width="1194" height="245" alt="image" src="https://github.com/user-attachments/assets/43371694-5092-468a-9b0d-5d2470e42a33" />

---


