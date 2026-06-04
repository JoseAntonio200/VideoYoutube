# EA BoldGlad-CML — Documentação da Estratégia

**Plataforma:** ProfitUltra (NTSL)
**Modo de execução:** Tick a Tick
**Tipo:** Estratégia de execução automatizada para o WIN (mini-índice)

---

## 1. Visão geral

O BoldGlad-CML é um robô seguidor de tendência que opera com base no **viés de duas médias móveis aritméticas (M1 e M2)** e na **vela anterior alinhada ao viés**, com **entrada por ordem stop** no rompimento da máxima (compra) ou mínima (venda) da última vela fechada, mais um pequeno offset.

O robô é bidirecional (compra e vende) e possui:

- **Rearme contínuo** do gatilho stop a cada nova vela enquanto o viés se mantiver.
- **Filtro de tendência dominante** (média lenta direcional) que só libera compra a favor da alta e venda a favor da baixa.
- **Proteção contra mercados lateralizados** via faixa de consolidação (com buffer, reconstrução, e gatilho adicional por proximidade das médias).
- **Pausa após saída** de N velas para cortar o rearme em sequência (overtrading).
- **Gestão de posição** com stop fixo, alvo fixo, breakeven gradual e trailing stop.
- **Controles** de horário, meta diária, perda máxima diária (à prova de reinício) e trailing de proteção do lucro acumulado.

**Diferenças em relação ao BoldGlad-CRTF:** o BoldGlad-CML **não inverte a posição** quando ocorre um cruzamento contrário. Se o EA estiver comprado e M1 cruzar abaixo de M2, a posição comprada permanece aberta sob a gestão normal (stop, alvo, breakeven, trailing, horário, lock diário). O CRTF fecha imediatamente nesse caso.

### 1.1 O que muda em relação ao BoldGlad-CM (origem)

O BoldGlad-CML nasceu como cópia exata do BoldGlad-CM e recebeu **três mudanças** motivadas por um pregão de forte tendência de queda (03/06/2026) em que o CM perdeu por operar contra a tendência e por excesso de entradas:

1. **Filtro de tendência dominante** (`pUsarFiltroTendencia`, `pPeriodoTendencia`): uma média lenta (default 50) define a direção do dia. O viés de compra só vira ordem se o preço estiver **acima** dessa média; o de venda, só **abaixo**. Em dia de queda, o EA deixa de comprar contra a tendência. Ver seção 2.1.3.
2. **Menos entradas:** os gatilhos de antecipação de viés (inclinação forte da M1 e padrão de 3 velas) vêm **desligados por padrão** (`pInclinacaoMediaRapida = False`, `pTradeVela3 = False`), e uma **pausa de N velas após cada saída** de posição (`pVelasPausaAposSaida`, default 3) evita o rearme imediato em sequência. Ver seção 3.6.
3. **Trava-dia à prova de reinício** (`pResetarPlacarAoReiniciar`, default False): por padrão o contador de perda do dia **não zera** ao reabilitar o EA no meio do pregão — o limite de perda diária vale para o dia inteiro. Ver seção 6.3.

E, numa segunda rodada, **duas rotas de entrada novas**:

4. **Entrada de abertura (P1)** (`pTradeAbertura`, default True): no rompimento do range da 1ª vela do pregão (09:00), entra a mercado, uma vez por dia, só a favor da média de 50. `pHoraInicio` passou a **0900**. Ver seção 3.7.
5. **Reentrada após lucro (P2)** (`pReentradaAposLucro`, default True): após um trade vencedor, repete na mesma direção no rompimento da vela que fechou, furando a pausa, exigindo viés (M1×M2) **e** média de 50 a favor; encadeia enquanto vencer. Ver seção 3.8.

Todos os comportamentos são **configuráveis**: para fazer o CML operar idêntico ao CM, basta `pUsarFiltroTendencia = False`, `pInclinacaoMediaRapida = True`, `pTradeVela3 = True`, `pVelasPausaAposSaida = 1`, `pResetarPlacarAoReiniciar = True`, `pTradeAbertura = False`, `pReentradaAposLucro = False` e `pHoraInicio = 0905`.

**Tag de log:** todos os logs do CML usam o prefixo `BG-CML:` / `BG-CML-DIAG:` (o CM usa `BG-CM:`), para poderem ser filtrados separadamente no painel de eventos do Profit.

---

## 2. Componentes principais

### 2.1 Médias móveis

- **M1 (rápida):** padrão de 5 períodos, calculada sobre o fechamento.
- **M2 (lenta):** padrão de 9 períodos, calculada sobre o fechamento.

Viés (com priorização da inclinação — ver 2.1.1):

- M1 acima de M2 → viés de compra (a menos que a inclinação forte aponte para baixo).
- M1 abaixo de M2 → viés de venda (a menos que a inclinação forte aponte para cima).

### 2.1.1 Inclinação da M1 — antecipação de viés

Se `pInclinacaoMediaRapida = True` (default), o EA monitora a inclinação da M1 entre duas velas fechadas (vela 1 e vela 2). Quando a magnitude da variação ultrapassa um limite proporcional ao preço corrente, o viés é **antecipado** sem esperar o cruzamento real.

**Cálculo:**

```
vLimiteInclinacao = Close × pValorInclinacaoMediaRapida / 100   (em pontos)
vInclinacaoM1     = vM1_ant − vM1_ant2                          (em pontos, com sinal)
```

- `vInclinacaoM1 > vLimiteInclinacao` → **inclinação alta forte** → antecipa viés de compra.
- `vInclinacaoM1 < −vLimiteInclinacao` → **inclinação baixa forte** → antecipa viés de venda.

Default `pValorInclinacaoMediaRapida = 0.04` (0,04%). Para WIN em ~178.000, o limite é ~71 pts entre vela 2 e vela 1.

**Priorização em caso de conflito:** a inclinação forte **vence o viés regular** (M1 vs M2 atual). Por exemplo, se M1 está acima de M2 (compra regular) mas M1 caiu fortemente entre vela 2 e vela 1 (inclinação baixa forte), o viés final é **venda**. A inclinação é um sinal mais antecipatório — indica que a reversão está em curso antes do cruzamento real.

**Fórmula final do viés:**

```
bSinalCompra = bInclinacaoAlta OR ((M1 > M2) AND NOT bInclinacaoBaixa)
bSinalVenda  = bInclinacaoBaixa OR ((M1 < M2) AND NOT bInclinacaoAlta)
```

**Desligando a feature:** `pInclinacaoMediaRapida = False` faz `bInclinacaoAlta` e `bInclinacaoBaixa` ficarem sempre `False`. O viés volta ao comportamento original (apenas M1 vs M2 atual).

**Mesma defesa de data do cruzamento:** a inclinação só é considerada quando vela 1 e vela 2 são do mesmo dia da barra atual. Evita inclinação artificial entre o último candle do dia anterior e o primeiro do dia atual.

### 2.1.2 Padrão de 3 velas — gatilho alternativo de viés

Se `pTradeVela3 = True` (default), o EA monitora um padrão de **3 velas consecutivas** como gatilho alternativo de viés.

**Padrão de COMPRA:**
- Vela 3 (`Close[3] ≤ Open[3]`) — negativa **ou doji**.
- Vela 2 (`Close[2] > Open[2]`) — positiva (estrita).
- Vela 1 (`Close[1] > Open[1]`) — positiva (estrita; essa vela 1 já é o filtro de `bVelaAntAlta` existente).
- Gatilho: **rompimento de `vHigh1 + pOffsetEntrada`** (mesmo gatilho do viés regular).

**Padrão de VENDA (espelhado):**
- Vela 3 (`Close[3] ≥ Open[3]`) — positiva **ou doji**.
- Vela 2 negativa (estrita), vela 1 negativa (estrita).
- Gatilho: rompimento de `vLow1 − pOffsetEntrada`.

**Por que esse padrão:** uma sequência bear-bull-bull é uma reversão clássica vinda de uma fundo — a primeira vela mostra fraqueza, mas as duas seguintes recuperam força. O rompimento da máxima da vela 1 confirma que a recuperação tem fôlego. Espelhado para topo.

**Priorização final (das 3 rotas de viés):**

```
inclinação forte  >  padrão V3  >  regular (M1 vs M2)
```

Concretamente:

```
bSinalCompra = bInclinacaoAlta
              OR (bPadraoCompraV3 AND NOT bInclinacaoBaixa)
              OR ((vM1 > vM2) AND NOT bInclinacaoBaixa AND NOT bPadraoVendaV3)

bSinalVenda  = bInclinacaoBaixa
              OR (bPadraoVendaV3 AND NOT bInclinacaoAlta)
              OR ((vM1 < vM2) AND NOT bInclinacaoAlta AND NOT bPadraoCompraV3)
```

**Comportamento em cenários típicos:**

