# WinBoat: inicializacao automatica indesejada no taiwan_lin

Registro: 17 de outubro de 2026. Computador: `asus-taiwan` (`taiwan_lin`). Fuso: America/Sao_Paulo.

**Estado:** contêiner desligado e política de reinicialização do Docker alterada e verificada. O comportamento após um novo boot e a persistência da edição no arquivo Compose ainda não foram testados.

## 1. Sintoma e diagnóstico

O usuário percebeu no monitor de processos um processo `windows` executado como `root`, PID histórico `3340`, com aproximadamente **7,1 GB de RAM**, embora não tivesse aberto a interface do WinBoat naquele dia. O usuário encerrou esse processo pelo monitor de sistema, usando **Kill Process**, antes da coleta de comandos.

A investigação encontrou:

| Evidência | Resultado |
|---|---|
| `ps -p 3340` | Processo não encontrado após o encerramento manual |
| `docker ps -a` | `WinBoat` em `Exited (0)` |
| Imagem Docker | `ghcr.io/dockur/windows:5.14` |
| Nome do projeto Compose | `winboat` |
| Arquivo Compose | `~/.winboat/docker-compose.yml` |
| Política anterior | `on-failure` |
| Serviço Docker | `enabled` e `active` |
| Busca de entradas WinBoat em autostart e systemd | Nenhuma correspondência nos locais examinados |
| Boot do Ubuntu | 2026-10-17 10:15:32 (-03:00) |
| Início do contêiner | 2026-10-17 10:15:49 (-03:00) |
| Término do contêiner | 2026-10-17 20:03:27 (-03:00) |

O contêiner iniciou **17 segundos após o boot** e permaneceu ativo aproximadamente **9h48min**.

Nos registros do daemon Docker, a sequência foi `Starting docker.service`, `Restoring containers: start`, eventos de rede do WinBoat e `Loading containers: done`. O evento `container start` do Docker também foi observado às 10:15:49.

**Interpretação:** a evidência indica que o Docker iniciou o contêiner durante a restauração no boot. A política `on-failure` existente permite reinícios automáticos após falhas. Os registros disponíveis não demonstram de forma isolada por que o daemon considerou esse contêiner elegível para iniciar naquele boot; não se comprovou um serviço WinBoat separado.

## 2. Solução aplicada

A política do contêiner existente foi alterada sem remover o contêiner nem os dados da máquina virtual:

```bash
sudo docker update --restart=no WinBoat
```

Em seguida, o usuário abriu o arquivo Compose no editor:

```bash
gedit ~/.winboat/docker-compose.yml
```

Foi orientado a substituir:

```yaml
restart: on-failure
```

por:

```yaml
restart: "no"
```

**Limite da confirmação:** o usuário executou o `gedit`, mas não foi exibido o conteúdo do arquivo após salvar. Portanto, não há confirmação independente da persistência dessa segunda alteração.

### Conferência executada

```bash
sudo docker inspect WinBoat \
  --format 'Restart={{.HostConfig.RestartPolicy.Name}} | Status={{.State.Status}}'
sudo docker ps --filter name=WinBoat
```

Resultado observado:

```text
Restart=no | Status=exited
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES
```

Isso confirma que, no momento da conferência, o contêiner estava parado e sua política de reinicialização automática estava desativada.

## 3. Resultado, limites e acompanhamento

- O Docker permanece disponível; não foi desabilitado.
- A política `restart=no` impede que o mecanismo de reinicialização automática do Docker inicie esse contêiner.
- Isso **não impede** que o próprio aplicativo WinBoat ou outro comando explícito inicie a máquina virtual.
- A alteração no contêiner já foi confirmada; é recomendável conferir também se o arquivo Compose contém `restart: "no"`, para não restaurar a política antiga quando o Compose recriar o contêiner.
- Ainda não houve um novo boot documentado para demonstrar o comportamento na próxima inicialização.
- A existência de uma senha em texto simples no Compose foi identificada; **o conteúdo da credencial não foi incluído neste relatório**. Convém substituí-la se estiver exposta ou reutilizada.

### Conferência futura, sem alterações

```bash
grep -n 'restart:' ~/.winboat/docker-compose.yml
sudo docker inspect WinBoat --format 'Restart={{.HostConfig.RestartPolicy.Name}} | Status={{.State.Status}}'
sudo docker ps --filter name=WinBoat
```

Se após reiniciar o Ubuntu o contêiner voltar a iniciar sem solicitação, recolher os eventos Docker e os registros de boot antes de alterar novas configurações.

## 4. Glossário

### Diagnóstico
- `ps -p`: consulta processo pelo PID; `-o`: escolhe colunas da saída.
- `docker ps -a`: lista contêineres ativos e parados.
- `uptime -s`: informa o horário de inicialização do sistema.
- `journalctl -u docker`: lê registros do serviço Docker.
- `docker events --filter container=WinBoat`: consulta eventos associados ao contêiner.
- `systemctl is-enabled` / `is-active`: verifica habilitação no boot e estado atual do serviço.

### Configuração
- `sudo`: executa o comando com privilégios administrativos.
- `docker update --restart=no WinBoat`: desativa a política de reinício automático no contêiner existente.
- `gedit`: editor gráfico de arquivos de texto.
- `~`: diretório pessoal; `docker-compose.yml`: configuração do projeto Docker Compose.
- `restart: "no"`: política Compose que desativa reinícios automáticos; as aspas preservam o valor textual em YAML.

### Verificação
- `docker inspect`: mostra propriedades do contêiner.
- `--format`: escolhe os campos de saída.
- `docker ps --filter name=WinBoat`: lista contêineres em execução cujo nome corresponde ao filtro.
- `grep -n`: procura texto e informa o número da linha.
- `Status=exited`: contêiner parado; `Restart=no`: política de reinicialização automática desativada.
