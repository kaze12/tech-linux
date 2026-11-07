# iCloud Drive — sincronização bilateral offline (taiwan_lin)

**Máquina:** asus-taiwan (Ubuntu 24.04). **Rclone observado:** 1.75.2. **Testado:** 04/11/2026.

## Objetivo e arquitetura

| Caminho | Uso |
|---|---|
| `icloud:` | Remote iCloud, origem e destino das alterações |
| `/home/icm/Nuvens/icloud` | Montagem FUSE online via `rclone@icloud.service` |
| `/mnt/taiwan_com/Offline/icloud` | Cópia local completa na partição compartilhada NTFS3 |

**Sincronizar diretamente `icloud:` ↔ pasta Offline**, não a montagem FUSE ↔ Offline. A montagem já expõe o mesmo remote; não é uma segunda cópia independente. A sincronização periódica não é instantânea nem substitui backup: exclusões podem se propagar.

## Comandos pessoais

- `kz icloud-bisync preview`: simula a **reconciliação inicial** (usa `--resync-mode newer --dry-run`).
- `kz icloud-bisync init`: executa a reconciliação inicial real ou recupera o estado após erro crítico.
- `kz icloud-bisync run` (ou `kz icloud-bisync`): ciclo incremental normal, usado pelo agendamento.
- `kz icloud-sync`: script legado com `rclone copy icloud: ... --update`, somente nuvem → Offline; mantido, **não agendado**. Não execute em paralelo com o Bisync.

O executável é `~/.local/bin/kz-icloud-bisync`. O script usa o filtro `~/.config/rclone/icloud-bisync.filters`, logs em `/mnt/taiwan_com/Offline/.icloud_logs/`, histórico local em `/mnt/taiwan_com/Offline/.icloud_history/` e histórico remoto em `icloud:__kz_icloud_history/`. O estado incremental fica em `~/.cache/rclone/bisync/*.lst`.

Os filtros ativos excluem `/__kz_icloud_history/` e `/Pages/Note Taking.pages`. O teste `--check-access` usa `RCLONE_TEST` na raiz de ambos os lados.

Proteções configuradas: `--backup-dir1`, `--backup-dir2`, `--max-delete 10` (limiar percentual), `--check-access`, `--conflict-resolve none` (conflitos no ciclo normal), `--resilient`, `--recover`, `--max-lock 2m`, `--transfers 2`, `--checkers 4` e bloqueio simultâneo via `flock`.

**Atenção:** `init` prefere a versão com data de modificação mais recente quando os dois lados divergem. Isso não garante preservação de ambos os conteúdos conflitantes durante a inicialização. Não use `init` no timer; após cada execução inicial bem-sucedida, usar apenas `run`. 

## Automação

Arquivos de usuário:

- `~/.config/systemd/user/kz-icloud-bisync.service`
- `~/.config/systemd/user/kz-icloud-bisync.timer`

Timer: `OnCalendar=*:0/15`, `Persistent=true`, `AccuracySec=1min`; disparos nominais aos minutos 00/15/30/45. Serviço: `Type=oneshot`, exige `/mnt/taiwan_com` montada e executa `/usr/bin/systemd-inhibit --what=sleep:idle --mode=block ... kz-icloud-bisync run`. Entre execuções é normal aparecer `inactive (dead)`. Timers anteriores `kz-icloud-sync.timer` e `rclone-icloud-offline.timer` foram desabilitados.

### Diagnóstico do serviço

```bash
systemctl --user status kz-icloud-bisync.timer --no-pager
systemctl --user list-timers --all | grep kz-icloud-bisync
journalctl --user -u kz-icloud-bisync.service -n 30 --no-pager
tail -n 30 "/mnt/taiwan_com/Offline/.icloud_logs/icloud-bisync-$(date +%F).log"
```

Para executar manualmente o **ciclo real**, sem alterar o agendamento:

```bash
systemctl --user start kz-icloud-bisync.service
systemctl --user show kz-icloud-bisync.service -p Result -p ExecMainStatus
```

## Ocorrência: suspensão e falha HTTP/2 (04/11/2026)

A primeira inicialização real foi interrompida após o notebook entrar em suspensão. O journal registrou retomada às **14:18:37** e o Rclone registrou, às **14:24:20**, erro durante upload:

```text
http2: Transport received Server's graceful shutdown GOAWAY
Bisync aborted. Must run --resync to recover.
```

A suspensão foi confirmada, **mas não foi comprovada como causa do encerramento HTTP/2**. Não foi preciso desativar HTTP/2. O processo falhou sem gerar os arquivos normais de estado `.lst`. Após revisar a simulação, a recuperação foi realizada com:

```bash
systemd-inhibit --what=sleep:idle --mode=block \
  --why="iCloud Bisync initialization" \
  "$HOME/.local/bin/kz-icloud-bisync" init
```

Às **15:14:29**, o Rclone exibiu `Bisync successful` e criou `.path1.lst` e `.path2.lst`. Às **16:36:36**, o ciclo `run` pelo serviço terminou com `No changes found` e `Bisync successful`. O timer disparou às **16:45** e **17:00**, com conclusão normal registrada no journal.

Se ocorrer nova falha crítica: examinar logs; **não repetir `init` automaticamente**; verificar divergências no modo `preview` e somente então executar `init` protegido contra suspensão. Não manter outro cron/timer sincronizando os mesmos caminhos.

## Segurança e limites

Arquivos de autenticação (`rclone.conf`, tokens e senhas) **não** devem ser versionados. Os históricos de versões são úteis para recuperação, mas não oferecem proteção contra perda simultânea da nuvem e da partição local. O bloqueio `systemd-inhibit` não impede desligamento forçado nem falta de energia.

## Glossário

### Comandos
- `rclone bisync`: sincronização bilateral por comparação de estados.
- `rclone mount`: montagem FUSE online.
- `rclone copy`: cópia unidirecional, sem excluir arquivos extras do destino por padrão.
- `--resync-mode newer`: reconcilia divergências escolhendo a versão mais recente.
- `--dry-run`: simula sem transferir ou excluir.

### Proteções
- `--check-access`: verifica arquivo sentinela nos dois caminhos.
- `--max-delete 10`: aborta diante de percentual excessivo de exclusões.
- `--backup-dir1/2`: move versões antigas para históricos respectivos.
- `flock -n`: evita instâncias simultâneas.

### Automação
- `systemd-inhibit`: bloqueia temporariamente operações de suspensão/inatividade.
- `OnCalendar=*:0/15`: agenda quatro execuções por hora.
- `Persistent=true`: recupera disparo perdido do calendário, conforme condições do timer.
- `journalctl --user -u`: consulta log da unidade de usuário.
- `systemctl --user`: controla serviços da sessão.
