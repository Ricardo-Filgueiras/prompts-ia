# Guia: Falar com o Claude Code e ouvir as respostas em voz alta

Este guia replica uma configuração de voz para o Claude Code:

- **Entrada por voz (STT):** você fala segurando uma tecla (hold-to-talk) e, ao soltar, a mensagem é **enviada automaticamente** — sem precisar apertar Enter.
- **Saída por voz (TTS):** ao final de cada resposta, o Claude **fala o texto em voz alta**, usando vozes neurais gratuitas da Microsoft (via `edge-tts`).
- **Interrupção ("levantar a mão para falar"):** se o Claude ainda estiver falando quando você enviar uma nova mensagem, a fala é cortada na hora, simulando uma conversa real.
- Um comando `/tts on` / `/tts off` para ligar e desligar a fala quando quiser.

Testado no Windows. Notas para Mac/Linux estão no final.

---

## Pré-requisitos

- Claude Code instalado e funcionando.
- Python 3.9+ instalado e acessível no terminal (teste com `python --version`).

---

## Passo 1 — Instalar as dependências Python

```bash
pip install --user edge-tts "playsound==1.2.2"
```

- `edge-tts`: gera áudio a partir de texto usando o mesmo serviço gratuito do "Ler em voz alta" do navegador Edge. Não precisa de conta, chave de API nem assinatura. Único requisito: internet.
- `playsound`: reproduz o arquivo de áudio gerado.

Descubra o caminho completo do seu Python (vai precisar dele no Passo 3):

```bash
where python      # Windows
which python3     # Mac/Linux
```

---

## Passo 2 — Criar o script que fala as respostas

Crie a pasta `scripts` dentro da sua pasta de configuração do Claude Code (`~/.claude/scripts/`) e salve o arquivo abaixo como `tts_speak.py`:

```python
"""Stop hook: le a ultima resposta do Claude no transcript e fala em voz alta via edge-tts.

Ativa/desativa via arquivo ~/.claude/tts_enabled (conteudo "on" ou "off").
Falha silenciosamente em qualquer erro para nunca travar o Claude Code.
"""
import asyncio
import json
import os
import re
import sys
import tempfile
from pathlib import Path

VOICE = "pt-BR-ThalitaMultilingualNeural"
MAX_CHARS = 3000
TOGGLE_FILE = Path.home() / ".claude" / "tts_enabled"
PID_FILE = Path.home() / ".claude" / "tts_pid"


def is_enabled() -> bool:
    try:
        return TOGGLE_FILE.read_text(encoding="utf-8").strip().lower() == "on"
    except FileNotFoundError:
        return False


def last_assistant_text(transcript_path: str) -> str:
    lines = Path(transcript_path).read_text(encoding="utf-8").splitlines()
    for line in reversed(lines):
        if not line.strip():
            continue
        entry = json.loads(line)
        if entry.get("type") != "assistant":
            continue
        content = entry.get("message", {}).get("content", [])
        texts = [b.get("text", "") for b in content if isinstance(b, dict) and b.get("type") == "text"]
        if texts:
            return "\n".join(texts)
    return ""


def clean_for_speech(text: str) -> str:
    text = re.sub(r"```.*?```", " bloco de codigo omitido. ", text, flags=re.DOTALL)
    text = re.sub(r"`([^`]*)`", r"\1", text)
    text = re.sub(r"!\[[^\]]*\]\([^)]*\)", " ", text)
    text = re.sub(r"\[([^\]]*)\]\([^)]*\)", r"\1", text)
    text = re.sub(r"[*_#>]+", " ", text)
    text = re.sub(r"^\s*[-•]\s+", " ", text, flags=re.MULTILINE)
    text = re.sub(r"\s+", " ", text).strip()
    if len(text) > MAX_CHARS:
        text = text[:MAX_CHARS].rsplit(" ", 1)[0] + "..."
    return text


async def synthesize(text: str, out_path: str) -> None:
    import edge_tts

    communicate = edge_tts.Communicate(text, VOICE)
    await communicate.save(out_path)


def main() -> None:
    try:
        if not is_enabled():
            return
        hook_input = json.load(sys.stdin)
        transcript_path = hook_input.get("transcript_path")
        if not transcript_path or not Path(transcript_path).exists():
            return
        text = clean_for_speech(last_assistant_text(transcript_path))
        if not text:
            return

        fd, mp3_path = tempfile.mkstemp(suffix=".mp3", prefix="claude_tts_")
        os.close(fd)
        try:
            asyncio.run(synthesize(text, mp3_path))
            PID_FILE.write_text(str(os.getpid()), encoding="utf-8")
            from playsound import playsound

            playsound(mp3_path)
        finally:
            try:
                PID_FILE.unlink()
            except OSError:
                pass
            try:
                os.remove(mp3_path)
            except OSError:
                pass
    except Exception:
        return


if __name__ == "__main__":
    main()
```

---

## Passo 2.1 — Criar o script que interrompe a fala

O Claude Code não tem um evento de hook para "tecla pressionada" ou "gravação iniciada" — só existe um hook para **mensagem enviada** (`UserPromptSubmit`). Como o modo de voz usa `autoSubmit: true`, soltar a tecla equivale a enviar a mensagem quase instantaneamente. Por isso, a interrupção funciona assim: **no momento em que você envia uma nova mensagem, qualquer fala em andamento é cortada** — não é possível interromper no instante exato em que você começa a segurar a tecla, pois esse evento não é exposto pelo Claude Code.

Salve o arquivo abaixo como `~/.claude/scripts/tts_interrupt.py`:

```python
"""UserPromptSubmit hook: interrompe a fala do TTS em andamento quando o usuario envia uma nova mensagem.

Simula "levantar a mao para falar": mata o processo que esta tocando o audio (se houver)
e limpa arquivos de audio temporarios que possam ter ficado orfaos.
"""
import glob
import os
import subprocess
import tempfile
from pathlib import Path

PID_FILE = Path.home() / ".claude" / "tts_pid"


def kill_current_speech() -> None:
    try:
        pid = int(PID_FILE.read_text(encoding="utf-8").strip())
    except (FileNotFoundError, ValueError):
        return
    try:
        subprocess.run(
            ["taskkill", "/F", "/PID", str(pid)],
            capture_output=True,
            timeout=5,
        )
    except Exception:
        pass
    try:
        PID_FILE.unlink()
    except OSError:
        pass


def cleanup_orphan_audio() -> None:
    pattern = os.path.join(tempfile.gettempdir(), "claude_tts_*.mp3")
    for path in glob.glob(pattern):
        try:
            os.remove(path)
        except OSError:
            pass


def main() -> None:
    try:
        kill_current_speech()
        cleanup_orphan_audio()
    except Exception:
        return


if __name__ == "__main__":
    main()
```

> **Nota Mac/Linux:** `taskkill /F /PID` é específico do Windows. Em Mac/Linux, troque essa chamada por `os.kill(pid, signal.SIGKILL)` (via `import signal`).

---

## Passo 3 — Configurar os hooks e a entrada por voz no `settings.json`

Abra (ou crie) o arquivo `~/.claude/settings.json`. **Importante: mescle com o conteúdo que já existir, não substitua o arquivo inteiro.**

Adicione estas chaves:

```json
{
  "hooks": {
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "CAMINHO_DO_SEU_PYTHON",
            "args": ["CAMINHO_PARA/.claude/scripts/tts_speak.py"],
            "async": true,
            "timeout": 60,
            "statusMessage": "Falando resposta..."
          }
        ]
      }
    ],
    "UserPromptSubmit": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "CAMINHO_DO_SEU_PYTHON",
            "args": ["CAMINHO_PARA/.claude/scripts/tts_interrupt.py"],
            "timeout": 10,
            "statusMessage": "Interrompendo fala..."
          }
        ]
      }
    ]
  },
  "voice": {
    "enabled": true,
    "mode": "hold",
    "autoSubmit": true
  }
}
```

Troque:
- `CAMINHO_DO_SEU_PYTHON` pelo caminho encontrado no Passo 1 (ex: `C:\\Python314\\python.exe` no Windows, ou `/usr/bin/python3` no Mac/Linux).
- `CAMINHO_PARA/.claude/scripts/tts_speak.py` e `tts_interrupt.py` pelo caminho completo dos arquivos criados nos Passos 2 e 2.1 (ex: `C:\\Users\\SeuUsuario\\.claude\\scripts\\tts_speak.py`).

O que cada campo faz:
- `hooks.Stop`: dispara o script de fala automaticamente ao final de cada resposta do Claude. `"async": true` faz isso rodar em segundo plano, sem travar o terminal.
- `hooks.UserPromptSubmit`: dispara o script de interrupção sempre que uma nova mensagem é enviada, cortando qualquer fala em andamento. Roda de forma síncrona (sem `"async"`) porque precisa completar rapidamente antes do turno seguir — mas como é só um "matar processo", é instantâneo.
- `voice.mode: "hold"`: modo "segurar para falar".
- `voice.autoSubmit: true`: envia a mensagem automaticamente ao soltar a tecla (sem precisar de Enter).

---

## Passo 4 — Criar o arquivo de ativação (toggle)

```bash
mkdir -p ~/.claude
printf 'on' > ~/.claude/tts_enabled
```

O script só fala se esse arquivo contiver exatamente `on`.

---

## Passo 5 — Criar o comando `/tts on` / `/tts off`

Salve o arquivo abaixo como `~/.claude/commands/tts.md`:

```markdown
---
description: Ativa ou desativa a leitura em voz alta (TTS) das respostas do Claude
argument-hint: on|off
---

