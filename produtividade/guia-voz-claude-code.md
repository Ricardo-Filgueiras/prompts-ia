# Guia: Falar com o Claude Code e ouvir as respostas em voz alta

Este guia replica uma configuração de voz para o Claude Code:

- **Entrada por voz (STT):** você fala segurando uma tecla (hold-to-talk) e, ao soltar, a mensagem é **enviada automaticamente** — sem precisar apertar Enter.
- **Saída por voz (TTS):** ao final de cada resposta, o Claude **fala o texto em voz alta**, usando vozes neurais gratuitas da Microsoft (via `edge-tts`). A fala **começa em cerca de 1–2 segundos**, mesmo em respostas longas, e caminhos de arquivo, links e tabelas são adaptados para serem ouvidos.
- **Interrupção ("levantar a mão para falar"):** se o Claude ainda estiver falando quando você enviar uma nova mensagem, a fala é cortada na hora, simulando uma conversa real. Funciona por janela: com duas sessões do Claude Code abertas, uma não corta a fala da outra.
- **`parar`:** enviar só a palavra `parar` corta a fala **sem mandar nada ao Claude** (não gasta crédito).
- **`reler`:** enviar só a palavra `reler` fala de novo a última resposta, também **sem mandar nada ao Claude** e mesmo com o TTS desligado.
- Um comando `/tts on` / `/tts off` para ligar e desligar a fala quando quiser.
- **Log de diagnóstico** em `~/.claude/tts_log.txt`, para descobrir por que a fala não saiu.

Testado no Windows. Notas para Mac/Linux estão no final.

---

## Onde usar (e onde não usar)

Este mecanismo existe para os lugares em que **não há opção nativa de ouvir a resposta do agente**:

- **Claude Code no terminal (CLI)**
- **Claude Code dentro do VS Code** (extensão oficial)

Os dois leem o mesmo `~/.claude/settings.json`, então uma única instalação atende aos dois.

**Não instale** onde já existe leitura em voz alta nativa (ex.: app do Claude no celular, claude.ai) — lá isso só duplicaria a fala.

Diferença entre CLI e VS Code:

| Recurso | CLI | VS Code |
|---|---|---|
| Fala ao final da resposta (hook `Stop`) | Sim | Sim |
| Interromper ao enviar mensagem, `parar`, `reler` | Sim | Sim |
| Entrada por voz segurando tecla (`voice` no settings) | Sim | Não — use o ditado do Windows (`Win + H`) na caixa de mensagem |

---

## Pré-requisitos

- Claude Code instalado e funcionando.
- Python 3.9+ instalado e acessível no terminal (teste com `python --version`).
- Internet (o `edge-tts` usa um serviço online da Microsoft).

---

## Passo 1 — Instalar as dependências Python

```bash
pip install --user --upgrade edge-tts "playsound==1.2.2"
```

- `edge-tts`: gera áudio a partir de texto usando o mesmo serviço gratuito do "Ler em voz alta" do navegador Edge. Não precisa de conta, chave de API nem assinatura.
- `playsound`: reproduz o arquivo de áudio gerado. Use a versão 1.2.2 — as mais novas têm problemas no Windows.
- Use sempre `--upgrade` no `edge-tts`: a Microsoft muda de tempos em tempos o que o serviço aceita, e versões antigas passam a receber **erro 403** (ver "Solução de problemas").
- O aviso `scripts ... not on PATH` que o pip pode mostrar é inofensivo — os scripts deste guia não dependem disso.

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
Com --reler, fala mesmo desligado (usado pelo comando "reler" do tts_interrupt.py).
Nunca trava o Claude Code: erros vao para ~/.claude/tts_log.txt em vez da tela.

A resposta e dividida em partes: a primeira comeca a tocar enquanto as seguintes
ainda estao sendo geradas, entao a fala comeca quase na hora.
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

