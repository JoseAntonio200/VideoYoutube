# EA BoldGlad11 — Documentação da Estratégia

**Plataforma:** ProfitUltra (NTSL)
**Modo de execução:** Tick a Tick
**Tipo:** Estratégia de execução automatizada para o WIN (mini-índice)

---

## 1. Visão geral

O BoldGlad11 é um robô seguidor de tendência que opera com base no cruzamento de duas médias móveis aritméticas, confirmando a entrada com o rompimento da máxima ou mínima da vela anterior. O robô é bidirecional (compra e vende) e possui mecanismos de proteção contra mercados lateralizados (faixa de consolidação), reversão por inversão das médias, gestão de posição com breakeven e trailing stop, e controles de horário, meta diária e perda máxima diária.

A filosofia da estratégia é capturar movimentos direcionais limpos, evitando operar em regiões de indecisão (onde as médias ficam se cruzando perto uma da outra) e protegendo o resultado financeiro à medida que a posição evolui a favor.

---

## 2. Componentes principais

### 2.1 Médias móveis

O robô utiliza duas médias móveis aritméticas calculadas sobre o preço de fechamento:

- **M1 (rápida):** padrão de 5 períodos. É a média que reage mais rápido às mudanças de preço.
- **M2 (lenta):** padrão de 9 períodos. Serve como referência da tendência mais estável.

A relação entre M1 e M2 define o viés do robô:

- Se M1 está acima de M2, o viés é de compra.
- Se M1 está abaixo de M2, o viés é de venda.

### 2.2 Vela anterior como confirmação

Além do viés das médias, o robô exige que a vela imediatamente anterior tenha direção alinhada ao viés:

- Para compra: a vela anterior deve ser de alta (fechamento acima da abertura).
- Para venda: a vela anterior deve ser de baixa (fechamento abaixo da abertura).

### 2.3 Gatilho de rompimento

Apenas o viés e a vela anterior alinhada não são suficientes — o robô espera que o preço efetivamente rompa a máxima (compra) ou mínima (venda) da vela anterior, com um pequeno deslocamento de segurança (offset). Esse rompimento é o gatilho que envia a ordem stop ao mercado.

---

## 3. Lógica de entrada

### 3.1 Compra

A ordem de compra é armada quando, no início de uma vela nova, todas as condições abaixo estão presentes:

1. M1 está acima de M2.
2. A vela anterior é de alta.
3. Não há posição aberta.
4. O horário está dentro da janela permitida.
5. Não está em lock diário (meta ou loss máximo).
6. Não está em faixa de consolidação.
7. A vela atual não é a vela em que houve saída ou reversão.

Quando armada, a ordem é uma stop de compra com gatilho calculado como **máxima da vela anterior + offset**. A ordem é enviada a cada tick até ser executada, cancelada ou substituída.

### 3.2 Venda

A ordem de venda segue lógica espelhada:

1. M1 está abaixo de M2.
2. A vela anterior é de baixa.
3. As demais condições gerais (sem posição, horário OK, sem lock, sem consolidação, sem ser vela de saída/reversão).

A ordem stop de venda tem gatilho em **mínima da vela anterior - offset**.

### 3.3 Comportamento atual do gatilho

No estado atual, o gatilho é armado uma vez na vela em que as condições foram detectadas e permanece com o mesmo preço até ser executado, cancelado ou desarmado por mudança de viés. **Esse comportamento está em revisão** para passar a recalcular o gatilho a cada nova vela enquanto as condições continuarem válidas (rearme contínuo).

---

## 4. Faixa de consolidação

A faixa de consolidação é o mecanismo de proteção contra mercados lateralizados. Quando o preço fica oscilando perto da região onde as médias se cruzam, é comum que a M1 cruze a M2 várias vezes em pouco tempo, gerando sinais conflitantes. Para evitar isso, o robô identifica essas situações e suspende a operação até que o preço se afaste da região de indecisão.

### 4.1 Detecção do cruzamento

