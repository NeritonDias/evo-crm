# Analise de Viabilidade: OAuth Codex (OpenAI) no Evo CRM

**Data:** 2026-04-16
**Repositorio:** neritondias/evo-crm

---

## 1. Arquitetura Atual do Evo CRM

O Evo CRM e uma plataforma composta por 6 microservicos:

| Servico | Stack | Porta | Funcao |
|---|---|---|---|
| **evo-auth** | Ruby/Rails + Doorkeeper | 3001 | Autenticacao, OAuth2, JWT, MFA, RBAC |
| **evo-crm** | Ruby/Rails | 3000 | CRM principal (Chatwoot-based) |
| **evo-core** | Go/Gin (ou Python) | 5555 | Gerenciamento de agentes IA e API keys |
| **evo-processor** | Python/FastAPI | 8000 | Execucao de agentes IA (ADK/CrewAI + LiteLLM) |
| **evo-bot-runtime** | Go/Gin | 8080 | Runtime de bots |
| **evo-frontend** | React/Next.js + Vite | 5173 | Interface do usuario |

---

## 2. Como Funciona Hoje a Autenticacao de APIs de IA

### Fluxo Completo (Atual)

```
Usuario (Frontend)
    |
    | 1. Abre dialog "API Keys" na pagina de Agentes
    | 2. Preenche: Nome, Provider (OpenAI/Anthropic/etc), API Key (sk-...)
    |
    v
evo-core (Backend API)
    |
    | 3. POST /api/v1/agents/apikeys
    | 4. encrypt_api_key(key_value) -> Fernet AES-128-CBC + HMAC
    | 5. Salva encrypted_key no PostgreSQL (tabela api_keys)
    |
    v
PostgreSQL (api_keys table)
    |  id | client_id | name | provider | encrypted_key | is_active |
    |
    v
Execucao do Agente (runtime)
    |
    | 6. AgentBuilder busca agent.api_key_id
    | 7. decrypt_api_key(encrypted_key) -> chave original
    | 8. LiteLlm(model="openai/gpt-4o", api_key="sk-...")
    |
    v
LiteLLM -> OpenAI API (Authorization: Bearer sk-...)
```

### Modelo de Dados

```sql
-- Tabela api_keys
CREATE TABLE api_keys (
    id UUID PRIMARY KEY,
    client_id UUID REFERENCES clients(id) ON DELETE CASCADE,
    name VARCHAR NOT NULL,
    provider VARCHAR NOT NULL,       -- "openai", "anthropic", "gemini", etc.
    encrypted_key VARCHAR NOT NULL,  -- Fernet-encrypted
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMPTZ,
    updated_at TIMESTAMPTZ
);

-- Tabela agents referencia api_keys
ALTER TABLE agents ADD COLUMN api_key_id UUID REFERENCES api_keys(id) ON DELETE SET NULL;
```

### Criptografia

- **Algoritmo:** Fernet (AES-128-CBC + HMAC-SHA256)
- **Chave mestra:** variavel de ambiente `ENCRYPTION_KEY`
- **Padrao:** Encrypt-on-write, decrypt-on-read (apenas no runtime do agente)

### Providers Suportados (via LiteLLM)

OpenAI, Anthropic, Google Gemini, Groq, Cohere, Mistral, Azure OpenAI, AWS Bedrock, e 100+ outros.

---

## 3. O Que e o "OAuth Codex" da OpenAI

### Esclarecimento Critico

**OAuth Codex NAO e um mecanismo generico de OAuth para APIs da OpenAI.** E especificamente o fluxo de autenticacao do produto **OpenAI Codex** (agente de codigo lancado em abril 2025), que permite login via conta ChatGPT.

### Detalhes Tecnicos

