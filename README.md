# Triagem de Currículos com IA (n8n + Gemini)

Sistema de automação que recebe currículos por email, analisa cada candidato com IA contra critérios configuráveis, e disponibiliza um assistente via Telegram para consultar candidatos e gerenciar o processo — sem precisar abrir planilha nenhuma.

Dois workflows n8n compõem o sistema:

- **Wf1 — Trigger Email**: monitora a caixa de entrada, extrai o texto do PDF, avalia o candidato com Gemini e distribui o resultado.
- **Wf2 — Assistente Telegram**: bot conversacional para consultar candidatos, alterar critérios de aprovação e disparar emails, com memória de conversa.

## Arquitetura

```
┌─────────────────────────────────────────────────────────────────┐
│ WORKFLOW 1 — Triagem automática                                  │
│                                                                    │
│  Gmail Trigger ──▶ OCR.space (PDF→texto) ──▶ Dedup por email     │
│        │                                          │                │
│        ▼                                          ▼                │
│  Busca critérios atuais ──▶ Gemini 2.5 Flash Lite (score + json) │
│        │                                          │                │
│        ▼                                          ▼                │
│  Aprovado? ──▶ SIM: Telegram + Sheets + DataTable                │
│           └──▶ NÃO: log em "rejeitados"                          │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│ WORKFLOW 2 — Assistente Telegram                                  │
│                                                                    │
│  Telegram Trigger ──▶ Identifica comando ──▶ Roteador             │
│        │                    │                     │                │
│        ▼                    ▼                     ▼                │
│   /criterios             /email             conversa livre        │
│   (ver/editar)      (dispara email)    (Gemini + memória +        │
│                                          dados de candidatos)      │
└─────────────────────────────────────────────────────────────────┘
```

## Stack

- **n8n** — orquestração dos workflows
- **Google Gemini 2.5 Flash Lite** — análise de currículo e conversa (via `@n8n/n8n-nodes-langchain.googleGemini`)
- **OCR.space API** — extração de texto de currículos em PDF
- **Gmail API** — trigger de novos emails e envio de respostas
- **Telegram Bot API** — interface de notificação e chat
- **Google Sheets** — planilha de acompanhamento
- **n8n Data Tables** — persistência (`candidatos`, `rejeitados`, `criterios`, `historico`, `sessao`)

## Workflow 1 — Trigger Email

1. **Monitor Resume Emails**: verifica a caixa a cada 1 minuto, filtrando por assunto (`vaga dev senior`), com download automático de anexos.
2. **Extract Text via OCR**: envia o PDF anexado para o OCR.space e recupera o texto.
3. **Check Duplicate Candidate**: consulta a data table `candidatos` pelo email do remetente e descarta se já existir.
4. **Buscar criterios**: lê os critérios de aprovação vigentes (idade mínima, inglês mínimo, experiência mínima, skills obrigatórias).
5. **Analyze Resume with Gemini**: envia o currículo + critérios num prompt estruturado e recebe de volta um JSON com `aprovado`, `nome`, `idade`, `ingles`, `anos_experiencia`, `skills`, `motivo` e `score` (0–100).
6. **Check Approval** → dois caminhos:
   - **Aprovado**: notifica via Telegram (mensagem formatada em Markdown), grava na planilha Google Sheets e na data table `candidatos`.
   - **Reprovado**: grava na data table `rejeitados` com o motivo, sem notificar.

O score é calculado por pesos (idade, inglês, anos de experiência, skills), permitindo ranquear candidatos aprovados por qualidade, não só aprovar/reprovar binariamente.

## Workflow 2 — Assistente Telegram

Roteamento por comando, identificado a partir do texto recebido:

| Comando | O que faz |
|---|---|
| `/criterios` | Mostra os critérios atuais de aprovação |
| `/criterios idade: 25` <br>`/criterios ingles: avancado` <br>`/criterios experiencia: 5` <br>`/criterios skills: Python,React,Node.js` | Atualiza um ou mais critérios (aceita múltiplas linhas na mesma mensagem) |
| `/email` | Dispara um email para um candidato (aprovação/rejeição) |
| qualquer outra mensagem | Cai no modo conversa livre com IA |

