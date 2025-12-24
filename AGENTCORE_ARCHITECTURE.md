# Arquitetura Segura: Agents e MCPs com Amazon AgentCore

## Contexto

- Comunicação 100% privada (sem tráfego pela internet)
- **MCPs de domínios diferentes acessados exclusivamente via AgentCore Gateway**
- **Comunicação Agent-to-Agent (A2A) direta via AgentCore Runtime com OAuth 2.0**
- Integração com recursos em VPC privada (RDS, ElastiCache, APIs internas)

---

## Tipos de Comunicação

| Protocolo | Propósito | Porta | Via Gateway | Autenticação |
|-----------|-----------|-------|-------------|--------------|
| **MCP** | Agent → Tools | 8000 | Sim (cross-domain) | OAuth 2.0 (Cognito M2M) |
| **A2A** | Agent → Agent | 9000 | Não (direto via Runtime) | OAuth 2.0 (Cognito M2M) |

> **Importante**: O AgentCore Gateway suporta **somente MCP**. A comunicação A2A é feita diretamente entre Runtimes.

---

## Definição de Domínios

| Tipo | Descrição | Acesso MCP | Acesso A2A |
|------|-----------|------------|------------|
| **Mesmo domínio** | MCPs/Agents do mesmo produto/time | Direto | Direto (com OAuth) |
| **Domínio diferente (interno)** | MCPs/Agents de outros produtos/times | Via Gateway | Direto via Runtime + OAuth |
| **Domínio diferente (externo)** | Serviços externos (3rd Party MCPs) | Via Gateway | Direto via Runtime + OAuth |

### Por que usar Gateway para MCPs de domínios diferentes?

| Benefício | Descrição |
|-----------|-----------|
| **Controle de acesso** | OAuth scopes definem quais agents acessam quais MCPs |
| **Auditoria** | CloudTrail registra todas as chamadas cross-domain |
| **Governança** | Time de plataforma controla integrações entre produtos |
| **Autenticação** | OAuth 2.0 (Cognito) para MCPs |
| **Rate limiting** | Por target |
| **Billing** | Por domínio |

### Por que A2A não usa AgentCore Gateway?

| Aspecto | Explicação |
|---------|------------|
| **Protocolo diferente** | AgentCore Gateway suporta somente MCP, não A2A |
| **Comunicação direta** | A2A usa JSON-RPC 2.0 direto entre Runtimes |
| **OAuth nativo** | A2A tem autenticação OAuth 2.0 built-in no protocolo |
| **Agent Card** | Discovery via `/.well-known/agent-card.json` no Runtime |

---

## Componentes de Autenticação

| Componente | Função | Localização |
|------------|--------|-------------|
| **Keycloak** | Identity Provider (IdP) para usuários | On-premises / VPC |
| **Cognito** | Identity Provider para server-to-server (M2M) | AWS |
| **Axway API Gateway** | API Gateway corporativo | DMZ / Edge |
| **AgentCore Gateway** | Gateway MCP cross-domain | AWS AgentCore |
| **AgentCore Identity** | Credential Provider (OAuth externo) | AWS AgentCore |

### Tipos de Autenticação

| Tipo | Descrição | Uso |
|------|-----------|-----|
| **User Token** | Token OIDC do usuário (Keycloak) | Usuário → Axway → Agent |
| **Machine-to-Machine (M2M)** | Client Credentials OAuth 2.0 (Cognito) | Agent → MCP, Agent → Gateway, Agent → Agent |
| **OAuth Externo** | OAuth via AgentCore Identity | Gateway → Serviços externos |

---

## Matriz de Autenticação por Cenário

| Origem | Destino | Tipo Auth | Token/Credencial | Validação |
|--------|---------|-----------|------------------|-----------|
| Usuário | Axway | User Token (OIDC) | Keycloak JWT | Axway → Keycloak JWKS |
| Axway | Agent | User Token (propagado) | Keycloak JWT | Agent (opcional) |
| Agent | MCP mesmo domínio | M2M OAuth | Cognito Client Credentials | MCP → Cognito JWKS |
| Agent | AgentCore Gateway | M2M OAuth | Cognito Client Credentials | Gateway → Cognito JWKS |
| Gateway | MCP domínio diferente | M2M OAuth | Cognito (interno) | MCP → Cognito JWKS |
| Gateway | MCP externo | OAuth Externo | AgentCore Identity | Serviço externo |
| Agent | Agent (A2A) | M2M OAuth | Cognito Client Credentials | Agent → Cognito JWKS |

---

## Fluxo Consolidado de Autenticação

Este diagrama mostra o fluxo completo desde o usuário até MCPs e Agents.

