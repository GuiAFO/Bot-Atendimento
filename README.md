# WhatsApp Bot — Cloud API Oficial (Meta)

Bot de atendimento reativo com custo Meta zero: responde apenas dentro da janela
de serviço de 24h aberta pelo cliente, com camada opcional de linguagem natural
(Claude) e escalonamento automático para atendimento humano.

## Estrutura

```
whatsapp-bot/
├── app/
│   ├── main.py      # FastAPI: GET/POST /webhook, regras de negócio
│   ├── config.py    # Configurações (lidas do .env)
│   ├── whatsapp.py  # Envio de mensagens (Graph API)
│   ├── ia.py        # Claude (linguagem natural) + menu fallback
│   └── db.py        # SQLite: janela 24h, modo bot/humano, histórico
├── requirements.txt
├── Procfile         # Comando de start (Railway/Render)
├── .env.example     # Modelo — copie para .env e preencha
└── .gitignore
```

## Rodando localmente

```bash
python -m venv .venv
.venv\Scripts\activate        # Windows
pip install -r requirements.txt
copy .env.example .env         # e preencha os valores
uvicorn app.main:app --reload --port 8000
```

Em outro terminal:

```bash
ngrok http 8000
```

Use a URL HTTPS do ngrok + `/webhook` na configuração do webhook no painel da Meta
(Fase 2 do guia). O `VERIFY_TOKEN` do `.env` deve ser idêntico ao digitado no painel.

## Teste rápido sem WhatsApp

Com o servidor rodando, simule uma mensagem recebida:

```bash
curl -X POST http://localhost:8000/webhook -H "Content-Type: application/json" -d "{\"entry\":[{\"changes\":[{\"value\":{\"contacts\":[{\"profile\":{\"name\":\"Teste\"}}],\"messages\":[{\"from\":\"5511999999999\",\"type\":\"text\",\"text\":{\"body\":\"oi\"}}]}}]}]}"
```

O envio real falhará sem token válido, mas os logs mostram todo o fluxo.

## Deploy (Railway)

1. Suba o projeto para um repositório no GitHub (sem o `.env`).
2. Railway → New Project → Deploy from GitHub repo.
3. Em Variables, cadastre as mesmas chaves do `.env.example`.
4. O `Procfile` já define o comando de start.
5. Copie o domínio público gerado e atualize a URL do webhook no painel da Meta.

## Regras de custo implementadas

- O bot **nunca inicia** conversa — apenas responde (mensagem de serviço = R$ 0).
- Fora da janela de 24h, o bot silencia e registra em log.
- Gatilhos (`humano`, `orçamento`, etc.) mudam a conversa para modo `humano`
  e o bot para de responder até intervenção manual no banco.


## Sistema de etiquetas

Cada conversa carrega uma etiqueta de estágio, visível nos endpoints administrativos:

| Etiqueta | Como é aplicada |
|---|---|
| `novo_contato` | Padrão ao criar a conversa |
| `aguardando_resposta_cliente` | Automática após o bot responder |
| `interessado` | Palavras-chave (preço, plano, valor...) ou IA |
| `visita_agendada` | Palavras-chave (agendar, confirmo...) ou IA |
| `atendimento_humano` | Escalonamento (automático ou manual) |
| `finalizado` | Apenas manual (via endpoint) |

Com a IA ativa, o Claude devolve resposta **e** etiqueta na mesma chamada (JSON),
sem custo adicional. Etiquetas de estágio avançado (`interessado`, `visita_agendada`)
não são rebaixadas por mensagens genéricas posteriores.

### Endpoints administrativos (header `X-Admin-Token`)

```bash
# Listar todas as conversas
curl -H "X-Admin-Token: SEU_TOKEN" https://seu-app/admin/conversas

# Filtrar por etiqueta
curl -H "X-Admin-Token: SEU_TOKEN" "https://seu-app/admin/conversas?etiqueta=visita_agendada"

# Definir etiqueta manualmente
curl -X POST -H "X-Admin-Token: SEU_TOKEN" -H "Content-Type: application/json" \
  -d '{"telefone":"5511999999999","etiqueta":"finalizado"}' https://seu-app/admin/etiqueta

# Devolver conversa escalada ao bot
curl -X POST -H "X-Admin-Token: SEU_TOKEN" -H "Content-Type: application/json" \
  -d '{"telefone":"5511999999999","modo":"bot"}' https://seu-app/admin/modo
```


## Segurança e proteção contra prompt injection

Defesa em camadas — nenhuma sozinha é suficiente:

1. **Assinatura HMAC do webhook** (`APP_SECRET`): todo POST é validado contra o header
   `X-Hub-Signature-256`. Requisições que não vieram da Meta são descartadas com 403.
   Obrigatório em produção; se vazio, apenas loga aviso (modo dev).
2. **Prompt de sistema endurecido**: bloco de regras invioláveis instruindo o modelo a
   tratar mensagens do cliente como dados, nunca como instruções (recusa troca de papel,
   vazamento de prompt, mudança de formato).
3. **Escalonamento fora da IA**: os gatilhos de atendimento humano são verificados por
   código ANTES da chamada ao modelo — injeção nenhuma desativa essa rota.
4. **Trava de entrada**: mensagens acima de `MAX_TAMANHO_MSG` são truncadas.
5. **Trava de saída**: respostas acima de `MAX_TAMANHO_RESPOSTA` são cortadas e a
   etiqueta sugerida pela IA só é aceita se estiver na lista fechada `ETIQUETAS_VALIDAS`.
6. **Rate limit por telefone** (`MAX_MSG_POR_HORA`): protege o custo de API e evita flood.
7. **Superfície mínima**: o bot não executa ações nem acessa dados de terceiros — o pior
   caso de uma injeção bem-sucedida é uma resposta fora de tom, nunca vazamento de dados.
   Por isso, nunca coloque segredos, listas de clientes ou preços internos no SYSTEM_PROMPT.


## Painel de acompanhamento (/painel)

Acesse `https://seu-app/painel` no navegador (funciona também no celular).
Login com o `ADMIN_TOKEN`. Recursos:

- **Lista de conversas** com nome, telefone, modo (🤖 bot / 👤 humano) e etiqueta colorida
- **Filtros por etiqueta** em um clique (ex.: só "Visita agendada")
- **Thread completa** da conversa em bolhas, com data e hora
- **Responder como humano** direto pelo painel — o envio respeita a janela de 24h
  (dentro dela = gratuito; fora, o botão desativa e explica o motivo)
- **Assumir / devolver ao bot** com um clique; ao responder manualmente, o bot
  é silenciado automaticamente naquela conversa
- **Trocar etiqueta manualmente** (ex.: marcar "Finalizado")
- Atualização automática a cada 10 segundos

O painel é um único arquivo (`app/painel.html`) servido pelo próprio FastAPI —
sem hospedagem extra, sem dependência de build.

## Próximos passos sugeridos

- Notificação ao dono quando houver escalonamento (e-mail ou mensagem template).
- Suporte a áudio/imagem (hoje apenas texto).
- Migração do SQLite para Postgres (Railway/Supabase) em produção.
