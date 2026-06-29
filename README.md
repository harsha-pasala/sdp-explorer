# SDP Explorer

Interactive feature demonstrations for **Lakeflow Spark Declarative Pipelines (SDP)** — a more intuitive, visual companion to the docs. Built for field enablement (scaling SDP adoption in Canada).

Each feature is a browsable card whose hero is an **animation of what the feature does to your data**, plus a "when to leverage / watch out" digest. Cards live in `static/index.html` and are driven by a `NAV` model + per-feature render/wire functions — adding a demo is one `NAV` entry, one `DEMOS[id]` render fn, and one wire fn.

## Structure

```
sdp-explorer/
├── app.py            # FastAPI static server (binds 0.0.0.0:$DATABRICKS_APP_PORT)
├── app.yaml          # Databricks Apps runtime command
├── databricks.yml    # DAB deployment config
├── requirements.txt  # fastapi + uvicorn
└── static/
    ├── index.html         # the app (all cards, styles, animation logic)
    └── databricks-logo.svg
```

## Run locally

```bash
pip install -r requirements.txt
python app.py            # serves on http://localhost:8000
```

## Deploy to Databricks Apps

**Via Asset Bundle (recommended):**

```bash
databricks bundle deploy --profile <PROFILE>
databricks bundle run sdp-explorer --profile <PROFILE>
```

**Or via the Apps CLI directly:**

```bash
databricks apps create sdp-explorer --profile <PROFILE>
databricks sync . "/Workspace/Users/<you>/sdp-explorer" --profile <PROFILE>
databricks apps deploy sdp-explorer \
  --source-code-path "/Workspace/Users/<you>/sdp-explorer" --profile <PROFILE>
```

Verify:

```bash
databricks apps get sdp-explorer --profile <PROFILE> -o json   # app_status.state == RUNNING
databricks apps logs sdp-explorer --follow --profile <PROFILE>
```
