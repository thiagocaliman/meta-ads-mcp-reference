# Meta Ads MCP — Referência Técnica Completa

**Português** | [English](README.en.md)

> Documentação não oficial das **82 ferramentas** do servidor MCP oficial da Meta Ads (`https://mcp.facebook.com/ads`), com tipo de operação, parâmetros, valores aceitos, padrões e limitações. Baseada nos schemas do conector e na documentação oficial da Meta.

**Última atualização:** julho/2026 · **API base:** Marketing API v25.0

> **Guia visual e material canônico:** [MCP Meta Ads — guia completo para gestores de tráfego](https://thiagocaliman.com.br/mcp-meta/)  
> Este repositório funciona como a documentação técnica pública e versionada do material.

- **Autor e mantenedor:** [Thiago Caliman](https://thiagocaliman.com.br/)
- **Organização/publicadora:** AI PRO Revolution
- **Índice legível por agentes:** [`data/tools-index.json`](data/tools-index.json)
- **Como citar:** [`CITATION.cff`](CITATION.cff)
- **Histórico de versões:** [`CHANGELOG.md`](CHANGELOG.md)
- **Licença:** [CC BY 4.0](LICENSE)

---

## Visão geral

- **Servidor:** `https://mcp.facebook.com/ads` — beta aberto pela Meta em 29/04/2026, com rollout gradual por conta de anúncio e por ferramenta individual.
- **Autenticação:** OAuth 2.0 da Meta. Escopos **Read** (leitura) e **Manage** (gestão) configuráveis por app em *facebook.com → Configurações → Business Integrations*, revogáveis a qualquer momento. Não exige developer app, app review nem token manual.
- **Clientes suportados:** Claude Desktop, Claude Code, ChatGPT (web), Codex (app/CLI), Cursor (app/CLI).
- **Rate limit:** mesmo throttling da Marketing API (~200 chamadas/hora por conta de anúncio, segundo relatos da comunidade). Chamadas pesadas ou repetidas podem ser desaceleradas ou bloqueadas temporariamente.
- **Rollout:** contas sem acesso retornam `is_ads_mcp_enabled = false`; ferramentas individuais podem retornar erro de rollout mesmo em contas habilitadas.

### Segurança (recomendações oficiais da Meta)

- Manter o escopo **Read** a menos que o agente precise escrever.
- Atenção a **prompt injection**: conteúdo não confiável lido pelo agente pode tentar induzir ações indevidas.
- Auditar integrações conectadas periodicamente e revogar as não utilizadas.
- Os *Meta Platform Terms* governam retenção e uso dos dados retornados.

## Mapa geral

| Categoria | Ferramentas | Leitura | Escrita | Exclusão |
|---|---|---|---|---|
| 1. Descoberta e leitura da conta | 17 | 17 | 0 | 0 |
| 2. Insights e diagnóstico | 6 | 6 | 0 | 0 |
| 3. Datasets (Pixel) e conversões | 5 | 5 | 0 | 0 |
| 4. Públicos personalizados | 7 | 3 | 3 | 1 |
| 5. Pixel: eventos e parâmetros | 8 | 2 | 4 | 2 |
| 6. Criação e gestão de campanhas | 7 | 0 | 7 | 0 |
| 7. Experimentos (A/B e lift) | 7 | 4 | 3 | 0 |
| 8. Catálogo — leitura | 15 | 15 | 0 | 0 |
| 9. Catálogo — escrita | 10 | 0 | 9 | 1 |
| **TOTAL** | **82** | **52** | **26** | **4** |

> Além das 4 ferramentas de exclusão nominais, a operação `REMOVE` de `ads_update_custom_audience_users` também é destrutiva.

### O que o MCP NÃO consegue fazer (limites da própria Meta)

- Fazer **upload de imagens ou vídeos** para a biblioteca da conta (apenas lista os existentes).
- **Excluir** campanhas, conjuntos, anúncios ou criativos (não existe ferramenta de delete — apenas pausar via update).
- Excluir catálogos, conjuntos de produtos ou feeds.
- **Editar o conteúdo de um criativo** existente (criativos são imutáveis — o fluxo é criar novo criativo + novo anúncio).
- Acessar contas com `is_ads_mcp_enabled = false` (rollout gradual).
- Extração em massa da Biblioteca de Anúncios (scraping vedado).
- Ver dados de faturamento/cobrança ou resolver contas restritas.

---

## 1. Descoberta e leitura de estrutura da conta

Ferramentas de leitura para descobrir contas, páginas, criativos, mídias e diagnosticar a estrutura. Nenhuma delas altera nada.

### `ads_get_ad_accounts`

**🟢 Leitura**

Lista as contas de anúncio acessíveis ao usuário, paginadas em blocos de 50. Retorna por conta: ad_account_id, ad_account_name, business_id, business_name, is_ads_mcp_enabled, is_queryable, account_status, has_payment_method, currency e min_daily_budget_cents (orçamento diário mínimo, ex.: 516 = R$5,16 em BRL).

**Parâmetros:**

- `cursor` (opcional) — cursor de paginação retornado como next_cursor na chamada anterior; nunca inventar
- `limit` (opcional, int) — quantidade por página; padrão 50

**Atenção:**

- ⚠️ Se `is_ads_mcp_enabled = false`, a conta NÃO pode ser usada em nenhuma outra ferramenta do MCP (rollout gradual da Meta).
- ⚠️ Se `is_queryable = false` (contas fechadas/desativadas), não usar `ads_get_ad_entities` nela; exibir `not_queryable_reason`.

### `ads_get_ad_account_pages`

**🟢 Leitura**

Lista as Páginas do Facebook promovidas sob uma conta de anúncio específica, com o campo `leadgen_tos_accepted` (indica se a Página aceitou os Termos de anúncios de cadastro).

**Parâmetros:**

- `ad_account_id` (obrigatório) — ID numérico sem prefixo `act_`
- `cursor` (opcional)
- `limit` (opcional; padrão 50)

**Atenção:**

- ⚠️ Anúncios de leads (LEAD_GENERATION / QUALITY_LEAD) exigem Página com `leadgen_tos_accepted = true`; aceite em facebook.com/legal/leadgen/tos.

### `ads_get_user_pages`

**🟢 Leitura**

Lista todas as Páginas em que o usuário tem permissão `CREATE_ADS` (administradas diretamente ou via Business).

**Parâmetros:**

- `cursor` (opcional)
- `limit` (opcional; padrão 50)

### `ads_get_pages_for_business`

**🟢 Leitura**

Lista as Páginas pertencentes a um Business Manager específico.

**Parâmetros:**

- `business_id` (obrigatório) — ID numérico
- `cursor` (opcional)
- `limit` (opcional; padrão 50)

### `ads_get_ad_entities`

**🟢 Leitura**

A principal ferramenta de dados e relatórios. Consulta conta, campanhas, conjuntos ou anúncios com métricas, atributos, breakdowns, filtros e ordenação — equivalente aos relatórios do Gerenciador de Anúncios.

**Parâmetros:**

- `ad_account_id` (obrigatório)
- `fields` (obrigatório, lista) — métricas/atributos; sempre incluir `id` e `name`; validar nomes com `ads_get_field_context`
- `level` (opcional) — `account` | `campaign` | `adset` | `ad`
- `date_preset` (opcional) — `today`, `yesterday`, `this_month`, `last_month`, `this_quarter`, `last_3d`, `last_7d`, `last_14d`, `last_30d`, `last_90d`, `last_week_sun_sat`, `last_quarter`, `last_year`, `this_week_sun_today`, `this_year`, `maximum`
- `time_range` (opcional) — JSON `{"since":"AAAA-MM-DD","until":"AAAA-MM-DD"}`; nunca junto com `date_preset`
- `time_increment` (opcional) — `1` a `90` (dias), `monthly` ou `all_days`
- `breakdowns` (opcional) — apenas UM por chamada: `age`, `gender`, `country`, `region`, `dma`, `device_platform`, `publisher_platform`, `platform_position`, `impression_device`, `hourly_stats` (fuso do anunciante ou do público), `product_id`, `frequency_value`, `user_segment_key`, e breakdowns de ação (`action_type`, `action_device`, `action_destination`, `action_reaction`, `action_video_sound`, `action_video_type`, `action_carousel_card_id/name`, `action_canvas_component_name`, `action_target_id`) e de assets (`image_asset`, `video_asset`, `title_asset`, `body_asset`, `description_asset`, `call_to_action_asset`, `link_url_asset`, `ad_format_asset`), além de `app_id`, `skan_campaign_id`, `skan_conversion_id`, `is_conversion_id_modeled`, `place_page_id`
- `filtering` (opcional) — lista de `{field: "nivel.campo", operator, value[]}`; respeitar operadores permitidos por campo
- `sort` (opcional) — formato `metrica_ascending` / `metrica_descending`
- `limit` (opcional) — máximo 1000

**Atenção:**

- ⚠️ Métricas, filtros e ordenação SÓ funcionam com intervalo de tempo definido; sem ele, retorna apenas atributos.
- ⚠️ Há teto interno de entidades retornadas — pode vir um subconjunto; para maior e menor valor de uma métrica, fazer 2 chamadas com ordenações opostas.
- ⚠️ Não repetir a mesma chamada com parâmetros idênticos (desperdiça rate limit).
- ⚠️ Não pedir `actions`/`action_values` como campo avulso.

### `ads_get_ad_preview`

**🟢 Leitura**

Gera preview visual de um anúncio ou criativo em diversos posicionamentos, com HTML embutido, link do preview e detalhes do criativo (texto, título, CTA).

**Parâmetros:**

- `ad_format` (obrigatório) — `DESKTOP_FEED_STANDARD`, `MOBILE_FEED_STANDARD`, `INSTAGRAM_STANDARD`, `INSTAGRAM_STORY`, `INSTAGRAM_REELS`, `RIGHT_COLUMN_STANDARD`, `MESSENGER_MOBILE_INBOX_MEDIA`, `THREADS_STREAM`
- `ad_id` OU `creative_id` (um dos dois)

### `ads_get_ad_images`

**🟢 Leitura**

Lista imagens ativas da biblioteca da conta (mais recentes primeiro). O hash identifica a imagem para uso em criativos.

**Parâmetros:**

- `ad_account_id` (obrigatório)
- `hashes` (opcional, lista) — busca imagens específicas; ignora name/limit/cursor
- `fields` (opcional) — `name`, `hash`, `status`, `width`, `height`, `original_width`, `original_height`, `url`, `url_128`, `permalink_url`, `created_time`, `updated_time`
- `name` (opcional) — filtro por substring, sem diferenciar maiúsculas
- `limit` (opcional; padrão 25, máx 100)
- `cursor` (opcional)

**Atenção:**

- ⚠️ Listagem sem `hashes` retorna SÓ `hash` + `name` (resultado parcial); para outros campos, rechamar com `hashes` ou `fields`.
- ⚠️ NÃO PODE: fazer upload de novas imagens (não suportado pelo MCP).

### `ads_get_ad_videos`

**🟢 Leitura**

Lista vídeos da conta com status OK (mais recentes primeiro). O `id` (FBID) é usado em criativos de vídeo.

**Parâmetros:**

- `ad_account_id` (obrigatório)
- `video_ids` (opcional, lista)
- `fields` (opcional) — `id`, `title`, `description`, `length`, `created_time`, `updated_time`, `permalink_url`, `picture`
- `title` (opcional) — filtro substring
- `limit` (opcional; padrão 25, máx 100)
- `cursor` (opcional)

**Atenção:**

- ⚠️ Listagem simples retorna só `id` + `title`.
- ⚠️ NÃO PODE: fazer upload de novos vídeos.

### `ads_get_creatives`

**🟢 Leitura**

Lista criativos da conta. No Gerenciador, `body` = "texto principal" e `title` = "título".

**Parâmetros:**

- `ad_account_id` (obrigatório)
- `creative_ids` (opcional, lista)
- `fields` (opcional) — `id`, `name`, `status`, `account_id`, `object_type`, `body`, `title`, `link_url`, `image_hash`, `image_url`, `video_id`, `thumbnail_url`, `call_to_action_type`, `object_story_id`, `effective_object_story_id`, `effective_instagram_media_id`, `product_set_id`, `child_attachments`
- `limit` (opcional; padrão 25, máx 100)
- `cursor` (opcional)

**Atenção:**

- ⚠️ Listagem sem `creative_ids` retorna só `id`, `name`, `account_id` e `status`.

### `ads_get_creative_ads`

**🟢 Leitura**

Lista os anúncios que usam determinado criativo.

**Parâmetros:**

- `creative_id` (obrigatório)
- `limit` (opcional; padrão 25, máx 100)
- `cursor` (opcional)

### `ads_get_ig_accounts`

**🟢 Leitura**

Lista contas do Instagram vinculadas à conta de anúncio, utilizáveis para impulsionamento.

**Parâmetros:**

- `ad_account_id` (obrigatório)
- `cursor` (opcional)
- `limit` (opcional; padrão 25)

**Atenção:**

- ⚠️ Zero resultados = nenhum IG vinculado ou falta da permissão `instagram_basic`.

### `ads_get_ig_media`

**🟢 Leitura**

Lista mídia anunciável do Instagram (posts, reels, stories) de uma conta IG retornada por `ads_get_ig_accounts`.

**Parâmetros:**

- `ad_account_id` (obrigatório) — o mesmo usado em `ads_get_ig_accounts`
- `ig_account_id` (obrigatório)
- `filters` (opcional, JSON) — `date_range` (in_range/not_in_range, timestamps unix), `media_type` (IMAGE, VIDEO, CAROUSEL_ALBUM), `product_type` (FEED, STORY, REELS)
- `limit` (opcional; padrão e máx 25)
- `cursor` (opcional)

### `ads_get_errors`

**🟢 Leitura**

Retorna erros que bloqueiam a entrega (hard-stop) de campanhas, conjuntos e anúncios.

**Parâmetros:**

- `entity_ids` (obrigatório, lista) — IDs numéricos; passar a conta retorna erros dos filhos
- `limit` (opcional; padrão 50)

**Atenção:**

- ⚠️ NÃO cobre: problemas de performance/pacing, conta desativada/restrita, rejeição de anúncio.

### `ads_get_field_context`

**🟢 Leitura**

Catálogo de metadados dos campos de relatório: tipo, descrição, níveis suportados, filtrável/ordenável, operadores e valores de enum. Resolve aliases (ex.: `spend` → `amount_spent`).

**Parâmetros:**

- `field_names` (opcional, lista) — vazio retorna o catálogo completo

**Atenção:**

- ⚠️ `cost_per_result` não existe no nível `account`; `daily_budget` não é ordenável; enums completos de `objective` e `status` retornados.

### `ads_get_help_article`

**🟢 Leitura**

Busca artigos da Central de Ajuda da Meta sobre conceitos, políticas e guias.

**Parâmetros:**

- `search_query` (obrigatório)

**Atenção:**

- ⚠️ Não usar para métricas da conta nem problemas de cobrança/conta restrita.

### `ads_get_opportunity_score`

**🟢 Leitura**

Retorna o Opportunity Score (0–100) da CONTA e recomendações priorizadas por ganho estimado em pontos, segundo as melhores práticas da Meta.

**Parâmetros:**

- `ad_account_id` (obrigatório)

**Atenção:**

- ⚠️ O score é sempre da conta inteira — nunca de campanha/conjunto/anúncio específico.

### `ads_account_get_activity_logs`

**🟢 Leitura**

Histórico de alterações da conta (espelho da página de histórico do Gerenciador), incluindo mudanças feitas pelo sistema da Meta, com autor, tipo de evento e valores antes/depois.

**Parâmetros:**

- `ad_account_id` (obrigatório)
- `start_time` / `end_time` (opcionais, ISO 8601; padrão últimos 3 meses)
- `event_category` (opcional) — `account`, `ad`, `ad_set`, `audience`, `bid`, `budget`, `campaign`, `date`, `status`, `targeting`, `ad_keywords`
- `object_id` (opcional) — campanha/conjunto inclui descendentes
- `user_id` (opcional)
- `limit` (opcional; 1–1000, padrão 100)

---

## 2. Insights e diagnóstico de performance

Ferramentas analíticas de leitura, exclusivas do MCP (não existem como endpoints públicos tradicionais na Marketing API).

### `ads_insights_advertiser_context`

**🟢 Leitura**

Visão geral do contexto de negócio e do funil do anunciante, para orientar a estratégia de otimização.

**Parâmetros:**

- `ad_account_id` (obrigatório)
- `entity_ids` (opcional, lista — todas do mesmo tipo)
- `date_preset` (opcional) — `TODAY`, `YESTERDAY`, `LAST_2_DAYS`, `LAST_7D`, `LAST_14D`, `LAST_28D`, `LAST_30D`, `THIS_WEEK`, `LAST_WEEK`, `THIS_MONTH`, `LAST_MONTH`, `LIFETIME`
- `date_from` + `date_to` (opcionais, AAAA-MM-DD; excludentes com `date_preset`)

### `ads_insights_anomaly_signal`

**🟢 Leitura**

Detecta anomalias de performance: picos, quedas e padrões incomuns na conta ou em entidades específicas. Indica onde investigar, não conclusões definitivas.

**Parâmetros:**

- `ad_account_id` (obrigatório)
- `entity_ids` (opcional)

**Atenção:**

- ⚠️ Não cobre erros de configuração/entrega (usar `ads_get_errors`).

### `ads_insights_auction_ranking_benchmarks`

**🟢 Leitura**

Mostra quais anúncios competem melhor no leilão e fatores de melhoria (lance, qualidade). Detecta sobreposição de leilão causada por audiências fragmentadas.

**Parâmetros:**

- `ad_account_id` (obrigatório)
- `entity_ids` (opcional)
- `date_preset` OU `date_from`+`date_to` (mesmos valores da ferramenta acima)

### `ads_insights_industry_benchmark`

**🟢 Leitura**

Compara a performance dos conjuntos contra benchmarks agregados de anunciantes similares, com filtros por tier de investimento e meta de otimização.

**Parâmetros:**

- `ad_account_id` (obrigatório)
- `entity_ids` (opcional)
- `analysis_metric` (opcional) — `CLICKS`, `COST_PER_LEAD`, `CPC`, `CPM`, `CPR`, `CTR`, `CVR`, `IMPRESSIONS`, `REACH`, `RESULT`, `ROAS`, `SPEND`
- `cas_segment` (opcional) — `PREMIUM_TOP`, `HIGH_MID`, `LOW_TAIL`, `Basic`
- `optimization_goal_override` (opcional) — `OFFSITE_CONVERSIONS`, `RETURN_ON_AD_SPEND`, `APP_INSTALLS`, `OFFSITE_CLICKS`, `LEAD_GENERATION`, `VIDEO_VIEWS_15S`, `REPLIES`, `REACH`, `POST_ENGAGEMENT`, `LANDING_PAGE_VIEWS`
- `conversation_intent` / `conversation_topic` (opcionais, classificação da pergunta)
- `date_preset` OU `date_from`+`date_to`

**Atenção:**

- ⚠️ Comparar custo por resultado (métrica de negócio) e não apenas CPM.

### `ads_insights_performance_trend`

**🟢 Leitura**

Série temporal de CPC, CPM, CPR, ROAS, CTR e CVR — direção e mudanças da performance ao longo do tempo.

**Parâmetros:**

- `ad_account_id` (obrigatório)
- `entity_ids` (opcional)
- `analysis_level` (opcional) — `AD` ou `ADSET`
- `analysis_metric` (opcional) — mesma lista do benchmark
- `conversation_intent` / `conversation_topic` (opcionais)

### `ads_library_search`

**🟢 Leitura**

Pesquisa na Biblioteca de Anúncios pública da Meta (espionagem de concorrentes, transparência). Retorna criativo, página, data e link do anúncio.

**Parâmetros:**

- `search_terms` OU `page_ids` OU `countries` — pelo menos um
- `countries` (opcional, lista ISO-2: BR, US, GB...)
- `ad_active_status` (opcional) — `ALL`, `ACTIVE`, `INACTIVE`
- `ad_type` (opcional) — `ALL`, `POLITICAL_AND_ISSUE_ADS`, `HOUSING_ADS`, `EMPLOYMENT_ADS`, `CREDIT_ADS`
- `limit` (opcional; padrão 25, máx 50)

**Atenção:**

- ⚠️ Requer ao menos 1 conta de anúncio ativa.
- ⚠️ NÃO PODE: extração em massa / scraping da biblioteca.

---

## 3. Datasets (Pixel) e conversões — leitura

Monitoramento de pixels, apps e conjuntos de dados de conversão.

### `ads_get_datasets`

**🟢 Leitura**

Lista datasets (pixels/aplicativos) de um business ou de uma conta de anúncio, com status e datas.

**Parâmetros:**

- `business_id` OU `ad_account_id` (apenas um dos dois)
- `cursor` (opcional)
- `limit` (opcional; padrão 25, máx 100)

### `ads_get_dataset_details`

**🟢 Leitura**

Metadados do dataset: nome, status, business, configuração de uso de dados, `last_fired_time` e `server_last_fired_time`.

**Parâmetros:**

- `dataset_id` (obrigatório)

**Atenção:**

- ⚠️ Sinalizar datasets inativos ou com `last_fired_time` antigo.

### `ads_get_dataset_quality`

**🟢 Leitura**

Qualidade do sinal: EMQ (Event Match Quality), cobertura por chave de correspondência e frescor dos uploads, agrupados por canal (web, offline, crm, custom_attribution).

**Parâmetros:**

- `dataset_id` (obrigatório)
- `query_type` (opcional, lista de canais)

### `ads_get_dataset_stats`

**🟢 Leitura**

Volume de eventos do dataset (janela máxima de 28 dias).

**Parâmetros:**

- `dataset_id` (obrigatório)
- `aggregation` (opcional) — `event` (padrão), `device_type`, `event_source`, `url`, `host`, `event_total_counts`
- `event_name` (opcional) — ex.: `Purchase`, `Lead`
- `event_source` (opcional) — `WEB_ONLY` ou `SERVER_ONLY`
- `start_time` / `end_time` (opcionais) — timestamps Unix em string; padrão últimos 7 dias

**Atenção:**

- ⚠️ Sinalizar eventos com volume zero usados em campanhas ativas.

### `ads_get_customconversions`

**🟢 Leitura**

Lista conversões personalizadas da conta (sem métricas).

**Parâmetros:**

- `ad_account_id` (obrigatório)
- `dataset_id` (opcional, filtro)
- `cursor` (opcional)
- `limit` (opcional; padrão 25…2541 tokens truncated…nce vem ativado por padrão: `age_min`/`max` viram sugestão; para trava rígida de idade, `targeting_automation.advantage_audience = 0`.
- ⚠️ Orçamento mínimo por moeda: consultar `min_daily_budget_cents` em `ads_get_ad_accounts` (R$5,16/dia em BRL).

### `ads_create_creative`

**🟠 Escrita (criação)**

Cria criativo de 3 formatos: imagem única, vídeo único ou carrossel Advantage+ de catálogo.

**Parâmetros:**

- `ad_account_id`, `page_id` (sempre obrigatórios)
- Imagem: `link_url` + `image_hash` OU `image_url` (nunca ambos)
- Vídeo: `video_id` + thumbnail obrigatória (`image_hash` OU `image_url`); `link_url` opcional
- Catálogo: `product_set_id` + `link_url` (mídia vem do catálogo)
- `message` (texto principal), `headline` (título), `description`, `name`
- `call_to_action_type` — dezenas de enums (`SHOP_NOW`, `LEARN_MORE`, `SIGN_UP`, `WHATSAPP_MESSAGE`, `BOOK_NOW`, `SUBSCRIBE`...); padrão `LEARN_MORE`
- `instagram_user_id` — sem ele o criativo NÃO entrega no Instagram
- `self_ai_disclosure` — `OPT_IN`/`OPT_OUT` (declaração de conteúdo gerado por IA)

**Atenção:**

- ⚠️ Criativos são IMUTÁVEIS: para editar, criar novo criativo + novo anúncio.

### `ads_create_ad`

**🟠 Escrita (criação)**

Cria anúncio em PAUSADO sob um conjunto, referenciando um criativo.

**Parâmetros:**

- `ad_account_id`, `ad_set_id`, `ad_name`, `creative` (obrigatórios)
- `creative` (JSON) — exatamente 1 fonte: `creative_id` | `object_story_id` ("pageID_postID", promove post existente) | `object_story_spec` inline (`page_id` obrigatório + `link_data`/`video_data`/`photo_data`/`template_data`)
- opcionais: `tracking_specs`, `conversion_domain`, `bid_amount`, `adlabels`, `ad_schedule_start`/`end_time`, `source_ad_id`

### `ads_update_entity`

**🟠 Escrita (alteração)**

Atualiza campos de campanha, conjunto ou anúncio existente: nome, orçamento, targeting, agenda, status (inclusive PAUSAR).

**Parâmetros:**

- `ad_account_id` (deve ser o dono real da entidade), `entity_id`, `entity_type` (`campaign`, `ad_set`, `ad`), `fields` (JSON campo→valor; orçamentos em centavos)

**Atenção:**

- ⚠️ Não edita conteúdo de criativo (imutável).
- ⚠️ É a ferramenta usada para PAUSAR entidades ativas (`status: PAUSED`).

### `ads_activate_entity`

**🟠 Escrita (alteração)**

Publica: muda status de PAUSED para ACTIVE. **INICIA O GASTO** do orçamento.

**Parâmetros:**

- `ad_account_id`, `entity_id`, `entity_type` (`campaign`, `ad_set`, `ad`)

**Atenção:**

- ⚠️ Ativar o pai não ativa os filhos; a entrega exige campanha + conjunto + anúncio todos ativos.

### `ads_boost_ig_post`

**🟠 Escrita (criação)**

Cria anúncio a partir de post do Instagram (impulsionamento), em fluxo de 2 passos: `confirmed=false` devolve o plano; `confirmed=true` cria tudo (em PAUSADO).

**Parâmetros:**

- `ad_account_id`, `ig_account_id`, `ig_media_id` (obrigatórios)
- Padrões: ~US$5/dia equivalentes na moeda da conta, 6 dias, OUTCOME_TRAFFIC, destino INSTAGRAM_PROFILE, targeting EUA, billing IMPRESSIONS, LOWEST_COST_WITHOUT_CAP
- Sobrescrever qualquer nível: `campaign_name`/`daily_budget`/`lifetime_budget`/`bid_strategy`, `objective`, `optimization_goal`, `destination_type`, `targeting`, `special_ad_categories`, `promoted_object`, `call_to_action`, datas, nomes

**Atenção:**

- ⚠️ O targeting padrão é **ESTADOS UNIDOS** — sempre definir BR explicitamente.

---

## 7. Experimentos — testes A/B e estudos de lift

Medição científica: testes A/B (isolar uma variável) e Conversion Lift (incrementalidade real via grupo de controle).

### `ads_experiment_check_eligibility`

**🟢 Leitura**

Verifica se a conta ou entidades específicas são elegíveis para testes A/B e estudos de lift.

**Parâmetros:**

- `ad_account_id` (obrigatório; aceita prefixo `act_`)
- `ad_entity_ids` (opcional) — sem ele, avalia lift sobre as 10 maiores campanhas por gasto

### `ads_experiment_list_tests`

**🟢 Leitura**

Lista testes A/B e lift studies de uma conta/campanha/conjunto/anúncio (ativos, agendados ou encerrados).

**Parâmetros:**

- `ad_account_id` OU `ad_entity_id` (um dos dois)
- `study_type` (opcional) — `lift`, `split_test`, `creative_testing`, `all`
- `include_finished` (opcional; padrão true)

**Atenção:**

- ⚠️ Se houver estudo ativo, alterar orçamento/targeting/criativo pode invalidar os resultados — checar antes de mexer.

### `ads_experiment_abtest_get_test`

**🟢 Leitura**

Detalhes de um teste A/B: nome, status, datas, células e entidades. Não retorna métricas de gasto (usar `ads_get_ad_entities` com a janela `metrics_window_*` retornada).

**Parâmetros:**

- `study_id` (obrigatório)

### `ads_experiment_lift_get_test`

**🟢 Leitura**

Resultados de lift study: conversões incrementais, confiança, custo por conversão incremental e ROAS incremental.

**Parâmetros:**

- `study_id` (obrigatório)

### `ads_experiment_abtest_create_test`

**🟠 Escrita (criação)**

Cria teste A/B em nível campanha, conjunto ou criativo (auto-detectado pelas entidades). Mínimo de 2 células.

**Parâmetros:**

- `ad_account_id`, `cells` (obrigatórios) — cells: `[{name, ad_entity_ids[]}]`
- `test_name` (opcional)
- `end_time` (opcional, AAAA-MM-DD; padrão 7 dias)
- `primary_kpi` (opcional; padrão `cost_per_result`) + `secondary_kpis[]`
- `budget_percentage` (testes de criativo; padrão 20%)
- `dry_run` (opcional) — `true` valida e mostra o plano sem criar

### `ads_experiment_abtest_update_test`

**🟠 Escrita (alteração)**

Edita ou encerra teste A/B: `update` (nome/datas), `end_test_now` (encerra preservando resultados) ou `cancel` (cancela sem preservar).

**Parâmetros:**

- `study_id`, `action` (obrigatórios)
- `test_name`, `start_time`, `end_time` (só com `action=update`; `start` só antes de iniciar)
- `reason` (opcional, para cancel)

### `ads_experiment_lift_create_test`

**🟠 Escrita (criação)**

Cria Conversion Lift study com defaults automáticos (objetivos Purchase/Subscribe/Lead por canal, célula única na conta). O estudo **INICIA IMEDIATAMENTE** após criação.

**Parâmetros:**

- `ad_account_id` (obrigatório)
- `study_name` (opcional)
- `start_time` (opcional; padrão +4h), `end_time` (opcional; padrão +30 dias; mínimo 5 dias)

---

## 8. Catálogo de produtos — leitura

Inspeção completa de catálogos, feeds, produtos, conjuntos de produtos e saúde de Dynamic Ads (Advantage+ Catalog).

### `ads_catalog_get_catalogs`

**🟢 Leitura**

Lista catálogos do usuário (até 100). Catálogos pertencem a um Business, não à conta de anúncio.

**Parâmetros:**

- `business_id` OU `ad_account_id` (resolve para o business dono; `business_id` tem precedência)
- `name` (opcional, filtro substring)
- `cursor` (opcional)
- `limit` (opcional; padrão 20, máx 100)

### `ads_catalog_get_details`

**🟢 Leitura**

Metadados do catálogo: nome, vertical, contagens de produtos e conjuntos, business e lista opcional de feeds. Alerta sobre conjuntos anunciados com itens bloqueados.

**Parâmetros:**

- `catalog_id` (obrigatório)
- `feed_limit` (opcional; omitir = sem feeds; padrão 25 com cursor)
- `feed_cursor` (opcional)

### `ads_catalog_get_data_sources`

**🟢 Leitura**

TODAS as fontes de dados do catálogo: feeds de arquivo/URL, Batch API, Graph API, integrações de parceiro (Shopify, WooCommerce, SFCC), smart pixel, crawling do site. Espelha a aba Fontes de Dados do Commerce Manager.

**Parâmetros:**

- `catalog_id` (obrigatório)
- `cursor` (opcional)
- `limit` (opcional; padrão 20, máx 100)

### `ads_catalog_get_diagnostics`

**🟢 Leitura**

Erros e avisos do catálogo: imagens quebradas, campos faltantes, violações de política, com severidade e canais afetados.

**Parâmetros:**

- `catalog_id` (obrigatório)
- `severity` (opcional) — `must_fix`, `should_fix`, `opportunity`
- `limit` (opcional; padrão 20, máx 100)

**Atenção:**

- ⚠️ Canais: `mini_shops` (Lojas FB/IG), `da` (Dynamic Ads), `marketplace`, `ig_shopping`, `whatsapp`.
- ⚠️ `number_of_affected_items` conta variações/SKUs, não produtos distintos.

### `ads_catalog_get_dynamic_ads_health`

**🟢 Leitura**

Checks pass/fail de saúde para Dynamic Ads: nível catálogo (pixel, match rate, produtos elegíveis) ou nível conjunto de produtos (qualidade criativa para slideshow/vídeo).

**Parâmetros:**

- `catalog_id` OU `product_set_id` (pelo menos um; `product_set_id` tem precedência)
- `checks` (opcional, lista de chaves específicas)
- `with_issue_only` (opcional; padrão true = só problemas)

### `ads_catalog_get_event_source_catalogs`

**🟢 Leitura**

Catálogos conectados a um pixel/CAPI/conjunto offline — base para entender o matching de eventos com produtos.

**Parâmetros:**

- `event_source_id` (obrigatório)

### `ads_catalog_get_feed_rules`

**🟢 Leitura**

Regras de transformação aplicadas ao feed na ingestão: `mapping_rule`, `value_mapping_rule`, `letter_case_rule`, `fallback_rule`, `regex_replace_rule`.

**Parâmetros:**

- `feed_id` (obrigatório)
- `cursor` (opcional)
- `limit` (opcional; padrão 20, máx 100)

### `ads_catalog_get_product_details`

**🟢 Leitura**

Busca produto pelo ID numérico da Meta (FBID, 15–19 dígitos). Para SKU alfanumérico do lojista (`retailer_id`), usar `ads_catalog_search_product`.

**Parâmetros:**

- `product_id` (obrigatório, apenas numérico)

### `ads_catalog_get_product_feed_details`

**🟢 Leitura**

Configuração do feed: nome, agendas (replace = completa; update = incremental), URL fonte, fuso, contagem de produtos e status/erros do último upload.

**Parâmetros:**

- `feed_id` (obrigatório)

### `ads_catalog_get_product_feed_upload_sessions`

**🟢 Leitura**

Histórico de uploads do feed: status, tempos, itens detectados/persistidos/inválidos/excluídos e contagens de erros/avisos.

**Parâmetros:**

- `product_feed_id` (obrigatório)
- `cursor` (opcional)
- `limit` (opcional; padrão 20, máx 100)

### `ads_catalog_get_product_sets`

**🟢 Leitura**

Lista conjuntos de produtos de um catálogo com regra de filtro, contagem, tipo e visibilidade.

**Parâmetros:**

- `catalog_id` (obrigatório)
- `name` (opcional)
- `cursor` (opcional)
- `limit` (opcional; padrão 20, máx 100)

### `ads_catalog_get_product_set_details`

**🟢 Leitura**

Detalhes de um conjunto de produtos específico.

**Parâmetros:**

- `product_set_id` (obrigatório, obtido em chamada anterior — nunca inventar)

### `ads_catalog_get_product_set_products`

**🟢 Leitura**

Produtos dentro de um conjunto, com filtros adicionais (combinados por E com a regra do conjunto).

**Parâmetros:**

- `product_set_id` (obrigatório)
- `availability` — `in stock`, `out of stock`, `preorder`, `available for order`, `discontinued`...
- `brand`, `category`, `product_type`, `condition` (`new`, `refurbished`, `used`)
- `price_min` / `price_max`
- `retailer_id` (SKU exato, sensível a maiúsculas)
- `fields` (opcional) — `product_id`, `retailer_id`, `name`, `availability`, `price` (padrão) + `description`, `url`, `sale_price`, `brand`, `image_url`, `visibility` etc.
- `cursor`, `limit` (padrão 20, máx 100)

### `ads_catalog_get_product_product_sets`

**🟢 Leitura**

Conjuntos que contêm determinado produto (para avaliar impacto em campanhas antes de mudanças).

**Parâmetros:**

- `product_id` (obrigatório)
- `name` (opcional)
- `cursor`, `limit` (padrão 20, máx 100)

### `ads_catalog_search_product`

**🟢 Leitura**

Busca/lista produtos com filtro estruturado JSON e retorna amostra + total de correspondências. É a ferramenta para busca por SKU e para pré-visualizar filtros antes de criar conjuntos.

**Parâmetros:**

- `catalog_id` (obrigatório)
- `filter` (obrigatório, JSON) — `{}` lista tudo; folha `{"campo":{"op":valor}}` combinável com `and`/`or`/`not`; ex.: `{"retailer_id":{"eq":"ABC-001"}}`
- `error_type` (opcional) — filtra produtos afetados por um diagnóstico (ex.: `PRODUCT_NOT_APPROVED`)
- `fields` (opcional), `cursor`, `limit` (padrão 20, máx 100)

---

## 9. Catálogo de produtos — escrita

Criação e alteração de catálogos, feeds, regras de feed, conjuntos de produtos e produtos, além da exclusão permanente de produto.

### `ads_catalog_create`

**🟠 Escrita (criação)**

Cria catálogo novo + upload de produtos em um passo (não cria catálogo vazio). Exige exatamente UMA fonte de dados: `feed_url`, `items` (Batch API) ou `feed_file_content` (arquivo base64 até ~10 MB).

**Parâmetros:**

- `business_id` (obrigatório — ID do Business Manager, não da conta de anúncio), `catalog_name` (obrigatório)
- `feed_url` + `feed_name` (+ `schedule` {interval: hourly/daily/weekly/monthly, hour, minute, timezone} + `feed_username`/`feed_password` p/ URL autenticada)
- `items` — `[{method: create/update/delete, data: {id, title, description, price, availability, image_link, link, brand, condition...}}]`
- `feed_file_content` + `feed_file_name` + `feed_file_type` (text/csv padrão, TSV, XML)
- `vertical` (opcional) — `commerce` (padrão), `vehicles`, `hotels`
- `update_only` (opcional)

**Atenção:**

- ⚠️ Meta recomenda 1 catálogo único por negócio: duplicados fragmentam sinal (até −14% ROAS, +18% CPA).

### `ads_catalog_create_product_feed`

**🟠 Escrita (criação)**

Cria feed (fonte de dados) sob catálogo existente, com ou sem agenda de busca automática.

**Parâmetros:**

- `catalog_id`, `name` (obrigatórios)
- `country` (padrão US), `default_currency` (padrão USD), `feed_type` (`products` padrão; `hotel`, `flight`, `destination`, `home_listing`, `vehicles`, `media_title`)
- `schedule` — `interval` (HOURLY/DAILY/WEEKLY/MONTHLY, obrigatório) + `url` (obrigatório), `hour`, `minute`, `day_of_week`, `day_of_month`, `interval_count` (1–255), `timezone`, `username`/`password` (SFTP)

### `ads_catalog_create_feed_rule`

**🟠 Escrita (criação)**

Cria regra de transformação no feed (mapear coluna, traduzir valores, caixa de letras, valor padrão, regex).

**Parâmetros:**

- `product_feed_id`, `attribute`, `rule_type` (obrigatórios; imutáveis após criação)
- `rule_type` — `mapping_rule`, `value_mapping_rule`, `letter_case_rule` (`to_upper`/`to_lower`/`capitalize_all`/`capitalize_first`), `fallback_rule`, `regex_replace_rule`
- `params` (opcional, JSON) — `map_from`, `type`, `user_default_value`, `dependent_field_name`/`value`, `map_mode` (COPY/MOVE)

**Atenção:**

- ⚠️ Combinação feed + tipo + atributo deve ser única.

### `ads_catalog_create_product_feed_upload_session`

**🟠 Escrita (criação)**

Força atualização imediata do feed a partir da URL configurada (sem esperar a agenda). Só funciona em feeds com URL remota.

**Parâmetros:**

- `product_feed_id` (obrigatório)

### `ads_catalog_create_product_set`

**🟠 Escrita (criação)**

Cria conjunto dinâmico de produtos a partir de regra de filtro (mesma sintaxe do `search_product`).

**Parâmetros:**

- `catalog_id`, `title`, `filter` (obrigatórios)
- `retailer_id` (opcional)

**Atenção:**

- ⚠️ Fluxo recomendado: preview do filtro com `ads_catalog_search_product` antes de criar.

### `ads_catalog_update_catalog`

**🟠 Escrita (alteração)**

Renomeia um catálogo.

**Parâmetros:**

- `catalog_id` + `name`

**Atenção:**

- ⚠️ Excluir catálogo NÃO é suportado pelo MCP.

### `ads_catalog_update_product`

**🟠 Escrita (alteração)**

Atualiza campos de um produto: nome, descrição, URL, imagem, marca, disponibilidade, condição, preço e visibilidade. `visibility=hidden` é a forma REVERSÍVEL de tirar um produto do ar sem excluir.

**Parâmetros:**

- `product_id` (obrigatório) + ao menos 1 campo
- `availability` — `in stock`, `out of stock`, `preorder`, `available for order`, `discontinued`
- `condition` — `new`, `refurbished`, `used`, `used_like_new`, `used_good`, `used_fair`, `cpo`, `open_box_new`
- `price` / `sale_price` — string decimal em unidades maiores ("19.99", NÃO centavos) + currency ISO-4217 obrigatória junto
- `visibility` — `published` ou `hidden`
- `name`, `description`, `url`, `image_url`, `brand`

### `ads_catalog_update_product_feed`

**🟠 Escrita (alteração)**

Edita configurações do feed: nome, parsing (`delimiter`, `encoding`, `quoted_fields_mode`), moeda padrão, agendas replace/update (independentes) e pausa de agendas (`clear_*_schedule`) sem excluir o feed.

**Parâmetros:**

- `product_feed_id` (obrigatório) + campos desejados
- `replace_schedule` / `update_schedule` — `interval` obrigatório; `url` opcional (mantém a atual)
- `clear_replace_schedule` / `clear_update_schedule` (bool) — excludentes com as agendas correspondentes

### `ads_catalog_update_product_set`

**🟠 Escrita (alteração)**

Edita conjunto de produtos: nome, regra de filtro (define a composição), visibilidade, `retailer_id`, `parent_id`.

**Parâmetros:**

- `product_set_id` (obrigatório) + ao menos 1 campo
- `filter` (JSON, máx 500 KiB), `name`, `visibility` (`visible`/`hidden`), `retailer_id`, `parent_id`

**Atenção:**

- ⚠️ Excluir conjunto NÃO é suportado pelo MCP. Membros individuais não são adicionados/removidos diretamente — só via regra.

### `ads_catalog_delete_product`

**🔴 Exclusão (destrutiva)**

Exclui produto PERMANENTEMENTE do catálogo (sem desfazer via API). O produto some imediatamente de anúncios e superfícies Meta.

**Parâmetros:**

- `product_id` (obrigatório)

---

## Boas práticas de rate limit

- Espaçar chamadas; nunca disparar rajadas de consultas idênticas.
- Preferir `date_preset` e `time_increment` (séries temporais em uma única chamada) a varreduras dia a dia.
- Usar apenas cursores de paginação reais (`next_cursor`) — cursor inventado gera erro e gasta cota.
- Validar campos com `ads_get_field_context` antes de montar consultas grandes.
- Verificar `is_queryable` / `is_ads_mcp_enabled` antes de operar em uma conta.
- Em erro de rate limit: aguardar e reduzir a frequência — insistir estende o bloqueio.

## Como citar e reutilizar

Referência sugerida:

> CALIMAN, Thiago. **Meta Ads MCP — Referência Técnica Completa**. AI PRO Revolution, versão 2026.07.2, 2026. Disponível em: [https://thiagocaliman.com.br/mcp-meta/](https://thiagocaliman.com.br/mcp-meta/).

Para softwares e gerenciadores bibliográficos compatíveis com GitHub, use o arquivo [`CITATION.cff`](CITATION.cff). A documentação é distribuída sob a licença [Creative Commons Attribution 4.0 International](LICENSE): reutilizações devem atribuir Thiago Caliman, a AI PRO Revolution e indicar o endereço do material original.

O conteúdo é independente e educacional. Meta, Facebook, Instagram e seus produtos são marcas de seus respectivos titulares. Este projeto não é patrocinado, endossado nem mantido pela Meta.

## Fontes

- [Meta for Developers — Connect AI Agents to Meta with MCP](https://developers.facebook.com/documentation/mcp)
- Schemas oficiais das 82 ferramentas do conector `mcp.facebook.com/ads`
- [Marketing API — Rate Limiting](https://developers.facebook.com/documentation/ads-commerce/marketing-api/overview/rate-limiting)
- [Model Context Protocol — especificação](https://modelcontextprotocol.io/)

---

*Documentação independente, sem afiliação com a Meta. Nomes de produtos pertencem aos seus respectivos donos.*

