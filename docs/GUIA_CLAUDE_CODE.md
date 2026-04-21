# Guia Rápido do Claude Code

Comandos e dicas práticas para quem está começando a usar o Claude Code no desenvolvimento do BoldGlad11.

---

## Abrir o Claude Code

```
cd C:\caminho\para\BoldGlad11
claude
```

(Substitua o caminho pelo da sua pasta de projeto)

---

## Comandos dentro do Claude Code

Depois de entrar no Claude Code, você digita no prompt. Além das conversas naturais, existem comandos especiais que começam com `/`:

| Comando | O que faz |
|---|---|
| `/help` | Lista todos os comandos disponíveis |
| `/resume` | Lista conversas anteriores nessa pasta para continuar |
| `/clear` | Limpa o histórico da conversa atual (começa do zero) |
| `/compact` | Compacta o histórico longo para economizar contexto |
| `/cost` | Mostra o custo/uso da sessão atual |
| `/login` | Faz login ou troca de conta |
| `/logout` | Sai da conta atual |
| `/terminal-setup` | Configura o terminal (ex: habilita Shift+Enter para nova linha) |
| `/exit` ou Ctrl+D | Sai do Claude Code |

---

## Primeira sessão com o Claude Code no projeto

Sugestão do que dizer logo ao entrar:

> "Leia os arquivos CONTEXTO.md e BoldGlad11_documentacao.md para entender o projeto. Depois, espere minhas instruções."

O Claude vai ler os arquivos, entender o contexto do EA, e ficar pronto para ajudar.

---

## Pedidos típicos que você pode fazer

### Para investigar algo

> "Leia o arquivo BoldGlad11.txt e me explique como funciona o bloco da faixa de consolidação"

> "Procure no código onde a variável vPrecoCruzAnt é atualizada"

### Para alterar o código

> "No BoldGlad11.txt, quero que o gatilho de compra seja recalculado a cada nova vela enquanto as condições continuarem válidas"

> "Remova todos os logs de diagnóstico que começam com 'BG11: ---'"

### Para git e GitHub

> "Faça o commit das mudanças com a mensagem 'Detecção de cruzamento em velas fechadas' e envie para o GitHub"

> "Me mostre o histórico dos últimos 5 commits"

> "Crie uma branch chamada 'rearme-continuo' e mude para ela"

### Para análise em geral

> "Veja o log de eventos que vou colar aqui e me diga o que aconteceu entre 09:11 e 09:16"

(E aí você cola o conteúdo)

---

## Diferenças em relação a este chat web

### O que o Claude Code consegue fazer (e eu aqui não)

- Ler e editar arquivos diretamente no seu computador.
- Executar comandos do terminal (git, ls, cd, etc).
- Fazer commits e push para o GitHub com suas credenciais locais.
- Rodar scripts, testes, comandos de build.
- Navegar por pastas, buscar arquivos com grep.

### O que eu consigo fazer aqui (e o Claude Code não faz tão bem)

- Ler imagens que você sobe na conversa (screenshots de gráficos, por exemplo).
- Gerar arquivos Word, Excel, PDF formatados.
- Pesquisar na web de forma simples.

Então, dependendo da tarefa, um ou outro faz mais sentido.

---

## Dicas práticas

### Contexto é rei

Quanto mais contexto você der no começo, melhor. Por isso vale a pena pedir que o Claude leia o CONTEXTO.md e a documentação antes de começar a trabalhar. Isso economiza muito ida-e-volta.

### Use o `/clear` entre tarefas diferentes

Se você terminou de resolver uma coisa e vai começar outra bem diferente, use `/clear`. Isso libera o contexto e o Claude começa com a mente limpa para a nova tarefa.

### Aprovação antes de executar

Por padrão, o Claude Code pede sua aprovação antes de executar comandos que alteram arquivos ou rodam no sistema. Você pode aceitar, recusar, ou pedir para modificar. Isso é uma proteção boa para não fazer besteira.

### Vale a pena ter um git inicializado

Se a pasta do projeto ainda não é um repositório git, vale inicializar antes de começar a trabalhar pesado:

```
git init
git add .
git commit -m "Estado inicial do projeto"
```

Assim, se o Claude fizer alguma mudança que você não goste, é fácil reverter com `git reset --hard HEAD~1`.

### O Claude pode se perder em sessões muito longas

Se a sessão ficar muito longa, o Claude pode esquecer coisas do começo. Use `/compact` para resumir o histórico sem perder muito contexto, ou `/clear` e recomece explicando brevemente onde estava.

---

## Sugestão de setup inicial na sua máquina

1. Crie uma pasta dedicada ao projeto, por exemplo `C:\Projetos\BoldGlad11`
2. Coloque dentro os arquivos:
   - BoldGlad11.txt
   - BoldGlad11_documentacao.md
   - CONTEXTO.md
   - ManualNTSL.pdf (opcional, mas ajuda)
3. Se for usar GitHub, inicialize o repositório na pasta:
   ```
   git init
   git add .
   git commit -m "Estado inicial"
   git remote add origin https://github.com/seuusuario/BoldGlad11.git
   git push -u origin main
   ```
4. Crie um atalho na área de trabalho que abre o PowerShell direto nessa pasta (botão direito → Novo → Atalho → escolha PowerShell, e nas propriedades defina a pasta de início).

Com isso, um clique no atalho, digita `claude`, e você está trabalhando.
