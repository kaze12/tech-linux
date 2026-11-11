# Aplicativos APT em partição NTFS via imagem ext4 (taiwan_com)

**Máquina de referência:** `asus-taiwan`, Ubuntu 24.04. **Procedimento testado:** Microsoft Edge 157.0.4341.46, em 10–11/11/2026. **Objetivo:** minimizar o uso da partição Linux (`taiwan_lin`) mantendo o gerenciamento do pacote pelo APT.

## Resultado comprovado

| Componente | Local / resultado |
|---|---|
| Partição Linux | `/` em ext4 (`/dev/nvme0n1p5`) |
| Partição compartilhada | `/mnt/taiwan_com` em NTFS3 (`/dev/nvme0n1p6`) |
| Imagem do aplicativo | `/mnt/taiwan_com/t_apps/microsoft-edge.ext4`, 2 GiB |
| Sistema de arquivos dentro da imagem | ext4 |
| Montagem do pacote | `/opt/microsoft/msedge` |
| Fonte de atualização | `https://packages.microsoft.com/repos/edge-stable` |
| Verificação após reinicialização | unidade `opt-microsoft-msedge.mount` ativa; Edge 157.0.4341.46 executável |
| Espaço livre em `/` | aumentou de 4,9 GB para 5,5 GB após remover o backup (valores arredondados) |

**O que ainda NÃO foi demonstrado:** uma atualização real do Edge para uma versão posterior com a imagem montada. O APT reconhece o pacote e seu repositório, mas isso não prova que um futuro upgrade seja bem-sucedido. Tampouco foi comprovada a instalação *sem intervenção* pelo `unattended-upgrades`.

## Princípio do método

A instalação convencional pelo APT registra os arquivos e metadados do pacote no Linux. Para um aplicativo cujos arquivos principais ficam concentrados em uma pasta (por exemplo `/opt/microsoft/msedge`), essa pasta pode ser substituída por um ponto de montagem ext4. O sistema ext4 fica em um **arquivo dentro da partição NTFS**, montado por um dispositivo de loop.

O APT continua usando o caminho original. Não se trata de modificar o prefixo de instalação de um pacote `.deb` nem de criar um link simbólico diretamente para NTFS.

**Limites:**
- Apenas o conteúdo do diretório montado migra. Metadados do APT, outras bibliotecas, atalhos, configurações e perfil podem continuar no Linux.
- Nem todo aplicativo APT possui arquivos agrupados em um único diretório. Não aplicar o procedimento cegamente a pacotes de sistema, bibliotecas compartilhadas ou serviços críticos.
- A imagem tem tamanho fixo (2 GiB neste exemplo). Conferir o espaço livre *dentro* dela antes de cada atualização e ampliá-la quando necessário.
- É indispensável manter a montagem ativa antes de executar atualizações do pacote. Se a imagem não estiver montada, o APT pode tentar gravar no diretório subjacente em `taiwan_lin`, criando uma instalação inconsistente.
- O parâmetro `nofail` evita bloquear o boot caso `taiwan_com` esteja ausente, mas **não protege sozinho** o APT de uma atualização quando a montagem estiver indisponível.
- Montagens de imagem dependem da saúde e disponibilidade da partição NTFS. Sempre manter backup dos dados importantes.

## Quando considerar este método

1. O instalador oficial é `.deb` e o aplicativo recebe atualizações de um repositório APT.
2. A maioria dos executáveis e recursos está em uma pasta própria, como `/opt/fabricante/aplicativo`.
3. A pasta não é exigida antes de `/mnt/taiwan_com` estar montada.
4. Há espaço suficiente na partição compartilhada e na imagem ext4 para futuras atualizações.
5. O procedimento pode ser validado e revertido antes de eliminar o backup original.

Para aplicativos distribuídos como Flatpak, Snap, AppImage ou programas portáteis, avaliar as formas de instalação e atualização próprias; este guia não transfere automaticamente tais formatos.

## Procedimento para um novo aplicativo APT

**Exemplo concreto:** Microsoft Edge. **Não repita os comandos de criação ou formatação no Edge já migrado**; eles são para uma implantação nova. Ajuste pacote, diretório, imagem e tamanho caso use outro aplicativo. Execute uma etapa por vez, verificando cada saída antes de avançar.

| Etapa | Objetivo |
|---|---|
| 1 | Instalar pelo APT e confirmar onde ficam os arquivos |
| 2 | Criar uma imagem ext4 dentro da partição NTFS |
| 3 | Copiar arquivos e comparar conteúdo |
| 4 | Montar a imagem no caminho original |
| 5 | Registrar no `/etc/fstab` com dependência da partição NTFS |
| 6 | Reiniciar, verificar e somente então remover o backup |

### 1. Verificar o pacote e a pasta

