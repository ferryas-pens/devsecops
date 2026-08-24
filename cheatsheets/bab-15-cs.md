# Cheatsheet Bab 15 — DAST, API Security Testing, dan Security Regression

Cheatsheet ini mengikuti pola Bab 4–14. Setiap berkas yang harus dibuat ditempatkan dalam **Kotak Script**, sedangkan setup, startup, pengujian, DAST, evidence, dan cleanup ditempatkan dalam **Kotak Perintah**. Laboratorium menggunakan aplikasi API lokal, OpenAPI, security regression test, Schemathesis, ZAP baseline scan, serta ZAP API active scan pada network Compose yang terisolasi.

> **Batas otorisasi.** Jalankan seluruh scanner hanya terhadap target lokal Bab 15 yang dimiliki dan diotorisasi. ZAP API scan melakukan active scanning dan dapat mengirim payload agresif atau request yang mengubah state. Jangan mengganti target dengan domain, IP, API, staging, atau production milik pihak lain tanpa izin tertulis, scope, jadwal, backup, dan prosedur penghentian yang jelas.

## Hasil Belajar

Setelah praktikum, mahasiswa mampu:

1. membedakan baseline passive scan dan active API scan;
2. menjalankan target DAST pada network container khusus tanpa `--network host`;
3. membuat kontrak OpenAPI yang memuat authentication dan response keamanan;
4. menguji BOLA/BFLA, authentication, error handling, dan security header;
5. menjalankan property-based API testing secara deterministik dan terbatas;
6. menetapkan ZAP rule sebagai `FAIL`, `WARN`, atau `IGNORE` berdasarkan policy;
7. membuktikan endpoint benar-benar tercakup melalui access log;
8. meredaksi credential dari laporan yang akan disimpan;
9. membuktikan gate melalui negative test; dan
10. mengumpulkan evidence dengan checksum SHA-256.

## Alur Pengujian Ringkas

```mermaid
flowchart TD
    A["Aplikasi lokal"] --> B["Regression test"]
    A --> C["Schemathesis"]
    A --> D["ZAP baseline"]
    A --> E["ZAP API active scan"]
    B --> F["Gate dan coverage"]
    C --> F
    D --> F
    E --> F
    F --> G["Evidence atau blokir"]
```

| Lapisan | Kekuatan | Keterbatasan |
| --- | --- | --- |
| Regression test | stabil untuk kelemahan yang sudah dipahami | hanya menguji kasus yang ditulis |
| Schemathesis | menghasilkan variasi input dari OpenAPI | kualitas bergantung pada kontrak API |
| ZAP baseline | crawling dan passive scan cepat | tidak melakukan active attack |
| ZAP API scan | active scan dari definisi API | dapat mengubah state dan tidak memahami seluruh logika bisnis |
| Coverage log | membuktikan URL menerima request | tidak membuktikan setiap parameter atau role diuji |

## Versi Baseline

Versi berikut diverifikasi pada 22 Agustus 2026. Periksa kembali release, digest image, add-on, dan perubahan CLI sebelum semester berikutnya.

| Komponen | Versi/tag | Fungsi |
| --- | --- | --- |
| Python | 3.13.x | aplikasi dan tooling lokal |
| Flask | 3.1.3 | API contoh |
| Gunicorn | 26.1.0 | server aplikasi |
| pytest | 9.1.1 | regression test |
| Requests | 2.32.5 | HTTP client test |
| PyYAML | 6.0.3 | membaca policy |
| Schemathesis | 4.25.0 | property-based API testing |
| ZAP | `zaproxy/zap-stable:2.17.0` | baseline dan active API scan |

## Kotak Perintah A — Menyiapkan Direktori Bab 15

```bash
$ mkdir -p ~/docker-lab/bab-15/{app,tests,policy,zap,secrets,scripts,fixtures/vulnerable,reports/raw,reports/sanitized,evidence}
$ cd ~/docker-lab/bab-15
$ touch reports/raw/.gitkeep reports/sanitized/.gitkeep evidence/.gitkeep
$ pwd
```

## Kotak Script 1 — File `.gitignore`

```bash
$ nano .gitignore
```

```gitignore
.venv/
__pycache__/
*.pyc
.pytest_cache/
.hypothesis/
.schemathesis/
secrets/*
reports/raw/*
reports/sanitized/*
evidence/*
!reports/raw/.gitkeep
!reports/sanitized/.gitkeep
!evidence/.gitkeep
```

## Kotak Script 2 — File `requirements-tooling.txt`

```bash
$ nano requirements-tooling.txt
```

```text
PyYAML==6.0.3
pytest==9.1.1
requests==2.32.5
schemathesis==4.25.0
```

## Kotak Script 3 — File `compose.yaml`

```bash
$ nano compose.yaml
```

```yaml
services:
  app:
    build:
      context: ./app
    ports:
      - "127.0.0.1:5015:8080"
    environment:
      TOKEN_FILE: /run/secrets/api_tokens.json
    secrets:
      - source: api_tokens
        target: api_tokens.json
    read_only: true
    tmpfs:
      - /tmp:size=32m,mode=1777
    cap_drop:
      - ALL
    security_opt:
      - no-new-privileges:true
    pids_limit: 100
    mem_limit: 256m
    cpus: 0.50
    networks:
      - bab15-net
    healthcheck:
      test:
        - CMD
        - python
        - -c
        - "import urllib.request; urllib.request.urlopen('http://127.0.0.1:8080/health')"
      interval: 5s
      timeout: 3s
      retries: 10
      start_period: 5s
    restart: unless-stopped

networks:
  bab15-net:
    name: bab15-net
    driver: bridge

secrets:
  api_tokens:
    file: ./secrets/api_tokens.json
```

