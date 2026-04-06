# Guia de Setup - B(ackstage) A(rgoCD) C(rossplane) K(yverno) Stack

Este guia demonstra como configurar um cluster Kubernetes local com Kind e ArgoCD, e como instalar a stack completa via ApplicationSets:
- Crossplane + Crossview
- Kyverno
- Ministack (emulador AWS local)
- NGINX Gateway Fabric (Gateway API + shared gateway)
- Crossplane AWS Upbound Provider (S3)

## Arquitetura

- **Cluster Kind**: Kubernetes v1.35.0
- **ArgoCD**: GitOps controller (App-of-Apps pattern)
- **NGINX Gateway Fabric**: Ingress/gateway para domínio `*.kleste.lab`
- **Stack**: Crossplane, Crossview, Kyverno, Ministack
- **Crossplane AWS Provider**: provider-family-aws + provider-aws-s3 v2.5.0 com endpoint no Ministack

## Pré-requisitos

- Kind instalado
- kubectl configurado
- Helm v3+
- Repositório Argo Helm adicionado: `helm repo add argo https://argoproj.github.io/argo-helm`
- Token do GitHub (para autenticação do ArgoCD)

## 0. Pré-configuração de DNS Local

Antes de iniciar, adicione ao `/etc/hosts` (Linux/Mac) ou `C:\Windows\System32\drivers\etc\hosts` (Windows):
```
127.0.0.1 argocd.kleste.lab
127.0.0.1 crossview.kleste.lab
```

## 1. Criando o Cluster Kind

```bash
kind create cluster --config kind/control-plane.yaml
```

**O que faz**: Cria um cluster chamado `cluster-hub` com:
- Kubernetes v1.35.0
- Port mappings: 8080→31437 (HTTP), 8443→30478 (HTTPS), 30000→30000

## 2. Instalando ArgoCD

```bash
# Criar namespace argocd
kubectl create ns argocd

# Label para permitir acesso via gateway compartilhado
kubectl label namespace argocd shared-gateway-access="true" --overwrite

# Atualizar repositório Helm
helm repo update

# Instalar ArgoCD com configurações customizadas
helm install argocd argo/argo-cd --version 9.3.7 -n argocd -f argocd/values.yaml
```

## 3. Configurar Secret do GitHub para ArgoCD

```bash
cp argocd/github-secret.yaml.example argocd/github-secret.yaml
# Edite argocd/github-secret.yaml, colocando seu token em password
kubectl apply -f argocd/github-secret.yaml
```

## 4. Aplicar Bootstrap para instalar addons

```bash
kubectl apply -f argocd/bootstrap.yaml
```

**O que faz**:
- Cria AppProject `back-stack`
- Cria Application `addons` que aplica recursivamente os manifests da pasta `back-stack/`
- Instala Crossplane, Crossview, Kyverno, Ministack e NGINX Gateway Fabric

### 4.1 O que está automatizado pelo bootstrap

O Application `addons` aplica automaticamente tudo em `back-stack/`:

| Componente | Tipo | Descrição |
|---|---|---|
| `nginx-gateway-fabric/gateway-api-crds.yaml` | Manifest | CRDs do Gateway API |
| `nginx-gateway-fabric/nginx-gateway-fabric-applicationset.yaml` | ApplicationSet | NGINX Gateway Fabric via Helm |
| `nginx-gateway-fabric/shared-gateway.yaml` | Manifest | Gateway compartilhado `*.kleste.lab` |
| `crossplane/crossplane-applicationset.yaml` | ApplicationSet | Crossplane via Helm |
| `crossplane/crossview-applicationset.yaml` | ApplicationSet | Crossview via Helm |
| `crossplane/crossview-namespace.yaml` | Manifest | Namespace crossview com label do gateway |
| `crossplane/crossview-routes.yaml` | Manifest | HTTPRoute para crossview.kleste.lab |
| `crossplane/aws-ministack-credentials.yaml` | Manifest | Secret com credenciais dummy para Ministack |
| `crossplane/crossplane-aws-upbound-applicationset.yaml` | ApplicationSet | Provider AWS S3 via chart local |
| `crossplane/xrds/s3-xrd.yaml` | Manifest | XRD para S3Bucket |
| `crossplane/compositions/s3-composition.yaml` | Manifest | Composition S3 (Pipeline mode) |
| `crossplane/functions/function-patch-and-transform.yaml` | Manifest | Function para patches |
| `crossplane/claims/s3-claim.yaml` | Manifest | Claim de teste (bucket `test-bucket-crossplane`) |
| `kyverno/kyverno-applicationset.yaml` | ApplicationSet | Kyverno via Helm |
| `ministack/` | Manifests | Namespace, Deployment e Service do Ministack |
| `argocd/argocd-routes.yaml` | Manifest | HTTPRoute para argocd.kleste.lab |

## 5. Patch do CoreDNS para compatibilidade com Ministack (S3 Control)

O Crossplane AWS provider v2.x usa a S3 Control API para gerenciar tags, que constrói URLs com o account ID como subdomínio (`000000000000.ministack.ministack.svc.cluster.local`). É necessário adicionar uma regra de rewrite no CoreDNS para redirecionar esse tráfego ao serviço do Ministack:

```bash
kubectl get configmap coredns -n kube-system -o json | python3 -c "
import json, sys
cm = json.load(sys.stdin)
corefile = cm['data']['Corefile']
rewrite = '    rewrite name regex (.*)\\\\.ministack\\\\.ministack\\\\.svc\\\\.cluster\\\\.local ministack.ministack.svc.cluster.local\n'
corefile = corefile.replace('    kubernetes cluster.local', rewrite + '    kubernetes cluster.local')
cm['data']['Corefile'] = corefile
print(json.dumps(cm))
" | kubectl apply -f -

kubectl rollout restart deployment coredns -n kube-system
kubectl rollout status deployment coredns -n kube-system --timeout=60s
```

**Por que é necessário**: O provider-family-aws v2.x usa o Terraform AWS provider v5.x que passou a gerenciar tags de bucket via S3 Control API com routing por account ID no subdomínio. O Ministack não suporta esse padrão de DNS, então o CoreDNS intercepta e redireciona para o serviço correto.

## 6. Acessando os Serviços

### ArgoCD
- **URL**: http://argocd.kleste.lab:8080
- **Usuário**: admin
- **Senha**: `kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d`

### Crossview
- **URL**: http://crossview.kleste.lab:8080
- **Usuário**: admin
- **Senha**: `ChangeThisImmediately2026!` (configurado no ApplicationSet)

### Ministack
- **Namespace**: ministack
- **Endpoint in-cluster**: `http://ministack.ministack.svc.cluster.local:4566`
- **Acesso externo**: `kubectl port-forward -n ministack svc/ministack 4566:4566`
- **Credenciais**: `AWS_ACCESS_KEY_ID=test` / `AWS_SECRET_ACCESS_KEY=test`

### Crossplane S3 (validação)

```bash
# Verificar estado do bucket de teste
kubectl get s3bucket -n default
kubectl get bucket.s3.aws.upbound.io

# Verificar logs do provider
kubectl logs -n crossplane-system -l pkg.crossplane.io/revision --tail=50 | grep -i "error\|bucket"
```

## Limpeza do Ambiente

```bash
kind delete cluster --name cluster-hub
```

## Estrutura de Arquivos

```
├── argocd/
│   ├── bootstrap.yaml                    # Bootstrap Application + AppProject
│   ├── github-secret.yaml.example        # Template de secret do GitHub
│   └── values.yaml                       # Configurações customizadas do ArgoCD
├── back-stack/                           # Manifests gerenciados pelo ArgoCD (recurse: true)
│   ├── argocd/
│   │   └── argocd-routes.yaml            # HTTPRoute para exposição do ArgoCD
│   ├── crossplane/
│   │   ├── aws-ministack-credentials.yaml # Secret com credenciais dummy
│   │   ├── crossplane-applicationset.yaml
│   │   ├── crossview-applicationset.yaml  # Crossview 3.8.0
│   │   ├── crossview-namespace.yaml       # Namespace com label do gateway
│   │   ├── crossview-routes.yaml          # HTTPRoute para crossview.kleste.lab
│   │   ├── crossplane-aws-upbound-applicationset.yaml
│   │   ├── claims/s3-claim.yaml           # Claim de teste S3
│   │   ├── compositions/s3-composition.yaml
│   │   ├── functions/function-patch-and-transform.yaml
│   │   └── xrds/s3-xrd.yaml
│   ├── kyverno/
│   │   └── kyverno-applicationset.yaml
│   ├── ministack/                         # Ministack (substitui LocalStack)
│   │   ├── namespace.yaml
│   │   ├── deployment.yaml
│   │   └── service.yaml
│   └── nginx-gateway-fabric/
│       ├── gateway-api-crds.yaml
│       ├── nginx-gateway-fabric-applicationset.yaml
│       └── shared-gateway.yaml
├── charts/
│   └── crossplane-aws-upbound/           # Helm chart local para o provider AWS
└── kind/
    └── control-plane.yaml                # Configuração do cluster Kind
```

## Componentes da Stack

### NGINX Gateway Fabric
- **Versão**: 2.4.2 | **Namespace**: nginx-gateway
- Gateway compartilhado para `*.kleste.lab` via label `shared-gateway-access=true`

### Crossplane
- **Versão**: 2.2.0 | **Namespace**: crossplane-system

### Crossview
- **Versão**: 3.8.0 | **Namespace**: crossview | **URL**: http://crossview.kleste.lab:8080
- Chart: `https://crossplane-contrib.github.io/crossview`

### Kyverno
- **Versão**: 3.7.1 (chart) | **Namespace**: kyverno

### Ministack
- **Imagem**: `nahuelnucera/ministack:latest` | **Namespace**: ministack | **Porta**: 4566

### Crossplane AWS Provider
- **provider-family-aws**: v2.5.0 | **provider-aws-s3**: v2.5.0
- Endpoint: `http://ministack.ministack.svc.cluster.local:4566`
- Path-style S3, skip credential/region/metadata validation

## Troubleshooting

- **Gateway não responde**: Verifique se as portas 8080 e 8443 estão livres no host
- **ArgoCD/Crossview inacessível**: Confirme se o namespace tem a label `shared-gateway-access=true`
- **DNS não resolve**: Adicione entradas no `/etc/hosts` (seção 0)
- **Bootstrap falha**: Verifique se o token do GitHub está correto e o repositório acessível
- **Bucket S3 não fica Ready**: Verifique se o patch do CoreDNS foi aplicado (seção 5)
- **Provider AWS não conecta ao Ministack**: Confirme se `AWS_ENDPOINT_URL` está nos logs do pod do provider

### Acesso local ao ArgoCD (fallback)

```bash
kubectl port-forward -n argocd svc/argocd-server 8080:443
# Acesse https://localhost:8080 (aceite certificado inseguro)
```
