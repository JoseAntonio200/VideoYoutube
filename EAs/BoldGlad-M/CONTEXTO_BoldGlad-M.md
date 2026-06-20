# Contexto do Projeto BoldGlad-M

Briefing inicial para continuidade do EA **BoldGlad-M**.

---

## Sobre o projeto

**Nome:** EA BoldGlad-M
**Plataforma:** ProfitUltra (Nelogica), NTSL — Tick a Tick
**Ativo de referência:** WIN (mini-índice)
**Tag de log:** `BG-M:`

**Arquivos:**
- `BoldGlad-M.txt` — código NTSL
- `BoldGlad-M_documentacao.md` — documentação da estratégia
- `CONTEXTO_BoldGlad-M.md` — este arquivo

---

## Estratégia (tick a tick por M3 + consolidação — 18/06/2026)

Opera **tick a tick** por uma média exponencial de sinal, com filtro de consolidação por 2 médias. **Bidirecional.**

- **3 médias:** M1(100) e M2(200) **só para consolidação** (faixa); **M3 = EMA(21)** = sinal.
- **Entrada por ROMPIMENTO (ordem stop, zerado, fora de consolidação, não na vela de uma saída):** compra = vela anterior **positiva** + **inclinação da M3 (`M3[1]−M3[2]`) > `pInclinacao`(15)** + rompimento da **máxima + `pDeslocamento`(10)**, com o nível **acima da M3**; venda = espelho. Só um lado armado por vez (`nLadoArmado`).
- **Saídas (tick a tick):** preço **cruza a M3** contra a posição **ou** **stop gain +1000** **ou** **stop loss −1000** **ou** horário **ou** trava-dia. (A saída antiga pela média rápida foi removida.)
- **Reentrada/reversão só na vela seguinte** à saída (`nBarraSaida`), seguindo a M3 (recompra mesmo lado após stop gain; inverte após cruzamento).
- **Piso de distância na reentrada (só após saída pela M3):** `pDistMinReentrada` (default 100 pts) — o nível do rompimento é empurrado para `max(High[1]+desloc, M3+100)` (espelho na venda), garantindo que a reentrada só dispare ≥100 da M3. Não atua após stop gain/loss/horário/trava. (Histórico: tentou-se congelar a M3 na saída — revertido; latch de `bExitM3Pendente` corrigido; bracket M3±100 da etapa anterior foi substituído pelo rompimento de vela.)
- **Trava de devolução do dia:** `pPercDevolucao` (default 20%) do pico, arma após `pLucroRef` — substitui os 30% fixos.
- **Consolidação ligada por padrão** (`pConsolicao = True`).
- **Gestão diária** interna em pontos + semeadura — logs "EA PAROU DE OPERAR".

---

## Histórico

1. **Fork do BoldGlad-CML1 (12/06/2026):** virou sistema de 2 médias (a mercado no fechamento).
2. **Reformulação por rompimento revertida (18/06/2026):** EMAs 9/21 + reentrada por rompimento; só operava 1x/dia em lateral — descartada.
3. **Tick a tick por M3 (18/06/2026):** desenho atual. Decisões: M3 fecha o trade ao cruzar; reentrada segue a média na vela seguinte (recompra após stop gain); stops 1000/1000; devolução 20%; consolidação ligada.

---

## Estado atual

Escrito e verificado localmente (begin/end 39/39; M3, stops, `nBarraSaida` e `pPercDevolucao` presentes; sem resíduo da saída pela M1). **Pendências:**

1. **Verificar sintaxe** no editor do Profit (Ctrl+A → tudo até o `end;` final; atenção ao nome `MediaExp`).
2. Validar em pregão/replay: frequência de entradas **tick a tick**, atuação do filtro de consolidação, e calibrar `pPeriodoM3`, `pStopGainPts`/`pStopLossPts` e `pPercDevolucao`.
3. Atenção ao alerta: stop loss 1000 = trava diária 1000 (uma perda cheia encerra o dia).
4. À medida que ajustar, documentar em `BoldGlad-M_documentacao.md`.
