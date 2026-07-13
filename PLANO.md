# PLANO.md — Transferência OP-RPG → Project Lunar

Plano de implementação para portar ao Project Lunar os padrões comprovados da engine
irmã **One Piece RPG** (`../one-piece-rpg`), com prioridade em **método de cache** e
**qualidade narrativa**. Todos os mecanismos são agnósticos de cenário; nenhum conteúdo
de One Piece é transferido.

O fio condutor: **nunca realimentar prosa crua do próprio narrador como histórico.** Isso é
ao mesmo tempo caro (é a maior fatia de tokens por turno) e nocivo à qualidade (ensina o
modelo a imitar os próprios tiques — auto-condicionamento em contexto). A solução ataca os
dois problemas com o mesmo movimento: crystals destilados cobrem o passado, e só a **cena
aberta** entra como prosa crua.

---

## Decisões tomadas

1. **Cache (FASE 2): adota o cloaking do OP-RPG.** Instruções vão dentro da primeira
   mensagem `user`, entre tags `<narrator-instructions>…</narrator-instructions>`, com
   `cache_control` nos blocos de conteúdo. Elimina a incerteza de como o CLIProxyAPI
   (OAuth, porta 8318) trata o parâmetro `system=`. O prefixo cloaked **é** o prefixo
   cacheado.
2. **Qualidade (FASE 3b): adota a arquitetura completa do OP-RPG** — gate `pre_emit_audit`
   na fonte (narrador migra para tool-call) **e** Auditor pós-hoc como rede final. Os dois
   em conjunto, como na engine irmã.

---

## Diagnóstico do estado atual

Achados que orientam o plano (confirmados por leitura direta do código):

- **Metade da arquitetura-alvo já existe no Lunar, desligada.** `narrator_engine.py` tem
  `build_system_prompt_parts()` (split estático/dinâmico, linhas 263-321) e
  `complete_single_call()` com `cache_control: ephemeral` (490-542) — ambos são **código
  morto**, porque `game_session._is_single_call_provider()` retorna `False` incondicional
  (game_session.py:691-698). O caminho vivo (`stream_narrative`, :415) manda **string
  monolítica sem cache**.
- **Redundância que consome ~90% dos tokens por turno.** Os mesmos eventos entram no prompt
  duas vezes: como prosa crua no `history_slice` (até 600 mensagens em janela de 1M —
  narrator_engine.py:361-374) **e** como JSON destilado nos crystals (bloco WORLD MEMORY).
  O `_dynamic_history_slice` (:376-413) desconhece o `_last_crystal_cursor`
  (memory_engine.py:224,297); nunca solta a prosa de cenas já crystallizadas.
- **Infra de crystals já funciona** — pirâmide SHORT→MEDIUM→LONG→MEMORY
  (memory_engine.py:50-77), schema fact-preserving (102-145), gatilho
  `AUTO_CRYSTALLIZE_THRESHOLD = 4` (:207). Só não substitui a prosa crua.
- **Combate a tics = anti-padrão pink-elephant.** As `_NARRATOR_RULES`
  (narrator_engine.py:149-152 EN / 181-184 PT) e o `opening_generator.py:38-42` nomeiam
  cada vício e dão **exemplos literais** dele ("AVOID the obsessive 'rule of three'"
  seguido de três exemplos de rule-of-three). É exatamente o que o experimento
  `narrator_determinism_prompt` do OP-RPG mediu: remover um anti-exemplo derrubou a
  fabricação de 50%→11%. Não há regex nem pós-processamento — é 100% proibição textual.
- **Narrador é texto puro**, sem tool schema (`stream_narrative`). Terreno limpo para o
  gate estruturado. O `_SINGLE_CALL_FORMAT` (:466-488) é um andaime JSON pronto e ocioso.
- **Transporte:** produção fala com CLIProxyAPI em `127.0.0.1:8318` (`.env:13`), o mesmo
  binário do OP-RPG, com seção `cloak` configurável (`proxy/cliproxyapi/config.example.yaml`
  :227-238). No caminho com proxy o `stream()` já cai para `acompletion(stream=False)`
  (llm_router.py:383) — **o streaming ao cliente hoje é ilusório**, o que barateia a
  migração do narrador para saída estruturada.