```mermaid
sequenceDiagram
    participant User as User
    participant Keycloak as Keycloak
    participant Axway as Axway
    participant Cognito as Cognito
    participant Agent as CRM Agent
    participant Gateway as Gateway
    participant MCP_CRM as MCP CRM
    participant MCP_FIN as MCP Finance
    participant Agent2 as Finance Agent
    
    Note over User,Keycloak: 1 - USER AUTHENTICATION
    User->>Keycloak: Login OIDC
    Keycloak-->>User: User Token JWT
    
    Note over User,Agent: 2 - REQUEST VIA AXWAY
    User->>Axway: Request + Bearer Token
    Axway->>Keycloak: Validate Token
    Axway->>Agent: Forward + Context Headers
    
    Note over Agent,Cognito: 3 - AGENT OBTAINS M2M TOKEN
    Agent->>Cognito: Client Credentials
    Cognito-->>Agent: M2M Token
    
    Note over Agent,MCP_CRM: 4 - MCP SAME DOMAIN via Gateway
    Agent->>Gateway: invoke_gateway crm-customers + Bearer M2M
    Gateway->>MCP_CRM: tools/call
    MCP_CRM-->>Gateway: Result
    Gateway-->>Agent: Result
    
    Note over Agent,MCP_FIN: 5 - MCP OTHER DOMAIN via Gateway
    Agent->>Gateway: invoke_gateway finance-invoices + Bearer M2M
    Gateway->>MCP_FIN: tools/call
    MCP_FIN-->>Gateway: Result
    Gateway-->>Agent: Result
    
    Note over Agent,Agent2: 6 - A2A M2M DIRECT
    Agent->>Agent2: GET /.well-known/agent-card.json
    Agent->>Cognito: Client Credentials M2M
    Cognito-->>Agent: M2M Token
    Agent->>Agent2: POST /message/send + Bearer M2M
    Agent2-->>Agent: Result
    
    Agent-->>User: Consolidated Response
```

---

## Cenário 1: Agent no EKS + MCP no AgentCore Runtime

### Visão Geral

- **Agents**: Rodam no EKS (seu cluster Kubernetes)
- **Todos os MCPs**: Acessados via AgentCore Gateway (mesmo domínio ou diferente)
- **A2A**: Comunicação externa via Axway (exposição) ou direta (consumo)

### Arquitetura

```mermaid
flowchart LR
    subgraph EKS["EKS Cluster"]
        agent["CRM Agent"]
    end
    
    subgraph AUTH["Internal Auth"]
        cognito["Cognito M2M"]
    end
    
    subgraph AGENTCORE["AgentCore"]
        subgraph GW["Gateway"]
            t1["target: crm-customers"]
            t2["target: finance-invoices"]
            t3["target: 3rdparty-mcp"]
        end
        
        subgraph RT1["Runtime CRM"]
            mcp1["mcp-crm-customers"]
        end
        
        subgraph RT2["Runtime Finance"]
            mcp2["mcp-finance-invoices"]
        end
        
        identity["Identity - External OAuth"]
    end
    
    subgraph DB["Internal Resources"]
        microservice["Microservice"]
    end
    
    subgraph EXT["External Service"]
        extmcp["3rd Party MCP"]
        extauth["Partner OAuth Server"]
    end
    
    agent -->|"1 M2M Token"| cognito
    agent -->|"2 Request"| GW
    t1 -->|"Cognito M2M"| mcp1
    t2 -->|"Cognito M2M"| mcp2
    t3 --> identity
    identity -->|"3 OAuth Token"| extauth
    identity -->|"4 Request"| extmcp
    mcp1 --> microservice
    mcp2 --> microservice
    
    style agent fill:#4caf50,color:#fff
    style GW fill:#9c27b0,color:#fff
    style cognito fill:#ff9800,color:#fff
    style identity fill:#2196f3,color:#fff
    style extauth fill:#9c27b0,color:#fff
    style extmcp fill:#ff5722,color:#fff
```

### Fluxo de Comunicação

| Passo | Origem | Destino | Via | Autenticação |
|-------|--------|---------|-----|--------------|
| 1 | Agent EKS | Cognito | VPC Endpoint | Client Credentials |
| 2 | Agent EKS | Gateway | VPC Endpoint | Cognito M2M Token |
| 3a | Gateway | MCP interno | Runtime | Cognito M2M (interno) |
| 3b | Gateway | 3rd Party MCP | Identity | OAuth Token do Parceiro |

> **Nota**: MCPs internos (Runtime) usam Cognito M2M. MCPs externos (3rd Party) usam OAuth do parceiro via Identity.

---

## Cenário 2: Agents e MCPs no AgentCore Runtime

### Visão Geral

- **Agents**: Hospedados no AgentCore Runtime (serverless)
- **Todos os MCPs**: Acessados via AgentCore Gateway
- **A2A**: Comunicação externa via Axway (exposição) ou direta (consumo)
- **Tudo gerenciado pela AWS**

### Arquitetura

```mermaid
flowchart LR
    user["User"]
    
    subgraph AUTH["Internal Auth"]
        cognito["Cognito M2M"]
    end
    
    subgraph AGENTCORE["AgentCore"]
        subgraph RT1["Runtime CRM"]
            agent["CRM Agent"]
        end
        
        subgraph GW["Gateway"]
            t1["target: crm-customers"]
            t2["target: finance-invoices"]
            t3["target: 3rdparty-mcp"]
        end
        
        subgraph RT1MCP["Runtime CRM MCPs"]
            mcp1["mcp-crm-customers"]
        end
        
        subgraph RT2["Runtime Finance"]
            mcp2["mcp-finance-invoices"]
        end
        
        identity["Identity - External OAuth"]
    end
    
    subgraph DB["Internal Resources"]
        microservice["Microservice"]
    end
    
    subgraph EXT["External Service"]
        extmcp["3rd Party MCP"]
        extauth["Partner OAuth Server"]
    end
    
    user --> agent
    agent -->|"1 M2M Token"| cognito
    agent -->|"2 Request"| GW
    t1 -->|"Cognito M2M"| mcp1
    t2 -->|"Cognito M2M"| mcp2
    t3 --> identity
    identity -->|"3 OAuth Token"| extauth
    identity -->|"4 Request"| extmcp
    mcp1 --> microservice
    mcp2 --> microservice
    
    style agent fill:#4caf50,color:#fff
    style GW fill:#9c27b0,color:#fff
    style cognito fill:#ff9800,color:#fff
    style identity fill:#2196f3,color:#fff
    style user fill:#607d8b,color:#fff
    style extauth fill:#9c27b0,color:#fff
    style extmcp fill:#ff5722,color:#fff
```

