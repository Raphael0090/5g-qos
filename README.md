# 5G QoS Prediction: cloud-connected system

Phone (Android app) -> internet -> Flask API on Render -> Postgres on Neon -> ML prediction -> back to the phone, plus a live web dashboard.

```
android/            Kotlin app (open this folder in Android Studio)
app.py              Flask API + dashboard host
config.py           feature list, paths, DATABASE_URL handling
ml_core.py          column mapping, labelling, synthetic data, predict()
train.py            training pipeline (compares 2 models, keeps the best)
db.py / compare.py  database layer / "live reading vs dataset" logic
templates/          dashboard.html
data/               qos_dataset.csv  <- put YOUR dataset here (replaces the synthetic one)
model/              trained model + model_info.json (created by train.py)
```

## 0. Use your own dataset
1. Replace `data/qos_dataset.csv` with your CSV.
2. If your column names differ, add them to `ALIASES` in `ml_core.py` (many common names already work: Signal Strength, Latency, Throughput, Network Type...).
3. If your CSV has no QoS label column, labels are derived from a rule-based score (see `qos_score` in `ml_core.py`). If it has one, your labels are used as they are.
4. The model trains on whichever of these features exist in your CSV: rsrp_dbm, rsrq_db, sinr_db, latency_ms, jitter_ms, packet_loss_pct, download_mbps, upload_mbps, is_5g. The dashboard and app show exactly which were used.

## 1. Run locally (no cloud)
```
python -m venv .venv && source .venv/bin/activate      # Windows: .venv\Scripts\activate
pip install -r requirements.txt
python train.py          # trains and prints metrics
python app.py            # http://localhost:5000  (SQLite file in instance/)
```
Open http://localhost:5000. Test ingest without the phone:
```
curl -X POST localhost:5000/api/ingest -H "Content-Type: application/json" \
  -d '{"device_id":"test","rsrp_dbm":-85,"sinr_db":18,"latency_ms":25,"download_mbps":120,"upload_mbps":20,"is_5g":1}'
```

## 2. Free cloud database (Neon)
1. Sign up at neon.tech, create a project, copy the connection string (`postgresql://user:pass@ep-xxx.neon.tech/neondb?sslmode=require`).
2. If the app later fails with a `channel_binding` error, delete `&channel_binding=require` from the string.
Tables are created automatically on first start, and your dataset is loaded into the database automatically.

## 3. Deploy the backend (Render, free)
1. Create a GitHub repo and push this folder (the Android folder can stay in it).
2. render.com -> New -> Blueprint (or Web Service) -> pick the repo. `render.yaml` sets everything.
3. Environment variables: `DATABASE_URL` = your Neon string. `API_KEY` is generated for you (Render dashboard -> Environment -> copy it).
4. Deploy. Your URL will look like `https://qos-cloud.onrender.com`. Check `/api/health`, then open `/` for the dashboard.
Notes: the free instance sleeps after ~15 minutes without traffic and takes up to a minute to wake; the app handles this (long timeouts, queued samples). An UptimeRobot monitor on `/api/health` keeps it awake.

## 4. Build the Android app
1. Android Studio -> Open -> select the `android/` folder. Accept any Gradle/AGP upgrade prompt.
2. Phone: Settings -> About -> tap Build number 7 times -> Developer options -> enable USB debugging. Plug in once to install (Run button), or use Build -> Build APK(s) and copy the APK to the phone. After install, no cable is needed.
3. Open the app, enter the Render URL and API key, tap Start, allow Phone and Location permissions.
4. Turn off battery optimization for the app (Settings -> Apps -> QoS Monitor -> Battery -> Unrestricted) so measuring continues with the screen off.

## 5. Upload a new dataset / retrain later
```
curl -X POST https://YOUR-APP.onrender.com/api/dataset/upload -H "X-API-Key: YOUR_KEY" \
  -F file=@mydata.csv -F mode=replace -F retrain=1
```
`mode=append` adds rows instead of replacing. `POST /api/retrain` retrains from the stored dataset. The retrained model is saved in the database, so it survives restarts.
To train on real phone data: download `/api/readings.csv` from the dashboard, add a QoS label column, and upload it with `mode=append`.

## API summary
| Endpoint | Purpose |
|---|---|
| POST /api/ingest (key) | phone sends a reading, gets the prediction |
| POST /api/predict | predict only, nothing saved |
| GET /api/readings, /api/readings.csv, /api/stats | history |
| GET /api/model | model name, features, metrics, importances |
| GET /api/compare, GET /api/dataset/summary | live vs dataset |
| POST /api/dataset/upload, /api/retrain (key) | dataset and model management |
| GET /api/ping, /api/speedtest/download, POST /api/speedtest/upload | used by the app's latency and speed tests |

## Things worth knowing
- Latency and speed are measured against your server over HTTP, so they include server distance and free-tier slowness. They are consistent for QoS tracking, not a replacement for a carrier-grade speed test.
- If the phone is on Wi-Fi, the app binds its tests to the mobile network (checkbox in the app) so the data describes cellular QoS. If mobile data is off, tests fall back to Wi-Fi and the `transport` column shows WIFI.
- Each measurement uses about 1.3 MB. At the default 30 s interval that is roughly 150 MB/hour on mobile data; raise the interval to reduce it.
- `usesCleartextTraffic` is enabled in the manifest only for LAN testing; remove it once you use the https URL.
- The API key protects writing. The dashboard and read endpoints are public, so do not share the URL if the data is sensitive.