Port host dibatasi ke loopback. ZAP memakai hostname internal `app:8080`, bukan host network atau alamat publik.

## Kotak Script 4 — File `app/requirements.txt`

```bash
$ nano app/requirements.txt
```

```text
Flask==3.1.3
gunicorn==26.1.0
```

## Kotak Script 5 — File `app/app.py`

```bash
$ nano app/app.py
```

```python
import json
import os
import secrets
from functools import wraps
from pathlib import Path

from flask import Flask, jsonify, request

app = Flask(__name__)
TOKEN_FILE = Path(os.environ.get("TOKEN_FILE", "/run/secrets/api_tokens.json"))
OPENAPI_FILE = Path(os.environ.get("OPENAPI_FILE", "/app/openapi.json"))
TOKENS = json.loads(TOKEN_FILE.read_text(encoding="utf-8"))

ITEMS = {
    1: {"id": 1, "owner": "viewer", "name": "viewer-document"},
    2: {"id": 2, "owner": "admin", "name": "admin-document"},
}


def current_role():
    authorization = request.headers.get("Authorization", "")
    if not authorization.startswith("Bearer "):
        return None
    presented = authorization.removeprefix("Bearer ")
    for role, expected in TOKENS.items():
        if secrets.compare_digest(presented, expected):
            return role
    return None


def require_role(*allowed_roles):
    def decorator(function):
        @wraps(function)
        def wrapped(*args, **kwargs):
            role = current_role()
            if role is None:
                return jsonify(error="authentication_required"), 401
            if role not in allowed_roles:
                return jsonify(error="forbidden"), 403
            return function(role, *args, **kwargs)

        return wrapped
    return decorator


@app.after_request
def security_headers(response):
    response.headers["X-Content-Type-Options"] = "nosniff"
    response.headers["X-Frame-Options"] = "DENY"
    response.headers["Content-Security-Policy"] = (
        "default-src 'none'; frame-ancestors 'none'; base-uri 'none'"
    )
    response.headers["Referrer-Policy"] = "no-referrer"
    response.headers["Cache-Control"] = "no-store"
    return response


@app.get("/health")
def health():
    return jsonify(status="healthy"), 200


@app.get("/openapi.json")
def openapi_document():
    document = json.loads(OPENAPI_FILE.read_text(encoding="utf-8"))
    return jsonify(document), 200


@app.get("/api/items")
@require_role("viewer", "admin")
def list_items(role):
    visible = list(ITEMS.values()) if role == "admin" else [ITEMS[1]]
    return jsonify(items=visible), 200


@app.get("/api/items/<int:item_id>")
@require_role("viewer", "admin")
def get_item(role, item_id):
    item = ITEMS.get(item_id)
    if item is None:
        return jsonify(error="not_found"), 404
    if role != "admin" and item["owner"] != role:
        return jsonify(error="forbidden"), 403
    return jsonify(item), 200


@app.post("/api/admin/reindex")
@require_role("admin")
def reindex(_role):
    return jsonify(status="accepted", operation="reindex"), 202


@app.errorhandler(400)
@app.errorhandler(404)
@app.errorhandler(405)
@app.errorhandler(500)
def safe_error(error):
    return jsonify(error=error.name.lower().replace(" ", "_")), error.code
```

Token dibandingkan secara konstan menggunakan `secrets.compare_digest`. Token lab tetap harus diperlakukan sebagai credential sementara dan tidak boleh masuk laporan.

## Kotak Script 6 — File `app/openapi.json`

```bash
$ nano app/openapi.json
```

