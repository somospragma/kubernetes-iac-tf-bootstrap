# kubernetes-iac-tf-bootstrap — Fase 2: Bootstrap del Cluster EKS

**Alcance:** Bootstrap de componentes de plataforma sobre el cluster desplegado en `kubernetes-iac-tf-base`.

**Pre-requisito:** Cluster EKS activo con Node Group del sistema y los 5 addons base instalados.

---

## Componentes incluidos en esta versión

| Componente | Descripción |
|---|---|
| Argo CD | GitOps engine — gestiona todos los demás componentes |
| node-config | NodeClass + NodePool para nodos Karpenter (Capa 2) |
| Istio | Service mesh con Kubernetes Gateway API |

---

## Estructura

```
bootstrap/
├── argocd/
│   └── values.yaml        ← Helm values para instalar Argo CD
└── bootstrap-app.yaml     ← App of Apps raíz (se aplica una sola vez)

apps/
├── node-config/           ← NodeClass + NodePool platform
│   ├── application.yaml
│   ├── nodeclass-platform.yaml
│   └── nodepool-platform.yaml
└── istio/                 ← Istio con Kubernetes Gateway API
    └── application.yaml   ← (valores se agregan en siguiente iteración)
```

---

## Paso 1 — Instalar Argo CD

```bash
# Agregar repo de Helm
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

# Instalar Argo CD en el cluster
helm upgrade --install argocd argo/argo-cd \
  --namespace argocd \
  --create-namespace \
  --version 10.7.1 \
  -f bootstrap/argocd/values.yaml

# Esperar que todos los pods estén Ready
kubectl get pods -n argocd -w
```

## Paso 2 — Acceder a Argo CD

```bash
# Port-forward al server de Argo CD
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Obtener la contraseña inicial del admin
kubectl get secret argocd-initial-admin-secret \
  -n argocd \
  -o jsonpath="{.data.password}" | base64 -d && echo

# Abrir en el browser: https://localhost:8080
# Usuario: admin
# Password: el valor del comando anterior
```

## Paso 3 — Actualizar el SG ID en node-config

Antes de aplicar el bootstrap-app, actualizar el SG real en el NodeClass:

```bash
# Obtener el SG ID de los nodos Karpenter
cd ../kubernetes-iac-tf-base
terraform output platform_nodes_sg_id

# Editar apps/node-config/nodeclass-platform.yaml
# Reemplazar PLACEHOLDER_SG_ID con el ID real
```

## Paso 4 — Aplicar el Bootstrap App of Apps

```bash
# Configurar kubeconfig si no está listo
aws eks update-kubeconfig \
  --region us-east-1 \
  --name pragma-sopp-dev-eks-main \
  --profile Sopp_Core_PoC

# Aplicar el App of Apps raíz (una sola vez)
kubectl apply -f bootstrap/bootstrap-app.yaml

# Verificar que Argo CD detectó las apps
kubectl get applications -n argocd
```

## Paso 5 — Verificar node-config

```bash
# Verificar NodeClass y NodePool creados
kubectl get nodeclass
kubectl get nodepool

# Desplegar un pod de prueba para verificar que Karpenter crea nodos
kubectl run test-karpenter \
  --image=nginx \
  --overrides='{"spec":{"nodeSelector":{"karpenter.sh/nodepool":"platform"}}}' \
  --restart=Never

# Ver el nodo que crea Karpenter
kubectl get nodes -o wide -w

# Limpiar
kubectl delete pod test-karpenter
```

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
✅ node-config: NodeClass + NodePool platform
🔜 Istio con Kubernetes Gateway API (próxima iteración)
🔜 Pipeline Azure DevOps para automatizar el bootstrap
🔜 External Secrets, Cert-Manager, Kyverno (futuro)
```

---

*Chapter CloudOps — Pragma | Septiembre 2026*
