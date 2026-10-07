# `kz`

Namespace pessoal de comandos Linux usado neste projeto.

## Estrutura

A função `kz()` fica em `~/.bashrc` e despacha subcomandos para executáveis em:

```text
~/.local/bin/kz-<command>
```

Uso geral:

```bash
kz <command> [arguments]
```

Comandos disponíveis atualmente:

```text
apps
drive
update
```

`kz list` lista os subcomandos executáveis encontrados em `~/.local/bin`.  
`kz help`, `kz -h` e `kz --help` exibem a ajuda.

Se o subcomando não existir ou não for executável, `kz` retorna código 127.

## `kz apps`

Arquivo:

```text
~/.local/bin/kz-apps
```

Script Python 3 que cria um inventário do software instalado na máquina.

### Fontes consultadas

O script coleta dados de:

- `dpkg` / APT;
- Snap;
- Flatpak.

Para cada item tenta registrar:

- gerenciador;
- nome público;
- pacote ou ID;
- comando de terminal;
- caminho/endereço;
- volume;
- tamanho;
- data de instalação;
- última atualização.

As datas são obtidas, quando disponíveis, dos históricos de `dpkg`, Snap e Flatpak. Quando o histórico não permite determinar uma data, o script pode usar timestamp de arquivo como estimativa, marcado com `~`. Quando não há informação defensável, usa `N/D`.

### Saída

Diretório preferencial:

```text
/mnt/taiwan_com/kz_reports/apps/
```

Fallback, caso esse diretório não possa ser usado:

```text
~/kz_reports/apps/
```

Arquivos principais:

```text
apps_latest.md
apps_full_latest.md
apps_latest.tsv
```

Também são mantidas cópias datadas, por exemplo:

```text
apps_2026-10-07_130825.md
apps_full_2026-10-07_130825.md
apps_2026-10-07_130825.tsv
```

`apps_latest.md` é a visão compacta; `apps_full_latest.md` preserva todos os campos; o TSV é a versão estruturada para processamento.

Uso:

```bash
kz apps
```

## `kz update`

Arquivo:

```text
~/.local/bin/kz-update
```

Atualiza os principais gerenciadores de pacotes e executa limpeza posterior.

Uso:

```bash
kz update
```

Sequência atual:

```bash
sudo -v
sudo snap refresh
sudo apt update
sudo apt upgrade -y
flatpak update -y
sudo apt autoremove
sudo apt autoclean
apt list --upgradable
```

O script usa:

```bash
set -e
```

Portanto, interrompe a sequência se um comando retornar erro.

Observação: `apt upgrade` e `flatpak update` recebem confirmação automática com `-y`. O `apt autoremove` atualmente não recebe `-y`.

## `kz drive`

Arquivo:

```text
~/.local/bin/kz-drive
```

Cria um backup completo do namespace `kz`, valida o pacote e envia o resultado para o Google Drive via rclone.

Uso:

```bash
kz drive
```

### Backup local

```text
/mnt/taiwan_com/Backup/Linux/kz/kz-backup.tar.gz
```

### Backup no Google Drive

Remote do rclone:

```text
Gdrive:Tech/Linux/Scripts/kz-backup.tar.gz
```

Equivale a:

```text
Meu Drive
└── Tech
    └── Linux
        └── Scripts
            └── kz-backup.tar.gz
```

### Conteúdo do pacote

```text
kz-backup/
├── kz-function.bash
├── MANIFEST.txt
├── RESTORE.txt
├── SHA256SUMS
└── scripts/
    ├── kz-apps
    ├── kz-drive
    └── kz-update
```

O script:

1. lê a função `kz()` carregando o Bash interativo;
2. copia todos os arquivos `~/.local/bin/kz-*`;
3. cria um manifesto com máquina, usuário, data e comandos encontrados;
4. cria instruções de restauração;
5. calcula hashes SHA-256;
6. gera `kz-backup.tar.gz`;
7. testa a integridade do gzip;
8. extrai uma cópia temporária;
9. valida os hashes;
10. cria o diretório remoto, se necessário;
11. envia o pacote com `rclone copyto`;
12. confirma a cópia remota com `rclone lsl`;
13. remove os arquivos temporários ao terminar.

