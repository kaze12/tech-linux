# Ícone genérico de aplicativo no Dash do GNOME

- **Status:** resolvido no exemplo; procedimento geral de diagnóstico
- **Ambiente do exemplo:** Ubuntu 24.04, GNOME/Wayland, instalação personalizada em partição compartilhada
- **Data:** 2026-10-27

## Problema

Um aplicativo funciona, mas sua janela aparece no Dash com uma engrenagem ou outro ícone genérico. Isso pode ocorrer mesmo quando o menu contém um lançador `.desktop` com `Icon=` definido. Instalar o programa fora da partição do Linux **não é, por si só, causa comprovada**.

Há pelo menos dois problemas distintos:

1. **Ícone do lançador:** caminho inválido, imagem inválida, permissões ou integração do tema de ícones.
2. **Associação janela–lançador:** o GNOME não relaciona a janela aberta ao arquivo `.desktop` correto; isso pode produzir ícone genérico ou entradas separadas no Dash.

## Diagnóstico (sem alterar arquivos)

Substituir `NOME` e os caminhos pelos do aplicativo. Primeiro identificar os lançadores existentes e os valores relevantes:

```bash
grep -Ril 'NOME' ~/.local/share/applications /usr/share/applications 2>/dev/null
grep -E '^(Name|Exec|Icon|StartupWMClass|StartupNotify)=' ~/.local/share/applications/ARQUIVO.desktop
desktop-file-validate ~/.local/share/applications/ARQUIVO.desktop
```

Se `Icon=` contiver um caminho absoluto, conferir arquivo e formato (e não somente a existência):

```bash
file /caminho/do/icone.png
test -f /caminho/do/icone.png && echo 'Arquivo encontrado'
```

Com o aplicativo aberto, conferir se foi iniciado pelo lançador esperado (a variável pode aparecer apenas em alguns dos processos):

```bash
for pid in $(pgrep -f '/caminho/do/aplicativo'); do
  tr '\0' '\n' < "/proc/$pid/environ" 2>/dev/null |
    grep -E '^(GIO_LAUNCHED_DESKTOP_FILE|BAMF_DESKTOP_FILE_HINT|DESKTOP_STARTUP_ID)='
done
```

A presença de `GIO_LAUNCHED_DESKTOP_FILE` confirma o arquivo usado para iniciar aquele processo; **não prova** que a janela foi associada ao lançador no Dash. Não inferir o identificador da janela apenas de `productName` ou do nome do executável.

## Correção geral

1. Corrigir `Icon=` se o caminho ou a imagem estiverem incorretos. Preferir uma imagem válida acessível na sessão do usuário.
2. Se o ícone estiver válido, verificar a associação janela–lançador. Investigar o identificador real da janela (classe X11 ou app ID Wayland); nem toda ferramenta X11 funciona em Wayland.
3. Fazer backup do lançador e **testar** um nome de arquivo `.desktop` compatível com o identificador do aplicativo e, quando aplicável, `StartupWMClass` compatível. A correspondência exata depende do toolkit e do protocolo; não há um valor universal.
4. Fechar inteiramente o aplicativo, abri-lo **pelo lançador testado** e verificar o ícone **da janela aberta** no Dash.
5. Depois de confirmado o funcionamento, eliminar entradas duplicadas com cuidado e conferir eventuais favoritos fixados no Dash. Preservar backup para reversão.

`update-desktop-database ~/.local/share/applications` pode ser executado após editar lançadores, mas atualiza principalmente associações MIME: **não força o GNOME a corrigir o ícone ou reconstruir seu cache**. Não é necessário reinstalar o aplicativo só por esse sintoma.

## Exemplo confirmado: RStudio em `taiwan_com`

Na máquina `asus-taiwan`, o RStudio `2026.10.0+233` estava em `/mnt/taiwan_com/t_apps/rstudio/rstudio-2026.10.0+233/`, iniciado por `~/.local/bin/rstudio-taiwan`. O lançador original `~/.local/share/applications/rstudio-taiwan.desktop` tinha:

```ini
Name=RStudio (taiwan_com)
Exec=/home/icm/.local/bin/rstudio-taiwan %F
Icon=/mnt/taiwan_com/t_apps/rstudio/rstudio-2026.10.0+233/resources/app/rstudio.png
StartupWMClass=RStudio
```

O PNG foi confirmado como válido (48 × 48 RGBA). `desktop-file-validate` emitiu apenas um **hint** sobre múltiplas categorias principais (`Development;Science;Math;`), não um erro relacionado ao ícone. A variável `GIO_LAUNCHED_DESKTOP_FILE` apontava para o lançador original. Ainda assim, a janela exibiu uma engrenagem no Dash.

Teste realizado (com backup do arquivo original):

```bash
desktop="$HOME/.local/share/applications/rstudio-taiwan.desktop"
target="$HOME/.local/share/applications/rstudio.desktop"

cp "$desktop" "${desktop}.bak"
sed -e 's/^StartupWMClass=.*/StartupWMClass=rstudio/' "$desktop" > "$target"
desktop-file-validate "$target"
update-desktop-database "$HOME/.local/share/applications"
```

Após fechar e reabrir o RStudio pelo menu de aplicativos, **o usuário confirmou que o ícone correto apareceu no Dash**.

**Limite da evidência:** foram alterados simultaneamente o nome do arquivo (`rstudio-taiwan.desktop` → `rstudio.desktop`) e o `StartupWMClass` (`RStudio` → `rstudio`). O teste confirma o resultado conjunto, mas não estabelece qual das duas alterações era necessária. O nome `productName: "rstudio"` no `package.json` foi identificado; não houve leitura conclusiva do app ID Wayland da janela (a tentativa com `org.gnome.Shell.Eval` retornou `(false, '')`).

**Pendência de manutenção:** o lançador original e o novo podem coexistir; antes de remover qualquer um, verificar qual entrada está sendo utilizada e/ou fixada no Dash. O backup `rstudio-taiwan.desktop.bak` também foi preservado. **Não há confirmação de limpeza das duplicatas.**

Esta correção é independente do problema anterior de sandbox/AppArmor do aplicativo, documentado em [RStudio no Ubuntu 24.04: erro de sandbox](rstudio-sandbox-ubuntu.md).

# Glossário

### Diagnóstico

- `grep -Ril`: busca recursivamente e mostra arquivos com correspondência, ignorando maiúsculas/minúsculas.
- `grep -E`: procura padrões com expressões regulares estendidas.
- `desktop-file-validate`: verifica o formato e as chaves do arquivo `.desktop`.
- `file`: identifica o formato real de um arquivo.
- `test -f`: verifica se existe um arquivo regular.
- `pgrep -f`: procura processos pela linha de comando completa.
- `tr '\0' '\n'`: converte separadores nulos em quebras de linha.
- `2>/dev/null`: oculta mensagens de erro.
- `|`: envia a saída de um comando para outro.

### Correção

- `Icon`: referência ao ícone definido no lançador.
- `Exec`: comando que o lançador executa.
- `StartupWMClass`: dica de associação de janela, especialmente em X11; pode não resolver identificação Wayland.
- `cp`: copia arquivo, usado aqui para backup.
- `sed -e`: aplica uma substituição de texto.
- `update-desktop-database`: atualiza associações MIME nos arquivos de aplicações.
