# ANCHOR

CS3346A group project. Tests whether LLMs reason about prediction-market
prices or just copy them. Each model forecasts resolved binary markets
under three conditions:

- **BLIND** — no price shown
- **INFORMED** — the real market price shown
- **DECOY** — a random price shown (≥20 points from the real one)

Models (via local Ollama): `llama3.2:3b`, `llama3.1:8b`, `qwen2.5:3b`,
`qwen2.5:7b`. Temperature 0.7, 5 runs per market × condition × model.

## Repo layout

```
config/          shared config (Kamyar)
data/            real markets.csv + splits (Evan + Aryan) — not in repo
fixtures/        small fake dataset matching the real schema
src/anchor/      pipeline (this part — Nima)
scripts/         setup helpers
tests/           unit tests
docs/            method_notes.md for the report (Talha)
results/         JSONL output (gitignored)
```

Module responsibilities live in `src/anchor/`: `config.py`, `prompts.py`,
`parser.py`, `decoy.py` (placeholder for Kamyar), `markets.py`,
`model_client.py`, `pipeline.py`, `cli.py`.

## First-time setup

1. Install Ollama and pull the four models — see **[SETUP.md](SETUP.md)**.
2. Create the Python environment:

   ```bash
   python -m venv .venv
   # Windows:
   .venv\Scripts\activate
   # macOS/Linux:
   source .venv/bin/activate
   pip install -r requirements.txt
   ```

3. Verify Ollama + model digests (share the output in the group chat so
   both GPUs match):

   ```bash
   python scripts/check_ollama.py
   ```

## Commands

Run unit tests:

```bash
python -m pytest tests/ -q
```

Dry-run the full pipeline on the fixture with a fake model (no Ollama
required — good for CI and offline checks):

```bash
PYTHONPATH=src python -m anchor.cli parser-test --n 10 --fixture --dry-run
```

Real parser test on 10 dev markets × 3 conditions × 4 models through
Ollama:

```bash
PYTHONPATH=src python -m anchor.cli parser-test --n 10 --fixture
# drop --fixture once data/markets.csv is in place
```

Runs are **resumable**: re-running the same command skips any
`(market_id, condition, run)` already logged in `results/`, so an
overnight run survives a crash.

## Output

One JSONL file per model under `results/<run_name>/<model>.jsonl`.
Each line contains: `model`, `model_digest`, `market_id`, `condition`,
`run`, `seed`, `shown_price`, `prompt`, `raw_response`,
`parsed_probability`, `retry_count`, `final_status`, `elapsed_s`,
`timestamp`.

## Group rules enforced here

- Every call is a fresh request, no chat history.
- Informed and decoy prompts use **identical wording** for the price
  line. The model is never told whether the price is real or random.
- The `outcome` column is never included in any prompt.
- Parse failure → one retry with a bumped seed. Over 10% parse failures
  on the dev set means the model is replaced or dropped.

## Ownership

- **Nima** — model pipeline (this directory tree, minus the files below)
- **Evan** — `data/markets.csv`
- **Aryan** — evaluation metrics, dev/test split
- **Kamyar** — `config/config.yaml`, decoy generator, repo setup
- **Talha** — report, paraphrased prompt set