---

## Comparação dos Cenários

| Aspecto | Cenário 1 (Agent no EKS) | Cenário 2 (Agent no Runtime) |
|---------|--------------------------|------------------------------|
| **Onde roda o Agent** | EKS (seu cluster) | AgentCore Runtime (AWS) |
| **MCPs mesmo domínio** | AgentCore Runtime (direto) | AgentCore Runtime (direto) |
| **MCPs domínio diferente** | Via Gateway | Via Gateway |
| **Gerenciamento do Agent** | Você (K8s) | AWS (serverless) |
| **Escalabilidade** | HPA/KEDA | Automática |
| **Isolamento** | Pod/Namespace | MicroVM por sessão |
| **Customização** | Total | Limitada ao container |
| **Custo fixo** | EKS + VPC endpoints (~$204/mês) | VPC endpoints (~$71/mês) |

### Quando Usar Cada Cenário

| Situação | Recomendação |
|----------|--------------|
| Já tem EKS e quer manter controle | Cenário 1 |
| Quer simplicidade operacional | Cenário 2 |
| Agent precisa de libs/deps específicas | Cenário 1 |
| Precisa de isolamento forte por sessão | Cenário 2 |
| Custo previsível é prioridade | Cenário 1 |
| Escala imprevisível | Cenário 2 |

---

## Propagação de Contexto do Usuário

Para garantir rastreabilidade e auditoria completa, o contexto do usuário deve ser propagado nas chamadas M2M. Existem duas abordagens principais:

### Abordagem 1: Context Headers

Propagar headers com informações do usuário em todas as chamadas.

#### Headers de Contexto

| Header | Propósito |
|--------|-----------|
| **X-On-Behalf-Of** | ID do usuário original |
| **X-User-Email** | Email do usuário |
| **X-User-Roles** | Roles/permissões do usuário |
| **X-Correlation-Id** | UUID único para rastreabilidade E2E |
| **X-Original-Token-Hash** | Hash do token original (integridade) |

#### Fluxo com Context Headers

```mermaid
sequenceDiagram
    participant User as User
    participant Keycloak as Keycloak
    participant Axway as Axway
    participant Agent as CRM Agent
    participant Cognito as Cognito
    participant Gateway as Gateway
    participant MCP as MCP Finance
    participant Logs as CloudWatch
    
    Note over User,Keycloak: 1 - USER LOGIN
    User->>Keycloak: Login OIDC
    Keycloak-->>User: User Token sub:user123 email:joao@emp.com
    
    Note over User,Agent: 2 - REQUEST VIA AXWAY
    User->>Axway: Request + Bearer user_token
    Axway->>Keycloak: Validate Token
    Keycloak-->>Axway: Valid Token
    
    Note over Axway,Agent: 3 - AXWAY PROPAGATES CONTEXT
    Axway->>Agent: Request + Bearer user_token
    Note right of Axway: Headers propagated by Axway
    Note right of Axway: X-User-Id: user123
    Note right of Axway: X-User-Email: joao@emp.com
    Note right of Axway: X-User-Roles: sales,viewer
    Note right of Axway: X-Correlation-Id: uuid-xxx
    
    Note over Agent,Cognito: 4 - AGENT OBTAINS M2M TOKEN
    Agent->>Agent: Extract and validate user context
    Agent->>Agent: Calculate original token hash
    Agent->>Cognito: Client Credentials
    Cognito-->>Agent: M2M Token
    
    Note over Agent,MCP: 5 - AGENT CALLS GATEWAY WITH CONTEXT
    Agent->>Gateway: invoke_gateway + Bearer M2M Token
    Note right of Agent: Context headers propagated
    Note right of Agent: X-On-Behalf-Of: user123
    Note right of Agent: X-User-Email: joao@emp.com
    Note right of Agent: X-User-Roles: sales,viewer
    Note right of Agent: X-Correlation-Id: uuid-xxx
    Note right of Agent: X-Original-Token-Hash: sha256-abc
    
    Gateway->>Cognito: Validate M2M Token
    Cognito-->>Gateway: Valid Token
    Gateway->>Logs: Audit Log user123 via crm-agent
    
    Note over Gateway,MCP: 6 - GATEWAY PROPAGATES HEADERS TO MCP
    Gateway->>MCP: tools/call get_invoices
    Note right of Gateway: Headers propagated to MCP
    Note right of Gateway: X-On-Behalf-Of: user123
    Note right of Gateway: X-Correlation-Id: uuid-xxx
    
    MCP->>MCP: Validate context present
    MCP->>MCP: Apply policies by user/role
    MCP->>Logs: Audit Log user123 via crm-agent called get_invoices
    MCP-->>Gateway: Result
    
    Gateway-->>Agent: Result
    Agent->>Logs: Audit Log user123 processed request
    Agent-->>User: Response
```

