## Workshop 22 — DNS & Service Discovery (CoreDNS)

---

## Objetivo (nivel enterprise / entrevistas)

Al finalizar este workshop podrás:

- Explicar **cómo Kubernetes resuelve nombres DNS**
- Entender **qué es CoreDNS y por qué es crítico**
- Saber **cómo un Pod encuentra a otro Pod/Service**
- Diagnosticar problemas de resolución DNS
- Responder preguntas típicas de entrevistas (AKS / EKS / GKE)

---

## Regla 0 — Contexto y namespace

Verifica contexto activo:

```
kubectl config current-context
```

Resultado esperado:

```
kind-k8s-lab
```

Verifica cluster:

```
kubectl get nodes
```

Namespace de trabajo:

```
practice
```

---

## Concepto clave (muy importante)

> Kubernetes **NO usa IPs directamente**
>
> Usa:
>
> - DNS interno
> - Nombres lógicos
> - Service Discovery automático

👉 **DNS es el pegamento del cluster**

---

## Parte A — CoreDNS (el corazón del DNS)

Ver Pods de CoreDNS:

```
kubectl -n kube-system get pods -l k8s-app=kube-dns
```

Resultado esperado:

- 2 Pods
- Estado: Running

Ver Service DNS:

```
kubectl -n kube-system get svc kube-dns
```

Observa:
- ClusterIP (ej: 10.96.0.10)
- Puerto 53 (UDP/TCP)

---

## Parte B — Convención DNS en Kubernetes

Formato general:

```
<service>.<namespace>.svc.cluster.local
```

Ejemplo real:

```
web-clusterip.practice.svc.cluster.local
```

Atajos válidos:

- `web-clusterip`
- `web-clusterip.practice`

👉 Kubernetes completa el resto automáticamente.

---

## Parte C — Deployment + Service base

Deployment:

```
kubectl -n practice create deployment web-app --image=nginx:1.27
kubectl -n practice scale deployment web-app --replicas=2
```

Service ClusterIP:

```
kubectl -n practice expose deployment web-app \
  --name web-svc \
  --port 80 \
  --target-port 80 \
  --type ClusterIP
```

Verificar:

```
kubectl -n practice get deploy,svc,pods
```

---

## Parte D — Resolver DNS desde un Pod

Crear Pod de pruebas:

```
kubectl -n practice run dns-test \
  --image=busybox:1.36 \
  --restart=Never \
  -it -- sh
```

Dentro del Pod:

```
nslookup web-svc
```

Resultado esperado:

- IP del Service (ClusterIP)

Probar FQDN completo:

```
nslookup web-svc.practice.svc.cluster.local
```

Salir:

```
exit
```

---

## Parte E — Flujo real de resolución (muy preguntado)

1. Pod hace request a `web-svc`
2. Consulta DNS → CoreDNS
3. CoreDNS responde ClusterIP
4. kube-proxy balancea a Pods
5. Request llega a un Pod sano

👉 **El Pod NUNCA conoce la IP del otro Pod**

---

## Parte F — Ver endpoints reales

Ver Endpoints asociados al Service:

```
kubectl -n practice get endpoints web-svc
```

Observa:
- IPs reales de Pods
- Actualización automática

Mata un Pod:

```
kubectl -n practice delete pod <pod-name>
```

Revisa endpoints otra vez:

```
kubectl -n practice get endpoints web-svc
```

👉 Kubernetes se autocura.

---

## Parte G — Error común (muy real)

❌ Error:
> “Mi Pod no resuelve el Service”

Checklist:

```
kubectl get svc -n practice
kubectl get endpoints -n practice
kubectl get pods -n kube-system | grep dns
```

Causas típicas:

- Service no existe
- Labels no coinciden
- CoreDNS caído
- Namespace incorrecto

---

## Parte H — Preguntas típicas de entrevista

**Pregunta:**
> ¿Cómo se comunican los microservicios en Kubernetes?

**Respuesta senior:**
> “Mediante Services y DNS interno gestionado por CoreDNS, lo que desacopla IPs y permite escalabilidad.”

---

## Parte I — Cleanup (opcional)

```
kubectl -n practice delete pod dns-test
kubectl -n practice delete svc web-svc
kubectl -n practice delete deployment web-app
```

---