- Apenas M1 vs M2 sinaliza → regular ganha.
- Padrão V3 compra + M1 < M2 → **padrão V3 ganha** (antecipa o cruzamento).
- Inclinação alta forte + padrão V3 venda → **inclinação ganha** (mais imediata).
- Padrão V3 compra e venda simultaneamente: impossível pelas definições espelhadas.

**Desligando a feature:** `pTradeVela3 = False` faz `bPadraoCompraV3` e `bPadraoVendaV3` ficarem sempre `False`. O viés cai de volta na hierarquia "inclinação forte > regular".

**Mesma defesa de data:** as 3 velas (`Date[1]`, `Date[2]`, `Date[3]`) precisam ser do dia da barra atual. Evita padrão "atravessando" o pregão.

### 2.1.3 Filtro de tendência dominante (BoldGlad-CML)

Mecanismo exclusivo do CML. Uma **média lenta de tendência** (`pPeriodoTendencia`, default 50, calculada sobre o fechamento) define a direção dominante do dia. O viés calculado em 2.1 / 2.1.1 / 2.1.2 só é aceito **a favor** dessa média:

```
bTendenciaCompra = (não usar filtro) OU (Close > média de tendência)
bTendenciaVenda  = (não usar filtro) OU (Close < média de tendência)

bSinalCompra = (viés de compra das regras 2.1) E bTendenciaCompra
bSinalVenda  = (viés de venda das regras 2.1)  E bTendenciaVenda
```

- Com o preço **acima** da média de tendência, o EA só arma **compras** — qualquer viés de venda é descartado.
- Com o preço **abaixo**, só arma **vendas**.
- `pUsarFiltroTendencia = False` desliga o filtro: `bTendenciaCompra` e `bTendenciaVenda` ficam sempre `True` e o comportamento volta a ser idêntico ao do CM.

**Por que:** num pregão de tendência forte (como 03/06/2026, queda contínua), o viés por médias rápidas vira de lado a cada repique e leva o EA a comprar contra a maré. O filtro impede que o EA opere na contramão da tendência do dia — que foi a maior fonte de prejuízo do CM nesse cenário.

A média de tendência é plotada no gráfico (Plot5) para o trader visualizar a linha que separa o lado comprador do vendedor.

### 2.2 Vela anterior como confirmação inicial

### 2.2 Vela anterior como confirmação inicial

Para a **primeira armação** do gatilho, a vela imediatamente anterior precisa estar alinhada ao viés:

- Compra: vela anterior de alta (`Close[1] > Open[1]`).
- Venda: vela anterior de baixa (`Close[1] < Open[1]`).

Doji (`Close[1] = Open[1]`) não habilita nenhum lado.

### 2.3 Gatilho stop por rompimento (com fallback a mercado)

Confirmado o setup, o robô envia uma **ordem stop**:

- Compra: gatilho em **máxima da vela anterior + `pOffsetEntrada`**.
- Venda: gatilho em **mínima da vela anterior − `pOffsetEntrada`**.

A ordem é reemitida a cada tick enquanto o gatilho estiver armado.

**Fallback a mercado quando o preço já passou do gatilho:** uma ordem stop por definição exige preço **acima** do mercado atual (compra) ou **abaixo** (venda). Em velas explosivas — especialmente na abertura do pregão com gap forte — o preço pode disparar acima do gatilho antes do EA conseguir enviar a stop, e o broker rejeita com "preço stop deve ter o preço acima/abaixo do preço atual do mercado".

Para não perder esses rompimentos, o EA detecta esse caso e envia uma **ordem a mercado**:

- Compra: se `Close ≥ vGatilhoCompra` no momento do envio → `BuyAtMarket(pQtdContratos)`.
- Venda: se `Close ≤ vGatilhoVenda` → `SellShortAtMarket(pQtdContratos)`.

**Controle de uma única tentativa por vela:** o mercado executa imediatamente, então só pode ser disparado uma vez. Flags `bMarketComprouEnviado` / `bMarketVendeuEnviado` impedem reenvio dentro da mesma vela. As flags resetam a cada nova vela (bloco 7), dando nova chance caso a tentativa anterior não tenha executado.

**Trade-off conhecido:** entrada a mercado num spike pode pegar topo/fundo e levar stop loss rápido se o movimento for falso (spike "fake"). Aceito como compromisso para não perder rompimentos rápidos que a estratégia stop pura deixaria escapar.

**Logs novos** (uma vez por vela quando o fallback dispara):

- `BG-CML-DIAG: BuyAtMarket enviado (Close=X >= gatilho=Y) - rompimento intra-vela | Qtd=N | Hora ...`
- `BG-CML-DIAG: SellShortAtMarket enviado (Close=X <= gatilho=Y) - rompimento intra-vela | Qtd=N | Hora ...`

### 2.4 Cruzamento de médias (somente para faixa de consolidação)

O cruzamento de M1 com M2 é identificado quando a posição relativa muda entre vela 1 e vela 2 (ambas fechadas). No BoldGlad-CML essa detecção é usada **exclusivamente para a faixa de consolidação** — não dispara entradas nem reverte posições.

---

## 3. Lógica de entrada

### 3.1 Compra

A ordem stop de compra é armada quando, em uma nova vela, todas as condições abaixo estão presentes:

1. Viés de compra ativo (ver 2.1, 2.1.1 e 2.1.2) — vale pelo cruzamento real (M1 > M2), pela antecipação por inclinação alta forte, ou pelo padrão V3 (vela 3 negativa ou doji + vela 2 positiva + vela 1 positiva), seguindo a priorização inclinação > padrão V3 > regular.
2. A vela anterior é de alta (somente na primeira armação).
3. Não há posição aberta.
4. Horário dentro da janela.
5. Sem lock diário (meta ou loss máximo).
6. Sem faixa de consolidação ativa.
7. A vela atual não é a vela de saída de uma posição anterior.

Gatilho: **`vHigh1 + pOffsetEntrada`**. Reemitido a cada tick via `BuyStop`.

### 3.2 Venda

Espelho da compra:

1. Viés de venda ativo (M1 < M2, inclinação baixa forte, ou padrão V3 venda — ver 2.1, 2.1.1 e 2.1.2; mesma priorização da compra).
2. Vela anterior de baixa (na primeira armação).
3. Demais condições gerais.

Gatilho: **`vLow1 − pOffsetEntrada`**. Reemitido a cada tick via `SellShortStop`.

### 3.3 Rearme contínuo do gatilho

O gatilho **não fica congelado** após a primeira armação:

- A cada nova vela, enquanto o viés se mantiver, o gatilho é recalculado para a máxima (compra) ou mínima (venda) da nova vela fechada, com o mesmo offset.
- O filtro de vela anterior alinhada **só é exigido na primeira armação**. No rearme não há refiltro.
- Se o viés virar (M1 atravessa M2), o gatilho é desarmado e a ordem pendente é cancelada.
- Se a faixa de consolidação se formar, ambos os gatilhos são desarmados.

### 3.4 Comportamento ao ligar o EA em tempo real

O EA não tem mecanismo de "entrada inicial a mercado". Ao ser habilitado, ele aguarda a próxima nova vela e arma o gatilho normalmente se as condições do setup forem satisfeitas. Se já houver posição aberta no momento do start, o gerenciamento (stop, alvo, breakeven, trailing) assume controle automaticamente.

### 3.5 Sem reversão

Cruzamento contrário com posição aberta **não fecha** a posição. O trade segue sob a gestão normal e só sai por stop, alvo, fim de horário ou lock diário.

### 3.6 Pausa após saída (BoldGlad-CML)

Para cortar o overtrading (rearmar e reentrar candle após candle, que no CM gerou sequências longas de pequenas perdas), o CML impõe uma **janela de pausa após qualquer saída de posição**:

- Quando uma posição fecha (por stop, alvo, breakeven, trailing ou saída forçada), o EA registra a vela da saída.
- Durante `pVelasPausaAposSaida` velas a partir daí (default 3), **nenhum gatilho é armado e nenhuma ordem de entrada é enviada**.
- Após a janela, o EA volta a armar normalmente se as condições do setup (incluindo o filtro de tendência) estiverem presentes.

`pVelasPausaAposSaida = 1` reproduz o comportamento antigo do CM (bloqueia apenas a própria vela da saída, sem pausa adicional). O diagnóstico do bloco 13 mostra o campo `PausaSaida=True/False` indicando se a pausa está ativa naquela vela.

> **A reentrada após lucro (3.8) NÃO respeita esta pausa** — um trade vencedor pode reentrar imediatamente na vela seguinte.

### 3.7 Entrada de abertura (P1 — BoldGlad-CML)

Controlada por `pTradeAbertura` (default ligado). É uma rota de entrada dedicada ao início do pregão:

1. Na virada do dia, o EA marca a **primeira vela do pregão (09:00)**. Quando essa vela fecha, ele grava o **topo** e o **fundo** dela.
2. A partir daí, vigia o rompimento desse range e entra **a mercado** (não por ordem stop):
   - `Close` rompe **abaixo do fundo** da 1ª vela → `SellShortAtMarket`, desde que o preço esteja **abaixo** da média de tendência.
   - `Close` rompe **acima do topo** da 1ª vela → `BuyAtMarket`, desde que o preço esteja **acima** da média de tendência.
