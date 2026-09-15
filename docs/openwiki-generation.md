# Local OpenWiki generation

Latest local `openwiki/` init completed on 2026-09-14.

| Field | Value |
| --- | --- |
| Tool | OpenWiki v0.5.1 |
| Command | `init` |
| Provider | OpenAI-compatible (`https://api.minimax.cn/v1`) |
| Model | MiniMax-M3 |
| Tracing | LangSmith enabled, run complete |
| Output | 32 pages under `openwiki/` |
| Duration | 140m 56s |
| Work | 1189 reads · 966 searches |
| Git HEAD | `f8c19bd766f9738c00a2f074970cfdaa36ee22a7` |
| Language | en |

Machine metadata: `openwiki/.last-update.json`. Page index: `openwiki/index.md`. Do not put API keys in the repo.

## Credentials

Config lives in `~/.openwiki/.env`, not this repository.

- `OPENAI_COMPATIBLE_API_KEY` + `OPENAI_COMPATIBLE_BASE_URL` are the **model** credentials. The MiniMax key and base URL must be the same region (`api.minimax.cn` for China, `api.minimax.io` for international).
- `LANGSMITH_API_KEY` / `OPENWIKI_LANGSMITH_API_KEY` are only for tracing. They are not a model key.

## Known MiniMax failures

- `402` / `1008` `insufficient balance` — no usable MiniMax balance. Top up; do not retry unchanged.
- `401` / `1004` `login fail` / missing `Authorization` — key missing, invalid, or region mismatch after `openwiki --init`. Re-paste the MiniMax key. Do not put the LangSmith `lsv2_…` key in `OPENAI_COMPATIBLE_API_KEY`.

Do not hand-edit generated `openwiki/` pages unless explicitly asked; change source code/docs and regenerate.
