# Case MarTech — aquisição, atribuição e CRM

## Evidência atual

Página HTML exportada de VSL com player e código de rastreamento condicional do pixel. A presença desse código não comprova ativação, consentimento, entrega de eventos ou conversões. Não foi encontrado um backend de CRM no repositório; a arquitetura abaixo é proposta.

Revisão por código, sem envio de lead real, acionamento de CRM ou alteração de campanha.
Não há evidência suficiente para afirmar uplift de conversão, receita ou ROI.

## Arquitetura de aquisição proposta

```text
Anúncio / orgânico → landing page → formulário/CTA → API de leads
                                                     ↓ validação / limite / idempotência
                                                fila + registro durável
                                                     ↓
                                                CRM → nutrição → oportunidade
                                                     ↓
                                              eventos agregados de conversão
```

Landing page apresenta oferta e coleta apenas dados necessários. A API valida e
normaliza, grava uma aceitação durável e responde com ID opaco. Um worker integra
CRM/planilha/e-mail com retries limitados e fila de falhas. Segredos ficam no servidor.
O CRM é a fonte operacional do lead; a planilha, quando necessária, é uma projeção
com acesso restrito. Não encadear três chamadas do navegador como se fossem transação.

## Contrato de atribuição

| Campo | Uso | Exemplo sintético |
| --- | --- | --- |
| utm_source | Origem | newsletter |
| utm_medium | Canal | email |
| utm_campaign | Campanha estável | curso_demo_2026 |
| utm_content | Criativo/variação | hero_a |
| utm_term | Palavra-chave quando aplicável | curso_demo |
| landing_path | Página sem query/fragmento | /campanha |
| lead_id | Identificador opaco do backend | UUID |

Padronizar minúsculas e nomes de campanha; limitar tamanho e aceitar somente campos
conhecidos. Nunca colocar nome, e-mail, telefone, documento ou segredo em UTMs/URLs.
Registrar primeiro toque e último toque separadamente, sem sobrescrever o primeiro
em visitas seguintes. A persistência depende de política de consentimento/retencão
definida para o produto; não há essa implementação completa nestes repositórios.
`(direct)`/`(unknown)` devem ser categorias explícitas, não atribuição inventada.

## API e segurança propostas

`POST /api/leads` recebe nome, email, telefone, dificuldade, source e utms; valida
novamente no servidor, aplica limites por origem/IP e proteção contra automação.
Use chave de idempotência opaca com unicidade e TTL no banco. CORS restrito não é
autenticação nem proteção suficiente contra abuso. Não expor chaves de CRM ou webhook
privado em JS, repositório, querystring ou logs. Não registrar corpos com dados pessoais.

Resposta aceita: 201/202 JSON `{"accepted":true,"lead_id":"identificador-opaco"}`
apenas depois de persistência durável. Erros 400/429/503 retornam código genérico e
request ID. Aceitar lead não significa e-mail entregue. Resposta opaca de `no-cors`
não comprova gravação; não apresentar sucesso nesse caso.

## Eventos e métricas propostas

`landing_view` → `cta_click` → `form_start` → `lead_accepted` → `crm_synced` →
`opportunity_created`. Últimos três confirmados pelo servidor, deduplicados por ID.
Sem PII nos eventos analíticos. Vincular consentimento quando aplicável e disponibilizar
aviso de privacidade com finalidade, retenção e canal de atendimento antes de operar.

Taxa de captura = leads aceitos / visitas elegíveis; custo por lead = gasto / leads
aceitos; conversão comercial = oportunidades / leads aceitos. Definir janela, timezone,
tratamento de bots, duplicatas e atribuição antes de comparar campanhas. Não confundir
clique em CTA com venda. Dados financeiros e de plataforma não foram auditados aqui.

## Plano de homologação

1. Fixtures sintéticas: UTMs ausentes, caracteres inválidos, campanha longa e navegação entre páginas.
2. Validação duplicada cliente/servidor; envio repetido cria um único lead.
3. CRM indisponível: aceitação durável, retry e reconciliação, sem falsa confirmação de e-mail.
4. Sem consentimento: nenhum rastreador opcional; retirada impede eventos posteriores.
5. Formulário por teclado, feedback de erro e mobile; revisar desempenho e acessibilidade.
6. Contrato de eventos e dashboard reconciliados com CRM; nenhuma PII em analytics.

O roadmap é proposta de engenharia. Não modifica campanhas, listas, automações de
e-mail ou infraestrutura ativa.
