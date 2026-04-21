# Contexto do Projeto BoldGlad11

Este arquivo serve como briefing inicial para continuidade do desenvolvimento do EA BoldGlad11 em sessões futuras, especialmente quando utilizando o Claude Code.

---

## Como usar este arquivo

Ao iniciar uma nova sessão com o Claude Code nesta pasta, peça para ele:

> "Leia os arquivos CONTEXTO.md e BoldGlad11_documentacao.md para entender o projeto antes de começarmos."

Isso dá ao Claude contexto completo sobre o que já foi feito e o que está em andamento.

---

## Sobre o projeto

**Nome:** EA BoldGlad11
**Plataforma:** ProfitUltra da Nelogica, linguagem NTSL
**Modo:** Tick a Tick
**Ativo de referência:** WIN (mini-índice)

**Arquivos do projeto:**

- `BoldGlad11.txt` — código atual do EA em NTSL
- `BoldGlad11_documentacao.md` — documentação descritiva completa da estratégia (ler primeiro para entender o EA)
- `CONTEXTO.md` — este arquivo (briefing para continuidade)
- `ManualNTSL.pdf` — manual de referência da linguagem NTSL (consultar quando tiver dúvidas de sintaxe)

---

## Preferência de comunicação

O autor do projeto prefere conversas **informais e focadas em comportamento**, não em sintaxe da linguagem. Quando explicar coisas ou propor mudanças, descreva em termos do que o robô faz (ou deixa de fazer) no mercado, não em termos de código.

Exemplo de como explicar:
- ✅ "Quando o preço atinge 200 pontos de lucro, o stop sobe um degrau"
- ❌ "A variável vStopAtual é atualizada no bloco condicional quando pLucroAberto >= pDistAcionaBE"

---

## Estado atual do EA

O EA está rodando em produção e funcionando corretamente nos seguintes aspectos:

- Cruzamento de médias detectado apenas em velas fechadas (vela 1 vs vela 2).
- Preço de referência do cruzamento = ponto médio entre M1 e M2 no fechamento da vela.
- Gestão de memória de cruzamentos implementada (descarte de cruzamento antigo quando distância > limite, atualização durante faixa ativa, etc).
- Varredura para topo/fundo da faixa com janela ampliada (5 velas antes do primeiro cruzamento).
- Logs em português com valores detalhados.
- Gerenciamento de posição com breakeven por steps e trailing stop.
- Controles de horário, meta diária e perda máxima diária.

Ver `BoldGlad11_documentacao.md` para descrição completa da estratégia.

---

## Situação em aberto no momento

**Investigação de falta de detecção de cruzamento em caso específico observado em 17/04/2026 entre 09:11 e 09:16.**

Visualmente no gráfico havia pelo menos dois cruzamentos claros entre M1 e M2, com distância aproximada de 16 pontos — o que deveria ter caracterizado faixa de consolidação. Porém, nenhum log `BG11: CRUZAMENTO_DETECTADO` apareceu no intervalo.

**Ação em andamento:** foram adicionados logs de diagnóstico temporários que disparam a cada nova vela, mostrando:
- Os valores de M1 e M2 da vela 1 (vela que acabou de fechar)
- Os valores de M1 e M2 da vela 2 (vela anterior a essa)
- Se `bCruzDetectado` resultou em True ou False

Esses logs permitem diagnosticar objetivamente se:
- Os valores das médias estão corretos comparados ao gráfico
- A lógica de comparação está funcionando
- O bloco de detecção está sendo alcançado

**Próximo passo:** rodar o EA, aguardar reproduzir uma situação de cruzamento visível sem detecção, capturar os logs desse período e analisar para identificar a causa.

---

## Pontos em revisão (aguardando aplicação)

### 1. Rearme contínuo do gatilho (Opção A)

O gatilho de entrada é armado uma vez quando o setup aparece e fica congelado no preço calculado. Queremos que passe a ser recalculado a cada nova vela enquanto as condições continuarem válidas, acompanhando o movimento do preço.

Quando aplicar: após a investigação da detecção de cruzamento ser concluída.

### 2. Varredura retroativa ao ligar o EA no meio do pregão

Quando o EA é acionado com o mercado já em andamento, queremos que ele leia os cruzamentos que aconteceram nas últimas velas do dia atual, aplicando toda a lógica de memória como se estivesse ligado desde o começo. Ao final da varredura, o EA estará em estado coerente.

Pontos já confirmados:
- Janela: 30 velas para trás (ainda a confirmar se limitado a 30 ou desde o início do dia).
- Simular o comportamento cronológico como se o EA estivesse ligado desde o começo.
- Faixas passadas já desfeitas são apenas histórico.
- Faixa ainda ativa no momento atual fica ativa quando a varredura termina.

### 3. Limpeza dos logs de diagnóstico

Depois que a detecção de cruzamento for confirmada como correta, remover os logs extras de diagnóstico (valores das médias por vela, "bCruzDetectado TRUE/FALSE", "Preco cruzamento anterior/atual", "Referencia atualizada com este cruzamento") para deixar o console mais limpo.

---

## Limitação importante conhecida da NTSL

Em algumas versões da plataforma, a função `FloatToString` não funciona corretamente em concatenações dentro de `ConsoleLog`. A solução é concatenar variáveis Float diretamente com o operador `+`, sem usar `FloatToString`. Exemplo:

```
ConsoleLog("Preco = " + vPreco);
```

E não:

```
ConsoleLog("Preco = " + FloatToString(vPreco, 0));  // pode gerar erro
```

Isso está documentado no manual oficial (exemplo da função ConsoleLog na seção 19.3).

---

## Convenções adotadas no código

- Parâmetros de entrada começam com `p` minúsculo (ex: `pPeriodoM1`).
- Variáveis globais começam com letra de tipo: `v` para Float, `b` para Boolean, `n` para Integer.
- Todos os logs são prefixados com `BG11:` para facilitar filtragem.
- Comentários em português, em blocos de chaves `{ }`.
- Estrutura dividida em blocos numerados (1. RESET, 2. DETECÇÃO DE NOVA BARRA, etc).

---

## Referências úteis

- Manual NTSL: arquivo `ManualNTSL.pdf` na mesma pasta
- Documentação da plataforma: https://www.nelogica.com.br
- Função ConsoleLog: seção 19.3 do manual
- Função Media: seção 25.8 do manual
- Controle de fluxo (If, For, While): seções 5.1 a 5.4 do manual
