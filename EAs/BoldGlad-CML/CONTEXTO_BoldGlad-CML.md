# Contexto do Projeto BoldGlad-CML

Este arquivo serve como briefing inicial para continuidade do desenvolvimento do EA BoldGlad-CML em sessões futuras, especialmente quando utilizando o Claude Code.

---

## Como usar este arquivo

Ao iniciar uma nova sessão com o Claude Code nesta pasta, peça para ele:

> "Leia os arquivos CONTEXTO_BoldGlad-CML.md e BoldGlad-CML_documentacao.md para entender o projeto antes de começarmos."

Isso dá ao Claude contexto completo sobre o que já foi feito e o que está em andamento.

---

## Sobre o projeto

**Nome:** EA BoldGlad-CML
**Plataforma:** ProfitUltra da Nelogica, linguagem NTSL
**Modo:** Tick a Tick
**Ativo de referência:** WIN (mini-índice)

**Arquivos do projeto:**

- `BoldGlad-CML.txt` — código atual do EA em NTSL
- `BoldGlad-CML_documentacao.md` — documentação descritiva completa da estratégia (ler primeiro para entender o EA)
- `CONTEXTO_BoldGlad-CML.md` — este arquivo (briefing para continuidade)
- `../../docs/ManualNTSL.pdf` — manual de referência da linguagem NTSL (consultar quando tiver dúvidas de sintaxe)

---

## Preferência de comunicação

O autor do projeto prefere conversas **informais e focadas em comportamento**, não em sintaxe da linguagem. Quando explicar coisas ou propor mudanças, descreva em termos do que o robô faz (ou deixa de fazer) no mercado, não em termos de código.

Exemplo de como explicar:
- ✅ "Quando o preço atinge 200 pontos de lucro, o stop sobe um degrau"
- ❌ "A variável vStopAtual é atualizada no bloco condicional quando pLucroAberto >= pDistAcionaBE"

---

## Origem do CML (fork do BoldGlad-CM)

O BoldGlad-CML foi criado em 04/06/2026 como cópia exata do BoldGlad-CM, com **três mudanças** motivadas pelo pregão de 03/06/2026 (queda contínua, CM perdeu R$4.450 operando contra a tendência e por excesso de entradas; reinícios do EA ainda desarmaram a trava de perda diária):

1. **Filtro de tendência dominante** (`pUsarFiltroTendencia`, `pPeriodoTendencia(50)`): só compra com preço acima da média de tendência e vende abaixo. Mata as entradas contra a tendência do dia.
2. **Menos entradas:** `pInclinacaoMediaRapida` e `pTradeVela3` agora vêm **desligados** por padrão, e há **pausa de `pVelasPausaAposSaida` velas (default 3) após cada saída** de posição.
3. **Trava-dia à prova de reinício** (`pResetarPlacarAoReiniciar = False`): o contador de perda do dia não zera ao reabilitar o EA no meio do pregão.

Numa **segunda rodada (04/06/2026)**, mais duas rotas de entrada (decisões do trader via perguntas guiadas — média de 50 sempre prevalece sobre ambas):

4. **Entrada de abertura (P1)** (`pTradeAbertura`, default True): definida a 1ª vela do pregão (09:00), entra **a mercado** no rompimento do fundo (vendido) / topo (comprado) dela, **uma vez por dia**, só a favor da média de 50. `pHoraInicio` mudado para **0900**.
5. **Reentrada após lucro (P2)** (`pReentradaAposLucro`, default True): ao fechar um trade no lucro (qualquer > 0), repete na **mesma direção** no rompimento da vela que fechou, **furando a pausa**, exigindo viés (M1×M2) **e** média de 50 a favor. **Encadeia enquanto vencer**; perde → volta ao normal.

Para operar idêntico ao CM: `pUsarFiltroTendencia=False`, `pInclinacaoMediaRapida=True`, `pTradeVela3=True`, `pVelasPausaAposSaida=1`, `pResetarPlacarAoReiniciar=True`, `pTradeAbertura=False`, `pReentradaAposLucro=False`, `pHoraInicio=0905`.

**Status:** aguarda primeira verificação de sintaxe no editor do Profit e validação em pregão real. Nenhuma das mudanças foi testada em produção ainda. **Commit pendente de autorização expressa do trader.**

---

## Estado atual do EA

O EA opera da seguinte forma:

- **Entrada por ordem stop** no rompimento da máxima (compra) / mínima (venda) da vela anterior + offset.
- **Filtro de tendência dominante (CML)** (`pUsarFiltroTendencia`, default ligado; média `pPeriodoTendencia`, default 50): o viés só vira ordem a favor da média de tendência — compra com `Close` acima dela, venda com `Close` abaixo. Plotado no gráfico (Plot5).
- **Pausa após saída (CML)** (`pVelasPausaAposSaida`, default 3): após qualquer saída de posição, o EA não arma nem envia entradas por N velas. `1` reproduz o comportamento antigo do CM. **Não vale para a reentrada após lucro (P2).**
- **Entrada de abertura (P1 - CML)** (`pTradeAbertura`, default ligado): no rompimento do range da 1ª vela do pregão (09:00), entra a mercado, uma vez por dia, só a favor da média de 50. É rompimento puro — não passa por viés nem vela anterior alinhada.
- **Reentrada após lucro (P2 - CML)** (`pReentradaAposLucro`, default ligado): trade fechou no lucro → rearma na mesma direção no rompimento da vela que fechou, furando a pausa, exigindo viés (M1×M2) + média de 50 a favor. Encadeia enquanto vencer; perde → volta ao normal.
- **Antecipação de viés por inclinação forte da M1** (parâmetro `pInclinacaoMediaRapida`, **default DESLIGADO no CML**, limite `pValorInclinacaoMediaRapida = 0.04%` do preço — ~71 pts no WIN): quando ligado e a M1 cai/sobe forte entre vela 2 e vela 1, o viés é antecipado antes do cruzamento real. Em conflito com o viés regular, a inclinação ganha.
- **Antecipação de viés por padrão de 3 velas** (parâmetro `pTradeVela3`, **default DESLIGADO no CML**): quando ligado, bear-bull-bull aciona compra; bull-bear-bear aciona venda. Priorização das 3 rotas: inclinação forte > padrão V3 > regular (M1 vs M2). O filtro de tendência é um AND final aplicado por cima de qualquer rota.
- **Fallback a mercado quando preço passou do gatilho** (sempre ativo, sem parâmetro): se na hora do envio o `Close` já está em cima ou acima do `vGatilhoCompra` (ou em cima/abaixo do `vGatilhoVenda`), o EA manda `BuyAtMarket`/`SellShortAtMarket` em vez de stop. Disparado uma única vez por vela (flags `bMarketComprou/VendeuEnviado` resetam a cada nova vela).
- **Filtro de vela anterior alinhada** ao viés na primeira armação (vela de alta para compra, vela de baixa para venda).
- **Rearme contínuo** do gatilho stop a cada nova vela enquanto o viés se mantiver.
- **Sem reversão** por cruzamento contrário — posição aberta segue sob gestão normal mesmo com inversão de viés.
- **Faixa de consolidação** com:
  - Buffer de ampliação (`pBufferFaixa`, default 100) aplicado ao topo e fundo após a varredura.
  - **Dois gatilhos:** (a) dois cruzamentos reais com distância ≤ `pDistMaxCruz`; (b) proximidade das médias na vela 1 — `Abs(M1−M2) ≤ Close × pPercProximidadeMedias / 100` (default 0,005%). Ambos disparam a mesma lógica.
  - Reconstrução da faixa quando um novo evento interno tem distância ≤ `pDistMaxCruz`.
  - Manutenção da faixa quando o novo evento interno tem distância > `pDistMaxCruz`.
  - Defesa por data (cruzamentos e faixas de pregões anteriores são descartados automaticamente).
- **Gestão de posição:**
  - Stop fixo e alvo fixo, emitidos a cada tick com qty = `Abs(PositionQty)` (defensivo contra ajuste manual da posição).
  - Breakeven gradual em steps.
  - Trailing stop por mínima/máxima da vela anterior, acionado a partir de `pDistAcionaTrailing` (default 150 pts).
- **Controles** de horário, meta diária, perda máxima diária (em pontos, com conversão de R$ via `pValorPontoRS`, **à prova de reinício no CML** via `pResetarPlacarAoReiniciar = False`) e **trailing de proteção do lucro acumulado** (`pLucroRef` / `vLucroAcumRef`, default 1100 pts — arma após atingir esse valor; bloqueia se cair para 70% do pico, ou seja perda de 30%).
- **Logs de diagnóstico** (`BG-CML-DIAG:`) nos blocos 13 (armar gatilho) e 14 (envio de ordens) para tornar visível o comportamento a cada nova vela.

> **Nota:** havia uma feature de **redução parcial de contratos** (fechar metade da posição a mercado ao bater +150 pts) que foi REMOVIDA em 15/05/2026 — o engine do Profit não aceitava de forma confiável a ordem parcial enquanto a OCO de cobertura estava ativa. Detalhes na seção "Redução parcial — feature removida" abaixo.

Ver `BoldGlad-CML_documentacao.md` para descrição completa.

---

## Situação em aberto no momento

CML recém-criado (04/06/2026). Pendências de validação:

1. **Verificar sintaxe** no editor de estratégias do Profit (clicar "Verificar Sintaxe"). Mudanças não triviais foram feitas — round-trip necessário.
2. **Filtro de tendência:** confirmar em pregão real que, num dia de tendência clara, o EA só opera a favor (compras em dia de alta, vendas em dia de baixa) e que a linha da média de tendência (Plot5) está coerente. Avaliar se `pPeriodoTendencia = 50` é o melhor período ou se um valor maior/menor filtra melhor.
3. **Pausa após saída:** observar se `pVelasPausaAposSaida = 3` reduz o overtrading sem perder boas reentradas. Ajustar.
4. **Trava-dia à prova de reinício:** reabilitar o EA depois de um prejuízo no dia e confirmar que ele volta já travado (não opera).
5. **Entrada de abertura (P1):** confirmar que o EA grava o topo/fundo da 1ª vela (09:00) — log `Range da 1a vela do dia capturado` — e entra a mercado no rompimento, só a favor da média de 50. Validar que dispara no máximo uma vez por dia.
6. **Reentrada após lucro (P2):** após um trade vencedor, confirmar pelos logs `Trade VENCEDOR ...` e `REENTRADA ... armada` que ele rearma na mesma direção sem esperar a pausa, e que encadeia em sequência de vitórias / para na primeira perda. Validar especialmente a detecção de lucro na saída (aproximação por `vUltimoLucroAberto`).

**Comparação sugerida:** rodar CM e CML em paralelo (ou no replay do dia 03/06/2026) para medir a diferença de resultado.

**Pontos a observar em produção:**
- P1 vigia o rompimento da 1ª vela o dia todo até disparar (não há janela de corte). Se quiser limitar a abertura aos primeiros X minutos, é um ajuste futuro.
- A detecção de "trade vencedor" do P2 usa o lucro aberto do último tick com posição — validar que casa com o resultado realizado nos casos de stop/alvo/trailing.

---

## Redução parcial — feature REMOVIDA em 15/05/2026

A feature de **redução parcial de contratos** foi removida do EA após cinco tentativas distintas de fazê-la funcionar em produção. O engine de automação do Profit (modo "Quando condição for satisfeita" confirmado) não aceitou de forma confiável uma ordem a mercado parcial enquanto havia OCO de cobertura ativa.

**Tentativas que falharam:**

| # | Abordagem | Resultado |
|---|---|---|
| 1 | `SellToCoverAtMarket(qty)` / `BuyToCoverAtMarket(qty)` direto | Absorvido pela OCO de cover orders |
| 2 | Idem + `CancelPendingOrders` antes + skip stop/alvo no tick da redução | OCO foi cancelada e recriada, mas o mercado parcial não executou |
| 3 | Atrelar qty da proteção a `Abs(PositionQty)` (defensivo) | Funcionou pra corrigir o sintoma residual de contratos descobertos, mas a redução continuou não executando |
| 4 | Trocar `*ToCoverAtMarket` por `SellShortAtMarket` / `BuyAtMarket` (auto-cover do manual 11.11) | Sem `CancelPendingOrders` também falhou |
| 5 | Variável `nBarraReducao` — skip da emissão de stop/alvo durante o **candle inteiro** da redução | Não validado — feature removida antes |

**Mantido após a remoção (ganho permanente da tentativa 3):**

- As ordens de cobertura `SellToCoverStop/Limit` e `BuyToCoverStop/Limit` são emitidas com `Abs(PositionQty)` (tamanho real da posição no broker), não com `pQtdContratos`. Isso cobre o cenário de ajuste manual da posição via UI do Profit.

**Removidos do código:**

- Parâmetros `pQtdReducao` e `pDistReducaoPts`.
- Variáveis `bReducaoFeita` e `nBarraReducao`.
- Bloco inteiro da redução parcial em `IsBought` e `IsSold` do bloco 12.
- Logs `BG-CML: Reducao parcial COMPRA/VENDA - fechou ...`.

**Também removido nesta mesma rodada (e DEPOIS restaurado, ver seção abaixo):**

- Log `BG-CML-DIAG: Armacao bloqueada na nova barra | ...` do bloco 13. Foi removido por gerar ruído, mas restaurado em 21/05/2026.

**Onde está documentado:** seções 5, 8.6, 10.5, 10.6 do `BoldGlad-CML_documentacao.md`.

---

## Validação de timeframe e proximidade das médias na faixa — adicionados em 16/05/2026

### Validação de timeframe (pedidos 1 e 2 do usuário)

- Novo parâmetro `pTimeframe` (default 1 min).
- Novo bloco **2.5 VERIFICACAO DE TIMEFRAME** inserido entre os blocos 2 e 3.
- A cada execução, compara `BarDuration` (manual NTSL 16.2) com `pTimeframe`. Se diferente:
  - Bloco 11 (saídas) fecha a posição aberta (`ClosePosition + CancelPendingOrders`), mesma postura do lock diário.
  - Bloco 12 (gestão da posição) fica inerte (`if IsBought and bTimeframeOK` + simétrico em IsSold).
  - Blocos 13 (armar) e 14 (envio de ordens) ficam bloqueados via gate `bTimeframeOK`.
