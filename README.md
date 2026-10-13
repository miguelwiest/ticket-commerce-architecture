# Arquitetura — Venda de Ingressos para Mega Show (Rock in Rio)

Desafio de arquitetura: sistema de venda de ingressos para um mega show extremamente concorrido, onde o número de ingressos é muito inferior à quantidade de acessos simultâneos no dia da venda.

## Os dois problemas centrais

**1. Ninguém pode comprar um ingresso que não existe mais.**
Não pode existir uma janela onde duas pessoas "vendem" o mesmo lugar, nem um usuário finalizar a compra sem estoque real.

**2. Justiça na fila.**
Um cliente com internet mais lenta não pode perder o ingresso para alguém com internet mais veloz.

## Decisões de arquitetura

Sistema **event-driven, horizontal e elástico**, com a lógica de estoque descentralizada do banco de dados relacional.

### O caminho síncrono (quente)
`Fãs → Front Door + WAF → Sala de Espera Virtual → API Gateway → Serviço de Ingressos → Redis`

O usuário entra em uma **sala de espera virtual** e recebe um **place na ordem de chegada**. Quando a sala libera, ele posta a reserva. O **Serviço de Ingressos executa um `DECR` atômico (script Lua) no Redis** — a autoridade do estoque. Ou tem estoque, ou a venda não acontece. Não há janela de race condition.

O ingresso fica em **hold com TTL de 10 minutos** enquanto o pagamento processa. Expira sozinho se a pessoa desistir.

### Por que a sala de espera resolve a justiça
O critério de desempate passa a ser **ordem de chegada na fila**, não latência de rede. Quem entrou primeiro, fecha primeiro — independente da velocidade do link.

Para não penalizar conexões lentas, o navegador manda apenas um **heartbeat leve a cada 30s**, em vez de polling agressivo. Uma conexão 3G mantém o lugar sem precisar "vencer" ninguém. As atualizações chegam por **push via SignalR**, não por polling.

### O backbone de eventos (assíncrono)
`RabbitMQ (ticket.exchange)` com filas para `reserva`, `pagamento`, `notificacao` e `auditoria`.

- O Serviço de Ingressos publica eventos via **outbox pattern** — o evento e o estado do banco são gravados na mesma transação, então nenhum evento se perde.
- **Cada serviço consome sua fila no próprio ritmo**, com prefetch limitado. Isso cria backpressure natural: um pico no pagamento não derruba a auditoria.
- **DLQ + retry exponencial** absorvem picos e falhas pontais sem perder dados.
- Eventos são **idempotentes** (chave = id da transação), então reprocessar é seguro.

### WebSockets descentralizados
**Azure SignalR Service** faz o fan-out das notificações. Não existe um servidor central de WebSocket virando gargalo — o SignalR Service escala para centenas de milhares de conexões e tira o push do caminho crítico.

### Saga e compensação
O **Orquestrador Saga** coordena o fluxo e compensa em qualquer falha: se o pagamento for recusado ou um serviço cair, o hold é devolvido ao estoque.

## Como cada requisito é atendido

| Requisito | Solução |
|---|---|
| Nenhuma venda sem estoque | `DECR` atômico em Lua no Redis + hold com TTL |
| Internet lenta não perde o lugar | Sala de espera por ordem de chegada + heartbeat a cada 30s |
| Demanda absurda no dia | Tudo stateless + CDN servindo o app + auto-scale |
| Notificações não derrubam a venda | SignalR Service descentralizado, fora do caminho crítico |
| Resiliência | Outbox pattern, DLQ, retry exponencial, idempotência, geo-DR |

## Stack

- **Nuvem:** Microsoft Azure (C# / .NET)
- **Mensageria:** RabbitMQ (topic exchange, durable, publisher confirms)
- **Cache / estoque:** Redis em cluster, com persistência habilitada
- **Banco relacional:** Azure SQL Database (pedidos e ingressos emitidos, fora do caminho crítico)
- **Push:** Azure SignalR Service

---

`arquitetura-ingressos.png` é o desenho. `arquitetura-ingressos.html` é a mesma arquitetura como SVG interativo (abre em qualquer navegador).