3. Dispara **no máximo uma vez por dia** (`bAberturaUsada`). Depois disso, a lógica normal e a reentrada (3.8) assumem.

**Filtro de tendência manda:** se o rompimento for para um lado contrário à média de 50, a entrada de abertura **não acontece** (o lado é descartado). Ou seja, a abertura só opera a favor da tendência dominante.

**Horário:** como a 1ª vela do pregão é a das 09:00, o `pHoraInicio` foi mudado para **0900** para a regra valer desde a abertura. Se preferir começar mais tarde, ajustar `pHoraInicio` adia também a entrada de abertura.

**Não passa pelo filtro de viés nem de vela anterior alinhada** — é rompimento puro do range de abertura, gated apenas pela média de 50.

Logs:
- `BG-CML-DIAG: Range da 1a vela do dia capturado - Topo X - Fundo Y`
- `BG-CML-DIAG: ABERTURA venda a mercado (Close=X < fundo 1a vela=Y) | Qtd=N | Hora ...`
- `BG-CML-DIAG: ABERTURA compra a mercado (Close=X > topo 1a vela=Y) | Qtd=N | Hora ...`

### 3.8 Reentrada após trade vencedor (P2 — BoldGlad-CML)

Controlada por `pReentradaAposLucro` (default ligado). Objetivo: surfar uma sequência a favor — quando um trade dá certo, tentar de novo na mesma direção.

1. Quando uma posição **fecha no lucro** (qualquer resultado `> 0`; trade zerado não conta), o EA arma uma reentrada na **mesma direção** do vencedor.
2. Na vela seguinte, arma o gatilho stop no **rompimento da vela que acabou de fechar** (topo + offset para compra, fundo − offset para venda).
3. A reentrada **fura a pausa pós-saída** (3.6), mas **exige confirmação dupla**: o viés das médias (M1×M2) **e** a média de tendência de 50 precisam estar a favor da direção. Se qualquer um não confirmar, a cadeia é cancelada e o EA volta à lógica normal.
4. **Encadeia enquanto vencer:** se a reentrada também fechar no lucro, arma de novo; na **primeira perda** (ou trade zerado), a cadeia morre e a pausa pós-saída volta a valer.

**Detecção de lucro:** o EA guarda o lucro aberto do último tick com posição (`vUltimoLucroAberto`); no tick da saída, o sinal desse valor indica se o trade fechou positivo. É uma aproximação confiável para saídas por stop/alvo/trailing.

Logs:
- `BG-CML-DIAG: Trade VENCEDOR (lucro X pts) - reentrada armada na mesma direcao do vencedor`
- `BG-CML-DIAG: REENTRADA compra armada @ X (Hi1=Y + offset=Z) | continuacao do trade vencedor comprado`
- `BG-CML-DIAG: REENTRADA venda armada @ X (Lo1=Y - offset=Z) | continuacao do trade vencedor vendido`
- `BG-CML-DIAG: Reentrada CANCELADA - vies ou media de 50 nao confirmam a continuacao`

---

## 4. Faixa de consolidação

Mecanismo de proteção contra mercados lateralizados. Quando duas tentativas de cruzamento ocorrem próximas, ou quando as médias estão muito próximas uma da outra, o robô identifica a região de indecisão e suspende a operação até o preço se afastar dela.

### 4.1 Detecção do evento de cruzamento

Um **evento de cruzamento** é qualquer um dos dois sinais abaixo:

- **Cruzamento real:** a posição relativa entre M1 e M2 muda entre vela 1 (acabou de fechar) e vela 2 (anterior a essa).
- **Proximidade das médias:** `Abs(vM1_ant − vM2_ant) ≤ vLimiteProximidade`, onde `vLimiteProximidade = Close * pPercProximidadeMedias / 100`. Ou seja: se a diferença entre M1 e M2 na vela 1 (já fechada) está abaixo de uma fração configurável do preço corrente, isso conta como evento equivalente a um cruzamento e dispara a mesma lógica.

Em ambos os casos:

- A detecção roda **apenas uma vez por vela**, no primeiro tick da nova vela.
- O evento só vale quando vela 1 (e vela 2, no caso do cruzamento real) é do **mesmo pregão** da barra atual.
- Cada evento gravado e cada faixa formada carregam a data (`nDataCruzAnt`, `nDataFaixa`).
- Faixas e referências de pregões anteriores são descartadas automaticamente pelo filtro de data.

> **Por que aceitar proximidade como cruzamento:** em mercados lentos, as médias podem oscilar muito próximas sem cruzar de fato — visualmente é uma consolidação, mas a regra do "cruzamento real" não dispararia. A proximidade preenche essa lacuna. Default de 0,005% sobre o preço dá aproximadamente 6–9 pontos no WIN.

### 4.2 Preço de referência do evento

`vPrecoCruzAtual = (vM1_ant + vM2_ant) / 2` — ponto médio entre M1 e M2 no fechamento da vela do evento (cruzamento real ou proximidade).

### 4.3 Regras de memória

**Sem faixa ativa** (estado normal):

- Distância > `pDistMaxCruz` → cruzamento antigo é descartado; o atual vira a 1ª referência.
- Distância ≤ `pDistMaxCruz` → **forma faixa** (varredura, ver 4.4) e atualiza referência.

**Com faixa ativa:**

- Distância > `pDistMaxCruz` → **faixa NÃO é desfeita** e a memória **mantém o cruzamento anterior** (não atualiza referência).
- Distância ≤ `pDistMaxCruz` → **RECONSTRÓI a faixa** (varredura entre o cruzamento em memória e o atual) e atualiza referência.

### 4.4 Cálculo do topo e fundo (varredura)

- Janela: começa em `pVelasAntesPrimCruz` velas antes do 1º cruzamento e termina na vela do 2º cruzamento (vela 1).
- Percorre cada vela da janela registrando a maior máxima (topo bruto) e a menor mínima (fundo bruto).
- Se o histórico for insuficiente, usa o que tiver disponível.
- **Buffer:** o topo recebe **+`pBufferFaixa`** e o fundo recebe **−`pBufferFaixa`**. Os valores armazenados em `vTopoFaixa` / `vFundoFaixa` já incluem o buffer.

### 4.5 Comportamento durante a faixa ativa

- O robô não arma gatilhos (`bArmCompra` / `bArmVenda` são forçados a False).
- Ordens pendentes são canceladas (quando não há posição).
- Candles são pintados de amarelo (sinalização visual).
- Topo e fundo da faixa (já com buffer) são plotados (Plot3, Plot4).
- Novos cruzamentos internos: tratados pelas regras de 4.3 (mantém ou reconstrói).

### 4.6 Desfazimento da faixa

A faixa é desfeita quando o fechamento se afasta `pDistSaidaFaixa` além dos limites já ampliados pelo buffer:

- `Close > vTopoFaixa + pDistSaidaFaixa` → desfaz.
- `Close < vFundoFaixa − pDistSaidaFaixa` → desfaz.

Distância efetiva total da saída = `pBufferFaixa + pDistSaidaFaixa` para cima ou para baixo dos limites da varredura.

Após o desfazimento, o robô volta a operar normalmente e **aguarda o próximo gatilho stop** a ser armado (não há entrada imediata por saída de faixa).

---

## 5. Gerenciamento da posição

Quando uma posição está aberta, o robô gerencia stop loss, take profit, breakeven e trailing stop continuamente.

### 5.1 Stop fixo e alvo fixo

Inicialização (primeiro tick com posição):

- Compra: stop em `entrada − pStopFixoPts`, alvo em `entrada + pAlvoFixoPts`.
- Venda: stop em `entrada + pStopFixoPts`, alvo em `entrada − pAlvoFixoPts`.

`SellToCoverStop` + `SellToCoverLimit` (compra) e `BuyToCoverStop` + `BuyToCoverLimit` (venda) são reemitidos a cada tick **com a quantidade explícita igual ao tamanho real da posição no broker** (`Abs(PositionQty)`). Isso cobre cenários de reinício do EA com posição já ativa, atende à exigência do manual NTSL (seção 11.11) de que o programador gerencie a quantidade de cada ordem de cobertura quando posicionado em mais de um lote, e também cobre qualquer ajuste manual de quantidade feito pela UI do Profit enquanto o EA está rodando.

### 5.2 Breakeven gradual (opcional)

Se `pUsarBreakeven = True`, o stop começa em `pStopFixoPts` da entrada e **se aproxima do breakeven aos poucos**: a cada `pStepBreakevenPts` de lucro a favor, o stop avança `pStepBreakevenPts` na direção da entrada.

Exemplo (compra com `pStopFixoPts = 150`, `pStepBreakevenPts = 50`):

