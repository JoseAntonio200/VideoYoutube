# Larry9.2 - Setup 9.2 de Larry Williams / Stormer

## Visao Geral

Expert Advisor para a plataforma **Profit (Nelogica)** em linguagem **NTSL**, implementando o **Setup 9.2** popularizado por Larry Williams e amplamente difundido no Brasil por Alexandre Wolwacz (Stormer).

O setup opera **pullbacks a favor da tendencia** definida pelo cruzamento de medias moveis exponenciais (EMA 9 x EMA 21), buscando entradas com risco controlado e boa relacao risco/retorno.

---

## Logica da Estrategia

### Etapa 1 - Definicao da Tendencia (Setup 9.1)

| Condicao | Tendencia |
|---|---|
| EMA(9) > EMA(21) | **Alta** - buscar compras |
| EMA(9) < EMA(21) | **Baixa** - buscar vendas |

### Etapa 2 - Identificacao do Candle de Correcao (Setup 9.2)

| Direcao | Regra do Candle de Correcao |
|---|---|
| **Compra** | Candle cuja **minima e menor** que a minima do candle anterior (pullback em tendencia de alta) |
| **Venda** | Candle cuja **maxima e maior** que a maxima do candle anterior (repique em tendencia de baixa) |

### Etapa 3 - Entrada

| Direcao | Gatilho de Entrada |
|---|---|
| **Compra** | Ordem **BuyStop** no rompimento da **maxima** do candle de correcao + offset (5 pts) |
| **Venda** | Ordem **SellShortStop** no rompimento da **minima** do candle de correcao - offset (5 pts) |

### Etapa 4 - Gerenciamento

| Elemento | Compra | Venda |
|---|---|---|
| **Stop Loss** | Minima do candle de correcao - 5 pts | Maxima do candle de correcao + 5 pts |
| **Take Profit** | Entrada + (Risco x Multiplo) | Entrada - (Risco x Multiplo) |
| **Saida por inversao** | EMA9 cruza abaixo da EMA21 | EMA9 cruza acima da EMA21 |
| **Encerramento** | Posicao fechada apos horario limite | Posicao fechada apos horario limite |

### Filtros

- **Risco minimo**: 20 pts (evitar sinais em ruido)
- **Risco maximo**: 300 pts (evitar exposicao excessiva)
- **Horario de operacao**: 09:05 - 17:00
- **Encerramento forcado**: 17:30
- **Meta diaria**: para de operar ao atingir (default: 2000 pts)
- **Loss maximo diario**: para de operar ao atingir (default: 1000 pts)

---

## Diagrama de Fluxo

```
                     INICIO DO CANDLE
                           |
                    Calcular EMAs
                     EMA9 vs EMA21
                           |
              +------------+------------+
              |                         |
         EMA9 > EMA21             EMA9 < EMA21
         ALTA (compra)            BAIXA (venda)
              |                         |
     Min[1] < Min[2] ?          Max[1] > Max[2] ?
     (candle correcao)          (candle correcao)
              |                         |
          SIM: armar                SIM: armar
      BuyStop na Max[1]        SellShortStop na Min[1]
      Stop na Min[1]           Stop na Max[1]
              |                         |
     Rompeu gatilho?           Rompeu gatilho?
              |                         |
          COMPRADO                  VENDIDO
              |                         |
      +-------+-------+         +-------+-------+
      |       |       |         |       |       |
    STOP    ALVO   INVERSAO   STOP    ALVO   INVERSAO
     TP      TP      EMA      TP      TP      EMA
```

---

## Parametros de Entrada

### Medias Moveis
| Parametro | Default | Descricao |
|---|---|---|
| `pPeriodoEMA9` | 9 | Periodo da EMA rapida |
| `pPeriodoEMA21` | 21 | Periodo da EMA lenta |

### Gatilho de Entrada
| Parametro | Default | Descricao |
|---|---|---|
| `pOffsetEntrada` | 5.0 | Deslocamento em pts alem da extrema do candle de correcao |

### Gerenciamento de Risco
| Parametro | Default | Descricao |
|---|---|---|
| `pMultiploAlvo` | 1.0 | Multiplicador do risco para alvo: 1.0=1:1, 2.0=1:2, 3.0=1:3 |
| `pRiscoMinimo` | 20.0 | Risco minimo em pontos para aceitar trade |
| `pRiscoMaximo` | 300.0 | Risco maximo em pontos para aceitar trade |
| `pStopOffsetPts` | 5.0 | Deslocamento em pts para o stop alem da extrema |
| `pQtdContratos` | 1 | Quantidade de mini contratos |

### Horario
| Parametro | Default | Descricao |
|---|---|---|
| `pHoraInicio` | 0905 | Horario de inicio das operacoes |
| `pHoraFim` | 1700 | Horario limite para novas entradas |
| `pHoraEncerra` | 1730 | Horario de encerramento forcado |

### Gestao Diaria
| Parametro | Default | Descricao |
|---|---|---|
| `pMetaDiariaPts` | 2000.0 | Meta diaria em pontos |
| `pLossMaxDiarioPts` | 1000.0 | Perda maxima diaria em pontos |

---

## Backtest de Referencia

Dados: **WINJ26 - 09/04/2026** - Grafico de 1 minuto

### Resultados por cenario de alvo

