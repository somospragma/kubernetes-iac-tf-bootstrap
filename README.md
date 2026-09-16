
# kubernetes-iac-tf-bootstrap — Fase 2: Bootstrap del Cluster EKS

**Alcance:** Bootstrap de componentes de plataforma sobre el cluster desplegado en `kubernetes-iac-tf-base`, gestionado 100% vía GitOps (Argo CD). Ningún componente transversal se aplica con `kubectl apply` manual después del bootstrap inicial.

**Pre-requisito:** Cluster EKS activo con Node Group del sistema y los addons base instalados (ver `kubernetes-iac-tf-base`).

---

## Filosofía: qué hace este repo y qué no

Este repo **solo instala Argo CD**. A partir de ahí, **Argo CD se encarga de todo lo demás** — cada componente transversal (node-config, Istio, y los que vengan después) es una `Application` de Argo CD que vive en `apps/` y se sincroniza sola desde Git.

Regla de oro: si necesitas correr un `kubectl apply` manual para algo que no sea el bootstrap inicial (Paso 1 y Paso 4 de esta guía), probablemente esa pieza debería ser una Application más. Ya lo demostramos con los CRDs de Gateway API — incluso un prerequisito de un proyecto externo (`kubernetes-sigs/gateway-api`) se resolvió como una Application apuntando directo al repo upstream, sin ningún paso manual.

---

## Componentes incluidos en esta versión

| Componente | Descripción | Sync-wave |
|---|---|---|
| Argo CD | GitOps engine — se instala manualmente una sola vez, gestiona todo lo demás | — |
| `gateway-api-crds` | CRDs de Kubernetes Gateway API (`Gateway`, `HTTPRoute`, `GatewayClass`...), leídos directo del repo oficial `kubernetes-sigs/gateway-api` | 0 |
| `istio-base` | CRDs propios de Istio (`VirtualService`, `AuthorizationPolicy`...) | 1 |
| `istiod` | Control plane de Istio — corre en el Node Group (Capa 1) | 2 |
| `node-config` | NodeClass + NodePool para nodos Karpenter (Capa 2) | — |
| `apps/istio/gateway.yaml` | Objeto `Gateway` real (Gateway API) — **preparado pero sin exponer al padre todavía** (ver sección "Qué falta") | 3 (futuro) |

---

## Estructura

Patrón App of Apps oficial de Argo CD: `apps/` en la raíz contiene **solo** manifiestos `Application`, de forma plana (sin `recurse`). Los recursos reales viven en subcarpetas y son gestionados por su propia Application hija.

```
bootstrap/
├── argocd/
│   └── values.yaml            ← Helm values para instalar Argo CD
└── bootstrap-app.yaml         ← App of Apps raíz (se aplica una sola vez, plano, sin recurse)

apps/
├── gateway-api-crds-app.yaml  ← Application → repo externo kubernetes-sigs/gateway-api (wave 0)
├── istio-base-app.yaml        ← Application → chart Helm istio-base (wave 1)
├── istiod-app.yaml            ← Application → chart Helm istiod (wave 2)
├── istio/
│   └── gateway.yaml           ← Objeto Gateway real (NO expuesto al padre aún, ver sección "Qué falta")
├── node-config-app.yaml       ← Application que gestiona apps/node-config/
└── node-config/               ← NodeClass + NodePool platform (Karpenter, Capa 2)
    ├── nodeclass-platform.yaml
    └── nodepool-platform.yaml
```

**Por qué esta estructura y no `recurse: true`:** una Application de tipo `Directory` en Argo CD, con `recurse: true`, aplicaría *cualquier* YAML que encuentre en subcarpetas — mezclando manifiestos `Application` con recursos reales (`NodeClass`, `Gateway`), y expondría componentes antes de tiempo (por ejemplo, instalaría Istio automáticamente al mismo tiempo que node-config, sin control). Por eso `apps/` se mantiene plano: cada `Application` hija solo se "activa" cuando su archivo se mueve a la raíz de `apps/`. Ver decisión #12 en `docs/decisiones-arquitectura.md`.

---

## Instalación desde cero

### Paso 0 — Prerequisitos