VOICE = "pt-BR-ThalitaMultilingualNeural"
RATE = "+0%"  # velocidade: "+20%" mais rapido, "-10%" mais devagar
MAX_CHARS = 3000
FIRST_CHUNK_CHARS = 200  # primeira parte curta: a fala comeca mais rapido
CHUNK_CHARS = 600
CLAUDE_DIR = Path.home() / ".claude"
TOGGLE_FILE = CLAUDE_DIR / "tts_enabled"
PID_DIR = CLAUDE_DIR / "tts_pids"
LOG_FILE = CLAUDE_DIR / "tts_log.txt"

# Caminhos de arquivo viram so o nome do arquivo (ninguem quer ouvir "C, dois pontos, barra, Users...")
WIN_PATH = re.compile(r"[A-Za-z]:[\\/](?:[^\\/\n`\"<>|*?]*[\\/])*[^\\/\s`\"<>|*?,;:)]*")
UNIX_PATH = re.compile(r"(?<![\w:/])(?:~|\.{1,2})?/(?:[\w.\-]+/)+[\w.\-]*")
TABLE_SEPARATOR = re.compile(r"^\s*\|?[\s:|-]*-{3,}[\s:|-]*$")


def session_key(hook_input: dict) -> str:
    # Cada sessao/janela do Claude Code tem seu proprio PID e seus proprios audios
    sid = re.sub(r"[^A-Za-z0-9_-]", "", str(hook_input.get("session_id", "")))
    return sid or "default"


def pid_file(key: str) -> Path:
    return PID_DIR / key


def audio_glob(key: str) -> str:
    return os.path.join(tempfile.gettempdir(), f"claude_tts_{key}_*.mp3")


def is_enabled() -> bool:
    try:
        return TOGGLE_FILE.read_text(encoding="utf-8").strip().lower() == "on"
    except FileNotFoundError:
        return False


def log(msg: str) -> None:
    # Registro de diagnostico; recomeça do zero quando passa de ~100 KB
    try:
        if LOG_FILE.exists() and LOG_FILE.stat().st_size > 100_000:
            LOG_FILE.unlink()
        with LOG_FILE.open("a", encoding="utf-8") as f:
            f.write(f"{datetime.now():%Y-%m-%d %H:%M:%S} {msg}\n")
    except OSError:
        pass


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


def _path_name(match: re.Match) -> str:
    parts = [p for p in re.split(r"[\\/]", match.group(0)) if p]
    return parts[-1] if parts else " "


def clean_for_speech(text: str) -> str:
    text = re.sub(r"```.*?```", "\nbloco de codigo omitido.\n", text, flags=re.DOTALL)
    text = re.sub(r"!\[[^\]]*\]\([^)]*\)", " ", text)
    text = re.sub(r"\[([^\]]*)\]\([^)]*\)", r"\1", text)
    text = re.sub(r"https?://\S+", "link", text)
    text = WIN_PATH.sub(_path_name, text)
    text = UNIX_PATH.sub(_path_name, text)
    text = text.replace("`", "")

    lines = []
    for line in text.splitlines():
        if TABLE_SEPARATOR.match(line):
            continue
        if "|" in line:
            # Linha de tabela: le as celulas separadas por virgula
            cells = [c.strip() for c in line.strip().strip("|").split("|")]
            line = ", ".join(c for c in cells if c)
        line = re.sub(r"^\s*[-•]\s+", "", line)
        line = re.sub(r"[*_#>]+", " ", line)
        line = re.sub(r"\s+", " ", line).strip()
        if not line:
            continue
        if line[-1] not in ".!?:;,":
            line += "."  # pausa no fim de titulos e itens de lista
        lines.append(line)

    text = " ".join(lines)
    if len(text) > MAX_CHARS:
        text = text[:MAX_CHARS].rsplit(" ", 1)[0] + "..."
    return text


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

    # O servico da Microsoft costuma falhar na primeira conexao; tenta ate 3 vezes
    for attempt in range(1, 4):
        try:
            asyncio.run(edge_tts.Communicate(text, VOICE, rate=RATE).save(out_path))
            return
        except Exception as e:
            log(f"sintese falhou (tentativa {attempt}): {type(e).__name__}: {e}")
            if attempt == 3:
                raise
            time.sleep(1)


