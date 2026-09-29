# `kz`

Namespace utilizado para os comandos pessoais criados no projeto Tech Linux.

## Uso

```bash
kz <command> [arguments]
```

Cada subcomando corresponde a um executável localizado em:

```text
~/.local/bin/kz-<command>
```

Por exemplo:

```bash
kz update
```

executa:

```text
~/.local/bin/kz-update
```

## Comandos auxiliares

Listar os comandos disponíveis:

```bash
kz list
```

Exibir ajuda:

```bash
kz help
```

## Criando um novo subcomando

Um novo subcomando é criado como um script dentro de:

```text
~/.local/bin/
```

Por exemplo, para criar `kz hello`:

```bash
gedit ~/.local/bin/kz-hello
```

O arquivo pode conter:

```bash
#!/usr/bin/env bash

echo "Hello"
```

Depois, torne-o executável:

```bash
chmod +x ~/.local/bin/kz-hello
```

Ele poderá ser usado como:

```bash
kz hello
```
