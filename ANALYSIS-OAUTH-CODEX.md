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

## 5. Analise do evo-nexus: Como Implementa OAuth Codex

### 5.1 Arquitetura do evo-nexus (Fundamental Diferente)

O evo-nexus e um **sistema de agentes baseado em CLI** (Claude Code), NAO um CRM multi-tenant.
Ele **delega a execucao para CLIs** (`claude` ou `openclaude`) via subprocess, nao faz chamadas
diretas de API como o evo-crm.

| Aspecto | evo-nexus | evo-crm |
|---|---|---|
| **Modelo** | CLI wrapper (spawna `claude`/`openclaude`) | Chamadas diretas via LiteLLM SDK |
| **Multi-tenant** | Single user/workspace | Multi-tenant (client_id por usuario) |
| **Storage tokens** | Arquivo `~/.codex/auth.json` | PostgreSQL (tabela `api_keys`) |
| **Token refresh** | Tratado pelo CLI (openclaude) | Teria que ser implementado manualmente |
| **Stack** | Flask dashboard + React frontend | Rails + Go + FastAPI + React |
| **Provider config** | JSON file (`providers.json`) | PostgreSQL + Fernet encryption |

### 5.2 Implementacao OAuth Codex no evo-nexus

O evo-nexus implementa o fluxo completo em `dashboard/backend/routes/providers.py`.

#### Constantes

```python
OPENAI_CLIENT_ID = "app_EMoamEEZ73f0CkXaXp7hrann"  # Client ID publico do Codex
OPENAI_AUTH_URL  = "https://auth.openai.com/oauth/authorize"
OPENAI_TOKEN_URL = "https://auth.openai.com/oauth/token"
CODEX_AUTH_FILE  = Path.home() / ".codex" / "auth.json"
```

#### Fluxo 1: Browser OAuth (PKCE)

```
Frontend                    Backend (/api/providers)         OpenAI Auth
   |                              |                              |
   |-- POST /openai/auth-start ->|                              |
   |                              |-- gera code_verifier (PKCE) |
   |                              |-- gera code_challenge (S256)|
   |                              |-- gera state token          |
   |<- {authorize_url} ----------|                              |
   |                              |                              |
   |-- abre URL no browser ------>|----------------------------->|
   |                              |     usuario faz login ChatGPT|
   |                              |<---- callback com ?code=... -|
   |   (usuario copia URL)        |                              |
   |                              |                              |
   |-- POST /openai/auth-complete |                              |
   |   {callback_url: "..."}  -->|                              |
   |                              |-- extrai code da URL        |
   |                              |-- POST /oauth/token -------->|
   |                              |   {grant_type, code,         |
   |                              |    code_verifier, client_id} |
   |                              |<--- {access_token,           |
   |                              |      refresh_token,          |
   |                              |      id_token} --------------|
   |                              |                              |
   |                              |-- salva em ~/.codex/auth.json|
   |                              |-- ativa provider "openai"    |
   |<- {status: "ok"} -----------|                              |
```

#### Fluxo 2: Device Code (para ambientes sem browser)

```
Frontend                    Backend                    OpenAI Auth
   |                              |                         |
   |-- POST /openai/device-start->|                         |
   |                              |-- POST /deviceauth/usercode ->|
   |                              |<- {device_auth_id,       |
   |                              |    user_code} -----------|
   |<- {user_code,                |                         |
   |    verification_url} --------|                         |
   |                              |                         |
   | usuario abre URL e digita codigo                       |
   |                              |                         |
   |-- POST /openai/device-poll ->|                         |
   |                              |-- POST /deviceauth/token ->|
   |                              |<- authorization_code ----|
   |                              |-- POST /oauth/token ---->|
   |                              |<- tokens ---------------|
   |                              |-- salva auth.json       |
   |<- {status: "authorized"} ---|                         |
```

#### Scopes Usados

```python
"scope": "openid profile email offline_access api.connectors.read api.connectors.invoke"
```

**Nota:** O evo-nexus inclui `api.connectors.read` e `api.connectors.invoke` que NAO estavam
na documentacao oficial do Codex CLI. Isso pode ajudar com permissoes adicionais.

#### Storage dos Tokens

```python
def _save_codex_auth(tokens: dict):
    auth_data = {
        "auth_mode": "Chatgpt",
        "tokens": {
            "access_token": tokens["access_token"],
            "refresh_token": tokens.get("refresh_token", ""),
            "id_token": tokens.get("id_token", access_token),
            "account_id": "<extraido do JWT payload>",
        },
        "last_refresh": "2026-04-16T00:00:00Z",
    }
    # Salva em ~/.codex/auth.json
```