```json
{
  "openapi": "3.0.3",
  "info": {
    "title": "DevSecOps Bab 15 API",
    "version": "1.0.0"
  },
  "servers": [
    {"url": "http://app:8080"}
  ],
  "paths": {
    "/health": {
      "get": {
        "operationId": "health",
        "responses": {
          "200": {
            "description": "Healthy",
            "content": {
              "application/json": {
                "schema": {
                  "type": "object",
                  "required": ["status"],
                  "properties": {"status": {"type": "string"}}
                }
              }
            }
          }
        }
      }
    },
    "/api/items": {
      "get": {
        "operationId": "listItems",
        "security": [{"BearerAuth": []}],
        "responses": {
          "200": {
            "description": "Visible items",
            "content": {
              "application/json": {
                "schema": {
                  "type": "object",
                  "required": ["items"],
                  "properties": {
                    "items": {
                      "type": "array",
                      "items": {"$ref": "#/components/schemas/Item"}
                    }
                  }
                }
              }
            }
          },
          "401": {"$ref": "#/components/responses/Unauthorized"}
        }
      }
    },
    "/api/items/{item_id}": {
      "get": {
        "operationId": "getItem",
        "security": [{"BearerAuth": []}],
        "parameters": [
          {
            "name": "item_id",
            "in": "path",
            "required": true,
            "schema": {"type": "integer", "minimum": 1, "maximum": 1000}
          }
        ],
        "responses": {
          "200": {
            "description": "Item",
            "content": {
              "application/json": {"schema": {"$ref": "#/components/schemas/Item"}}
            }
          },
          "401": {"$ref": "#/components/responses/Unauthorized"},
          "403": {"$ref": "#/components/responses/Forbidden"},
          "404": {"$ref": "#/components/responses/NotFound"}
        }
      }
    },
    "/api/admin/reindex": {
      "post": {
        "operationId": "reindex",
        "description": "Operasi idempotent khusus lab; aman untuk active scan lokal.",
        "security": [{"BearerAuth": []}],
        "responses": {
          "202": {
            "description": "Accepted",
            "content": {
              "application/json": {
                "schema": {
                  "type": "object",
                  "required": ["status", "operation"],
                  "properties": {
                    "status": {"type": "string"},
                    "operation": {"type": "string"}
                  }
                }
              }
            }
          },
          "401": {"$ref": "#/components/responses/Unauthorized"},
          "403": {"$ref": "#/components/responses/Forbidden"}
        }
      }
    }
  },
  "components": {
    "securitySchemes": {
      "BearerAuth": {"type": "http", "scheme": "bearer"}
    },
    "schemas": {
      "Item": {
        "type": "object",
        "required": ["id", "owner", "name"],
        "additionalProperties": false,
        "properties": {
          "id": {"type": "integer"},
          "owner": {"type": "string", "enum": ["viewer", "admin"]},
          "name": {"type": "string"}
        }
      },
      "Error": {
        "type": "object",
        "required": ["error"],
        "additionalProperties": false,
        "properties": {"error": {"type": "string"}}
      }
    },
    "responses": {
      "Unauthorized": {
        "description": "Authentication required",
        "content": {"application/json": {"schema": {"$ref": "#/components/schemas/Error"}}}
      },
      "Forbidden": {
        "description": "Forbidden",
        "content": {"application/json": {"schema": {"$ref": "#/components/schemas/Error"}}}
      },
      "NotFound": {
        "description": "Not found",
        "content": {"application/json": {"schema": {"$ref": "#/components/schemas/Error"}}}
      }
    }
  }
}
```

Kontrak mendeklarasikan `401`, `403`, dan `404` agar tool tidak menganggap respons keamanan tersebut sebagai respons tak terdokumentasi.

## Kotak Script 7 — File `app/Dockerfile`

```bash
$ nano app/Dockerfile
```

```dockerfile
FROM python:3.13.7-slim

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PIP_DISABLE_PIP_VERSION_CHECK=1

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir --requirement requirements.txt

COPY --chown=65532:65532 app.py openapi.json ./
USER 65532:65532
EXPOSE 8080

CMD ["gunicorn", "--bind", "0.0.0.0:8080", "--workers", "2", "--access-logfile", "-", "--error-logfile", "-", "app:app"]
```

## Kotak Script 8 — File `app/.dockerignore`

```bash
$ nano app/.dockerignore
```

```dockerignore
.git
.env
__pycache__
*.pyc
secrets
reports
tests
```

## Kotak Script 9 — File `policy/dast-policy.yaml`

```bash
$ nano policy/dast-policy.yaml
```

```yaml
policy_version: 1

authorization:
  allowed_targets:
    - http://app:8080
    - http://127.0.0.1:5015
  active_scan_allowed: true
  maximum_duration_minutes: 10

alerts:
  maximum_high: 0
  fail_plugin_ids:
    - "10020"
    - "10021"
    - "10038"

coverage:
  required_paths:
    - /health
    - /openapi.json
    - /api/items
    - /api/items/1
    - /api/admin/reindex

evidence:
  forbid_raw_reports: true
  require_redaction_check: true
```

Policy active scan hanya berlaku pada target lokal yang tercantum. Daftar target bukan izin untuk sistem lain yang kebetulan memakai path serupa.

## Kotak Script 10 — File `zap/rules.tsv`

```bash
$ nano zap/rules.tsv
```

```text
10020	FAIL	X-Frame-Options harus tersedia
10021	FAIL	X-Content-Type-Options harus tersedia
10038	FAIL	Content-Security-Policy harus tersedia
```

Rule lain tetap mengikuti default ZAP. Warning dapat ditinjau tanpa otomatis memblokir, sedangkan tiga regression rule di atas menjadi `FAIL`.

## Kotak Script 11 — File `scripts/setup-lab.sh`

```bash
$ nano scripts/setup-lab.sh
```

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

mkdir -p secrets reports/raw reports/sanitized evidence
umask 077
python3 - <<'PY'
import json
import secrets
from pathlib import Path

Path("secrets/api_tokens.json").write_text(
    json.dumps({
        "viewer": secrets.token_urlsafe(32),
        "admin": secrets.token_urlsafe(32),
    }, indent=2, sort_keys=True) + "\n",
    encoding="utf-8",
)
PY
chmod 600 secrets/api_tokens.json
docker compose config >/dev/null
printf 'PASS: token sintetis Bab 15 dibuat\n'
```

Menjalankan setup ulang merotasi token. Setelah rotasi, restart container aplikasi sebelum menjalankan test.

## Kotak Script 12 — File `tests/test_security_regression.py`

```bash
$ nano tests/test_security_regression.py
```

```python
import json
import os
from pathlib import Path

import pytest
import requests

BASE_URL = os.environ.get("BASE_URL", "http://127.0.0.1:5015")
TOKENS = json.loads(Path("secrets/api_tokens.json").read_text(encoding="utf-8"))


def bearer(role):
    return {"Authorization": f"Bearer {TOKENS[role]}"}


@pytest.mark.parametrize("path", ["/health", "/openapi.json"])
def test_security_headers(path):
    response = requests.get(f"{BASE_URL}{path}", timeout=3)
    assert response.status_code == 200
    assert response.headers["X-Content-Type-Options"] == "nosniff"
    assert response.headers["X-Frame-Options"] == "DENY"
    assert "default-src 'none'" in response.headers["Content-Security-Policy"]
    assert response.headers["Cache-Control"] == "no-store"


