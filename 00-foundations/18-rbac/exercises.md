## Workshop 18 — Service Accounts & RBAC (Seguridad y permisos en Kubernetes)

---

## Objetivo (nivel enterprise / entrevistas)

Al finalizar este workshop podrás:

- Explicar **qué es RBAC** y por qué es crítico en Kubernetes
- Diferenciar **User vs ServiceAccount**
- Entender **Role / ClusterRole / RoleBinding / ClusterRoleBinding**
- Crear permisos **mínimos necesarios (least privilege)**
- Ver cómo Kubernetes **autoriza** cada request
- Detectar errores de seguridad comunes
- Responder preguntas típicas de entrevistas (AKS / EKS / GKE)

---

## Regla 0 — Contexto correcto

```
kubectl config current-context
kubectl get nodes
```

Contexto esperado:

```
kind-k8s-lab
```

Namespace de trabajo:

```
practice
```

---

## Concepto clave (muy importante)

> **RBAC = Authorization**
>
> Autenticación → ¿Quién eres?  
> Autorización → ¿Qué puedes hacer?

Kubernetes **SIEMPRE** evalúa RBAC antes de ejecutar una acción.

---

## Parte A — Identidades en Kubernetes

### Tipos de identidades

1. **Users**
   - Humanos
   - Externos (OIDC, Azure AD, IAM, etc.)
   - Kubernetes no los gestiona directamente

2. **ServiceAccounts**
   - Identidad **interna**
   - Usada por Pods y controladores
   - Vive dentro de un Namespace

Ver ServiceAccounts por defecto:

```
kubectl -n practice get serviceaccounts
```

Salida esperada:

```
default
```

---

## Parte B — Qué es RBAC

RBAC se compone de 4 objetos:

1. **Role**
   - Permisos en un namespace
2. **ClusterRole**
   - Permisos a nivel cluster
3. **RoleBinding**
   - Une Role + identidad
4. **ClusterRoleBinding**
   - Une ClusterRole + identidad

---

## Parte C — Crear ServiceAccount dedicada

```
kubectl -n practice create serviceaccount app-sa
```

Verificar:

```
kubectl -n practice get serviceaccounts
```

---

## Parte D — Crear Role (permisos mínimos)

Objetivo:

> El Pod solo puede **listar y leer Pods** en su namespace.

Archivo: `role-pod-reader.yaml`

```
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: practice
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]
```

Aplicar:

```
kubectl apply -f role-pod-reader.yaml
```

---

## Parte E — Asociar Role a ServiceAccount

Archivo: `rolebinding-pod-reader.yaml`

```
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pod-reader-binding
  namespace: practice
subjects:
- kind: ServiceAccount
  name: app-sa
  namespace: practice
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

Aplicar:

```
kubectl apply -f rolebinding-pod-reader.yaml
```

---

## Parte F — Usar ServiceAccount en un Pod

Archivo: `pod-rbac-test.yaml`

```
apiVersion: v1
kind: Pod
metadata:
  name: rbac-test-pod
  namespace: practice
spec:
  serviceAccountName: app-sa
  containers:
  - name: tester
    image: bitnami/kubectl:1.30
    command:
    - sh
    - -c
    - sleep 3600
```

Aplicar:

```
kubectl apply -f pod-rbac-test.yaml
```

---

## Parte G — Probar permisos desde el Pod

Entrar al Pod:

```
kubectl -n practice exec -it rbac-test-pod -- sh
```

### G.1 Acción permitida

```
kubectl get pods
```

Resultado esperado:

```
Lista de pods
```

---

### G.2 Acción NO permitida

```
kubectl get services
```

Resultado esperado:

```
Error from server (Forbidden)
```

Esto confirma que **RBAC está funcionando**.

---

## Parte H — ClusterRole (visión general)

ClusterRole se usa para:

- Nodes
- Storage
- Namespaces
- Recursos cluster-wide

Ejemplo (NO aplicar todavía):

```
kind: ClusterRole
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "list"]
```

---

## Parte I — Errores críticos en producción

❌ Errores comunes:

- Usar `cluster-admin` para todo
- Compartir ServiceAccounts
- No revisar permisos heredados
- No auditar RBAC

✅ Buenas prácticas:

- Least privilege
- Un SA por app
- Revisión periódica
- Namespaces bien definidos
- GitOps para RBAC

---

## Parte J — Pregunta típica de entrevista

**Pregunta:**
> ¿Cómo aseguras que un Pod solo haga lo que necesita?

**Respuesta senior:**
> “Usamos ServiceAccounts dedicadas con Roles mínimos
> y RoleBindings por namespace,
> evitando permisos cluster-wide y reduciendo superficie de ataque.”

---

## Cleanup (opcional)

```
kubectl -n practice delete pod rbac-test-pod
kubectl -n practice delete rolebinding pod-reader-binding
kubectl -n practice delete role pod-reader
kubectl -n practice delete serviceaccount app-sa
```

---

