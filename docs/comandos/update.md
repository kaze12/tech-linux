# `kz update`

Atualiza os principais gerenciadores de pacotes do sistema e executa a limpeza posterior.

## Uso

```bash
kz update
```

O comando executa o script:

```text
~/.local/bin/kz-update
```

## Operações realizadas

Na ordem:

1. Atualiza pacotes Snap:

```bash
sudo snap refresh
```

2. Atualiza o índice de pacotes APT:

```bash
sudo apt update
```

3. Instala atualizações disponíveis pelo APT:

```bash
sudo apt upgrade -y
```

4. Atualiza aplicativos e runtimes Flatpak:

```bash
flatpak update -y
```

5. Remove dependências APT que não são mais necessárias:

```bash
sudo apt autoremove -y
```

6. Limpa pacotes antigos do cache APT:

```bash
sudo apt autoclean
```

7. Mostra se ainda existe algum pacote atualizável:

```bash
apt list --upgradable
```

## Comportamento do script

O script utiliza:

```bash
set -e
```

para interromper a execução caso algum comando falhe.

Também utiliza:

```bash
sudo -v
```

no início para solicitar e validar as credenciais administrativas antes de iniciar a sequência de atualização.