def test_authentication_is_required():
    response = requests.get(f"{BASE_URL}/api/items", timeout=3)
    assert response.status_code == 401


def test_invalid_token_is_rejected():
    response = requests.get(
        f"{BASE_URL}/api/items",
        headers={"Authorization": "Bearer INVALID_SYNTHETIC_TOKEN"},
        timeout=3,
    )
    assert response.status_code == 401


def test_viewer_only_sees_owned_item():
    response = requests.get(
        f"{BASE_URL}/api/items", headers=bearer("viewer"), timeout=3
    )
    assert response.status_code == 200
    assert [item["id"] for item in response.json()["items"]] == [1]


def test_bola_is_blocked():
    response = requests.get(
        f"{BASE_URL}/api/items/2", headers=bearer("viewer"), timeout=3
    )
    assert response.status_code == 403


def test_bfla_is_blocked():
    response = requests.post(
        f"{BASE_URL}/api/admin/reindex", headers=bearer("viewer"), timeout=3
    )
    assert response.status_code == 403


def test_admin_operation_is_allowed():
    response = requests.post(
        f"{BASE_URL}/api/admin/reindex", headers=bearer("admin"), timeout=3
    )
    assert response.status_code == 202


def test_unsupported_method_and_error_are_safe():
    response = requests.delete(f"{BASE_URL}/health", timeout=3)
    assert response.status_code == 405
    assert "traceback" not in response.text.lower()
    assert "exception" not in response.text.lower()


def test_openapi_declares_auth_and_denials():
    document = requests.get(f"{BASE_URL}/openapi.json", timeout=3).json()
    operation = document["paths"]["/api/items/{item_id}"]["get"]
    assert operation["security"] == [{"BearerAuth": []}]
    assert {"401", "403", "404"}.issubset(operation["responses"])
```

## Kotak Script 13 — File `scripts/run-regression.sh`

```bash
$ nano scripts/run-regression.sh
```

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

mkdir -p reports/raw
BASE_URL="${BASE_URL:-http://127.0.0.1:5015}" \
  .venv/bin/pytest -q tests/test_security_regression.py \
  --junitxml=reports/raw/pytest-security.xml
```

## Kotak Script 14 — File `scripts/run-schemathesis.sh`

```bash
$ nano scripts/run-schemathesis.sh
```

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

viewer_token="$(python3 -c "import json; print(json.load(open('secrets/api_tokens.json'))['viewer'])")"
mkdir -p reports/raw

.venv/bin/st run http://127.0.0.1:5015/openapi.json \
  --base-url http://127.0.0.1:5015 \
  --header "Authorization: Bearer $viewer_token" \
  --checks not_a_server_error,status_code_conformance,content_type_conformance,response_schema_conformance,negative_data_rejection,ignored_auth \
  --mode all \
  --max-examples 20 \
  --seed 1500 \
  --generation-deterministic \
  --rate-limit 5/s \
  --max-response-time 2 \
  --request-timeout 3 \
  --max-failures 5 \
  --report junit,ndjson \
  --report-junit-path reports/raw/schemathesis-junit.xml \
  --report-ndjson-path reports/raw/schemathesis.ndjson
```

Seed, deterministic mode, batas kasus, rate, timeout, dan failure cap menjaga test dapat direproduksi serta tidak membebani target secara berlebihan.

## Kotak Script 15 — File `scripts/run-zap-baseline.sh`

```bash
$ nano scripts/run-zap-baseline.sh
```

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

name="bab15-zap-baseline"
trap 'docker rm -f "$name" >/dev/null 2>&1 || true' EXIT
docker rm -f "$name" >/dev/null 2>&1 || true
mkdir -p reports/raw

set +e
docker run --name "$name" \
  --network bab15-net \
  --volume "$PWD/zap:/zap/config:ro" \
  zaproxy/zap-stable:2.17.0 \
  zap-baseline.py \
  -t http://app:8080 \
  -c /zap/config/rules.tsv \
  -J zap-baseline.json \
  -r zap-baseline.html \
  -m 1 \
  -I
status=$?
set -e

docker cp "$name:/zap/wrk/zap-baseline.json" reports/raw/zap-baseline.json
docker cp "$name:/zap/wrk/zap-baseline.html" reports/raw/zap-baseline.html
test "$status" -eq 0 || { printf 'FAIL: ZAP baseline exit %s\n' "$status" >&2; exit "$status"; }
printf 'PASS: ZAP baseline selesai\n'
```

Baseline melakukan spider dan passive scan; ia tidak mengirim active attack. `-I` hanya mencegah warning default menjadi kegagalan, sedangkan rule yang ditetapkan `FAIL` tetap harus ditindaklanjuti.

## Kotak Script 16 — File `scripts/run-zap-api.sh`

```bash
$ nano scripts/run-zap-api.sh
```

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

name="bab15-zap-api"
viewer_token="$(python3 -c "import json; print(json.load(open('secrets/api_tokens.json'))['viewer'])")"
trap 'docker rm -f "$name" >/dev/null 2>&1 || true' EXIT
docker rm -f "$name" >/dev/null 2>&1 || true
mkdir -p reports/raw