def speak(chunks: list, key: str) -> None:
    # Uma thread gera os audios em sequencia; a principal toca cada um assim que fica pronto
    prefix = os.path.join(tempfile.gettempdir(), f"claude_tts_{key}_{os.getpid()}_")
    ready = queue.Queue()

    def produce() -> None:
        try:
            for i, chunk in enumerate(chunks):
                path = f"{prefix}{i}.mp3"
                synthesize(chunk, path)
                ready.put(path)
        except Exception:
            log("ERRO ao gerar audio:\n" + traceback.format_exc())
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
    try:
        # --reler: chamado pelo comando "reler", fala mesmo com o TTS desligado
        if "--reler" not in sys.argv and not is_enabled():
            log("ignorado: TTS desligado")
            return
        hook_input = json.load(sys.stdin)
        key = session_key(hook_input)
        transcript_path = hook_input.get("transcript_path")
        if not transcript_path or not Path(transcript_path).exists():
            log(f"ignorado: transcript nao encontrado ({transcript_path})")
            return
        text = clean_for_speech(last_assistant_text(transcript_path))
        if not text:
            log("ignorado: ultima resposta sem texto")
            return

        # PID gravado ANTES de gerar o audio: uma nova mensagem ja consegue interromper
        # mesmo enquanto o audio ainda esta sendo gerado
        PID_DIR.mkdir(parents=True, exist_ok=True)
        my_pid_file = pid_file(key)
        my_pid_file.write_text(str(os.getpid()), encoding="utf-8")
        try:
            chunks = split_chunks(text)
            log(f"falando {len(text)} caracteres em {len(chunks)} parte(s) [sessao {key[:8]}]")
            speak(chunks, key)
        finally:
            try:
                if my_pid_file.read_text(encoding="utf-8").strip() == str(os.getpid()):
                    my_pid_file.unlink()
            except OSError:
                pass
    except Exception:
        log("ERRO:\n" + traceback.format_exc())


if __name__ == "__main__":
    main()
```

Pontos importantes deste script:
- **Nunca mostra erro na tela nem trava o Claude Code** — mas registra tudo em `~/.claude/tts_log.txt` (fala, motivo de ter ignorado, ou o erro completo). Sem esse log, uma falha fica invisível: você só percebe que "não falou".
- **Fala em partes (começa rápido):** o texto é dividido em partes por frase — a primeira curta (~200 caracteres), as demais de até ~600. Uma thread gera os áudios em sequência enquanto a principal já toca o que está pronto. Sem isso, uma resposta de 3000 caracteres levava vários segundos de silêncio antes de começar.
- **Texto adaptado para ouvir** (`clean_for_speech`):
  - caminhos (`C:\Users\...\tts_log.txt`, `~/.claude/settings.json`) viram só o nome do arquivo;
  - links (`https://...`) viram a palavra "link";
  - tabelas: a linha `|---|---|` é pulada e as células são lidas separadas por vírgula;
  - blocos de código viram "bloco de código omitido";
  - títulos e itens de lista ganham um ponto final, para a voz fazer pausa entre eles.
- **PID gravado antes de gerar o áudio:** assim uma nova mensagem (ou `parar`) consegue interromper mesmo enquanto o áudio ainda está sendo gerado. Antes, nesse intervalo a interrupção não encontrava o processo e a resposta antiga tocava por cima da conversa nova.
- **Um PID por sessão** (`~/.claude/tts_pids/<session_id>`), e os áudios temporários levam o `session_id` no nome — por isso uma janela não interrompe a outra.
- **Tenta até 3 vezes** gerar cada parte do áudio. Na prática, o serviço da Microsoft frequentemente responde `NoAudioReceived` na primeira conexão e funciona na segunda.
- **`--reler`** faz o script ignorar o `/tts off`, para o comando `reler` funcionar sempre.

---

## Passo 2.1 — Criar o script de interrupção, `parar` e `reler`

O Claude Code não tem um evento de hook para "tecla pressionada" ou "gravação iniciada" — só existe um hook para **mensagem enviada** (`UserPromptSubmit`). Como o modo de voz usa `autoSubmit: true`, soltar a tecla equivale a enviar a mensagem quase instantaneamente. Por isso, a interrupção funciona assim: **no momento em que você envia uma nova mensagem, qualquer fala em andamento é cortada** — não é possível interromper no instante exato em que você começa a segurar a tecla, pois esse evento não é exposto pelo Claude Code.

