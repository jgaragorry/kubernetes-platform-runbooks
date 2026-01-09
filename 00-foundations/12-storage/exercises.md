## Workshop 13 — Volumes & Persistent Storage (Estado en Kubernetes)

---

## Objetivo (lo que vas a dominar)
Al finalizar este workshop entenderás **con claridad absoluta**:

- Por qué **los Pods son efímeros**
- Qué es un **Volume** en Kubernetes
- Diferencias reales entre:
  - `emptyDir`
  - `hostPath`
  - `PersistentVolume (PV)`
  - `PersistentVolumeClaim (PVC)`
  - `StorageClass`
- Qué se usa **en labs**, **en producción** y **en cloud**
- Cómo explicarlo **en entrevistas técnicas**

---

## Regla 0 — Contexto y estado base (NO saltar)

Verifica contexto:

```
kubectl config current-context
kubectl get nodes
```

Contexto esperado:

```
kind-k8s-lab
```

Namespace esperado:

```
practice
```

Si no existe:

```
kubectl create namespace practice
```

---

## Estado inicial requerido

Deployment base `web-app` **NO es necesario** en este workshop.  
Trabajaremos con Pods controlados para observar comportamiento de storage.

---

## Concepto clave (mentalidad Kubernetes)

> **Un Pod puede morir en cualquier momento**  
>  
> Si los datos viven dentro del contenedor → **se pierden**

Por eso existen **Volumes**.

---

## Parte A — emptyDir (volumen efímero compartido)

---

## A.1 Qué es `emptyDir`

- Vive mientras el **Pod exista**
- Se borra cuando el Pod muere
- Se comparte entre contenedores del mismo Pod

**Casos reales:**
- Cache
- Archivos temporales
- IPC entre contenedores

---

## A.2 Crear Pod con emptyDir

```
cat <<EOF | kubectl -n practice apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: emptydir-pod
spec:
  containers:
  - name: app
    image: nginx:1.27
    volumeMounts:
    - name: cache
      mountPath: /cache
  volumes:
  - name: cache
    emptyDir: {}
EOF
```

---

## A.3 Escribir datos en el volumen

```
kubectl -n practice exec -it emptydir-pod -- sh -c "echo hola > /cache/data.txt"
kubectl -n practice exec -it emptydir-pod -- cat /cache/data.txt
```

---

## A.4 Eliminar Pod y validar pérdida

```
kubectl -n practice delete pod emptydir-pod
```

Vuelve a crearlo y verifica:

```
kubectl -n practice exec -it emptydir-pod -- ls /cache
```

Resultado esperado:

```
(empty)
```

---

## Parte B — hostPath (NO recomendado en producción)

---

## B.1 Qué es `hostPath`

- Monta una ruta del **nodo**
- Acopla Pod ↔ Node
- **ANTI-PATRÓN en cloud**

Solo se usa para:
- Labs
- Debug
- Componentes de sistema

---

## B.2 Crear Pod con hostPath

```
cat <<EOF | kubectl -n practice apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: hostpath-pod
spec:
  containers:
  - name: app
    image: nginx:1.27
    volumeMounts:
    - name: host-data
      mountPath: /data
  volumes:
  - name: host-data
    hostPath:
      path: /tmp/k8s-data
      type: DirectoryOrCreate
EOF
```

---

## B.3 Probar persistencia

```
kubectl -n practice exec -it hostpath-pod -- sh -c "echo persistente > /data/file.txt"
kubectl -n practice delete pod hostpath-pod
```

Recrear Pod y verificar:

```
kubectl -n practice exec -it hostpath-pod -- cat /data/file.txt
```

---

## Concepto clave (entrevista)

> **hostPath NO escala, NO es portable, NO es cloud-native**

---

## Parte C — PersistentVolume (PV) y PersistentVolumeClaim (PVC)

---

## C.1 Qué problema resuelve PV/PVC

Separan:
- **Infraestructura de storage** (PV)
- **Consumo de storage** (PVC)

Esto permite:
- Reutilización
- Independencia de aplicaciones
- Automatización cloud

---

## C.2 Ver StorageClass disponible (kind)

```
kubectl get storageclass
```

Resultado esperado:

```
standard (local-path)
```

---

## C.3 Crear PersistentVolumeClaim (PVC)

```
cat <<EOF | kubectl -n practice apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: web-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
EOF
```

Verificar:

```
kubectl -n practice get pvc
```

Estado esperado:

```
Bound
```

---

## C.4 Usar PVC en un Pod

```
cat <<EOF | kubectl -n practice apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: pvc-pod
spec:
  containers:
  - name: app
    image: nginx:1.27
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: web-pvc
EOF
```

---

## C.5 Validar persistencia real

```
kubectl -n practice exec -it pvc-pod -- sh -c "echo persistente > /data/info.txt"
kubectl -n practice delete pod pvc-pod
```

Recrear Pod y validar:

```
kubectl -n practice exec -it pvc-pod -- cat /data/info.txt
```

Resultado esperado:

```
persistente
```

---

## Parte D — Qué se usa en producción (cloud)

---

| Tipo | Producción |
|----|----|
| emptyDir | ❌ |
| hostPath | ❌ |
| PV/PVC | ✅ |
| StorageClass | ✅ |
| CSI Drivers | ✅ |

Cloud:
- EKS → EBS / EFS
- AKS → Azure Disk / Files
- GKE → Persistent Disk

---

## Parte E — Errores comunes (entrevista)

- ❌ Pensar que los Pods guardan estado
- ❌ Usar hostPath en cloud
- ❌ No entender accessModes
- ❌ Confundir PVC con PV

---

## Cleanup (opcional)

```
kubectl -n practice delete pod emptydir-pod hostpath-pod pvc-pod
kubectl -n practice delete pvc web-pvc
```

---