set +e
docker run --name "$name" \
  --network bab15-net \
  --env ZAP_AUTH_HEADER=Authorization \
  --env "ZAP_AUTH_HEADER_VALUE=Bearer $viewer_token" \
  --env ZAP_AUTH_HEADER_SITE=app \
  --volume "$PWD/zap:/zap/config:ro" \
  zaproxy/zap-stable:2.17.0 \
  zap-api-scan.py \
  -t http://app:8080/openapi.json \
  -f openapi \
  -c /zap/config/rules.tsv \
  -J zap-api.json \
  -r zap-api.html \
  -T 10 \
  -I
status=$?
set -e

docker cp "$name:/zap/wrk/zap-api.json" reports/raw/zap-api.json
docker cp "$name:/zap/wrk/zap-api.html" reports/raw/zap-api.html
docker compose logs --no-color app > reports/raw/app-access.log
test "$status" -eq 0 || { printf 'FAIL: ZAP API exit %s\n' "$status" >&2; exit "$status"; }
printf 'PASS: active API scan lokal selesai\n'
```

`-T 10` membatasi waktu startup serta pemindaian sekitar sepuluh menit. Nilai token diberikan sebagai environment container sementara; container selalu dihapus melalui `trap`.

## Kotak Script 17 — File `scripts/sanitize_reports.py`

```bash
$ nano scripts/sanitize_reports.py
```

```python
#!/usr/bin/env python3
import argparse
import json
from pathlib import Path


def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("--tokens", required=True)
    parser.add_argument("--input", required=True)
    parser.add_argument("--output", required=True)
    args = parser.parse_args()

    token_values = list(json.loads(Path(args.tokens).read_text(encoding="utf-8")).values())
    input_dir = Path(args.input)
    output_dir = Path(args.output)
    output_dir.mkdir(parents=True, exist_ok=True)

    copied = 0
    for source in sorted(input_dir.iterdir()):
        if not source.is_file() or source.name == ".gitkeep":
            continue
        data = source.read_bytes()
        for token in token_values:
            data = data.replace(token.encode(), b"[REDACTED]")
        destination = output_dir / source.name
        destination.write_bytes(data)
        copied += 1

    for destination in output_dir.iterdir():
        if destination.is_file():
            data = destination.read_bytes()
            if any(token.encode() in data for token in token_values):
                raise SystemExit(f"FAIL: token masih terdapat pada {destination}")
    print(f"PASS: {copied} laporan disanitasi")


if __name__ == "__main__":
    main()
```

Raw report dipertahankan lokal untuk triage dan tidak masuk evidence. Sanitasi tidak menggantikan pengamanan access control dan retention pada raw evidence.

## Kotak Script 18 — File `scripts/evaluate_zap.py`

```bash
$ nano scripts/evaluate_zap.py
```

```python
#!/usr/bin/env python3
import argparse
import json
from pathlib import Path

import yaml


def alerts_from(path):
    document = json.loads(Path(path).read_text(encoding="utf-8"))
    alerts = []
    for site in document.get("site", []):
        alerts.extend(site.get("alerts", []))
    return alerts


def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("--baseline", required=True)
    parser.add_argument("--api", required=True)
    parser.add_argument("--policy", required=True)
    parser.add_argument("--output", required=True)
    args = parser.parse_args()

    policy = yaml.safe_load(Path(args.policy).read_text(encoding="utf-8"))
    alerts = alerts_from(args.baseline) + alerts_from(args.api)
    high = [item for item in alerts if str(item.get("riskcode")) == "3"]
    fail_ids = set(policy["alerts"]["fail_plugin_ids"])
    explicit = [item for item in alerts if str(item.get("pluginid")) in fail_ids]
    blockers = []
    if len(high) > int(policy["alerts"]["maximum_high"]):
        blockers.append(f"High alerts {len(high)} melebihi batas")
    if explicit:
        blockers.append(f"Regression rule alerts ditemukan: {len(explicit)}")

    result = {
        "status": "PASS" if not blockers else "FAIL",
        "summary": {
            "alerts_total": len(alerts),
            "high_alerts": len(high),
            "regression_rule_alerts": len(explicit),
        },
        "blockers": blockers,
        "high_findings": [
            {"pluginid": item.get("pluginid"), "name": item.get("name")}
            for item in high
        ],
        "regression_findings": [
            {"pluginid": item.get("pluginid"), "name": item.get("name")}
            for item in explicit
        ],
    }
    Path(args.output).write_text(
        json.dumps(result, indent=2, sort_keys=True) + "\n", encoding="utf-8"
    )
    print(f"{result['status']}: hasil ditulis ke {args.output}")
    raise SystemExit(0 if result["status"] == "PASS" else 1)


if __name__ == "__main__":
    main()
```

## Kotak Script 19 — File `scripts/evaluate_coverage.py`

```bash
$ nano scripts/evaluate_coverage.py
```

```python
#!/usr/bin/env python3
import argparse
import json
from pathlib import Path

import yaml


def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("--log", required=True)
    parser.add_argument("--policy", required=True)
    parser.add_argument("--output", required=True)
    args = parser.parse_args()

    log_text = Path(args.log).read_text(encoding="utf-8", errors="replace")
    policy = yaml.safe_load(Path(args.policy).read_text(encoding="utf-8"))
    required = policy["coverage"]["required_paths"]
    covered = [path for path in required if path in log_text]
    missing = [path for path in required if path not in log_text]
    result = {
        "status": "PASS" if not missing else "FAIL",
        "required": required,
        "covered": covered,
        "missing": missing,
        "limitation": "Path coverage tidak membuktikan seluruh parameter, role, atau state tercakup.",
    }
    Path(args.output).write_text(
        json.dumps(result, indent=2, sort_keys=True) + "\n", encoding="utf-8"
    )
    print(f"{result['status']}: coverage {len(covered)}/{len(required)}")
    raise SystemExit(0 if result["status"] == "PASS" else 1)