- **Sem guard de temperature por modelo.** `llm_router.py` sempre envia `temperature`
  (333/352/386/416) — dá 400 em Opus/Sonnet/Fable 5.
- **Config por-cenário:** `language` / `tone_instructions` / `lore_text` vivem na dataclass
  `Scenario` (scenario_store.py:18-30), não em `Campaign`. Story cards têm `created_at`
  (scenario_store.py:33-40) — chave estável para ordenação append-only.

---

## FASE 0 — Instrumentação e baseline
**Habilitador · risco nulo.** Sem medir, nenhuma fase seguinte é comprovável. O OP-RPG é
empírico por design.

**Mudanças**
- `llm_router.py`: capturar `usage` de cada resposta e expor
  `cache_read_input_tokens` / `cache_creation_input_tokens` / `input` / `output`
  (espelhar `_usage()` do OP-RPG em `proxy/client.py:47-57`). Logar por chamada com um
  `label` (narrator, detect_mode, journal, crystallize…).
- Somar tokens por **ação do jogador** (as ~8 chamadas) num contador de turno, exposto no
  devtools do frontend.
- Capturar 20-30 saídas reais do narrador num cenário fixo como **baseline de tics**:
  contagem de rule-of-three, recaps de NPC, métrica falsa, travessão.

**Validação / pronto quando:** existe um número "tokens/turno" e uma taxa de tics/resposta
_antes_ de qualquer mudança. É a régua das FASES 1-3.

---

## FASE 1 — Cortar a realimentação de prosa crua
**Maior retorno isolado · ataca token E tic.**

**Objetivo.** O narrador passa a ver: crystals (passado destilado, já no prompt) + **janela
curta de prosa só da cena aberta**. Elimina os ~90% de tokens do `stream_narrative` e o
auto-condicionamento (o modelo para de reler os próprios tics como se fossem cânone de
estilo).

**Diagnóstico (hoje).** `_dynamic_history_slice` (narrator_engine.py:376-413) corta por nº
de mensagens e budget de tokens, nunca pela fronteira de crystallização. O
`_last_crystal_cursor` (memory_engine.py:224,297) marca o último evento destilado — a
fronteira "cena fechada vs. aberta" que ninguém consome.

**Mudanças**
1. `_handle_narrative` (game_session.py:~1048): montar o `history` do narrador apenas com
   eventos `created_at > _last_crystal_cursor`, mais um **buffer de overlap** de 1 batch
   (não abrir buraco entre a última prosa e o último crystal).
2. `_dynamic_history_slice`: aceitar a janela já recortada; manter o piso de 4 mensagens
   (:411) como rede de coerência de curtíssimo prazo.
3. `_rebuild_history_from_events` (game_session.py:135-153): aplicar o mesmo corte por
   cursor no boot, para o comportamento pós-restart bater com o de sessão viva.
4. **Invariante de cobertura contígua:** nada sai da janela sem já estar num crystal. Se a
   crystallização assíncrona ainda não fechou o intervalo, manter a prosa até ela rodar.

**Risco/Mitigação.** Perda de detalhe fino de médio prazo. O schema de crystal já é
fact-preserving (memory_engine.py:102-145); ajustar o overlap (1-2 batches) e validar
contra o baseline. Reversível por flag.

**Validação / pronto quando:** tokens/turno do `stream_narrative` caem de ~133k para dezenas
de k; history_slice típico ≈ cena aberta (unidades de mensagens); rule-of-three/recap caem
mesmo antes da FASE 3; sem regressão de coerência no baseline.

---

## FASE 2 — Cache em 4 zonas (cloaking)
**Multiplica a economia da FASE 1.** Pré-requisito: FASE 1 (a janela de prosa precisa ficar
_fora_ do prefixo cacheado — cache é prefix-dependent).

**Objetivo.** Transformar o prefixo estático (instruções + catálogos) em cache read a partir
do 2º turno.

