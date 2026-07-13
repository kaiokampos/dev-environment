# dev-environment

Construção, do zero e com entendimento profundo de cada camada, de um ambiente
de desenvolvimento terminal-first para engenharia de software.

Este repositório não documenta apenas _configurações_ — documenta o **motivo**
de cada ferramenta existir, o **problema** que ela resolve, e a **verificação
empírica** de que o entendimento está correto antes de qualquer config entrar
em produção no ambiente real (versionado separadamente via chezmoi).

## Metodologia

Todo tópico segue o mesmo ciclo:

1. Problema que motivou a criação da ferramenta/conceito
2. Modelo mental
3. Comando mínimo
4. Verificação empírica
5. Commit

## Progresso

- [x] **Fase 0 — Fundamentos do terminal**: TTY, PTY, Shell, Processo, Sessão,
      Controle de terminal. Ver [`docs/fase-0-fundamentos.md`](docs/fase-0-fundamentos.md).
- [x] **Fase 1 — Emulação e multiplexação**: WezTerm (canal nightly),
      estrutura de config em Lua, tmux, domínios WezTerm vs tmux.
      Ver [`docs/fase-1-wezterm.md`](docs/fase-1-wezterm.md).
- [ ] Fase 2 — Editor como ambiente (Neovim, Lua, LSP, DAP, Treesitter)
- [ ] Fase 3 — Ferramentas Unix e produtividade
- [ ] Fase 4 — Diagnóstico e baixo nível (strace, gdb, valgrind, perf)
- [ ] Fase 5 — Integração profissional (ssh, Git, automação, chezmoi)

## Ambiente base

Ubuntu 26.04 LTS · zsh + Zinit + Starship · Git com commits assinados via SSH ·
mise · VS Code / Zed · Podman rootless · dotfiles via chezmoi.
