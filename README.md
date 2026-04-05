# Guia de Setup - B(ackstage) A(rgoCD) C(rossplane) K(yverno) Stack

Este guia demonstra como configurar um cluster Kubernetes local com Kind e ArgoCD, e como instalar a stack completa via ApplicationSets:
- Crossplane + Crossview
- Kyverno
- LocalStack
- NGINX Gateway Fabric (Gateway API + shared gateway)

## Arquitetura

- **Cluster Kind**: Kubernetes v1.35.0
- **ArgoCD**: GitOps controller
- **NGINX Gateway Fabric**: Ingress/gateway
- **Stack**: Crossplane, Crossview, Kyverno, LocalStack

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

# Label para permitir acesso, caso use o gateway compartilhado (opcional)
kubectl label namespace argocd shared-gateway-access="true" --overwrite

# Atualizar repositório Helm
helm repo update

# Instalar ArgoCD com configurações customizadas
helm install argocd argo/argo-cd --version 9.3.7 -n argocd -f argocd/values.yaml
```

**O que faz**:
- Cria namespace `argocd`
- Instala ArgoCD no cluster

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
- Cria Application `back-stack` que aplica os manifests da pasta `back-stack/`
- Instala Crossplane, Crossview, Kyverno, LocalStack e NGINX Gateway Fabric

### 4.1 O que está automatizado pelo bootstrap

O Application `back-stack` aplica automaticamente:

- `back-stack/nginx-gateway-fabric/gateway-api-crds.yaml`
- `back-stack/nginx-gateway-fabric/nginx-gateway-fabric-applicationset.yaml`
- `back-stack/nginx-gateway-fabric/shared-gateway.yaml`
- `back-stack/crossplane/crossplane-applicationset.yaml`
- `back-stack/crossplane/crossview-applicationset.yaml`
- `back-stack/kyverno/kyverno-applicationset.yaml`
- `back-stack/localstack/localstack-applicationset.yaml`
- `back-stack/argocd/argocd-routes.yaml`

## 5. Acessando os Serviços

### ArgoCD
- **URL**: http://argocd.kleste.lab:8080
- **Usuário**: admin
- **Senha**: `kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d`

### Crossview (opcional - requer configuração adicional)
- **Namespace**: crossview
- **Acesso**: Via port-forward ou ingress (não configurado por padrão)

### LocalStack
- **Namespace**: localstack
- **Porta**: 4566 (ClusterIP)
- **Acesso**: Via port-forward: `kubectl port-forward -n localstack svc/localstack 4566:4566`

## Limpeza do Ambiente

```bash
kind delete cluster --name cluster-hub
```

## Estrutura de Arquivos

```
├── .gitignore                    # Exclusões do Git (inclui secrets)
├── argocd/
│   ├── bootstrap.yaml           # Bootstrap Application + AppProject
│   ├── github-secret.yaml.example # Template de secret do GitHub
│   └── values.yaml              # Configurações customizadas do ArgoCD
├── back-stack/                  # Manifests gerenciados pelo ArgoCD
│   ├── argocd/
│   │   └── argocd-routes.yaml   # HTTPRoute para exposição do ArgoCD
│   ├── crossplane/
│   │   ├── crossplane-applicationset.yaml
│   │   └── crossview-applicationset.yaml
│   ├── kyverno/
│   │   └── kyverno-applicationset.yaml
│   ├── localstack/
│   │   └── localstack-applicationset.yaml
│   └── nginx-gateway-fabric/    # NGINX Gateway Fabric e Gateway API
│       ├── gateway-api-crds.yaml
│       ├── nginx-gateway-fabric-applicationset.yaml
│       └── shared-gateway.yaml
├── kind/
│   └── control-plane.yaml       # Configuração do cluster Kind
└── README.md                    # Este guia
```

## Componentes da Stack B(A)C(K)

### NGINX Gateway Fabric
- **Versão**: 2.4.2
- **Namespace**: nginx-gateway
- **Função**: Gateway API implementation para roteamento HTTP/HTTPS
- **Documentação**: https://docs.nginx.com/nginx-gateway-fabric/

### Gateway API CRDs
- **Versão**: v2.3.0
- **Namespace**: gateway-api
- **Função**: Custom Resource Definitions para Gateway API
- **Fonte**: https://github.com/nginx/nginx-gateway-fabric

### Shared Gateway
- **Namespace**: gateway-infrastructure
- **Função**: Gateway compartilhado para domínio *.kleste.lab
- **Configuração**: Permite acesso via label `shared-gateway-access=true`

### Crossplane
- **Versão**: 2.2.0
- **Namespace**: crossplane-system
- **Função**: Plataforma de infraestrutura como código
- **Documentação**: https://docs.crossplane.io/

### Crossview
- **Versão**: 3.6.0
- **Namespace**: crossview
- **Função**: Interface web para visualização de recursos Crossplane
- **Documentação**: https://github.com/crossplane-contrib/crossview

### Kyverno
- **Versão**: 1.17.1
- **Namespace**: kyverno
- **Função**: Gerenciamento de políticas nativo do Kubernetes
- **Documentação**: https://kyverno.io/

### LocalStack
- **Versão**: 0.7.0
- **Namespace**: localstack
- **Função**: Emulação completa de serviços AWS localmente
- **Documentação**: https://docs.localstack.cloud/

## Troubleshooting

- **Gateway não responde**: Verifique se as portas 8080 e 8443 estão livres no host
- **ArgoCD inacessível**: Confirme se o namespace tem a label `shared-gateway-access=true`
- **DNS não resolve**: Adicione entrada no `/etc/hosts` conforme seção "Configuração de DNS Local"
- **Bootstrap falha**: Verifique se o token do GitHub está correto e se o repositório é acessível
- **Addons não sincronizam**: Confirme se o Application `addons` está em estado Healthy no ArgoCD

### Acesso local ao ArgoCD (fallback)

Se o Gateway API não estiver funcionando ou para debug local, use `kubectl port-forward`:

```bash
kubectl port-forward -n argocd svc/argocd-server 8080:443
# Acesse https://localhost:8080 (aceite certificado inseguro)
```

Alternativa via proxy:

```bash
kubectl proxy --port=8001
# Acesse:
# http://localhost:8001/api/v1/namespaces/argocd/services/https:argocd-server:/proxy/
```
