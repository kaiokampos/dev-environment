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