if __name__ == "__main__":
    main()
```

## Kotak Script 20 — File `fixtures/vulnerable/app.py`

```bash
$ nano fixtures/vulnerable/app.py
```

```python
from flask import Flask, jsonify

app = Flask(__name__)


@app.get("/health")
def health():
    return jsonify(status="healthy"), 200


@app.get("/openapi.json")
def openapi_document():
    return jsonify(openapi="3.0.3", info={"title": "fixture", "version": "1"}, paths={})
```

Fixture sengaja tidak mempunyai security header dan hanya digunakan untuk membuktikan regression test dapat gagal.

## Kotak Script 21 — File `fixtures/vulnerable/Dockerfile`

```bash
$ nano fixtures/vulnerable/Dockerfile
```

```dockerfile
FROM python:3.13.7-slim
RUN pip install --no-cache-dir Flask==3.1.3 gunicorn==26.1.0
WORKDIR /app
COPY app.py .
USER 65532:65532
EXPOSE 8080
CMD ["gunicorn", "--bind", "0.0.0.0:8080", "app:app"]
```

## Kotak Script 22 — File `scripts/negative-tests.sh`

```bash
$ nano scripts/negative-tests.sh
```

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

name="bab15-vulnerable"
tmp_dir="$(mktemp -d)"
trap 'docker rm -f "$name" >/dev/null 2>&1 || true; rm -rf "$tmp_dir"' EXIT

docker build --tag bab15-vulnerable:negative fixtures/vulnerable >/dev/null
docker run --rm -d --name "$name" \
  --network bab15-net \
  --publish 127.0.0.1:5115:8080 \
  bab15-vulnerable:negative >/dev/null

for _ in $(seq 1 20); do
  curl --fail --silent http://127.0.0.1:5115/health >/dev/null && break
  sleep 1
done
curl --fail --silent http://127.0.0.1:5115/health >/dev/null \
  || { printf 'FAIL: fixture vulnerable tidak sehat\n' >&2; exit 1; }

if BASE_URL=http://127.0.0.1:5115 \
  .venv/bin/pytest -q tests/test_security_regression.py \
  --junitxml="$tmp_dir/vulnerable-junit.xml"; then
  printf 'FAIL: regression test menerima fixture tanpa header\n' >&2
  exit 1
fi
grep -q 'test_security_headers' "$tmp_dir/vulnerable-junit.xml" \
  || { printf 'FAIL: kegagalan header tidak terbukti\n' >&2; exit 1; }

python3 - "$tmp_dir" <<'PY'
import json
import sys
from pathlib import Path

root = Path(sys.argv[1])
safe = {"site": [{"alerts": []}]}
unsafe = {"site": [{"alerts": [{
    "pluginid": "99999", "riskcode": "3", "name": "Synthetic High Alert"
}]}]}
(root / "safe.json").write_text(json.dumps(safe), encoding="utf-8")
(root / "unsafe.json").write_text(json.dumps(unsafe), encoding="utf-8")
PY

if .venv/bin/python scripts/evaluate_zap.py \
  --baseline "$tmp_dir/safe.json" \
  --api "$tmp_dir/unsafe.json" \
  --policy policy/dast-policy.yaml \
  --output "$tmp_dir/policy-result.json"; then
  printf 'FAIL: evaluator menerima alert High sintetis\n' >&2
  exit 1
fi
grep -q 'Synthetic High Alert' "$tmp_dir/policy-result.json"

printf 'PASS: regression fixture dan alert High sintetis ditolak\n'
```

Negative test sengaja mengharapkan pytest dan evaluator menghasilkan exit code gagal. Alert `99999` adalah fixture sintetis, bukan rule ZAP nyata.

## Kotak Script 23 — File `scripts/collect-evidence.sh`

```bash
$ nano scripts/collect-evidence.sh
```

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

test -s reports/sanitized/zap-baseline.json
test -s reports/sanitized/zap-api.json
test -s reports/sanitized/pytest-security.xml
test -s reports/sanitized/schemathesis-junit.xml
test -s reports/sanitized/zap-policy.json
test -s reports/sanitized/coverage.json