| Lucro a favor | Stop |
|---|---|
| 0 pts | entrada − 150 |
| 50 pts | entrada − 100 |
| 100 pts | entrada − 50 |
| 150 pts | entrada (breakeven) |
| 200 pts+ | breakeven; trailing assume o avanço |

O stop nunca ultrapassa o breakeven por esse mecanismo, e nunca afrouxa.

### 5.3 Trailing stop por candle (opcional)

Se `pUsarTrailing = True`, ativa quando o lucro atinge `pDistAcionaTrailing`. A partir daí, a cada nova vela:

- Compra: stop = `mínima da vela anterior − pOffsetEntrada` (apenas se for maior que o stop atual).
- Venda: stop = `máxima da vela anterior + pOffsetEntrada` (apenas se for menor que o stop atual).

Breakeven e trailing trabalham juntos — o stop efetivo é sempre o mais protetivo dos dois.

---

## 6. Controles de tempo e proteções diárias

### 6.1 Horário de operação

Janela definida por `pHoraInicio` (padrão **09:00** no CML, para a entrada de abertura valer desde a 1ª vela) e `pHoraFim` (padrão 17:30). Ao atingir o horário de fim, qualquer posição aberta é fechada e ordens pendentes canceladas.

### 6.2 Meta diária, loss máximo e trailing de proteção do lucro

O robô monitora o resultado acumulado em **pontos** (convertido de R$ via `pValorPontoRS`):

```
vResultadoDia = (DailyResult(True) − vBaselineResultado) / (pValorPontoRS × pQtdContratos)
```

Três condições disparam o lock diário (qualquer uma fecha posição aberta e impede novas operações até o próximo dia, ou até reinício do EA conforme 6.3):

**a) Loss máximo diário — sempre ativo:**
- `vResultadoDia ≤ −pLossMaxDiarioPts` → lock.
- Log: `BG-CML: Lock diario - loss maximo diario atingido @ X pts (limite -Y)`.

**b) Trailing de proteção do lucro — ativo após `pLucroRef` ser atingido:**

A cada novo dia, `vLucroAcumRef` é inicializado em `pLucroRef` (default 1100 pts). A cada fechamento de vela, se `vResultadoDia > vLucroAcumRef`, o pico é atualizado.

Quando `vResultadoDia ≥ pLucroRef` pela primeira vez no dia, o trailing **arma** (`bLucroRefAtingido := True`):
- Log: `BG-CML: Lucro de referencia atingido - trailing armado @ X pts - bloqueio se cair para Y pts ou menos do pico`.

A partir desse momento:
- `vResultadoDia ≤ vLucroAcumRef × 0.70` (lucro cai para 70% do pico — perda de 30%) → lock.
- Log: `BG-CML: Lock diario - lucro caiu para 70% do pico (perda de 30%) - pico X pts - atual Y pts`.

**c) Meta diária — ativa SOMENTE enquanto o trailing não armou:**
- Se `bLucroRefAtingido = False` E `vResultadoDia ≥ pMetaDiariaPts` → lock.
- Log: `BG-CML: Lock diario - meta diaria atingida @ X pts (limite Y)`.
- Uma vez que o trailing arma, a meta diária deixa de bloquear (opção A do requisito) — o EA segue rodando até o trailing disparar.

**Resumo das prioridades:**

| Estado | Quem bloqueia |
|---|---|
| Antes de `vResultadoDia` atingir `pLucroRef` | Meta diária + loss diário |
| Após `vResultadoDia` atingir `pLucroRef` (trailing armado) | Trailing (vLucroAcumRef × 0.70) + loss diário |

### 6.3 Comportamento ao reiniciar o EA no meio do dia (à prova de reinício no CML)

Controlado por `pResetarPlacarAoReiniciar`:

- **`False` (default do CML — à prova de reinício):** ao reabilitar o EA no meio do pregão, o contador do dia **não é zerado**. O `vResultadoDia` reflete o resultado real do dia inteiro (`DailyResult`), então o limite de perda diária (`pLossMaxDiarioPts`) e a meta diária continuam valendo independentemente de quantas vezes o EA foi religado. Se o dia já estava no prejuízo máximo, o EA volta **já travado**.
- **`True` (comportamento antigo do CM):** ao entrar em tempo real, o EA captura o resultado atual como **baseline** (`vBaselineResultado`) e zera o lock. A contagem reinicia do zero, e o lock só volta a ativar se `pMetaDiariaPts` ou `pLossMaxDiarioPts` forem atingidos **a partir desse ponto**.

**Motivação da mudança:** no pregão de 03/06/2026 o EA (versão CM) foi reiniciado durante o dia. Cada reinício zerou o contador de perda, desarmando a trava de R$2.000 justamente no pior dia — o prejuízo chegou a R$4.450. Com o default `False`, esse vazamento não acontece.

A memória de cruzamentos e a faixa de consolidação **não são zeradas** nesse momento (em nenhum dos modos) — a defesa contra dados de pregões anteriores é por **data**, então cruzamentos do dia atual processados durante o histórico permanecem válidos.

Memória de cruzamentos e faixa de consolidação **não são zeradas** nesse momento — a defesa contra dados de pregões anteriores é por **data**, então cruzamentos do dia atual processados durante o histórico permanecem válidos.

### 6.4 Reset diário

Ao detectar mudança de data, o robô reseta: lock diário, baseline, gatilhos armados, faixa de consolidação, memória de cruzamentos, flags de log e cancela ordens pendentes. **`vLucroAcumRef` volta a `pLucroRef`** e `bLucroRefAtingido` volta a `False` — o trailing de proteção do lucro começa do zero a cada novo pregão. Variáveis de breakeven e trailing-stop da posição são resetadas no bloco "sem posição".

---

## 7. Sinalização visual e logs

### 7.1 Plotagens e marcações

- **M1 e M2** plotadas como linhas (Plot, Plot2).
- **Topo e fundo da faixa** (já com buffer) plotados como linhas (Plot3, Plot4) quando ativa.
- **Candles verdes** quando comprado.
- **Candles vermelhos** quando vendido.
- **Candles amarelos** quando em faixa de consolidação (sem posição).
- **"BUY"** plotado no gráfico quando gatilho de compra está armado.
- **"SELL"** plotado quando gatilho de venda está armado.

### 7.2 Logs do console

Todos prefixados com `BG-CML:` (operacionais) ou `BG-CML-DIAG:` (diagnóstico — podem ser reduzidos depois que o comportamento estiver validado).

**Cruzamentos e faixa:** (os logs de "Cruzamento Detectado" também aparecem quando o evento foi disparado por proximidade das médias — o texto é genérico)

- `BG-CML: Cruzamento Detectado - 1o cruzamento - Consolidacao - OFF`
- `BG-CML: Cruzamento Detectado - Dist. X pts - Consolidacao - OFF`
- `BG-CML: EA bloqueado - Faixa de consolidacao ativa - TP <topo> - FS <fundo>`
- `BG-CML: Faixa RECONSTRUIDA - Dist. X pts - TP <topo> - FS <fundo>`
- `BG-CML: Cruzamento na faixa - Dist. X pts > max - Faixa MANTIDA`
- `BG-CML: EA liberado para operacao - Faixa desfeita`
- `BG-CML: Faixa de outro pregao descartada`

**Gestão diária (lock e trailing):**

- `BG-CML: Lucro de referencia atingido - trailing armado @ X pts - bloqueio se cair para Y pts ou menos do pico` (uma única vez por dia, quando vResultadoDia atinge pLucroRef)
- `BG-CML: Lock diario - loss maximo diario atingido @ X pts (limite -Y)`
- `BG-CML: Lock diario - lucro caiu para 70% do pico (perda de 30%) - pico X pts - atual Y pts`
- `BG-CML: Lock diario - meta diaria atingida @ X pts (limite Y)`

**Detecção de posição (no primeiro tick com posição):**

- `BG-CML: Posicao comprada detectada - Entrada X - Stop Y - Alvo Z`
- `BG-CML: Posicao vendida detectada - Entrada X - Stop Y - Alvo Z`

**Diagnóstico do bloco 8 (inclinação forte da M1 — uma linha por nova vela quando ativo):**

- `BG-CML-DIAG: Inclinacao M1 ALTA forte (X pts, limite Y pts) - antecipa vies COMPRA`
- `BG-CML-DIAG: Inclinacao M1 BAIXA forte (X pts, limite -Y pts) - antecipa vies VENDA`

**Diagnóstico do bloco 8 (padrão de 3 velas — uma linha por nova vela quando o padrão estiver presente):**

- `BG-CML-DIAG: Padrao V3 COMPRA detectado (Vela3 neg/doji, Vela2 pos, Vela1 pos) - gatilho @ H1+offset`
- `BG-CML-DIAG: Padrao V3 VENDA detectado (Vela3 pos/doji, Vela2 neg, Vela1 neg) - gatilho @ L1-offset`

