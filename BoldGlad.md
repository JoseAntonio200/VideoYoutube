**Nome do EA: BoldGlad10** 

**Descrição sumária**  
Este EA é um robô de execução em NTSL com proposta de operar de forma intrabar/Tick a Tick, usando duas camadas principais: direção por médias de sinal e bloqueio/liberação operacional por faixa de consolidação. Além disso, ele combina uma regra excepcional para a primeira barra do dia, reabilitação após dois cruzamentos, reversão por novo sinal oposto, stop fixo, alvo fixo, breakeven por steps, trailing pelo candle anterior e saída por retração do lucro máximo. Isso está explícito no cabeçalho e nos parâmetros do código.

## **1\. Arquitetura geral do EA**

A estrutura do código está organizada em blocos numerados de processamento: reset inicial, detecção de nova barra, leitura de médias e séries, reset diário, controle de horário e resultado diário, primeira operação do dia, sincronização de posição, formação e controle da faixa de consolidação, permissão final de operação, expiração de gatilhos, entradas/reversões, gestão da posição e gestão por retração do lucro máximo. Isso mostra um EA orientado a estado, e não apenas um conjunto simples de sinais de compra e venda.

A lógica foi escrita para trabalhar com reprocessamento de candle e atualização de ordens de cobertura, algo coerente com o funcionamento da NTSL e do Profit. O manual informa que o mesmo candle pode ser processado diversas vezes, e que, quando há aumento de posição, o fluxo pode ser reprocessado para ativar ordens de cobertura. Também informa que o modo Tick a Tick usa o fluxo real dos preços dentro do candle, com maior aderência operacional do que OHLC.

## **2\. Estratégia-base do robô**

A estratégia central do EA é:

1. Detectar contexto de tendência pelas médias de sinal rápida e lenta.  
2. Detectar consolidação por cruzamentos ou aproximação das médias de consolidação.  
3. Formar uma faixa de consolidação quando dois eventos de cruzamento/aproximação ocorrerem suficientemente próximos.  
4. Bloquear novas entradas enquanto o preço estiver dentro da faixa ou ainda não suficientemente afastado dela.  
5.   
6.   
7. Liberar operação quando o preço sair da faixa com distância mínima.  
8. Armar gatilhos de rompimento da máxima ou mínima da barra anterior com deslocamento.  
9. Executar entrada ou reversão a mercado quando o rompimento ocorrer e o alinhamento continuar válido.

Em termos práticos, é um EA de rompimento condicionado por tendência, mas com forte filtro de consolidação. Não é um seguidor de tendência puro, nem um robô de mean reversion. Ele tenta evitar operar “miolo de congestão” e só aceita operar após afastamento operacional da faixa.

## **3\. Parâmetros e função de cada grupo**

### **3.1. Consolidação**

* `pMediaConsol1(3)` e `pMediaConsol2(5)`: médias usadas para detectar cruzamentos de consolidação.  
* `pTolAproxCruzConsol(5.0)`: tolerância para considerar aproximação como equivalente a cruzamento.  
* `pDistanciaMaxCruz(100.0)`: distância máxima entre dois cruzamentos para validar formação da faixa.  
* `pLarguraFaixa(200.0)`: semi-largura da faixa.  
* `pDistSaidaFaixa(80.0)`: distância mínima para considerar que o preço saiu da faixa em condição operável.

### **3.2. Sinal direcional e entrada**

* `pMediaSinalRapida(5)` e `pMediaSinalLenta(9)`: definem o alinhamento direcional.  
* `pDeslocamentoEntrada(15.0)`: deslocamento sobre máxima/mínima da barra anterior para armar o gatilho de entrada.

### **3.3. Gestão da posição**

* `pAlvoCandleEntradaPts(300.0)`: alvo especial ainda no candle de entrada.  
* `pAlvoFixoPts(500.0)`: alvo fixo da operação.  
* `pStopPerdaTradePts(500.0)`: stop fixo por trade.  
* `pUsaProtecaoBE(True)` e `pStepBreakevenPts(50.0)`: ativam o breakeven por steps.  
* `pUsaTrailingCandle(True)`: trailing pela extrema da barra anterior depois do breakeven.  
* `pAlvoLucroPts(5000.0)` e `pPercRetracaoLucro(80.0)`: ativação e regra da saída por retração de lucro máximo.

