# 🌉 WhatsApp Bridge Agent

**Bridge inteligente WhatsApp ↔ IA** — pipeline completo que conecta mensagens do WhatsApp a um agente Hermes com processamento de linguagem natural, transcrição de áudio e busca contextual.

---

## 📋 Visão Geral

O **WhatsApp Bridge Agent** estabelece uma ponte bidirecional entre o WhatsApp e um agente de IA (Hermes Agent). Mensagens de texto e áudio são recebidas via Evolution API, roteadas por um bridge server dedicado, e processadas pelo Hermes Agent que retorna respostas inteligentes — incluindo buscas rápidas via DuckDuckGo e transcrição de áudio com Groq Whisper.

### Pipeline de Dados

```
WhatsApp  →  Evolution API (:8081)  →  Bridge v7 (:8645)  →  Hermes API (:8642)
                                                    ↑
                                         Groq Whisper (áudio)
                                         DDGS (busca rápida)
```

### Stack Tecnológica

| Componente           | Tecnologia                        |
|-----------------------|-----------------------------------|
| Mensageria            | Evolution API (WhatsApp Gateway)  |
| Bridge Server         | Python 3.11+ (v7)                 |
| Agente de IA          | Hermes Agent API                  |
| Transcrição de áudio  | Groq Whisper                      |
| Fast-path de busca    | DuckDuckGo Search (DDGS)          |
| Keep-alive            | Healthcheck automático            |

---

## 🚀 Funcionalidades

- 💬 **Recepção e envio** de mensagens WhatsApp via Evolution API
- 🎙️ **Transcrição de áudio** com Groq Whisper (suporte a múltiplos idiomas)
- 🧠 **Processamento inteligente** com Hermes Agent (IA contextual)
- 🔍 **Fast-path DuckDuckGo** — respostas instantâneas para perguntas factuais sem passar pelo LLM
- 🔄 **Keep-alive automático** — healthcheck contínuo dos serviços
- 📦 **Arquitetura desacoplada** — cada componente roda em porta isolada
- 🔁 **Resiliência** — retry automático e reconexão em caso de falhas

---

## 📁 Estrutura do Projeto

```
whatsapp-bridge-agent/
├── assets/                  # Diagramas e recursos visuais
├── docs/
│   └── ARCHITECTURE.md      # Documentação de arquitetura
├── bridge/                  # Código do Bridge Server v7
├── skills/                  # Skills do Hermes Agent (versionadas)
├── .env.example             # Template de variáveis de ambiente
├── .gitignore
├── README.md
└── requirements.txt
```

---

## ⚙️ Configuração

### 1. Pré-requisitos

- Python 3.11+
- Evolution API em execução na porta `:8081`
- Hermes Agent API em execução na porta `:8642`
- Conta Groq (para transcrição de áudio)

### 2. Variáveis de Ambiente

Copie o arquivo de exemplo e preencha as credenciais:

```bash
cp .env.example .env
```

| Variável                   | Descrição                                    |
|----------------------------|----------------------------------------------|
| `EVO_API_KEY`              | Token da Evolution API                       |
| `EVO_INSTANCE_NAME`        | Nome da instância WhatsApp na Evolution      |
| `EVO_BASE_URL`             | URL base da Evolution API (`http://...:8081`) |
| `HERMES_API_URL`           | URL da API do Hermes Agent (`http://...:8642`) |
| `HERMES_API_KEY`           | Chave de API do Hermes Agent                 |
| `GROQ_API_KEY`             | Chave de API da Groq (Whisper)               |
| `BRIDGE_PORT`              | Porta do Bridge Server (padrão: `8645`)      |
| `KEEPALIVE_INTERVAL`       | Intervalo de healthcheck (segundos)           |

### 3. Instalação

```bash
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

### 4. Execução

```bash
# Iniciar o Bridge Server
python -m bridge.server

# Em outro terminal (ou via process manager):
# Certifique-se de que Evolution API e Hermes API estão ativos
```

---

## 🏗️ Portas de Serviço

| Serviço           | Porta   | Descrição                             |
|--------------------|---------|---------------------------------------|
| Evolution API      | `:8081` | Gateway WhatsApp                      |
| Bridge v7          | `:8645` | Roteamento e orquestração            |
| Hermes API         | `:8642` | Agente de IA                          |

---

## 🎙️ Fluxo de Áudio

1. Usuário envia áudio pelo WhatsApp
2. Evolution API recebe e notifica o Bridge via webhook
3. Bridge faz download do arquivo de áudio
4. Áudio é enviado ao **Groq Whisper** para transcrição
5. Texto transcrito é encaminhado ao Hermes Agent
6. Resposta é devolvida ao usuário pelo WhatsApp

---

## 🔍 Fast-Path DuckDuckGo

Para consultas factuais simples (ex: "Qual a cotação do dólar?", "Quem ganhou a Copa de 2022?"), o Bridge identifica o padrão e consulta diretamente o DuckDuckGo Instant Answer API, evitando latência do LLM.

---

## 🔒 Segurança

⚠️ **Nunca versione credenciais.** Todas as secrets (tokens Evolution, chaves Hermes, Groq API Key) devem ser armazenadas exclusivamente no arquivo `.env`, que está listado no `.gitignore`.

- `.env` e `.env.*` → ignorados pelo Git
- `credentials*.json` e `*service-account*.json` → ignorados
- `config.yaml` → ignorado (pode conter segredos)
- Diretório `scripts/` → ignorado (exceto `!skills/`, que são versionados)

---

## 📄 Licença

MIT © Alan Costa · [alancosta.dev](https://alancosta.dev)
