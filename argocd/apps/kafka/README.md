# Kafka ArgoCD Deployment

Esta configuração implementa um deployment completo do Apache Kafka usando Strimzi no ArgoCD com o padrão "App of Apps" para garantir a ordem correta de execução.

## Estrutura

- `kafka-dev.yaml` - Application "pai" (App of Apps)
- `applications/strimzi-operator-application.yaml` - Application do Strimzi Operator
- `applications/kafka-cluster-application.yaml` - Application do Kafka Cluster
- `applications/kafka-ingress-application.yaml` - Application do Ingress
- `resources/kafka-cluster.yaml` - Definição do cluster Kafka (KafkaNodePool + Kafka)
- `resources/kafka-ingress.yaml` - Ingress para exposição externa

## Ordem de Execução

1. **Strimzi Operator (sync-wave: 1)** - Instala o operador Strimzi via Helm Chart
2. **Kafka Cluster (sync-wave: 2)** - Cria o cluster Kafka usando CRDs do Strimzi
3. **Kafka Ingress (sync-wave: 3)** - Cria o Ingress para exposição externa

## Funcionalidades

### Kafka Cluster
- **1 replica** (adequado para desenvolvimento)
- **KRaft mode** (sem Zookeeper)
- **NodePools** habilitado para melhor gerenciamento
- **Armazenamento persistente** (100Gi por volume)
- **Múltiplos listeners**:
  - Internal PLAINTEXT (9092)
  - Internal TLS (9093)
  - External NodePort PLAINTEXT (9094 → 31094)
  - External NodePort TLS (9095 → 31096)

### Configuração para Desenvolvimento
- Replication factor: 1
- Min ISR: 1
- Single broker setup

## Como Usar

### Pré-requisitos

1. ArgoCD instalado e configurado
2. Adicionar o repositório Helm do Strimzi:
   ```bash
   argocd repo add https://strimzi.io/charts/ --type helm --name strimzi
   ```

### Deploy

Para fazer o deploy completo, basta aplicar o Application "pai":

```bash
kubectl apply -f https://raw.githubusercontent.com/eitatech/deployments/main/argocd/apps/kafka/kafka-dev.yaml
```

Ou se você clonou o repositório:

```bash
kubectl apply -f argocd/apps/kafka/kafka-dev.yaml
```

### Deploy com Domínio Customizado

Por padrão, o Ingress usa `kafka.localhost`. Para usar um domínio personalizado:

#### Opção 1: Usando a CLI do ArgoCD

```bash
argocd app create kafka-dev \
  --repo https://github.com/eitatech/deployments.git \
  --path argocd/apps/kafka/applications \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace argocd \
  --revision main \
  --sync-policy automated \
  --auto-prune \
  --self-heal \
  --directory-recurse \
  --helm-set domain=meudominio.com
```

#### Opção 2: Editando o arquivo de values

1. Edite o arquivo `values/ingress-values.yaml` no seu fork do repositório
2. Altere a linha `domain: localhost` para `domain: seudominio.com`
3. Faça commit e push das alterações
4. Aplique a aplicação

### Verificação

1. Verifique os Applications criados:
   ```bash
   kubectl get applications -n argocd | grep kafka
   ```

2. Acompanhe o status das aplicações:
   ```bash
   argocd app list | grep kafka
   ```

3. Verifique se o Kafka está rodando:
   ```bash
   kubectl get kafka -n kafka
   kubectl get pods -n kafka
   ```

4. Teste a conectividade externa:
   ```bash
   # Descubra o IP do node
   NODE_IP=$(kubectl get node -o jsonpath='{.items[0].status.addresses[?(@.type=="InternalIP")].address}')
   
   # Teste com kcat (se disponível)
   kcat -b $NODE_IP:31094 -L
   
   # Ou com kafka tools
   kafka-topics.sh --bootstrap-server $NODE_IP:31094 --list
   ```

## Endpoints Disponíveis

### Internos (dentro do cluster)
- **PLAINTEXT**: `cluster-kafka-bootstrap.kafka.svc.cluster.local:9092`
- **TLS**: `cluster-kafka-bootstrap.kafka.svc.cluster.local:9093`

### Externos (NodePort)
- **PLAINTEXT**: `<NODE_IP>:31094`
- **TLS**: `<NODE_IP>:31096`

### Via Ingress
- **HTTP**: `kafka.${DOMAIN}` (onde ${DOMAIN} é o domínio configurado)

## Configuração do DNS

Para usar o Ingress, adicione no `/etc/hosts`:
```
<NODE_IP>  kafka.${DOMAIN}
```

Exemplo com domínio padrão:
```
<NODE_IP>  kafka.localhost
```

Exemplo com domínio customizado:
```
<NODE_IP>  kafka.meudominio.com
```

## Comandos Úteis

### Criar um tópico de teste
```bash
kubectl run kafka-producer -ti --image=quay.io/strimzi/kafka:0.45.0-kafka-4.0.0 --rm=true --restart=Never -- bin/kafka-topics.sh --bootstrap-server cluster-kafka-bootstrap:9092 --create --topic test --partitions 1 --replication-factor 1
```

### Enviar mensagens
```bash
kubectl run kafka-producer -ti --image=quay.io/strimzi/kafka:0.45.0-kafka-4.0.0 --rm=true --restart=Never -- bin/kafka-console-producer.sh --bootstrap-server cluster-kafka-bootstrap:9092 --topic test
```

### Consumir mensagens
```bash
kubectl run kafka-consumer -ti --image=quay.io/strimzi/kafka:0.45.0-kafka-4.0.0 --rm=true --restart=Never -- bin/kafka-console-consumer.sh --bootstrap-server cluster-kafka-bootstrap:9092 --topic test --from-beginning
```

### Listar tópicos
```bash
kubectl run kafka-topics -ti --image=quay.io/strimzi/kafka:0.45.0-kafka-4.0.0 --rm=true --restart=Never -- bin/kafka-topics.sh --bootstrap-server cluster-kafka-bootstrap:9092 --list
```

## Monitoramento

### Status do Cluster
```bash
kubectl get kafka cluster -n kafka -o wide
kubectl describe kafka cluster -n kafka
```

### Logs do Operador
```bash
kubectl logs -f deployment/strimzi-cluster-operator -n kafka
```

### Logs do Kafka
```bash
kubectl logs -f cluster-kafka-0 -n kafka
```

## Troubleshooting

### Operador não instala
- Verifique se o repositório Helm foi adicionado ao ArgoCD
- Verifique se há recursos suficientes no cluster

### Kafka não inicia
- Verifique se o operador está rodando
- Verifique se há storage suficiente disponível
- Verifique logs do operador e do pod do Kafka

### Conectividade externa não funciona
- Verifique se os NodePorts estão acessíveis (firewall, security groups)
- Verifique se o IP do node está correto
- Teste conectividade de rede

### Ingress não funciona
- Verifique se o Traefik está instalado e funcionando
- Verifique se o DNS está configurado corretamente
- Verifique se o serviço do Kafka está respondendo

## Estrutura de Arquivos

```
argocd/apps/kafka/
├── kafka-dev.yaml                               # App of Apps (arquivo principal)
├── applications/
│   ├── strimzi-operator-application.yaml        # Application do Strimzi Operator
│   ├── kafka-cluster-application.yaml           # Application do Kafka Cluster  
│   └── kafka-ingress-application.yaml           # Application do Ingress
└── resources/
    ├── kafka-cluster.yaml                       # Kafka CRDs (KafkaNodePool + Kafka)
    └── kafka-ingress.yaml                       # Ingress para exposição
```