**Modo conversa livre**: o workflow busca em paralelo o histórico de conversa, todos os candidatos aprovados e rejeitados, e o estado de paginação (offset) salvo na data table `sessao`. Tudo isso vira contexto de um prompt único enviado ao Gemini, que entende perguntas como:

- "quantos candidatos foram rejeitados?"
- "mostra os 3 mais qualificados"
- "quem se candidatou essa semana?"
- "ver mais" (pagina os resultados anteriores)

A IA responde em JSON (`texto` + `novoOffset`), o histórico é salvo (mensagem do usuário + resposta) e o offset de paginação é persistido para a próxima pergunta.

## Modelo de dados (n8n Data Tables)

| Tabela | Campos | Uso |
|---|---|---|
| `candidatos` | nome, email, idade, ingles, experiencia, skills, aprovado, score, data_envio | Aprovados |
| `rejeitados` | nome, email, idade, ingles, experiencia, skills, motivo, score, data_envio | Reprovados |
| `criterios` | idade_minima, ingles_minimo, experiencia_minima, skills_obrigatorias | Regras de aprovação, editáveis via Telegram |
| `historico` | role, mensagem, timestamp | Memória de conversa do bot |
| `sessao` | chave, valor | Estado (ex.: offset de paginação) |

## Configuração

Credenciais necessárias no n8n:

- Gmail OAuth2 (leitura + envio)
- Google Sheets OAuth2
- Telegram Bot API
- Google Gemini (PaLM) API

Variáveis a configurar antes de importar os workflows (os arquivos já vêm com placeholders):

- `SUA_API_KEY_OCR_SPACE` — chave gratuita em [ocr.space/ocrapi](https://ocr.space/ocrapi)
- `SEU_CHAT_ID_TELEGRAM` — ID do chat/usuário que recebe as notificações

> ⚠️ **Nota de segurança**: os JSONs originais desses workflows continham a API key do OCR.space e o chat ID do Telegram hardcoded em texto plano. Antes de publicar qualquer export de workflow n8n, sempre revise por credenciais expostas — o n8n não sanitiza isso automaticamente ao exportar.

## Limitações conhecidas

- Chat ID do Telegram fixo no workflow (não suporta múltiplos usuários sem adaptação).
- Sem tratamento de rate limit da API do Gemini.
- Paginação de candidatos depende de um único registro de sessão (não é por usuário).

## Arquivos

- `Wf1_Trigger_Email.json` — export do workflow de triagem
- `Wf2_Assistente_Telegram.json` — export do assistente conversacional

Importe ambos diretamente no n8n (`Import from File`), configure as credenciais e ajuste os placeholders antes de ativar.

## ⚙️ Como configurar

### Pré-requisitos
- Uma instância do n8n (n8n Cloud ou instalação própria)
- Conta Google com Gmail (autenticação OAuth2)
- Chave de API do Google Gemini
- Bot do Telegram criado pelo @BotFather (token do bot)

### Passo a passo
1. No n8n, vá em **Workflows > Import from file** e importe os dois arquivos:
   - `WorkFlow_Trigger_Email.json` (triagem automática de currículos)
   - `WorkFlow_Assistente_Telegram.json` (assistente via Telegram)
2. Crie as credenciais usadas pelos nós: **Gmail OAuth2**, **Google Gemini (API)** e **Telegram API**.
3. Crie as **Data Tables** usadas pelos fluxos (candidatos, critérios, sessão e histórico) e confira os nomes configurados nos nós.
4. Revise os critérios de aprovação e ajuste-os ao perfil da vaga.
5. Ative os dois workflows.

### Como testar
- Envie um e-mail com um currículo em PDF para a caixa monitorada e confira se o candidato é analisado e registrado.
- Envie uma mensagem ao bot do Telegram para consultar candidatos e alterar os critérios.

> As credenciais não estão incluídas neste repositório; cada usuário deve criar as suas.