---

### Abordagem 2: Token Exchange (RFC 8693)

Trocar o user token por um M2M token que já contém claims do usuário no campo `act` (actor).

#### Vantagens do Token Exchange

| Aspecto | Benefício |
|---------|-----------|
| **Segurança** | Claims do usuário assinados no token, não em headers |
| **Padrão** | RFC 8693 é um padrão OAuth 2.0 |
| **Verificável** | Token pode ser validado criptograficamente |
| **Auditoria** | Identidade do Agent E do usuário no mesmo token |

#### Fluxo com Token Exchange

```mermaid
sequenceDiagram
    participant User as User
    participant Keycloak as Keycloak
    participant Axway as Axway
    participant Agent as CRM Agent
    participant Cognito as Cognito
    participant Gateway as Gateway
    participant MCP as MCP Finance
    participant Logs as CloudWatch
    
    Note over User,Keycloak: 1 - USER LOGIN
    User->>Keycloak: Login OIDC
    Keycloak-->>User: User Token sub:user123 email:joao@emp.com
    
    Note over User,Agent: 2 - REQUEST VIA AXWAY
    User->>Axway: Request + Bearer user_token
    Axway->>Keycloak: Validate Token
    Axway->>Agent: Forward + Bearer user_token
    
    Note over Agent,Cognito: 3 - TOKEN EXCHANGE RFC 8693
    Agent->>Cognito: POST /oauth2/token
    Note right of Agent: grant_type: token-exchange
    Note right of Agent: subject_token: user_token
    Note right of Agent: subject_token_type: jwt
    Note right of Agent: scope: gateway/invoke mcp-finance/read
    
    Cognito->>Keycloak: Validate user_token via JWKS
    Keycloak-->>Cognito: Valid Token
    Cognito->>Cognito: Extract user claims
    Cognito->>Cognito: Generate M2M Token with actor claim
    Cognito-->>Agent: M2M Token with user claims
    Note right of Cognito: Token contains:
    Note right of Cognito: sub: crm-agent
    Note right of Cognito: act: sub:user123
    Note right of Cognito: user_email: joao@emp.com
    Note right of Cognito: scope: gateway/invoke
    
    Note over Agent,MCP: 4 - AGENT CALLS GATEWAY
    Agent->>Gateway: invoke_gateway + Bearer exchanged_token
    Note right of Agent: X-Correlation-Id: uuid-xxx
    
    Gateway->>Cognito: Validate Token
    Cognito-->>Gateway: Valid Token
    Gateway->>Gateway: Extract actor claim - user123
    Gateway->>Logs: Audit Log user123 via crm-agent
    
    Note over Gateway,MCP: 5 - GATEWAY CALLS MCP
    Gateway->>MCP: tools/call get_invoices + Bearer gateway_token
    Note right of Gateway: Token contains act claim with user
    
    MCP->>Cognito: Validate Token
    Cognito-->>MCP: Valid Token
    MCP->>MCP: Extract actor claim - original user
    MCP->>MCP: Apply policies by user
    MCP->>Logs: Audit Log user123 via crm-agent called get_invoices
    MCP-->>Gateway: Result
    
    Gateway-->>Agent: Result
    Agent-->>User: Response
```

---

### Comparação das Abordagens

| Aspecto | Context Headers | Token Exchange |
|---------|-----------------|----------------|
| **Segurança** | Headers podem ser forjados | Claims assinados no token |
| **Complexidade** | Simples de implementar | Requer suporte a RFC 8693 |
| **Padrão** | Convenção customizada | RFC 8693 OAuth 2.0 |
| **Validação** | Confiança no caller | Verificação criptográfica |
| **Suporte Cognito** | Nativo | Requer configuração adicional |
| **Overhead** | Baixo | Chamada extra ao IdP |

### Quando usar cada abordagem

| Cenário | Recomendação |
|---------|--------------|
| Ambiente interno confiável | Context Headers |
| Compliance rigoroso | Token Exchange |
| Simplicidade operacional | Context Headers |
| Múltiplos IdPs | Token Exchange |
| Alta segurança | Token Exchange |

### Benefícios de ambas abordagens

- ✅ **Auditoria completa**: logs mostram "user123 via crm-agent chamou get_invoices"
- ✅ **Controle de acesso granular** por usuário/role no MCP
- ✅ **Rastreabilidade E2E** com correlation ID
- ✅ **Integridade verificável** com hash do token original (Context Headers) ou assinatura (Token Exchange)

---

## Comunicação Agent-to-Agent (A2A)

### Visão Geral

| Aspecto | Descrição |
|---------|-----------|
| **Protocolo** | JSON-RPC 2.0 sobre HTTP(S) |
| **Porta** | 9000 (padrão AgentCore) |
| **Discovery** | Agent Card em `/.well-known/agent-card.json` |
| **Autenticação** | OAuth 2.0 Bearer Token (Cognito M2M) |
| **Streaming** | Server-Sent Events (SSE) |

> **Importante**: A2A é sempre DIRETO entre Agents. Não usa AgentCore Gateway.

---

### Tipos de Comunicação A2A

