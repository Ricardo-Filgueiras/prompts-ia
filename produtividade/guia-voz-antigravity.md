# Guia: Falar com o Google Antigravity e ouvir as respostas em voz alta

Este guia implementa uma configuração completa de voz para o **Google Antigravity (AGY CLI e Antigravity IDE)**:

- **Entrada por voz (STT):** utilize o ditado inteligente do Windows (`Win + H`) ou microfone integrado no terminal/IDE para enviar comandos por voz diretamente.
- **Saída por voz (TTS):** o Antigravity fala suas respostas em voz alta utilizando vozes neurais de alta definição gratuitas da Microsoft (via `edge-tts`). A fala começa quase instantaneamente (1 a 2 segundos) através de processamento em chunks (partes assíncronas).
- **Dois Modos de Operação (Flexibilidade Total):**
  1. **Modo MCP Server (Recomendado):** o agente possui a ferramenta nativa `falar_com_usuario`, narrando explicações concisas e humanas sem ler blocos de código ou URLs longas.
  2. **Modo Lifecycle Hooks (Automático):** ao final de qualquer comando ou resposta (`Stop`), o áudio é sintetizado e executado em segundo plano.
- **Interrupção Automática:** se o agente estiver falando quando você enviar uma nova instrução, o hook `PreInvocation` corta a fala imediatamente, simulando um diálogo real.
- **Controle Rápido (`on` / `off`):** habilite ou desabilite o áudio a qualquer momento via arquivo de controle ou comando.

Testado no Windows (com compatibilidade documentada para macOS e Linux).

---

## 🏛️ Onde Usar

Esta solução foi desenvolvida especificamente para o ecossistema do Google Antigravity:

| Ambiente | Suporte MCP (`falar_com_usuario`) | Suporte Hooks (`Stop` / `PreInvocation`) | Entrada por Voz (STT) |
|---|---|---|---|
| **Antigravity IDE** | Sim (Tool nativa) | Sim (`hooks.json`) | `Win + H` (Ditado do Windows) |
| **Antigravity CLI (`agy`)** | Sim (Tool nativa) | Sim (`hooks.json`) | Ditado do sistema ou microfone |

---

## 📋 Pré-requisitos

1. **Google Antigravity** instalado e funcional.
2. **Python 3.9+** instalado e adicionado ao `PATH` do sistema (verifique com `python --version`).
3. **Acesso à Internet** (o `edge-tts` conecta aos servidores da Microsoft).

Descubra o caminho completo do seu executável Python:

```bash
where python      # No Windows (ex: C:\Python314\python.exe)
which python3     # No macOS / Linux
```

---

## Passo 1 — Instalar as dependências Python

No terminal, instale os pacotes necessários:

```bash
pip install --user --upgrade edge-tts "playsound==1.2.2"
```

* **`edge-tts`**: biblioteca que consome o serviço neural do Microsoft Edge gratuitamente, sem necessidade de chaves de API ou cadastros.
* **`playsound==1.2.2`**: biblioteca leve para execução do áudio `.mp3` gerado no Windows.

---

## Passo 2 — Estrutura de Diretórios

O Antigravity carrega configurações de dois níveis:
- **Global:** pasta `~/.gemini/config/` (afeta todos os seus projetos).
- **Workspace:** pasta `.agents/` na raiz do seu repositório (específico do projeto).

Crie a pasta de scripts caso ainda não exista:

```bash
# Windows PowerShell
mkdir -p "$HOME\.gemini\config\scripts"
```

---

## Passo 3 — Criar o Script de Síntese (`tts_speak.py`)

Salve o script abaixo em `~/.gemini/config/scripts/tts_speak.py`:

```python
"""Stop hook & TTS Worker para Google Antigravity.
Sintetiza e reproduz áudio via edge-tts em streaming assíncrono por chunks.
"""
import asyncio
import json
import os
import queue
import re
import sys
import tempfile
import threading
import time
import traceback
from datetime import datetime
from pathlib import Path

VOICE = "pt-BR-FranciscaNeural"
RATE = "+0%"
MAX_CHARS = 3000
FIRST_CHUNK_CHARS = 200
CHUNK_CHARS = 600

CONFIG_DIR = Path.home() / ".gemini" / "config"
TOGGLE_FILE = CONFIG_DIR / "tts_enabled"
PID_FILE = CONFIG_DIR / "tts_pid.txt"
LOG_FILE = CONFIG_DIR / "tts_log.txt"

WIN_PATH = re.compile(r"[A-Za-z]:[\\/](?:[^\\/\n`\"<>|*?]*[\\/])*[^\\/\s`\"<>|*?,;:)]*")
UNIX_PATH = re.compile(r"(?<![\w:/])(?:~|\.{1,2})?/(?:[\w.\-]+/)+[\w.\-]*")
TABLE_SEPARATOR = re.compile(r"^\s*\|?[\s:|-]*-{3,}[\s:|-]*$")


def is_enabled() -> bool:
    try:
        return TOGGLE_FILE.read_text(encoding="utf-8").strip().lower() == "on"
    except FileNotFoundError:
        return True  # Habilitado por padrão se o arquivo não existir


