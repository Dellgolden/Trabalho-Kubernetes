# Documentação do Projeto

## Pré-requisitos

Para rodar este projeto, certifique-se de ter os seguintes itens instalados:

- **Docker Compose**: https://docs.docker.com/compose/install/
- **Git**: https://git-scm.com/book/en/v2/Getting-Started-Installing-Git

---

## **Git**:
O repositório já deve ter sido clonado para o seu ambiente. Caso não tenha feito isso, execute o seguinte comando:

  ```bash
  git clone https://github.com/Dellgolden/Trabalho-Docker_Guess_Game.git
  ```
  
Após clonar o repositório, navegue até a pasta correspondente:

  ```bash
  cd Trabalho-Docker_Guess_Game
  ```
Depois, execute:

```bash
docker-compose up --build
```

Você pode acessar a aplicação a partir de sua máquina utilizando o link do localhost na porta 80: http://localhost

Você devera visualizar uma pagina como essa abaixo:

<img src="localhost.png">

### Observação Técnica:
 
A porta do host 80 deve estar disponível para garantir o funcionamento adequado dos serviços. Caso haja um servidor web (como Apache ou Nginx) em execução localmente, pode ocorrer um conflito de porta. Nesse caso, será necessário modificar o arquivo docker-compose.yml localizado na raiz do projeto, especificamente no bloco de configuração do Nginx, para mapear a porta do contêiner para uma porta alternativa no host.

---

## Estrutura do Repositório

- **Arquivo Docker Compose:** Localizado na raiz do projeto. Nome: `docker-compose.yml`
- **Dockerfile do Backend (Python):** Localizado na raiz do projeto. Nome: `Dockerfile`
- **Dockerfile do Frontend (React):** Localizado na pasta `frontend`. Nome: `frontend/Dockerfile`
- **Configuração do NGINX:** Localizado na raiz do projeto. Nome: `nginx.conf`

### Observação Técnica:
 
Foi adicionado o parâmetro `restart: always` em todos os serviços no arquivo `docker-compose.yml`.

Essa configuração garante que, em caso de falhas, o Docker tentará reiniciar automaticamente os contêineres, aumentando a resiliência da aplicação.

---

## Configuração e Estrutura

### 1. Arquivo `docker-compose.yml`

O arquivo `docker-compose.yml` foi configurado para conter todos os serviços necessários para iniciar o sistema.

#### 1.1. Frontend

Foi criado um **Dockerfile** específico para o frontend dentro da pasta `frontend`, separado do backend para facilitar atualizações. No arquivo `docker-compose.yml`, o serviço frontend é configurado para realizar o build utilizando o contexto da pasta `frontend`:

```yaml
frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
    restart: always
    environment:
    - REACT_APP_BACKEND_URL=http://localhost
    networks:
      - app-network
    ports:
      - "80:80"
    depends_on:
      - backend
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf  
```

#### 1.2. Backend

O backend contém um `Dockerfile` específico localizado na raiz do projeto. As variáveis de ambiente, que anteriormente estavam codificadas diretamente no código, foram movidas para o arquivo `docker-compose.yml`. 

Adicionalmente, foi adicionado um **healthcheck** para garantir que o serviço do backend esteja pronto antes que o NGINX e o balanceamento de carga sejam ativados. 

Para isso foi necessário adicionar uma linha para instalar o curl dentro do Dockerfile para poder realizar **healthcheck**. 

Abaixo está a configuração do backend no arquivo `docker-compose.yml`:

```yaml
  backend:
    build:
      context: .
      dockerfile: Dockerfile
    restart: always
    environment:
      FLASK_APP: run.py
      FLASK_DB_TYPE: postgres
      FLASK_DB_USER: postgres
      FLASK_DB_PASSWORD: secretpass
      FLASK_DB_NAME: guess_game
      FLASK_DB_HOST: db
      FLASK_DB_PORT: 5432
    networks:
      - app-network
    depends_on:
      - db
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:5000/health"]
      interval: 10s
      timeout: 5s
      retries: 3
      start_period: 20s
```