| Tipo | Descrição | Autenticação | Gateway |
|------|-----------|--------------|---------|
| **Interno (mesmo domínio)** | Agents do mesmo produto/time | Cognito M2M | Não |
| **Intra-domínios** | Agents de domínios diferentes internos | Cognito M2M + Scopes | Não |
| **Externo - Consumo** | Seu Agent chama Agent 3rd Party | OAuth do Parceiro | Não |
| **Externo - Exposição** | Agent 3rd Party chama seu Agent | Cognito M2M | **Axway obrigatório** |

---

### Fluxo A2A Unificado

Visão consolidada de todos os cenários A2A, independente de onde os Agents rodam (EKS ou AgentCore Runtime).

```mermaid
sequenceDiagram
    participant Partner as Partner Agent - 3rd Party
    participant Axway as Axway API Gateway
    participant ExtAuth as Partner OAuth Server
    participant Cognito as Cognito
    participant Agent1 as Agent CRM - Domain A
    participant Agent2 as Agent Finance - Domain A
    participant Agent3 as Agent HR - Domain B
    participant Agent4 as Partner Agent - 3rd Party
    
    rect rgb(227, 242, 253)
        Note over Agent1,Agent2: SCENARIO 1 - A2A INTERNAL same domain
        Agent1->>Agent2: GET /.well-known/agent-card.json
        Agent2-->>Agent1: Agent Card
        Agent1->>Cognito: Client Credentials M2M
        Note right of Agent1: scope: a2a-finance/read
        Cognito-->>Agent1: M2M Token
        Agent1->>Agent2: POST /message/send + Bearer M2M
        Agent2->>Cognito: Validate Token
        Cognito-->>Agent2: Valid Token
        Agent2-->>Agent1: Result
    end
    
    rect rgb(232, 245, 233)
        Note over Agent1,Agent3: SCENARIO 2 - A2A CROSS-DOMAINS Domain A to Domain B
        Agent1->>Agent3: GET /.well-known/agent-card.json
        Agent3-->>Agent1: Agent Card
        Agent1->>Cognito: Client Credentials M2M
        Note right of Agent1: scope: a2a-hr/read domain-b
        Cognito-->>Agent1: M2M Token with cross-domain scope
        Agent1->>Agent3: POST /message/send + Bearer M2M
        Agent3->>Cognito: Validate Token + Verify scope
        Cognito-->>Agent3: Valid Token - scope authorized
        Agent3-->>Agent1: Result
    end
    
    rect rgb(255, 243, 224)
        Note over Agent1,Agent4: SCENARIO 3 - CONSUMPTION of external Agent 3rd Party
        Agent1->>Agent4: GET /.well-known/agent-card.json
        Agent4-->>Agent1: Agent Card with external OAuth
        Agent1->>ExtAuth: Partner client credentials
        Note right of Agent1: Credentials registered with partner
        ExtAuth-->>Agent1: External Access Token
        Agent1->>Agent4: POST /message/send + Bearer external
        Agent4->>ExtAuth: Validate Token
        ExtAuth-->>Agent4: Valid Token
        Agent4-->>Agent1: Result
    end
    
    rect rgb(255, 235, 238)
        Note over Partner,Agent1: SCENARIO 4 - EXPOSURE via Axway required
        Partner->>Axway: GET /.well-known/agent-card.json
        Axway->>Agent1: Forward
        Agent1-->>Axway: Agent Card
        Axway-->>Partner: Agent Card
        Partner->>Cognito: Registered partner client credentials
        Note right of Partner: client_id created for partner
        Cognito-->>Partner: M2M Token
        Partner->>Axway: POST /message/send + Bearer M2M
        Axway->>Cognito: Validate Token
        Cognito-->>Axway: Valid Token
        Axway->>Axway: Audit + Rate Limit + WAF
        Axway->>Agent1: Forward
        Agent1-->>Axway: Result
        Axway-->>Partner: Result
    end
```

---

### Matriz de Cenários A2A

| Cenário | Agent Origem | Agent Destino | Autenticação | Gateway | Observação |
|---------|--------------|---------------|--------------|---------|------------|
| Interno mesmo domínio | Domínio A | Domínio A | Cognito M2M | Não | Comunicação interna |
| Intra-domínios | Domínio A | Domínio B | Cognito M2M + Scopes | Não | Scopes controlam acesso cross-domain |
| Consumo externo | Interno | 3rd Party | OAuth do Parceiro | Não | Credenciais registradas no parceiro |
| **Exposição externa** | 3rd Party | Interno | Cognito M2M | **Axway obrigatório** | Segurança na borda |

### Controle de Acesso A2A por Scopes

| Scope | Descrição | Exemplo |
|-------|-----------|---------|
| `a2a-{agent}/read` | Leitura no agent do mesmo domínio | `a2a-finance/read` |
| `a2a-{agent}/write` | Escrita no agent do mesmo domínio | `a2a-finance/write` |
| `a2a-{agent}/read:{dominio}` | Leitura cross-domain | `a2a-hr/read:dominio-b` |
| `a2a-{agent}/write:{dominio}` | Escrita cross-domain | `a2a-hr/write:dominio-b` |

> **Importante**: Para exposição de Agents a parceiros externos (3rd Party), o tráfego deve obrigatoriamente passar pelo Axway API Gateway para garantir:
> - Controle de acesso na borda (DMZ)
> - Rate limiting por parceiro
> - Auditoria centralizada
> - Proteção contra ataques (WAF, DDoS)

