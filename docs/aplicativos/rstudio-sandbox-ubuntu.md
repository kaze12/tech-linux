# RStudio no Ubuntu 24.04: erro de sandbox na particao NTFS

- **Data do diagnostico:** 2026-10-22
- **Computador:** taiwan (asus-taiwan)
- **Sistema:** Ubuntu 24.04 LTS
- **R:** 4.6.1, em `/mnt/taiwan_com/t_apps/R/4.6.1/bin/R`
- **RStudio:** 2026.09.0+174, em `/mnt/taiwan_com/t_apps/rstudio/2026.09.0-174/`
- **Launcher:** `~/.local/share/applications/rstudio-taiwan.desktop`
- **Script:** `~/.local/bin/rstudio-taiwan`

## Sintoma

Clicar em **RStudio (taiwan_com)** no menu *Programming* nao abre nenhuma janela.

O launcher executava:

```ini
Name=RStudio (taiwan_com)
Exec=/home/icm/.local/bin/rstudio-taiwan %F
Terminal=false
```

O script configurava as variaveis `RSTUDIO_WHICH_R`, `PATH`, `XDG_CONFIG_HOME`, `XDG_DATA_HOME`, `XDG_CACHE_HOME`, `RSTUDIO_CONFIG_HOME`, `RSTUDIO_DATA_HOME`, `RS_LOG_DIR`, `RS_INITIAL_WD`, `TMPDIR` e `R_LIBS_USER`, e executava o binario do RStudio na particao compartilhada.

## Diagnostico

A montagem foi confirmada com `findmnt /mnt/taiwan_com`: o volume era `ntfs3`, montado em modo leitura/escrita.

A execucao `bash -x ~/.local/bin/rstudio-taiwan 2>&1` mostrou que o R 4.6.1 funcionava e que o script chegava a executar o RStudio. A falha ocorria no sandbox Chromium/Electron:

```text
FATAL:sandbox/linux/suid/client/setuid_sandbox_host.cc:166
The SUID sandbox helper binary was found, but is not configured correctly.
You need to make sure that .../chrome-sandbox is owned by root and has mode 4755.
```

O `chrome-sandbox` depende de permissoes Unix de proprietario e setuid, inadequadas para esta instalacao em NTFS.

Tentativa seguinte:

```bash
~/.local/bin/rstudio-taiwan --disable-setuid-sandbox
```

Resultado:

```text
FATAL:content/browser/zygote_host/zygote_host_impl_linux.cc:128
No usable sandbox!
```

Assim, o fallback para user namespaces tambem estava indisponivel, compativel com as restricoes de AppArmor aplicadas pelo Ubuntu.

## Correcao aplicada e testada

Criar um perfil AppArmor especifico para o executavel instalado:

```bash
sudo tee /etc/apparmor.d/rstudio-taiwan >/dev/null <<'EOF'
abi <abi/4.0>,
include <tunables/global>

/mnt/taiwan_com/t_apps/rstudio/2026.09.0-174/rstudio flags=(unconfined) {
    userns,
}
EOF

sudo apparmor_parser -r /etc/apparmor.d/rstudio-taiwan
```

Testar a inicializacao com:

```bash
~/.local/bin/rstudio-taiwan --disable-setuid-sandbox
```

**Resultado observado:** o RStudio abriu e exibiu o console R 4.6.1, com diretorio inicial `/mnt/taiwan_com/t_docs/RStudio`. A imagem enviada pelo usuario confirma a inicializacao. O AppArmor foi ajustado apenas para este executavel; `flags=(unconfined)` nao implica que o RStudio inteiro esteja confinado por regras restritivas adicionais.

## Pendencia: abertura pelo menu

O script original ainda usa:

```sh
exec "/mnt/taiwan_com/t_apps/rstudio/2026.09.0-174/rstudio" "$@"
```

Para que a opcao testada seja usada pelo atalho, alterar essa linha para:

```sh
exec "/mnt/taiwan_com/t_apps/rstudio/2026.09.0-174/rstudio" --disable-setuid-sandbox "$@"
```

**Status:** essa alteracao no script local ainda nao foi confirmada. Depois de aplica-la, testar o icone no GNOME.

## Manutencao e limites

- Se a versao ou o caminho do executavel do RStudio mudar, revisar **o script e o perfil AppArmor**; o perfil desta ocorrencia referencia um caminho absoluto.
- Nao substituir por `--no-sandbox`: essa opcao desativa os mecanismos de isolamento do Chromium.
- O argumento `--disable-setuid-sandbox` apenas desativa o mecanismo legado SUID; o sucesso depende de user namespaces estarem disponiveis.
- O perfil com `flags=(unconfined)` permite `userns`, mas nao e equivalente a um perfil de confinamento restritivo completo.
- A janela exibiu um aviso de disponibilidade da versao 2026.10.0+233; a atualizacao nao faz parte desta correcao.

## Referencia tecnica

- [Chromium: AppArmor user namespace restrictions](https://chromium.googlesource.com/chromium/src/+/main/docs/security/apparmor-userns-restrictions.md)
