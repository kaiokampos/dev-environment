# Fase 0 — Fundamentos do terminal

## 1. TTY

Abstração do kernel para dispositivo de terminal, criada na era dos terminais
físicos ligados por cabo serial. É a camada que liga um dispositivo de
entrada/saída de texto aos processos que rodam "dentro" dele.

**Verificação:**

```sh
tty
# /dev/pts/0
```

## 2. PTY

Par virtual (master/slave) que simula um TTY sem hardware físico. O emulador
de terminal segura o lado master; o shell fala com o lado slave, achando que
está conversando com um terminal real. Essa ilusão permite que qualquer
programa de terminal rode sem modificação dentro de um emulador gráfico.

**Verificação:**

```sh
tty   # aba 1 -> /dev/pts/0
tty   # aba 2 -> /dev/pts/1
```

Números diferentes por aba confirmam alocação dinâmica de par PTY por sessão.

## 3. Shell

Processo comum (sem privilégio de kernel) sentado no lado slave do PTY, cujo
trabalho é ler bytes, interpretar como comandos, e decidir executar
internamente ou criar um processo novo.

**Verificação:**

```sh
echo $$
# 13612  (PID do próprio shell)
```

## 4. Processo

Instância de um programa em execução, criada via `fork` (duplica o processo
pai) + `exec` (substitui o código da cópia pelo novo binário). O pai nunca é
destruído nesse processo — apenas espera o filho terminar.

**Verificação:**

```sh
sleep 5 &
jobs -l
# [1]  + 21341 running    sleep 5
```

> Nota: `$!` captura o PID do último job em background, mas só é confiável se
> lido na mesma linha lógica do comando — background jobs de plugins
> assíncronos (ex: Zinit turbo mode) podem sobrescrever seu valor entre
> linhas. `jobs -l` é mais robusto para inspeção posterior.

## 5. Sessão

Agrupamento de todos os processos originados do mesmo terminal, identificado
por um SID fixo — o líder de sessão é o shell de login. Dentro da sessão
existem grupos de processos menores (um por pipeline/comando).

**Verificação:**

```sh
sleep 30 & ps -o pid,ppid,pgid,sid,tty,cmd
```

```
    PID    PPID    PGID     SID TT       CMD
  13612   12526   13612   13612 pts/1    /usr/bin/zsh
  21498   13612   21498   13612 pts/1    sleep 30
  21499   13612   21499   13612 pts/1    ps -o ...
```

Mesmo SID nos três processos; PGIDs distintos por comando/pipeline.

## 6. Controle de terminal

Mecanismo pelo qual o PTY marca um único grupo de processos como
"foreground" por vez — só esse grupo recebe sinais de teclado (Ctrl+C =
SIGINT, Ctrl+Z = SIGTSTP). É o que permite alternar entre primeiro e segundo
plano sem afetar o shell.

**Verificação:**

```sh
sleep 20
# Ctrl+Z
jobs -l
# [1]  + 21874 suspended  sleep 20
```

Processo suspenso, não morto; controle do terminal devolvido ao shell
imediatamente.

---

**Cadeia lógica da fase:** um emulador cria um **PTY** → o **shell** senta no
lado slave dele → cada comando digitado vira um **processo** via fork+exec →
todos os processos de um terminal pertencem à mesma **sessão** → dentro
dela, o **controle de terminal** decide quem ouve o teclado agora.