---

## Controle de Acesso por Domínio

### Matriz de Acesso

| Agent | MCPs Mesmo Domínio (Direto) | MCPs Domínio Diferente (Gateway) |
|-------|----------------------------|----------------------------------|
| crm-agent | mcp-crm-customers, mcp-crm-leads | finance-invoices, hr-employees, 3rdparty-mcp |
| finance-agent | mcp-finance-invoices, mcp-finance-payments | crm-customers, 3rdparty-mcp, jira-mcp |
| hr-agent | mcp-hr-employees, mcp-hr-payroll | finance-payments, 3rdparty-mcp |

---

## Diagrama Final: Cadeia Completa

```mermaid
sequenceDiagram
    participant User as User
    participant Keycloak as Keycloak
    participant Axway as Axway
    participant Cognito as Cognito
    participant Agent as CRM Agent
    participant MCP_CRM as MCP CRM
    participant Gateway as Gateway
    participant MCP_FIN as MCP Finance
    participant Identity as Identity
    participant ExtMCP as 3rd Party MCP
    participant Agent_FIN as Finance Agent
    
    rect rgb(232, 245, 233)
        Note over User,Agent: 1 - USER AUTHENTICATION OIDC
        User->>Keycloak: Login OIDC
        Keycloak-->>User: User Token
        User->>Axway: Request + Token
        Axway->>Agent: Forward + Context
    end
    
    rect rgb(227, 242, 253)
        Note over Agent,MCP_CRM: 2 - MCP SAME DOMAIN M2M
        Agent->>Cognito: Client Credentials
        Cognito-->>Agent: M2M Token
        Agent->>MCP_CRM: tools/call + M2M Token
        MCP_CRM-->>Agent: CRM Data
    end
    
    rect rgb(255, 243, 224)
        Note over Agent,MCP_FIN: 3 - MCP DIFFERENT DOMAIN Gateway
        Agent->>Gateway: invoke_gateway + M2M Token
        Gateway->>MCP_FIN: tools/call
        MCP_FIN-->>Gateway: Finance Data
        Gateway-->>Agent: Result
    end
    
    rect rgb(255, 235, 238)
        Note over Agent,ExtMCP: 4 - EXTERNAL MCP OAuth via Identity
        Agent->>Gateway: invoke_gateway 3rdparty-mcp
        Gateway->>Identity: get_credential
        Identity-->>Gateway: OAuth Token
        Gateway->>ExtMCP: API Call
        ExtMCP-->>Agent: Result
    end
    
    rect rgb(243, 229, 245)
        Note over Agent,Agent_FIN: 5 - A2A M2M DIRECT
        Agent->>Agent_FIN: GET /agent-card.json
        Agent->>Cognito: Client Credentials M2M
        Cognito-->>Agent: M2M Token
        Agent->>Agent_FIN: POST /message/send
        Agent_FIN-->>Agent: Risk Analysis
    end
    
    Agent-->>User: Consolidated Response
```

---

## Comparação: Acesso Direto vs Gateway Obrigatório

| Aspecto | Acesso Direto (mesmo domínio) | Gateway Obrigatório |
|---------|-------------------------------|---------------------|
| **MCP mesmo domínio** | Agent → MCP (direto) | Agent → Gateway → MCP |
| **MCP domínio diferente** | Agent → Gateway → MCP | Agent → Gateway → MCP |
| **Auditoria** | Parcial (só cross-domain) | Completa (todos os MCPs) |
| **Ponto de controle** | Distribuído | Centralizado |
| **Latência** | Menor (mesmo domínio) | Maior (hop adicional) |
| **Complexidade** | Maior (2 padrões) | Menor (1 padrão) |

### Quando usar Gateway Obrigatório

| Cenário | Recomendação |
|---------|--------------|
| Compliance rigoroso (SOX, PCI) | ✅ Gateway obrigatório |
| Auditoria centralizada | ✅ Gateway obrigatório |
| Múltiplos times/domínios | ✅ Gateway obrigatório |
| Baixa latência crítica | ❌ Acesso direto (mesmo domínio) |
| Simplicidade operacional | ✅ Gateway obrigatório |

---

## Policies sem AgentCore Gateway

> **Observação importante**: As policies do AgentCore estão integradas ao ecossistema de serviços, especialmente ao Gateway. Sem o Gateway, você perde a camada de governança centralizada para ferramentas/tools.

| Cenário | Solução sem Gateway |
|---------|---------------------|
| Controle de acesso a recursos AWS | IAM policies tradicionais |
| Filtragem de conteúdo | Bedrock Guardrails |
| Controle de tools/MCPs | Implementar manualmente no código do agente |
| Auditoria | CloudWatch + CloudTrail |
| Rate limiting por tool | Implementar no código ou usar API Gateway |

**Resumo**: O Gateway é o ponto onde as policies de ferramentas são aplicadas de forma centralizada. Sem ele, você precisa implementar esses controles de forma distribuída no seu código ou usar mecanismos nativos da AWS (IAM, Guardrails).

---

## Custos Estimados

### Cenário 1: Agent no EKS

| Componente | Custo/mês |
|------------|-----------|
| EKS Cluster | ~$73 |
| EC2 Nodes (2x t3.medium) | ~$60 |
| VPC Endpoints (AgentCore + Gateway) | ~$28 |
| VPC Endpoints auxiliares | ~$43 |
| AgentCore Runtime (MCPs) | Variável |
| **Total fixo** | **~$204/mês** |

