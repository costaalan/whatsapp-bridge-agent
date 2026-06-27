# 🏗️ Arquitetura — WhatsApp Bridge Agent

Documentação técnica da arquitetura do bridge WhatsApp ↔ IA.

---

## 1. Visão Arquitetural

O WhatsApp Bridge Agent adota uma **arquitetura de micro-serviços desacoplados por porta**, com um barramento central (Bridge v7) que orquestra o fluxo entre o gateway WhatsApp (Evolution API) e o agente de IA (Hermes Agent).

```
                          ┌───────────────────────────────────────┐
                          │           Bridge v7 (:8645)           │
                          │                                       │
  WhatsApp ──▶ Evolution  │  ┌──────────┐    ┌───────────────┐   │
              API (:8081) │  │ Message  │───▶│ Intent Router │   │
                          │  │ Receiver │    └───┬───────┬───┘   │
                          │  └──────────┘        │       │       │
                          │                      ▼       ▼       │
                          │               ┌──────────┐ ┌──────┐  │
                          │               │ Hermes   │ │DDGS  │  │
                          │               │ (:8642)  │ │Fast  │  │
                          │               └────┬─────┘ └──┬───┘  │
                          │                    │          │       │
                          │                    ▼          ▼       │
                          │               ┌──────────────────┐   │
                          │               │ Response Builder │   │
                          │               └────────┬─────────┘   │
                          └────────────────────────│─────────────┘
                                                   ▼
                                          Evolution API (:8081)
                                                   │
                                                   ▼
                                              WhatsApp
```

### Serviços e Portas

| Serviço         | Porta   | Responsabilidade                          |
|-----------------|---------|-------------------------------------------|
| Evolution API   | `:8081` | Gateway WhatsApp (envio/recebimento)      |
| Bridge v7       | `:8645` | Orquestração, roteamento, keep-alive      |
| Hermes API      | `:8642` | Processamento de IA / NLP                 |
| *(externo)*     | —       | Groq Whisper (transcrição)                |
| *(externo)*     | —       | DuckDuckGo Instant Answer                 |

---

## 2. Componentes

### 2.1 Evolution API Gateway

**Responsabilidade:** Abstrair o protocolo WhatsApp (Baileys) e expor uma API REST.

- Conexão WebSocket mantida com servidores WhatsApp
- QR Code scanning para autenticação inicial
- Webhooks configuráveis para eventos de mensagem
- Endpoints utilizados pelo Bridge:
  - `POST /message/sendText/{instance}` — enviar texto
  - `POST /message/sendMedia/{instance}` — enviar mídia
  - Webhook `messages.upsert` — receber novas mensagens

**Configuração de Webhook:**
```json
{
  "url": "http://<bridge-host>:8645/webhook/evolution",
  "events": ["messages.upsert", "messages.update"],
  "enabled": true
}
```

### 2.2 Bridge v7 (`bridge/`)

O Bridge é o coração do sistema. Componentes internos:

```
bridge/
├── server.py          # FastAPI/Flask app, expõe :8645
├── receiver.py        # Endpoint de webhook da Evolution
├── router.py          # Intent Router — decide o caminho
├── handlers/
│   ├── text.py        # Mensagens de texto → Hermes
│   ├── audio.py       # Mensagens de áudio → Groq → Hermes
│   ├── media.py       # Imagens, docs (encaminhamento)
│   └── fast_path.py   # DuckDuckGo instant answer
├── sender.py          # Envio de respostas via Evolution API
├── keepalive.py       # Healthcheck dos 3 serviços
└── session.py         # Gerenciamento de sessão/conversa
```

#### 2.2.1 Intent Router (`router.py`)

Lógica de decisão que classifica a mensagem recebida:

```
Mensagem recebida
    │
    ├── É áudio? ─────────▶ AudioHandler → Groq Whisper → texto
    │
    ├── É texto curto + pergunta factual?
    │       │
    │       ├── Sim ──────▶ FastPathHandler → DDGS → resposta direta
    │       │
    │       └── Não ──────▶ TextHandler → Hermes API → resposta IA
    │
    └── É mídia (imagem/doc)? ──▶ MediaHandler (forward p/ Hermes)
```

#### 2.2.2 Fast-Path DuckDuckGo (`fast_path.py`)

Critérios para ativar o fast-path:
- Mensagem ≤ 100 caracteres
- Contém padrão de pergunta (`qual`, `quem`, `quanto`, `onde`, `when`, `what`, etc.)
- Não contém contexto de conversa anterior (primeira mensagem ou tópico novo)

Fluxo:
```python
def fast_path_query(text: str) -> str | None:
    results = DDGS().answers(text)
    if results and results[0].get("text"):
        return results[0]["text"]
    return None  # fallback to Hermes
```

#### 2.2.3 Audio Handler (`audio.py`)

```
WhatsApp Áudio → Evolution (download URL) → Bridge (download) → Groq Whisper → Transcrição → Hermes → Resposta
```

