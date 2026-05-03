# Codex JSONL Extraction Patterns

The primary foreground extraction pattern is in SKILL.md. This file covers background execution, alternative modes, and fallbacks.

## Parser Tool

`cx-parse` is a zero-dependency Python 3 CLI at `$HOME/.claude/skills/codex-cli/scripts/cx-parse`. Exit codes: 0=ok, 2=not-found/invalid, 3=parse-error. UUID validation is built in.

```bash
CX="$HOME/.claude/skills/codex-cli/scripts/cx-parse"
```

## Background Extraction

```bash
CX="$HOME/.claude/skills/codex-cli/scripts/cx-parse"
JSONL="$(mktemp -t codex.XXXXXX.jsonl)"
ERR="$(mktemp -t codex.XXXXXX.err)"

codex exec --json [OPTIONS] "$PROMPT" >"$JSONL" 2>"$ERR" &
CODEX_PID=$!

# Poll for session ID (reads only the first line, not the whole file)
SESSION_ID=""
for i in $(seq 1 120); do
  [ -s "$JSONL" ] && SESSION_ID="$($CX session-id "$JSONL" 2>/dev/null)" && [ -n "$SESSION_ID" ] && break
  sleep 0.5
done

# When checking status: NEVER read the full JSONL. Use targeted extraction:
#   tail -n 5 "$ERR"              # last few stderr lines for diagnostics
#   wc -l < "$JSONL"              # event count as progress indicator

# After completion, extract response:
RESPONSE="$($CX response "$JSONL" --max-chars 12288)"
```

**IMPORTANT:** Redirect stdout and stderr to **separate files** (`>"$JSONL" 2>"$ERR"`). Never mix them with `2>&1` — mixing corrupts the JSONL stream and inflates the output with thinking tokens.

## JSON Extraction (session_id + text + metadata in one call)

```bash
$CX extract "$JSONL" --max-chars 12288
# stdout: {"session_id":"...","text":"...","text_truncated":false,"metadata":{"source":"..."}}
```

## Pipe Mode with `--tee`

Save stdin copy then parse:

```bash
codex exec --json [OPTIONS] "$PROMPT" 2>/dev/null | $CX extract --tee "$JSONL" --max-chars 12288
```

## Fallback: Inline Python

If `cx-parse` is unavailable:

```bash
SESSION_ID="$(python3 -c "
import json,sys
with open(sys.argv[1],'r',errors='replace') as f: obj=json.loads(f.readline())
print(obj.get('thread_id',''))
" "$JSONL")"

RESPONSE="$(python3 -c "
import json,sys; path,cap=sys.argv[1],int(sys.argv[2]); last=''
with open(path,'r',errors='replace') as f:
    for line in f:
        try: obj=json.loads(line)
        except: continue
        if obj.get('type')=='item.completed':
            item=obj.get('item') or {}
            if item.get('type')=='agent_message': last=item.get('text') or ''
if len(last)>cap: print(last[:cap]+f'\n[truncated {len(last)-cap} chars]')
else: print(last)
" "$JSONL" 12288)"
```

## Fallback: stderr grep

If `--json` is unavailable, extract from stderr:

```bash
codex exec [OPTIONS] "$PROMPT" 2>/tmp/codex_stderr.txt
SESSION_ID=$(grep "session id:" /tmp/codex_stderr.txt | awk '{print $NF}')
```

> **WARNING:** The old `find ~/.codex/sessions -newer ... | jq` method is **deprecated** — it is unreliable and has produced garbage values in production. Do not use it.
