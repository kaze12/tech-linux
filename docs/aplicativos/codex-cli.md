# Codex CLI no Linux — instalação, configuração, plugins e uso

Máquina: `asus-taiwan`  
Sistema: Ubuntu Linux  
Arquitetura: `x86_64`

## 1. Estado inicial

Foi verificado se o Codex e seu sandbox já estavam instalados:

```bash
command -v codex || echo "Codex not installed"
command -v bwrap || echo "Bubblewrap not installed"
uname -m
```

Resultado:

```text
/home/icm/.local/bin/codex
/usr/bin/bwrap
x86_64
```

Conclusão:

- Codex já estava instalado.
- Bubblewrap já estava instalado.
- A máquina usa arquitetura `x86_64`.
- Não era necessário reinstalar o Codex.

O executável utilizado pelo terminal era:

```text
/home/icm/.local/bin/codex
```

Esse arquivo é um link simbólico para a instalação standalone:

```text
/home/icm/.local/bin/codex
→ /home/icm/.codex/packages/standalone/current/bin/codex
```

## 2. Validação da instalação existente

Foi executado:

```bash
codex --version
ls -lh "$(command -v codex)"
codex login status
```

Resultado inicial:

```text
codex-cli 0.142.2
```

Autenticação:

```text
Logged in using ChatGPT
```

O Codex já estava autenticado com a conta ChatGPT.

## 3. Problema com o primeiro bloco de diagnóstico

Inicialmente foi utilizado um bloco com:

```bash
set -e
```

dentro de:

```bash
{
    ...
}
```

Isso tinha um efeito indesejado.

`{ ... }` executa os comandos no shell atual. Com `set -e`, caso um comando retornasse erro, o Bash poderia encerrar a sessão inteira.

Como o Codex apresentou um erro durante o teste, a janela do terminal chegou a fechar.

### Solução adotada

Os testes seguintes passaram a usar:

```bash
(
    ...
)
```

Esse formato executa os comandos em um subshell.

Também foi utilizado:

```bash
set +e
```

para impedir que um erro intermediário encerrasse a execução.

Assim, mesmo quando o Codex retorna erro, o terminal principal permanece aberto.

## 4. Diagnóstico da primeira falha do Codex

O teste utilizado foi:

```bash
codex exec \
    --skip-git-repo-check \
    --ephemeral \
    --sandbox read-only \
    "Reply with exactly: CODEX_OK"
```

O Codex iniciou com:

```text
model: gpt-6-astra
sandbox: read-only
```

Mas retornou:

```text
The 'gpt-6-astra' model requires a newer version of Codex.
Please upgrade to the latest app or CLI and try again.
```

Também apareceu:

```text
failed to load models cache: missing field `base_instructions`
```

A versão `0.142.2` estava antiga demais para utilizar o modelo `gpt-6-astra`.

## 5. Atualização do Codex

Foi utilizado o instalador standalone oficial:

```bash
curl -fsSL https://chatgpt.com/codex/install.sh | CODEX_NON_INTERACTIVE=true sh
```

Resultado:

```text
Updating Codex CLI from 0.142.2 to 0.160.1
```

A instalação passou a utilizar:

```text
/home/icm/.codex/packages/standalone/releases/0.160.1-x86_64-unknown-linux-musl
```

e manteve:

```text
/home/icm/.local/bin/codex
```

como comando principal.

Foi executado:

```bash
hash -r
```

para fazer o Bash atualizar o caminho armazenado para o executável.

### Atualizações futuras

Não é necessário utilizar `CODEX_NON_INTERACTIVE=true`.

Pode ser usado simplesmente:

```bash
curl -fsSL https://chatgpt.com/codex/install.sh | sh
```

`CODEX_NON_INTERACTIVE=true` foi incluído apenas para evitar perguntas durante aquele bloco automatizado e não é necessário para o uso normal.

## 6. Estado após a atualização

A versão instalada passou a ser:

```text
codex-cli 0.160.1
```

A autenticação continuou:

```text
Logged in using ChatGPT
```

O teste foi repetido:

```bash
codex exec \
    --skip-git-repo-check \
    --ephemeral \
    --sandbox read-only \
    "Reply with exactly: CODEX_OK"
```

Resultado:

```text
CODEX_OK
```

e:

```text
Codex exit code: 0
```

Portanto, ficaram validados:

- Codex CLI funcionando;
- login com ChatGPT funcionando;
- `gpt-6-astra` funcionando;
- execução pelo terminal funcionando;
- sandbox `read-only` funcionando;
- Bubblewrap disponível.

## 7. Como usar o Codex