rm -rf evidence/*
cp reports/sanitized/* evidence/
cp policy/dast-policy.yaml zap/rules.tsv evidence/

{
  printf 'collected_at_utc=%s\n' "$(date -u +%Y-%m-%dT%H:%M:%SZ)"
  printf 'zap_image=zaproxy/zap-stable:2.17.0\n'
  printf 'schemathesis=4.25.0\n'
  printf 'target=http://app:8080\n'
  printf 'authorization=local-lab-only\n'
} > evidence/tool-context.txt

find evidence -maxdepth 1 -type f ! -name SHA256SUMS.txt -print0 \
  | sort -z | xargs -0 sha256sum > evidence/SHA256SUMS.txt
printf 'PASS: sanitized evidence dan checksum tersedia\n'
```

## Kotak Script 24 — File `Makefile`

> Baris perintah pada Makefile harus diawali karakter **Tab**, bukan spasi.

```bash
$ nano Makefile
```

```makefile
.PHONY: setup up regression schema-test baseline api-scan evaluate negative-test evidence all clean

setup:
	bash scripts/setup-lab.sh
	python3 -m venv .venv
	.venv/bin/pip install --requirement requirements-tooling.txt

up:
	docker compose up -d --build
	docker compose ps

regression:
	bash scripts/run-regression.sh

schema-test:
	bash scripts/run-schemathesis.sh

baseline:
	bash scripts/run-zap-baseline.sh

api-scan:
	bash scripts/run-zap-api.sh

evaluate:
	.venv/bin/python scripts/sanitize_reports.py --tokens secrets/api_tokens.json --input reports/raw --output reports/sanitized
	.venv/bin/python scripts/evaluate_zap.py --baseline reports/sanitized/zap-baseline.json --api reports/sanitized/zap-api.json --policy policy/dast-policy.yaml --output reports/sanitized/zap-policy.json
	.venv/bin/python scripts/evaluate_coverage.py --log reports/sanitized/app-access.log --policy policy/dast-policy.yaml --output reports/sanitized/coverage.json

negative-test:
	bash scripts/negative-tests.sh

evidence:
	bash scripts/collect-evidence.sh

all: regression schema-test baseline api-scan evaluate negative-test evidence

clean:
	docker rm -f bab15-zap-baseline bab15-zap-api bab15-vulnerable 2>/dev/null || true
	docker image rm bab15-vulnerable:negative 2>/dev/null || true
	rm -rf reports/raw/* reports/sanitized/* evidence/* .hypothesis .schemathesis
```

## Kotak Perintah B — Setup, Validasi, dan Startup

```bash
$ chmod +x scripts/*.sh scripts/*.py
$ bash -n scripts/*.sh
$ python3 -m py_compile app/app.py tests/*.py scripts/*.py fixtures/vulnerable/app.py
$ python3 -m json.tool app/openapi.json >/dev/null
$ python3 -c "import yaml; yaml.safe_load(open('policy/dast-policy.yaml'))"
$ make setup
$ make up
```

Tunggu hingga aplikasi `healthy`:

```bash
$ docker compose ps
$ curl --fail http://127.0.0.1:5015/health
```

## Kotak Perintah C — Functional dan Security Regression

```bash
$ make regression
$ python3 -m xml.etree.ElementTree reports/raw/pytest-security.xml >/dev/null
```

## Kotak Perintah D — Property-Based API Testing

```bash
$ make schema-test
$ python3 -m xml.etree.ElementTree reports/raw/schemathesis-junit.xml >/dev/null
```

Jika ditemukan case gagal, simpan seed dan reproducer yang dihasilkan Schemathesis. Jangan sekadar menaikkan failure limit.

## Kotak Perintah E — ZAP Baseline Passive Scan

```bash
$ make baseline
$ python3 -m json.tool reports/raw/zap-baseline.json >/dev/null
```

## Kotak Perintah F — ZAP API Active Scan

```bash
$ make api-scan
$ python3 -m json.tool reports/raw/zap-api.json >/dev/null
```

Active scan hanya boleh dijalankan setelah memastikan target berikut:

```text
http://app:8080
```

## Kotak Perintah G — Sanitasi, Policy, dan Coverage Gate

```bash
$ make evaluate
$ python3 -m json.tool reports/sanitized/zap-policy.json
$ python3 -m json.tool reports/sanitized/coverage.json
```

Kedua hasil wajib berstatus `PASS`. Finding nol tanpa coverage yang memadai bukan hasil aman.

## Kotak Perintah H — Negative Test

```bash
$ make negative-test
```

Keluaran wajib:

```text
PASS: regression fixture dan alert High sintetis ditolak
```

## Kotak Perintah I — Mengumpulkan Evidence

```bash
$ make evidence
$ sha256sum --check evidence/SHA256SUMS.txt
$ find evidence -maxdepth 1 -type f -printf '%f\n' | sort
```

Raw report dan token tidak ikut evidence. Jika organisasi harus menyimpan raw traffic, gunakan artifact store terbatas, enkripsi, retention, dan redaction workflow yang disetujui.

## Kriteria PASS/FAIL

| Pengujian | PASS | FAIL |
| --- | --- | --- |
| Scope | hanya `app:8080` dan loopback Bab 15 | scanner diarahkan ke target lain |
| Runtime | app sehat, non-root, read-only | root, writable filesystem, atau restart loop |
| Authentication | tanpa/invalid token menghasilkan 401 | request diterima |
| BOLA | viewer tidak dapat membaca item admin | object ID melewati otorisasi |
| BFLA | viewer tidak dapat menjalankan fungsi admin | role rendah memperoleh 2xx |
| Header | regression header lulus | header wajib hilang |
| Error handling | JSON aman tanpa traceback | stack trace atau detail internal bocor |
| OpenAPI | operasi, security, 401/403 terdokumentasi | endpoint tidak masuk inventaris |
| Schemathesis | conformance/security checks lulus | 5xx atau kontrak tidak sesuai |
| ZAP baseline | tidak ada regression rule alert | rule header muncul |
| ZAP API | tidak ada High dan active scope tepat | High finding atau target salah |
| Coverage | seluruh path wajib terlihat pada log | satu atau lebih path tidak dikunjungi |
| Negative test | fixture insecure dan High sintetis ditolak | salah satu diterima |
| Redaction | token tidak ditemukan pada sanitized report | token masih terdapat dalam evidence |
| Evidence | checksum valid dan hanya sanitized report | raw report/secret ikut tersimpan |

## Troubleshooting Cepat

| Gejala | Kemungkinan penyebab | Tindakan |
| --- | --- | --- |
| ZAP tidak mencapai aplikasi | network atau hostname salah | pastikan `bab15-net` aktif dan target `http://app:8080` |
| ZAP keluar dengan kode 2 | warning belum diklasifikasikan | tinjau alert; ubah rule berdasarkan risk decision, bukan agar pipeline hijau |
| ZAP report tidak dapat di-copy | container terhapus terlalu cepat atau path salah | pertahankan `trap`; periksa `/zap/wrk` sebelum cleanup |
| API scan menghasilkan 401 semua | auth header environment tidak diterapkan | periksa nama site `app` dan secret token, tanpa mencetak token |
| Schemathesis gagal pada 403 | OpenAPI tidak mendeklarasikan response keamanan | tambahkan response yang sah atau perbaiki otorisasi |
| Schemathesis terlalu berat | kasus/rate/schema terlalu luas | kurangi `max-examples`, rate, scope, dan durasi pada lab |
| Coverage gagal untuk admin endpoint | scan memakai viewer token | regression test admin harus tercatat; jalankan regression sebelum mengambil log |
| Alert CSP muncul | header tidak diterapkan pada seluruh response | periksa `after_request`, error response, dan proxy |
| Sanitasi gagal | token masih tertanam pada format laporan | jangan publikasikan raw report; perbaiki sanitizer dan rotasi token bila terekspos |
| Hasil berubah antar-run | add-on, image, seed, data, atau timing berubah | pin versi/digest, simpan seed, konfigurasi, dan waktu scan |

## Kotak Perintah J — Cleanup Aman

```bash
$ make clean
$ docker compose down --volumes --remove-orphans
$ rm -rf .venv secrets/api_tokens.json
```

Cleanup menghapus token, laporan, evidence, dan fixture lokal. Pastikan evidence yang dibutuhkan telah dipindahkan sebelum menjalankannya.

## Integrasi dengan Bab Sebelumnya

- **Bab 10:** jalankan regression dan Schemathesis pada setiap perubahan; jalankan active DAST hanya pada deployment test yang berizin.
- **Bab 11:** tautkan BOLA, BFLA, broken authentication, dan security misconfiguration ke threat serta security requirement.
- **Bab 12:** ubah temuan DAST yang telah dipahami menjadi unit/abuse/security regression test.
- **Bab 13:** catat versi ZAP, Schemathesis, dan image target bersama SBOM serta hasil SCA.
- **Bab 14:** scan hanya digest image yang signature dan provenance-nya telah diverifikasi.

## Catatan Interpretasi

1. ZAP baseline menjalankan spider dan passive scan, sedangkan ZAP API scan melakukan active scan terhadap endpoint yang diimpor dari OpenAPI.
2. `-I` tidak boleh menjadi alasan untuk mengabaikan warning. Warning perlu triage, owner, disposition, dan expiry bila diterima.
3. DAST anonymous tidak membuktikan keamanan area authenticated. Lab ini menambahkan viewer token, tetapi pengujian seluruh role tetap memerlukan regression test.
4. OpenAPI yang tidak lengkap menyebabkan endpoint tidak diuji. Inventaris API dan access log adalah bagian dari evidence coverage.
5. Tidak ada finding tidak sama dengan tidak ada kerentanan. Hasil dibatasi oleh crawler, schema, authentication, scanner, waktu, state, dan data uji.
6. Active scan tidak memahami seluruh abuse case bisnis. BOLA/BFLA diuji eksplisit melalui regression test.

## Rujukan Resmi

- [ZAP Docker documentation](https://www.zaproxy.org/docs/docker/)
- [ZAP Baseline Scan](https://www.zaproxy.org/docs/docker/baseline-scan/)
- [ZAP API Scan](https://www.zaproxy.org/docs/docker/api-scan/)
- [ZAP 2.17.0 releases](https://www.zaproxy.org/docs/desktop/releases/)
- [Schemathesis CLI](https://schemathesis.readthedocs.io/en/latest/reference/cli/)
- [Schemathesis checks](https://schemathesis.readthedocs.io/en/stable/reference/checks/)
- [Schemathesis v4.25.0](https://github.com/schemathesis/schemathesis/releases/tag/v4.25.0)
- [OWASP API Security Top 10 2023](https://owasp.org/API-Security/editions/2023/en/0x11-t10/)
- [OWASP Top 10:2025](https://owasp.org/Top10/2025/)

## Checklist Akhir

- [ ] Scanner hanya berada pada network `bab15-net`.
- [ ] Active scan hanya mengarah ke `http://app:8080`.
- [ ] Aplikasi berjalan non-root, read-only, dan hanya dipublikasikan ke loopback.
- [ ] OpenAPI mendokumentasikan authentication serta response 401/403.
- [ ] BOLA, BFLA, invalid token, header, dan safe error diuji.
- [ ] Schemathesis memakai seed, rate, timeout, dan case limit.
- [ ] ZAP baseline dan API report tersedia.
- [ ] Coverage path diverifikasi melalui access log.
- [ ] Negative test membuktikan test dan evaluator dapat gagal.
- [ ] Token tidak ditemukan pada sanitized report atau evidence.
- [ ] Checksum evidence valid.

**[High confidence]** Struktur, policy, kontrak API, regression test, dan evaluator dapat diperiksa secara statis.  
**[Medium confidence]** Alert aktual dan kompatibilitas runtime bergantung pada Docker, arsitektur CPU, add-on ZAP, jaringan, timing, serta versi tool saat praktikum dijalankan.
