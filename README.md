# Hello World com Kubernetes local

Este projeto executa um servidor HTTP mínimo em um cluster Kubernetes local. Ele usa o **k3d**, que executa o **K3s** (uma distribuição leve do Kubernetes) dentro do Docker, e o servidor HTTP incluído no BusyBox.

A aplicação tem limite de **8 MiB** de memória. O cluster e o Docker também consomem memória: em uma medição deste ambiente, o nó do cluster usou aproximadamente 339 MiB, incluindo a aplicação. O limite do nó é 768 MiB; esses limites não significam que toda a memória esteja sempre em uso.

## Como funciona

- **Cluster**: conjunto de máquinas, chamadas nós, gerenciado pelo Kubernetes. Aqui existe um único nó, que é um contêiner Docker criado pelo `k3d`, com o controle do cluster e a aplicação no mesmo nó.
- **kubectl**: ferramenta de linha de comando que envia instruções à API do Kubernetes. O arquivo kubeconfig informa como acessá-la.
- **Pod**: unidade em que o Kubernetes executa um ou mais contêineres. Neste exemplo, o pod contém o servidor BusyBox.
- **Deployment**: mantém a quantidade desejada de pods. Aqui, uma réplica é suficiente; se ela falhar, o Kubernetes tenta substituí-la.
- **ConfigMap**: guarda o texto `Hello, World!`, disponibilizado ao contêiner como arquivo `index.html`.
- **Service**: fornece um endereço estável dentro do cluster para os pods da aplicação. O `port-forward` permite acessá-lo pelo próprio computador, sem expor uma porta na rede externa.

O arquivo [hello.yaml](hello.yaml) descreve ConfigMap, Deployment e Service. Ele **não cria o cluster**: isso é feito antes, pelo comando `k3d cluster create`.

## Pré-requisitos

São necessários Docker em execução, `k3d` e `kubectl`. Confira com:

```sh
docker info
k3d version
kubectl version --client
```

Na primeira execução, é necessário baixar as imagens do K3s e do BusyBox. Depois de baixadas, o cluster e a aplicação executam localmente. As imagens podem precisar ser baixadas novamente se forem removidas do armazenamento local. Os comandos abaixo não alteram o contexto Kubernetes padrão.

## Criar o cluster e executar

Execute na pasta deste projeto:

```sh
# Cria um cluster com um único nó, sem balanceador e componentes opcionais.
k3d cluster create hello-local --servers 1 --agents 0 --no-lb \
  --api-port 127.0.0.1:0 --kubeconfig-update-default=false \
  --kubeconfig-switch-context=false --servers-memory 768m \
  --k3s-arg '--disable=traefik,servicelb,metrics-server,local-storage,coredns@server:0'

# Salva a configuração de acesso separadamente e ajusta a porta escolhida pelo Docker.
k3d kubeconfig get hello-local > /tmp/hello-local-kubeconfig
kubectl --kubeconfig /tmp/hello-local-kubeconfig config set-cluster k3d-hello-local \
  --server="https://$(docker port k3d-hello-local-server-0 6443/tcp)"

# Cria ou atualiza os recursos e aguarda o pod ficar pronto.
kubectl --kubeconfig /tmp/hello-local-kubeconfig apply -f hello.yaml
kubectl --kubeconfig /tmp/hello-local-kubeconfig rollout status deployment/hello --timeout=120s
```

O cluster usa uma porta de API escolhida pelo Docker. O ajuste no kubeconfig é necessário neste ambiente porque o k3d registra a porta solicitada `0` em vez da porta efetivamente atribuída. Se o cluster já existe, pule a criação e gere o kubeconfig novamente a partir dos dois comandos seguintes. O arquivo em `/tmp` é temporário e pode precisar ser recriado depois de reiniciar o computador ou o cluster.

Para acessar a aplicação, mantenha este comando em execução em um terminal:

```sh
kubectl --kubeconfig /tmp/hello-local-kubeconfig port-forward --address 127.0.0.1 service/hello 8080:8080
```

Em outro terminal, execute `curl http://127.0.0.1:8080` ou abra <http://127.0.0.1:8080>. A resposta esperada é `Hello, World!`. Pressione `Ctrl+C` para encerrar o encaminhamento. Se a porta 8080 estiver ocupada, substitua `8080:8080` por `8081:8080` e acesse a porta 8081.

## Verificar

Os comandos abaixo usam o kubeconfig criado na seção anterior:

```sh
# Lista clusters k3d e mostra o nó e os recursos da aplicação.
k3d cluster list
kubectl --kubeconfig /tmp/hello-local-kubeconfig get nodes
kubectl --kubeconfig /tmp/hello-local-kubeconfig get deployments,pods,services,configmaps

# Mostra detalhes, eventos e logs da aplicação.
kubectl --kubeconfig /tmp/hello-local-kubeconfig describe deployment hello
kubectl --kubeconfig /tmp/hello-local-kubeconfig describe pods -l app=hello
kubectl --kubeconfig /tmp/hello-local-kubeconfig logs deployment/hello

# Mostra o consumo do nó inteiro, incluindo Kubernetes e aplicação.
docker stats --no-stream k3d-hello-local-server-0
```

O `metrics-server` foi desativado para economizar memória; por isso, `kubectl top` não funciona sem instalá-lo. O BusyBox pode não produzir logs em uma requisição normal.

## Parar, reiniciar e recriar

```sh
# Para o cluster e, mais tarde, inicia o mesmo cluster novamente.
k3d cluster stop hello-local
k3d cluster start hello-local

# Remove somente os recursos da aplicação; o cluster continua existindo.
kubectl --kubeconfig /tmp/hello-local-kubeconfig delete -f hello.yaml
# Para criá-los novamente:
kubectl --kubeconfig /tmp/hello-local-kubeconfig apply -f hello.yaml

# Remove o cluster e seus recursos. Para recriá-lo, execute a seção de criação.
k3d cluster delete hello-local
```

Depois de reiniciar o cluster, gere e ajuste o kubeconfig novamente conforme a seção de criação, pois a porta da API pode mudar. Para acessar a aplicação, inicie novamente o `port-forward`.