O mesmo hook trata duas palavras especiais. Quando a mensagem é **só** uma delas, o hook responde `{"decision": "block"}`: a mensagem **não é enviada ao Claude** (não gasta crédito nem gera resposta) e o Claude Code mostra o motivo na tela.

| Mensagem | Efeito |
|---|---|
| `parar` (ou `/parar`) | Corta a fala e não faz mais nada |
| `reler` (ou `/reler`) | Corta a fala atual (se houver) e fala de novo a última resposta |
| Qualquer outra | Corta a fala e a mensagem segue normalmente para o Claude |

Salve o arquivo abaixo como `~/.claude/scripts/tts_interrupt.py`:

```python
"""UserPromptSubmit hook: interrompe a fala do TTS em andamento quando o usuario envia uma nova mensagem.

Simula "levantar a mao para falar": mata o processo que esta tocando o audio desta sessao
(se houver) e limpa arquivos de audio temporarios que possam ter ficado orfaos.
Outras janelas/sessoes do Claude Code nao sao afetadas.

Palavras especiais (a mensagem e bloqueada e nao vai para o Claude, entao nao gasta credito):
- "reler": fala novamente a ultima resposta.
- "parar": so interrompe a fala.
"""
import glob
import json
import os
import subprocess
import sys
import tempfile
import time
from pathlib import Path

sys.path.insert(0, str(Path(__file__).parent))
from tts_speak import audio_glob, pid_file, session_key  # mesmos caminhos do script de fala

SPEAK_SCRIPT = Path(__file__).with_name("tts_speak.py")
REPEAT_WORDS = {"reler", "/reler"}
STOP_WORDS = {"parar", "/parar"}
ORPHAN_MAX_AGE = 3600  # audios de qualquer sessao com mais de 1 hora sao lixo


def kill_current_speech(key: str) -> None:
    path = pid_file(key)
    try:
        pid = int(path.read_text(encoding="utf-8").strip())
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
        path.unlink()
    except OSError:
        pass


def cleanup_orphan_audio(key: str) -> None:
    for path in glob.glob(audio_glob(key)):
        try:
            os.remove(path)
        except OSError:
            pass
    old_limit = time.time() - ORPHAN_MAX_AGE
    for path in glob.glob(os.path.join(tempfile.gettempdir(), "claude_tts_*.mp3")):
        try:
            if os.path.getmtime(path) < old_limit:
                os.remove(path)
        except OSError:
            pass


def repeat_last_answer(hook_input: dict) -> None:
    # Processo separado e desanexado: continua falando depois que este hook termina
    proc = subprocess.Popen(
        [sys.executable, str(SPEAK_SCRIPT), "--reler"],
        stdin=subprocess.PIPE,
        stdout=subprocess.DEVNULL,
        stderr=subprocess.DEVNULL,
        creationflags=subprocess.DETACHED_PROCESS | subprocess.CREATE_NEW_PROCESS_GROUP,
    )
    proc.stdin.write(json.dumps(hook_input).encode("utf-8"))
    proc.stdin.close()


def main() -> None:
    try:
        hook_input = json.load(sys.stdin)
        key = session_key(hook_input)
        kill_current_speech(key)
        cleanup_orphan_audio(key)
        prompt = hook_input.get("prompt", "").strip().lower()
        if prompt in REPEAT_WORDS:
            repeat_last_answer(hook_input)
            print(json.dumps({"decision": "block", "reason": "Relendo a ultima resposta..."}))
        elif prompt in STOP_WORDS:
            print(json.dumps({"decision": "block", "reason": "Fala interrompida."}))
    except Exception:
        return


if __name__ == "__main__":
    main()
```

Por que o `reler` abre um processo separado: o hook `UserPromptSubmit` é síncrono e tem `timeout` curto. Se ele mesmo tocasse o áudio, seria morto pelo timeout no meio da fala. Por isso ele só dispara o `tts_speak.py --reler` desanexado e termina na hora.

