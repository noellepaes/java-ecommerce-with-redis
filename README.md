# Ecommerce Monolith Modular

Projeto **monolito modular** em **Java 17** (um único projeto Maven/Spring Boot), preparado para evoluir para microsserviços.

**Diferencial deste repositório:** uso de **Redis** para recomendações colaborativas em tempo real, com histórico efêmero de visualizações — sem persistir essas associações no PostgreSQL.

## 🔴 Redis no projeto

O módulo `recommendation/` usa **Redis 7** como store de visualizações. Cada clique ou visualização de produto alimenta um **grafo bidirecional** em memória:

| Chave Redis | Tipo | Conteúdo |
|-------------|------|----------|
| `ecommerce:views:user:{customerId}` | SET | IDs dos produtos que o cliente viu |
| `ecommerce:views:product:{productId}` | SET | IDs dos clientes que viram o produto |

**Por que Redis aqui?**

- Dados **efêmeros** — histórico de navegação com TTL configurável (`app.recommendation.customer-history-ttl`, padrão `30d`)
- **Leitura e escrita rápidas** para sugestões colaborativas (quem viu X também viu Y)
- **Sem entidade JPA** — PostgreSQL continua responsável pelos dados transacionais; Redis cuida do sinal de comportamento

Fluxo resumido:

1. `POST /api/recommendations/customers/{id}/views` registra a visualização nos dois SETs e renova o TTL
2. `GET /api/recommendations/customers/{id}` monta sugestões a partir do grafo no Redis
3. Se não houver sinal suficiente, a API completa com produtos disponíveis do catálogo (PostgreSQL)

### Redis Insight — visualização das chaves