Abaixo está a configuração do Dockerfile da raiz do projeto alterado com a instalação do curl na 3a linha:

```yaml
FROM python:3.9-slim

WORKDIR /app

RUN apt-get update && apt-get install curl -y

# Copia os requisitos e instala as dependências
COPY requirements.txt .
# RUN pip install --no-cache-dir -r requirements.txt
RUN pip install --trusted-host pypi.org --trusted-host pypi.python.org --trusted-host files.pythonhosted.org -r requirements.txt


# Copia o restante do código para o container
COPY . .

# ENV FLASK_APP=run.py
EXPOSE 5000

CMD ["flask", "run", "--host=0.0.0.0", "--port=5000"]
```

#### 1.3. Banco de Dados (PostgreSQL)

Foi criada uma entrada para o serviço PostgreSQL, passando as credenciais e o banco de dados via variáveis de ambiente. Além disso, foi mapeado um volume chamado "postgres-data" para persistir os dados, garantindo que o banco seja preservado em caso de reinicialização do container.

```yaml
  db:
    image: postgres:14
    restart: always
    container_name: postgres
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: secretpass
      POSTGRES_DB: guess_game
    volumes:
      - db_data:/var/lib/postgresql/data
    networks:
      - app-network
```
#### 1.4. Load Balancer (NGINX)

Um serviço NGINX foi configurado para atuar como balanceador de carga, distribuindo requisições tanto para o frontend quanto para o backend. O arquivo de configuração nginx.conf, localizado na raiz do repositório, é mapeado dentro do container Frontend, como arquivo de configuração do NGINX.

```yaml
FROM nginx:alpine
COPY --from=build /app/build /usr/share/nginx/html
EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```
#### 1.5. Volume para Persistência do PostgreSQL:

Foi criado um volume postgres-data para persistir os dados do banco de dados PostgreSQL e ser montado no bloco do container do PostgreSQL:

```yaml
volumes:
  postgres-data:
```
---

### 2. Balanceamento de Carga com NGINX:

Foi criada uma configuração do NGINX que direciona a rota / para o frontend. As outras rotas (/create, /breaker, /guess) são direcionadas para o backend, foi utilizada expressão regular para filtrar as rotas.

```yaml
events {}

http {
    include /etc/nginx/mime.types;
    sendfile on;    
    
    upstream backend {
        server backend:5000;
    }

    server {
        listen 80;

        location / {
            root /usr/share/nginx/html;
            index index.html;
            try_files $uri /index.html;
        }

        location /api/ {
            proxy_pass http://backend/;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }
    }
}
        
```
---

### 3. Guia de Atualização: Docker Compose, Backend, Frontend e Load Balancer
Para realizar atualizações backend Python, frontend ou Load Balancer Nginx, siga as instruções abaixo.

#### 3.1. Balanceamento de Carga com NGINX:
Para alterar rotas, portas ou outras configurações, edite o arquivo nginx.conf na ***raiz do repositório***.<br>
Para alterar a versão da imagem do nginx altere o arquivo docker-compose.yml.

Depois, execute:

```bash
docker-compose down
docker-compose up
```

#### 3.2. Atualize o Frontend:
Modifique o Dockerfile localizado na pasta ***frontend*** da aplicação ou o arquivo docker-compose.yml, conforme necessário.

Depois, execute:

```bash
docker-compose down
docker-compose up
```

#### 3.3. Atualize o Backend:
Altere as variáveis de ambiente no arquivo docker-compose.yml ou modifique o Dockerfile que está na ***raiz do repositório*** da aplicação conforme necessário.

Depois, execute:

```bash
docker-compose down
docker-compose up
```

#### 3.4. Atualize o docker-compose.yml:
Faça as alterações necessárias (troca de portas, variáveis de ambiente, healthcheck, etc) no arquivo `docker-compose.yml` e execute os comandos abaixo para aplicar as mudanças:

Depois, execute:

```bash
docker-compose down
docker-compose up
```