| Metrica | 1:1 | 1:2 | 1:3 |
|---|---|---|---|
| Total de trades | 54 | 36 | 31 |
| Win Rate | **53.7%** | 33.3% | 32.3% |
| Ganho medio | +152 pts | +260 pts | +332 pts |
| Perda media | -135 pts | -138 pts | -138 pts |
| PnL Total | **+1.025 pts** | -55 pts | +555 pts |
| Fator de lucro | **1.30x** | 0.98x | 1.20x |
| Drawdown maximo | **785 pts** | 1.695 pts | 1.340 pts |

### Recomendacao

O cenario **1:1** (parametro `pMultiploAlvo = 1.0`) apresentou o melhor resultado neste dia, com:
- Maior PnL absoluto (+1.025 pts)
- Maior win rate (53.7%)
- Menor drawdown (785 pts)
- Melhor fator de lucro (1.30x)

---

## Como Usar

### Instalacao no Profit

1. Abrir o **Editor de Estrategias** no Profit
2. Criar nova estrategia do tipo **Execucao**
3. Copiar o conteudo do arquivo `Larry9.2.txt` para o editor
4. Clicar em **Verificar Sintaxe** para compilar
5. Aplicar no grafico de **1 minuto** do WIN/WINFUT

### Configuracao da Automacao

1. Apos compilar, clicar em **Automatizar Estrategia**
2. Configurar:
   - **Ativo**: WINFUT (ou contrato vigente, ex: WINJ26)
   - **Quantidade por Ordem**: conforme `pQtdContratos`
   - **Modo de Execucao**: Tick a Tick (recomendado)
3. Ajustar parametros conforme perfil de risco

### Parametros sugeridos por perfil

| Perfil | pMultiploAlvo | pRiscoMaximo | pQtdContratos | pMetaDiariaPts |
|---|---|---|---|---|
| Conservador | 1.0 | 200 | 1 | 1000 |
| Moderado | 1.0 | 300 | 5 | 2000 |
| Agressivo | 2.0 | 300 | 10 | 5000 |

---

## Estrutura do Codigo

```
Secao  1  - Reset no primeiro candle
Secao  2  - Deteccao de nova barra e novo dia
Secao  3  - Controle de horario
Secao  4  - Gestao diaria (meta e loss)
Secao  5  - Calculo das EMAs (MediaExp)
Secao  6  - Definicao da tendencia (EMA9 vs EMA21)
Secao  7  - Plotagem das EMAs no grafico
Secao  8  - Saidas (inversao EMA, horario, lock, stop/alvo)
Secao  9  - Leitura de dados anteriores (High[1], Low[1], etc.)
Secao 10  - Identificacao do candle de correcao
Secao 11  - Calculo dos gatilhos, stops e alvos
Secao 12  - Envio de ordens de entrada (BuyStop / SellShortStop)
Secao 13  - Sinalizacao visual (PaintBar, PlotText)
```

---

## Funcoes NTSL Utilizadas

| Funcao | Descricao |
|---|---|
| `MediaExp(periodo, serie)` | Media movel exponencial |
| `BuyStop(stop, limit, qty)` | Ordem de compra tipo stop |
| `SellShortStop(stop, limit, qty)` | Ordem de venda tipo stop |
| `SellToCoverStop(stop, limit)` | Stop de protecao para posicao comprada |
| `SellToCoverLimit(limit)` | Alvo limite para posicao comprada |
| `BuyToCoverStop(stop, limit)` | Stop de protecao para posicao vendida |
| `BuyToCoverLimit(limit)` | Alvo limite para posicao vendida |
| `ClosePosition()` | Encerra posicao aberta |
| `CancelPendingOrders()` | Cancela ordens pendentes |
| `IsBought()` | Verifica se esta comprado |
| `IsSold()` | Verifica se esta vendido |
| `HasPosition()` | Verifica se tem posicao aberta |
| `BuyPrice()` | Preco de entrada da compra |
| `SellPrice()` | Preco de entrada da venda |
| `DailyResult(bool)` | Resultado financeiro do dia |
| `PaintBar(cor)` | Pinta o candle no grafico |
| `PlotText(texto, cor, pos, tam)` | Texto no grafico |
| `Plot() / Plot2()` | Plota indicadores no grafico |
| `CurrentBar()` | Indice do candle atual |
| `Date` | Data do candle atual |
| `Time` | Hora do candle atual (formato HHMM) |
| `High[n] / Low[n]` | Maxima/Minima de n candles atras |

---

## Observacoes Importantes

1. **Backtest vs Tempo Real**: O backtest no Profit pode divergir do comportamento em tempo real devido ao reprocessamento de candles. Recomenda-se usar o modo **Tick a Tick** para maior fidelidade.

2. **Slippage**: Em automacao real, considerar que ordens stop podem ser executadas com slippage, especialmente em momentos de alta volatilidade.

3. **Custos Operacionais**: O setup pode gerar muitos trades no grafico de 1 minuto. Considerar corretagem, emolumentos e slippage ao avaliar o resultado liquido.

4. **Validacao**: Este EA foi desenvolvido com base na analise de um unico dia (09/04/2026). Recomenda-se fortemente realizar backtest em periodo extenso (minimo 3 meses) antes de operar ao vivo.

5. **Grafico Recomendado**: 1 minuto (conforme analise original), mas pode ser adaptado para 5 ou 15 minutos ajustando os periodos das EMAs.

---

## Historico de Versoes

| Versao | Data | Descricao |
|---|---|---|
| 1.0 | 2026-04-13 | Versao inicial - Setup 9.2 com alvo configuravel, filtros de risco/horario e gestao diaria |