```bash
# Kubeconfig apuntando al cluster de kubernetes-iac-tf-base
aws eks update-kubeconfig \
  --region us-east-1 \
  --name pragma-sopp-dev-eks-main \
  --profile Sopp_Core_PoC

kubectl get nodes   # debe mostrar los nodos del Node Group, Ready
```

### Paso 1 — Instalar Argo CD

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

helm upgrade --install argocd argo/argo-cd \
  --namespace argocd \
  --create-namespace \
  --version 10.7.1 \
  -f bootstrap/argocd/values.yaml

# Esperar que todos los pods estén Ready (deben quedar en los nodos
# del Node Group, no en Karpenter — la affinity ya está en values.yaml)
kubectl get pods -n argocd -w
```

### Paso 2 — Acceder a Argo CD

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443

kubectl get secret argocd-initial-admin-secret \
  -n argocd \
  -o jsonpath="{.data.password}" | base64 -d && echo

# Abrir https://localhost:8080 — usuario admin, password del comando anterior
```

> ⚠️ **Pendiente de esta fase:** el `argocd-initial-admin-secret` debe eliminarse (o rotarse por un mecanismo de auth definitivo — SSO, usuarios locales con RBAC) antes de considerar esto listo para algo más que un PoC. Ver sección "Qué falta" más abajo.

### Paso 3 — Actualizar valores reales antes de aplicar el bootstrap

Estos valores YA están actualizados en este repo con los reales del cluster `pragma-sopp-dev-eks-main`, pero si se reconstruye para otro cluster, hay que regenerarlos:

```bash
cd ../kubernetes-iac-tf-base

terraform output platform_nodes_sg_id     # → apps/node-config/nodeclass-platform.yaml
terraform output iam_node_role_arn        # → apps/node-config/nodeclass-platform.yaml (role)
terraform output service_subnet_ids       # → apps/node-config/nodeclass-platform.yaml
terraform output cluster_names            # → nombre del cluster en los comandos aws eks
```

### Paso 3.5 — Access Entry para el rol del NodeClass (obligatorio, manual, fuera de Argo CD)

EKS Auto Mode solo crea Access Entries automáticamente para el `NodeClass` `default` y los NodePools built-in. Cualquier `NodeClass` custom con un rol propio necesita su Access Entry manual de **tipo `EC2`** — Argo CD no puede crear esto porque es un recurso de IAM/EKS a nivel de cuenta AWS, no un objeto de Kubernetes:

```bash
aws eks create-access-entry \
  --cluster-name pragma-sopp-dev-eks-main \
  --principal-arn arn:aws:iam::<account-id>:role/<rol-del-nodeclass> \
  --type EC2

aws eks associate-access-policy \
  --cluster-name pragma-sopp-dev-eks-main \
  --principal-arn arn:aws:iam::<account-id>:role/<rol-del-nodeclass> \
  --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSAutoNodePolicy \
  --access-scope type=cluster
```

Sin este paso, el `NodeClass` queda bloqueado con `InstanceProfileReady: False` / `UnauthorizedNodeRole`, y en cascada el `NodePool` nunca llega a `Ready`. Ver decisión #11 en `docs/decisiones-arquitectura.md`.

### Paso 4 — Aplicar el Bootstrap App of Apps (única vez manual)

```bash
kubectl apply -f bootstrap/bootstrap-app.yaml

# Verificar que Argo CD descubrió y sincronizó todas las apps hijas
kubectl get applications -n argocd
# Esperado: bootstrap, gateway-api-crds, istio-base, istiod, node-config
# Todas en SYNC STATUS=Synced, HEALTH STATUS=Healthy
```

Si alguna app queda `OutOfSync` de forma persistente después de un sync exitoso, revisar si es un caso de campos inyectados en runtime (ver "Problemas conocidos" abajo) antes de asumir que algo está mal.

### Paso 5 — Verificar node-config (Karpenter, Capa 2)

