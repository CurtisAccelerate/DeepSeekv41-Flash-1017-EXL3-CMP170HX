# Frozen synthetic likelihood fixture

`teacher-data.json.gz` decompresses to the exact 24-case fixture used by both
new likelihood screens. `MANIFEST.json` pins the original bytes; `target-spans.json`
records expected text and critical-field token spans. The two screens each
score 138 gold tokens per pass across approximately 0.4K/7.3K histories.

```python
import gzip, hashlib, json
from pathlib import Path
root = Path("benchmarks/2026-11-01/fixtures")
data = gzip.decompress((root / "teacher-data.json.gz").read_bytes())
assert hashlib.sha256(data).hexdigest() == json.loads((root / "MANIFEST.json").read_text())["uncompressed_sha256"]
cases = json.loads(data)
```

The fixture includes exact prompt and target token IDs; validate the prompt-ID
hash after applying the pinned template/tokenizer before comparing scores.
The source package's teacher probe requires a qualified private model load.
CPU parsing this data does not run a model. Teacher forcing bypasses drafting;
these data do not measure whole-agent success or standard-corpus perplexity.
