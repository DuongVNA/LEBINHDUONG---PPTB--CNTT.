#!/usr/bin/env bash
set -euo pipefail

# Config
HEALTH_URL="${HEALTH_URL:?missing HEALTH_URL}"   # export trước trong CI
TIMEOUT_SEC="${TIMEOUT_SEC:-3}"                  # timeout cho mỗi request
MAX_ATTEMPTS="${MAX_ATTEMPTS:-10}"               # số lần thử
NOTIFY_CMD="${NOTIFY_CMD:-}"                     # ví dụ: ./scripts/notify.sh

attempt=0
until (( attempt >= MAX_ATTEMPTS )); do
  attempt=$((attempt+1))
  echo "Healthcheck attempt ${attempt}/${MAX_ATTEMPTS} -> ${HEALTH_URL}"

  # readyz trước, fallback sang healthz nếu cần
  if curl -fsS --max-time "$TIMEOUT_SEC" "${HEALTH_URL%/}/readyz" \
    || curl -fsS --max-time "$TIMEOUT_SEC" "${HEALTH_URL%/}/healthz"; then
    echo "✅ Service healthy"
    exit 0
  fi

  # exponential backoff: 1s, 2s, 4s, 8s, ...
  sleep_sec=$(( 2 ** (attempt-1) ))
  echo "❌ Not healthy yet, retry in ${sleep_sec}s"
  sleep "$sleep_sec"
done

echo "❌ Healthcheck failed after ${MAX_ATTEMPTS} attempts"
# Gửi cảnh báo (không fail nếu notify lỗi)
if [[ -n "$NOTIFY_CMD" ]]; then
  set +e
  "$NOTIFY_CMD" "Healthcheck failed for ${HEALTH_URL} after ${MAX_ATTEMPTS} attempts"
  set -e
fi
exit 1
#!/usr/bin/env bash
set -euo pipefail

# Yêu cầu xác định version/sha trước đó
PREV_SHA="${PREV_SHA:?missing PREV_SHA}"
ENVIRONMENT="${ENVIRONMENT:-production}"

echo "↩️  Rolling back ${ENVIRONMENT} to ${PREV_SHA}"
# Ví dụ với Kubernetes + image tag theo SHA
# kubectl set image deploy/my-svc my-svc=myrepo/my-svc:${PREV_SHA} --record
# kubectl rollout status deploy/my-svc --timeout=120s

echo "✅ Rollback command issued to ${PREV_SHA}"

#!/usr/bin/env bash
set -euo pipefail
MSG="${1:-(no message)}"
# Ví dụ gửi Slack webhook
if [[ -n "${SLACK_WEBHOOK_URL:-}" ]]; then
  curl -fsS -X POST -H 'Content-type: application/json' \
    --data "{\"text\":\"${MSG}\"}" "$SLACK_WEBHOOK_URL" >/dev/null
else
  echo "(no SLACK_WEBHOOK_URL) ${MSG}"
fi
# .github/workflows/deploy.yml (trích đoạn)
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build & Deploy
        run: ./scripts/deploy.sh
        env:
          IMAGE_SHA: ${{ github.sha }}

      - name: Healthcheck (retry+backoff)
        run: ./scripts/healthcheck.sh
        env:
          HEALTH_URL: ${{ secrets.HEALTH_URL }}
          TIMEOUT_SEC: 3
          MAX_ATTEMPTS: 8
          NOTIFY_CMD: ./scripts/notify.sh
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}

      - name: Rollback if failed
        if: failure()
        run: ./scripts/rollback.sh
        env:
          PREV_SHA: ${{ vars.PREV_SHA }}   # cập nhật PREV_SHA trước deploy
          ENVIRONMENT: production

      - name: Notify failure
        if: failure()
        run: ./scripts/notify.sh "Deploy failed, rolled back to ${{ vars.PREV_SHA }}"
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}

WITH events AS (
  SELECT
    date_trunc('day', event_time) AS event_day,
    origin, destination,
    session_id,
    action
  FROM booking_logs
  -- WHERE event_time >= now() - interval '30 days'
),
per_session AS (
  -- mỗi session đếm tối đa 1 SEARCH và 1 BOOK (hoặc theo logic của bạn)
  SELECT
    event_day, origin, destination, session_id,
    (COUNT(*) FILTER (WHERE action='SEARCH') > 0)::int AS has_search,
    (COUNT(*) FILTER (WHERE action='BOOK')   > 0)::int AS has_book
  FROM events
  GROUP BY 1,2,3,4
)
SELECT
  event_day,
  origin, destination,
  SUM(has_book)::float   AS books,
  SUM(has_search)::float AS searches,
  CASE WHEN SUM(has_search)=0 THEN NULL
       ELSE SUM(has_book)::float / SUM(has_search)::float
  END AS book_rate,        -- BOOK / SEARCH
  CASE WHEN SUM(has_book)=0 THEN NULL
       ELSE SUM(has_search)::float / SUM(has_book)::float
  END AS ltb_ratio         -- SEARCH / BOOK (cách gọi LTB phổ biến)
FROM per_session
GROUP BY 1,2,3
ORDER BY event_day DESC, origin, destination;
from flask import Flask, jsonify, request
import os, time

app = Flask(__name__)
START_TS = time.time()
READY = False

@app.before_first_request
def _warmup():
    global READY
    time.sleep(0.5)
    READY = True

@app.get("/healthz")
def healthz():
    return jsonify(status="ok", uptime=time.time()-START_TS), 200

@app.get("/readyz")
def readyz():
    if READY:
        return jsonify(ready=True), 200
    return jsonify(ready=False), 503

@app.get("/api/v1/ping")
def ping():
    return jsonify(pong=True, ip=request.remote_addr)

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=int(os.environ.get("PORT", "8000")))