~~~bash
dpkg-query -L microsoft-edge-stable | head -n 35
sudo du -sh /opt/microsoft/msedge
findmnt /mnt/taiwan_com
df -hT / /mnt/taiwan_com
pgrep -af 'msedge|microsoft-edge' || true
~~~

Fechar completamente o aplicativo antes de copiar. Não prosseguir se o caminho de destino já estiver montado ou se a imagem proposta já existir.

### 2. Criar a imagem ext4

~~~bash
test ! -e /mnt/taiwan_com/t_apps/microsoft-edge.ext4
sudo fallocate -l 2G /mnt/taiwan_com/t_apps/microsoft-edge.ext4
sudo mkfs.ext4 -F /mnt/taiwan_com/t_apps/microsoft-edge.ext4
sudo mkdir -p /mnt/edge-staging
sudo mount -o loop /mnt/taiwan_com/t_apps/microsoft-edge.ext4 /mnt/edge-staging
df -hT /mnt/edge-staging
~~~

**Atenção:** `mkfs.ext4 -F` apaga o conteúdo da imagem indicada; não executar sobre uma imagem que contenha dados. O `test ! -e` é uma verificação inicial: se falhar, **pare**. A execução em linhas separadas não impede o Bash de continuar após um erro; não cole o bloco inteiro sem acompanhar os resultados.

### 3. Copiar e validar

~~~bash
sudo cp -a /opt/microsoft/msedge/. /mnt/edge-staging/
sudo diff -qr -x lost+found /opt/microsoft/msedge /mnt/edge-staging
echo "Diff exit code: $?"
sudo du -sh /opt/microsoft/msedge /mnt/edge-staging
sync
~~~

Exigir `Diff exit code: 0` antes de prosseguir. O diretório `lost+found` é criado pelo ext4 e não faz parte do pacote; por isso é excluído da comparação. `diff` compara conteúdo, mas não certifica todos os metadados; `cp -a` é usado para preservar atributos.

### 4. Substituir a pasta por um ponto de montagem

~~~bash
sudo umount /mnt/edge-staging
sudo mv /opt/microsoft/msedge /opt/microsoft/msedge.backup
sudo mkdir /opt/microsoft/msedge
sudo mount -o loop /mnt/taiwan_com/t_apps/microsoft-edge.ext4 /opt/microsoft/msedge
findmnt /opt/microsoft/msedge
microsoft-edge-stable --version
~~~

**Não executar esta etapa duas vezes.** Se `mv` ou `mount` falhar, interromper o procedimento e diagnosticar; não tentar repetir as operações às cegas. Manter o `.backup` até os testes pós-reinicialização.

### 5. Montagem persistente

Salvar uma cópia do `fstab`:

~~~bash
sudo cp -a /etc/fstab /etc/fstab.bak-edge
~~~

Acrescentar **uma única vez** a linha abaixo a `/etc/fstab` (para edição gráfica, usar `gedit` com acesso administrativo):

~~~fstab
/mnt/taiwan_com/t_apps/microsoft-edge.ext4 /opt/microsoft/msedge ext4 loop,nofail,x-systemd.requires-mounts-for=/mnt/taiwan_com 0 0
~~~

Recarregar e verificar:

~~~bash
sudo systemctl daemon-reload
sudo findmnt --verify --verbose
systemctl show opt-microsoft-msedge.mount -p LoadState -p ActiveState -p RequiresMountsFor --no-pager
findmnt /opt/microsoft/msedge
~~~

A opção `x-systemd.requires-mounts-for=/mnt/taiwan_com` declara a dependência da montagem NTFS. A unidade `opt-microsoft-msedge.mount` é **gerada** a partir do `fstab`: o resultado `generated` de `systemctl is-enabled` é normal. **Não usar** `systemctl enable` nessa unidade.

### 6. Reiniciar, validar e liberar espaço

Após reiniciar o Ubuntu, executar:

~~~bash
findmnt /opt/microsoft/msedge
systemctl is-active opt-microsoft-msedge.mount
microsoft-edge-stable --version
sudo du -sh /opt/microsoft/msedge.backup
df -h /
~~~

Somente se a montagem estiver ativa, o programa funcionar e o backup original tiver sido conferido, remover a cópia do Linux:

~~~bash
sudo rm -rf -- /opt/microsoft/msedge.backup
df -h /
~~~

No caso real, o backup tinha aproximadamente 680 MB. O espaço disponível em `/` passou de 4,9 GB para 5,5 GB (arredondado).

## Atualizações: APT versus atualização sem intervenção

O pacote **permanece registrado no APT** e seu repositório pode oferecer novas versões:

~~~bash
sudo apt update
apt-cache policy microsoft-edge-stable
findmnt /opt/microsoft/msedge
~~~