| Aspecto | Detalhe |
|---|---|
| **Tipo** | OAuth 2.0 Authorization Code + PKCE |
| **Client ID** | `app_EMoamEEZ73f0CkXaXp7hrann` (fixo, publico) |
| **Auth Endpoint** | `https://auth.openai.com/oauth/authorize` |
| **Token Endpoint** | `https://auth.openai.com/oauth/token` |
| **Redirect URI** | `http://localhost:1455/auth/callback` |
| **Scopes** | `openid profile email offline_access` |
| **Tokens** | Access token (curta duracao) + Refresh token (rotativo) |
| **Cobranca** | Assinatura ChatGPT (Plus $20/mo, Pro $200/mo) |
| **Fluxo Alt.** | Device Code Flow para ambientes headless |

### Diferenca Fundamental

| API Key (atual) | OAuth Codex |
|---|---|
| `sk-...` token estatico | Access token dinamico + refresh |
| Pay-per-use (API billing) | Assinatura ChatGPT |
| Funciona com QUALQUER modelo OpenAI | Projetado APENAS para o produto Codex |
| Auto-contido, sem expiracao | Expira, precisa de refresh automatico |
| Suportado por todos os SDKs | Scopes insuficientes para API geral |

### Problema Conhecido (abril 2026)

Tokens OAuth Codex **NAO incluem os scopes necessarios** para chamadas gerais de API:
- Falta `model.request` (necessario para chamadas de modelo)
- Falta `api.responses.write` (necessario para POST /v1/responses)

Isso causa erros **401 Unauthorized** quando ferramentas third-party tentam usar tokens Codex OAuth para chamar a API da OpenAI diretamente. O CLI oficial do Codex contorna isso internamente, mas integradores externos enfrentam essa limitacao.

---

## 4. Analise de Viabilidade

### 4.1 E Possivel Implementar?

**Tecnicamente sim, mas com ressalvas significativas.**

A implementacao e possivel porque:
- O evo-auth-service ja possui infraestrutura OAuth2 completa (Doorkeeper)
- O endpoint `/api/v1/dynamic_oauth` sugere suporte a provedores OAuth dinamicos
- O backend Python suporta refresh de tokens (pode ser adaptado)

Porem, **NAO e recomendado** porque:

1. **OAuth Codex nao se destina a uso generico de API** - foi projetado para o produto Codex, nao para substituir API keys em chamadas gerais de modelo.

2. **Scopes insuficientes** - Os tokens OAuth nao possuem `model.request` e `api.responses.write`, essenciais para chamadas de API que o evo-crm faz (chat completions, etc.).

3. **Sem registro de client para terceiros** - OpenAI nao oferece registro de client ID proprio. Usar o client ID publico do Codex (`app_EMoamEEZ73f0CkXaXp7hrann`) e uma pratica nao oficial e pode ser bloqueada a qualquer momento.

4. **Billing incompativel** - OAuth Codex cobra via assinatura ChatGPT, nao via API usage. Isso muda completamente o modelo de cobranca para os usuarios do evo-crm.

5. **LiteLLM nao suporta OAuth nativamente** - LiteLLM espera `api_key` como string estatica. Seria necessario um wrapper para gerenciar token refresh antes de cada chamada.

### 4.2 Avaliacao de Complexidade

**Complexidade: ALTA (se implementado da forma correta)**

#### Mudancas Necessarias por Servico

**Frontend (evo-ai-frontend) - Complexidade MEDIA:**
- Novo componente de OAuth flow (botao "Conectar com OpenAI")
- Popup/redirect para `auth.openai.com/oauth/authorize`
- Callback handler para receber authorization code
- UI para status de conexao OAuth (conectado/desconectado/expirado)
- Estimativa: ~15-20 arquivos novos/modificados

**Backend - evo-core / evo-ai (API/DB) - Complexidade ALTA:**
- Nova tabela `oauth_tokens` para armazenar access/refresh tokens
- Migracoes de banco de dados
- Servico de OAuth token lifecycle (create, refresh, revoke)
- Background job para refresh automatico de tokens proximo a expiracao
- Endpoint de callback OAuth
- Adaptar `AgentBuilder._get_api_key()` para retornar OAuth token quando aplicavel
- Estimativa: ~20-25 arquivos novos/modificados

**Backend - evo-processor (Runtime) - Complexidade MEDIA:**
- Wrapper em volta do LiteLLM para injetar OAuth tokens
- Logica de retry com token refresh quando receber 401
- Estimativa: ~5-8 arquivos modificados

