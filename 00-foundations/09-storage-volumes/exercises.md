## Workshop 10 — Volúmenes y Persistencia (Storage en Kubernetes)

---

## Objetivo (lo que vas a dominar)
Al finalizar este workshop podrás **diseñar, usar y explicar**:

- Por qué los **pods son efímeros**
- Diferencias **reales** entre:
  - `emptyDir`
  - `hostPath`
  - `PersistentVolume (PV)`
  - `PersistentVolumeClaim (PVC)`
- Cuándo usar cada tipo (labs vs enterprise)
- Cómo Kubernetes separa **cómputo** de **storage**
- Cómo responder **preguntas de entrevista** sobre estado y persistencia

---

## Regla 0 — Contexto correcto

Antes de empezar:

```
kubectl config current-context
kubectl get nodes
```

Debes estar en:

```
kind-k8s-lab
```

---

## Estado inicial esperado

- Cluster KIND operativo
- Namespace `practice`
- Ningún volumen creado

Verifica:

```
kubectl get pv,pvc --all-namespaces
```

Resultado esperado:
- Ningún PV ni PVC creado (o solo storage interno de KIND)

---

## Concepto base (clave mental)

> **Los pods mueren.  
> El storage no debería morir con ellos.**

---

## Tipos de Volúmenes (visión global)

| Tipo | Persistente | Scope | Uso real |
|----|-----------|------|--------|
| emptyDir | ❌ | Pod | Cache, tmp |
| hostPath | ⚠️ | Nodo | Labs |
| PV + PVC | ✅ | Cluster | Producción |

---

## Paso 1 — emptyDir (volumen efímero)

### Qué es
- Vive **mientras el pod vive**
- Se borra cuando el pod muere
- Ideal para cache temporal

---

### 1.1 Crear Deployment con emptyDir

Archivo `deployment-emptydir.yaml`:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-emptydir
  namespace: practice
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web-emptydir
  template:
    metadata:
      labels:
        app: web-emptydir
    spec:
      containers:
      - name: nginx
        image: nginx:1.27
        volumeMounts:
        - name: cache-volume
          mountPath: /cache
      volumes:
      - name: cache-volume
        emptyDir: {}
```

Aplicar:

```
kubectl apply -f deployment-emptydir.yaml
```

---

### 1.2 Probar comportamiento

```
kubectl -n practice exec deploy/web-emptydir -- sh -c "echo hola > /cache/test.txt"
kubectl -n practice exec deploy/web-emptydir -- cat /cache/test.txt
```

Eliminar pod:

```
kubectl -n practice delete pod -l app=web-emptydir
```

Verificar:

```
kubectl -n practice exec deploy/web-emptydir -- ls /cache
```

Resultado esperado:
- Archivo **desapareció**

---

## Paso 2 — hostPath (NO recomendado en producción)

### Qué es
- Monta un path **real del nodo**
- Acopla el pod al nodo
- Solo para **labs y pruebas**

---

### 2.1 Deployment con hostPath

Archivo `deployment-hostpath.yaml`:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-hostpath
  namespace: practice
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web-hostpath
  template:
    metadata:
      labels:
        app: web-hostpath
    spec:
      containers:
      - name: nginx
        image: nginx:1.27
        volumeMounts:
        - name: host-storage
          mountPath: /data
      volumes:
      - name: host-storage
        hostPath:
          path: /tmp/k8s-data
          type: DirectoryOrCreate
```

Aplicar:

```
kubectl apply -f deployment-hostpath.yaml
```

---

### 2.2 Probar persistencia

```
kubectl -n practice exec deploy/web-hostpath -- sh -c "echo persistente > /data/file.txt"
kubectl -n practice delete pod -l app=web-hostpath
kubectl -n practice exec deploy/web-hostpath -- cat /data/file.txt
```

Resultado:
- El archivo **permanece** (vive en el nodo)

---

## Advertencia enterprise

> **hostPath NO se usa en producción**
- No escala
- No es portable
- No es seguro

---

## Paso 3 — PersistentVolume (PV)

### Qué es
- Recurso **cluster-wide**
- Representa storage físico o lógico
- Administra capacidad

---

### 3.1 Crear PersistentVolume

Archivo `pv.yaml`:

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-lab
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /tmp/pv-lab
```

Aplicar:

```
kubectl apply -f pv.yaml
```

Verificar:

```
kubectl get pv
```

---

## Paso 4 — PersistentVolumeClaim (PVC)

### Qué es
- Pedido de storage
- Lo usa la app, no el PV directo

---

### 4.1 Crear PVC

Archivo `pvc.yaml`:

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-lab
  namespace: practice
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

Aplicar:

```
kubectl apply -f pvc.yaml
```

Verificar:

```
kubectl get pvc -n practice
kubectl get pv
```

Estado esperado:
- PVC: Bound
- PV: Bound

---

## Paso 5 — Usar PVC en Deployment

Archivo `deployment-pvc.yaml`:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-pvc
  namespace: practice
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web-pvc
  template:
    metadata:
      labels:
        app: web-pvc
    spec:
      containers:
      - name: nginx
        image: nginx:1.27
        volumeMounts:
        - name: storage
          mountPath: /data
      volumes:
      - name: storage
        persistentVolumeClaim:
          claimName: pvc-lab
```

Aplicar:

```
kubectl apply -f deployment-pvc.yaml
```

---

### 5.1 Probar persistencia real

```
kubectl -n practice exec deploy/web-pvc -- sh -c "echo datos-importantes > /data/db.txt"
kubectl -n practice delete pod -l app=web-pvc
kubectl -n practice exec deploy/web-pvc -- cat /data/db.txt
```

Resultado:
- El archivo **permanece**

---

## Tabla mental definitiva (memorizar)

| Recurso | Vive más que el Pod | Uso |
|------|------------------|-----|
| emptyDir | ❌ | Cache |
| hostPath | ⚠️ | Labs |
| PV + PVC | ✅ | Producción |

---

## Preguntas de entrevista (respuestas pro)

- “Los pods no deben manejar estado.”
- “La app consume PVC, no PV.”
- “El storage debe ser independiente del lifecycle del pod.”
- “En cloud uso CSI + StorageClasses.”

---

## Errores comunes

### ❌ Usar hostPath en producción
### ❌ Guardar datos críticos en emptyDir
### ❌ Acoplar app a infraestructura
### ❌ No entender accessModes

---

## Cleanup (recomendado)

```
kubectl delete -f deployment-emptydir.yaml
kubectl delete -f deployment-hostpath.yaml
kubectl delete -f deployment-pvc.yaml
kubectl delete -f pvc.yaml
kubectl delete -f pv.yaml
```

---
