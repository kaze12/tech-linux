| Etapa | Objetivo |
|---|---|
| 1. Observar o Excel em Windows Desktop | Separar falha do aplicativo de falha da janela RemoteApp. |
| 2. Preservar o trabalho | Salvar planilhas antes de encerrar aplicativos ou mudar a conexão. |
| 3. Corrigir conforme o resultado | Investigar RemoteApp ou Excel com um teste por vez. |

# WinBoat: congelamento do Excel em RemoteApp no taiwan_lin

Registro de 9 de outubro de 2026 — computador `asus-taiwan`, referido como `taiwan_lin`. Horário de referência: America/Sao_Paulo.

**Estado ao publicar:** houve recuperação temporária, seguida de recorrência. O teste com `/gfx:RFX` foi iniciado, mas o resultado visual e a repetição do gatilho ainda não foram confirmados pelo usuário. A causa interna e uma correção permanente permanecem sem confirmação.

## 1. Sintomas e sequência das intervenções

O usuário confirmou que somente a janela do Excel travou; Linux e WinBoat continuaram respondendo. Depois confirmou que o mesmo Excel respondia em Windows Desktop. Essas observações apontam para o caminho RemoteApp, pois o aplicativo e a máquina Windows funcionavam pelo desktop completo. Esse teste foi anterior à recorrência e não foi repetido depois dela. Ainda falta identificar o mecanismo específico na conexão/gerenciamento de janelas. Não há evidência suficiente para atribuir o problema diretamente ao Wayland.

O usuário esclareceu que a travada aconteceu na tela inicial de um arquivo novo, sem planilha de trabalho preexistente. Assim, não existe evidência de que um arquivo específico tenha provocado esta ocorrência.

Primeira recuperação: o usuário não conseguiu fechar a janela. Foi identificado o cliente FreeRDP exclusivo do Excel, PID 95389, e outra conexão FreeRDP, PID 212212. Com aprovação do usuário, foi enviado SIGTERM ao cliente Excel; ele permaneceu ativo após três segundos. Com nova aprovação, SIGKILL encerrou somente esse cliente. A verificação confirmou o cliente Linux do Excel encerrado, a outra conexão preservada e o contêiner Windows ainda ativo. Isso comprova um cliente que não respondeu ao encerramento normal; não determina a causa interna do congelamento. Não foi usado encerramento indiscriminado de todas as conexões FreeRDP.

Recuperação inicial: após a orientação para deixar Windows Desktop fechado e abrir Microsoft Excel pela lista Apps do WinBoat, o usuário confirmou que a nova janela responde. Encerrar o cliente FreeRDP preso e abrir uma nova conexão recuperou temporariamente o acesso, mas houve outra travada. Portanto, esse procedimento não resolveu a recorrência.

Na nova ocorrência, o usuário confirmou que abriu uma janela, menu ou aviso dentro do Excel imediatamente antes do congelamento e que não há alterações não salvas a preservar. Ainda falta o nome ou a sequência exata desse gatilho. Esse comportamento aproxima o caso dos relatos de falha no gerenciamento de popups RemoteApp; não comprova o mesmo defeito interno.

O novo cliente congelado, PID 235841, manteve conexão TCP estabelecida com o Windows e apresentou 16.185 bytes na fila de recebimento. Das 24 threads, a maioria aguardava eventos e duas aguardavam futex. Isso é compatível com uma paralisação no cliente, mas não identifica por si só um deadlock nem a função responsável. A coleta de pilhas não ficou disponível sem autenticação administrativa; nenhuma configuração de segurança foi alterada.

