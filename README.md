# APS 3 - Cadastro e Priorização de Ordens de Entrega

- **Aluno:** Dheyvid de Bastos Pinto
- **Disciplina:** Modelagem de Software (Prof. Sidney L. da Silva)
- **Projeto do grupo:** Sistema de Logística de Entregas Autônomas

## O que é a funcionalidade

Escolhi o **cadastro de ordens de entrega**, que é por onde toda entrega começa no sistema.

O operador informa de onde sai o material, para onde vai, qual material, a quantidade e o prazo. Antes de aceitar a ordem, o sistema confere duas coisas ao mesmo tempo:
- se a origem e o destino existem (API de Geolocalização);
- se tem material suficiente no galpão (API de Armazéns / WMS).

Se estiver tudo certo, o material é reservado, o sistema calcula a prioridade da ordem e coloca ela numa fila do RabbitMQ. Se não, a ordem é rejeitada e o operador vê o motivo.

A prioridade leva em conta a urgência e o prazo. Se uma ordem ficar muito tempo esperando, a prioridade dela aumenta.

## Operações (regra de consistência)

As mensagens do diagrama de sequência são operações de quem recebe:

- Aplicação Web: `preencherOrdem(dados)`
- API de Orquestração: `criarOrdem(NovaOrdemDTO)`, `calcularPrioridade(OrdemDTO)`
- API de Geolocalização: `validarPontos(origem, destino)`
- WMS: `consultarEstoque(materialId, galpaoId)`, `reservarMaterial(ReservaDTO)`
- Banco de Dados: `salvarOrdem(NovaOrdemDTO)`, `atualizarOrdem(ordemId, status, prioridade)`
- RabbitMQ: `publicarOrdem(OrdemDTO, prioridade)`

## 1. Diagrama de Sequência

Usei `->>` para chamadas síncronas, `-)` para a mensagem assíncrona da fila e `-->>` para os retornos com DTO. O `par` mostra as duas validações rodando juntas.

```mermaid
sequenceDiagram
    actor Operador
    participant Web as Aplicação Web
    participant API as API de Orquestração
    participant Geo as API de Geolocalização
    participant WMS as API de Armazéns (WMS)
    participant BD as Banco de Dados
    participant Fila as RabbitMQ

    Operador->>Web: preencherOrdem(dados)
    Web->>API: criarOrdem(NovaOrdemDTO)
    API->>BD: salvarOrdem(NovaOrdemDTO)
    BD-->>API: OrdemDTO (status = SOLICITADA)

    par validar pontos
        API->>Geo: validarPontos(origem, destino)
        Geo-->>API: PontosDTO (validos)
    and verificar estoque
        API->>WMS: consultarEstoque(materialId, galpaoId)
        WMS-->>API: EstoqueDTO (quantidade)
    end

    alt pontos válidos e estoque >= quantidade pedida
        API->>WMS: reservarMaterial(ReservaDTO)
        WMS-->>API: ReservaDTO (reservaId)
        API->>API: calcularPrioridade(OrdemDTO)
        API->>BD: atualizarOrdem(ordemId, NA_FILA, prioridade)
        BD-->>API: OrdemDTO
        API-)Fila: publicarOrdem(OrdemDTO, prioridade)
        API-->>Web: OrdemResponseDTO (201, status = NA_FILA)
    else ponto inválido ou estoque insuficiente
        API->>BD: atualizarOrdem(ordemId, REJEITADA)
        BD-->>API: OrdemDTO
        API-->>Web: ErroDTO (422, motivo)
    end

    Web-->>Operador: mostra o resultado
```

## 2. Diagrama de Atividades

```mermaid
flowchart TB
    subgraph R1[Operador]
        I((início))
        A1[Preencher origem, destino, material e prazo]
        A2[Corrigir os dados]
        A3[Acompanhar a ordem na fila]
        F((fim))
    end

    subgraph R2[API de Orquestração]
        B1[Salvar ordem como SOLICITADA]
        FORK[ ]:::barra
        JOIN[ ]:::barra
        D1{Ordem válida?}
        B2[Marcar como REJEITADA]
        B3[Calcular prioridade]
        B4[Publicar na fila com prioridade]
    end

    subgraph R3[Serviços Externos]
        C1[Validar origem e destino - Geolocalização]
        C2[Consultar estoque no galpão - WMS]
        C3[Reservar material - WMS]
    end

    I --> A1 --> B1 --> FORK
    FORK --> C1 --> JOIN
    FORK --> C2 --> JOIN
    JOIN --> D1
    D1 -->|"[pontos inválidos ou estoque < quantidade]"| B2 --> A2 --> A1
    D1 -->|"[pontos válidos e estoque >= quantidade]"| C3 --> B3 --> B4 --> A3 --> F

    classDef barra fill:#000,stroke:#000,color:#000
```