Um cruzamento é identificado quando a posição relativa entre M1 e M2 muda entre duas velas **já fechadas**. Especificamente, o robô compara as médias da vela que acabou de fechar (vela 1) com as médias da vela anterior a essa (vela 2). Se elas trocaram de lado entre essas duas velas, há cruzamento.

**A detecção roda apenas uma vez por vela**, no primeiro tick de cada vela nova. Isso garante que cruzamentos sejam eventos discretos e objetivos — ou o cruzamento aconteceu entre as duas últimas velas fechadas, ou não aconteceu. Não há detecção durante a vela em formação, eliminando falsos cruzamentos causados pela oscilação intra-vela da média rápida.

### 4.2 Preço de referência do cruzamento

Quando um cruzamento é detectado, o robô precisa gravar em memória um "preço" associado a ele, para depois poder medir a distância até o próximo cruzamento. Esse preço é calculado como o **ponto médio entre M1 e M2 no fechamento da vela do cruzamento**. Como as duas médias acabaram de passar uma pela outra nesse momento, o ponto médio representa bem o "local visual" onde o cruzamento aconteceu — que é o que faz sentido conceitualmente para medir distância entre cruzamentos.

### 4.3 Regras de memória dos cruzamentos

O robô mantém em memória o preço do último cruzamento registrado (uma única referência). Quando um novo cruzamento é detectado, a lógica é:

**Se não havia cruzamento anterior em memória** (início do dia ou memória recém-apagada): o novo cruzamento vira a referência. Apenas isso — nenhuma faixa é formada ainda.

**Se havia cruzamento anterior em memória:** calcula a distância entre o preço do cruzamento atual e o da referência.

- Se a distância é **menor ou igual a `pDistMaxCruz`** (padrão 50 pontos):
  - Se não há faixa ativa: forma a faixa de consolidação. Calcula topo e fundo. Bloqueia o robô.
  - Se já há faixa ativa: não forma nova faixa. A faixa atual continua intocada.
  - Em ambos os casos: o cruzamento atual vira a nova referência em memória.
- Se a distância é **maior que `pDistMaxCruz`**:
  - O cruzamento antigo é descartado da memória.
  - O cruzamento atual vira a nova referência em memória.
  - Nenhuma faixa é formada.

Essa lógica permite que cruzamentos muito distantes entre si (indicando movimento direcional) não sejam confundidos com consolidação, e que sequências de cruzamentos próximos formem faixa corretamente mesmo quando alguns cruzamentos intermediários ocorrem longe da região.

### 4.4 Cálculo do topo e fundo da faixa

Uma vez confirmada a formação da faixa, o robô faz uma varredura para determinar o topo e o fundo:

- A janela de varredura começa em **N velas anteriores ao primeiro cruzamento** (parâmetro `pVelasAntesPrimCruz`, padrão 5) e termina na **vela do segundo cruzamento** (vela já fechada).
- A varredura percorre cada vela da janela, registrando a maior máxima encontrada (que vira o topo da faixa) e a menor mínima encontrada (que vira o fundo da faixa).
- Se não há histórico suficiente (cruzamento muito cedo no dia/gráfico), o robô usa as velas que tiver disponíveis.
- A vela em curso não entra na varredura — apenas velas já fechadas.

### 4.5 Comportamento durante a faixa ativa

Enquanto a faixa está ativa:

- O robô não arma novos gatilhos de compra ou venda.
- Ordens pendentes são canceladas.
- Os candles são pintados de amarelo no gráfico para sinalização visual.
- O topo e fundo da faixa são plotados como linhas no gráfico (Plot3 e Plot4).
- Novos cruzamentos detectados continuam sendo **registrados na memória** (atualizando a referência), mas **não formam nova faixa** e **não afetam a faixa atual** (topo, fundo e bloqueio permanecem).

### 4.6 Desfazimento da faixa

A faixa é desfeita quando o preço de fechamento se afasta o suficiente do topo ou do fundo:

- Se Close > topo da faixa + `pDistSaidaFaixa`, a faixa é desfeita.
- Se Close < fundo da faixa - `pDistSaidaFaixa`, a faixa é desfeita.

Após o desfazimento, o robô volta a operar normalmente. A memória de cruzamentos não é limpa automaticamente — ela contém o último cruzamento registrado durante a faixa, e o próximo cruzamento depois da faixa será comparado com esse. Isso serve de proteção: se o preço saiu momentaneamente da faixa mas o mercado continua lateralizado, uma nova faixa pode se formar imediatamente após um novo cruzamento próximo ao último registrado.

---

## 5. Reversão por inversão das médias

Se houver uma posição aberta e as médias inverterem de posição relativa, o robô fecha a posição imediatamente:

- Comprado e M1 vira abaixo de M2 → fecha compra.
- Vendido e M1 vira acima de M2 → fecha venda.

Após a reversão, o robô bloqueia novas entradas no mesmo candle (para evitar reabrir posição instantaneamente). O gatilho na direção contrária é armado normalmente na próxima vela nova, se as condições estiverem presentes.

---

## 6. Gerenciamento da posição

Quando uma posição está aberta, o robô gerencia stop loss, take profit, breakeven e trailing stop continuamente.

### 6.1 Stop fixo e alvo fixo

Inicialmente, o stop é colocado a `pStopFixoPts` pontos contra a entrada e o alvo a `pAlvoFixoPts` pontos a favor. Esses são os limites básicos de proteção e ganho.

### 6.2 Breakeven por steps (opcional)

Se ativado pelo parâmetro `pUsarBreakeven`, o breakeven funciona em degraus:

1. Quando o lucro aberto atinge `pDistAcionaBE` pontos a favor, o stop é movido para `entrada + pStepBreakevenPts` (no caso de compra), garantindo uma pequena margem positiva.
2. A cada novo step alcançado (lucro atinge `pDistAcionaBE + N × pStepBreakevenPts`), o stop sobe mais um step.
3. O breakeven só se move a favor — nunca contra.

Para venda, a lógica é espelhada (stop desce a cada step).

### 6.3 Trailing stop por candle (opcional)

Se ativado pelo parâmetro `pUsarTrailing`, o trailing começa a operar quando o lucro aberto atinge `pDistAcionaTrailing`:

- Para compra: a cada nova vela, o stop é ajustado para `mínima da vela anterior - offset`, mas só se esse valor for maior que o stop atual (nunca afrouxa).
- Para venda: o stop é ajustado para `máxima da vela anterior + offset`, com a mesma regra de só apertar.

Breakeven e trailing trabalham juntos — o stop efetivo é sempre o mais protetivo dos dois.

---

## 7. Controles de tempo e proteções diárias

### 7.1 Horário de operação

O robô só arma gatilhos e abre posições dentro da janela definida por `pHoraInicio` (padrão 09:05) e `pHoraFim` (padrão 17:30). Quando atinge o horário de fim, qualquer posição aberta é fechada e ordens pendentes canceladas.

### 7.2 Meta diária e loss máximo

O robô monitora o resultado acumulado do dia em pontos:

- Se o resultado positivo atinge `pMetaDiariaPts`, o robô ativa o lock diário.
- Se o resultado negativo atinge `pLossMaxDiarioPts`, o robô ativa o lock diário.

Quando o lock diário está ativo, qualquer posição aberta é fechada e o robô não abre novas operações até o próximo dia (quando o lock é resetado automaticamente).

### 7.3 Reset diário

Sempre que o robô detecta uma mudança de data, ele reseta os controles do dia: lock diário, gatilhos armados, faixa de consolidação, flag de reversão, memória de cruzamentos e cancela ordens pendentes. As variáveis de breakeven e trailing são resetadas separadamente, sempre que o robô fica sem posição.

---

## 8. Sinalização visual e logs

### 8.1 Plotagens e marcações no gráfico