#### Endpoints Implementados

| Endpoint | Metodo | Funcao |
|---|---|---|
| `/api/providers/openai/auth-start` | POST | Inicia OAuth PKCE, retorna authorize URL |
| `/api/providers/openai/auth-complete` | POST | Recebe callback URL, troca code por tokens |
| `/api/providers/openai/device-start` | POST | Inicia Device Code flow |
| `/api/providers/openai/device-poll` | POST | Polling ate usuario autorizar |
| `/api/providers/openai/status` | GET | Verifica se auth.json existe e tem tokens |
| `/api/providers/openai/logout` | POST | Remove auth.json e reseta provider |

### 5.3 POR QUE Funciona no evo-nexus mas NAO Pode Ser Copiado Diretamente

O evo-nexus **NAO usa os tokens OAuth para fazer chamadas diretas de API**.
Ele salva os tokens em `~/.codex/auth.json` e o CLI `openclaude` lida com tudo:
- Token refresh automatico
- Protocolo de comunicacao com OpenAI
- Gerenciamento de sessao

**No evo-crm, a situacao e completamente diferente:**
- O evo-crm faz chamadas diretas via `LiteLLM(model=..., api_key=...)`
- LiteLLM espera uma API key estatica, nao um OAuth access token
- Nao ha CLI intermediario para gerenciar o lifecycle dos tokens
- E multi-tenant: cada usuario/client precisa de seus proprios tokens

### 5.4 O Que PODE Ser Portado do evo-nexus

| Componente | Portabilidade | Adaptacao Necessaria |
|---|---|---|
| OAuth PKCE flow | ALTA | Adaptar para Rails/Python ao inves de Flask |
| Device Code flow | ALTA | Mesma logica, diferente framework |
| Frontend UI (Providers.tsx) | MEDIA | Adaptar para React existente do evo-crm |
| Token storage em arquivo | BAIXA | Precisa ser PostgreSQL com criptografia |
| Token refresh automatico | NAO PORTAVEL | No nexus o CLI faz; no crm precisa implementar |
| Uso dos tokens para API calls | NAO PORTAVEL | Nexus delega ao CLI; crm faz chamadas diretas |

---

## 6. Viabilidade de Implementacao no evo-crm (Baseado no evo-nexus)

### 6.1 Abordagem Proposta (Se Decidir Implementar)

Seria necessario adaptar o fluxo do evo-nexus para a arquitetura do evo-crm:

```
Frontend (React)
    |
    | 1. Botao "Conectar com OpenAI (Codex OAuth)"
    | 2. Popup OAuth -> auth.openai.com/oauth/authorize (PKCE)
    | 3. Callback com code
    |
    v
evo-core (Backend)
    |
    | 4. Troca code por tokens (POST auth.openai.com/oauth/token)
    | 5. Criptografa tokens com Fernet
    | 6. Salva na NOVA tabela oauth_tokens (PostgreSQL)
    |    {client_id, provider, access_token_enc, refresh_token_enc, expires_at}
    |
    v
Background Job (Sidekiq/Celery)
    |
    | 7. A cada 4 minutos, verifica tokens proximos de expirar
    | 8. Faz refresh automatico (POST auth.openai.com/oauth/token)
    | 9. Atualiza tokens no banco
    |
    v
AgentBuilder (Runtime)
    |
    | 10. Verifica se agente usa OAuth ou API key
    | 11. Se OAuth: busca access_token descriptografado
    | 12. Passa como api_key ao LiteLLM
    |     LiteLlm(model="openai/gpt-4o", api_key=<oauth_access_token>)
    |
    v
OpenAI API (Authorization: Bearer <oauth_access_token>)
    |
    | *** RISCO: Token pode nao ter scope model.request ***
    | *** Resultado: 401 Unauthorized ou funciona parcialmente ***
```

### 6.2 Estimativa de Complexidade Revisada

Com base no codigo real do evo-nexus como referencia:

| Componente | Complexidade | Arquivos | Referencia no evo-nexus |
|---|---|---|---|
| OAuth PKCE endpoints | BAIXA | ~3 | `providers.py` (linhas 200-280) |
| Device Code endpoints | BAIXA | ~2 | `providers.py` (linhas 285-350) |
| Token storage model + migration | BAIXA | ~3 | Novo (nexus usa arquivo) |
| Token encryption service | BAIXA | ~2 | Reusar `crypto.py` existente |
| Token refresh background job | MEDIA | ~5 | Novo (nexus delega ao CLI) |
| Adaptar AgentBuilder | MEDIA | ~4 | `agent_builder.py` |
| Frontend OAuth UI | MEDIA | ~8 | `Providers.tsx` |
| Frontend Device Code UI | MEDIA | ~5 | `Providers.tsx` |
| Adaptar ApiKeysDialog | BAIXA | ~2 | Existente no evo-crm |
| Testes | MEDIA | ~10 | Novos |
| **TOTAL** | **MEDIA** | **~44 arquivos** | |