O script usa:

```bash
set -euo pipefail
```

e registra uma função de limpeza com `trap ... EXIT`.

## Criando novos subcomandos

Novo subcomando:

```bash
gedit ~/.local/bin/kz-example
```

Estrutura mínima:

```bash
#!/usr/bin/env bash

echo "Example"
```

Torne-o executável:

```bash
chmod +x ~/.local/bin/kz-example
```

Depois:

```bash
kz example
```

Como `kz drive` copia todos os arquivos `kz-*`, novos subcomandos executáveis passam a entrar no backup automaticamente na próxima execução de:

```bash
kz drive
```

# Glossário

## Namespace `kz`

`~/.bashrc`: arquivo de configuração do Bash do usuário.

`~/.local/bin`: diretório usado para executáveis pessoais do usuário.

`$HOME`: diretório pessoal do usuário.

`$1`: primeiro argumento recebido por uma função ou script.

`shift`: remove o primeiro argumento e desloca os demais.

`find`: procura arquivos no sistema.

`-maxdepth 1`: limita a busca ao diretório informado.

`-type f`: seleciona arquivos regulares.

`-name 'kz-*'`: seleciona nomes iniciados por `kz-`.

`-executable`: seleciona arquivos executáveis.

`sed 's/^kz-//'`: remove o prefixo `kz-` da listagem.

`sort`: ordena a saída.

`-x`: testa se um caminho é executável.

`return 127`: código tradicionalmente usado para indicar comando não encontrado.

## `kz apps`

`python3`: interpretador usado pelo script.

`dpkg-query`: consulta os pacotes registrados pelo dpkg.

`/var/log/dpkg.log*`: históricos utilizados para datas de instalação e atualização de pacotes dpkg.

`snap list --all`: lista revisões Snap instaladas.

`snap changes --abs-time`: consulta o histórico de operações do Snap.

`flatpak list`: lista aplicativos e runtimes Flatpak.

`flatpak history`: consulta o histórico de operações do Flatpak.

`TSV`: formato tabular separado por tabulações.

`N/D`: informação não disponível de forma defensável.

`~data`: data estimada a partir de timestamps locais.

## `kz update`

`sudo -v`: valida ou renova as credenciais administrativas.

`snap refresh`: atualiza pacotes Snap.

`apt update`: atualiza o índice de pacotes APT.

`apt upgrade -y`: instala atualizações APT, aceitando automaticamente as confirmações.

`flatpak update -y`: atualiza Flatpaks, aceitando automaticamente as confirmações.

`apt autoremove`: remove dependências que não são mais necessárias.

`apt autoclean`: remove do cache pacotes que não podem mais ser baixados.

`apt list --upgradable`: lista atualizações APT ainda disponíveis.

`set -e`: encerra o script quando um comando retorna erro.

## `kz drive`

`set -euo pipefail`: encerra em erros, uso de variável indefinida ou falha em pipelines.

`mktemp -d`: cria um diretório temporário.

`trap ... EXIT`: executa uma rotina quando o script termina.

`declare -f kz`: imprime a definição da função `kz`.

`cp -a`: copia preservando atributos.

`sha256sum`: calcula hashes SHA-256.

`sha256sum -c`: valida arquivos contra hashes previamente salvos.

`tar -czf`: cria um arquivo tar comprimido com gzip.

`tar -xzf`: extrai um arquivo tar comprimido com gzip.

`gzip -t`: testa a integridade de um arquivo gzip.

`rclone mkdir`: cria um diretório remoto quando necessário.

`rclone copyto`: copia um arquivo para um caminho remoto exato.

`rclone lsl`: lista arquivo remoto com tamanho e data.

`--progress`: mostra o andamento da transferência.