- **M1 e M2** plotadas como linhas (Plot e Plot2).
- **Topo e fundo da faixa de consolidação** plotados como linhas (Plot3 e Plot4) quando a faixa está ativa.
- **Candles pintados de verde** quando comprado.
- **Candles pintados de vermelho** quando vendido.
- **Candles pintados de amarelo** quando o robô está em faixa de consolidação (sem posição).
- **Texto "BUY" em verde** abaixo do candle quando o gatilho de compra está armado.
- **Texto "SELL" em vermelho** acima do candle quando o gatilho de venda está armado.

### 8.2 Mensagens no console

Os logs são gravados no console da plataforma (visíveis na janela de Eventos da Automação). As mensagens disponíveis incluem:

**Logs de cruzamento e faixa (operacionais):**

- `BG11: CRUZAMENTO_DETECTADO` — quando o robô identifica um novo cruzamento.
- `BG11: Preco cruzamento anterior` e `BG11: Preco cruzamento atual` — valores de referência usados no cálculo da distância.
- `BG11: PRIMEIRO_CRUZAMENTO - referencia gravada` — no primeiro cruzamento da sequência.
- `BG11: SEGUNDO_CRUZAMENTO - distancia entre cruzamentos` — para cruzamentos subsequentes, com o valor calculado.
- `BG11: Limite maximo (pDistMaxCruz)` — parâmetro atual.
- `BG11: Distancia maior que limite - cruzamento antigo descartado` — quando a distância excede o limite.
- `BG11: Referencia atualizada com este cruzamento` — quando a memória é sobrescrita.
- `BG11: Consolidacao on - Faixa de consolidacao detectada` — quando a faixa é formada.
- `BG11: Topo da consolidacao` e `BG11: Fundo da consolidacao` — valores do topo e fundo.
- `BG11: Consolidacao off - Faixa desfeita, EA liberado para operacao` — quando a faixa é desfeita.
- `BG11: REVERSAO - fechando compra/venda` — quando a posição é fechada por reversão das médias.

**Logs de diagnóstico (temporários, para investigação):**

A cada nova vela, os seguintes logs são gravados para permitir rastreamento do que o robô "vê" nas médias:

- `BG11: --- Nova vela ---` — marcador de início do bloco de diagnóstico.
- `BG11: M1_vela1`, `BG11: M2_vela1` — valores das médias na vela que acabou de fechar.
- `BG11: M1_vela2`, `BG11: M2_vela2` — valores das médias na vela anterior à que acabou de fechar.
- `BG11: >>> bCruzDetectado = TRUE` ou `BG11: bCruzDetectado = FALSE` — resultado da verificação de cruzamento entre as duas velas.

Esses logs de diagnóstico serão removidos após a conclusão da investigação em andamento sobre a detecção de cruzamentos.

---

## 9. Parâmetros de entrada

### 9.1 Médias móveis

| Parâmetro | Padrão | Descrição |
|---|---|---|
| `pPeriodoM1` | 5 | Período da média rápida M1 |
| `pPeriodoM2` | 9 | Período da média lenta M2 |

### 9.2 Gatilho de entrada

| Parâmetro | Padrão | Descrição |
|---|---|---|
| `pOffsetEntrada` | 5.0 | Deslocamento em pontos além da máxima/mínima da vela anterior |

### 9.3 Stop e alvo

| Parâmetro | Padrão | Descrição |
|---|---|---|
| `pStopFixoPts` | 150.0 | Stop loss fixo em pontos |
| `pAlvoFixoPts` | 500.0 | Take profit fixo em pontos |

### 9.4 Breakeven

| Parâmetro | Padrão | Descrição |
|---|---|---|
| `pUsarBreakeven` | True | Ativa o gerenciamento de breakeven |
| `pDistAcionaBE` | 200.0 | Pontos a favor para acionar o primeiro step de breakeven |
| `pStepBreakevenPts` | 50.0 | Tamanho de cada step do breakeven |

### 9.5 Trailing stop