**Auth Service (evo-auth) - Complexidade BAIXA-MEDIA:**
- Registrar OpenAI como OAuth provider dinamico
- Configurar redirect URIs
- Estimativa: ~3-5 arquivos modificados

#### Estimativa de Esforco Total

| Componente | Complexidade | Arquivos |
|---|---|---|
| Frontend OAuth flow | Media | ~15-20 |
| Modelo de dados (migracao) | Baixa | ~3-5 |
| Servico OAuth token lifecycle | Alta | ~10-15 |
| Background job refresh | Media | ~5-8 |
| Adaptacao AgentBuilder | Media | ~5-8 |
| Wrapper LiteLLM | Media | ~5-8 |
| Auth service config | Baixa | ~3-5 |
| Testes | Alta | ~15-20 |
| **TOTAL** | **ALTA** | **~60-90 arquivos** |

---

## 5. Recomendacao

### NAO implementar OAuth Codex da OpenAI neste momento.

**Razoes:**

1. **Produto errado** - OAuth Codex e para o produto Codex (agente de codigo), nao para uso generico da API OpenAI. Usar tokens Codex para chamadas de chat/completion e um hack, nao uma integracao suportada.

2. **Risco de quebra** - OpenAI pode bloquear o uso do client ID publico por terceiros a qualquer momento.

3. **Scopes incompletos** - Tokens atuais nao autorizam chamadas de API necessarias.

4. **Custo-beneficio** - A complexidade de implementacao e alta (~60-90 arquivos, 4-6 semanas) para um beneficio questionavel, dado que a autenticacao por API key funciona perfeitamente para todos os providers.

### Alternativas Recomendadas

**Se o objetivo e melhorar a seguranca das API keys:**
1. **Key rotation automatico** - Implementar rotacao periodica de API keys
2. **Key validation** - Validar a API key ao cadastrar (fazer uma chamada de teste)
3. **Usage tracking** - Monitorar uso por API key
4. **Key masking** - Mostrar apenas ultimos 4 caracteres na UI (ja parcialmente implementado)

**Se o objetivo e simplificar a UX de conexao com OpenAI:**
1. **Aguardar** a OpenAI lancar um OAuth generico para plataforma (ainda nao existe)
2. **Implementar validacao inline** - ao colar a API key, validar e mostrar os modelos disponiveis automaticamente

**Se no futuro a OpenAI lancar OAuth para API Platform:**
A arquitetura do evo-crm esta bem preparada para isso:
- O auth service ja tem Doorkeeper + `dynamic_oauth`
- O modelo de dados pode ser estendido facilmente
- O LiteLLM pode receber tokens OAuth como `api_key`
- A adaptacao seria significativamente mais simples

---

## 6. Arquivos-Chave Analisados

### evo-ai (Core + Processor)
- `src/models/models.py` - Modelos ApiKey e Agent
- `src/utils/crypto.py` - Criptografia Fernet
- `src/services/apikey_service.py` - CRUD de API keys
- `src/services/adk/agent_builder.py` - Builder com LiteLLM
- `src/services/crewai/agent_builder.py` - Builder CrewAI
- `src/api/agent_routes.py` - Endpoints REST
- `src/schemas/schemas.py` - Schemas Pydantic

### Frontend
- `frontend/app/agents/page.tsx` - Pagina de agentes
- `frontend/app/agents/dialogs/ApiKeysDialog.tsx` - Dialog de API keys
- `frontend/app/agents/config/LLMAgentConfig.tsx` - Config de agente LLM
- `frontend/services/agentService.ts` - Cliente API
- `frontend/types/aiModels.ts` - Providers e modelos

### Infraestrutura
- `docker-compose.yml` - Orquestracao de servicos
- `.env.example` - Variaveis de ambiente (ENCRYPTION_KEY, DOORKEEPER_JWT_*)
- `nginx/nginx.conf` - Gateway com rotas /oauth/* e /api/v1/dynamic_oauth/*
