# Kubernetes Platform Runbooks — Conventions & Standards

## Propósito de este documento

Este documento define las **convenciones oficiales**, **estándares técnicos** y **criterios metodológicos**
utilizados en el repositorio **kubernetes-platform-runbooks**.

Su objetivo es asegurar que todo el contenido del repositorio sea:

- Profesional (nivel empresa / consultora)
- Reproducible
- Didáctico
- Entendible en entrevistas técnicas
- Útil como runbook operativo real
- Escalable en el tiempo

Este archivo es el **contrato técnico** del repositorio.

---

## Audiencia objetivo

Este repositorio está diseñado para:

- Platform Engineers
- Cloud Engineers
- DevOps / SRE
- SysAdmins evolucionando a Kubernetes
- Personas preparándose para entrevistas AKS / EKS / Kubernetes en general
- Equipos que necesitan runbooks operativos claros

No está orientado a principiantes absolutos sin base en Linux.

---

## Filosofía del repositorio

Principios rectores:

1. **Infraestructura como código**
2. **Estado deseado siempre explícito**
3. **Automatización antes que pasos manuales**
4. **Documentación como parte del sistema**
5. **Cada comando debe tener un porqué**
6. **Lo que no se explica, no se considera aprendido**
7. **Todo debe poder eliminarse sin costos residuales**

---

## Estructura general del repositorio

El repositorio sigue una estructura **por dominios de conocimiento**, no por herramientas.

```
kubernetes-platform-runbooks/
├── 00-foundations/
│   ├── 00-environment/
│   ├── 01-kind-cluster/
│   └── 02-kubectl-basics/
├── 01-workloads/
├── 02-networking/
├── 03-storage/
├── 04-security/
├── 05-observability/
├── 06-operations/
├── 07-troubleshooting/
├── docs/
└── README.md
```

---

## Convenciones de nombres (Naming Standards)

### Directorios

- Usar **kebab-case**
- Prefijo numérico para orden lógico
- Nombres descriptivos y no genéricos

Ejemplos correctos:
- `00-foundations`
- `02-kubectl-basics`
- `05-observability`

Ejemplos incorrectos:
- `basics`
- `k8s`
- `test`

---

### Archivos Markdown

Convenciones:

- README.md → explica el **contexto completo** del directorio
- exercises.md → solo práctica guiada
- troubleshooting.md → errores reales y cómo resolverlos
- interview-notes.md → guiones de entrevista

---

### Scripts

Convenciones obligatorias:

- Bash con `#!/usr/bin/env bash`
- `set -euo pipefail`
- Idempotentes siempre que sea posible
- Logs claros
- No hardcodear valores sensibles

Nombre de scripts:
- `bootstrap-k8s-lab.sh`
- `create-kind-cluster.sh`
- `cleanup-cluster.sh`

---

## Convenciones de Kubernetes

### Namespaces

Regla base:

- Nunca trabajar todo en `default` salvo para pruebas rápidas
- Cada ejercicio define explícitamente su namespace

Ejemplos:
- `labs`
- `practice`
- `observability`
- `security`

---

### Recursos Kubernetes

Orden lógico de aprendizaje y uso:

1. Namespace
2. Pod (concepto)
3. Deployment
4. ReplicaSet (conceptual, no manual)
5. Service
6. ConfigMap / Secret
7. Ingress
8. NetworkPolicy
9. HPA
10. RBAC

Nunca se salta el orden sin explicación.

---

## Convenciones de kubectl

Reglas:

- Siempre indicar namespace con `-n`
- Preferir `kubectl apply` sobre `create` en YAML
- `run` solo para aprendizaje, no para producción
- Usar `-o wide`, `-o yaml`, `describe` de forma explícita

Ejemplo correcto:
```
kubectl -n practice get pods -o wide
```

Ejemplo incorrecto:
```
kubectl get pods
```

---

## Convenciones de documentación

Cada ejercicio debe responder explícitamente:

- ¿Qué estoy haciendo?
- ¿Por qué lo hago?
- ¿Qué pasa si no existe?
- ¿Qué pasa si se borra?
- ¿Qué componente se autocura?
- ¿Cómo lo preguntan en entrevistas?

No se acepta documentación sin contexto.

---

## Convenciones de entrevistas técnicas

Cada concepto aprendido debe poder explicarse así:

- Definición simple
- Qué problema resuelve
- Cómo funciona internamente
- Diferencia con otro recurso
- Ejemplo real
- Error común

---

## Convenciones FinOps / Costos

Principios aplicados incluso en labs locales:

- Todo debe poder eliminarse
- Nada debe quedar corriendo sin propósito
- Cada workshop debe tener sección de cleanup
- Los conceptos deben mapearse fácilmente a cloud real (AKS / EKS)

---

## Convenciones de control de versiones

- Commits pequeños y explicativos
- Inglés técnico claro
- Ejemplo:
  - `feat: add kubectl deployment lifecycle lab`
  - `docs: add troubleshooting for pod recreation`

---

## Regla final (no negociable)

Si algo:

- No se entiende
- No se puede repetir
- No se puede explicar en una entrevista
- No se puede destruir limpiamente

Entonces **no pertenece a este repositorio**.

---

## Estado del documento

Este archivo es **vivo** y se actualizará a medida que el repositorio crezca.

Última revisión: inicial
Autor: José Garagorry
Rol objetivo: Platform Engineer / Kubernetes Engineer