**Diagnóstico do bloco 13 (armar/rearmar gatilho — uma linha por nova vela):**

- `BG-CML-DIAG: Arm. block nova barra | HasPos=... | HoraOK=... | LockDiario=... | EmConsol=... | PausaSaida=... | Hora ...`
- `BG-CML-DIAG: Compra ARMADA gatilho @ X (Hi1=Y + offset=Z) | M1=... M2=... | bVelaAntAlta=True`
- `BG-CML-DIAG: Compra NAO armada | Vies OK ... | mas vela anterior nao alinhada ...`
- `BG-CML-DIAG: Compra DESARMADA - vies virou (M1=... <= M2=...)`
- `BG-CML-DIAG: Venda ARMADA gatilho @ X (Lo1=Y - offset=Z) | M1=... M2=... | bVelaAntBaixa=True`
- `BG-CML-DIAG: Venda NAO armada | Vies OK ... | mas vela anterior nao alinhada ...`
- `BG-CML-DIAG: Venda DESARMADA - vies virou (M1=... >= M2=...)`

**Diagnóstico do bloco 14 (envio de ordens — uma vez por armamento):**

- `BG-CML-DIAG: BuyStop enviado @ X (Hi1=Y + offset=Z) | Qtd=N | Hora ...`
- `BG-CML-DIAG: SellShortStop enviado @ X (Lo1=Y - offset=Z) | Qtd=N | Hora ...`
- `BG-CML-DIAG: BuyAtMarket enviado (Close=X >= gatilho=Y) - rompimento intra-vela | Qtd=N | Hora ...` (fallback quando preço já passou do gatilho)
- `BG-CML-DIAG: SellShortAtMarket enviado (Close=X <= gatilho=Y) - rompimento intra-vela | Qtd=N | Hora ...`

---

## 8. Parâmetros de entrada

### 8.1 Médias móveis

| Parâmetro | Padrão | Descrição |
|---|---|---|
| `pPeriodoM1` | 5 | Período da média rápida M1 |
| `pPeriodoM2` | 9 | Período da média lenta M2 |
| `pUsarFiltroTendencia` | True | **(CML)** Ativa o filtro de tendência dominante. Só libera compra com `Close` acima da média de tendência e venda com `Close` abaixo. |
| `pPeriodoTendencia` | 50 | **(CML)** Período da média de tendência (mais lenta que M1/M2). Maior = filtro mais robusto e lento; menor = mais sensível. |
| `pInclinacaoMediaRapida` | **False** | **(CML: default mudado de True para False)** Ativa o gatilho de antecipação de viés via inclinação forte da M1. Desligado por padrão para reduzir o flip-flop de viés. |
| `pValorInclinacaoMediaRapida` | 0.04 | Percentual do `Close` que define o limite da inclinação forte. Para WIN em ~178.000 dá ~71 pts entre vela 2 e vela 1. Em conflito com o viés regular (M1 vs M2), a inclinação ganha. |
| `pTradeVela3` | **False** | **(CML: default mudado de True para False)** Ativa o gatilho de viés por padrão de 3 velas (bear-bull-bull para compra; bull-bear-bear para venda). Desligado por padrão para reduzir entradas. |

### 8.2 Offset

| Parâmetro | Padrão | Descrição |
|---|---|---|
| `pOffsetEntrada` | 5.0 | Deslocamento em pontos aplicado ao gatilho stop de entrada e ao trailing |

### 8.3 Stop e alvo

| Parâmetro | Padrão | Descrição |
|---|---|---|
| `pStopFixoPts` | 150.0 | Stop loss fixo em pontos |
| `pAlvoFixoPts` | 500.0 | Take profit fixo em pontos |

### 8.4 Breakeven

| Parâmetro | Padrão | Descrição |
|---|---|---|
| `pUsarBreakeven` | True | Ativa breakeven gradual |
| `pStepBreakevenPts` | 50.0 | A cada X pts de lucro, o stop se aproxima X pts do breakeven |

### 8.5 Trailing stop

| Parâmetro | Padrão | Descrição |
|---|---|---|
| `pUsarTrailing` | True | Ativa o trailing stop |
| `pDistAcionaTrailing` | 150.0 | Pontos a favor para acionar o trailing |

### 8.6 Quantidade

| Parâmetro | Padrão | Descrição |
|---|---|---|
| `pQtdContratos` | 10.0 | Quantidade de contratos por trade |
| `pVelasPausaAposSaida` | 3 | **(CML)** Velas de pausa após qualquer saída de posição antes de armar nova entrada. `1` reproduz o comportamento antigo do CM. Não se aplica à reentrada após lucro (P2). |
| `pTradeAbertura` | True | **(CML)** Ativa a entrada de abertura (P1): entra a mercado no rompimento do range da 1ª vela do pregão (09:00), uma vez por dia, só a favor da média de 50. |
| `pReentradaAposLucro` | True | **(CML)** Ativa a reentrada após trade vencedor (P2): repete na mesma direção no rompimento da vela que fechou, furando a pausa, exigindo viés + média de 50 a favor. Encadeia enquanto vencer. |

### 8.7 Horário

| Parâmetro | Padrão | Descrição |
|---|---|---|
| `pHoraInicio` | **0900** | **(CML: mudado de 0905 para 0900)** Horário de início (HHMM). 0900 para a entrada de abertura (P1) valer desde a 1ª vela do pregão. |
| `pHoraFim` | 1730 | Horário de término (HHMM) |

### 8.8 Gestão diária

| Parâmetro | Padrão | Descrição |
|---|---|---|
| `pMetaDiariaPts` | 2000.0 | Ganho máximo no dia em pontos. **Só bloqueia enquanto o trailing (`pLucroRef`) não estiver armado** — após armar, é ignorado. |
| `pLossMaxDiarioPts` | 1000.0 | Perda máxima no dia em pontos. Sempre ativo. |
| `pValorPontoRS` | 0.20 | Valor financeiro de 1 ponto do ativo, em R$ (0.20 = WIN) |
| `pLucroRef` | 1100.0 | Lucro de referência em pontos. Quando `vResultadoDia` atinge esse valor, o trailing de proteção arma; daí em diante, o EA bloqueia se o lucro cair para 70% do pico (`vLucroAcumRef`), ou seja, uma perda de 30% do pico. |
| `pResetarPlacarAoReiniciar` | False | **(CML)** Se `True`, reabilitar o EA no meio do dia zera o contador de perda/lucro do dia (comportamento antigo do CM). Se `False` (default), o limite de perda diária vale para o dia inteiro mesmo após reinícios. |

### 8.9 Faixa de consolidação

| Parâmetro | Padrão | Descrição |
|---|---|---|
| `pDistMaxCruz` | 50.0 | Distância máxima entre dois cruzamentos para formar (ou reconstruir) faixa |
| `pBufferFaixa` | 100.0 | Acréscimo ao topo e decréscimo ao fundo da faixa após a varredura |
| `pDistSaidaFaixa` | 50.0 | Distância adicional para o preço sair da faixa (além dos limites já ampliados pelo buffer) |
| `pVelasAntesPrimCruz` | 5 | Velas anteriores ao 1º cruzamento incluídas na varredura |
| `pPercProximidadeMedias` | 0.005 | Percentual do `Close` — se `Abs(M1−M2) ≤ Close × pct / 100`, gera evento equivalente a um cruzamento. Default ~6–9 pts no WIN |

---

## 9. Fluxo resumido por tick

1. Detecta nova barra / novo dia, atualiza referências e reseta o que for do dia.
2. Atualiza horário, checa trailing de proteção do lucro (atualiza pico em nova vela, arma se `vResultadoDia ≥ pLucroRef`) e checa lock diário (trailing OR meta diária OR loss diário, conforme 6.2).
3. Calcula M1, M2 (atuais e das velas 1 e 2), lê dados da vela 1 (Open1/High1/Low1/Close1).
4. Atualiza sinais: viés (tick a tick), evento de cruzamento (vela 1 vs vela 2 OU proximidade das médias na vela 1, mesma data).
5. **Faixa de consolidação:** se nova barra e evento detectado, aplica as regras de memória (4.3 e 4.4). Verifica desfazimento por afastamento do preço (4.6).
6. Detecta saída de posição → bloqueia reentrada na mesma vela.
7. **Saídas forçadas:** fim de horário ou lock diário.
8. **Gerenciamento da posição:** stop/alvo iniciais (se primeiro tick), breakeven gradual, trailing stop, reemissão de stop/alvo a cada tick (qty = `Abs(PositionQty)`).
9. Reset de variáveis de gestão quando não há posição.
10. **Armar/rearmar gatilho** (apenas em nova barra): aplica viés + vela anterior alinhada (1ª armação) ou só viés (rearme). Desarma se viés virou ou se está em faixa.
11. **Envio das ordens** (a cada tick): `BuyStop` ou `SellShortStop` reemitido se gatilho armado.
12. Reset do gatilho após entrada executada.
13. Cancela ordens pendentes se não há posição nem gatilho.
14. Plotagem da faixa (se ativa) e sinalização visual (candles e textos).

