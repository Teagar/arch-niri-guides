# OpenCode e OpenChamber no mesmo servidor

## Objetivo

Usar o OpenCode pelo terminal no Arch Linux e continuar a mesma sessão pelo aplicativo Android do OpenChamber, com respostas, perguntas e permissões sincronizadas entre o computador e o celular.

O Niri e o Wayland não interferem nesse fluxo: a comunicação ocorre entre serviços HTTP locais e, no acesso remoto, pelo Private Relay do OpenChamber.

## Ambiente verificado

- Arch Linux com Niri/Wayland
- OpenCode 2.0.22 instalado pelo repositório oficial do Arch
- OpenChamber 2.1.0
- aplicativo OpenChamber para Android
- OpenCode acessível somente por `127.0.0.1`
- celular pareado por QR code com a opção **Anywhere**

## Por que as respostas não apareciam no terminal

Por padrão, os programas podem iniciar servidores diferentes:

- `opencode` conecta ao serviço compartilhado do OpenCode;
- OpenChamber, quando não encontra um servidor configurado, inicia e gerencia outra instância do OpenCode.

As duas interfaces podem mostrar sessões armazenadas no mesmo usuário, mas não acompanham necessariamente os mesmos eventos em tempo real. Uma resposta enviada pelo celular pode, portanto, não aparecer no TUI que está conectado ao outro processo.

O diagnóstico pode ser feito com:

```bash
opencode service status
openchamber status
openchamber logs
```

Nos logs, a mensagem `Starting OpenCode on allocated port ...` indica que o OpenChamber iniciou sua própria instância. Depois da configuração correta, deve aparecer algo semelhante a:

```text
Using external OpenCode server at http://127.0.0.1:49374 (skip-start mode)
Detected OpenCode port: 49374
[PushWatcher] connected
```

A porta é um exemplo e pode ser diferente.

## Configuração aplicada

### 1. Confirmar e atualizar os pacotes

No Arch, evite instalar outra cópia do OpenCode por `curl` sobre o pacote do `pacman`:

```bash
sudo pacman -Syu opencode
opencode --version
openchamber --version
```

### 2. Iniciar o serviço compartilhado do OpenCode

```bash
opencode service start
opencode service status
```

O segundo comando imprime a origem local do serviço, por exemplo:

```text
http://127.0.0.1:49374
```

### 3. Reiniciar o OpenChamber no mesmo servidor

O serviço compartilhado é protegido por senha. Passe a origem e a senha diretamente pelos comandos do OpenCode, sem copiá-la para o histórico do terminal:

```bash
OPENCODE_HOST="$(opencode service status)" \
OPENCODE_SKIP_START=true \
OPENCODE_PASSWORD="$(opencode service get password)" \
openchamber restart
```

Essa configuração:

- manda o OpenChamber usar o servidor já utilizado pelo TUI;
- impede o OpenChamber de iniciar uma segunda instância;
- mantém o servidor restrito a `127.0.0.1`;
- não abre a porta do OpenCode no roteador nem na rede local.

Depois, confirme:

```bash
openchamber status
openchamber logs
```

Procure por `Using external OpenCode server` e `[PushWatcher] connected`. Um erro `401 Unauthorized` indica que `OPENCODE_PASSWORD` não foi fornecida ou não corresponde à senha atual do serviço.

## Uso diário

Abra o TUI normalmente:

```bash
opencode
```

No celular, abra a mesma sessão pelo OpenChamber. Se o TUI já estava aberto durante a troca de servidor, feche-o e abra novamente para renovar a conexão.

Evite enviar duas mensagens ao mesmo tempo, uma pelo terminal e outra pelo celular, para a mesma sessão. Uma pergunta ou pedido de permissão respondido em uma interface fica resolvido para a outra também.

## O fluxo do agente muda?

Compartilhar o servidor não altera a forma de raciocínio do OpenCode. Permanecem os mesmos:

- modelo e nível de raciocínio;
- agente, instruções e skills;
- ferramentas e servidores MCP;
- contexto e histórico da sessão;
- regras de permissão;
- consumo de tokens para a mesma tarefa.

O resultado pode mudar se, pelo OpenChamber, forem selecionados outro modelo, agente ou nível de raciocínio, ou se forem ativados recursos como **Auto Model Routing**, **Session Goals**, **Safety Net** ou **Multi-run**.

Para reproduzir o fluxo tradicional do TUI, use o mesmo modelo, agente e esforço de raciocínio nas duas interfaces e deixe esses recursos adicionais desativados.

## Acesso remoto seguro

Mantenha o OpenCode em `127.0.0.1`. O aplicativo móvel deve acessar o computador pelo pareamento do OpenChamber:

```bash
openchamber connect-url --relay --qr
```

No Android, remova um pareamento antigo se necessário, toque em **Scan QR code** e leia o novo código. O modo **Anywhere** usa o Private Relay com criptografia ponta a ponta e não exige encaminhamento de portas.

Não configure o servidor OpenCode em `0.0.0.0` e não exponha sua porta diretamente à internet.

## Depois de reiniciar o computador

Se o OpenChamber voltar a criar uma instância própria depois de reiniciar o sistema, execute novamente:

```bash
opencode service start

OPENCODE_HOST="$(opencode service status)" \
OPENCODE_SKIP_START=true \
OPENCODE_PASSWORD="$(opencode service get password)" \
openchamber restart
```

Isso não apaga sessões, configurações ou credenciais. Não é necessário remover `~/.config/opencode`, reinstalar o OpenCode nem refazer o pareamento enquanto a identidade do OpenChamber permanecer a mesma.

## Desvantagens do servidor compartilhado

- se o servidor reiniciar, TUI e OpenChamber desconectam juntos;
- ações simultâneas na mesma sessão podem deixar o fluxo confuso;
- permissões e perguntas são compartilhadas entre as interfaces;
- uma alteração de modelo ou agente afeta os próximos turnos daquela sessão;
- todas as interfaces dividem o mesmo processo e seus recursos.

Para uso individual, essas limitações normalmente são menores que a vantagem de continuar a mesma sessão no computador e no celular.