O Codex não fica permanentemente executando em segundo plano.

Ele é iniciado quando necessário.

### Uso interativo

Entrar primeiro no projeto:

```bash
cd /mnt/taiwan_com/t_rpg/t_zhang_qian
codex
```

O Codex passa a trabalhar tomando esse diretório como contexto.

Exemplos de comandos dentro da interface:

```text
Analise a estrutura deste projeto.

Procure erros no código.

Explique como este projeto funciona.

Corrija o problema da página de criaturas.

Crie um novo arquivo seguindo o padrão dos arquivos existentes.
```

Também é possível iniciar já com uma solicitação:

```bash
cd /mnt/taiwan_com/t_rpg/t_zhang_qian
codex "Analise a estrutura deste projeto."
```

## 8. Uso não interativo

Para executar uma tarefa e encerrar:

```bash
cd /mnt/taiwan_com/t_rpg/t_zhang_qian
codex exec "Explique a estrutura deste projeto."
```

Nesse modo o Codex:

1. recebe a instrução;
2. executa a tarefa;
3. apresenta a resposta;
4. encerra.

## 9. Teste seguro

Para testar o Codex sem permitir alteração dos arquivos:

```bash
codex exec \
    --skip-git-repo-check \
    --ephemeral \
    --sandbox read-only \
    "Reply with exactly: CODEX_OK"
```

Resultado esperado:

```text
CODEX_OK
```

e:

```text
exit code: 0
```

## 10. Plugins e MCP

O Codex pode acessar ferramentas externas por mais de um mecanismo.

É importante distinguir:

### Plugins / Apps

São integrações disponibilizadas ao Codex e podem fornecer ferramentas para serviços externos.

O Codex pode selecionar essas ferramentas automaticamente quando entende que elas são necessárias.

Também é possível mencionar explicitamente a integração desejada no prompt.

Exemplo:

```text
Use o plugin do Netlify para listar meus sites.
```

Mas, em muitos casos:

```text
Liste meus sites no Netlify.
```

já é suficiente para o Codex selecionar a ferramenta correspondente.

### MCP local

Também existem servidores MCP configurados diretamente no ambiente local do Codex.

Eles podem ser consultados com:

```bash
codex mcp list
```

Na máquina `asus-taiwan`, apareceram:

```text
code-review
codex_app
cua_repl
node_repl
zotero
```

O estado observado foi:

| MCP | Estado |
|---|---|
| `code-review` | disabled |
| `codex_app` | disabled |
| `cua_repl` | enabled |
| `node_repl` | enabled |
| `zotero` | enabled |

O Zotero está configurado como um MCP local:

```text
/home/icm/.local/bin/zotero-mcp
```

## 11. Caso específico do Netlify

Durante um teste inicial apareceu:

```text
AuthRequired
```

associado a:

```text
https://netlify-mcp.netlify.app/
```

Inicialmente isso parecia indicar que seria necessário executar:

```bash
codex mcp login netlify
```

Mas a listagem mostrou:

```text
Error: No MCP server named 'netlify' found.
```

Isso não significava que o Codex não tinha acesso ao Netlify.

Significava apenas que **não existia um MCP local chamado `netlify`**.

## 12. Netlify disponível por plugin/app

Em seguida foi solicitado ao próprio Codex:

```text
Use the Netlify MCP server to run a test call to get-user. Do not modify anything.
```

O Codex encontrou automaticamente:

```text
codex_apps/netlify.netlify-user-get-user
```

A ferramenta iniciou:

```text
mcp: codex_apps/netlify.netlify-user-get-user started
```

e terminou normalmente:

```text
mcp: codex_apps/netlify.netlify-user-get-user (completed)
```

O Codex informou que o teste `get-user` foi concluído com sucesso e que a conta Netlify estava autenticada.

Nada foi modificado.

### Conclusão

O Netlify já está disponível e autenticado por meio do sistema de plugins/apps do Codex.

Não é necessário:

```bash
codex mcp login netlify
```

e não é recomendável criar agora um segundo MCP local do Netlify, pois isso poderia duplicar a integração existente.

## 13. Diferença prática entre Netlify e Zotero

Situação atual:

| Serviço | Como chega ao Codex | Estado |
|---|---|---|
| Netlify | Plugin/App do Codex | Funcionando e autenticado |
| Zotero | MCP configurado localmente | Habilitado |

Portanto, `codex mcp list` não necessariamente mostra todos os serviços que o Codex consegue utilizar.

Ele mostra principalmente os MCPs configurados naquele ambiente.

Plugins/apps podem disponibilizar ferramentas adicionais separadamente.