- Formatos suportados: OGG (padrão WhatsApp), MP3, WAV
- Groq Whisper: modelo `whisper-large-v3`, idioma auto-detect
- Timeout: 30s para áudios de até 5 minutos

#### 2.2.4 Keep-Alive (`keepalive.py`)

Monitora a saúde dos três serviços com heartbeat configurável:

```python
HEALTH_CHECKS = {
    "evolution": "http://...:8081/health",
    "bridge":    "http://...:8645/health",
    "hermes":    "http://...:8642/health",
}
```

- Intervalo padrão: 30 segundos
- Em falha: 3 retries com backoff 5s/10s/20s
- Log e métricas Prometheus expostas em `:8645/metrics`

---

## 3. Fluxo Completo de Mensagem (Diagrama de Sequência)

```
WhatsApp    Evolution(:8081)    Bridge(:8645)     Groq        Hermes(:8642)    DDGS
  │               │                  │               │              │            │
  │──mensagem────▶│                  │               │              │            │
  │               │──webhook────────▶│               │              │            │
  │               │                  │──[classifica]─│              │            │
  │               │                  │               │              │            │
  │               │     ╔═══════════╪═══════════════╪══════════════╪════════╗   │
  │               │     ║ FAST-PATH (texto factual curto)                 ║   │
  │               │     ║           │               │              │      ║   │
  │               │     ║           │──DDGS query─────────────────────────▶║   │
  │               │     ║           │◀─instant answer──────────────────────║   │
  │               │     ╚═══════════╪═══════════════╪══════════════╪════════╝   │
  │               │                  │               │              │            │
  │               │     ╔═══════════╪═══════════════╪══════════════╪════════╗   │
  │               │     ║ TEXTO (conversa contextual)                      ║   │
  │               │     ║           │               │──prompt─────▶│      ║   │
  │               │     ║           │               │◀─response────│      ║   │
  │               │     ╚═══════════╪═══════════════╪══════════════╪════════╝   │
  │               │                  │               │              │            │
  │               │     ╔═══════════╪═══════════════╪══════════════╪════════╗   │
  │               │     ║ ÁUDIO                                              ║   │
  │               │     ║           │──download audio──▶│              │      ║   │
  │               │     ║           │◀──transcrição─────│              │      ║   │
  │               │     ║           │────────────prompt──────────────▶│      ║   │
  │               │     ║           │◀───────────response─────────────│      ║   │
  │               │     ╚═══════════╪═══════════════╪══════════════╪════════╝   │
  │               │                  │               │              │            │
  │               │◀──sendText───────│               │              │            │
  │◀──resposta────│                  │               │              │            │
```

---

## 4. Estrutura de Dados

### 4.1 Webhook Payload (Evolution → Bridge)

```json
{
  "event": "messages.upsert",
  "instance": "instance_name",
  "data": {
    "key": {
      "remoteJid": "5511999999999@s.whatsapp.net",
      "fromMe": false,
      "id": "MESSAGE_ID"
    },
    "message": {
      "conversation": "Texto da mensagem",
      "audioMessage": { "url": "https://...", "mimetype": "audio/ogg" }
    },
    "messageTimestamp": "1719876543",
    "pushName": "Nome do Contato"
  }
}
```

### 4.2 Requisição Hermes API

```json
{
  "message": "texto para processar",
  "session_id": "5511999999999@s.whatsapp.net",
  "context": { "previous_messages": [...] },
  "metadata": { "source": "whatsapp", "media_type": "text" }
}
```

### 4.3 Resposta Hermes API

```json
{
  "reply": "Resposta gerada pelo agente",
  "confidence": 0.92,
  "sources": ["url1", "url2"],
  "tokens_used": 450
}
```

---

## 5. Resiliência & Tratamento de Falhas

| Cenário                        | Estratégia                                    |
|--------------------------------|-----------------------------------------------|
| Evolution API offline          | Retry 3x, notificar admin, fila de mensagens  |
| Hermes API timeout (>30s)      | Fallback: "Estou processando, aguarde..."     |
| Groq Whisper erro              | Responder: "Não consegui transcrever o áudio" |
| WhatsApp desconexão            | Evolution reconecta automaticamente (Baileys) |
| Bridge crash                   | Systemd auto-restart (VPS)                    |
| Todas as dependências offline  | Modo degradado — respostas estáticas          |

---

## 6. Segurança

| Camada              | Medida                                         |
|---------------------|------------------------------------------------|
| Transporte          | Todas as APIs internas usam HTTPS              |
| Autenticação        | API keys em header `x-api-key`                 |
| Webhook             | Validação de assinatura (HMAC) no payload      |
| Secrets             | `.env` (não versionado)                        |
| Logs                | Sanitizados — sem tokens ou conteúdo de áudio  |
| Rate limiting       | 30 msg/min por contato WhatsApp                |
