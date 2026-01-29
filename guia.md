Criando cluster kind:

kind create cluster --config control-plane.yaml

Criando CRDs do gateway api:
kubectl kustomize "https://github.com/nginx/nginx-gateway-fabric/config/crd/gateway-api/standard?ref=v2.3.0" | kubectl apply -f -

Instalando o nginx gateway fabric:

helm install ngf oci://ghcr.io/nginx/charts/nginx-gateway-fabric --create-namespace -n nginx-gateway --set nginx.service.type=NodePort --set-json 'nginx.service.nodePorts=[{"port":31437,"listenerPort":80}, {"port":30478,"listenerPort":8443}]'

Criando o gateway default:

kubectl apply -f kind/default-gateway.yaml

Instalando o argocd:

kubectl create ns argocd
kubectl label namespace argocd shared-gateway-access="true" --overwrite
helm install argocd argo/argo-cd --version 9.3.7 -n argocd -f argocd/values.yaml

Aplicando as rotas para o argo:

kubectl apply -f argocd/argocd-routes.yaml

limpando o ambiente:

kind delete cluster --name cluster-hub