## 14. Uso do Netlify pelo Codex

Pode-se pedir diretamente:

```text
Liste meus sites no Netlify.
```

ou:

```text
Verifique o último deploy do meu projeto no Netlify.
```

ou, caso se queira determinar explicitamente a fonte:

```text
Use o plugin do Netlify para listar meus sites.
```

O Codex pode utilizar a ferramenta correspondente sem necessidade de abrir manualmente o MCP.

## 15. Estado final da configuração

```text
Codex CLI: 0.160.1
Instalação: standalone
Executável: ~/.local/bin/codex
Login: ChatGPT
Modelo testado: gpt-6-astra
Bubblewrap: instalado
Sandbox testado: read-only
Teste do Codex: CODEX_OK
Netlify: plugin/app funcionando e autenticado
Zotero: MCP local habilitado
```

Não há atualmente uma pendência necessária para colocar o Codex em funcionamento.

# Glossário

## `=== Environment ===`

`command -v codex`: mostra qual executável será utilizado quando o comando `codex` for chamado.

`command -v bwrap`: verifica se o executável do Bubblewrap está instalado.

`uname -m`: mostra a arquitetura da máquina.

`x86_64`: arquitetura de 64 bits utilizada pelo computador.

## `=== Installed version ===`

`codex --version`: mostra a versão instalada do Codex CLI.

`ls -lh`: mostra informações sobre um arquivo.

`$(...)`: executa um comando e utiliza sua saída como parte de outro comando.

`hash -r`: limpa o cache de localização de executáveis do Bash.

## `=== Authentication ===`

`codex login status`: verifica se o Codex possui autenticação válida.

`Logged in using ChatGPT`: indica que a autenticação está sendo feita com a conta do ChatGPT.

## `=== Updating Codex ===`

`curl`: transfere dados pela rede.

`-f`: faz o `curl` retornar erro em falhas HTTP.

`-s`: oculta a barra de progresso.

`-S`: mantém a exibição dos erros mesmo quando `-s` está ativo.

`-L`: segue redirecionamentos HTTP.

`|`: envia a saída de um comando para a entrada do comando seguinte.

`sh`: executa um script no shell.

`CODEX_NON_INTERACTIVE=true`: variável utilizada naquele teste para evitar perguntas interativas. Não é necessária nas atualizações normais.

## `=== Shell isolation ===`

`( ... )`: executa os comandos em um subshell separado.

`{ ... }`: executa um bloco no shell atual.

`set -e`: faz o shell encerrar a execução quando determinados comandos retornam erro.

`set +e`: desativa esse comportamento.

Para testes que podem falhar, preferir:

```bash
(
    ...
)
```

para proteger o shell principal.

## `=== Codex test ===`

`codex exec`: executa uma tarefa sem abrir a interface interativa.

`--skip-git-repo-check`: permite utilizar o Codex fora de um repositório Git.

`--ephemeral`: executa uma sessão temporária.

`--sandbox read-only`: permite leitura, mas impede alteração dos arquivos locais.

`$?`: contém o código de saída do último comando.

`exit code 0`: indica que o comando terminou normalmente.

## `=== Interactive Codex ===`

`cd`: muda o diretório atual.

`codex`: abre a interface interativa do Codex no diretório atual.

`codex "prompt"`: abre o Codex já enviando uma solicitação inicial.

O diretório em que o Codex é iniciado determina o ambiente local que ele utilizará como contexto de trabalho.

## `=== MCP servers ===`

`codex mcp list`: lista os servidores MCP configurados localmente no Codex.

`MCP`: Model Context Protocol; protocolo utilizado para disponibilizar ferramentas e serviços externos aos modelos.

`enabled`: integração habilitada.

`disabled`: integração presente, mas desativada.

`zotero-mcp`: servidor MCP local utilizado para integrar o Zotero ao Codex.

## `=== Netlify authentication ===`

`codex mcp login netlify`: tenta autenticar um MCP local chamado `netlify`.

`No MCP server named 'netlify' found`: indica que não existe uma entrada MCP local com esse nome. Não significa que o Codex não possua acesso ao Netlify.

## `=== Netlify test ===`

`codex_apps/netlify.netlify-user-get-user`: ferramenta do plugin/app do Netlify disponibilizada ao Codex.

`get-user`: consulta o usuário autenticado no Netlify sem realizar alterações.

`started`: a ferramenta começou a execução.

`completed`: a ferramenta terminou a execução.

O sucesso desse teste confirmou que o Netlify já estava acessível e autenticado pelo sistema de plugins/apps do Codex.