### **3.4. Operação e travas**

* `pQtdContratos(10)`: lote padrão.  
* `pHoraInicio(0900)` e `pHoraFim(1730)`: janela de novas entradas.  
* `pMetaDiariaFin(5000.0)` e `pLossMaxDiarioFin(5000.0)`: trava financeira diária via `DailyResult(False)`.

### **3.5. Parâmetros mantidos por compatibilidade**

`pDistAcionaTrailing`, `pLucroMinProtecaoBE` e `pOffset` aparecem como “mantidos por compatibilidade”. Nesta versão, o trailing real passa a depender do estado do breakeven, e não desses parâmetros antigos. Isso é visível no comentário dos inputs e no bloco de gestão.

## **4\. Fluxo operacional do EA**

### **4.1. Reset inicial e reset diário**

Na primeira barra do histórico e a cada troca de data, o EA zera praticamente todos os estados internos: flags de consolidação, gatilhos armados, controle da primeira operação do dia, estado de posição, stops, alvo, lucro máximo aberto, contadores de fechamento fora da faixa e logs. No reset diário ele também cancela ordens pendentes.

Isso é importante porque o robô depende fortemente de memória de estado. Sem esse reset, uma faixa de consolidação, um gatilho ou uma trava poderiam “vazar” de um dia para o outro.

### **4.2. Controle diário e horário**

O EA consulta `DailyResult(False)` e trava o operacional quando atinge a meta diária financeira ou a perda máxima diária. Também considera:

* `bHora`: pode abrir novas operações apenas entre `pHoraInicio` e antes de `pHoraFim`.  
* `bOpera`: mantém operação geral até `pHoraFim`; depois disso cancela ordens e fecha posição.

Na prática, se o robô atingir a meta ou o loss do dia, ou se passar do horário final, ele cancela pendências e força zeragem se ainda houver posição.

## **5\. Primeira operação excepcional do dia**

Esse é um dos comportamentos mais característicos do EA. Na primeira barra do dia, se ainda não houve processamento dessa regra, se não estiver travado e se não houver posição:

* compra a mercado se `Close > vMS1_0`;  
* vende a mercado se `Close < vMS1_0`.

Depois dessa primeira entrada, o EA:

* marca que a primeira operação do dia foi feita;  
* ativa a espera por dois cruzamentos das médias de consolidação após o horário de entrada;  
* limpa os estados de faixa.

### **Leitura estratégica dessa regra**

Essa operação excepcional serve como uma “entrada inaugural” do dia, tentando capturar direção logo nos primeiros ticks/barra, antes do fluxo normal por rompimento da barra anterior voltar a valer.

### **Observação crítica**

Embora o cabeçalho resuma a primeira operação como ligada aos “primeiro(s) ticks em relação à média rápida”, o bloco efetivo usa apenas a comparação do `Close` com a média rápida. E, mais importante, **não exige** nesse trecho que a média rápida esteja acima da lenta para compra, nem abaixo da lenta para venda. Então, nesta versão do código, a primeira operação do dia é decidida apenas pelo preço versus média rápida.

## **6\. Reabilitação após dois cruzamentos**

Após a primeira operação do dia, o EA entra em estado `bAguardaDoisCruzPosHora = True`. Enquanto isso, ele não pode operar. Em cada nova barra, se estiver dentro do horário e ocorrer cruzamento de consolidação, ele conta:

* primeiro cruzamento;  
* segundo cruzamento, que zera a espera e libera novamente o operacional.

Essa regra tem uma função de “cooldown estrutural”: o EA não volta ao fluxo normal logo depois da operação inaugural; ele só reabre o operacional depois que o mercado gerar dois eventos de cruzamento das médias curtas de consolidação.