O usuário passou o argumento: "$1"

Se o argumento for exatamente "on" ou "off" (minúsculas), execute este comando via Bash, sem pedir confirmação:

​```
mkdir -p "$HOME/.claude" && printf '%s' "$1" > "$HOME/.claude/tts_enabled"
​```

Se o argumento não for "on" nem "off" (ex: vazio, ou outro valor), NÃO execute nada — pergunte ao usuário se ele quer ativar ou desativar o TTS.

Depois de executar, confirme ao usuário em português se o TTS foi ativado ou desativado.
```

(Remova os `​` invisíveis antes dos crases triplas ao colar — foram usados aqui só para não quebrar a formatação deste guia.)

---

## Passo 6 — Reiniciar o Claude Code

Hooks são carregados na inicialização da sessão. Feche e abra uma nova conversa (ou use `/hooks` para forçar o recarregamento) para a configuração entrar em vigor.

---

## Testando

1. Segure a tecla configurada para falar, diga algo, solte — a mensagem deve ser enviada sozinha, sem Enter.
2. Espere o Claude responder — ao final, a resposta deve ser lida em voz alta.
3. Enquanto o Claude ainda estiver falando, segure a tecla de voz de novo, diga algo e solte — a fala deve ser cortada na hora em que a nova mensagem for enviada.
4. Para desativar a fala temporariamente: `/tts off`. Para reativar: `/tts on`.

---

## Personalizando

**Trocar a voz** — edite a constante `VOICE` no `tts_speak.py`. Para ver todas as vozes disponíveis:

```bash
python -m edge_tts --list-voices
```

Vozes em português mais úteis:

| Voz | Gênero | Estilo |
|---|---|---|
| `pt-BR-FranciscaNeural` | Feminina | Amigável |
| `pt-BR-AntonioNeural` | Masculina | Amigável |
| `pt-BR-ThalitaMultilingualNeural` | Feminina | Amigável, lida melhor com termos técnicos/inglês misturado |
| `pt-PT-DuarteNeural` / `pt-PT-RaquelNeural` | Masc./Fem. | Sotaque de Portugal |

**Tamanho máximo do texto falado** — constante `MAX_CHARS` no script (padrão: 3000 caracteres, ~500-600 palavras; respostas maiores que isso são cortadas com "..."). Aumentar demais faz respostas longas falarem por vários minutos sem parar.

---

## Notas para Mac/Linux

- No `command` do hook, use o caminho do `python3` (`which python3`), não `python`.
- `playsound` no Linux pode exigir dependências de sistema (ex: GStreamer) para tocar `.mp3`. Se der erro, troque a chamada de `playsound` no script por uma chamada via `subprocess` para `afplay` (Mac) ou `mpg123`/`ffplay` (Linux).
- Os caminhos com `~` (`Path.home()`) funcionam igual em todos os sistemas — não precisa adaptar essa parte do script.
