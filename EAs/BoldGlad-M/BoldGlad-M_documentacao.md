# EA BoldGlad-M — Documentação da Estratégia

**Plataforma:** ProfitUltra (NTSL) — Tick a Tick — ativo de referência **WIN** (mini-índice)
**Tag de log:** `BG-M:` (operacional)

---

## 1. Visão geral

O BoldGlad-M opera **tick a tick** (não espera o fechamento da vela) por uma **média exponencial de sinal (M3, default 21)**, com um **filtro de consolidação** formado por duas médias (M1=100 / M2=200). **Bidirecional.** As saídas são por **cruzamento da própria M3** contra a posição, **stop gain** e **stop loss** em pontos.

- **M1 (100) e M2 (200):** servem **apenas** para detectar consolidação (a faixa). Não geram entrada nem saída.
- **M3 (EMA, default 21):** é o **sinal de operação** — preço acima compra, preço abaixo vende.

---

## 2. Lógica de entrada e saída

### 2.1 Entrada — rompimento por ordem stop

Estando **zerado**, dentro do horário, sem trava-dia, **fora de consolidação** e **não na mesma vela de uma saída** (`CurrentBar <> nBarraSaida`), o robô arma **um** gatilho stop (só um lado por vez). Para **compra**, todas necessárias:

- **(a)** o **nível do rompimento está acima da M3** (`vGatilhoCompra > vM3`);
- **(b)** *(só na reentrada — ver 2.3)* o rompimento ocorre a **≥ `pDistMinReentrada`(100)** da M3;
- **(c)** **inclinação da M3 > `pInclinacao`(15)** — inclinação = `M3[1] − M3[2]` (pontos na última vela fechada, valor estável);
- **(d)** a **vela anterior é positiva** (`Close[1] > Open[1]`) e o preço **rompe a máxima dela + `pDeslocamento`(10)**.

→ arma `BuyStop` em `High[1] + pDeslocamento` (na reentrada, no mínimo `M3 + pDistMinReentrada`). A execução ocorre quando o preço alcança o nível (ordem stop — garante o preço, sem slippage de mercado).

**Venda** é o espelho: vela anterior **negativa**, **inclinação < −`pInclinacao`**, rompimento da **mínima − `pDeslocamento`** (e, na reentrada, ≤ `M3 − pDistMinReentrada`), nível **abaixo** da M3 → `SellShortStop`.

Como a vela anterior é positiva **ou** negativa, só um lado fica armado por vez (`nLadoArmado` cancela o pendente ao trocar de lado). O gatilho é re-armado/atualizado a cada tick e cancelado se as condições deixam de valer.

### 2.2 Saída (tick a tick)

Fecha a posição (qualquer tick) quando ocorre **o primeiro** de:

- **Cruzamento da M3 contra a posição:** comprado sai com preço **abaixo** da M3; vendido sai com preço **acima** da M3.
- **Stop gain:** lucro do trade **≥ `pStopGainPts`** (default 1000 pts).
- **Stop loss:** prejuízo do trade **≥ `pStopLossPts`** (default 1000 pts).
- **Fim de horário** (`pHoraFim`) e **trava-dia**.

Na prática, como a M3 fecha o trade assim que o preço a cruza, o **stop loss** funciona como **rede de segurança** (movimentos rápidos/gaps que passam direto), e o **stop gain** **limita o ganho** de um trade muito a favor (depois reentra na vela seguinte, se o preço seguir do mesmo lado da M3).

### 2.3 Reentrada (vela seguinte) e o piso de distância da M3

Ao sair, a vela da saída é gravada (`nBarraSaida`) e a entrada fica **bloqueada nessa vela** — só reentra **na vela seguinte**. Toda entrada usa o rompimento de 2.1.

**Piso de distância (só após saída pela M3).** Quando a saída foi **pelo cruzamento da M3** (`bFiltroDistReentrada`, armado no bloco 10; não vale para stop gain/loss/horário/trava), o nível do rompimento é **empurrado** para ficar a **≥ `pDistMinReentrada`(100)** da M3: `vGatilhoCompra = max(High[1]+pDeslocamento, M3+100)` (e o espelho na venda). Assim a reentrada só dispara se o rompimento ocorrer pelo menos 100 pts afastado da M3. Após stop gain/loss esse piso **não** atua — o rompimento é só `High[1]+pDeslocamento` (mais perto da M3).

O filtro desarma quando o gatilho dispara (posição aberta, bloco 12-A).

### 2.4 Seletividade

A entrada exige rompimento **+** acima/abaixo da M3 **+** M3 inclinada ≥ `pInclinacao` na última vela. É um filtro de **tendência**: em pregão de lado as condições raramente alinham, então o robô **opera pouco / fica de fora** — comportamento esperado. O **filtro de consolidação** (seção 4) reforça o bloqueio em range.

---

## 3. Gestão diária (em pontos)