---

## 10. Histórico de ajustes recentes

### 10.1 Origem — fork do BoldGlad-CRTF

O BoldGlad-CML foi criado a partir do BoldGlad-CRTF e diverge nas seguintes decisões:

- **Sem reversão** por cruzamento contrário: a posição segue sob gestão normal mesmo com inversão de viés.

### 10.2 Mantido do BoldGlad-CRTF

- Entrada por **ordem stop** no rompimento da máxima/mínima da vela anterior + offset.
- Filtro de vela anterior alinhada na primeira armação.
- **Rearme contínuo** do gatilho enquanto o viés se mantiver.
- Detecção de cruzamento por velas fechadas (vela 1 vs vela 2), preço de referência = ponto médio das médias.
- Stop fixo, alvo fixo, breakeven gradual, trailing por vela anterior.
- Reset do lock diário ao reiniciar o EA no meio do dia (baseline em `DailyResult`).
- Reemissão das ordens de stop/alvo a cada tick.
- Defesa por data de cruzamentos e faixas (cruzamentos do dia atual processados no histórico permanecem válidos).

### 10.3 Buffer da faixa de consolidação

Novo parâmetro `pBufferFaixa` (default 100). Após a varredura, o topo recebe +buffer e o fundo recebe −buffer. Os valores armazenados (`vTopoFaixa`, `vFundoFaixa`) já incluem o buffer. A saída da faixa exige uma distância **adicional** de `pDistSaidaFaixa` além desses limites já ampliados.

### 10.4 Reconstrução da faixa por novo cruzamento interno

Mudança no comportamento durante faixa ativa:

- Antes: novos cruzamentos só atualizavam a referência em memória; a faixa permanecia intocada.
- Agora:
  - **Dist ≤ `pDistMaxCruz`** → **reconstrói** a faixa com nova varredura entre o cruzamento em memória e o atual, atualiza referência.
  - **Dist > `pDistMaxCruz`** → **faixa não é desfeita** e a memória **mantém** o cruzamento anterior.

### 10.5 Logs de diagnóstico no bloco 13 e 14

Logs `BG-CML-DIAG:` adicionados para tornar visível, a cada nova vela e a cada envio de ordem, por que o gatilho armou, não armou, foi desarmado ou foi bloqueado. Esses logs podem ser reduzidos/removidos depois que o comportamento estiver validado em produção.

**Histórico do log `Armacao bloqueada na nova barra`:** foi removido temporariamente por gerar ruído excessivo, mas **restaurado** em 21/05/2026 após um caso em que o EA deixou de operar em 8 velas seguidas e ficamos sem nenhuma pista no console sobre qual condição estava bloqueando. Esse log é a única janela de diagnóstico do bloco 13 quando uma das 6 condições gerais bloqueia a armação (HasPos / HoraOK / LockDiario / EmConsol / BarraSaida / TimeframeOK). Foi adicionado o campo `TimeframeOK` ao log na restauração.

### 10.6 Redução parcial de contratos — feature adicionada e depois REMOVIDA

**Adicionada em 14/05/2026:** parâmetros `pQtdReducao` e `pDistReducaoPts`. Quando o trade andasse `pDistReducaoPts` a favor, o EA fecharia `pQtdContratos − pQtdReducao` contratos a mercado.

**Investigação em 15/05/2026:** após cinco tentativas distintas, a ordem a mercado parcial **nunca executou em produção**. Sintoma sempre o mesmo: o log `BG-CML: Reducao parcial ...` aparecia no console, mas nenhum evento de execução de mercado correspondente era registrado pelo Profit e a posição seguia inteira até o alvo ou stop fechar tudo.

Tentativas que falharam (todas em automação real, modo "Quando condição for satisfeita"):

| # | Abordagem | Resultado |
|---|---|---|
| 1 | `SellToCoverAtMarket(qty)` / `BuyToCoverAtMarket(qty)` direto | Absorvido pela OCO de cover orders |
| 2 | Idem + `CancelPendingOrders` antes do mercado parcial + skip stop/alvo no mesmo tick | OCO foi cancelada e recriada, mas o mercado parcial não executou |
| 3 | Atrelar qty da proteção a `Abs(PositionQty)` para evitar contratos descobertos | Funcionou — corrigiu sintoma residual, mas a redução continuou não executando |
| 4 | Trocar `*ToCoverAtMarket` por `SellShortAtMarket` / `BuyAtMarket` (manual 11.11: auto-cover quando enviada em posição oposta) | Sem `CancelPendingOrders` também falhou |
| 5 | Skip da emissão de stop/alvo durante o **candle inteiro** da redução (variável `nBarraReducao`) | Não validado — feature removida antes |

**Decisão (15/05/2026):** remover a feature inteira. O engine de automação do Profit (na versão usada) não aceita de forma confiável ordens a mercado parciais enquanto há OCO de cobertura ativa, e nenhuma das estratégias acima de "isolar" o mercado parcial do bundle OCO funcionou. A gestão de risco fica por conta do **stop fixo + breakeven gradual + trailing stop**, que já funcionam corretamente.

**Mantido:** a quantidade das ordens de cobertura continua sendo emitida com `Abs(PositionQty)` (ganho da tentativa 3) — torna o código robusto a ajuste manual da posição via UI do Profit.

**Removidos no código:** parâmetros `pQtdReducao` e `pDistReducaoPts`; variáveis `bReducaoFeita` e `nBarraReducao`; bloco inteiro da redução parcial em `IsBought` e `IsSold`; logs `BG-CML: Reducao parcial COMPRA/VENDA`.

### 10.7 Default de `pDistAcionaTrailing`

`pDistAcionaTrailing` default ficou em 150 pts (antes era 200 pts) — definido durante a fase em que a redução parcial existia e ainda faz sentido para o trailing assumir o controle logo após o breakeven completar.

### 10.8 Validação de timeframe (feature REMOVIDA em 22/05/2026) e proximidade das médias na faixa

**Validação de timeframe — REMOVIDA.** Foi adicionada em 16/05/2026 e removida em 22/05/2026. Histórico abaixo apenas para registro.

**Validação de timeframe (16/05/2026, removida em 22/05/2026):** novo parâmetro `pTimeframe` (default 1 min). O EA passava a comparar a cada execução com `BarDuration()` (manual NTSL 16.2). Se o gráfico estivesse em timeframe diferente do esperado, o EA fechava posição aberta, bloqueava toda operação e logava uma única vez `BG-CML: Timeframe incorreto ...`.

**Proximidade das médias como segundo gatilho da faixa:** novo parâmetro `pPercProximidadeMedias` (default 0.005, equivalente a 0,005% do `Close`). A cada nova vela, o EA verifica se `Abs(M1_ant − M2_ant) ≤ Close × pct / 100`. Se sim, isso conta como **evento equivalente a um cruzamento** e dispara a mesma lógica de formação/reconstrução da faixa de consolidação (4.1). Motivação: em mercados lentos as médias podem ficar coladas sem cruzar de fato, e o usuário visualmente já interpretaria como consolidação. A regra do cruzamento real continua valendo — proximidade é mais um gatilho, não substitui o existente.

**Variáveis novas no código:** `bTimeframeOK`, `bLogTimeframeFeito`, `vLimiteProximidade`, `bMediasProximas`. **Novo bloco 2.5** (verificação de timeframe) inserido entre os blocos 2 e 3.

**Correção aplicada em 21/05/2026 — `BarDuration()` precisa de parênteses:** a primeira versão usou `BarDuration` sem parênteses na comparação `bTimeframeOK := (BarDuration = pTimeframe)`. O Profit aceitou na verificação de sintaxe e a estratégia foi habilitada normalmente, mas em runtime **nenhum log apareceu por mais de uma hora de pregão** — o EA ficou completamente inerte. Hipótese: NTSL tratou `BarDuration` (sem parênteses) como referência de função em vez de chamada, e a tentativa de comparar/operar com isso travou silenciosamente a execução. Corrigido para `BarDuration()` nas duas ocorrências do bloco 2.5. Lição: funções da seção 16 do manual (Calendário) **sempre** precisam de parênteses, mesmo sem argumentos.

**Segunda correção em 21/05/2026 — comparação Integer-Float falha silenciosamente:** mesmo após colocar os parênteses, o EA ficou bloqueado em todas as novas velas com `TimeframeOK=False` no log de `Armacao bloqueada`. Causa: `BarDuration()` retorna **Integer** (1) e `pTimeframe(1)` cria um parâmetro **Float** (1.0). A comparação `(BarDuration() = pTimeframe)` em NTSL **não auto-promove o Integer para Float** — retornava False mesmo com os valores numericamente iguais.