Com aprovação, somente esse cliente congelado foi encerrado, mantendo o Windows ligado. Foi aberta uma conexão temporária com `/gfx:RFX`, que seleciona RemoteFX e desabilita os caminhos AVC/H.264 e Progressive segundo o [parser FreeRDP 3.32.1](https://github.com/FreeRDP/FreeRDP/blob/3.32.1/client/common/cmdline.c#L2219). Esse teste separa a hipótese do codec gráfico da hipótese de gerenciamento de janelas. Não altera a configuração permanente do WinBoat.

A primeira tentativa de reproduzir o lançamento reutilizou a senha mascarada em memória pelo FreeRDP e terminou com falha de autenticação, código 134 (`ERRCONNECT_LOGON_FAILURE`). Isso não foi um novo congelamento. A tentativa corrigida utilizou as credenciais existentes do WinBoat sem exibi-las, permaneceu ativa após cinco segundos e chegou ao processamento de janelas RAIL. O resultado do teste de repetir o mesmo menu ou aviso ainda depende da observação do usuário.

Os avisos `xf_Pointer_get_window: invalid appWindow` e `xf_rail_monitored_desktop: TODO ...` encontrados nessa conexão não foram tratados como causa comprovada. No [código de ponteiro FreeRDP 3.32.1](https://github.com/FreeRDP/FreeRDP/blob/3.32.1/client/X11/xf_graphics.c#L229), a ausência de janela com foco no modo RemoteApp é prevista e o chamador pode retornar sucesso. No [tratamento do desktop monitorado](https://github.com/FreeRDP/FreeRDP/blob/3.32.1/client/X11/xf_rail.c#L990), esses avisos também não representam uma falha fatal por si só.

Todos os PIDs, filas, tempos e consumos abaixo são amostras históricas desta investigação. **Não reutilizar os PIDs registrados para encerrar processos numa nova ocorrência.**

## 2. Ambiente e evidências coletadas

| Item | Resultado | Interpretação |
|---|---|---|
| Sistema | Ubuntu 24.04.5 LTS, GNOME, sessão Wayland | O cliente usado é `xfreerdp`, integrado via XWayland. |
| WinBoat e agente Windows | 0.9.2 | Ambos estão na mesma versão. |
| FreeRDP efetivamente usado | Flatpak, 3.32.1 | Confirmado pelo comando de lançamento e pelo próprio executável. |
| Executável usado | `flatpak run --command=xfreerdp com.freerdp.FreeRDP` | Não havia cliente `xfreerdp` nativo no PATH na coleta. |
| Máquina Windows | Contêiner Docker `WinBoat`, imagem `ghcr.io/dockur/windows:5.14` | Backend identificado no ambiente existente. |
| Recursos atribuídos à VM | 8 GiB de RAM, 4 núcleos, disco virtual de 64 GiB | Nenhuma dessas alocações foi modificada. |
| Windows | Docker ativo há cerca de três horas; serviço de saúde respondeu HTTP 200 | A máquina e o serviço Windows não estavam completamente travados. |
| Encerramento por falta de RAM | `OOMKilled=false`; sem registro de OOM recente | Não foi detectado encerramento por falta de memória. Isso não exclui episódios de lentidão. |
| Windows: recursos | CPU 0% na amostra; RAM 2.437/8.187 MiB (29,8%); C: 66,8% ocupado | Não estava saturado na coleta. Uma amostra não descreve todo o período da falha. |
| Windows na recorrência | CPU 0%; RAM 5.429/8.187 MiB (66%); serviço saudável | O consumo aumentou, sem evidência de esgotamento nessa amostra. |
| Linux: recursos | RAM 23 GiB; 3,2–4,3 GiB disponíveis; swap 7,6–7,7 de 8 GiB usada | Pouca folga de swap; possível agravante, não causa comprovada. |
| Paginação e espera | Um pico de saída para swap (~23 MiB/s), seguido de zero; pressão recente de RAM 0,00% e I/O próxima de zero | Não apareceu pressão sustentada na pequena janela de observação. |
| Disco da VM | `/mnt/taiwan_com/winboat`, na partição compartilhada NTFS | Já segue a preferência de armazenamento compartilhado. Sem evidência para recomendar migração agora. |
| Cliente do Excel | Processo `xfreerdp` ainda ativo, CPU baixa | Não equivale a prova de conexão funcional nem de Excel respondendo. |
| Registro da ocorrência | Lançamento do Excel em RemoteApp, sem erro de saída posterior na coleta inicial | Não apareceu `ERR_CHILD_PROCESS_STDIO_MAXBUFFER`. |

O endpoint de estado RDP respondeu `false`. Não foi tratado como prova de desconexão: o código identifica a sessão buscando as palavras inglesas `active` e `rdp` na saída de `quser`, o que depende do idioma e do estado da sessão. A tela de console do Windows estava na tela de bloqueio, compatível com uso por RDP.

Configurações existentes verificadas: `scale=100`, `scaleDesktop=100`, `advancedFeatures=true`, `rdpMonitoringEnabled=false`, `multiMonitor=0`; argumento personalizado com substituição `+f → -f`. Foram consultados a configuração e o Compose existentes, mas eles não foram modificados nem anexados integralmente.

Também foram examinados processos e threads do cliente, conexão TCP local, registros do WinBoat/contêiner, saúde do agente Windows, memória, swap, pressão de recursos e registros de falhas do sistema. Não foi encontrada evidência de OOM, segfault ou erro de GPU que explicasse a ocorrência. Isso não elimina eventos que não ficaram registrados.

Na recorrência, `gdb` e `strace` estavam instalados, `eu-stack` não estava disponível e `ptrace_scope=1`. A tentativa não interativa de obter uma pilha com privilégios administrativos não conseguiu a autenticação necessária. Não foram instalados símbolos ou ferramentas nem alterada a proteção de depuração.

## 3. Pesquisa e interpretação das pistas

- [FreeRDP #12391](https://github.com/FreeRDP/FreeRDP/issues/12391): regressão histórica de RemoteApp na versão 3.23.0, corrigida em versões posteriores. Não corresponde à versão 3.32.1 instalada.
- [FreeRDP #12453](https://github.com/FreeRDP/FreeRDP/issues/12453): relato de congelamento de aplicativos RemoteApp com popups, inclusive em X11. É uma pista, não comprovação para este computador.
- [FreeRDP #11726](https://github.com/FreeRDP/FreeRDP/issues/11726): aplicativos Office congelam na conexão ao apresentar tooltips ou sobreposições, enquanto encerrar e reconectar recupera o acesso. A versão relatada difere da instalada; é uma correspondência de sintomas, não confirmação da causa.
- [WinBoat #767](https://github.com/winboat-org/winboat/issues/767): aplicativos isolados não abrem enquanto Windows Desktop e o serviço Windows funcionam. A causa proposta pelo autor não foi demonstrada para este caso.
- [WinBoat #829](https://github.com/winboat-org/winboat/issues/829): Excel e outras janelas desaparecem com erro de excesso de saída. Esse erro não foi encontrado no registro local; o sintoma também difere de uma janela congelada.
- [FreeRDP 3.32.1](https://github.com/FreeRDP/FreeRDP/releases/tag/3.32.1): inclui correção de copiar/colar dados grandes. Essa versão já está instalada; não há motivo demonstrado para downgrade.
- [Microsoft: Excel sem resposta](https://support.microsoft.com/en-us/excel/excel-not-responding-hangs-freezes-or-stops-working): modo seguro permite isolar complementos e configurações de inicialização se o aplicativo também falhar dentro do desktop completo.

Foram pesquisados todos os sites pedidos no projeto: Stack Overflow/Linux, Reddit r/linux4noobs e r/linux, Ask Ubuntu, Ubuntu Discourse e GitHub. A correspondência comunitária mais próxima foi [WinBoat Experience? em r/linux](https://www.reddit.com/r/linux/comments/1rp2edn/winboat_experience/), com relato de aplicativos individuais congelando enquanto Windows Desktop funciona. [Ubuntu Discourse](https://discourse.ubuntu.com/t/how-to-install-winboat-on-any-ubuntu-base-distro/70019) trata da instalação e dependências, sem diagnóstico equivalente. Não foi localizado caso específico equivalente nos demais sites; relatos apenas relacionados não foram usados para determinar a causa.

## 4. Outros testes, resultados e limites

1. No WinBoat, o usuário abriu `Apps → ⚙️ Windows Desktop` e confirmou que o Excel responde. O desktop completo é um contorno funcional confirmado.
2. O Windows foi encontrado desligado com saída 0 durante os testes. Com aprovação do usuário, foi iniciado novamente. A verificação posterior confirmou contêiner ativo e serviço Windows respondendo HTTP 200. O motivo específico do desligamento não foi determinado.
3. Teste concluído: após encerrar o cliente FreeRDP preso, o usuário abriu Microsoft Excel pela lista Apps e confirmou que responde. A [orientação no issue #316](https://github.com/winboat-org/winboat/issues/316) é fechar a conexão Desktop antes de abrir aplicativos individuais; isso não comprova que sobreposição causou a travada atual.
4. Houve recorrência ao abrir uma janela, menu ou aviso. A conexão temporária `/gfx:RFX` está em teste, sem mudança permanente. Caso ela também congele, priorizar a investigação do gerenciamento de janelas e a comparação com Windows Desktop. Um teste sem redirecionamento de clipboard fica em segundo plano enquanto não houver gatilho de copiar/colar. O WinBoat já tem um argumento personalizado; há [relato #873](https://github.com/winboat-org/winboat/issues/873) sobre múltiplos argumentos, portanto a alteração exige cuidado e um teste isolado.
5. Se uma futura ocorrência também afetar Excel no desktop completo, após preservar o trabalho, abrir `Run` e executar `excel /safe` para verificar complementos e inicialização, conforme a Microsoft. Esse teste não é prioritário no cenário atual, em que Excel responde no desktop completo.

Configurações verificadas no [código WinBoat 0.9.2](https://github.com/winboat-org/winboat/blob/v0.9.2/src/renderer/lib/winboat.ts): `RDP Monitoring` somente consulta o estado e mostra um banner; não evita sessões simultâneas. O argumento personalizado existente `+f → -f` não substitui a opção de tela cheia de Windows Desktop, porque `+f` é acrescentado depois da aplicação dessas substituições. Esse detalhe não explica a travada de Excel RemoteApp. Nenhuma dessas configurações foi alterada.

Uma tentativa de observar o console web do Windows mostrou a tela de bloqueio. Após a mudança das portas no reinício, novas tentativas no navegador foram limitadas pelo erro/política de navegação da ferramenta. Esse impedimento da ferramenta não foi considerado evidência de falha do Windows.

Os registros históricos continham códigos de conexão 131, 145 e 147 em outras datas. Não foi demonstrado que expliquem o congelamento atual. Mensagens sobre Kerberos sem realm, cursor sem janela e funcionalidades RAIL incompletas também não foram tomadas isoladamente como causas.

Não foram executados: `excel /safe`, teste sem clipboard, troca para sessão Xorg, downgrade/upgrade, alteração permanente de argumentos RDP, realocação do disco, mudança de RAM da VM, reinstalação ou alteração de `RDP Monitoring`. `/gfx:off` e `/app:disable-window-management` não são opções válidas verificadas para esse teste. Não foi localizada uma opção específica para desativar o gerenciamento de popups RAIL; [argumentos FreeRDP 3.32.1](https://github.com/FreeRDP/FreeRDP/blob/3.32.1/client/common/cmdline.h#L37).

## 5. Arquivos e artefatos da investigação

| Caminho local | Finalidade | Tratamento neste registro |
|---|---|---|
| `~/.winboat/winboat.config.json` | Configuração do WinBoat | Apenas valores relevantes resumidos. |
| `~/.winboat/docker-compose.yml` | Recursos, armazenamento e credenciais do Windows | Arquivo bruto excluído da publicação. |
| `~/.winboat/winboat.log` e `container.log` | Lançamentos e eventos | Apenas evidências sanitizadas resumidas. |
| `/tmp/winboat-rfx-test.py` | Helper criado para identificar/encerrar o cliente e abrir o teste RFX | Artefato temporário; depende do ambiente desta ocorrência. |
| `/tmp/winboat-rfx-42_2y4e5.log` | Saída do teste corrigido | Log bruto não publicado. |
| `/tmp/winboat-rfx-test-metadata.json` | PID do launcher, modo gráfico, resultado inicial e caminho do log | Launcher histórico PID 257092; ativo após 5 segundos, sem alteração permanente. |

Arquivos em `/tmp` podem desaparecer. O registro permanente é este documento. Não foram publicadas credenciais, tokens, configurações brutas ou comandos completos de lançamento. O FreeRDP mascara o argumento de senha na memória do processo; copiar o lançamento de `/proc` não recupera uma senha utilizável. A conexão corrigida leu as credenciais existentes diretamente da configuração do WinBoat em memória.

## 6. Roteiro para uma nova ocorrência

| Etapa | Ação | Objetivo |
|---|---|---|
| 1 | Anotar horário, nome do menu/aviso e sequência exata de cliques | Identificar o gatilho reproduzível. |
| 2 | Abrir `Apps → ⚙️ Windows Desktop` e observar o mesmo Excel | Separar problema do aplicativo de problema da janela integrada. |
| 3 | Salvar o trabalho pelo Desktop, quando ele responder | Preservar alterações antes de encerrar qualquer cliente. |
| 4 | Coletar estado e registros sanitizados antes de encerrar | Preservar evidências do congelamento. |
| 5 | Identificar o cliente atual do Excel e encerrar somente ele, se necessário | Recuperar acesso sem desligar o Windows ou afetar outras conexões. |
| 6 | Registrar o resultado e repetir o mesmo gatilho no teste escolhido | Distinguir recuperação temporária de mudança eficaz. |

Se o teste RFX também congelar, repetir exatamente o mesmo menu/aviso no Windows Desktop é o próximo controle recomendado. O Desktop foi um contorno funcional na primeira ocorrência. Se ele continuar funcionando, pode ser usado para trabalhar enquanto se investiga RemoteApp. Capturar as pilhas de todas as threads do cliente ainda congelado é o próximo passo para tentar identificar a função bloqueada; depende de acesso administrativo local.

Uma comparação em Xorg fica para depois: exige sair da sessão Linux e ajuda a separar a participação de XWayland, sem garantir solução. Se Excel também congelar dentro do Desktop, a investigação passa a incluir o aplicativo, complementos e `excel /safe`.

### Coleta inicial, sem encerrar processos

Este bloco é uma orientação para a próxima ocorrência; não foi executado novamente para publicar o documento. Executar no terminal do computador afetado, fora do sandbox do agente:

```bash
hostnamectl
printenv XDG_SESSION_TYPE
flatpak run --command=xfreerdp com.freerdp.FreeRDP /version
ps -C xfreerdp -o pid,comm,stat,pcpu,rss
free -h
vmstat 1 5
docker inspect --format 'running={{.State.Running}} status={{.State.Status}} exit={{.State.ExitCode}} oom={{.State.OOMKilled}}' WinBoat
docker port WinBoat
```

O bloco não exibe os argumentos dos processos nem o Compose inteiro, que podem conter credenciais. As portas devem ser consultadas novamente: podem mudar depois de iniciar o contêiner. Para identificar com segurança qual PID corresponde ao Excel, o agente pode examinar os argumentos localmente e apresentar apenas a identificação sanitizada.

Para retomar com Codex, fornecer o link deste documento, a máquina utilizada, o horário da falha, o menu/aviso que abriu, se há trabalho não salvo e se o mesmo Excel responde em Windows Desktop. Solicitar consulta aos registros locais antes de encerrar o cliente. Não reaplicar o helper temporário ou PIDs antigos automaticamente.

## 7. Pendências

- Confirmar o nome e os cliques exatos do menu/aviso que causou a recorrência.
- Confirmar se a janela do teste RFX aparece, responde e permanece utilizável após repetir esse gatilho.
- Se necessário, repetir o gatilho no Desktop e obter pilhas do cliente congelado.
- Identificar o mecanismo interno e validar uma correção permanente antes de alterar a configuração habitual.
- Recuperar as conversas antigas sobre WinBoat quando o usuário quiser retomar essa parte.

Histórico de conversas: adiado por escolha do usuário. A listagem disponível mostrava apenas as 50 conversas mais recentes, sem conversas antigas do WinBoat. O navegador ChatGPT estava sem login. A pasta `sources/` desta conversa estava vazia, sem inventário ou arquivos `Prompt`. Este relatório não afirma ter identificado todas as conversas nem confirmado uma correção permanente. A recuperação inicial foi confirmada pelo usuário, mas houve recorrência e o teste gráfico está pendente.

## 8. Glossário

- `RemoteApp`: exibe somente a janela do aplicativo Windows no Linux; `Windows Desktop`: exibe a área de trabalho Windows inteira; `Apps`: lista de aplicativos do WinBoat.
- `FreeRDP`/`xfreerdp`: cliente da conexão com Windows; `XWayland`: permite aplicativos X11 em uma sessão Wayland; `Xorg`: servidor gráfico X11.
- `/gfx:RFX`: seleciona o codec gráfico RemoteFX para a conexão; `RAIL`: protocolo de integração das janelas remotas; `clipboard`: área de transferência.
- `PID`: identificador de processo; `SIGTERM`: solicita encerramento normal; `SIGKILL`: força o encerramento do processo escolhido; `futex`: mecanismo de espera e sincronização entre threads.
- `swap`: disco usado para páginas de memória; `OOM`: falta de memória; `Recv-Q`: dados recebidos pelo socket que ainda aguardam leitura; `deadlock`: bloqueio causado por esperas entre operações.
- `hostnamectl`: mostra identificação do computador e sistema; `printenv XDG_SESSION_TYPE`: mostra o tipo da sessão gráfica.
- `flatpak run`: executa um aplicativo Flatpak; `--command=xfreerdp`: escolhe seu cliente RDP; `com.freerdp.FreeRDP`: identificador do aplicativo; `/version`: mostra a versão.
- `ps`: lista processos; `-C xfreerdp`: filtra pelo nome; `-o`: escolhe colunas; `pid`, `comm`, `stat`, `pcpu`, `rss`: identificador, nome, estado, CPU percentual e memória residente em KiB.
- `free -h`: mostra memória e swap em unidades legíveis; `vmstat 1 5`: mostra cinco amostras com intervalo de um segundo; a primeira representa médias desde o boot.
- `docker inspect --format`: consulta dados selecionados do contêiner; `Running`, `Status`, `ExitCode`, `OOMKilled`: execução, estado, código de saída e encerramento por falta de memória; `docker port WinBoat`: mostra as portas mapeadas; `docker start WinBoat`: inicia o contêiner existente.
- `gdb`/`strace`: ferramentas de depuração de pilhas e chamadas de sistema; `eu-stack`: ferramenta de pilhas; `ptrace_scope`: controle de permissão para observar processos.
- `RDP Monitoring`: consulta e mostra o estado da sessão; `scale`/`scaleDesktop`: escala do aplicativo/desktop; `advancedFeatures`: recursos avançados; `multiMonitor`: configuração de monitores; `rdpArgs`: argumentos personalizados.
- `+f`/`-f`: ativam/desativam tela cheia; `/p:`: argumento de senha; `Run`: executa comandos no Windows; `excel /safe`: abre Excel em modo seguro; `Sign out`: encerra a sessão Windows; `Shut down`: desliga o Windows.
- `~`: diretório pessoal do usuário; `PATH`: diretórios pesquisados para localizar executáveis; `/tmp`: armazenamento temporário; `/proc`: informações de processos mantidas pelo kernel.
