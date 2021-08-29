# Configurar cluster kubernetes local
referencia: https://codeburst.io/creating-a-local-development-kubernetes-cluster-with-k3s-and-traefik-proxy-7a5033cb1c2d

# k3d
k3d é um helper que permite rodar o k3s (cluster kubernetes leve) dentro de um container docker.

## Instalação
### linux:
```bash
curl -s https://raw.githubusercontent.com/rancher/k3d/main/install.sh | bash
```
### windows:
```powershell
choco install k3d
```

## Criar cluster
```bash
k3d cluster create devcluster \
--api-port 127.0.0.1:6443 \
-p 80:80@loadbalancer \
-p 443:443@loadbalancer \
--k3s-server-arg "--no-deploy=traefik"
```

onde:
- `--api-port`: porta de acesso ao cluster
- `-p`: porta de acesso ao cluster
- `--k3s-server-arg`: argumento do k3s

As portas 80 e 443 vão ser mapeadas para o loadbalancer virtual, permitindo que os recursos de ingress sejam acessados diretamente da maquina pelo localhost. E o cluster sera criado sem o traefik padrão (1.0) para que seja instalado sua versão mais recente.

## Salvar configuração
Salvar configurações do cluster em arquivo e exportar para variável:
```bash
k3d kubeconfig get devcluster > kubeconfig
export KUBECONFIG=kubeconfig
```

## Instalar Helm
### linux:
```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3

chmod 700 get_helm.sh

./get_helm.sh
```
### windows:
```powershell
choco install kubernetes-helm
```
## Instalar traefik

```bash
helm repo add traefik https://containous.github.io/traefik-helm-chart
helm install traefik traefik/traefik
```

Verifiqye se o traefik está funcionando
```bash
kubectl port-forward $(kubectl get pods --selector "app.kubernetes.io/name=traefik" --output=name) 9000:9000
```
Acessar: http://localhost:9000/dashboard


## Deploy de uma aplicação

Um deploy de uma aplicação simples para validar o Ingress Controller

```bash
kubectl create deploy whoami --image containous/whoami

kubectl expose deploy whoami --port 80
```

E então  fazer uso do traefik definindo uma regra de ingress para a aplicação:
```bash
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: whoami
  annotations:
    traefik.ingress.kubernetes.io/router.entrypoints: web,websecure
    traefik.ingress.kubernetes.io/router.tls: "true"
spec:
  rules:
  - http:
      paths:
      - path: /
        backend:
          serviceName: whoami
          servicePort: 80
EOF
```

Nesse exemplo vamos export o serviço nas duas portas: 80 e 443. E podera ser acessado pelo localhost:80 e pelo localhost:443.