def log(msg: str) -> None:
    try:
        if LOG_FILE.exists() and LOG_FILE.stat().st_size > 100_000:
            LOG_FILE.unlink()
        with LOG_FILE.open("a", encoding="utf-8") as f:
            f.write(f"{datetime.now():%Y-%m-%d %H:%M:%S} {msg}\n")
    except OSError:
        pass


def clean_for_speech(text: str) -> str:
    text = re.sub(r"```.*?```", "\nbloco de código omitido.\n", text, flags=re.DOTALL)
    text = re.sub(r"!\[[^\]]*\]\([^)]*\)", " ", text)
    text = re.sub(r"\[([^\]]*)\]\([^)]*\)", r"\1", text)
    text = re.sub(r"https?://\S+", "link", text)
    text = WIN_PATH.sub(lambda m: [p for p in re.split(r"[\\/]", m.group(0)) if p][-1], text)
    text = UNIX_PATH.sub(lambda m: [p for p in re.split(r"[\\/]", m.group(0)) if p][-1], text)
    text = text.replace("`", "")

    lines = []
    for line in text.splitlines():
        if TABLE_SEPARATOR.match(line):
            continue
        if "|" in line:
            cells = [c.strip() for c in line.strip().strip("|").split("|")]
            line = ", ".join(c for c in cells if c)
        line = re.sub(r"^\s*[-•]\s+", "", line)
        line = re.sub(r"[*_#>]+", " ", line).strip()
        if not line:
            continue
        if line[-1] not in ".!?:;,":
            line += "."
        lines.append(line)

    clean_text = " ".join(lines)
    if len(clean_text) > MAX_CHARS:
        clean_text = clean_text[:MAX_CHARS].rsplit(" ", 1)[0] + "..."
    return clean_text


def split_chunks(text: str) -> list:
    sentences = re.split(r"(?<=[.!?:;])\s+", text)
    chunks, current = [], ""
    for sentence in sentences:
        limit = FIRST_CHUNK_CHARS if not chunks else CHUNK_CHARS
        if current and len(current) + 1 + len(sentence) > limit:
            chunks.append(current)
            current = sentence
        else:
            current = f"{current} {sentence}".strip()
    if current:
        chunks.append(current)
    return chunks


def synthesize(text: str, out_path: str) -> None:
    import edge_tts
    for attempt in range(1, 4):
        try:
            asyncio.run(edge_tts.Communicate(text, VOICE, rate=RATE).save(out_path))
            return
        except Exception as e:
            log(f"Sintese tentativa {attempt} falhou: {e}")
            if attempt == 3:
                raise
            time.sleep(1)


def speak_text(text: str) -> None:
    text = clean_for_speech(text)
    if not text:
        return
    chunks = split_chunks(text)
    prefix = os.path.join(tempfile.gettempdir(), f"agy_tts_{os.getpid()}_")
    ready = queue.Queue()

    def produce():
        try:
            for i, chunk in enumerate(chunks):
                path = f"{prefix}{i}.mp3"
                synthesize(chunk, path)
                ready.put(path)
        except Exception:
            log("Erro ao sintetizar áudio:\n" + traceback.format_exc())
        finally:
            ready.put(None)

    threading.Thread(target=produce, daemon=True).start()
    from playsound import playsound

    while (path := ready.get()) is not None:
        try:
            playsound(path)
        finally:
            try:
                os.remove(path)
            except OSError:
                pass


def main() -> None:
    if not is_enabled():
        return
    PID_FILE.write_text(str(os.getpid()), encoding="utf-8")
    try:
        if len(sys.argv) > 1:
            msg = " ".join(sys.argv[1:])
        else:
            msg = sys.stdin.read().strip()
        if msg:
            speak_text(msg)
    finally:
        try:
            if PID_FILE.exists() and PID_FILE.read_text(encoding="utf-8").strip() == str(os.getpid()):
                PID_FILE.unlink()
        except OSError:
            pass


if __name__ == "__main__":
    main()
```

---

## Passo 4 — Criar o Script de Interrupção (`tts_interrupt.py`)

Salve o script abaixo em `~/.gemini/config/scripts/tts_interrupt.py`:

```python
"""PreInvocation Hook para cortar falas em andamento ao iniciar nova ação."""
import glob
import os
import subprocess
import tempfile
from pathlib import Path

PID_FILE = Path.home() / ".gemini" / "config" / "tts_pid.txt"


def stop_audio():
    if PID_FILE.exists():
        try:
            pid = int(PID_FILE.read_text(encoding="utf-8").strip())
            subprocess.run(["taskkill", "/F", "/PID", str(pid)], capture_output=True, timeout=5)
        except Exception:
            pass
        try:
            PID_FILE.unlink()
        except OSError:
            pass

    # Limpeza de resíduos de áudio temporários
    for f in glob.glob(os.path.join(tempfile.gettempdir(), "agy_tts_*.mp3")):
        try:
            os.remove(f)
        except OSError:
            pass


