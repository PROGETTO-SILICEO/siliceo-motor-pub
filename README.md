# siliceo-motor

**Un motore di inferenza LLM sovrano in Rust nativo** — dal formato GGUF al server OpenAI-compatible, senza wrapper, senza binding C++, senza dipendenze HTTP/JSON.

```
curl → s-server → JSON (in casa) → tokenizer BPE nativa → forward f32 → sampler → risposta OpenAI-compatible
```

## Perché

I motori dominanti sono ottimi, ma usarli significa delegare: kernel C++, graph capture,
licenze AGPL, stack Python. *Sovrano* vuol dire possedere il codice — ogni livello di
questo motore è scritto da noi e verificabile:

- **s-gguf**: parser GGUF v3 completo con dequantizzazione K-quants
- **s-tokenizer**: BPE byte-level che legge vocabolario e merges direttamente dal GGUF
- **s-models**: architetture (qwen2/llama) con forward CPU naive f32
- **s-server**: HTTP/1.1 su `std::net`, zero dipendenze esterne

## Stato: parity verificata contro llama.cpp

Non è un prototipo "sembra funzionare": ogni fase è chiusa con verifica oggettiva.

| Fase | Componente | Verifica |
|---|---|---|
| F0 | s-gguf | **291 tensori al byte** identici a `llama-gguf` su Qwen2.5-0.5B Q4_K_M |
| F1 | s-tokenizer | IDs **identici** a `llama-tokenize` |
| F1 | forward qwen2 | generazione greedy **token-identica** a llama.cpp (12/12 token); top-6 logprob combaciante |
| F2 | sampling + server | endpoint OpenAI-compatible via curl; greedy/top-k/top-p seedabili; stop conditions |

### Riprodurre la parity

```bash
# tokenizer parity
S_TOKENIZER_QWEN05B=model.gguf cargo test -p s-tokenizer --test parity

# logits parity (top-6 identico a llama.cpp)
S_MODELS_QWEN05B=model.gguf cargo test -p s-models --release --test parity_logits -- --ignored

# GGUF parser parity
S_GGUF_QWEN05B=model.gguf cargo test -p s-gguf --test parity_llamacpp
```

## Quick start

```bash
cargo build --release -p s-server

# serve un modello GGUF (testato con Qwen2.5-0.5B-Instruct Q4_K_M)
./target/release/s-server ./model.gguf --port 8096

curl http://127.0.0.1:8096/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"messages":[{"role":"user","content":"Ciao!"}],"max_tokens":50}'
```

### Parametri supportati

`temperature`, `top_p`, `top_k`, `max_tokens`, `seed`, `stop` (stringa o array).
Greedy con `temperature: 0` — deterministico al token.

## Configurazione dinamica

Un solo binario, comandabile a runtime senza ricompilare né riavviare:

```bash
# motor.json (caricato automaticamente se presente nella directory corrente)
{
  "server":   { "port": 8096 },
  "model":    { "path": "model.gguf", "template": "auto" },
  "generate": { "temperature": 0.0, "max_tokens": 256 }
}

# cambiare i default A SERVER ACCESO
curl -X POST http://127.0.0.1:8096/v1/config -d '{"temperature": 0.7}'

# sostituire il modello senza spegnere nulla (le richieste in corso
# finiscono col modello su cui sono iniziate)
curl -X POST http://127.0.0.1:8096/v1/model -d '{"path": "altro.gguf"}'

# leggere la configurazione effettiva
curl http://127.0.0.1:8096/v1/config
```

I default si compongono a strati (il più forte vince): valori di fabbrica →
`/etc/siliceo-motor/motor.json` → `./motor.json` → flag CLI → parametri della
singola richiesta → patch via `/v1/config`.

Il template di chat si rileva dal GGUF (`tokenizer.chat_template`): ChatML
(Qwen2.5, SmolLM2) e Llama3 sono supportati; override manuale con
`"template": "chatml" | "llama3"`.

## Onestà sulle performance

La generazione usa **KV cache incrementale**: prefill una volta sola, poi un
token per passo. Su una CPU desktop x86 il modello 0.5B gira a ~0.7 tok/s —
output identico al byte rispetto a llama.cpp, verificato dai test. Il backend
CUDA (F3) è il prossimo salto — vedi [ROADMAP.md](ROADMAP.md).

## Licenza

MIT OR Apache-2.0 (a scelta dell'utente), come l'ecosistema Rust.
