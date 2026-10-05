# sarthi

Sarthi is a SEBI Track B build that helps retail investors protect accounts, fix nominations, transmit shares after a death, and file IEPF claims without getting rejected on paperwork.

Four repos work as one system. This repo is the map. Code lives in the other three.

```
+------------------------+      +------------------------+
| extension (Saathi)     |      | portal (Viraasat)      |
| Chrome MV3 side panel  |      | Next.js 14 on port 3000|
| voice and text intake  |      | wealth map and docs    |
| DOM autofill for SCORES|      | nominee audit and PDFs |
+-----------+------------+      +-----------+------------+
            |                               |
            | HTTP to 127.0.0.1:8787        | HTTP to 127.0.0.1:8787
            +---------------+----------------+
                            |
                +-----------v------------+
                | sarthi-engine          |
                | FastAPI on port 8787   |
                | speech, LLM, docs,     |
                | AA mock, shared data   |
                +-----------+------------+
                            |
            +---------------+---------------+
            |               |               |
      Sarvam API    Gemini API      mock AA data
      speech only   chat and vision  Setu and Finvu shape
```

## Repos

- `sarthi-contracts` holds JSON Schemas, TypeScript types, and `config.json`. Every shared field name is defined there once.
- `sarthi-engine` holds the FastAPI backend on port 8787. It owns the API keys and serves speech, LLM, docs, AA, and shared data.
- `sarthi-portal` holds the Viraasat Next.js app on port 3000. It calls the engine and holds no keys.
- `extension` (teammate repo `pranshoe/saathi`) holds the Saathi grievance agent. It calls the engine and holds no keys.

## Ports

- Engine - FastAPI - 127.0.0.1:8787
- Portal - Next.js - localhost:3000
- Mock SCORES and IEPF pages - static HTML on 8788 (teammate owned)

## Contract flow

`sarthi-contracts` defines `apiEnvelope`, `holding`, `aaConsent`, `aaFetchResponse`, `grievanceState`, and `affidavit`. The engine validates against these with Pydantic models plus `pytest` and `jsonschema`. The portal validates fixtures with `ajv` through `npm run test:contracts`. A field change starts in `sarthi-contracts`, then the engine model and the TS types get updated. Builds fail loudly when a consumer lags.

Most responses use the envelope with `ok`, `data`, `meta`, and `error`. The extension passthrough routes (`/stt`, `/tts`, `/gemini/*`, `/llm/chat`) return raw vendor shapes because the extension already parses them.

## Data flows

- Grievance - extension calls `/llm/chat` and `/speech/*`, extracts `GrievanceState`, drafts the broker email, then autofills mock SCORES.
- Wealth map - portal calls `/aa/consent`, then `/aa/verify`, then `/aa/fetch`, then renders the nominee audit.
- Legal and IEPF - portal uploads to `/docs/ocr`, checks with `/docs/match`, generates with `/docs/affidavit` or `/docs/name-affidavit`, downloads PDFs.
- Shared truth - `/data/rules`, `/data/brokers`, and `/data/nodal` serve SEBI timelines, broker grievance emails, and company routing. Extension and portal read the same JSON.

## Run everything

```powershell
# terminal 1 - engine
cd sarthi-engine
python -m venv .venv
.\.venv\Scripts\activate
pip install -r requirements.txt
copy .env.example .env
python run.py

# terminal 2 - portal
cd sarthi-portal
npm install
$env:NEXT_PUBLIC_MOCK_AA="false"
$env:NEXT_PUBLIC_API_BASE="http://127.0.0.1:8787"
npm run dev
```

Open http://localhost:3000. The footer shows engine status. Mock mode needs no keys. Live Gemini OCR needs `GEMINI_API_KEY` in the engine `.env` with `MOCK_MODE=false`.

## Docs in this repo

- `ARCHITECTURE.md` records locked ports, routes, and ownership.
- `TECHNICAL.md` maps the build to the judging rubrics with mock vs live called out.
- `VOICEOVER.md` holds the demo narration.
- `DEMO.md` holds the click by click demo run.
