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

## Estrutura de configuração (`init.lua` e XDG)

Neovim segue a XDG Base Directory Specification: config mora em
`~/.config/nvim/init.lua`, não solto na raiz do `$HOME` (padrão de
convivência entre ferramentas Linux, evita bagunça de dotfiles soltos).

```lua
vim.opt.number = true
```

`vim` é uma tabela global exposta automaticamente pelo Neovim (diferente do
`wezterm`, que precisa de `require`). `vim.opt` representa as opções do
editor.

**Verificação causal:** `vim.opt.number = true` liga numeração de linha --
efeito visual impossível de confundir com coincidência, mesmo princípio do
teste de opacidade do WezTerm na Fase 1.

### Modularização via `require`

```lua
-- init.lua
require("config.opcoes")
```

```lua
-- lua/config/opcoes.lua
vim.opt.number = true
```

Dentro de `require()`, o `.` representa caminho de pasta (não acesso de
tabela) -- `require("config.opcoes")` resolve para
`~/.config/nvim/lua/config/opcoes.lua`. A pasta `lua/` é raiz de busca
implícita, não entra no caminho escrito.

**Verificação:** após mover o conteúdo para `lua/config/opcoes.lua` e deixar
`init.lua` só com o `require`, o comportamento visual (números de linha)
permaneceu idêntico -- confirma que a modularização não quebra o
carregamento, só reorganiza onde o código mora. Essa estrutura é a base
esperada pelo Lazy.nvim (próximo tópico).

## Lazy.nvim — gerenciador de plugins

Instalar plugins manualmente (clonar em `~/.config/nvim/pack/...`) tem dois
problemas em escala: **startup lento** (todos os plugins carregam sempre,
mesmo os raramente usados) e **falta de reprodutibilidade** (sem lockfile
de versões, reinstalar do zero pode pegar versões diferentes). Lazy.nvim
resolve os dois: lazy loading (carregar plugin sob demanda -- por tipo de
arquivo, por comando) e `lazy-lock.json` (equivalente a `package-lock.json`
do npm).

### Bootstrap

```lua
local lazypath = vim.fn.stdpath("data") .. "/lazy/lazy.nvim"

if not vim.loop.fs_stat(lazypath) then
  vim.fn.system({
    "git", "clone", "--filter=blob:none",
    "https://github.com/folke/lazy.nvim.git",
    "--branch=stable", lazypath,
  })
end

vim.opt.rtp:prepend(lazypath)
```

- `vim.fn.stdpath("data")` -- caminho de dados do Neovim (`~/.local/share/nvim`,
  XDG data dir -- diferente de `~/.config`, que é só config)
- `vim.loop.fs_stat(caminho)` -- verifica existência no disco; `not` inverte
  para "só clona se ainda não existir"
- `vim.fn.system({...})` -- executa comando de sistema (tabela = lista de
  argumentos do `git clone`)
- `vim.opt.rtp:prepend(...)` -- adiciona ao runtime path (onde o Neovim
  procura plugins/scripts), no início da lista

### Ativação e primeiro plugin

```lua
require("lazy").setup({
  {
    "folke/tokyonight.nvim",
    priority = 1000,
    config = function()
      vim.cmd.colorscheme("tokyonight")
    end,
  },
})
```

Cada plugin é uma tabela: posição 1 (sem chave) = `usuario/repositorio` do
GitHub; `priority` controla ordem de carregamento (temas precisam carregar
cedo, evita flash de cor padrão); `config = function() ... end` roda depois
da instalação.

**Verificação empírica:** bootstrap confirmado via `ls` no diretório de
destino do clone (arquivos reais do Lazy.nvim presentes, não assumido).
Ativação confirmada via efeito visual causal: UI de instalação do Lazy.nvim
aparece, e o esquema de cores muda de fato (preto/branco padrão -> tons de
azul do tokyonight) -- mesma disciplina de prova causal usada na Fase 1
com `window_background_opacity`.

## Treesitter — parsing real em vez de regex

Highlight tradicional usa regex: reconhece padrões de texto locais, sem
noção de estrutura (quebra em strings multi-linha aninhadas, código dentro
de código). Treesitter constrói uma árvore sintática real (parse tree) --
mesma estrutura que compiladores usam -- permitindo ao Neovim entender
"isso é uma função", "isso é uma string dentro do corpo", não só padrões de
aparência. Essa árvore é reusada depois por LSP, navegação e refatoração.

### Correção 1: API `main` vs `master`

A branch `main` do `nvim-treesitter` é reescrita completa e incompatível: o
plugin só instala parsers agora, não ativa highlight -- isso virou
responsabilidade do próprio Neovim core (`vim.treesitter.start()`).

```lua
{
  "nvim-treesitter/nvim-treesitter",
  branch = "main",
  build = ":TSUpdate",
  config = function()
    local ts = require("nvim-treesitter")
    ts.install({ "lua" }):wait(300000)

    vim.api.nvim_create_autocmd("FileType", {
      pattern = { "lua" },
      callback = function()
        pcall(vim.treesitter.start)
      end,
    })
  end,
},
```

- `require("nvim-treesitter").install({...})` -- não mais
  `require("nvim-treesitter.configs").setup({...})`
- `install()` retorna um objeto `Task` **assíncrono** -- `:wait(timeout)`
  (método, sintaxe `:`) força espera síncrona, necessário em contexto de
  bootstrap
- `vim.api.nvim_create_autocmd("FileType", {...})` -- autocomando nativo do
  Neovim: registra "quando este evento acontecer, rode esta função"
- `pcall(vim.treesitter.start)` -- protected call, evita que uma falha
  (parser ainda não pronto) trave o Neovim inteiro

### Correção 2: Neovim 0.11.6 insuficiente

`branch = "main"` exige Neovim **0.12+**. A versão 0.11.6 (Ubuntu 26.04,
considerada "suficiente" na decisão original) não atende esse plugin
específico -- mesmo padrão de canal desatualizado visto no WezTerm.

**Decisão:** trocar o pacote APT pelo tarball oficial pré-compilado do
GitHub Releases (não PPA -- não mantido pela equipe do projeto, historicamente
anos desatualizado; não build from source -- sem necessidade).

```sh
sudo apt remove neovim
curl -LO https://github.com/neovim/neovim/releases/latest/download/nvim-linux-x86_64.tar.gz
sudo tar -C /opt -xzf nvim-linux-x86_64.tar.gz
sudo ln -sf /opt/nvim-linux-x86_64/bin/nvim /usr/local/bin/nvim
```

**Verificação:** `nvim --version` → `NVIM v0.12.4`

### Correção 3: `tree-sitter-cli` ausente

Dependência separada do motor genérico Tree-sitter (não é o parser de
linguagem em si), usada internamente para compilação. Distribuída via npm.

```sh
npm install -g tree-sitter-cli
```

**Diagnóstico usado:** `:checkhealth nvim-treesitter` -- ferramenta correta
para expor exatamente qual requisito está faltando, em vez de inferir
causas por tentativa e erro.

### Localização dos parsers na nova arquitetura

Diferente da API antiga (parsers dentro da própria pasta do plugin), a
branch `main` instala em um diretório separado, próprio do Neovim:

```sh
ls ~/.local/share/nvim/site/parser/   # lua.so
```

### Verificação final

Após as três correções: `nvim <arquivo.lua>` abre sem erro, highlight
aparece com cores diferenciadas para strings/palavras-chave, e o parser
compilado (`lua.so`) existe fisicamente no diretório correto -- prova dupla
(efeito visual + arquivo em disco), mesma disciplina usada desde a Fase 1.
