# Guia de Setup - B(A)C(K) Stack: Backstage, ArgoCD, Crossplane, Kyverno

Este guia demonstra como configurar um cluster Kubernetes local usando Kind, com NGINX Gateway Fabric para roteamento e ArgoCD para GitOps, incluindo a instalação automática da stack completa: **Crossplane**, **Crossview**, **Kyverno** e **LocalStack**.

## Arquitetura

- **Cluster Kind**: Kubernetes v1.35.0 com port mappings para acesso externo
- **NGINX Gateway Fabric**: Implementação do Gateway API para roteamento HTTP/HTTPS
- **ArgoCD**: Ferramenta de GitOps para deployment contínuo
- **Crossplane**: Plataforma de infraestrutura como código
- **Crossview**: Interface de visualização para recursos Crossplane
- **Kyverno**: Gerenciamento de políticas nativo do Kubernetes
- **LocalStack**: Emulação completa de serviços AWS localmente
- **Domínio**: `*.kleste.lab` (configurar no `/etc/hosts` se necessário)

## Pré-requisitos

- Kind instalado
- kubectl configurado
- Helm v3+
- Repositório Argo Helm adicionado: `helm repo add argo https://argoproj.github.io/argo-helm`
- Token do GitHub (para autenticação do ArgoCD)

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

# Atualizar repositório Helm (necessário se você já tinha o repo configurado)
helm repo update

# Instalar ArgoCD com configurações customizadas
helm install argocd argo/argo-cd --version 9.3.7 -n argocd -f argocd/values.yaml
```

**O que faz**:
- Cria namespace `argocd` com acesso ao gateway compartilhado
- Atualiza o índice do repositório Helm para garantir acesso às versões mais recentes
- Instala ArgoCD v9.3.7 com configurações para funcionar atrás de proxy
- Habilita modo `--insecure` para SSL termination no gateway

> **⚠️ Nota**: Se você receber o erro `chart "argo-cd" matching 9.3.7 not found`, execute `helm repo update` antes da instalação para atualizar o índice do repositório.

## 6. Configurando Rotas do ArgoCD

```bash
kubectl apply -f argocd/argocd-routes.yaml
```

**O que faz**: 
- Cria HTTPRoute para expor ArgoCD em `argocd.kleste.lab`
- Roteia tráfego do gateway para o serviço ArgoCD na porta 443

## 7. Bootstrap da Stack B(A)C(K)

### 7.1 Configurando Credenciais do GitHub

```bash
# Copie o template de secret
cp argocd/github-secret.yaml.example argocd/github-secret.yaml

# Edite o arquivo e substitua <GITHUB_TOKEN> pelo seu token do GitHub
# Token pode ser gerado em: https://github.com/settings/tokens
nano argocd/github-secret.yaml

# Aplique a secret
kubectl apply -f argocd/github-secret.yaml
```

**O que faz**:
- Cria credenciais para o ArgoCD acessar este repositório Git
- Permite sincronização automática dos addons

### 7.2 Aplicando Bootstrap

```bash
kubectl apply -f argocd/bootstrap.yaml
```

**O que faz**:
- Cria o AppProject `back-stack` com permissões para todos os repositórios necessários
- Cria o Application `addons` que instala automaticamente todos os componentes:
  - **Crossplane** v2.2.0 (plataforma de infraestrutura)
  - **Crossview** v3.5.3 (interface de visualização)
  - **Kyverno** v1.17.1 (gerenciamento de políticas)
  - **LocalStack** v0.7.0 (emulação AWS)

## 8. Acessando os Serviços

### Configuração de DNS Local
Adicione ao `/etc/hosts` (Linux/Mac) ou `C:\Windows\System32\drivers\etc\hosts` (Windows):
```
127.0.0.1 argocd.kleste.lab
```

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

## 9. Limpeza do Ambiente

```bash
kind delete cluster --name cluster-hub
```

## Estrutura de Arquivos

```
├── .gitignore                    # Exclusões do Git (inclui secrets)
├── addons/                       # ApplicationSets dos addons
│   ├── crossplane/
│   │   ├── crossplane-applicationset.yaml
│   │   └── crossview-applicationset.yaml
│   ├── kyverno/
│   │   └── kyverno-applicationset.yaml
│   └── localstack/
│       └── localstack-applicationset.yaml
├── argocd/
│   ├── bootstrap.yaml           # Bootstrap Application + AppProject
│   ├── github-secret.yaml.example # Template de secret do GitHub
│   ├── values.yaml              # Configurações customizadas do ArgoCD
│   └── argocd-routes.yaml       # HTTPRoute para exposição do ArgoCD
├── kind/
│   ├── control-plane.yaml       # Configuração do cluster Kind
│   └── default-gateway.yaml     # Gateway compartilhado e namespace
└── README.md                    # Este guia
```

## Componentes da Stack B(A)C(K)

### Crossplane
- **Versão**: 2.2.0
- **Namespace**: crossplane-system
- **Função**: Plataforma de infraestrutura como código
- **Documentação**: https://docs.crossplane.io/

### Crossview
- **Versão**: 3.5.3
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