- Log emitido **uma única vez** por execução: `BG-CML: Timeframe incorreto - configurado para X min - grafico esta em Y min - EA bloqueado`. Controle via flag `bLogTimeframeFeito`.
- Variáveis novas: `bTimeframeOK : Boolean`, `bLogTimeframeFeito : Boolean`.

### Proximidade das médias como segundo gatilho da faixa (pedido 3 do usuário)

- Novo parâmetro `pPercProximidadeMedias` (default 0.005, equivalente a 0,005%).
- Cálculo no bloco 8: `vLimiteProximidade := Close * (pPercProximidadeMedias / 100)`. Para WIN em ~178.000 dá ~8,9 pts.
- Detecção: `bMediasProximas := (Abs(vM1_ant - vM2_ant) <= vLimiteProximidade) and (Date[1] = Date)`.
- Integração: `bCruzDetectado := bCruzouParaCima or bCruzouParaBaixo or bMediasProximas`. Ou seja, a proximidade alimenta o mesmo sinal já existente — o bloco 9 (faixa) trata um evento de proximidade exatamente como trataria um cruzamento real.
- Saída da faixa: **inalterada** — só por rompimento de preço (`Close > vTopoFaixa + pDistSaidaFaixa` ou `Close < vFundoFaixa − pDistSaidaFaixa`).
- Variáveis novas: `vLimiteProximidade : Float`, `bMediasProximas : Boolean`.

**Documentação atualizada:** seções 4.1, 4.2, 6.5, 7.2, 8.0, 8.9, 9 e 10.8 do `BoldGlad-CML_documentacao.md`.

**Próximo passo de validação:**

- **Timeframe:** habilitar o EA num gráfico de timeframe diferente de 1 min (ex.: 5 min) e confirmar que o log `BG-CML: Timeframe incorreto ...` aparece uma vez e o EA não opera. Em seguida, mudar pra 1 min e confirmar que volta a operar normalmente.
- **Proximidade:** observar em sessão real se candles amarelos aparecem em situações de médias coladas que **não** envolveriam dois cruzamentos próximos (o caso que a regra antiga não pegava).

### Correção aplicada em 21/05/2026 — `BarDuration()` exige parênteses

**Sintoma:** EA habilitado às 08:03 do dia 21/05 (pré-mercado). Pregão abriu às 09:00. Durante mais de uma hora de pregão **nenhum log apareceu no painel de eventos** — nem `BG-CML:`, nem `BG-CML-DIAG:`, nem mensagem de erro do Profit. EA ficou totalmente inerte, embora o status mostrasse "Executando".

**Causa:** uso de `BarDuration` sem parênteses no bloco 2.5 — `bTimeframeOK := (BarDuration = pTimeframe)` e `Round(BarDuration)` dentro do `ConsoleLog`. Manual NTSL seção 16.2 mostra `BarDuration()` com parênteses no exemplo oficial. Sem os parênteses, a expressão parece ter sido tratada como referência de função, fazendo a comparação/concatenação travar silenciosamente em runtime e abortar a execução do script.

**Detalhe importante:** a verificação de sintaxe do Profit **aceitou** o código sem parênteses. O erro foi puramente de runtime, e silencioso.

**Correção:** ambas as ocorrências passaram a usar `BarDuration()` com parênteses.

**Lição para o repositório:** funções da seção 16 do manual (Calendário) — `BarDuration`, `BarDurationF`, `CurrentDay`, `CurrentMonth`, `CurrentYear`, `CurrentTime`, `IsWeekend` — **sempre** exigem parênteses na chamada, mesmo sem argumentos. Difere de propriedades built-in como `CurrentBar`, `Date`, `Time`, `Close`, etc. que funcionam sem parênteses.

---

## Restauração do log `Armacao bloqueada` — 21/05/2026

**Sintoma observado:** após o desfazimento de uma faixa de consolidação às 09:37:52, o EA ficou 8 velas seguidas (09:38–09:45) sem nenhum log no painel — nem operacional `BG-CML:`, nem diagnóstico `BG-CML-DIAG:` — embora o gráfico mostrasse condições claras de entrada (candle de alta após desfazimento). Quando finalmente um novo evento de cruzamento foi detectado às 09:45, ele já tinha dist=248 pts (médias já separadas, oportunidade de entrada já perdida).

**Investigação:** sem o log `Armacao bloqueada na nova barra` (que tinha sido removido por ruído numa rodada anterior), não havia como saber qual das 6 condições gerais do bloco 13 (HasPos / HoraOK / LockDiario / EmConsol / BarraSaida / TimeframeOK) estava bloqueando a armação. O EA estava silencioso e a causa-raiz ficou inacessível.

**Reversão aplicada:** o log `BG-CML-DIAG: Armacao bloqueada na nova barra | ...` foi restaurado no bloco 13, agora também incluindo o campo `TimeframeOK` (que não existia na versão original). É uma linha por nova vela quando o EA estiver bloqueado — é ruidoso, mas é a única observabilidade desse caminho.