As raias são Operador, API de Orquestração e Serviços Externos. O fork/join é a mesma parte do `par` no diagrama de sequência.

## 3. Diagrama de Estados (OrdemEntrega)

```mermaid
stateDiagram-v2
    [*] --> Solicitada

    Solicitada --> EmValidacao : validar()
    Solicitada --> Cancelada : cancelar() [usuario == ADMIN]

    state EmValidacao {
        [*] --> ConferindoDados
        ConferindoDados --> ReservandoMaterial : dadosOk [pontosValidos e estoque >= quantidade]
    }

    ConferindoDados --> Rejeitada : dadosInvalidos [ponto inválido ou estoque < quantidade] / informarMotivo()
    ReservandoMaterial --> NaFila : reservaConfirmada / calcularPrioridade()

    state "NaFila" as NaFila
    NaFila : entry / publicarNaFila()
    NaFila : do / verificarTempoDeEspera()

    NaFila --> NaFila : tempoExcedido [espera > SLA] / aumentarPrioridade()
    NaFila --> Planejada : despachoPegouOrdem / criarMissao()
    NaFila --> Cancelada : cancelar() [usuario == ADMIN] / liberarReserva()

    Planejada --> EmAndamento : agvSaiu
    EmAndamento --> Entregue : entregaConfirmada

    Rejeitada --> [*]
    Cancelada --> [*]
    Entregue --> [*]
```

Os estados finais são Entregue, Rejeitada e Cancelada. Uma ordem rejeitada não volta: se o operador corrigir os dados, ele cria uma ordem nova. A ordem só pode ser cancelada por um ADMIN e só antes de virar missão.

## 4. C4 - Nível 2 (Containers)

```mermaid
flowchart TB
    op["Operador Logístico<br/>[Pessoa]"]

    subgraph sistema[Sistema de Entregas Autônomas]
        web["Aplicação Web<br/>[Container: React]<br/>Cadastro e acompanhamento"]
        api["API de Orquestração<br/>[Container: Node.js + NestJS]<br/>Ordens, prioridade e despacho"]
        planejador["Planejador de Rotas<br/>[Container: Python]<br/>Calcula rotas"]
        tel["Serviço de Telemetria<br/>[Container: Node.js]<br/>Recebe dados dos AGVs"]
        bd[("Banco de Dados<br/>[Container: PostgreSQL]")]
        fila["RabbitMQ<br/>[Container: fila de ordens e eventos]"]
        broker["Broker MQTT<br/>[Container: Mosquitto]"]
    end

    geo["API de Geolocalização<br/>[Sistema externo]"]
    wms["API de Armazéns - WMS<br/>[Sistema externo]"]
    agv["AGVs<br/>[Sistema externo]"]

    op -->|HTTPS| web
    web -->|"HTTPS / REST (JSON)"| api
    api -->|WebSocket| web
    api -->|SQL| bd
    api -->|"HTTPS / REST"| geo
    api -->|"HTTPS / REST"| wms
    api -->|"AMQP (publica ordem)"| fila
    fila -->|"AMQP (consome ordem)"| api
    api -->|gRPC| planejador
    planejador -->|"HTTPS / REST"| geo
    api -->|MQTT| broker
    broker <-->|MQTT| agv
    broker -->|MQTT| tel
    tel -->|SQL| bd
    tel -->|AMQP| fila

    classDef pessoa fill:#08427b,color:#fff
    classDef container fill:#1168bd,color:#fff
    classDef ext fill:#999,color:#fff
    class op pessoa
    class web,api,planejador,tel,bd,fila,broker container
    class geo,wms,agv ext
```

## 5. ADR

[ADR 0001 - Fila com prioridade no RabbitMQ para as ordens](docs/architecture/adr-0001.md)