| Parâmetro | Padrão | Descrição |
|---|---|---|
| `pUsarTrailing` | True | Ativa o trailing stop |
| `pDistAcionaTrailing` | 200.0 | Pontos a favor para acionar o trailing |

### 9.6 Quantidade

| Parâmetro | Padrão | Descrição |
|---|---|---|
| `pQtdContratos` | 10 | Quantidade de contratos por operação |

### 9.7 Horário

| Parâmetro | Padrão | Descrição |
|---|---|---|
| `pHoraInicio` | 0905 | Horário de início (formato HHMM) |
| `pHoraFim` | 1730 | Horário de término (formato HHMM) |

### 9.8 Gestão diária

| Parâmetro | Padrão | Descrição |
|---|---|---|
| `pMetaDiariaPts` | 2000.0 | Ganho máximo no dia em pontos (lock diário) |
| `pLossMaxDiarioPts` | 1000.0 | Perda máxima no dia em pontos (lock diário) |

### 9.9 Faixa de consolidação

| Parâmetro | Padrão | Descrição |
|---|---|---|
| `pDistMaxCruz` | 50.0 | Distância máxima entre dois cruzamentos para formar faixa |
| `pDistSaidaFaixa` | 50.0 | Distância para o preço se afastar da faixa e liberar operação |
| `pVelasAntesPrimCruz` | 5 | Quantidade de velas anteriores ao 1º cruzamento incluídas na varredura do topo/fundo |

---

## 10. Fluxo resumido de decisão

A cada tick processado pelo robô, a sequência de avaliações é aproximadamente esta:

1. Atualiza referências de barra atual e detecta se é nova vela ou novo dia.
2. Verifica horário e atualiza flag de lock diário (meta ou loss).
3. Calcula M1 e M2 da vela atual, da vela anterior (vela 1) e de duas velas atrás (vela 2).
4. Avalia sinais: viés das médias (tick a tick), vela anterior, e cruzamento (comparando vela 1 com vela 2).
5. Se é nova vela, grava log de diagnóstico com os valores das médias.
6. Se é nova vela e há cruzamento detectado:
   - Calcula o preço do cruzamento como ponto médio de M1 e M2 na vela 1.
   - Se havia cruzamento anterior em memória, calcula a distância.
   - Aplica as regras de memória (descartar antigo se distância grande; formar faixa se distância pequena e sem faixa ativa).
   - Atualiza a referência na memória.
7. Verifica desfazimento da faixa pelo preço ter se afastado.
8. Se está com posição e as médias inverteram, fecha posição (reversão).
9. Se está fora de horário ou em lock diário e tem posição, fecha posição.
10. Gerencia posição aberta: atualiza stop conforme breakeven e trailing, mantém alvo.
11. Se for nova vela, reavalia condições e arma gatilhos de compra ou venda.
12. Envia ordens stop conforme gatilhos armados.
13. Cancela ordens pendentes se não há posição nem setup ativo.
14. Atualiza sinalização visual (cores, textos, plotagens).

---

## 11. Histórico de ajustes recentes

### 11.1 Varredura para topo/fundo da faixa

Substituída a lógica antiga (que usava apenas a vela anterior a cada cruzamento) pela varredura de uma janela mais ampla, percorrendo até N velas antes do primeiro cruzamento até a vela do segundo cruzamento, registrando a maior máxima e menor mínima.

### 11.2 Detecção de cruzamento baseada em velas fechadas

A detecção do cruzamento roda apenas no primeiro tick de cada vela nova, comparando as médias de duas velas já fechadas (vela 1 versus vela 2). Essa abordagem elimina falsos cruzamentos intra-vela que ocorriam quando a M1 oscilava durante a formação de uma vela.

### 11.3 Preço de referência do cruzamento

Mudança da referência de preço do cruzamento: em vez de usar o `Close[1]`, passou a usar o ponto médio entre M1 e M2 no fechamento da vela do cruzamento. Essa alteração fez o preço do cruzamento corresponder ao "local visual" onde as médias se cruzam no gráfico, corrigindo distorções que apareciam em velas com corpos grandes.