**Política daqui pra frente:** o `CLAUDE.md` foi atualizado com uma seção nova ("Protecting working code — REQUIRED before any `.txt` edit") que proíbe explicitamente a remoção de logs `BG-*-DIAG:` sem confirmação do trader. A regra existe para impedir que mudanças "cosméticas" destruam observabilidade essencial.

**Próximo passo de validação:** na próxima ocorrência do problema, o log `Armacao bloqueada` vai dizer **qual** condição estava em True/False quando o EA não armou, e aí vamos saber se a causa é a faixa, o timeframe, ou alguma outra coisa.

---

## Correção aplicada em 21/05/2026 (mesmo dia, depois do log restaurado) — comparação Integer-Float em `bTimeframeOK`

**Sintoma:** com o log `Armacao bloqueada` restaurado, ficou claro que o EA bloqueava todas as novas velas com `TimeframeOK=False`. Mas o gráfico era 1 min e `pTimeframe = 1` — tudo parecia certo.

**Causa:** `BarDuration()` retorna **Integer** (1). `pTimeframe(1)` cria um input que NTSL armazena como **Float** (1.0). A comparação `(BarDuration() = pTimeframe)` em NTSL **não auto-promove Integer para Float** — retornava `False` mesmo com os valores numericamente iguais.

**Tentativa intermediária (não funcionou):** envolver os dois lados com `Round()`:

```pascal
bTimeframeOK := (Round(BarDuration()) = Round(pTimeframe));
```

**Por que não funcionou:** o manual NTSL (capítulo 13) define `Round(Valor: Float): Integer` — a função espera Float. Quando `Round(BarDuration())` é chamado, está-se passando Integer para função que espera Float. NTSL falha silenciosamente em runtime e o `Round()` retorna 0 (ou lixo). Resultado: `0 = Round(pTimeframe)` → False → bloqueio persistiu.

**Correção definitiva:** trocar `BarDuration()` (Integer) por `BarDurationF()` (Float, manual seção 16.3). Comparação Float-Float funciona nativamente, sem precisar de Round nenhum.

```pascal
bTimeframeOK := (BarDurationF() = pTimeframe);
```

**Mudança no código:** duas linhas do bloco 2.5 (a comparação e a referência dentro do log). Nenhuma outra parte do código foi tocada. Comentários do bloco 2.5 atualizados com "ATENCAO 3" explicando a evolução das tentativas.

**Lição geral para NTSL — atualizada:**
1. **Não envolver chamadas em `Round()` por reflexo** — `Round()` em NTSL tem assinatura estrita (`Round(Valor: Float): Integer`) e falha silenciosamente se receber Integer. Antes de usar Round, confirmar no manual que a função interna retorna Float.
2. **Preferir variantes `*F` das funções de calendário** quando comparando com inputs Float: `BarDurationF()` em vez de `BarDuration()`, etc.
3. **Comparação Float-Float em NTSL funciona nativamente.** O problema não está em `=` ser estrito demais — está em forçar tipos incompatíveis para os dois lados.

---

## Trailing de proteção do lucro acumulado — adicionado em 22/05/2026

**Motivação do usuário:** a meta diária sozinha (`pMetaDiariaPts`, default 2000) encerra o EA cedo em dias bons. Quer um mecanismo que permita continuar operando após atingir um lucro de referência, e só pare quando o lucro acumulado cair muito do pico.

**Mudanças no código (bloco 4 — gestão diária):**

1. **Novo input:** `pLucroRef(1100.0)` — gatilho do trailing.
2. **Novas variáveis:**
   - `vLucroAcumRef : Float` — pico do lucro acumulado do dia.
   - `bLucroRefAtingido : Boolean` — flag de "trailing armado".
3. **Reset (bloco 1 e bloco 2 novo dia):** `vLucroAcumRef := 0; vLucroAcumRef := pLucroRef; bLucroRefAtingido := False;` (item 6 do requisito — duas linhas como pedido).
4. **Atualização do pico (bloco 4, em nova vela):** `if (bNovaBarra) and (vResultadoDia > vLucroAcumRef) then vLucroAcumRef := vResultadoDia`.
5. **Armar o trailing:** `if (not bLucroRefAtingido) and (vResultadoDia >= pLucroRef)` → marca `bLucroRefAtingido := True` e emite log uma vez.
6. **Lógica de `bLockDiario` reformulada:** três condições independentes (cada uma com seu log próprio):
   - Loss diário (sempre ativo).
   - Trailing armado E `vResultadoDia ≤ vLucroAcumRef × 0.70` → bloqueia (perda de 30% do pico).
   - Trailing NÃO armado E `vResultadoDia ≥ pMetaDiariaPts` → bloqueia.

**Item 5 do requisito (priorização entre trailing e meta diária):** opção (A) — uma vez que o trailing arma, `pMetaDiariaPts` deixa de bloquear. Consequência: o lucro pode ultrapassar a meta diária naturalmente, e `vLucroAcumRef` continua subindo até o trailing disparar.