**Reducao de ~60-90 para ~44 arquivos** por ter o evo-nexus como referencia de implementacao.

### 6.3 Riscos Persistentes (Mesmo Com Referencia do evo-nexus)

1. **Scope `model.request` ainda e um problema** - O evo-nexus contorna isso usando o CLI que tem tratamento interno. No evo-crm, chamadas diretas podem falhar com 401.

2. **Token refresh rotativo** - Refresh tokens da OpenAI sao single-use. Se o refresh falhar (rede, timing), o usuario perde acesso e precisa re-autenticar.

3. **Multi-tenancy** - O evo-nexus e single-user. No evo-crm, cada client_id teria seus tokens OAuth, multiplicando a complexidade de refresh.

4. **Client ID publico** - Tanto o evo-nexus quanto o evo-crm usariam `app_EMoamEEZ73f0CkXaXp7hrann`. OpenAI pode revogar ou restringir a qualquer momento.

---

## 7. Recomendacao Final

### Cenario A: Se voce quer o MESMO comportamento do evo-nexus

**POSSIVEL, complexidade MEDIA (~44 arquivos, 2-3 semanas)**

O fluxo OAuth do evo-nexus pode ser portado para o evo-crm. Porem, o evo-nexus
usa os tokens apenas para alimentar o CLI `openclaude`, que gerencia tudo internamente.
No evo-crm, os tokens seriam passados diretamente como `api_key` ao LiteLLM, o que
**pode funcionar para alguns endpoints** mas nao e garantido pela OpenAI.

**Quando faz sentido:** Se voce quer oferecer aos usuarios uma alternativa de autenticacao
"zero config" onde eles simplesmente logam com a conta ChatGPT ao inves de copiar/colar
uma API key. O billing seria via assinatura ChatGPT (nao pay-per-use).

### Cenario B: Se voce quer autenticacao robusta para API OpenAI

**NAO implementar OAuth Codex. Manter API keys.**

**Razoes:**
1. **Scopes insuficientes** - Tokens Codex podem nao ter `model.request`, causando 401 em chamadas diretas.
2. **Client ID nao oficial** - Risco de bloqueio pela OpenAI.
3. **Billing incompativel** - Muda de pay-per-use para assinatura ChatGPT.
4. **API keys funcionam perfeitamente** para todos os 100+ providers via LiteLLM.

### Cenario C: Implementacao Hibrida (Recomendado se quiser seguir em frente)

Oferecer **ambas as opcoes** ao usuario na UI:
- **API Key** (existente, funciona com todos os providers)
- **OAuth Codex** (novo, apenas para OpenAI, experimental)

Isso minimiza risco: se OAuth falhar, o usuario sempre pode cair de volta para API key.

### Alternativas que Agregam Mais Valor

1. **Key validation automatica** - Ao colar a API key, validar com uma chamada de teste e listar modelos disponiveis
2. **Usage tracking** - Dashboard de uso por API key/provider
3. **Key rotation** - Alertas quando uma key esta antiga ou comprometida
4. **Monitoramento de custos** - Integrar com API de billing dos providers

---

## 8. Arquivos-Chave Analisados

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

### Infraestrutura (evo-crm)
- `docker-compose.yml` - Orquestracao de servicos
- `.env.example` - Variaveis de ambiente (ENCRYPTION_KEY, DOORKEEPER_JWT_*)
- `nginx/nginx.conf` - Gateway com rotas /oauth/* e /api/v1/dynamic_oauth/*

### evo-nexus (Referencia OAuth)
- `dashboard/backend/routes/providers.py` - **Implementacao completa do OAuth Codex** (PKCE + Device Code)
- `dashboard/frontend/src/pages/Providers.tsx` - UI de configuracao de providers com OAuth
- `config/providers.example.json` - Configuracao de providers (OpenAI, OpenRouter, Gemini, etc.)
- `social-auth/app.py` - OAuth para redes sociais (YouTube, Instagram, etc.)
- `social-auth/env_manager.py` - Gerenciamento multi-conta de tokens OAuth
- `.env.example` - Variaveis de ambiente para providers de IA