### Cenário 2: Agent no Runtime

| Componente | Custo/mês |
|------------|-----------|
| VPC Endpoints (AgentCore + Gateway) | ~$28 |
| VPC Endpoints auxiliares | ~$43 |
| AgentCore Runtime (Agent + MCPs) | Variável |
| **Total fixo** | **~$71/mês** |

---

## Resumo

### Princípios de Comunicação

| Tipo | Caminho | Autenticação |
|------|---------|--------------|
| **MCP mesmo domínio** | Agent → MCP (direto) | OAuth M2M (Cognito) |
| **MCP domínio diferente** | Agent → Gateway → MCP | OAuth M2M (Cognito) |
| **MCP externo** | Agent → Gateway → Identity → Serviço | OAuth Externo |
| **A2A** | Agent → Agent (direto) | OAuth M2M (Cognito) |

### Garantias

- ✅ Comunicação 100% privada via VPC Endpoints
- ✅ Auditoria completa via CloudTrail
- ✅ Controle de acesso por domínio via OAuth scopes
- ✅ Rastreabilidade E2E via Correlation ID
- ✅ Isolamento por sessão (MicroVM no Runtime)

---

## Diagramas Alternativos: Todos os MCPs via Gateway

Neste modelo, **todos os MCPs são acessados exclusivamente via AgentCore Gateway**, mesmo os do mesmo domínio. Isso garante:
- Ponto único de controle e auditoria
- Políticas centralizadas para todos os MCPs
- Simplificação da arquitetura de rede

---

### Alternativa 1: Agent no Runtime - Gateway Obrigatório

Cenário serverless onde todos os MCPs passam pelo Gateway.

```mermaid
sequenceDiagram
    participant User as User
    participant Keycloak as Keycloak
    participant Axway as Axway
    participant Cognito as Cognito
    participant Agent as CRM Agent - Runtime
    participant Gateway as AgentCore Gateway
    participant MCP_CRM as MCP CRM - same domain
    participant MCP_FIN as MCP Finance - other domain
    participant Identity as AgentCore Identity
    participant ExtMCP as 3rd Party MCP
    participant Agent_FIN as Finance Agent
    
    rect rgb(232, 245, 233)
        Note over User,Agent: 1 - USER AUTHENTICATION OIDC
        User->>Keycloak: Login OIDC
        Keycloak-->>User: User Token
        User->>Axway: Request + Token
        Axway->>Keycloak: Validate Token
        Axway->>Agent: Forward + Context Headers
    end
    
    rect rgb(227, 242, 253)
        Note over Agent,Cognito: 2 - AGENT OBTAINS M2M TOKEN
        Agent->>Cognito: Client Credentials
        Note right of Agent: scope: agentcore-gateway/invoke
        Cognito-->>Agent: M2M Token
    end
    
    rect rgb(255, 249, 196)
        Note over Agent,MCP_CRM: 3 - MCP SAME DOMAIN via Gateway
        Agent->>Gateway: invoke_gateway crm-customers + M2M Token
        Note right of Agent: X-On-Behalf-Of: user123
        Gateway->>Cognito: Validate M2M Token
        Cognito-->>Gateway: Valid Token
        Gateway->>Gateway: Audit Log - CloudTrail
        Gateway->>MCP_CRM: tools/call get_customer
        MCP_CRM-->>Gateway: Customer data
        Gateway-->>Agent: Result
    end
    
    rect rgb(255, 243, 224)
        Note over Agent,MCP_FIN: 4 - MCP DIFFERENT DOMAIN via Gateway
        Agent->>Gateway: invoke_gateway finance-invoices + M2M Token
        Gateway->>Cognito: Validate M2M Token
        Cognito-->>Gateway: Valid Token
        Gateway->>Gateway: Audit Log - CloudTrail
        Gateway->>MCP_FIN: tools/call get_invoices
        MCP_FIN-->>Gateway: Invoice list
        Gateway-->>Agent: Result
    end
    
    rect rgb(255, 235, 238)
        Note over Agent,ExtMCP: 5 - EXTERNAL MCP via Gateway + Identity
        Agent->>Gateway: invoke_gateway 3rdparty-mcp + M2M Token
        Gateway->>Cognito: Validate M2M Token
        Cognito-->>Gateway: Valid Token
        Gateway->>Identity: get_credential 3rdparty-oauth
        Identity-->>Gateway: OAuth Token
        Gateway->>ExtMCP: API Call
        ExtMCP-->>Gateway: Result
        Gateway-->>Agent: Result
    end
    
    rect rgb(243, 229, 245)
        Note over Agent,Agent_FIN: 6 - A2A DIRECT - does not use Gateway
        Agent->>Agent_FIN: GET /.well-known/agent-card.json
        Agent_FIN-->>Agent: Agent Card
        Agent->>Cognito: Client Credentials M2M
        Note right of Agent: scope: a2a-finance/read
        Cognito-->>Agent: M2M Token
        Agent->>Agent_FIN: POST /message/send + M2M Token
        Agent_FIN->>Cognito: Validate M2M Token
        Cognito-->>Agent_FIN: Valid Token
        Agent_FIN-->>Agent: Risk Analysis
    end
    
    Agent-->>Axway: Complete Report
    Axway-->>User: Consolidated Response
```