```bash
kubectl get nodeclass
kubectl get nodepool

# Prueba real: forzar a Karpenter a crear un nodo
kubectl run test-karpenter \
  --image=nginx \
  --overrides='{"spec":{"nodeSelector":{"karpenter.sh/nodepool":"platform"}}}' \
  --restart=Never

kubectl get nodes -o wide -w   # debe aparecer un nodo nuevo en ~90s
kubectl get pod test-karpenter -o wide   # debe quedar Running en ese nodo

kubectl delete pod test-karpenter   # limpiar; Karpenter consolida el nodo solo tras ~5 min vacío
```

### Paso 6 — Verificar Istio (Gateway API controller)

```bash
kubectl get crd | grep gateway.networking.k8s.io   # 10 CRDs de Gateway API
kubectl get all -n istio-system                      # istiod, 2 réplicas Running

# Confirmar que istiod corre en el Node Group, no en Karpenter
kubectl get pods -n istio-system -o wide
```

No hay un objeto `Gateway` real todavía — ver siguiente sección.

---

## Qué falta para cerrar esta fase (estado al cierre de esta iteración)

### 1. Exponer el objeto `Gateway` real (bloqueante para exponer cualquier app hacia afuera)

`apps/istio/gateway.yaml` existe pero **no tiene una Application propia en la raíz de `apps/`**, así que el padre `bootstrap` no lo descubre. Falta:
- Agregar `infrastructure.parametersRef` apuntando a un `ConfigMap` con las anotaciones AWS reales (`service.beta.kubernetes.io/aws-load-balancer-scheme`, `-nlb-target-type`, subnets) — mismo patrón que ya usamos en `nodeclass-platform.yaml` con el SG real
- Crear `apps/istio-gateway-app.yaml` (o similar) en la raíz, siguiendo el mismo criterio que `node-config-app.yaml`
- Ver diagrama completo en `docs/diagrama-5-istio-gateway-api.md`

No es urgente hasta que haya un servicio de negocio real que necesite salir del cluster — cuando eso pase, cada app agrega su propio `HTTPRoute` apuntando a este mismo `Gateway` (no necesita crear infraestructura nueva).

### 2. Seguridad de Argo CD — pendiente real

- **`argocd-initial-admin-secret` sigue activo** (usuario `admin` con la contraseña generada en la instalación). Debe rotarse o eliminarse una vez exista un mecanismo de auth definitivo (SSO/OIDC, o al menos usuarios locales con roles diferenciados)
- **RBAC vacío** (`argocd-rbac-cm` → `policy.csv: ""`) — no hay roles distintos a admin. Sin esto, cualquiera con la contraseña de `admin` tiene control total
- **`AppProject default` sin restricciones** (`sourceRepos: ['*']`, `destinations: ['*']`) — cualquier Application nueva puede apuntar a cualquier repo Git y desplegar en cualquier namespace. Para un entorno más allá de PoC, conviene un `AppProject` acotado a los repos y namespaces reales que gestionamos

### 3. Notificaciones de Argo CD sin configurar

`argocd-notifications-cm` sigue con el placeholder por defecto del chart (`argocdUrl: https://argocd.example.com`), sin integración a Slack/Teams/email. Si un sync falla hoy, nadie se entera salvo que alguien entre a revisar manualmente.

### 4. Recursos sin límites explícitos

`istiod` (y otros componentes) solo tienen `requests` configurados vía Helm, sin `limits`. Vale la pena fijarlos antes de tener carga real en el cluster.

### 5. Gobierno de versiones de charts externos

