## Workshop 25 — Volumes, PersistentVolume y PersistentVolumeClaim

---

## Objetivo (nivel enterprise / entrevistas)

Al finalizar este workshop podrás:

- Explicar **por qué los Pods son efímeros**
- Diferenciar **Volume vs PersistentVolume**
- Entender **PV / PVC / StorageClass**
- Usar almacenamiento **persistente real**
- Explicar storage en **AKS / EKS / GKE**
- Responder preguntas típicas de entrevistas

---

## Regla 0 — Contexto y namespace

Verifica contexto:

```
kubectl config current-context
```

Resultado esperado:

```
kind-k8s-lab
```

Namespace de trabajo:

```
practice
```

Si no existe:

```
kubectl create namespace practice
```

---

## Concepto clave (muy importante)

> **Un Pod NO es persistente**

- Si el Pod muere → los datos del filesystem interno se pierden
- Kubernetes separa:
  - **compute** (Pod)
  - **storage** (Volume)

---

## Parte A — Volume efímero (emptyDir)

### Crear deployment con emptyDir

```
kubectl -n practice create deployment vol-demo \
  --image=nginx:1.27
```

Editar:

```
kubectl -n practice edit deployment vol-demo
```

Agregar:

```
volumes:
  - name: temp-data
    emptyDir: {}
```

Y en el container:

```
volumeMounts:
  - name: temp-data
    mountPath: /usr/share/nginx/html
```

---

## Parte B — Probar pérdida de datos

Entrar al Pod:

```
kubectl -n practice exec -it <pod-name> -- sh
```

Crear archivo:

```
echo "Hola desde emptyDir" > /usr/share/nginx/html/index.html
exit
```

Eliminar Pod:

```
kubectl -n practice delete pod <pod-name>
```

Entrar al nuevo Pod:

```
kubectl -n practice exec -it <nuevo-pod> -- sh
ls /usr/share/nginx/html
```

Resultado esperado:

> ❌ El archivo **no existe**

---

## Parte C — Introducción a Persistent Storage

### Diagrama mental (entrevista)

```
Pod → PVC → PV → Storage real
```

- Pod pide storage (PVC)
- Kubernetes asigna un volumen real (PV)
- Storage sobrevive al Pod

---

## Parte D — Ver StorageClass (infra)

```
kubectl get storageclass
```

En kind deberías ver algo como:

```
local-path (default)
```

---

## Parte E — Crear PersistentVolumeClaim

Crear archivo `pvc.yaml`:

```
cat <<EOF > pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: web-pvc
  namespace: practice
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
EOF
```

Aplicar:

```
kubectl apply -f pvc.yaml
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

## Parte F — Usar PVC en Deployment

Editar deployment:

```
kubectl -n practice edit deployment vol-demo
```

Cambiar volume:

```
volumes:
  - name: persistent-data
    persistentVolumeClaim:
      claimName: web-pvc
```

Y en container:

```
volumeMounts:
  - name: persistent-data
    mountPath: /usr/share/nginx/html
```

---

## Parte G — Probar persistencia real

Entrar al Pod:

```
kubectl -n practice exec -it <pod-name> -- sh
```

Crear archivo:

```
echo "Hola desde PVC" > /usr/share/nginx/html/index.html
exit
```

Eliminar Pod:

```
kubectl -n practice delete pod <pod-name>
```

Entrar al nuevo Pod:

```
kubectl -n practice exec -it <nuevo-pod> -- sh
cat /usr/share/nginx/html/index.html
```

Resultado esperado:

```
Hola desde PVC
```

✔ Persistencia confirmada

---

## Parte H — Diferencias clave (entrevista)

| Recurso | Vive más que el Pod | Uso |
|------|-------------------|----|
| emptyDir | ❌ No | cache / tmp |
| PVC | ✔ Sí | datos app |
| PV | ✔ Sí | disco real |

---

## Parte I — Relación con Cloud Providers

| Cloud | Storage típico |
|-----|---------------|
| AKS | Azure Disk / Azure Files |
| EKS | EBS / EFS |
| GKE | Persistent Disk |

---

## Parte J — Errores comunes

❌ PVC en `Pending`

Checklist:

```
kubectl describe pvc web-pvc
kubectl get storageclass
```

Causas:
- no hay StorageClass default
- request inválido
- permisos

---

## Parte K — Buenas prácticas enterprise

✔ Separar:
- lifecycle app
- lifecycle storage

✔ Nunca:
- borrar PVC sin backup
- montar RWX sin entender impacto

✔ En producción:
- snapshots
- backup (Velero)
- reclaimPolicy controlado

---

## Parte L — Cleanup (opcional)

```
kubectl -n practice delete deployment vol-demo
kubectl -n practice delete pvc web-pvc
kubectl delete -f pvc.yaml
```

---