Antes de atualizar, confirmar que a montagem está ativa e que a imagem tem espaço. Em execução manual, pode-se condicionar a operação:

~~~bash
mountpoint -q /opt/microsoft/msedge && sudo apt upgrade
~~~

Isto é uma **checagem operacional**, não um bloqueio global confiável para outras rotinas automáticas. O comando pessoal `kz update` também precisa respeitar essa condição antes de atualizar o Edge.

**Importante:** um repositório de terceiros configurado no APT não entra automaticamente nas origens permitidas pelo `unattended-upgrades` do Ubuntu. Para atualizações realmente autônomas é necessário, em trabalho separado, configurar a origem Microsoft na política do `unattended-upgrades`, verificar a montagem antes de qualquer upgrade automático e testar uma atualização real. **Essa automação ainda não foi implantada nem validada neste registro.**

Documentação oficial: [Ubuntu — Automatic updates](https://ubuntu.com/server/docs/how-to/software/automatic-updates/) e [Ubuntu — Security updates](https://documentation.ubuntu.com/security/security-updates/).

## Diagnóstico e reversão

Verificar montagem e dependências:

~~~bash
findmnt /opt/microsoft/msedge
systemctl show opt-microsoft-msedge.mount -p ActiveState -p RequiresMountsFor --no-pager
sudo losetup -j /mnt/taiwan_com/t_apps/microsoft-edge.ext4
df -h / /opt/microsoft/msedge /mnt/taiwan_com
~~~

**Não atualizar pacotes se a imagem não estiver montada.** Em caso de falha de montagem, verificar primeiramente `/mnt/taiwan_com`, o arquivo `.ext4` e as mensagens do systemd.

Para retornar à instalação convencional em `taiwan_lin`, fechar o Edge, retirar sua linha de `/etc/fstab`, recarregar o systemd, desmontar a imagem e reinstalar o pacote pelo APT. A reversão volta a exigir aproximadamente 700 MB na partição Linux; se não houver espaço, liberar espaço antes. Não apagar a imagem até confirmar que o pacote reinstalado funciona.

## Referências

- [systemd.mount — opções de montagem e dependências](https://www.freedesktop.org/software/systemd/man/latest/systemd.mount.html)
- [Ubuntu — Automatic updates](https://ubuntu.com/server/docs/how-to/software/automatic-updates/)
- [Ubuntu — Security updates](https://documentation.ubuntu.com/security/security-updates/)

# Glossário

### Inspeção
- `dpkg-query -L`: lista os caminhos registrados para um pacote Debian.
- `du -sh`: mede o espaço ocupado por um diretório.
- `findmnt`: identifica sistemas de arquivos montados.
- `df -hT`: informa espaço disponível e tipo do sistema de arquivos.
- `pgrep -af`: procura processos mostrando sua linha de comando.
- `head -n`: limita o número de linhas exibidas.

### Criação da imagem
- `test ! -e`: verifica que o caminho ainda não existe.
- `fallocate -l 2G`: aloca um arquivo de 2 GiB.
- `mkfs.ext4 -F`: cria um sistema ext4 em arquivo comum; destrói conteúdo prévio.
- `mkdir -p`: cria diretórios intermediários.
- `mount -o loop`: monta um arquivo de imagem como dispositivo de loop.
- `umount`: desmonta um sistema de arquivos.

### Cópia e migração
- `cp -a`: copia preservando atributos e links.
- `diff -qr -x lost+found`: compara conteúdos recursivamente, ignorando a pasta padrão do ext4.
- `$?`: código de retorno do comando anterior; `0` indica sucesso.
- `sync`: solicita gravação dos dados pendentes.
- `mv`: move ou renomeia um diretório.
- `rm -rf --`: remove recursivamente sem confirmação; operação destrutiva restrita ao caminho informado.

### Montagem persistente
- `/etc/fstab`: configuração de montagens persistentes.
- `loop`: utiliza um arquivo como dispositivo de bloco.
- `nofail`: não torna a falha de montagem impeditiva para o boot.
- `x-systemd.requires-mounts-for`: declara dependência de montagem de caminho.
- `systemctl daemon-reload`: recarrega unidades do systemd.
- `findmnt --verify --verbose`: verifica e detalha configurações de montagem.
- `systemctl show -p`: mostra propriedades selecionadas da unidade.
- `systemctl is-active`: consulta o estado atual da unidade.

### Atualizações
- `apt update`: atualiza o índice de pacotes dos repositórios.
- `apt-cache policy`: informa versões candidatas e repositórios.
- `apt upgrade`: instala atualizações disponíveis para os pacotes gerenciados pelo APT.
- `mountpoint -q`: testa silenciosamente se o caminho é ponto de montagem.
- `unattended-upgrades`: mecanismo de aplicação autônoma de atualizações quando suas origens estão autorizadas.