**Cenários esperados:**

| Cenário | O que acontece |
|---|---|
| Lucro chega a 500 e cai pra -1000 | Loss diário bloqueia em -1000. |
| Lucro chega a 800 e cai pra 200 | Nada — trailing não armou (não atingiu 1100). Continua operando. |
| Lucro chega a 2000 sem armar trailing | Meta diária bloqueia. |
| Lucro atinge 1100 → trailing armado. Sobe a 1500 (pico) → cai pra 1050 | Trailing bloqueia (1050 ≤ 1500 × 0.70). |
| Lucro atinge 1100 → armado. Sobe a 3000 (pico) | Meta diária IGNORADA. Continua operando. Bloqueia se cair pra 2100 (3000 × 0.70). |

**Escopo do diff:** blocos `input`, `var`, 1 (reset), 2 (novo dia) e 4 (gestão diária). **Não tocou** em nenhum outro bloco (2.5 timeframe, 9 faixa, 11–14 entradas/posição etc.). Todos os logs `BG-CML-DIAG:` existentes preservados.

**Documentação:** seção 6.2 reescrita, seção 6.4 atualizada, seção 7.2 com novos logs, seção 8.8 com `pLucroRef`, seção 10.9 do changelog.

---

## Feature de timeframe REMOVIDA em 22/05/2026

A validação de timeframe (`pTimeframe` + `bTimeframeOK` + bloco 2.5 + gates nos blocos 11-14) foi adicionada em 16/05/2026 mas **nunca funcionou em produção**, mesmo após múltiplas tentativas de correção:

| Tentativa | Resultado |
|---|---|
| `bTimeframeOK := (BarDuration = pTimeframe)` (sem parênteses) | EA travava silenciosamente, nenhum log no painel |
| `bTimeframeOK := (BarDuration() = pTimeframe)` (Integer = Float) | TimeframeOK=False em chart 1Min |
| `bTimeframeOK := (Round(BarDuration()) = Round(pTimeframe))` | Round() esperava Float, fail silencioso |
| `bTimeframeOK := (BarDurationF() = pTimeframe)` (Float = Float) | TimeframeOK=False em chart 1Min |

A causa-raiz final não foi identificada — possivelmente precision Float, ou armazenamento Integer do input mesmo declarado com literal `1`, ou algum bug na função em runtime real (vs documentação).

**Decisão do trader (22/05/2026):** remover a feature inteira. EA volta a operar em qualquer timeframe sem aviso.

**Mudanças no código:**
- Removido `pTimeframe` do bloco input.
- Removidas variáveis `bTimeframeOK` e `bLogTimeframeFeito` do bloco var.
- Removida inicialização `bLogTimeframeFeito := False` no bloco 1 (reset).
- **Removido o bloco 2.5 inteiro** (verificação de timeframe).
- Removida a condição `if (HasPosition) and (not bTimeframeOK) then ClosePosition` do bloco 11.
- Removidos os gates `(bTimeframeOK)` dos `IsBought` e `IsSold` no bloco 12.
- Removida a condição `(not bTimeframeOK)` e o campo `TimeframeOK=...` do log no bloco 13.
- Removida a condição `(bTimeframeOK)` do bloco 14.

**Preservado:** todos os outros gotchas aprendidos no caminho ficam documentados no `CLAUDE.md` (parênteses obrigatórios em funções da seção 16, comparação Integer-Float falha silenciosamente, `Round()` exige Float).

**Documentação atualizada:** seção 1 (visão geral), seção 6.5 (removida), seção 7.2 (logs), seção 8.0 (parâmetros), seção 9 (fluxo), seção 10.8 do changelog (marcada como removida).

---

## Inclinação forte da M1 antecipa viés — adicionado em 22/05/2026

**Motivação do usuário:** o cruzamento real de M1 com M2 às vezes vem tarde. Antecipar o viés quando a inclinação da M1 está forte permite operar a reversão antes do cruzamento se consumar.

**Inputs novos:**
- `pInclinacaoMediaRapida(True)` — Boolean, liga/desliga.
- `pValorInclinacaoMediaRapida(0.04)` — percentual do preço (0,04% = ~71 pts no WIN em ~178.000).

**Variáveis novas:**
- `vLimiteInclinacao : Float` — limite em pontos (`Close × pct / 100`).
- `vInclinacaoM1 : Float` — `vM1_ant − vM1_ant2` (variação da M1 entre vela 2 e vela 1, em pontos com sinal).
- `bInclinacaoAlta : Boolean` — feature ligada AND inclinação > limite AND data ok.
- `bInclinacaoBaixa : Boolean` — feature ligada AND inclinação < −limite AND data ok.

**Mudança no bloco 8 (sinais das médias):**