Outros detalhes:
- Este script **importa do `tts_speak.py`** as funções que definem onde ficam o PID e os áudios de cada sessão (`session_key`, `pid_file`, `audio_glob`). Os dois arquivos precisam estar **na mesma pasta**.
- Ele só mata a fala **da própria sessão**. Áudios temporários de qualquer sessão com mais de 1 hora também são apagados, para não acumular lixo se uma janela for fechada no meio de uma fala.

> **Nota Mac/Linux:** `taskkill /F /PID` e `DETACHED_PROCESS` são específicos do Windows. Veja "Notas para Mac/Linux" no final.

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
- `hooks.UserPromptSubmit`: dispara o script de interrupção sempre que uma nova mensagem é enviada — corta a fala em andamento e trata `parar` / `reler`. **Precisa ser síncrono (sem `"async"`)**: só assim o bloqueio (`"decision": "block"`) impede a mensagem de chegar ao Claude.
- `voice.mode: "hold"`: modo "segurar para falar" (só no CLI).
- `voice.autoSubmit: true`: envia a mensagem automaticamente ao soltar a tecla (sem precisar de Enter).

---

## Passo 4 — Criar o arquivo de ativação (toggle)

```bash
mkdir -p ~/.claude
printf 'on' > ~/.claude/tts_enabled
```

O script só fala sozinho (ao final das respostas) se esse arquivo contiver exatamente `on`. O `reler` funciona mesmo com `off`.

---

## Passo 5 — Criar o comando `/tts on` / `/tts off`

Salve o arquivo abaixo como `~/.claude/commands/tts.md`:

````markdown
---
description: Ativa ou desativa a leitura em voz alta (TTS) das respostas do Claude
argument-hint: on|off
---

O usuário passou o argumento: "$1"

Se o argumento for exatamente "on" ou "off" (minúsculas), execute este comando via Bash, sem pedir confirmação:

```bash
mkdir -p "$HOME/.claude" && printf '%s' "$1" > "$HOME/.claude/tts_enabled"
```

Se o argumento não for "on" nem "off" (ex: vazio, ou outro valor), NÃO execute nada — pergunte ao usuário se ele quer ativar ou desativar o TTS.

Depois de executar, confirme ao usuário em português se o TTS foi ativado ou desativado.
````

Observação: diferente de `parar` e `reler`, o `/tts` **passa pelo Claude** (é ele quem executa o comando), então consome um pouco de crédito.

---

## Passo 6 — Reiniciar o Claude Code

Hooks são carregados na inicialização da sessão. Feche e abra uma nova conversa (ou use `/hooks` para forçar o recarregamento) para a configuração entrar em vigor. No VS Code, se não pegar, recarregue a janela (`Ctrl + Shift + P` → "Developer: Reload Window").

---

## Testando

Primeiro, teste os scripts fora do Claude Code (pega erros de instalação antes de tudo):

```bash
python -c "import asyncio, edge_tts; asyncio.run(edge_tts.Communicate('Teste de voz.', 'pt-BR-ThalitaMultilingualNeural').save('teste.mp3')); print('ok')"
```

Depois, dentro do Claude Code:

1. Mande uma mensagem qualquer — ao final da resposta, ela deve ser lida em voz alta.
2. Enquanto o Claude ainda estiver falando, mande `parar` — a fala deve cortar e aparecer "Fala interrompida.", sem resposta do Claude.
3. Mande `reler` — a última resposta deve ser lida de novo, com o aviso "Relendo a ultima resposta...".
4. Enquanto estiver falando, mande uma mensagem normal — a fala corta e o Claude responde normalmente.
5. (Só CLI) Segure a tecla de voz, diga algo, solte — a mensagem deve ser enviada sozinha, sem Enter.
6. `/tts off` desliga a fala automática; `/tts on` religa.
7. Abra `~/.claude/tts_log.txt` — deve haver uma linha `falando N caracteres em K parte(s) [sessao ...]` para cada resposta lida.
8. (Opcional) Com duas janelas do Claude Code abertas, mande `parar` em uma enquanto a outra fala — a outra deve continuar falando.

---