## **7\. Como a faixa de consolidação é formada**

A faixa nasce dentro de `bNovaBarra`. O EA detecta cruzamento das médias de consolidação de duas formas:

1. cruzamento clássico entre `[2]` e `[1]`;  
2. aproximação equivalente, quando a distância entre as médias em `[1]` fica menor ou igual a `pTolAproxCruzConsol`, desde que antes estivesse maior que essa tolerância.

Quando dois pontos de cruzamento/aproximação recentes existem e a distância entre eles é menor ou igual a `pDistanciaMaxCruz`, o EA:

* calcula o ponto médio;  
* define topo \= ponto médio \+ largura;  
* define fundo \= ponto médio \- largura;  
* marca `bEmConsolidacao := True`;  
* cancela ordens pendentes e desarma gatilhos.

### **Leitura estratégica**

A lógica não usa diretamente máxima/mínima de candles para formar a faixa. Ela usa o comportamento de duas médias curtas, o que faz da consolidação uma leitura de compressão/zigue-zague do preço, não uma caixa clássica de preço puro.

## **8\. Como a faixa bloqueia e libera operações**

Depois de formada a faixa, o EA trabalha com dois estados:

* faixa existente;  
* faixa operacionalmente liberada ou não.

Se o preço estiver dentro da faixa, ou ainda não tiver se afastado o suficiente, a operação fica bloqueada. A faixa só fica “liberada” quando:

* `Close >= topo + pDistSaidaFaixa`, ou  
* `Close <= fundo - pDistSaidaFaixa`.

Se, após liberada, o preço voltar para dentro da faixa, o EA:

* rebloqueia a faixa;  
* zera contadores de fechamento fora;  
* cancela ordens;  
* desarma gatilhos.

### **Consequência prática**

O EA pode continuar reconhecendo a existência da faixa e, ao mesmo tempo, permitir operar fora dela. Ou seja, não basta a faixa existir para bloquear; o bloqueio depende de o preço ainda não estar fora em condição operável.

## **9\. Como a faixa é cancelada definitivamente**

O código prevê **duas rotas** de desmontagem da faixa:

### **9.1. Regra antiga associada à entrada fora da faixa**

Quando uma posição é aberta com a faixa existente e liberada, o EA marca uma janela de confirmação. Depois, nas barras `entrada + 2` e `entrada + 3`, ele verifica:

* para compra: se a mínima da barra anterior ficou acima do topo da faixa;  
* para venda: se a máxima da barra anterior ficou abaixo do fundo da faixa.

Se a confirmação falhar, o cancelamento é abortado. Se se confirmar até a barra prevista, a faixa é cancelada.

### **9.2. Regra nova por 3 fechamentos consecutivos fora**

Em cada nova barra, se a faixa existir, o EA conta fechamentos consecutivos:

* acima do topo; ou  
* abaixo do fundo.

Se chegar a 3 fechamentos consecutivos fora, a faixa é desmontada integralmente.

### **Leitura estratégica**

A faixa não some logo no primeiro rompimento. O robô exige persistência do mercado fora da zona, o que evita desmontar a faixa por falso rompimento isolado.

## **10\. Permissão final de operação**

A permissão final depende basicamente de três fatores:

* não estar aguardando os dois cruzamentos pós-primeira operação;  
* se houver faixa, estar fora dela em condição operável;  
* estar dentro da janela horária e sem lock diário.

Se qualquer dessas condições falhar, o EA cancela ordens pendentes e zera gatilhos. Isso é um bloqueio ativo, não apenas passivo.

## **11\. Entradas normais do EA**

Fora a primeira operação do dia, as entradas normais só ocorrem se:

* não for o primeiro candle do dia;  
* estiver no horário;  
* não houver trava diária;  
* não estiver aguardando os dois cruzamentos;  
* houver permissão operacional da faixa;  
* o preço estiver efetivamente fora da faixa em condição operável.

### **Compra**

Se `vMS1_0 > vMS2_0`, o EA arma compra no rompimento de:  
`máxima da barra anterior + pDeslocamentoEntrada`.