if __name__ == "__main__":
    stop_audio()
```

---

## Passo 5 — Criar o Servidor MCP de Voz (`voice_mcp_server.py`)

O servidor MCP permite que o Antigravity utilize a ferramenta `falar_com_usuario` diretamente durante o raciocínio ou ao responder.

Salve em `~/.gemini/config/scripts/voice_mcp_server.py`:

```python
"""Servidor MCP em Python para Saída de Voz do Antigravity."""
import asyncio
import os
import subprocess
import sys
from pathlib import Path
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("voice-tts")
SPEAK_SCRIPT = Path(__file__).parent / "tts_speak.py"


@mcp.tool()
def falar_com_usuario(mensagem: str) -> str:
    """Fala a mensagem especificada em voz alta para o usuário via Text-to-Speech (edge-tts).
    
    Use esta ferramenta para narrar em voz alta mensagens importantes ou respostas enviadas ao usuário.
    """
    try:
        subprocess.run(
            [sys.executable, str(SPEAK_SCRIPT), mensagem],
            check=True,
            timeout=45,
            capture_output=True
        )
        return "Mensagem falada em voz alta com sucesso."
    except Exception as e:
        return f"Erro ao executar voz: {e}"


if __name__ == "__main__":
    mcp.run()
```

---

## Passo 6 — Integrar Hooks e MCP no Antigravity

### 1. `mcp_config.json`
Abra ou crie `~/.gemini/config/mcp_config.json` (ou `.agents/mcp_config.json` no seu projeto):

```json
{
  "mcpServers": {
    "voice-tts": {
      "command": "C:\\Python314\\python.exe",
      "args": [
        "C:\\Users\\Campeao Lub\\.gemini\\config\\scripts\\voice_mcp_server.py"
      ]
    }
  }
}
```
*(Substitua os caminhos pelo seu executável do Python e caminho do seu usuário).*

### 2. `hooks.json`
Abra ou crie `~/.gemini/config/hooks.json` (ou `.agents/hooks.json`):

```json
{
  "voice-tts": {
    "enabled": true,
    "Stop": [
      {
        "type": "command",
        "command": "C:\\Python314\\python.exe",
        "args": [
          "C:\\Users\\Campeao Lub\\.gemini\\config\\scripts\\tts_speak.py"
        ],
        "async": true,
        "timeout": 60
      }
    ],
    "PreInvocation": [
      {
        "type": "command",
        "command": "C:\\Python314\\python.exe",
        "args": [
          "C:\\Users\\Campeao Lub\\.gemini\\config\\scripts\\tts_interrupt.py"
        ],
        "timeout": 5
      }
    ]
  }
}
```

### 3. Regra de Voz do Agente (`rules/voice.md`)
Crie o arquivo em `~/.gemini/config/rules/voice.md` (ou `.agents/rules/voice.md`):

````markdown
# Regra de Saída de Voz (MCP Voice)

Sempre que você enviar uma resposta, explicação ou mensagem ao usuário, você DEVE utilizar a ferramenta `falar_com_usuario` para narrar a sua mensagem em voz alta.
````

---

## Passo 7 — Controle de Ativação (`tts_enabled`)

Para ligar ou desligar a fala sem reiniciar nada:

* **Ligar o áudio**:
  ```powershell
  Set-Content -Path "$HOME\.gemini\config\tts_enabled" -Value "on"
  ```
* **Desligar o áudio**:
  ```powershell
  Set-Content -Path "$HOME\.gemini\config\tts_enabled" -Value "off"
  ```

---

## 🎙️ Vozes Disponíveis

Para listar todas as vozes neurais instaladas no serviço:

```bash
python -m edge_tts --list-voices
```

Principais vozes em Português:

| Voz | Gênero | Características |
|---|---|---|
| `pt-BR-ThalitaMultilingualNeural` | Feminina | **Recomendada:** Multilíngue, pronuncia termos técnicos e termos em inglês com perfeição |
| `pt-BR-FranciscaNeural` | Feminina | Natural, fluida e amigável |
| `pt-BR-AntonioNeural` | Masculina | Voz executiva, clara e profissional |
| `pt-PT-DuarteNeural` | Masculina | Sotaque de Portugal |

Para alterar, edite a variável `VOICE` no cabeçalho de `tts_speak.py`.

---

## 🛠️ Solução de Problemas

| Sintoma | Causa Mais Comum | Solução |
|---|---|---|
| `NoAudioReceived` ou erro de síntese | Queda momentânea na conexão com a Microsoft ou caracteres inválidos | O script tenta 3 vezes automaticamente; verifique sua conexão com a internet |
| `WSServerHandshakeError: 403` | Pacote `edge-tts` desatualizado | Execute `pip install --user --upgrade edge-tts` |
| Sem som ao rodar o comando | Volume do sistema ou conflito na versão do `playsound` | Verifique se está usando exatamente `playsound==1.2.2` |
| Fala continua tocando após interrupção | Processo órfão preso | O hook `PreInvocation` limpa com `taskkill`, ou execute `tts_interrupt.py` manualmente |
