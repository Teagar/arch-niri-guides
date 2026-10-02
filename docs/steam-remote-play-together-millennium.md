# Steam Remote Play Together no Niri com modal de convite travado

## Sintoma

No Arch Linux com Wayland e Niri, o Steam Remote Play Together reconhecia o jogo e mostrava a tela **Invite Anyone to Play**, mas os botões do modal não respondiam aos cliques.

O problema ocorreu com o Steam customizado pelo Millennium, usando o tema `Material-Theme` e injeção de CSS e JavaScript.

## Ambiente verificado

- Arch Linux
- Niri em sessão Wayland iniciada por `niri-session`
- Steam nativo do repositório Arch
- PipeWire + WirePlumber
- `xdg-desktop-portal-gnome`
- `xwayland-satellite`
- Millennium 3.5.0 com `Material-Theme`

## Diagnóstico

Primeiro, confirme que os componentes de captura estão instalados:

```bash
pacman -Q \
  steam \
  pipewire \
  pipewire-audio \
  lib32-pipewire \
  wireplumber \
  xdg-desktop-portal \
  xdg-desktop-portal-gnome \
  xdg-desktop-portal-gtk \
  xwayland-satellite
```

Confirme os serviços:

```bash
systemctl --user status \
  pipewire.service \
  wireplumber.service \
  xdg-desktop-portal.service \
  xdg-desktop-portal-gnome.service
```

Confirme a sessão:

```bash
printf 'desktop=%s\nsession=%s\n' \
  "$XDG_CURRENT_DESKTOP" \
  "$XDG_SESSION_TYPE"
```

O resultado esperado é `desktop=niri` e `session=wayland`.

Os registros do Steam ficam em:

```text
~/.local/share/Steam/logs/
```

Arquivos úteis:

- `streaming_log.txt`: captura de vídeo e PipeWire;
- `remote_connections.txt`: conexões Remote Play;
- `webhelper_js.txt`: erros da interface web do Steam;
- `console_log.txt`: inicialização de jogos e cliente de streaming.

Neste caso, a captura PipeWire abria corretamente, mas nenhum cliente chegava a conectar. Ao mesmo tempo, `webhelper_js.txt` registrava erro JavaScript na interface modificada. Isso isolou a falha no modal do Steam/Millennium, antes da etapa de transmissão.

## Correção aplicada

### 1. Fechar o jogo e o Steam

```bash
steam -shutdown
```

Espere até não haver processos do Steam:

```bash
pgrep -a steam
```

### 2. Fazer backup da configuração do Millennium

```bash
cp ~/.config/millennium/config.json \
  ~/.config/millennium/config.json.before-remote-play
```

### 3. Desativar temporariamente as injeções

Edite:

```text
~/.config/millennium/config.json
```

Na seção `general`, altere:

```json
{
  "general": {
    "injectCSS": false,
    "injectJavascript": false
  }
}
```

Não substitua o arquivo inteiro por esse trecho; preserve as demais opções existentes.

Uma forma segura de alterar somente essas duas chaves é:

```bash
python - <<'PY'
import json
from pathlib import Path

path = Path.home() / ".config/millennium/config.json"
data = json.loads(path.read_text())
data["general"]["injectCSS"] = False
data["general"]["injectJavascript"] = False
path.write_text(json.dumps(data, indent=2, ensure_ascii=False) + "\n")
PY
```

### 4. Reiniciar o Steam com captura PipeWire

```bash
steam -pipewire
```

Confirme que o argumento foi aplicado:

```bash
pgrep -a -f 'steam.*pipewire'
```

Ao iniciar, o Steam pode pedir autorização do portal para capturar uma tela. Selecione o monitor usado pelo jogo e permita a interação remota quando solicitado.

### 5. Criar os convites

1. Abra o jogo compatível com Remote Play Together.
2. Pressione `Shift+Tab`.
3. Abra **Remote Play Together**.
4. Use **Invite Anyone to Play** ou convide pela lista de amigos.
5. Não abra no host os links criados para os convidados.

Depois de desativar as injeções do Millennium e reiniciar o Steam, o modal voltou a aceitar cliques e o convite funcionou.

## Como restaurar o Millennium

Feche o Steam e restaure o backup:

```bash
steam -shutdown
cp ~/.config/millennium/config.json.before-remote-play \
  ~/.config/millennium/config.json
steam -pipewire
```

Também é possível manter o arquivo atual e mudar `injectCSS` e `injectJavascript` novamente para `true`. Se o modal voltar a travar, mantenha as injeções desativadas durante sessões de Remote Play Together.

## Observações

- O sistema operacional dos convidados não foi a causa: clientes Windows e Linux podem participar da mesma sessão.
- O portal e o PipeWire são relevantes no computador host, responsável por capturar o jogo.
- Avisos de `libcamera` no WirePlumber normalmente afetam câmeras, não o Remote Play.
- URLs de convite incluem credenciais temporárias. Não publique esses links em issues, logs ou capturas de tela.
- A relação com o Millennium foi confirmada operacionalmente: o modal passou a funcionar após desativar suas injeções. Um tema ou plugin específico pode ser a causa mais restrita; reative-os individualmente para identificar o componente exato.