### **Venda**

Se `vMS1_0 < vMS2_0`, o EA arma venda no rompimento de:  
`mínima da barra anterior - pDeslocamentoEntrada`.

### **Expiração do gatilho**

Se o gatilho armado não executar até passar a barra de sinal, ele expira, é cancelado e limpo.

### **Leitura estratégica**

O robô opera rompimento da barra anterior com filtro direcional por médias. Não entra simplesmente porque a média cruzou; precisa haver disparo de preço.

## **12\. Reversões**

Se houver posição vendida e surgir disparo válido de compra, o EA envia `BuyAtMarket(nQtdRev)`, com `nQtdRev = 2 * pQtdContratos`. O mesmo ocorre na direção oposta para reverter compra em venda.

A reversão só é permitida se **não** estiver em consolidação. Se estiver, o código registra log de reversão bloqueada.

### **Leitura estratégica**

A reversão é agressiva e líquida: em vez de zerar e depois reentrar, o EA usa o dobro da mão para virar a posição numa única ação lógica. Isso é coerente com conta netting.

## **13\. Registro da posição e coberturas**

Ao detectar posição aberta e `vEntrada = 0`, o EA registra:

* preço de entrada (`BuyPrice` ou `SellPrice`);  
* alvo fixo;  
* stop fixo;  
* barra de entrada;  
* máxima e mínima do candle de entrada.

Depois, envia coberturas:

* comprado: `SellToCoverLimit` e `SellToCoverStop`;  
* vendido: `BuyToCoverLimit` e `BuyToCoverStop`.

Isso está alinhado ao manual da plataforma, que recomenda uso explícito de ordens `ToCover` para saída de posição e informa que elas são gerenciadas como OCO no automatizador.

## **14\. Saídas da posição**

### **14.1. Alvo no próprio candle de entrada**

Se, ainda no candle de entrada:

* compra atingir `entrada + pAlvoCandleEntradaPts`, zera;  
* venda atingir `entrada - pAlvoCandleEntradaPts`, zera.

### **14.2. Saída no candle seguinte por invalidação**

Na barra imediatamente seguinte:

* compra sai se perder a mínima do candle de entrada;  
* venda sai se romper a máxima do candle de entrada.

### **14.3. Alvo fixo e stop fixo**

Se nada disso ocorrer, a posição segue com alvo fixo e stop fixo pelas ordens de cobertura atualizadas.

### **Leitura estratégica**

Esse desenho cria três camadas de saída:

1. saída muito rápida no próprio candle;  
2. saída defensiva na barra seguinte;  
3. gestão contínua por alvo/stop/trailing/retração.

## **15\. Breakeven por steps**

O breakeven só funciona se `pUsaProtecaoBE = True` e `pStepBreakevenPts > 0`. Enquanto o stop ainda não tiver chegado ao preço de entrada, o EA vai aproximando o stop em degraus de `pStepBreakevenPts`. Quando o stop alcança ou ultrapassa a entrada, ele fixa no preço de entrada e marca o breakeven como armado.

### **Leitura estratégica**

Não é um breakeven disparado por um único limiar fixo. É uma progressão do stop por degraus até zerar o risco. Isso é mais sofisticado do que o breakeven simples.

## **16\. Trailing stop**

O trailing só entra em ação se:

* `pUsaTrailingCandle = True`;  
* o breakeven já estiver armado.

A atualização ocorre em nova barra:

* comprado: stop candidato \= mínima da barra anterior;  
* vendido: stop candidato \= máxima da barra anterior.

### **Leitura estratégica**

O trailing não começa cedo. Ele depende de a operação já ter, antes, eliminado o risco inicial pelo breakeven. Ou seja: primeiro protege o capital; depois tenta “carregar” o lucro.

## **17\. Gestão por retração do lucro máximo**

Esse módulo é separado da gestão de stop/alvo. Em nova barra, com posição aberta:

* mede o lucro com base no fechamento anterior versus entrada;  
* se atingir `pAlvoLucroPts`, ativa o modo de retração;  
* passa a guardar o maior lucro aberto;  
* calcula um nível de saída como `lucro máximo * pPercRet`;  
* se o lucro atual cair para esse nível, zera a posição.

### **Leitura estratégica**

Essa camada funciona como trava de lucro para movimentos mais longos. Exemplo: se a operação abrir bastante lucro e depois devolver demais, o EA encerra antes de devolver tudo.

## **18\. Papel do `bSkip`**

Após várias saídas por `ClosePosition`, o EA ativa `bSkip := True`. Isso impede reentrada imediata no mesmo contexto. Quando chega uma nova barra, o `bSkip` é resetado para `False`.

### **Leitura estratégica**

É um anti-overtrading de curtíssimo prazo. Serve para não zerar e reentrar instantaneamente no mesmo ruído.

## **19\. Logs de diagnóstico**

O código usa `ConsoleLog` em praticamente todos os eventos importantes:

* reset diário;  
* primeira operação do dia;  
* cruzamentos;  
* formação/liberação/rebloqueio/cancelamento da faixa;  
* bloqueio por meta/loss;  
* gatilhos armados, expirados e abortados;  
* entradas, reversões e saídas;  
* breakeven, trailing e retração.

Isso é coerente com o manual, que define `ConsoleLog` como ferramenta de depuração e acompanhamento do comportamento do código.

## **20\. Pontos fortes da estratégia**

Os principais méritos deste EA são:

* forte controle de contexto operacional por consolidação;  
* proteção contra operar “dentro do miolo” da congestão;  
* primeira operação do dia separada do fluxo normal;  
* reabilitação condicionada a dois cruzamentos;  
* reversão estruturada;  
* múltiplas camadas de proteção da posição;  
* travas de resultado diário e de horário;  
* alto nível de rastreabilidade via logs.

## **21\. Pontos de atenção e observações críticas**

### **21.1. Divergência entre comentário e lógica da primeira barra**

O resumo do EA sugere uma operação da primeira barra “pelos primeiro(s) ticks em relação à média rápida”, mas o código efetivo decide por `Close > média rápida` ou `Close < média rápida`. Além disso, não exige alinhamento da média rápida com a lenta nesse trecho. Isso pode gerar comportamento diferente do que o nome/resumo do EA sugere.

### **21.2. Consolidação não impede existência de operação fora da faixa**

O robô pode manter a faixa existente e ainda assim operar, desde que o preço esteja fora dela em condição operável. Portanto, “estar em consolidação” no código não significa “proibição absoluta de operar”; a proibição absoluta vale quando o preço ainda não se afastou o suficiente.

### **21.3. Dependência forte do modo de automação**

Como o EA foi pensado com comportamento intrabar, ele tende a ficar mais fiel no modo em que a condição é satisfeita durante a formação do candle. O manual alerta que esse modo não é compatível com backtest tradicional e que Tick a Tick é o que mais se aproxima da execução real.

### **21.4. Algumas variáveis foram mantidas por compatibilidade**

Há parâmetros antigos mantidos, mas a lógica atual relevante está centrada em stop fixo, breakeven por steps, trailing por candle e retração do lucro. Isso é bom para compatibilidade, mas pode gerar leitura confusa se alguém achar que todos os parâmetros afetam de fato a lógica corrente.

## **22\. Conclusão técnica**

O **BoldGlad10** é um EA relativamente sofisticado, com foco em:

* evitar entradas em consolidação;  
* operar rompimentos em favor do alinhamento das médias;  
* tratar de maneira especial a abertura do dia;  
* controlar retomada de operação após a primeira entrada;  
* gerenciar a posição por várias camadas de defesa e proteção de lucro.

Em essência, ele é um **robô de rompimento com filtro estrutural de consolidação e gestão progressiva de risco**. O núcleo estratégico é bom e bem organizado, mas a primeira operação do dia, nesta versão específica, merece atenção porque o comportamento real do código é mais simples do que a descrição sugere.