---

### Alternativa 2: Agent no EKS - Gateway Obrigatório

Cenário híbrido com Agent no EKS onde todos os MCPs passam pelo Gateway via VPC Endpoint.

```mermaid
sequenceDiagram
    participant User as User
    participant Keycloak as Keycloak
    participant Axway as Axway
    participant Cognito as Cognito
    participant Agent as CRM Agent - EKS Pod
    participant VPCE as VPC Endpoint
    participant Gateway as AgentCore Gateway
    participant MCP_CRM as MCP CRM - same domain
    participant MCP_FIN as MCP Finance - other domain
    participant Identity as AgentCore Identity
    participant ExtMCP as 3rd Party MCP
    participant Agent_FIN as Finance Agent - Runtime
    
    rect rgb(232, 245, 233)
        Note over User,Agent: 1 - USER AUTHENTICATION OIDC
        User->>Keycloak: Login OIDC
        Keycloak-->>User: User Token
        User->>Axway: Request + Token
        Axway->>Keycloak: Validate Token
        Axway->>Agent: Forward + Context Headers
    end
    
    rect rgb(227, 242, 253)
        Note over Agent,Cognito: 2 - AGENT EKS OBTAINS M2M TOKEN
        Agent->>Cognito: Client Credentials
        Note right of Agent: scope: agentcore-gateway/invoke
        Cognito-->>Agent: M2M Token
    end
    
    rect rgb(255, 249, 196)
        Note over Agent,MCP_CRM: 3 - MCP SAME DOMAIN via Gateway
        Agent->>VPCE: invoke_gateway crm-customers + M2M Token
        Note right of Agent: VPC Endpoint: bedrock-agentcore.gateway
        VPCE->>Gateway: Forward PrivateLink
        Gateway->>Cognito: Validate M2M Token
        Cognito-->>Gateway: Valid Token
        Gateway->>Gateway: Audit Log - CloudTrail
        Gateway->>MCP_CRM: tools/call get_customer
        MCP_CRM-->>Gateway: Customer data
        Gateway-->>VPCE: Result
        VPCE-->>Agent: Result
    end
    
    rect rgb(255, 243, 224)
        Note over Agent,MCP_FIN: 4 - MCP DIFFERENT DOMAIN via Gateway
        Agent->>VPCE: invoke_gateway finance-invoices + M2M Token
        VPCE->>Gateway: Forward PrivateLink
        Gateway->>Cognito: Validate M2M Token
        Cognito-->>Gateway: Valid Token
        Gateway->>Gateway: Audit Log - CloudTrail
        Gateway->>MCP_FIN: tools/call get_invoices
        MCP_FIN-->>Gateway: Invoice list
        Gateway-->>VPCE: Result
        VPCE-->>Agent: Result
    end
    
    rect rgb(255, 235, 238)
        Note over Agent,ExtMCP: 5 - EXTERNAL MCP via Gateway + Identity
        Agent->>VPCE: invoke_gateway 3rdparty-mcp + M2M Token
        VPCE->>Gateway: Forward
        Gateway->>Cognito: Validate M2M Token
        Cognito-->>Gateway: Valid Token
        Gateway->>Identity: get_credential 3rdparty-oauth
        Identity-->>Gateway: OAuth Token
        Gateway->>ExtMCP: API Call
        ExtMCP-->>Gateway: Result
        Gateway-->>VPCE: Result
        VPCE-->>Agent: Result
    end
    
    rect rgb(243, 229, 245)
        Note over Agent,Agent_FIN: 6 - A2A DIRECT via VPC Endpoint
        Agent->>VPCE: GET /.well-known/agent-card.json
        Note right of Agent: VPC Endpoint: bedrock-agentcore Runtime
        VPCE->>Agent_FIN: Forward
        Agent_FIN-->>VPCE: Agent Card
        VPCE-->>Agent: Agent Card
        Agent->>Cognito: Client Credentials M2M
        Cognito-->>Agent: M2M Token
        Agent->>VPCE: POST /message/send + M2M Token
        VPCE->>Agent_FIN: Forward PrivateLink
        Agent_FIN->>Cognito: Validate M2M Token
        Cognito-->>Agent_FIN: Valid Token
        Agent_FIN-->>VPCE: Risk Analysis
        VPCE-->>Agent: Result
    end
    
    Agent-->>Axway: Complete Report
    Axway-->>User: Consolidated Response
```

---

### Legenda dos Diagramas

| Componente | Função | Autenticação |
|------------|--------|--------------|
| **Keycloak** | IdP para usuários | OIDC Authorization Code |
| **Axway** | API Gateway corporativo | Valida User Token |
| **Cognito** | IdP para M2M | Client Credentials |
| **AgentCore Gateway** | Gateway MCP - ponto único | OAuth M2M Cognito |
| **AgentCore Identity** | Credential Provider | OAuth para serviços externos |
| **VPC Endpoint** | Conectividade privada EKS para AgentCore | PrivateLink |

| Tipo de Comunicação | Via Gateway | Autenticação |
|---------------------|-------------|--------------|
| MCP mesmo domínio | SIM | Cognito M2M |
| MCP domínio diferente | SIM | Cognito M2M |
| MCP externo | SIM | OAuth via Identity |
| A2A Agent para Agent | NÃO - direto | Cognito M2M |
