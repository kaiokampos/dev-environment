# Fase 1 — Emulação e multiplexação

## Emuladores de terminal — histórico e papel

Emuladores de terminal são programas 100% userspace que abrem o lado master
de um PTY, lançam um shell no lado slave, e traduzem bytes crus (incluindo
sequências de escape ANSI/VT100 herdadas dos anos 70-80) em pixels numa
janela gráfica — e teclado em bytes de volta para o shell.

A variável `TERM` declara qual "dialeto" de sequências de escape o emulador
suporta. Praticamente todo emulador moderno (Ptyxis, WezTerm, Ghostty,
GNOME Terminal) anuncia `xterm-256color` por compatibilidade histórica,
mesmo usando renderização GPU e linguagens modernas (Rust, Zig) por baixo.

**Verificação:** `echo $TERM` → `xterm-256color`, idêntico em Ptyxis e WezTerm.

## Decisão: WezTerm como emulador principal

Motivo: WezTerm é escrito em Rust, acelerado por GPU, com configuração em
Lua executável (não declarativa) — a mesma linguagem usada depois para
Neovim, reforçando uma única linguagem de configuração no ambiente. Suporte
maduro em Linux via Vulkan. Mantido lado a lado com o Ptyxis até estar
totalmente configurado e validado no uso diário.

## Correção de instalação: stable vs nightly

O pacote `wezterm` do repositório APT oficial (`apt.fury.io/wez`) está
travado na última tag stable, `20240203-110809-5046fc22` (fev/2024) — não é
repositório desatualizado, é o próprio projeto que raramente corta releases
estáveis, concentrando desenvolvimento ativo nas builds nightly.

**Decisão:** instalar `wezterm-nightly` (mesmo repositório, pacote
alternativo, não pode coexistir com o `wezterm` stable).

```sh
sudo apt remove wezterm
sudo apt install wezterm-nightly
```

**Verificação:** `wezterm --version` → build datado do dia da instalação
(ex: `20260713-...`), confirmando rastreio do canal ativo do projeto.

## Confirmação de ambiente correto: WEZTERM_PANE

`echo $TERM` sozinho não basta para confirmar que um comando está rodando
dentro do WezTerm — o valor é idêntico ao do Ptyxis (herança de
compatibilidade xterm). A variável `WEZTERM_PANE` só existe dentro de um
processo WezTerm, e é o critério objetivo correto.

**Verificação:**

```sh
echo $WEZTERM_PANE   # "0" dentro do WezTerm, vazio em qualquer outro terminal
echo $TERM           # xterm-256color nos dois -- não serve como diferenciador
```

## Estrutura de configuração (`~/.wezterm.lua`)

WezTerm trata o arquivo de configuração como um **programa Lua executado**
a cada início/reload, não como dados declarativos lidos. O script precisa
literalmente `return` uma tabela de configuração no final.

```lua
local wezterm = require("wezterm")

local config = wezterm.config_builder()

return config
```

- `require("wezterm")` — importa o módulo embutido no binário, expõe a API
  (`wezterm.font`, `wezterm.color`, etc).
- `wezterm.config_builder()` — helper que cria a tabela de config com
  validação de opções embutida (evolução do antigo `local config = {}`).
- `return config` — mecanismo real: o WezTerm usa o que a execução do
  script retorna, não apenas o que está escrito no arquivo.

**Verificação causal:** ausência de erro na janela não prova que o arquivo
foi lido (WezTerm cai em defaults silenciosamente se o arquivo não existir).
Prova real: adicionar `config.window_background_opacity = 0.5` e confirmar
efeito visual (janela translúcida) antes de remover o valor de teste.

---

**Nota registrada:** ao lançar o WezTerm pela primeira vez, apareceu um erro
não bloqueante de `mux::ssh_agent` ao tentar espelhar o `SSH_AUTH_SOCK` do
agente persistente configurado no `.zshrc`. Investigar na Fase 5
(Integração profissional / SSH).