```
bSinalCompra := bInclinacaoAlta OR ((vM1 > vM2) AND NOT bInclinacaoBaixa)
bSinalVenda  := bInclinacaoBaixa OR ((vM1 < vM2) AND NOT bInclinacaoAlta)
```

A inclinação **ganha em caso de conflito** com o viés regular. Cenário típico: M1 ainda acima de M2 (compra regular) mas caindo forte entre vela 2 e vela 1 → `bSinalVenda = True`, EA arma venda antecipada.

**Log novo:** `BG-CML-DIAG: Inclinacao M1 ALTA/BAIXA forte (X pts, limite Y pts) - antecipa vies COMPRA/VENDA` — uma linha por nova vela quando a inclinação está ativa.

**Não foi tocado:** vela anterior alinhada continua exigida na primeira armação. Gatilho stop no rompimento da máxima/mínima continua. Faixa de consolidação, breakeven, trailing, gestão diária — todos intactos.

**Documentação:** seção 2.1 atualizada com a priorização, nova seção 2.1.1 explicando a inclinação, seções 3.1 e 3.2 atualizadas para incluir antecipação, seção 7.2 com novos logs, seção 8.1 com os parâmetros novos, seção 10.9 do changelog.

---

## Padrão de 3 velas como gatilho alternativo de viés — adicionado em 22/05/2026

**Motivação do usuário:** adicionar uma rota de entrada baseada em padrão clássico de reversão (bear-bull-bull para compra, bull-bear-bear para venda). Gatilho de entrada continua o mesmo (rompimento da máxima/mínima da vela 1 + offset), só muda a condição de viés.

**Input novo:**
- `pTradeVela3(True)` — Boolean. Liga/desliga o gatilho.

**Variáveis novas:**
- `bPadraoCompraV3 : Boolean` — vela 3 negativa ou doji + vela 2 positiva + vela 1 positiva, todas do mesmo dia, com feature ligada.
- `bPadraoVendaV3 : Boolean` — espelhado (vela 3 positiva ou doji).

**Mudança no bloco 8 (sinais das médias):**

Cálculo dos padrões (com defesa de data e gate da feature):

```pascal
bPadraoCompraV3 := (pTradeVela3)
                  and (Close[3] < Open[3])
                  and (Close[2] > Open[2])
                  and (Close[1] > Open[1])
                  and (Date[1] = Date) and (Date[2] = Date) and (Date[3] = Date);
{ bPadraoVendaV3: espelhado }
```

Reformulação de `bSinalCompra` / `bSinalVenda` com **3 níveis de prioridade**:

```
inclinação forte (highest) > padrão V3 > regular M1 vs M2 (lowest)
```

```pascal
bSinalCompra := bInclinacaoAlta
              or (bPadraoCompraV3 and (not bInclinacaoBaixa))
              or ((vM1 > vM2) and (not bInclinacaoBaixa) and (not bPadraoVendaV3));

bSinalVenda  := bInclinacaoBaixa
              or (bPadraoVendaV3 and (not bInclinacaoAlta))
              or ((vM1 < vM2) and (not bInclinacaoAlta) and (not bPadraoCompraV3));
```

**Cenários:**

| Situação | Vence |
|---|---|
| Só M1 vs M2 sinaliza | regular |
| Padrão V3 compra + M1 < M2 | padrão V3 (antecipa) |
| Inclinação alta + padrão V3 venda | inclinação (mais imediata) |
| Padrão V3 + regular alinhados | ambos confirmam, sem conflito |

**Logs novos (uma linha por nova vela quando padrão presente):**

- `BG-CML-DIAG: Padrao V3 COMPRA detectado (Vela3 neg/doji, Vela2 pos, Vela1 pos) - gatilho @ H1+offset`
- `BG-CML-DIAG: Padrao V3 VENDA detectado (Vela3 pos/doji, Vela2 neg, Vela1 neg) - gatilho @ L1-offset`

**Não foi tocado:** vela anterior alinhada continua exigida na primeira armação (e o padrão V3 já implica isso). Gatilho stop no rompimento da máxima/mínima continua. Faixa de consolidação, breakeven, trailing-stop, gestão diária — todos intactos.

**Documentação:** nova seção 2.1.2 explicando o padrão V3, seções 3.1 e 3.2 atualizadas, seção 7.2 com novos logs, seção 8.1 com o parâmetro novo, seção 10.10 do changelog (numerada após inclinação).

---

## Fallback a mercado quando preço passou do gatilho — adicionado em 01/06/2026

**Sintoma observado (01/06/2026):** abertura do pregão com alta explosiva de ~1.600 pts em poucos minutos (de 174.700 a 176.300). O EA tentou enviar `BuyStop` repetidamente, sempre rejeitado pelo broker com mensagem `Não foi possível criar BuyStop - Sua ordem stop deve ter o preço acima do preço atual do mercado`:

| Hora | Gatilho | Preço Mercado | Resultado |
|---|---|---|---|
| 09:03:18 | 175.520 | 175.530 | Rejeitado |
| 09:04:31 | 175.615 | 175.615 | Rejeitado (empate) |
| 09:05:22 | 175.840 | **176.075** | Rejeitado (preço 235 pts acima) |

A cada nova vela o EA atualizava o gatilho para `Hi1 + offset`, mas o preço estava sempre vários pontos acima — o EA simplesmente não conseguiu acompanhar a velocidade do movimento. **Todo o rompimento foi perdido.**

**Mudança aplicada (escopo cirúrgico):**

- **Novas vars:** `bMarketComprouEnviado` e `bMarketVendeuEnviado` (Boolean).
- **Bloco 1, 2, 7:** inicialização/reset das duas flags (CurrentBar=1, novo dia, e a cada nova vela — esse último em bloco 7 dá nova chance de tentar mercado a cada vela).
- **Bloco 14 inteiramente refatorado:** antes de enviar ordem stop, checa:
  - Compra: `if Close >= vGatilhoCompra then BuyAtMarket else BuyStop`
  - Venda: `if Close <= vGatilhoVenda then SellShortAtMarket else SellShortStop`
- A flag `bMarketCompr/VendeuEnviado` impede disparar o mercado mais de uma vez por vela.

**Logs novos:**

- `BG-CML-DIAG: BuyAtMarket enviado (Close=X >= gatilho=Y) - rompimento intra-vela | Qtd=N | Hora ...`
- `BG-CML-DIAG: SellShortAtMarket enviado (Close=X <= gatilho=Y) - rompimento intra-vela | Qtd=N | Hora ...`

**Trade-off aceito pelo trader:** entrada a mercado num spike pega o preço pior (pode ser topo/fundo do movimento) e pode levar stop loss rápido se o movimento for falso. O trader escolheu explicitamente a alternativa B (fallback a mercado) ao invés da alternativa A (manter como está, aceitar perda de spikes).

**Não foi tocado:** hierarquia de viés (inclinação > padrão V3 > regular) permanece, vela anterior alinhada continua exigida, faixa de consolidação, gestão de posição, gestão diária com trailing — todos intactos. Os logs existentes `BuyStop enviado` / `SellShortStop enviado` continuam aparecendo quando o caminho normal é tomado.

**Documentação:** seção 2.3 reescrita, seção 7.2 com novos logs, seção 10.12 do changelog (numeração ajustada — antiga 10.12 lock diário virou 10.13).

---

## Pontos em revisão (aguardando aplicação)

### 1. Limpeza dos logs de diagnóstico

Depois que o comportamento dos blocos 13 e 14 estiver validado em produção, remover os logs `BG-CML-DIAG:` para deixar o console mais limpo. Manter apenas os logs operacionais (`BG-CML:`).

---

## Limitação importante conhecida da NTSL

Em algumas versões da plataforma, a função `FloatToString` não funciona corretamente em concatenações dentro de `ConsoleLog`. A solução é concatenar variáveis Float diretamente com o operador `+`, sem usar `FloatToString`. Exemplo:

```
ConsoleLog("Preco = " + vPreco);
```

E não:

```
ConsoleLog("Preco = " + FloatToString(vPreco, 0));  { pode gerar erro }
```

Isso está documentado no manual oficial (exemplo da função ConsoleLog na seção 19.3).

---

## Convenções de nomenclatura para fechamento de posição (NTSL)

Funções usadas no código que diferem do padrão "prefixo + sufixo":

- **Fechamento parcial a mercado:** `SellToCoverAtMarket(qty)` (long) e `BuyToCoverAtMarket(qty)` (short). Atenção: é `AtMarket`, com o "At".
- **Stops/alvos de proteção:** `SellToCoverStop`, `SellToCoverLimit`, `BuyToCoverStop`, `BuyToCoverLimit` — sem o "At".

A inconsistência (`Stop`/`Limit` sem "At", `Market` com "At") é uma peculiaridade da NTSL; manter como está.

---

## Convenções adotadas no código

- Parâmetros de entrada começam com `p` minúsculo (ex: `pPeriodoM1`).
- Variáveis globais começam com letra de tipo: `v` para Float, `b` para Boolean, `n` para Integer.
- Todos os logs operacionais são prefixados com `BG-CML:` para facilitar filtragem.
- Logs de diagnóstico (temporários) são prefixados com `BG-CML-DIAG:`.
- Comentários em português, em blocos de chaves `{ }`.
- Estrutura dividida em blocos numerados (1. RESET, 2. DETECÇÃO DE NOVA BARRA, etc).

---

## Referências úteis

- Manual NTSL: arquivo `../../docs/ManualNTSL.pdf`
- Documentação da plataforma: https://www.nelogica.com.br
- Função ConsoleLog: seção 19.3 do manual
- Função Media: seção 25.8 do manual
- Controle de fluxo (If, For, While): seções 5.1 a 5.4 do manual