**Tentativa intermediária (Round() nos dois lados) também falhou.** Causa: o manual NTSL (capítulo 13) define `Round(Valor: Float): Integer` — a função espera **Float** como entrada. Quando `Round(BarDuration())` é chamado, está-se passando um Integer para uma função que espera Float. NTSL falha silenciosamente em runtime, e a função retorna 0 (ou lixo). Daí `0 = Round(pTimeframe)` → False → bloqueio persiste.

**Correção definitiva:** usar **`BarDurationF()`** (manual seção 16.3) que retorna Float. A comparação `(BarDurationF() = pTimeframe)` é Float = Float e funciona nativamente em NTSL sem precisar de Round em nenhum lado.

```pascal
bTimeframeOK := (BarDurationF() = pTimeframe);
```

**Lição geral para NTSL:**
1. Funções têm assinatura estrita. Antes de envolver uma chamada em `Round()`/`Floor()`/`Ceil()`, **conferir no manual** se a função interna retorna Float (compatível) ou Integer (incompatível).
2. Quando uma família de funções existe em variantes `Integer` e `Float` (ex.: `BarDuration` / `BarDurationF`, `CurrentBar` / `CurrentBarF`), **escolher a variante cujo tipo case com o operando do outro lado da comparação**. Esse é o caminho mais limpo.
3. Para comparações com parâmetros de input (sempre armazenados como Float em NTSL), preferir as variantes `*F` das funções nativas.

**Remoção da feature em 22/05/2026:** mesmo após múltiplas tentativas (parênteses, `Round()`, `BarDurationF()`), o comparativo `bTimeframeOK` continuou retornando `False` em produção mesmo com chart 1Min e `pTimeframe = 1`. Causa-raiz não totalmente identificada — possivelmente precision Float ou armazenamento Integer do input. A feature foi removida inteira por decisão do trader: input, variáveis, bloco 2.5, lógica de bloqueio nos blocos 11/12/13/14 e campo `TimeframeOK` no log de diagnóstico. EA volta a operar em qualquer timeframe sem aviso. Os gotchas de NTSL aprendidos no caminho (parênteses obrigatórios em funções da seção 16, comparação Integer-Float falha silenciosamente, `Round()` exige Float) permanecem registrados no `CLAUDE.md` para uso futuro.

### 10.9 Inclinação forte da M1 antecipa viés (`pInclinacaoMediaRapida` / `pValorInclinacaoMediaRapida`)

**Motivação:** o cruzamento real de M1 com M2 às vezes acontece tarde — quando o preço já se moveu bastante na nova direção. A inclinação forte da M1 é um sinal mais antecipatório: quando a M1 está caindo (ou subindo) rápido, a reversão de tendência já está em curso antes do cruzamento se consumar.

**Mecânica (bloco 8):**

1. Novo input `pInclinacaoMediaRapida(True)` — Boolean. Toggle da feature.
2. Novo input `pValorInclinacaoMediaRapida(0.04)` — percentual do `Close` que define o limite (0,04% por default). Para WIN em ~178.000, dá ~71 pts.
3. Novas variáveis (var):
   - `vLimiteInclinacao` (Float) — magnitude limite em pontos.
   - `vInclinacaoM1` (Float) — `vM1_ant − vM1_ant2`, em pontos, com sinal.
   - `bInclinacaoAlta` (Boolean) — `True` se `vInclinacaoM1 > vLimiteInclinacao` (e feature ligada, e velas 1 e 2 do mesmo dia).
   - `bInclinacaoBaixa` (Boolean) — `True` se `vInclinacaoM1 < −vLimiteInclinacao` (mesmas defesas).
4. `bSinalCompra` e `bSinalVenda` reformulados com **priorização da inclinação em caso de conflito**:
   - `bSinalCompra := bInclinacaoAlta OR ((vM1 > vM2) AND NOT bInclinacaoBaixa)`
   - `bSinalVenda  := bInclinacaoBaixa OR ((vM1 < vM2) AND NOT bInclinacaoAlta)`
5. Novo log diagnóstico no bloco 8 (uma linha por nova vela quando ativo):
   - `BG-CML-DIAG: Inclinacao M1 ALTA forte (X pts, limite Y pts) - antecipa vies COMPRA`
   - `BG-CML-DIAG: Inclinacao M1 BAIXA forte (X pts, limite -Y pts) - antecipa vies VENDA`

**Comportamento da feature:**

- Cenário tranquilo (M1 e M2 alinhados, inclinação suave): viés segue o cruzamento regular (M1 vs M2). Comportamento idêntico ao anterior.
- Cenário de inclinação forte alinhada com o cruzamento (ex.: M1 acima de M2 + subindo forte): ambos confirmam compra. Sem mudança prática.
- Cenário de conflito (ex.: M1 acima de M2 + caindo forte): **inclinação ganha**. EA passa a operar do lado vendido, antecipando o cruzamento.
- Com `pInclinacaoMediaRapida = False`: feature desligada, viés volta a ser apenas M1 vs M2 (comportamento original).

**Escopo do diff aplicado:** apenas bloco `input`, `var` e bloco 8. Nenhum outro bloco tocado. Vela anterior alinhada continua sendo exigida na primeira armação, gatilho stop no rompimento da máxima/mínima continua, faixa de consolidação, breakeven, trailing, gestão diária — todos intactos.

### 10.10 Padrão de 3 velas como gatilho alternativo de viés (`pTradeVela3`)

**Motivação:** o cruzamento real (M1 vs M2) e a inclinação forte capturam tendências por médias móveis. Um padrão **bear-bull-bull** (ou bull-bear-bear no espelho) é uma assinatura clássica de reversão de preço — uma vela contra, seguida de duas a favor. Adicionar esse gatilho dá ao EA uma rota de entrada em reversões claras antes do cruzamento se consumar.

**Mecânica (bloco 8):**

1. Novo input `pTradeVela3(True)` — Boolean. Toggle.
2. Novas variáveis (var):
   - `bPadraoCompraV3 : Boolean` — `True` se feature ligada AND vela 3 negativa ou doji AND vela 2 positiva AND vela 1 positiva AND as 3 velas do mesmo dia.
   - `bPadraoVendaV3 : Boolean` — espelhado (vela 3 positiva ou doji, vela 2 negativa, vela 1 negativa).
3. `bSinalCompra` e `bSinalVenda` reformulados com priorização em 3 níveis:
   ```
   bSinalCompra := bInclinacaoAlta
                  OR (bPadraoCompraV3 AND NOT bInclinacaoBaixa)
                  OR ((vM1 > vM2) AND NOT bInclinacaoBaixa AND NOT bPadraoVendaV3)

   bSinalVenda  := bInclinacaoBaixa
                  OR (bPadraoVendaV3 AND NOT bInclinacaoAlta)
                  OR ((vM1 < vM2) AND NOT bInclinacaoAlta AND NOT bPadraoCompraV3)
   ```
   Priorização: **inclinação forte > padrão V3 > regular (M1 vs M2)**.
4. Novo log diagnóstico no bloco 8 (uma linha por nova vela quando o padrão estiver presente):
   - `BG-CML-DIAG: Padrao V3 COMPRA detectado (Vela3 neg/doji, Vela2 pos, Vela1 pos) - gatilho @ H1+offset`
   - `BG-CML-DIAG: Padrao V3 VENDA detectado (Vela3 pos/doji, Vela2 neg, Vela1 neg) - gatilho @ L1-offset`

**Gatilho de entrada:** mesmo do viés regular — `vHigh1 + pOffsetEntrada` para compra e `vLow1 − pOffsetEntrada` para venda. Como vela 1 é positiva no padrão de compra (e negativa no de venda), o filtro de vela anterior alinhada (`bVelaAntAlta`/`bVelaAntBaixa`) já passa naturalmente.

**Escopo do diff aplicado:** apenas `input`, `var` e bloco 8. Nenhum outro bloco tocado. Faixa de consolidação, gatilho stop, gestão de posição, gestão diária — todos intactos.

### 10.11 Trailing de proteção do lucro acumulado (`pLucroRef` / `vLucroAcumRef`)

**Motivação:** com a meta diária (`pMetaDiariaPts`) sozinha como teto, o EA encerra cedo em dias bons — basta tocar a meta e parar. Em dias muito favoráveis, isso deixa lucro na mesa. O trailing de proteção permite que o EA continue rodando após bater um lucro de referência (`pLucroRef`, default 1100 pts), e só pare quando o lucro acumulado cair para 70% do pico do dia (perda de 30%).

**Mecânica (bloco 4):**

1. Novo input `pLucroRef` (default 1100.0).
2. Nova variável `vLucroAcumRef` (Float) — pico do lucro acumulado do dia. Inicia em `pLucroRef` no novo dia, e sobe ao fechar nova vela se `vResultadoDia > vLucroAcumRef`.
3. Nova flag `bLucroRefAtingido` (Boolean) — `False` até `vResultadoDia` atingir `pLucroRef` pela primeira vez no dia. Quando isso acontece, **o trailing arma** e `pMetaDiariaPts` deixa de bloquear.
4. Bloqueio diário tem três condições independentes:
   - `vResultadoDia ≤ −pLossMaxDiarioPts` → sempre ativo.
   - Trailing armado E `vResultadoDia ≤ vLucroAcumRef × 0.70` → bloqueia (perda de 30% do pico).
   - Trailing **não** armado E `vResultadoDia ≥ pMetaDiariaPts` → bloqueia.

