---
title: "Event Sourcing: CDC vs. Outbox Pattern"
date: 2025-02-20
categories:
  - "Arquitetura de Software"
  - "Event-driven"
  - "Fundamentos"
---

A captura e propagação de eventos em sistemas distribuídos é um dos desafios mais complexos da arquitetura moderna. Duas técnicas se destacam neste cenário: **Change Data Capture (CDC)** e **Outbox Pattern**. Ambas resolvem o problema fundamental de como garantir que mudanças de estado sejam propagadas de forma confiável, mas cada uma com suas próprias características, vantagens e trade-offs.

## O Problema Fundamental

Antes de mergulharmos nas soluções, vamos entender o problema. Em sistemas distribuídos, frequentemente precisamos:

1. Atualizar o estado local (banco de dados)
2. Notificar outros sistemas sobre essa mudança
3. Garantir consistência entre essas operações

O desafio surge porque estas são operações distintas que podem falhar independentemente. Se atualizarmos o banco mas falharmos ao enviar o evento, teremos inconsistência. Se enviarmos o evento mas falharmos ao atualizar o banco, também teremos problemas.

## Change Data Capture (CDC)

### O que é?

CDC é uma técnica que captura mudanças em tempo real diretamente do log de transações do banco de dados. Em vez de modificar a aplicação, o CDC "escuta" o banco e transforma cada mudança em um evento.

### Como funciona?

```
┌─────────────┐     ┌──────────────┐     ┌─────────────┐
│  Aplicação  │────▶│ Banco de     │────▶│ Transaction │
│             │     │ Dados        │     │ Log         │
└─────────────┘     └──────────────┘     └──────┬───────┘
                                                  │
                                                  ▼
                    ┌──────────────┐     ┌─────────────┐
                    │ CDC Connector│────▶│ Event Stream│
                    │ (Debezium)   │     │ (Kafka)     │
                    └──────────────┘     └─────────────┘
```

### Implementação com Debezium

```json
{
  "name": "inventory-connector",
  "config": {
    "connector.class": "io.debezium.connector.mysql.MySqlConnector",
    "database.hostname": "mysql",
    "database.port": "3306",
    "database.user": "debezium",
    "database.password": "dbz",
    "database.server.id": "184054",
    "database.server.name": "fullfillment",
    "database.include.list": "inventory",
    "database.history.kafka.bootstrap.servers": "kafka:9092",
    "database.history.kafka.topic": "dbhistory.fullfillment",
    "include.schema.changes": "true"
  }
}
```

### Vantagens do CDC

1. **Zero mudança no código**: Não requer alterações na aplicação existente
2. **Captura completa**: Pega TODAS as mudanças, mesmo de sistemas legados
3. **Performance**: Leitura assíncrona do log não impacta a aplicação
4. **Histórico completo**: Pode capturar o estado anterior e posterior

### Desvantagens do CDC

1. **Complexidade operacional**: Requer infraestrutura adicional
2. **Acoplamento ao banco**: Específico para cada tipo de banco de dados
3. **Dados técnicos**: Eventos contêm estrutura do banco, não do domínio
4. **Latência**: Pequeno delay entre a mudança e a captura

### Exemplo de Evento CDC

```json
{
  "before": {
    "id": 1,
    "name": "Produto A",
    "price": 100.00,
    "stock": 50
  },
  "after": {
    "id": 1,
    "name": "Produto A",
    "price": 100.00,
    "stock": 45
  },
  "source": {
    "version": "1.9.0.Final",
    "connector": "mysql",
    "name": "fullfillment",
    "ts_ms": 1645123456789,
    "db": "inventory",
    "table": "products"
  },
  "op": "u",
  "ts_ms": 1645123456800
}
```

## Outbox Pattern

### O que é?

O Outbox Pattern resolve o problema da consistência dual criando uma tabela especial (outbox) no mesmo banco de dados da aplicação. Eventos são escritos nesta tabela na mesma transação que modifica o estado, garantindo atomicidade.

### Como funciona?

```
┌─────────────┐
│  Aplicação  │
└──────┬──────┘
       │ BEGIN TRANSACTION
       ▼
┌──────────────┐     ┌──────────────┐
│ Domain Table │     │ Outbox Table │
│ (Orders)     │     │ (Events)     │
└──────────────┘     └──────┬───────┘
                            │ COMMIT
                            ▼
                    ┌──────────────┐
                    │ Event Relay  │
                    │ (Publisher)  │
                    └──────┬───────┘
                            │
                            ▼
                    ┌──────────────┐
                    │ Event Stream │
                    │ (Kafka)      │
                    └──────────────┘
```

