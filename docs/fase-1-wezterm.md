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

## tmux — persistência de sessão

Sessões de terminal (Fase 0) estão amarradas ao PTY que as criou: fechar o
terminal mata o PTY, e todo processo daquela sessão recebe SIGHUP e morre
junto. tmux resolve isso com uma arquitetura **cliente-servidor**: um
processo servidor independente, dono real dos shells/processos, e um
cliente (o que aparece na tela) que só se conecta/desconecta desse servidor.

```sh
sudo apt install tmux   # não requer PPA externo, pacote estável do Ubuntu
tmux -V                 # 3.6
```

### Modelo mental

```
[WezTerm] --PTY--> [zsh] --> [tmux client] ~~socket unix~~> [tmux server]
                                                                   │
                                                                   └── [zsh interno] --> [processo]

fechar a janela → sessão do WezTerm morre → tmux server continua,
processo interno sobrevive, dono de um PTY próprio gerenciado pelo tmux
```

### Comandos usados

- `tmux new -s <nome>` — cria sessão nomeada (inicia o servidor se não existir)
- `Ctrl+b` — prefix key: sinaliza ao tmux que a próxima tecla é um comando
  seu, não texto normal
- `Ctrl+b` `d` — detach: sai do cliente sem matar a sessão
- `tmux ls` — lista sessões vivas no servidor
- `tmux attach -t <nome>` — reconecta um cliente novo a uma sessão existente
- `tmux kill-session -t <nome>` — encerra sessão (e o processo interno)

### Verificação empírica (progressiva)

1. `tmux new -s teste` + `sleep 300` rodando em foreground
2. Detach (`Ctrl+b d`) → `tmux ls` mostra sessão viva, `ps aux` mostra
   `sleep 300` com TTY próprio do tmux (`pts/0`), independente do TTY do
   terminal externo
3. **Teste decisivo:** fechar a janela inteira do terminal original (não
   apenas detach) → abrir um terminal novo, inclusive um emulador
   diferente → `tmux attach -t teste` → `sleep 300` reaparece intacto,
   contando de onde parou

Isso confirma: o processo nunca dependeu do terminal que o criou, apenas do
servidor tmux, que roda independente de qualquer PTY externo.

### Limpeza

```sh
tmux kill-session -t teste
tmux ls   # "no server running" confirma encerramento sem processos órfãos
```

## WezTerm vs tmux — domínios de aplicação

WezTerm tem multiplexador próprio (`wezterm-mux-server`), mas é um recurso
**do binário WezTerm**, só existe onde o WezTerm está instalado. tmux é
independente de qualquer emulador gráfico específico, e está quase
universalmente disponível em servidores Linux remotos.

**Regra prática:** panes/tabs nativos do WezTerm para organização **local**
no próprio desktop. tmux especificamente ao conectar em máquinas remotas via
SSH, onde a persistência de sessão contra queda de conexão importa de fato
— o processo remoto sobrevive porque o servidor tmux roda do lado de lá,
indiferente à conexão do cliente.

---

## Fase 1 — fechada

1. Emuladores de terminal: papel puramente userspace, tradução PTY <-> pixels,
   herança de compatibilidade `xterm-256color`
2. WezTerm escolhido sobre Ptyxis (Lua compartilhado com Neovim futuro,
   maturidade em Linux); canal nightly em vez de stable (congelada desde 2024)
3. Estrutura de config (`~/.wezterm.lua`) como programa Lua executado, não
   dados declarativos -- `return config` é o mecanismo real
4. tmux: arquitetura cliente-servidor, sobrevivência de sessão comprovada
   empiricamente até com troca completa de terminal
5. WezTerm mux (local) vs tmux (remoto/SSH) -- domínios complementares, não
   concorrentes