Resultado do dia contabilizado **internamente, em pontos** (soma dos trades fechados + flutuante da posição aberta), sem depender de `DailyResult` — funciona em **replay e ao vivo**. Ao reabilitar o EA no meio do pregão, o acumulador é **semeado** com o resultado real do dia (`DailyResult` → pontos via `pValorPontoRS`).

Travas (qualquer uma fecha a posição e bloqueia até o próximo pregão), todas com log `BG-M: EA PAROU DE OPERAR - ...`:
- **Perda máxima** (`pLossMaxDiarioPts`).
- **Devolução do pico** (`pPercDevolucao`, default **20%**): após o lucro do dia atingir `pLucroRef`, a trava **arma**; depois, se o lucro **recuar `pPercDevolucao` do pico** (ex.: pico 4000 → 3200 = −20%), encerra. (Em pontos é o mesmo percentual que em R$.)
- **Meta de lucro** (`pMetaDiariaPts`, só enquanto a trava de devolução não armou).

---

## 4. Faixa de consolidação (M1/M2)

Controlada por **`pConsolicao` (default `True` — ligada)**. Forma faixas a partir de **cruzamentos próximos de M1 × M2** (ou proximidade entre elas) e **bloqueia entradas** enquanto o preço não se afastar da faixa. É a rotina herdada (buffer, reconstrução, defesa por data). Com `pConsolicao = False`, toda a rotina é ignorada e o robô opera só pela M3.

---

## 5. Parâmetros de entrada

| Parâmetro | Padrão | Descrição |
|---|---|---|
| `pPeriodoM1` | **100** | Média para **consolidação** (rápida da faixa) |
| `pPeriodoM2` | **200** | Média para **consolidação** (lenta da faixa) |
| `pPeriodoM3` | **21** | Média **exponencial** — **sinal** (preço acima compra / abaixo vende) |
| `pInclinacao` | **15.0** | Inclinação mínima da M3 na última vela (`M3[1]−M3[2]`, pts); compra exige `> +`, venda `< −` |
| `pDeslocamento` | **10.0** | Deslocamento (pts) além da máxima/mínima da vela anterior para o rompimento |
| `pDistMinReentrada` | **100.0** | Após saída **pela M3**, piso de distância do rompimento à M3 na reentrada (pts) |
| `pStopGainPts` | **1000.0** | Stop gain por trade (pts) |
| `pStopLossPts` | **1000.0** | Stop loss por trade (pts) |
| `pQtdContratos` | 10.0 | Contratos por trade |
| `pHoraInicio` | 0900 | Início (HHMM) |
| `pHoraFim` | 1730 | Término (HHMM) |
| `pMetaDiariaPts` | 2000.0 | Ganho máximo no dia (pts) |
| `pLossMaxDiarioPts` | 1000.0 | Perda máxima no dia (pts) |
| `pLucroRef` | 1100.0 | Lucro que **arma** a trava de devolução do dia |
| `pPercDevolucao` | **0.20** | Devolução máxima do pico do lucro do dia (0.20 = 20%) |
| `pValorPontoRS` | 0.20 | R$ por ponto (WIN) — usado só na semeadura do resultado do dia ao reativar |
| `pConsolicao` | **True** | Liga/desliga a faixa de consolidação (default ligada) |
| `pDistMaxCruz` | 50.0 | Distância máxima entre cruzamentos para formar faixa |
| `pBufferFaixa` | 100.0 | Buffer aplicado ao topo/fundo da faixa |
| `pDistSaidaFaixa` | 50.0 | Distância adicional para sair da faixa |
| `pVelasAntesPrimCruz` | 5 | Velas antes do 1º cruzamento na varredura |
| `pPercProximidadeMedias` | 0.005 | % do `Close` para tratar M1≈M2 como cruzamento |

---

## 6. Risco operacional

- **Whipsaw em lateral:** entrar/sair pela M3 tick a tick gera idas e vindas no mercado de lado. O filtro de consolidação (M1/M2) atenua, mas é lento (SMA 100/200).
- **Stop loss (1000) ≈ trava de perda do dia (`pLossMaxDiarioPts` 1000):** uma perda cheia já encerra o dia. Como a M3 normalmente fecha o trade antes dos −1000, isso tende a só acontecer em movimento rápido/gap.
- **Ganho/perda 1:1** nos stops (1000/1000): exige acerto > 50% para ser positivo.
- **Devolução de 20%** protege mais o lucro do que os 30% anteriores, porém encerra o dia mais cedo numa correção normal.

---

## 7. Origem / changelog

### 7.1 Fork do BoldGlad-CML1 → BoldGlad-M (12/06/2026)

Criado como cópia do BoldGlad-CML1 e reformulado num sistema de 2 médias (a mercado no fechamento).

### 7.2 Tentativa de reformulação (rompimento) revertida (18/06/2026)

Testou-se um sistema de EMAs 9/21 com reentrada por rompimento; em replay no dia lateral só operava uma vez por dia (armava o gatilho sem disparar). Descartado a pedido do trader.