### 11.4 Regras de memória dos cruzamentos

Implementada a lógica completa de gestão da memória:

- Distância maior que o limite descarta o cruzamento antigo, mas o atual vira a nova referência (não apaga tudo — mantém o mais recente para comparação futura).
- Durante faixa ativa, novos cruzamentos continuam atualizando a memória, mas não formam nova faixa nem afetam a atual.
- Quando a faixa se desfaz, o próximo cruzamento é comparado com o último registrado durante a faixa — protegendo contra mercados que "fingem" sair da consolidação e voltam.

### 11.5 Logs em português

Logs padronizados em português com informações detalhadas sobre os cruzamentos e faixas, incluindo valores numéricos dos preços e distâncias. Isso facilita o acompanhamento em tempo real e a análise posterior.

### 11.6 Logs de diagnóstico por vela

Adicionados logs temporários que disparam a cada nova vela, mostrando os valores de M1 e M2 das velas 1 e 2 e o resultado da verificação de `bCruzDetectado`. Esses logs são ferramenta de investigação para o problema de detecção de cruzamento em situações específicas observadas no mercado real.

---

## 12. Investigação em andamento

**Problema observado:** em 17/04/2026, entre 09:11 e 09:16, dois cruzamentos visualmente claros entre M1 e M2 foram identificados no gráfico do WIN, com distância aproximada de 16 pontos entre eles. Essa situação deveria ter formado uma faixa de consolidação, mas nenhum log de cruzamento apareceu no período.

**Hipóteses a verificar com os dados de diagnóstico:**

1. Os valores das médias capturados pelo EA batem com o gráfico nos momentos dos cruzamentos?
2. A comparação entre vela 1 e vela 2 está capturando corretamente a troca de posição relativa?
3. O bloco de detecção está sendo alcançado em todas as novas velas?

**Próximo passo:** rodar o EA, aguardar reproduzir uma situação similar, capturar os logs `--- Nova vela ---` com os valores de M1 e M2 e identificar objetivamente a causa.

---

## 13. Pontos em revisão (aguardando implementação)

### 13.1 Rearme contínuo do gatilho (Opção A)

Atualmente o gatilho é armado uma vez e fica congelado no preço calculado naquele momento. A revisão prevista é fazer o gatilho ser recalculado a cada nova vela enquanto as condições continuarem válidas, acompanhando o movimento do preço. Quando o setup deixar de valer, o gatilho será desarmado e a ordem pendente cancelada.

### 13.2 Varredura retroativa ao ligar o EA no meio do pregão

Funcionalidade prevista mas ainda não implementada: quando o EA é acionado com o mercado já em andamento, fazer uma leitura dos cruzamentos que aconteceram nas últimas velas do dia atual, aplicando toda a lógica de memória como se o EA estivesse ligado desde o começo. Ao final da varredura, o EA estará em estado coerente: sem faixa, com memória parcial, ou com faixa ativa se ainda estiver dentro dos limites.

Pontos já confirmados para a implementação:
- Janela de 30 velas para trás (ainda a confirmar se limitado a 30 ou "desde o início do dia").
- Simular o comportamento cronológico como se o EA estivesse ligado desde o começo.
- Faixas passadas já desfeitas são apenas histórico.
- Faixa ainda ativa no momento atual fica ativa quando a varredura termina.

### 13.3 Limpeza dos logs de diagnóstico

Após a conclusão da investigação, remover os logs extras de diagnóstico:

- `BG11: --- Nova vela ---`
- `BG11: M1_vela1`, `BG11: M2_vela1`, `BG11: M1_vela2`, `BG11: M2_vela2`
- `BG11: bCruzDetectado = TRUE/FALSE`
- `BG11: Preco cruzamento anterior`, `BG11: Preco cruzamento atual`
- `BG11: Referencia atualizada com este cruzamento`

Manter apenas os logs operacionais essenciais (cruzamento detectado, consolidação on/off, topo, fundo, reversão).