`istio-base-app.yaml` e `istiod-app.yaml` apuntan a `targetRevision: "1.26.0"` fijo contra `https://blob.istio.io/istio-release/charts` (repo oficial de Istio). Fijar versión da estabilidad, pero **nada avisa automáticamente cuando sale una versión nueva** (por ejemplo, un parche de seguridad). Por ahora el proceso es manual: revisar el [changelog de Istio](https://istio.io/latest/news/releases/) periódicamente y actualizar el `targetRevision` vía PR. Evaluar Renovate/Dependabot como mejora futura cuando el pipeline de Azure DevOps esté en juego.

### 6. Repo público de prueba, credenciales para cuando migre a privado

Este repo vive hoy en GitHub público (`somospragma/kubernetes-iac-tf-bootstrap`) para la prueba. Cuando migre a un repo privado (ej. Azure DevOps), Argo CD necesitará un Secret de credenciales (SSH o PAT) — diseño completo documentado en el ítem 7 de `docs/deuda-tecnica.md`, no implementado aún.

### 7. Fuera de alcance explícito de esta fase

- Service Mesh interno (Istio Ambient Mesh — mTLS, `AuthorizationPolicy` entre servicios) — diferido a fase 2, ver ítem 8 de `docs/deuda-tecnica.md`
- External Secrets, Cert-Manager, Kyverno, External-DNS — no evaluados aún
- Pipeline de Azure DevOps para automatizar el bootstrap (`helm install argocd` + `kubectl apply bootstrap-app`) — la metodología acordada es implementar y validar todo manual primero

---

## Problemas conocidos y sus soluciones (para no perder tiempo redescubriéndolos)

### App `OutOfSync` permanente después de un sync exitoso

**Síntoma:** `kubectl get applications -n argocd` muestra `Healthy` pero `OutOfSync` de forma indefinida, incluso después de forzar refresh.

**Causa típica:** algún controlador (istiod, cert-manager, etc.) inyecta campos en runtime dentro de un `ValidatingWebhookConfiguration`/`MutatingWebhookConfiguration` (el `caBundle`, más los defaults que Kubernetes completa como `matchPolicy`, `namespaceSelector`, `scope`, `timeoutSeconds`) que nunca van a coincidir con el manifiesto que Argo CD renderizó desde Helm/Git.

**Solución:** `ignoreDifferences` con `jqPathExpressions` ignorando el array `.webhooks` completo (un `jsonPointers` apuntando solo al `caBundle` no es suficiente — quedan otros campos en diff). Ejemplo real en `apps/istiod-app.yaml`:

```yaml
ignoreDifferences:
  - group: admissionregistration.k8s.io
    kind: ValidatingWebhookConfiguration
    jqPathExpressions:
      - '.webhooks'
```

Ref: [argo-cd issue #12961](https://github.com/argoproj/argo-cd/issues/12961). Ver decisión #13 en `docs/decisiones-arquitectura.md` para el detalle completo del caso real.

### `NodeClass` bloqueado en `InstanceProfileReady: False`

Ver Paso 3.5 arriba — falta el Access Entry tipo `EC2` para el rol del NodeClass.

### Cambios en `apps/` no se reflejan en Argo CD

Argo CD hace polling cada ~3 minutos por defecto. Para forzar inmediatamente:

```bash
kubectl patch application bootstrap -n argocd --type merge \
  -p '{"metadata":{"annotations":{"argocd.argoproj.io/refresh":"hard"}}}'
```

Si el cambio afecta el propio `spec` de una Application hija (por ejemplo, agregar `ignoreDifferences`), hay que refrescar el **padre** (`bootstrap`), no solo la app hija — el padre es quien posee el manifiesto real de la Application en el cluster.

---

## Outputs necesarios de kubernetes-iac-tf-base

```bash
cd ../kubernetes-iac-tf-base

terraform output platform_nodes_sg_id    # → nodeclass-platform.yaml
terraform output iam_node_role_arn        # → nodeclass-platform.yaml (role)
terraform output service_subnet_ids      # → referencia para nodeclass
terraform output cluster_names           # → nombre del cluster
```

---

## Roadmap

```
✅ Argo CD instalado via Helm
✅ node-config: NodeClass + NodePool platform (Karpenter, Capa 2) — validado con pod de prueba
✅ Gateway API CRDs instalados 100% via Argo CD (sin kubectl apply manual)
✅ Istio (istio-base + istiod) instalado y sincronizado, corriendo en Node Group
🔜 Objeto Gateway real con infrastructure.parametersRef (subnets/SG AWS reales)
🔜 Primera app de negocio expuesta vía HTTPRoute + Gateway compartido
🔜 Seguridad de Argo CD: rotar admin secret, definir RBAC, acotar AppProject
🔜 Pipeline Azure DevOps para automatizar el bootstrap
🔜 External Secrets, Cert-Manager, Kyverno (futuro)
🔜 Service Mesh interno / Ambient Mesh (fase 2, ver deuda-tecnica.md #8)
```

---

*Chapter CloudOps — Pragma | Septiembre 2026*