**Mudanças**
1. **Split de 2→4 zonas** (evoluir `build_system_prompt_parts`, :263):
   - **Zona 0 (cache):** papel + idioma + `TONE AND STYLE` + `_OPENING_CANON_HEADER` +
     `_build_narrator_rules`. Byte-idêntico por cenário.
   - **Zona 1 (cache):** catálogos near-static — subconjunto **estável** de story cards +
     crystals tier MEMORY (canon imutável). **Ordenar por `(created_at, id)`**
     (append-only, `StoryCard.created_at` scenario_store.py:33-40) para que um card novo só
     anexe e re-caceie o delta.
   - **Zona 2 (volátil, sem cache):** crystals recentes, inventário, NPCs, hints, grafo +
     **diretiva de idioma** (um prefixo cacheado serve PT e EN — padrão
     `language.py:46-47` / `_EN_DIRECTIVE` do OP-RPG).
   - **Zona 3 (dinâmica):** janela de prosa da cena aberta + `player_input`.
2. **Cloaking:** portar `build_content()` do OP-RPG (`proxy/client.py:110-142`) — instruções
   na 1ª mensagem `user` com tags, `cache_control` (`{"type":"ephemeral","ttl":"1h"}`) nos
   blocos das zonas 0 e 1, header `extended-cache-ttl-2025-04-11` (client.py:63-64) para
   sobreviver ao think-time.
3. **Emitir blocos com `cache_control` no caminho de streaming** (portar o padrão de
   `complete_single_call:525-540` para `stream_narrative`).
4. **Guard de temperature por modelo** no `llm_router.py`: portar `_NO_SAMPLING_MODELS` +
   `_accepts_temperature` (client.py:69-73); omitir `temperature` para Opus/Sonnet/Fable 5
   (hoje enviado em 333/352/386/416).

**Risco/Mitigação.** Byte-identidade frágil — qualquer reordenação ou timestamp no prefixo
estoura o cache. Render determinístico, catálogos ordenados por chave estável, telemetria da
FASE 0 confirmando `cache_read > 0` no turno 2. Se o proxy não repassar o header de TTL,
cair para cache efêmero de 5 min (GA, sem header).

**Validação / pronto quando:** `cache_read_input_tokens > 0` do 2º turno em diante; custo/turno
cai na proporção do prefixo (referência OP-RPG: 134k→34k); nenhum 400 de sampling; PT e EN
compartilham o mesmo prefixo cacheado.

---

## FASE 3a — Despink-elephant
**Quick win · risco baixo · retorno já medido no OP-RPG.** Independente das FASES 1-2 (pode
ir junto com a 0).

**Mudanças.** Reescrever as regras anti-tic de **proibição+exemplo** para **princípio
afirmativo curto**, removendo os anti-exemplos literais:
- `_NARRATOR_RULES` EN (narrator_engine.py:149-152) e PT-BR (:181-184): trocar "AVOID the
  obsessive 'rule of three' [três exemplos]" por afirmação de cadência variada (pares,
  frases longas, trio ocasional) — sem citar o vício. Recap → "NPCs reagem à intenção com
  uma pergunta, um gesto ou um silêncio". Métrica falsa → "observação sensorial
  qualitativa".
- `opening_generator.py:38-42`: mesmo tratamento.
- **Manter** as regras afirmativas que já funcionam: render do input em cena (:141/:173),
  nomes completos, coerência de inventário/MEMORY.

**Validação / pronto quando:** A/B cego contra o baseline da FASE 0; queda de tic/fabricação
ao _remover_ os anti-exemplos (expectativa pela medição do OP-RPG).

---

## FASE 3b — Gate `pre_emit_audit` + Auditor
**Refinamento · mais invasivo.** Como o streaming via proxy já é ilusório
(llm_router.py:383), o custo real é baixo. Adota a arquitetura completa do OP-RPG (as duas
camadas).

**Camada 1 — gate na fonte (Rota A).** Migrar o narrador para tool-call, estendendo o
`_SINGLE_CALL_FORMAT` (:466-488) com um campo `pre_emit_audit`: enum de compromissos
afirmativos preenchidos **antes** do `narrative_text` e **descartados no parse** (padrão
`agents.py:64-124` do OP-RPG; descarte em `agents.py:238-241`). Reativar o caminho
single-call (`_is_single_call_provider`, game_session.py:691-698). Autocondicionamento local:
o modelo reafirma as regras no ponto de geração.

