# Trabalho prático Unidade 2 Kubernetes

# Jogo de Adivinhação com Flask

Este é um simples jogo de adivinhação desenvolvido utilizando o framework Flask. O jogador deve adivinhar uma senha criada aleatoriamente, e o sistema fornecerá feedback sobre o número de letras corretas e suas respectivas posições.

## Funcionalidades

- Criação de um novo jogo com uma senha fornecida pelo usuário.
- Adivinhe a senha e receba feedback se as letras estão corretas e/ou em posições corretas.
- As senhas são armazenadas  utilizando base64.
- As adivinhações incorretas retornam uma mensagem com dicas.
  
## Requisitos

- Kubernetes
- Docker
- Git

---
## Instalação

Clonar o repositório

```bash
git clone https://github.com/Dellgolden/Trabalho-Kubernetes.git
```

Navegar até a pasta

```bash
cd guess_game_k8s
```

Para subir a aplicação, use o comando abaixo:

```bash
./deploy.sh
```

ou

```bash
kubectl apply -f kub01/ -R
```

Executar o comando abaixo e aguardar todos estarem com STATUS 'Running'.

```bash
kubectl get pods
```

Acessar a aplicação via browser na URL http://localhost:30081 (NodePort)

A porta do frontend-svc (NodePort), informada acima, pode ser encontrada obervando o serviço com comando abaixo

```bash
kubectl get svc
```

---
## Como Jogar

### 1. Criar um novo jogo

- Acesse a url acima

- Digite uma frase secreta

- Envie

- Salve o game-id


### 2. Adivinhar a senha

- Vá para o endponint breaker

- Entre com o game_id que foi gerado pelo Creator

- Tente adivinhar

---
## Estrutura do Código

### Repositório

- **Arquivos YAML** Instruções para subir utilizado para subir a aplicação separadas em Deployments (ConfigMaps, PV e PVC), Serviços, Secrets, HorizontalPodAutoscaler.
- **HorizontalPodAutoscaler:** Configurado para autoscaling do backend considerando as metricas de CPU, minimo 1 e máximo 6 replicas.
- **Ingress:** Ingress Controler usando NGINX para o frontend e backend.
- **Configuração do NGINX:** `nginx.conf` ajustando dentro do frontend.yaml no ConfigMap.

---
## Melhorias implementadas

- Utilização de imagens (bportilho/gg_backend:1.0 e bportilho/gg_frontend:2.0) já disponíveis no DockerHub
- Pull de imagem quando não presente.
- Utilização de StatefulSet para o banco de dados Postgres
- HPA para balanceamento de carga no backend
- Reinicio de container implementado com `restart: always`.
- Volume persistente para o Banco de Dados PostgreSQL por meio de PV e PVC.

---
## Em caso de qualquer alteração de código:
Qualquer alteração realizada no código pode ser aplicada executando os comandos abaixo: 

```bash
kubectl apply -f kub01/ -R
```

---
## Para finalizar todo processo executar:

```bash
 kubectl delete -f kub01/ -R
```
