# Fase 2 — Editor como ambiente

## Neovim — modos de edição

Editores convencionais tratam o teclado como "modo de inserir texto" o
tempo todo; mover/deletar/copiar dependem de mouse ou combinações Ctrl+algo.
Vim/Neovim resolve isso com **modos**: no Modo Normal, cada tecla é um
comando (movimento, edição), não um caractere digitado. Comandos se
compõem numa gramática (`d` + `w` = apagar palavra), similar a pipelines
Unix.

Neovim não é "Vim com plugins" -- reescreveu o núcleo para expor uma API
real, viabilizando Lua nativo como linguagem de configuração/extensão e
LSP embutido (próximos tópicos).

### Instalação

Ubuntu 26.04 já traz Neovim 0.11.6 no repositório padrão -- versão
suficiente para o ecossistema moderno de plugins (Lazy.nvim, Mason,
Treesitter geralmente pedem 0.11.2+). Diferente do WezTerm, não há lacuna
real que justifique canal alternativo (PPA/AppImage/nightly).

```sh
sudo apt install neovim
nvim --version | head -1   # NVIM v0.11.6
```

### Modelo mental

```
[Modo Normal]  <-- default, teclas = COMANDOS
     │  tecla de comando (i, a, o, O...)
     ▼
[Modo Insert]  <-- teclado = texto normal
     │  Esc
     ▼
[Modo Normal]

[Modo Visual]   -- seleção vira alvo de comandos
[Modo Comando]  -- linha ':' para comandos internos (:w, :q, etc)
```

### Verificação empírica

Buffer vazio, digitado "hello" sem entrar em Insert manualmente:

- `h`, `e`, `l`, `l` -- comandos de movimento, sem efeito visível em buffer
  vazio (não há texto para mover)
- `o` -- comando "abrir linha nova + entrar em Modo Insert" -- confirmado
  pela mensagem `-- INSERT --` no rodapé da tela
- `Esc` -- retorno ao Modo Normal, mensagem `-- INSERT --` desaparece
- `:q!` -- Modo Comando, sai sem salvar

Isso prova, na prática, que teclas comuns no Modo Normal disparam comandos
reais (inclusive mudança de modo), não inserção de caracteres.

## Lua — por que o Neovim usa

Vim clássico usa Vimscript: linguagem só de configuração, sem tipos reais,
sem ecossistema, performance imprevisível para operações pesadas (ex:
análise de código em tempo real). Neovim embute um interpretador LuaJIT no
próprio binário -- configuração vira código real (variáveis, funções,
condicionais), reusável e testável, não sintaxe proprietária de editor.
Mesma linguagem já usada no `~/.wezterm.lua` (Fase 1) -- decisão deliberada
de reduzir número de linguagens de configuração no ambiente.

### Sintaxe mínima (testada via `:lua` dentro do próprio Neovim)

**Variável:**

```lua
local nome = "kaio"
```

`local` restringe escopo -- sem isso a variável vira global, risco de
colisão de nomes entre plugins. Vimos isso sem explicar já no
`local wezterm = require(...)` da Fase 1.

**Tabela:** único tipo de dado composto em Lua -- substitui array, dict,
struct e objeto. Toda configuração do Neovim (e do WezTerm) é, no fundo,
uma tabela.

```lua
local config = { largura = 80, altura = 24 }
config.largura      -- acesso por ponto
config["largura"]   -- acesso por colchete (obrigatório se a chave tiver
                     -- espaço ou caractere especial)
```

**Função:** funções são valores -- podem ser guardadas em variável, tabela,
passadas como argumento. Base do modelo "quando X acontecer, rode esta
função" usado em toda configuração de editor.

```lua
local function saudacao(nome)
  return "olá, " .. nome   -- ".." concatena string (não "+")
end
```

`return` é o mesmo mecanismo do `return config` visto no `~/.wezterm.lua`.
Lua fecha blocos com `end`, não chaves `{}`.

### Verificação

Todas as três testadas via `:lua <expressão>` dentro do próprio Neovim,
sem precisar de arquivo: `print(nome)` → `kaio`; `config.largura` → `80`,
`config["altura"]` → `24`; `saudacao("kaio")` → `olá, kaio`.