**Item 5 do requisito (priorização):** opção (A) escolhida — uma vez que o trailing arma, a meta diária deixa de bloquear. Como consequência, `vLucroAcumRef` pode ultrapassar `pMetaDiariaPts` naturalmente (lucro continua subindo passado o teto antigo).

**Logs novos:** `BG-CML: Lucro de referencia atingido ...` (uma vez por dia ao armar), `BG-CML: Lock diario - lucro caiu para 70% do pico (perda de 30%) ...`, `BG-CML: Lock diario - loss maximo diario atingido ...`, `BG-CML: Lock diario - meta diaria atingida ...`.

### 10.12 Fallback a mercado quando preço já passou do gatilho

**Motivação:** caso observado em 01/06/2026 — abertura do pregão com forte alta. O EA tentou enviar `BuyStop` várias vezes seguidas com mensagens `Não foi possível criar BuyStop - Sua ordem stop deve ter o preço acima do preço atual do mercado` (preço mercado 175.530, stop 175.520; preço 176.075, stop 175.840). O movimento explosivo de ~1.600 pts em poucos minutos foi inteiramente perdido — todas as tentativas de stop foram rejeitadas pelo broker porque o preço já estava acima do gatilho no momento do envio.

**Mecânica adicionada (bloco 14):**

Antes de enviar a ordem stop, o EA verifica:
- **Compra:** se `Close ≥ vGatilhoCompra` → dispara `BuyAtMarket(pQtdContratos)` em vez de `BuyStop`.
- **Venda:** se `Close ≤ vGatilhoVenda` → dispara `SellShortAtMarket(pQtdContratos)` em vez de `SellShortStop`.

**Controle de uma única tentativa por vela:**

- Novas variáveis `bMarketComprouEnviado` e `bMarketVendeuEnviado` (Boolean).
- Setadas para True após o envio do mercado.
- Resetadas a cada nova vela (bloco 7) — o gatilho recalcula com `Hi1`/`Lo1` da nova vela, dando uma nova oportunidade.
- Resetadas no início do dia e no CurrentBar=1 (defesas habituais).

**Trade-off conhecido:** entrada a mercado num spike pode pegar topo/fundo e levar stop loss rápido se o movimento for "fake". O trader aceitou explicitamente esse compromisso ao optar pela alternativa B na discussão.

**Não impacta:** a hierarquia de viés (inclinação > padrão V3 > regular) permanece intacta. O fallback opera **depois** que o viés já decidiu armar compra ou venda — só muda o tipo de ordem efetivamente enviada.

**Logs novos no bloco 14:**

- `BG-CML-DIAG: BuyAtMarket enviado (Close=X >= gatilho=Y) - rompimento intra-vela | Qtd=N | Hora ...`
- `BG-CML-DIAG: SellShortAtMarket enviado (Close=X <= gatilho=Y) - rompimento intra-vela | Qtd=N | Hora ...`

### 10.13 Correção: lock diário em pontos

`DailyResult(True)` retorna R$, mas os parâmetros de meta/loss são em pontos. Correção via `pValorPontoRS` (default 0.20 = WIN) e conversão:

```pascal
vResultadoDia := (DailyResult(True) - vBaselineResultado) / (pValorPontoRS * pQtdContratos);
```

### 10.14 Fork do BoldGlad-CM → BoldGlad-CML (criação do CML)

Criado a partir de cópia exata do BoldGlad-CM, motivado pelo pregão de 03/06/2026 (tendência de queda contínua) em que o CM perdeu R$4.450: metade das operações foram compras contra a tendência, com 38 entradas, 13% de acerto e 16 perdas seguidas; além disso reinícios do EA durante o dia desarmaram a trava de perda diária.

**Três mudanças aplicadas (escopo cirúrgico):**

1. **Filtro de tendência dominante** — inputs `pUsarFiltroTendencia(True)` e `pPeriodoTendencia(50.0)`; variáveis `vMediaTendencia`, `bTendenciaCompra`, `bTendenciaVenda`. Média calculada no bloco 5, plotada (Plot5) no bloco 6. No bloco 8, `bSinalCompra`/`bSinalVenda` recebem um AND final com a direção da tendência. Seção 2.1.3.
2. **Menos entradas** — defaults `pInclinacaoMediaRapida` e `pTradeVela3` mudados de `True` para `False`. Novo input `pVelasPausaAposSaida(3)` e variável `bEmPausaSaida` (calculada no bloco 10); blocos 13 e 14 passam a usar `bEmPausaSaida` no lugar de `CurrentBar = nBarraSaida`. O log do bloco 13 troca o campo `BarraSaida` por `PausaSaida`. Seção 3.6.
3. **Trava-dia à prova de reinício** — input `pResetarPlacarAoReiniciar(False)`. No bloco 4, a captura de baseline ao entrar em tempo real só ocorre se esse parâmetro for `True`; com `False`, o contador do dia reflete o `DailyResult` real do dia inteiro e a trava sobrevive a reinícios. Seção 6.3.

**Logs renomeados:** prefixo `BG-CM:` / `BG-CM-DIAG:` → `BG-CML:` / `BG-CML-DIAG:`.

**Não foi tocado:** faixa de consolidação (bloco 9), saídas forçadas (bloco 11), gestão de stop/alvo/breakeven/trailing (bloco 12), reset de gatilho (15), cancelamento de pendentes (16), plotagem da faixa (17) e sinalização visual (18). Todos os logs `BG-CML-DIAG:` existentes foram preservados.

**Para operar idêntico ao CM:** `pUsarFiltroTendencia = False`, `pInclinacaoMediaRapida = True`, `pTradeVela3 = True`, `pVelasPausaAposSaida = 1`, `pResetarPlacarAoReiniciar = True`.

### 10.15 Entrada de abertura (P1) e reentrada após lucro (P2)

Duas rotas de entrada novas, alinhadas às regras: a média de 50 (P3) é sempre a chefe; nenhuma das rotas opera contra ela.

**P1 — Entrada de abertura** (`pTradeAbertura`, default True). Bloco `input`/`var`, blocos 1/2 (reset do estado da abertura), bloco 7 (captura do topo/fundo da 1ª vela do pregão quando ela fecha), bloco 14 (ordem a mercado no rompimento do range, gated pela média de 50, uma vez por dia via `bAberturaUsada`). `pHoraInicio` mudado de 0905 para **0900** para a regra valer desde a abertura. Variável `bEntradaEnviadaNoTick` evita envio duplo no mesmo tick. Seção 3.7.

**P2 — Reentrada após lucro** (`pReentradaAposLucro`, default True). Bloco 12 grava direção (`nDirecaoPos`) e lucro aberto (`vUltimoLucroAberto`) a cada tick com posição; bloco 10 detecta, na saída, se o trade fechou positivo e arma a reentrada (`bReentradaArmada`, `nDirecaoReentrada`); bloco 13 ganhou um ramo `else if (bReentradaArmada)` que arma o gatilho na direção do vencedor furando a pausa e o filtro de vela anterior, mas exigindo viés (M1×M2) **e** média de 50 a favor — senão cancela a cadeia; bloco 14 libera o envio durante a pausa via `or bReentradaArmada`. Encadeia enquanto vencer; perde → volta ao normal. Seção 3.8.

**Decisões do trader (04/06/2026):** média de 50 prevalece sobre P1 e P2; "1ª vela do dia" = 09:00 (pregão), com `pHoraInicio = 0900`; reentrada arma no rompimento da vela que acabou de fechar; "lucro" = qualquer resultado > 0; reentrada exige viés a favor e encadeia enquanto vencer.

**Não foi tocado:** faixa de consolidação (bloco 9), saídas forçadas (bloco 11), matemática de stop/alvo/breakeven/trailing (bloco 12), reset de gatilho (15), cancelamento de pendentes (16), plotagem (17) e sinalização visual (18). Todos os logs `BG-CML-DIAG:` preservados; novos logs de P1 e P2 adicionados.

---

## 11. Pontos em revisão (aguardando aplicação)

### 11.1 Limpeza dos logs de diagnóstico restantes

Todos os logs `BG-CML-DIAG:` (Armacao bloqueada, Compra/Venda ARMADA, NÃO armada, DESARMADA, BuyStop/SellShortStop enviado) **estão ativos** no código e são parte essencial da observabilidade da estratégia. **Não remover sem confirmação explícita do trader** — uma tentativa anterior de remover o log `Armacao bloqueada` deixou o EA em um estado opaco em que um caso de não-entrada não pôde ser diagnosticado pelos eventos disponíveis.