**Camada 2 — Auditor pós-hoc (Rota B).** Gate LLM final sobre a prosa antes de revelar ao
jogador: default limpo, cirúrgico, best-effort com timeout (~90s no OP-RPG, `AUDIT_TIMEOUT_S`
em config), libera a prosa original se estourar. Referência: reveal-after-audit em
`runner.py:~2922` do OP-RPG.

**Nota empírica (OP-RPG).** Gates curam tics sintaticamente localizáveis (travessão,
glosa/contraste) mas são **inertes** em drift difuso de registro/fabricação — por isso 3a
(despink) tem retorno maior e vem antes. 3b é otimização de segunda ordem.

**Fora de escopo (recomendação, não código):** reescrever os `tone_instructions` dos seeds
(ex. bloco "REGRAS ABSOLUTAS" com "NÃO/NUNCA" do `scenario-sword-god.json`) de proibição
para style anchor afirmativo — é conteúdo do usuário.

---

## Ordem e dependências

```
FASE 0 (instrumentação)  ──►  habilita medir tudo
   │
   ├──► FASE 3a (despink)      ← quick win independente, pode ir junto com a 0
   │
   └──► FASE 1 (janela curta)  ← maior retorno; pré-requisito da 2
              │
              └──► FASE 2 (cache 4 zonas, cloaking)  ← multiplica a economia
                          │
                          └──► FASE 3b (gate pre_emit_audit + Auditor)  ← só depois
```

---

## Referência: padrões-fonte no OP-RPG (`../one-piece-rpg`)

- **Cache / cloaking:** `backend/app/proxy/client.py` — `build_content()` (110-142, 4 blocos
  cloaked, 2 breakpoints), `_CACHE_CONTROL` (63), `_CACHE_HEADERS` (64),
  `_NO_SAMPLING_MODELS` + `_accepts_temperature` (69-73), `_usage()` (47-57).
- **Split estável/volátil + append-only:** `backend/app/pipeline/director.py` — catálogo
  ordenado por `(created_at, id)` e stable-vs-dynamic (1473-1530).
- **Idioma no bloco volátil:** `backend/app/pipeline/language.py` — `output_directive()`
  (46-47), `_EN_DIRECTIVE` (35-43).
- **Gate `pre_emit_audit`:** `backend/app/pipeline/agents.py` — schema (64-124), descarte no
  parse (238-241).
- **Root-cause (prosa crua como histórico):** `backend/app/pipeline/runner.py` (60-64);
  reveal-after-audit (~2922).
- **Auditor / timeout:** `backend/app/config.py` (`AUDIT_TIMEOUT_S`).

> Estado: **FASE 0 concluída** (baseline em `docs/fase0_baseline.md`). Decisões travadas:
> cloaking (FASE 2) e gate+Auditor (FASE 3b). Próxima: FASE 3a (despink) ou FASE 1 (janela curta).

## FASE 0 — concluída

Deltas em relação ao plano original:
- **Metade da instrumentação já existia** (`_call_log`/`reset_call_log`/`get_call_summary`/
  `_log_call` no `llm_router.py`; `reset`+`summary` já chamados em `routes_game.py`). A FASE 0
  acrescentou: captura de `cache_read`/`cache_creation` (`_cache_tokens`, espelhando `_usage()`
  do OP-RPG), somatório no `get_call_summary()`, frame SSE `[USAGE]{json}` antes de `[DONE]`, e
  o readout de devtools no header (`api.js` `onUsage` → `store.js` `lastUsage` → `GameCanvas`).
- **Baseline de tics medido** (61 respostas): rule_of_three 22 (~0,36/resp), npc_recaps 16
  (~0,26/resp, o mais estrutural), fake_metrics 19 (~0,31/resp), travessão 436 total → ~283
  interruptivos (~4,6/resp). Detalhe e metodologia em `docs/fase0_baseline.md`.
- **Limitação conhecida:** o `[USAGE]` cobre as chamadas síncronas (~90% dos tokens); as 6
  fire-and-forget correm com o resumo e ficam fora do snapshot (logam individualmente).
  Atribuição por turno das assíncronas fica para depois (mudança de comportamento).
- **Confirmado para a FASE 3a:** os anti-tics do commit `b718ba5` são pink-elephant e vazam
  conteúdo One Piece-específico no prompt do engine agnóstico.
