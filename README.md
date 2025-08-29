# stock-manager-project

## Diagrama


<img width="700" height="600" alt="image" src="https://github.com/user-attachments/assets/befd72f8-ad2b-4aec-86d8-ae2e1067d6ee" />
## Resumo

Este projeto é uma plataforma de e-commerce modular e escalável, construída sobre arquitetura de microsserviços.
Os serviços se comunicam entre si utilizando Feign Client para chamadas síncronas e Apache Kafka para mensageria assíncrona.
Todos os serviços rodam em containers Docker.


O desenvolvimento deste sistema foi organizado através do recurso de projetos do GitHub (com o processo de desenvolvimento Kanban), e todos os microsserviços estão vinculados ao projeto principal chamado [Stock Manager](https://github.com/users/Math012/projects/1/views/1). Porém, cada serviço que compõe o Stock Manager possui um repositório independente.

- [stock-authentication-service](https://github.com/Math012/stock-authentication-service)
- [stock-catalog-service](https://github.com/Math012/stock-catalog-service)
- [stock-inventory-service](https://github.com/Math012/stock-inventory-service)
- [stock-order-service](https://github.com/Math012/stock-order-service)
- [stock-payment-service](https://github.com/Math012/stock-payment-service)
- [stock-bff-service](https://github.com/Math012/stock-bff-service)

## DTOs do projeto

UserRequestDTO

```json
{
	"email": "math@math2.com",
	"password": "12345"
}
```
UserResponseDTO

```json
{
	"id": 1,
	"email": "math@math2.com",
	"password": "$2a$10$hSAeJwdpXZY8oT823qk6iOpC2QKCwzM2A29MtfKqZuZLOl9sUmMly"
}
```
OrderResponseDTO
```json
{
	"productName": "Bola Mega 5619",
	"totalQuantity": 9,
	"productPrice": 463.63,
}
```

### Serviço BFF
Requisições para a API devem seguir os padrões:
| Método | URL | Descrição | Body
|---|---|---|---|
| `POST` | localhost:8085/auth/register | Recebe em seu corpo de requisição um body de UserRequestDTO, em caso de sucesso, a requisição retorna um status code 200 e o usuário é cadastrado no sistema. Caso o usuário tente registrar um e-mail já cadastrado, a requisição retorna um status code 400 informando o erro. | UserRequestDTO
| `POST` | localhost:8085/auth/login | Recebe em seu corpo de requisição um body de UserRequestDTO, em caso de sucesso, a requisição retorna um token JWT. | UserRequestDTO
| `GET` | localhost:8085/catalog/{categoria} | Endpoint para consultar o catálogo de produtos, a requisição recebe um path variable informando a categoria desejada, este endpoint possui paginação e dois parâmetros opcionais, o primeiro parâmetro é um número inteiro com o nome de "page" usado para trocar a página de exibição, o segundo parâmetro também é um número inteiro com o nome "size" que controla a quantidade de elementos que a pagina irá exibir. Caso o usuário informe uma categoria inválida, um erro 404 será retornado. | N/A
| `POST` | localhost:8085/order/send | Recebe em seu corpo de requisição um body de OrderResponseDTO, e como o serviço de pedidos possui o Spring Security, é necessário informar o token JWT via header. O endpoint envia o pedido via Kafka para os serviços de estoque e pagamento. | OrderResponseDTO

## Fluxo de pedidos

- **order-service**: envia o pedido para o **inventory-service** via Kafka.
  
- **order-service**: recebe a resposta do **inventory-service**, caso não tenha estoque, uma mensagem é lançada avisando. Se o produto tiver estoque, ele será enviado ao serviço de pagamento.
  
- **payment-service**: recebe o pedido do **order-service** via Kafka, salva o resultado da transação (se o pagamento foi aprovado ou não) no banco de dados e retorna ao **order-service** informando o resultado da transação via Kafka.
  
- **order-service**: recebe a resposta do **payment-service**. Em caso de sucesso no pagamento, o **order-service** envia uma mensagem de confirmação de pagamento. Em caso de falha, o serviço envia uma mensagem de rollback.
  
- **inventory-service**: recebe a mensagem do **order-service**, caso seja uma solicitação de rollback, o serviço retira a quantidade reservada e adiciona na quantidade disponível. Em caso de sucesso no pagamento, a quantidade informada é retirada da quantidade reservada e também da quantidade total.

Exemplos de consulta:


 - banco de dados do inventory-service


<img width="700" height="600" alt="image" src="https://github.com/user-attachments/assets/2449eb42-e825-4e85-a315-bbe27a55af6e" />



- banco de dados do payment-service
  

<img width="400" height="300" alt="image" src="https://github.com/user-attachments/assets/9d1ad28f-d757-4a10-9024-ac5483dc35aa" />




# Como executar o projeto

### Conflito ao executar via docker
- Em alguns casos, o Kafka pode entrar em um ciclo de reprocessamento com o order-service, gerando um loop infinito de mensagens. Caso isso ocorra, é necessário reiniciar os serviços que possuem comunicação direta com o Kafka:
 - order-service
 - inventory-service
 - payment-service
 - Comando para reiniciar os serviços:
      ``` bash
      docker-compose restart order-service inventory-service payment-service
      ```
      
## Via Docker Hub

- Todos os serviços estão disponíveis no docker-hub

- Arquivo docker-compose.yml preenchido.

- Para executar o compose digite:
``` bash
docker-compose up
```

``` yaml
services:
  db_authentication:
    image: postgres:latest
    container_name: db_authentication
    environment:
      - POSTGRES_DB=db_user_stock
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=1234
    ports:
      - "5432:5432"
    networks:
      - backend
    healthcheck:
      test: [ "CMD-SHELL", "pg_isready -U postgres" ]
      interval: 5s
      timeout: 5s
      retries: 10

  db_catalog:
    image: postgres:latest
    container_name: db_catalog
    environment:
      - POSTGRES_DB=db_catalog_stock
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=1234
    ports:
      - "5433:5432"
    networks:
      - backend
    healthcheck:
      test: [ "CMD-SHELL", "pg_isready -U postgres" ]
      interval: 5s
      timeout: 5s
      retries: 10

  db_inventory:
    image: postgres:latest
    container_name: db_inventory
    environment:
      - POSTGRES_DB=db_inventory_stock
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=1234
    ports:
      - "5434:5432"
    networks:
      - backend
    healthcheck:
      test: [ "CMD-SHELL", "pg_isready -U postgres" ]
      interval: 5s
      timeout: 5s
      retries: 10

  db_transaction:
    image: mongo:latest
    container_name: db_transaction
    environment:
      - MONGO_INITDB_DATABASE=db_transaction_stock
    ports:
      - "27017:27017"
    networks:
      - backend

  kafka:
    container_name: kafka
    image: apache/kafka-native
    ports:
      - "9092:9092"
    environment:
      KAFKA_LISTENERS: CONTROLLER://localhost:9091,HOST://0.0.0.0:9092,DOCKER://0.0.0.0:9093
      KAFKA_ADVERTISED_LISTENERS: HOST://localhost:9092,DOCKER://kafka:9093
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: CONTROLLER:PLAINTEXT,DOCKER:PLAINTEXT,HOST:PLAINTEXT
      KAFKA_NODE_ID: 1
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@localhost:9091
      KAFKA_INTER_BROKER_LISTENER_NAME: DOCKER
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
    networks:
      - backend

  kafka-ui:
    image: kafbat/kafka-ui:main
    ports:
      - "8080:8080"
    environment:
      DYNAMIC_CONFIG_ENABLED: "true"
      KAFKA_CLUSTERS_0_NAME: local
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: kafka:9093
    networks:
      - backend
    depends_on:
      - kafka

  order-service:
    image: math012i/order-service
    ports:
      - "8082:8082"
    networks:
      - backend
    environment:
      - SPRING_KAFKA_BOOTSTRAP_SERVERS=kafka:9093
      - USER_URL=authentication-service:8092/authentication/find

  stock-bff-service:
    image: math012i/bff-service
    ports:
      - "8085:8085"
    networks:
      - backend
    environment:
      - CATALOG_URL=catalog-service:8081/catalog
      - AUTHENTICATION_URL=authentication-service:8092/authentication
      - ORDER_URL=order-service:8082/

  payment-service:
    image: math012i/payment-service
    ports:
      - "8084:8084"
    networks:
      - backend
    environment:
      - SPRING_KAFKA_BOOTSTRAP_SERVERS=kafka:9093
      - SPRING_DATA_MONGODB_URI=mongodb://db_transaction:27017/db_transaction_stock
      - SPRING_DATA_MONGODB_DATABASE=db_transaction_stock
    depends_on:
      db_transaction:
        condition: service_healthy

  inventory-service:
    image: math012i/inventory-service
    ports:
      - "8093:8093"
    networks:
      - backend
    environment:
      - SPRING_KAFKA_BOOTSTRAP_SERVERS=kafka:9093
      - SPRING_DATASOURCE_URL=jdbc:postgresql://db_inventory:5432/db_inventory_stock
      - SPRING_DATASOURCE_USERNAME=postgres
      - SPRING_DATASOURCE_PASSWORD=1234
      - SPRING_JPA_HIBERNATE_DDL_AUTO=create
    depends_on:
      db_inventory:
        condition: service_healthy

  catalog-service:
    image: math012i/catalog-service
    ports:
      - "8081:8081"
    networks:
      - backend
    environment:
      - SPRING_DATASOURCE_URL=jdbc:postgresql://db_catalog:5432/db_catalog_stock
      - SPRING_DATASOURCE_USERNAME=postgres
      - SPRING_DATASOURCE_PASSWORD=1234
      - SPRING_JPA_HIBERNATE_DDL_AUTO=create
    depends_on:
      db_catalog:
        condition: service_healthy

  authentication-service:
    image: math012i/authentication-service:latest
    ports:
      - "8092:8092"
    networks:
      - backend
    environment:
      - SPRING_DATASOURCE_URL=jdbc:postgresql://db_authentication:5432/db_user_stock
      - SPRING_DATASOURCE_USERNAME=postgres
      - SPRING_DATASOURCE_PASSWORD=1234
      - SPRING_JPA_HIBERNATE_DDL_AUTO=create
      - SECRET_KEY=ebca22c718d8a0107f9f81c2f1245d73
    depends_on:
      db_authentication:
        condition: service_healthy

networks:
  backend:
    driver: bridge
```

## Via docker-compose.yml local

- Para executar via docker-compose é necessário baixar/clonar todos os projetos e adicionar em uma única pasta.
<img width="263" height="172" alt="image" src="https://github.com/user-attachments/assets/68ca4927-57e2-4a6c-8b82-031a8738b9f1" />

- Executar o clean build individualmente para cada projeto
  
<img width="400" height="200" alt="clean" src="https://github.com/user-attachments/assets/45d71897-3455-478b-89d5-aee270392c42" />

- Após executar o build dos projetos, é preciso criar o docker-compose.yml na raiz da pasta onde estão os projetos.

- Arquivo modelo para o docker-compose.yml.
  - é possível alterar cada variável (=${}) via hardcode ou criando um arquivo ".env" na raiz da pasta onde estão os projetos


- docker-compose.yml

- Para executar o compose digite:
    
``` bash
docker-compose up
```
  
``` yaml
services:
  db_authentication:
    image: postgres:latest
    container_name: db_authentication
    environment:
      - POSTGRES_DB=${DB_DATABASE_AUTH}
      - POSTGRES_USER=${DB_USERNAME_AUTH}
      - POSTGRES_PASSWORD=${DB_PASSWORD_AUTH}
    ports:
      - "5432:5432"
    networks:
      - backend
    healthcheck:
      test: [ "CMD-SHELL", "pg_isready -U postgres" ]
      interval: 5s
      timeout: 5s
      retries: 10

  db_catalog:
    image: postgres:latest
    container_name: db_catalog
    environment:
      - POSTGRES_DB=${DB_DATABASE_CATALOG}
      - POSTGRES_USER=${DB_USERNAME_CATALOG}
      - POSTGRES_PASSWORD=${DB_PASSWORD_CATALOG}
    ports:
      - "5433:5432"
    networks:
      - backend
    healthcheck:
      test: [ "CMD-SHELL", "pg_isready -U postgres" ]
      interval: 5s
      timeout: 5s
      retries: 10

  db_inventory:
    image: postgres:latest
    container_name: db_inventory
    environment:
      - POSTGRES_DB=${DB_DATABASE_INVENTORY}
      - POSTGRES_USER=${DB_USERNAME_INVENTORY}
      - POSTGRES_PASSWORD=${DB_PASSWORD_INVENTORY}
    ports:
      - "5434:5432"
    networks:
      - backend
    healthcheck:
      test: [ "CMD-SHELL", "pg_isready -U postgres" ]
      interval: 5s
      timeout: 5s
      retries: 10

  db_transaction:
    image: mongo:latest
    container_name: db_transaction
    environment:
      - MONGO_INITDB_DATABASE=${DB_DATABASE_PAYMENT}
    ports:
      - "27017:27017"
    networks:
      - backend

  authentication-service:
    build:
      context: ./authentication-service
      dockerfile: Dockerfile
    ports:
      - "8092:8092"
    networks:
      - backend
    environment:
      - SPRING_DATASOURCE_URL=${SPRING_DATASOURCE_URL_AUTH}${DB_DATABASE_AUTH}
      - SPRING_DATASOURCE_USERNAME=${DB_USERNAME_AUTH}
      - SPRING_DATASOURCE_PASSWORD=${DB_PASSWORD_AUTH}
      - SPRING_JPA_HIBERNATE_DDL_AUTO=${SPRING_JPA_HIBERNATE_DDL_AUTO_AUTH}
      - SECRET_KEY=${SECRET_KEY_AUTH}
    depends_on:
      db_authentication:
        condition: service_healthy

  catalog-service:
    build:
      context: ./catalog-service
      dockerfile: Dockerfile
    ports:
      - "8081:8081"
    networks:
      - backend
    environment:
      - SPRING_DATASOURCE_URL=${SPRING_DATASOURCE_URL_CATALOG}${DB_DATABASE_CATALOG}
      - SPRING_DATASOURCE_USERNAME=${DB_USERNAME_CATALOG}
      - SPRING_DATASOURCE_PASSWORD=${DB_PASSWORD_CATALOG}
      - SPRING_JPA_HIBERNATE_DDL_AUTO=${SPRING_JPA_HIBERNATE_DDL_AUTO_CATALOG}
    depends_on:
      db_catalog:
        condition: service_healthy

  inventory-service:
    build:
      context: ./inventory-service
      dockerfile: Dockerfile
    ports:
      - "8093:8093"
    networks:
      - backend
    environment:
      - SPRING_KAFKA_BOOTSTRAP_SERVERS=${SPRING_KAFKA_BOOTSTRAP_SERVERS}
      - SPRING_DATASOURCE_URL=${SPRING_DATASOURCE_URL_INVENTORY}${DB_DATABASE_INVENTORY}
      - SPRING_DATASOURCE_USERNAME=${DB_USERNAME_INVENTORY}
      - SPRING_DATASOURCE_PASSWORD=${DB_PASSWORD_INVENTORY}
      - SPRING_JPA_HIBERNATE_DDL_AUTO=${SPRING_JPA_HIBERNATE_DDL_AUTO_INVENTORY}
    depends_on:
      db_inventory:
        condition: service_healthy

  payment-service:
    build:
      context: ./payment-service
      dockerfile: Dockerfile
    ports:
      - "8084:8084"
    networks:
      - backend
    environment:
      - SPRING_KAFKA_BOOTSTRAP_SERVERS=${SPRING_KAFKA_BOOTSTRAP_SERVERS}
      - SPRING_DATA_MONGODB_URI=${SPRING_DATA_MONGODB_URI}${DB_DATABASE_PAYMENT}
      - SPRING_DATA_MONGODB_DATABASE=${SPRING_DATA_MONGODB_DATABASE}
    depends_on:
      db_transaction:
        condition: service_healthy

  order-service:
    build:
      context: ./order-service
      dockerfile: Dockerfile
    ports:
      - "8082:8082"
    networks:
      - backend
    environment:
      - SPRING_KAFKA_BOOTSTRAP_SERVERS=${SPRING_KAFKA_BOOTSTRAP_SERVERS}
      - USER_URL=${USER_URL}

  stock-bff-service:
    build:
      context: ./stock-bff-service
      dockerfile: Dockerfile
    ports:
      - "8085:8085"
    networks:
      - backend
    environment:
      - CATALOG_URL=${CATALOG_URL}
      - AUTHENTICATION_URL=${AUTHENTICATION_URL}
      - ORDER_URL=${ORDER_URL}

  kafka:
    container_name: kafka
    image: apache/kafka-native
    ports:
      - "9092:9092"
    environment:
      KAFKA_LISTENERS: CONTROLLER://localhost:9091,HOST://0.0.0.0:9092,DOCKER://0.0.0.0:9093
      KAFKA_ADVERTISED_LISTENERS: HOST://localhost:9092,DOCKER://kafka:9093
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: CONTROLLER:PLAINTEXT,DOCKER:PLAINTEXT,HOST:PLAINTEXT
      KAFKA_NODE_ID: 1
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@localhost:9091
      KAFKA_INTER_BROKER_LISTENER_NAME: DOCKER
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
    networks:
      - backend

  kafka-ui:
    image: kafbat/kafka-ui:main
    ports:
      - "8080:8080"
    environment:
      DYNAMIC_CONFIG_ENABLED: "true"
      KAFKA_CLUSTERS_0_NAME: local
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: kafka:9093
    networks:
      - backend
    depends_on:
      - kafka

networks:
  backend:
    driver: bridge
```

- Modelo de env.
``` env
# authentication service
DB_DATABASE_AUTH=db_user_stock
DB_USERNAME_AUTH=postgres
DB_PASSWORD_AUTH=1234
SECRET_KEY_AUTH=ebca22c718d8a0107f9f81c2f1245d73
SPRING_DATASOURCE_URL_AUTH=jdbc:postgresql://db_authentication:5432/
SPRING_JPA_HIBERNATE_DDL_AUTO_AUTH=create

# catalog service
DB_DATABASE_CATALOG=db_catalog_stock
DB_USERNAME_CATALOG=postgres
DB_PASSWORD_CATALOG=1234
SPRING_DATASOURCE_URL_CATALOG=jdbc:postgresql://db_catalog:5432/
SPRING_JPA_HIBERNATE_DDL_AUTO_CATALOG=create

# inventory service
DB_DATABASE_INVENTORY=db_inventory_stock
DB_USERNAME_INVENTORY=postgres
DB_PASSWORD_INVENTORY=1234
SPRING_DATASOURCE_URL_INVENTORY=jdbc:postgresql://db_inventory:5432/
SPRING_JPA_HIBERNATE_DDL_AUTO_INVENTORY=create

# payment service
DB_DATABASE_PAYMENT=db_transaction_stock
DB_USERNAME_PAYMENT=root
DB_PASSWORD_PAYMENT=1234
SPRING_DATA_MONGODB_URI=mongodb://db_transaction:27017/
SPRING_DATA_MONGODB_DATABASE=db_transaction_stock

# order service
USER_URL=authentication-service:8092/authentication/find

# bff service
CATALOG_URL=catalog-service:8081/catalog
AUTHENTICATION_URL=authentication-service:8092/authentication
ORDER_URL=order-service:8082/

# kafka
SPRING_KAFKA_BOOTSTRAP_SERVERS=kafka:9093
```

## Local 

- A primeira maneira é baixando cada projeto individualmente e executando através da IDEA.
   - Para executar de forma local é preciso utilizar o Java 17 ou superior.
   - E preciso alterar as credenciais de acesso ao banco de dados PostgreSQL dentro do application-test.properties dos projetos: [stock-authentication-service](https://github.com/Math012/stock-authentication-service) , [stock-catalog-service](https://github.com/Math012/stock-catalog-service) e [stock-inventory-service](https://github.com/Math012/stock-inventory-service)
   - É necessário criar 3 bancos de dados Postgres: db_inventory_stock, db_catalog_stock, db_usuario_stock
     
- Link de todos os projetos para baixar ou clonar:
  - [stock-authentication-service](https://github.com/Math012/stock-authentication-service)
  - [stock-catalog-service](https://github.com/Math012/stock-catalog-service)
  - [stock-inventory-service](https://github.com/Math012/stock-inventory-service)
  - [stock-order-service](https://github.com/Math012/stock-order-service)
  - [stock-payment-service](https://github.com/Math012/stock-payment-service)
  - [stock-bff-service](https://github.com/Math012/stock-bff-service)


