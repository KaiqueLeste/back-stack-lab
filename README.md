# Guia de Setup - Kind Cluster com ArgoCD e NGINX Gateway Fabric

Este guia demonstra como configurar um cluster Kubernetes local usando Kind, com NGINX Gateway Fabric para roteamento e ArgoCD para GitOps.

## Arquitetura

- **Cluster Kind**: Kubernetes v1.35.0 com port mappings para acesso externo
- **NGINX Gateway Fabric**: Implementação do Gateway API para roteamento HTTP/HTTPS
- **ArgoCD**: Ferramenta de GitOps para deployment contínuo
- **Domínio**: `*.kleste.lab` (configurar no `/etc/hosts` se necessário)

## Pré-requisitos

- Kind instalado
- kubectl configurado
- Helm v3+
- Repositório Argo Helm adicionado: `helm repo add argo https://argoproj.github.io/argo-helm`

## 1. Criando o Cluster Kind

```bash
kind create cluster --config kind/control-plane.yaml
```

**O que faz**: Cria um cluster chamado `cluster-hub` com:
- Kubernetes v1.35.0
- Port mappings: 8080→31437 (HTTP), 8443→30478 (HTTPS), 30000→30000

## 2. Instalando Gateway API CRDs

```bash
kubectl kustomize "https://github.com/nginx/nginx-gateway-fabric/config/crd/gateway-api/standard?ref=v2.3.0" | kubectl apply -f -
```

**O que faz**: Instala os Custom Resource Definitions necessários para o Gateway API.

## 3. Instalando NGINX Gateway Fabric

```bash
helm install ngf oci://ghcr.io/nginx/charts/nginx-gateway-fabric \
  --create-namespace -n nginx-gateway \
  --set nginx.service.type=NodePort \
  --set-json 'nginx.service.nodePorts=[{"port":31437,"listenerPort":80}, {"port":30478,"listenerPort":8443}]'
```

**O que faz**: 
- Instala o NGINX Gateway Fabric no namespace `nginx-gateway`
- Configura NodePort para expor serviços nas portas mapeadas no Kind
- Mapeia porta 80→31437 e 8443→30478

## 4. Criando Gateway Compartilhado

```bash
kubectl apply -f kind/default-gateway.yaml
```

**O que faz**:
- Cria namespace `gateway-infrastructure` 
- Configura Gateway compartilhado para domínio `*.kleste.lab`
- Permite acesso de namespaces com label `shared-gateway-access=true`

## 5. Instalando ArgoCD

```bash
# Criar namespace e configurar acesso ao gateway
kubectl create ns argocd
kubectl label namespace argocd shared-gateway-access="true" --overwrite

# Instalar ArgoCD com configurações customizadas
helm install argocd argo/argo-cd --version 9.3.7 -n argocd -f argocd/values.yaml
```

**O que faz**:
- Cria namespace `argocd` com acesso ao gateway compartilhado
- Instala ArgoCD v9.3.7 com configurações para funcionar atrás de proxy
- Habilita modo `--insecure` para SSL termination no gateway

## 6. Configurando Rotas do ArgoCD

```bash
kubectl apply -f argocd/argocd-routes.yaml
```

**O que faz**: 
- Cria HTTPRoute para expor ArgoCD em `argocd.kleste.lab`
- Roteia tráfego do gateway para o serviço ArgoCD na porta 443

## 7. Acessando os Serviços

### ArgoCD
- **URL**: http://argocd.kleste.lab:8080
- **Usuário**: admin
- **Senha**: `kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d`

### Configuração de DNS Local
Adicione ao `/etc/hosts` (Linux/Mac) ou `C:\Windows\System32\drivers\etc\hosts` (Windows):
```
127.0.0.1 argocd.kleste.lab
```

## 8. Limpeza do Ambiente

```bash
kind delete cluster --name cluster-hub
```

## Estrutura de Arquivos

```
├── kind/
│   ├── control-plane.yaml    # Configuração do cluster Kind
│   └── default-gateway.yaml  # Gateway compartilhado e namespace
├── argocd/
│   ├── values.yaml          # Configurações customizadas do ArgoCD
│   └── argocd-routes.yaml   # HTTPRoute para exposição do ArgoCD
└── guia.md                  # Este guia
```

## Troubleshooting

- **Gateway não responde**: Verifique se as portas 8080 e 8443 estão livres no host
- **ArgoCD inacessível**: Confirme se o namespace tem a label `shared-gateway-access=true`
- **DNS não resolve**: Adicione entrada no `/etc/hosts` conforme seção "Configuração de DNS Local"