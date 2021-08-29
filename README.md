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
# Instalar traefik
referencia:
 * https://github.com/traefik/traefik-helm-chart
 * https://medium.com/dev-genius/quickstart-with-traefik-v2-on-kubernetes-e6dff0d65216

1.Adicione o repo do traefik e atualize
```bash
helm repo add traefik https://helm.traefik.io/traefik
helm repo update
```

2.Edite o arquivo <b>traefik-values.sample.yaml</b> com seu email

3.Acesse o [painel](https://dash.cloudflare.com/profile/api-tokens) de tokens do cloudflare e consiga seu token

4.Gere um secret no kubernetes com seu token
```bash
kubectl create secret generic cloudflare --from-literal=dns-token=<my-cloudflare-token-here>
```

5.Instale o traefik passando as configurações
```bash
helm install traefik traefik/traefik -f traefik-values.yaml
```

Verifique se o traefik está funcionando
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
        pathType: Prefix
        backend:
          service:
            name: whoami
            port: 
              number: 80
EOF
```

Nesse exemplo vamos export o serviço nas duas portas: 80 e 443. E podera ser acessado pelo localhost:80 e pelo localhost:443.