### Implementação

```sql
-- Tabela de domínio
CREATE TABLE orders (
    id UUID PRIMARY KEY,
    customer_id UUID NOT NULL,
    total_amount DECIMAL(10,2),
    status VARCHAR(50),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Tabela outbox
CREATE TABLE outbox_events (
    id UUID PRIMARY KEY,
    aggregate_id UUID NOT NULL,
    aggregate_type VARCHAR(255) NOT NULL,
    event_type VARCHAR(255) NOT NULL,
    event_data JSONB NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    processed_at TIMESTAMP NULL
);

-- Índice para polling eficiente
CREATE INDEX idx_outbox_unprocessed 
ON outbox_events(created_at) 
WHERE processed_at IS NULL;
```

### Código da Aplicação

```java
@Transactional
public Order createOrder(CreateOrderRequest request) {
    // 1. Criar pedido
    Order order = new Order(
        UUID.randomUUID(),
        request.getCustomerId(),
        request.getItems(),
        OrderStatus.PENDING
    );
    
    // 2. Salvar no banco
    orderRepository.save(order);
    
    // 3. Criar evento na mesma transação
    OutboxEvent event = new OutboxEvent(
        UUID.randomUUID(),
        order.getId(),
        "Order",
        "OrderCreated",
        JsonUtils.toJson(new OrderCreatedEvent(
            order.getId(),
            order.getCustomerId(),
            order.getTotalAmount()
        ))
    );
    
    outboxRepository.save(event);
    
    // 4. Commit automático via @Transactional
    return order;
}
```

### Event Publisher

```java
@Component
public class OutboxEventPublisher {
    
    @Scheduled(fixedDelay = 5000)
    public void publishEvents() {
        List<OutboxEvent> unpublishedEvents = 
            outboxRepository.findUnprocessedEvents(100);
        
        for (OutboxEvent event : unpublishedEvents) {
            try {
                // Publicar no Kafka
                kafkaProducer.send(
                    event.getAggregateType(), 
                    event.getEventData()
                );
                
                // Marcar como processado
                event.setProcessedAt(Instant.now());
                outboxRepository.save(event);
                
            } catch (Exception e) {
                log.error("Erro ao publicar evento: {}", event.getId(), e);
                // Será tentado novamente no próximo ciclo
            }
        }
    }
}
```

### Vantagens do Outbox Pattern

1. **Garantia transacional**: Estado e evento sempre consistentes
2. **Eventos de domínio**: Eventos refletem conceitos de negócio
3. **Flexibilidade**: Controle total sobre estrutura dos eventos
4. **Resiliência**: Retry automático de eventos não publicados

### Desvantagens do Outbox Pattern

1. **Mudança no código**: Requer modificação em toda operação que gera eventos
2. **Polling**: Pode gerar carga adicional no banco
3. **Manutenção**: Necessário limpar eventos antigos
4. **Delay**: Latência depende da frequência do polling

### Exemplo de Evento Outbox

```json
{
  "eventId": "550e8400-e29b-41d4-a716-446655440000",
  "eventType": "OrderCreated",
  "aggregateId": "123e4567-e89b-12d3-a456-426614174000",
  "aggregateType": "Order",
  "timestamp": "2024-02-20T10:15:30Z",
  "payload": {
    "orderId": "123e4567-e89b-12d3-a456-426614174000",
    "customerId": "987e6543-e21b-12d3-a456-426614174000",
    "items": [
      {
        "productId": "prod-001",
        "quantity": 2,
        "price": 50.00
      }
    ],
    "totalAmount": 100.00,
    "status": "PENDING"
  }
}
```

## Comparação Detalhada

### Tabela Comparativa

| Aspecto | CDC | Outbox Pattern |
|---------|-----|----------------|
| **Implementação** | Infraestrutura externa | Código na aplicação |
| **Garantia de entrega** | At-least-once | Exactly-once (com idempotência) |
| **Performance** | Leitura assíncrona do log | Polling do banco |
| **Tipo de eventos** | Mudanças de dados | Eventos de domínio |
| **Flexibilidade** | Limitada à estrutura do BD | Total controle |
| **Complexidade operacional** | Alta | Média |
| **Impacto no código** | Nenhum | Significativo |
| **Latência** | Baixa (ms) | Depende do polling (segundos) |
| **Histórico** | Completo | Configurável |
| **Custo** | Infraestrutura adicional | Espaço no BD principal |

### Quando usar CDC?

CDC é ideal quando:

1. **Sistema legado**: Não é possível modificar o código
2. **Auditoria completa**: Precisa capturar TODAS as mudanças
3. **Multi-tenant**: Várias aplicações modificam o mesmo banco
4. **Replicação**: Objetivo é sincronizar dados entre sistemas
5. **Event sourcing retroativo**: Quer capturar histórico existente

**Exemplo de caso de uso**: 
Sistema de e-commerce legado onde múltiplas aplicações (web, mobile, backoffice) modificam o catálogo de produtos e você precisa sincronizar com um sistema de busca Elasticsearch.

### Quando usar Outbox Pattern?

Outbox Pattern é melhor quando:

1. **Eventos de domínio**: Eventos representam conceitos de negócio
2. **Controle total**: Precisa definir exatamente o que publicar
3. **Garantias fortes**: Consistência transacional é crítica
4. **Greenfield**: Desenvolvendo do zero ou pode modificar código
5. **GDPR/Privacidade**: Precisa filtrar dados sensíveis

**Exemplo de caso de uso**: 
Sistema de pagamentos onde cada transação precisa gerar eventos específicos de domínio (PaymentAuthorized, PaymentCaptured, PaymentRefunded) com garantia de que o evento só é publicado se a transação foi persistida.

## Implementação Híbrida

Em alguns casos, faz sentido combinar ambas as abordagens:

```java
// 1. Aplicação escreve evento de domínio no Outbox
@Transactional
public void processPayment(Payment payment) {
    paymentRepository.save(payment);
    outboxRepository.save(new PaymentProcessedEvent(payment));
}

// 2. CDC captura mudanças para auditoria
// Configuração Debezium captura tabela payments

// 3. Consumidores diferentes para propósitos diferentes
@KafkaListener(topics = "payments.outbox.events")
public void handleBusinessEvent(PaymentProcessedEvent event) {
    // Lógica de negócio
}

@KafkaListener(topics = "payments.cdc.changes")
public void handleAuditEvent(CDCEvent event) {
    // Auditoria e compliance
}
```

## Considerações de Implementação

### Para CDC

1. **Permissões do banco**: CDC precisa acesso ao transaction log
2. **Retenção de logs**: Configure adequadamente para não perder eventos
3. **Schema evolution**: Mudanças no banco podem quebrar consumidores
4. **Snapshot inicial**: Como lidar com dados existentes
5. **Transformação**: Geralmente necessário transformar eventos

### Para Outbox

1. **Limpeza**: Implemente rotina para deletar eventos antigos
2. **Ordenação**: Garanta ordem de processamento quando necessário
3. **Idempotência**: Consumidores devem ser idempotentes
4. **Particionamento**: Use aggregate_id para garantir ordem por entidade
5. **Monitoramento**: Alerte sobre eventos não processados

## Ferramentas e Tecnologias

### Para CDC
- **Debezium**: Solução open source mais popular
- **AWS DMS**: Serviço gerenciado da AWS
- **Oracle GoldenGate**: Solução enterprise
- **Maxwell**: CDC para MySQL
- **Postgres Logical Replication**: Nativo do PostgreSQL

### Para Outbox
- **Frameworks**: Spring, Axon Framework
- **Bibliotecas**: Eventuate Tram, MassTransit
- **Patterns**: Polling Publisher, Transaction Log Tailing
- **Bancos**: Qualquer banco relacional com suporte a transações

## Conclusão

Tanto CDC quanto Outbox Pattern são soluções válidas para event sourcing, cada uma com seus méritos. A escolha depende do contexto:

- **Use CDC** quando precisar de uma solução não-invasiva que capture todas as mudanças automaticamente
- **Use Outbox Pattern** quando precisar de controle fino sobre os eventos e garantias transacionais fortes

Em arquiteturas maduras, é comum ver ambos os padrões coexistindo, cada um servindo a propósitos específicos. O importante é entender os trade-offs e escolher a ferramenta certa para cada problema.

Lembre-se: não existe bala de prata. A melhor solução é aquela que resolve seu problema específico com o menor custo de complexidade possível.

## Referências e Leituras Adicionais

- [Debezium Documentation](https://debezium.io/documentation/)
- [Microservices.io - Outbox Pattern](https://microservices.io/patterns/data/transactional-outbox.html)
- [Event Sourcing - Martin Fowler](https://martinfowler.com/eaaDev/EventSourcing.html)
- [Building Event-Driven Microservices - Adam Bellemare](https://www.oreilly.com/library/view/building-event-driven-microservices/9781492057888/)