## Solução de problemas

**Sempre comece pelo log `~/.claude/tts_log.txt`.**

| O que aparece no log | Causa | Solução |
|---|---|---|
| Nada (arquivo vazio ou inexistente) | O hook nem está rodando | Confira caminhos no `settings.json`, rode `/hooks`, reinicie a sessão / recarregue o VS Code |
| `ignorado: TTS desligado` | Toggle em `off` | `/tts on` |
| `WSServerHandshakeError: 403` | `edge-tts` desatualizado; a Microsoft passou a recusar a versão instalada | `python -m pip install --user --upgrade edge-tts` |
| `sintese falhou (tentativa 1)` sem `ERRO` depois | Falha momentânea do serviço, já resolvida pela nova tentativa | Nada a fazer |
| `NoAudioReceived` nas 3 tentativas / `ERRO ao gerar audio` | Serviço fora do ar, sem internet, ou voz/`RATE` inválidos | Verifique a internet; confira `VOICE` e o formato de `RATE` (ex.: `"+20%"`) |
| `falando ...` mas sem som | Problema de áudio do sistema / `playsound` | Confira volume e saída de som; reinstale `playsound==1.2.2` |
| `ERRO:` com outro traceback | Erro inesperado | O traceback mostra a linha exata |

Outros sintomas:
- **`/reler` dá "comando desconhecido":** use a palavra sem a barra (`reler`).
- **A fala de uma resposta antiga continua tocando:** mande `parar`.
- **`ModuleNotFoundError: tts_speak` ao enviar mensagem:** o `tts_interrupt.py` não está na mesma pasta do `tts_speak.py`.
- **A voz lê um caminho ou símbolo de forma estranha:** ajuste `clean_for_speech` no `tts_speak.py` (os padrões de caminho estão em `WIN_PATH` e `UNIX_PATH`).

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

**Velocidade da voz** — constante `RATE` no `tts_speak.py`. Padrão `"+0%"`; use `"+20%"` para mais rápido ou `"-10%"` para mais devagar. Precisa ter sinal e `%`.

**Tamanho máximo do texto falado** — constante `MAX_CHARS` no script (padrão: 3000 caracteres, ~500-600 palavras; respostas maiores que isso são cortadas com "..."). Como a fala começa logo, aumentar não deixa o início mais lento — mas respostas longas passam a falar por vários minutos (use `parar` para cortar).

**Tamanho das partes** — `FIRST_CHUNK_CHARS` (padrão 200) define a primeira parte: menor = começa a falar mais rápido. `CHUNK_CHARS` (padrão 600) define as demais. Partes muito pequenas podem gerar pausas curtas entre elas.

**Trocar as palavras de `parar` / `reler`** — edite `STOP_WORDS` e `REPEAT_WORDS` no `tts_interrupt.py` (sempre em minúsculas). Escolha palavras que você nunca mandaria sozinhas como mensagem de verdade, porque elas nunca chegam ao Claude.

---

## Notas para Mac/Linux

- No `command` do hook, use o caminho do `python3` (`which python3`), não `python`.
- No `tts_interrupt.py`:
  - troque o `taskkill` por `os.kill(pid, signal.SIGKILL)` (com `import signal`);
  - no `subprocess.Popen` do `repeat_last_answer`, troque `creationflags=...` por `start_new_session=True`.
- A biblioteca `playsound` não é mais mantida. Ela funciona com a versão 1.2.2, mas se parar de funcionar numa versão nova do Python, a troca fica isolada no `speak()` do `tts_speak.py`: basta tocar o arquivo de outro jeito (ex.: `subprocess.run(["ffplay", "-nodisp", "-autoexit", path])` com o FFmpeg instalado). Isso vale para todos os sistemas.
- `playsound` no Linux pode exigir dependências de sistema (ex: GStreamer) para tocar `.mp3`. Se der erro, troque a chamada de `playsound` no script por uma chamada via `subprocess` para `afplay` (Mac) ou `mpg123`/`ffplay` (Linux).
- Os caminhos com `~` (`Path.home()`) funcionam igual em todos os sistemas — não precisa adaptar essa parte do script.