### 7.3 Sistema tick a tick por M3 + consolidação (18/06/2026)

Reformulação atual. Decisões do trader:

1. **3 médias:** M1(100) e M2(200) só para consolidação; **M3 = EMA(21)** como sinal.
2. **Entrada tick a tick** seguindo a M3 (preço acima compra / abaixo vende) — não espera o fechamento.
3. **Saídas:** cruzamento da M3 contra a posição (fecha), **stop gain** e **stop loss** (1000/1000), horário e trava-dia. (A saída antiga pela média rápida foi removida.)
4. **Reentrada/reversão na vela seguinte** à saída; segue a M3 (recompra mesmo lado após stop gain; inverte após cruzamento).
5. **Trava de devolução do dia** configurável: `pPercDevolucao` (default 20%) substitui os 30% fixos; arma após `pLucroRef`.
6. **Consolidação ligada por padrão** (`pConsolicao = True`).

### 7.4 Filtro de distância na reentrada após saída pela M3 (18/06/2026)

Novo input `pDistMinReentrada` (default 100 pts): quando a saída foi **pelo cruzamento da M3**, a reentrada (qualquer sentido) só ocorre com o preço a ≥ 100 pts da M3, somado à regra da vela seguinte. Não atua após stop gain/loss/horário/trava (preserva o "recompra na vela seguinte" após stop gain). Implementado com `bExitM3Pendente` (marca a saída-por-M3 no bloco 11) → `bFiltroDistReentrada` (armado na detecção da saída, bloco 10) → condição de distância na entrada (bloco 12), zerada ao entrar.

### 7.5 Distância contra a M3 congelada — testada e revertida (18/06/2026)

Chegou-se a medir a distância de reentrada contra a M3 **congelada na saída** (`vM3SaidaRef`); a pedido do trader, **revertido** — a distância voltou a ser medida contra a **M3 ao vivo** (seção 2.3).

### 7.6 Correção do latch da saída-por-M3 (18/06/2026)

Bug de timing: o filtro de distância (7.4) muitas vezes **não armava** quando o pregão ficava colado na M3. O sinalizador `bExitM3Pendente` era **zerado a cada tick** e só religado se, naquele tick, o preço ainda estivesse abaixo/acima da M3. Como entre **enviar** a zeragem e ela **executar** passam vários ticks (e o preço oscila em cima da M3 nesse intervalo), na detecção da saída o sinalizador já costumava estar `False` → reentrada colada na M3. **Correção:** `bExitM3Pendente` virou um **latch** — ligado quando a saída pela M3 é disparada (bloco 11) e zerado só na detecção da saída (bloco 10); removido o reset por-tick.

### 7.7 Reentrada após saída pela M3 vira ordem STOP (18/06/2026)

Com a reentrada a mercado, a distância dos 100 pts era verificada na **decisão**, mas a **execução** caía mais perto da M3 por slippage (preço recuava entre criar e preencher a ordem). Trocado para **ordem stop** (opção do trader): com o filtro armado, o robô mantinha a cada tick um BuyStop e um SellShortStop em `M3 ± pDistMinReentrada`. **Superado pela 7.8.**

### 7.8 Entrada por rompimento de vela + inclinação da M3 (18/06/2026)

Redefinidas as condições de entrada (a pedido do trader). **Compra** (espelho na venda): (a) nível acima da M3; (b) *na reentrada* o rompimento ≥ `pDistMinReentrada` da M3; (c) inclinação da M3 `M3[1]−M3[2]` > `pInclinacao`(15); (d) **vela anterior positiva** e rompimento da **máxima + `pDeslocamento`(10)**. A entrada vira **ordem stop** no nível do rompimento (na reentrada, empurrado para `M3+pDistMinReentrada`). Saiu a entrada a mercado por cruzamento e o bracket M3±100 da 7.7. Como a vela anterior é positiva **ou** negativa, só um lado fica armado por vez (`nLadoArmado`) — acabou o risco de duas pernas simultâneas. Novos inputs `pInclinacao`/`pDeslocamento`; vars `vM3_ant`/`vM3_ant2`/`vInclinacaoM3`/`vOpen1`/`bVelaPositivaAnt`/`bVelaNegativaAnt`/`bArmaCompra`/`bArmaVenda`/`vGatilhoCompra`/`vGatilhoVenda`/`nLadoArmado`. As saídas (M3/stops/horário/trava), o latch da saída-M3 e a gestão diária seguem inalterados.

**A verificar no replay:** seletividade (pode operar pouco em pregão de lado); calibrar `pInclinacao`/`pDeslocamento`; e como a M3(21) tem inclinação medida em pts/vela, conferir se 15 pts não está alto/baixo demais para o WIN.

**Pendente:** verificar sintaxe no editor do Profit (atenção ao `MediaExp`) e validar em pregão/replay; observar a frequência de entradas tick a tick e calibrar `pPeriodoM3`, `pDistMinReentrada`, stops e `pPercDevolucao`.