Conecte o [Redis Insight](https://redis.io/insight/) em `localhost:6379` e filtre por `ecommerce:views:*` para inspecionar os SETs em tempo real.

Após subir a aplicação em DEV, o seed popula visualizações automaticamente — os logs indicam IDs de clientes e produtos para testar.

<!-- Adicione sua captura de tela em docs/images/redis-insight.png -->
![Visualização das chaves `ecommerce:views:*` no Redis Insight](docs/images/redis-insight.png)

> **Para incluir a foto:** salve a captura em `docs/images/redis-insight.png` (ou altere o caminho acima).

## 🧱 Estrutura (1 projeto, módulos por pacote)

O projeto é **um único app executável** e os Bounded Contexts ficam separados por pacote dentro de `src/main/java`:

```
src/main/java/com/ecommerce/
 ├── auth/
 ├── product/
 ├── customer/
 ├── order/
 ├── payment/
 ├── recommendation/   ← Redis (visualizações + sugestões)
 └── shared/
```

### Estrutura DDD dentro de cada módulo

Cada contexto segue:

```
module/
 ├── domain/
 │    ├── model/
 │    ├── service/
 │    ├── repository/
 │    └── exception/
 ├── application/
 │    ├── usecase/
 │    └── dto/
 ├── infrastructure/
 │    └── repository/
 └── presentation/
      ├── *Controller.java
      └── *Request.java
```

## 🗄️ Banco de Dados

### Estratégia: PostgreSQL com Schemas Separados

- **1 banco PostgreSQL** com schemas por contexto:
  - `product_schema`
  - `customer_schema`
  - `order_schema`
  - `payment_schema`
  - `auth` (tabelas de usuário)

### Flyway como Fonte Única de Verdade

Migrations em `src/main/resources/db/migration/v1/`:
- `V1__01_create_schemas.sql`
- `V2__create_customer_tables.sql`
- `V3__create_product_tables.sql`
- `V4__create_order_tables.sql`
- `V5__create_payment_tables.sql`
- `V6__create_auth_tables.sql`

## 🚀 Como Executar

### 1. Subir PostgreSQL e Redis com Docker

```bash
docker-compose up -d
```

Isso sobe:
- **PostgreSQL** em `localhost:5432`
- **Redis** em `localhost:6379`
- **App** em `localhost:8080` (profile `docker`)

Para rodar só a infra (PostgreSQL + Redis) e a app localmente:

```bash
docker-compose up -d postgres redis
```

### 2. Compilar o Projeto

```bash
mvn clean install
```

### 3. Executar a Aplicação

```bash
mvn spring-boot:run
```

A aplicação estará disponível em: `http://localhost:8080`

Swagger UI: `http://localhost:8080/swagger-ui.html`

## 📋 Endpoints

### Recomendações (Redis)

- `POST /api/recommendations/customers/{customerId}/views` — Registrar visualização/clique (grava no Redis)
- `GET /api/recommendations/customers/{customerId}` — Sugestões colaborativas com base no histórico

### Product

- `POST /api/products` - Criar produto
- `GET /api/products/{id}` - Buscar produto
- `GET /api/products` - Listar produtos
- `PUT /api/products/{id}` - Atualizar produto
- `POST /api/products/{id}/decrease-stock` - Reduzir estoque

### Customer

- `POST /api/customers` - Criar cliente
- `GET /api/customers/{id}` - Buscar cliente
- `GET /api/customers` - Listar clientes
- `PUT /api/customers/{id}` - Atualizar cliente
- `DELETE /api/customers/{id}` - Desativar cliente

### Order

- `POST /api/orders` - Criar pedido
- `GET /api/orders/{id}` - Buscar pedido
- `GET /api/orders/customer/{customerId}` - Listar pedidos do cliente
- `POST /api/orders/{id}/items` - Adicionar item ao pedido
- `POST /api/orders/{id}/pay` - Pagar pedido
- `POST /api/orders/{id}/cancel` - Cancelar pedido

### Payment

- `POST /api/payments` - Criar pagamento
- `GET /api/payments/{id}` - Buscar pagamento
- `GET /api/payments/order/{orderId}` - Listar pagamentos do pedido

## 🔄 Comunicação entre Módulos

### Atual (Monolito Modular)

- **Módulo `shared/`**: Classes comuns (BaseEntity, DomainEvent, BusinessException)
- **Referências por UUID**: Cada módulo referencia outros por UUID
- **Comunicação síncrona**: Services podem chamar outros Services diretamente
- **Redis no `recommendation/`**: histórico de visualizações fora do PostgreSQL

### Futuro (Microsserviços)

- **Eventos de domínio**: `OrderCreatedEvent`, `PaymentApprovedEvent`, etc.
- **Message broker**: RabbitMQ/Kafka para comunicação assíncrona
- **API Gateway**: Para comunicação síncrona entre serviços

## 🎯 Regras de Negócio Implementadas

### Order (Aggregate Root)

- ✅ Só pode ir de `PENDING` → `PAID`
- ✅ Nunca pode ir de `CANCELLED` → `PAID`
- ✅ Pedido pago não pode ser cancelado

### Product

- ✅ Validação de estoque antes de reduzir
- ✅ Produto deve estar ativo e com estoque para estar disponível

### Payment

- ✅ Só pode aprovar pagamentos `PENDING`

### Recomendação (Redis)

- ✅ Visualizações gravadas em SETs bidirecionais com TTL
- ✅ Sugestões priorizam co-visualização entre clientes
- ✅ Fallback para catálogo quando não há sinal no Redis

## 🛠️ Tecnologias

- Java 17
- Spring Boot 3.2.0
- Spring Data JPA
- **Spring Data Redis**
- **Redis 7**
- PostgreSQL 15
- Flyway (migrations)
- Lombok
- Maven

## 📝 Próximos Passos

1. ✅ Estrutura multi-módulo
2. ✅ Schemas separados
3. ✅ Flyway como fonte única
4. ✅ Recomendações com Redis
5. ⏳ Grafana + Prometheus (observabilidade)
6. ⏳ Implementar eventos de domínio
7. ⏳ Migrar para WebFlux (não bloqueante)
8. ⏳ Extrair para microsserviços

## 📚 Documentação

- `docs/ARQUITETURA.md` - Explicação detalhada da arquitetura
- `docs/COMUNICACAO_MODULOS.md` - Como os módulos se comunicam
- `docs/GUIA_RAPIDO.md` - Guia rápido de testes
