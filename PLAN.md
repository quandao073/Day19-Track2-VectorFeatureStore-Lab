# Plan: Day 19 Lab — Docker Path

## Tổng quan

**Stack:** Qdrant Server + Redis + Postgres + bge-m3 embeddings + FastAPI (local)
**Thời gian ước tính:** ~4-5 giờ
**RAM cần thiết:** ~3 GB free
**Prerequisite:** Docker Desktop ≥ 4.x, Python ≥ 3.10, port 6333/6379/5432 trống

---

## Phase 0 — Chuẩn bị môi trường (15 phút)

### 0.1 Kiểm tra Docker
```powershell
docker --version         # >= 24.x
docker compose version   # >= 2.x (bundled với Desktop)
docker ps                # daemon đang chạy
```

### 0.2 Kiểm tra port trống
```powershell
netstat -ano | findstr "6333"   # Qdrant HTTP
netstat -ano | findstr "6379"   # Redis
netstat -ano | findstr "5432"   # Postgres
```
Nếu bị conflict → kill process hoặc đổi port trong `docker-compose.yml`.

### 0.3 Kiểm tra Python
```powershell
python --version   # >= 3.10
```

---

## Phase 1 — Bootstrap Docker Stack (10-15 phút lần đầu)

### 1.1 Chạy setup-docker.sh (từ Git Bash hoặc WSL)
```bash
bash setup-docker.sh
```

Script này làm 6 việc theo thứ tự:
1. Docker preflight check
2. `docker compose up -d` — kéo images Qdrant v1.12.4 + Redis 7 + Postgres 16
3. Chờ 3 services healthy (poll đến 30s)
4. Tạo Python venv + install `requirements.txt` + `requirements-full.txt`
5. Jupytext convert notebooks `.py` → `.ipynb`
6. Chạy `seed_corpus.py` + `verify_docker.py`

**Trên Windows nếu không có bash:** thực hiện thủ công từng bước:
```powershell
# Bước 2-3
docker compose up -d
Start-Sleep 30
docker compose ps   # xem Health = healthy

# Bước 4
python -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install -r requirements.txt -r requirements-full.txt

# Bước 5
jupytext --to notebook --update notebooks/*.py

# Bước 6
python scripts/seed_corpus.py
python scripts/verify_docker.py
```

### 1.2 Cấu hình .env cho Docker mode
Copy `.env.example` thành `.env`, sau đó chỉnh:
```env
QDRANT_MODE=server
QDRANT_URL=http://localhost:6333
EMBEDDING_BACKEND=bge-m3
FEAST_ONLINE_STORE=redis
REDIS_URL=redis://localhost:6379/0
FEAST_OFFLINE_STORE=postgres
POSTGRES_URL=postgresql://feast:feast@localhost:5432/feast_offline
```

### 1.3 Verify services
```bash
make verify-docker
# hoặc
python scripts/verify_docker.py
```
Kết quả mong đợi:
- Qdrant: `http://localhost:6333` trả về JSON health
- Redis: `PONG`
- Postgres: `pg_isready` OK
- `data/corpus_vn.jsonl` tồn tại với 1000 docs

---

## Phase 2 — Notebook 1: Embeddings & Vector Indexing

**File:** `notebooks/01_embeddings_index.py`
**Mục tiêu:** Index 1000 vectors vào Qdrant **server** (thay vì in-memory)

### Sự khác biệt Docker vs Lite
Notebook mặc định dùng `QdrantClient(":memory:")`. Với Docker path, cần đổi sang:
```python
import os
from dotenv import load_dotenv
load_dotenv()

if os.getenv("QDRANT_MODE") == "server":
    client = QdrantClient(url=os.getenv("QDRANT_URL", "http://localhost:6333"))
else:
    client = QdrantClient(":memory:")
```

### Bước thực hiện
1. Mở Jupyter: `make lab` → http://localhost:8888
2. Mở `01_embeddings_index.ipynb`
3. Chạy cell 1-3 (load corpus, init embedder)
4. **Sửa cell 3** — đổi `QdrantClient(":memory:")` → `QdrantClient(url="http://localhost:6333")`
5. Chạy cell 4 — embed + upsert 1000 docs (lần đầu bge-m3 download ~500MB, sau đó cache)
6. Chạy cell 5-6 — similarity search + paraphrase query

**Deliverable cần chụp màn hình:**
- `Indexed: 1000 vectors`
- Top-5 results cho query tiếng Việt
- Paraphrase query vẫn trả đúng cluster

**Verify trên Qdrant Dashboard:** http://localhost:6333/dashboard → collection `lab19` → 1000 points

---

## Phase 3 — Notebook 2: Hybrid Search + RRF

**File:** `notebooks/02_hybrid_search_rrf.py`
**Mục tiêu:** BM25 + vector + RRF(k=60), đo Precision@10 trên 50 golden queries

### Bước thực hiện
1. Mở `02_hybrid_search_rrf.ipynb`
2. Chạy cells — notebook dùng cùng Qdrant client từ phase 2
3. Kiểm tra RRF formula: `score = 1/(k + rank)` với **rank bắt đầu từ 1** (không phải 0)
4. Chạy eval loop trên 50 golden queries từ `data/golden_set.jsonl`

**Deliverable cần chụp màn hình:**
- Bảng Precision@10 với 3 mode: `kw` / `sem` / `hyb`
- `hybrid` phải thắng cả 2 mode còn lại (trung bình tổng thể)
- Bảng slice theo query type: `exact` / `paraphrase` / `mixed`

**Lưu ý:**
- Nếu hybrid không thắng → check rank 1-based trong RRF
- `mixed` slice thường là nơi hybrid thắng rõ nhất

---

## Phase 4 — Notebook 3: FastAPI + Latency Benchmark

**File:** `notebooks/03_search_api_benchmark.py`
**Mục tiêu:** `/search?q=...&mode=...` endpoint, P99 < 50ms

### Bước thực hiện

1. **Start FastAPI server** (terminal riêng):
   ```powershell
   .\.venv\Scripts\Activate.ps1
   make api
   # hoặc: uvicorn app.main:app --reload --port 8000
   ```

2. **Verify API hoạt động:**
   ```powershell
   curl "http://localhost:8000/search?q=cloud+computing&mode=hybrid"
   ```

3. **Mở notebook** `03_search_api_benchmark.ipynb`

4. Chạy benchmark harness: 10 warmup queries trước, sau đó đo 100 queries × 3 modes

5. **Nếu P99 > 50ms:** chạy 10 warmup queries trước khi benchmark (cold start issue)

**Deliverable cần chụp màn hình:**
- API response JSON sample
- Bảng P50/P95/P99 cho 3 mode
- `Hybrid P99 < 50ms` PASS

---

## Phase 5 — Notebook 4: Feast Feature Store (Docker config)

**File:** `notebooks/04_feast_feature_store.py`
**Mục tiêu:** Redis online store + Postgres offline store thay vì SQLite

### 5.1 Cập nhật feature_store.yaml cho Docker
Mở `app/feast_repo/feature_store.yaml` và thay toàn bộ nội dung:
```yaml
project: lab19
provider: local
registry: registry.db
online_store:
  type: redis
  connection_string: localhost:6379
offline_store:
  type: postgres
  host: localhost
  port: 5432
  database: feast_offline
  user: feast
  password: feast
  db_schema: public
entity_key_serialization_version: 3
```

### 5.2 Chạy Notebook 4
1. Cell 1: Sinh 3 Parquet files → `app/feast_repo/data/`
2. Cell 2: `feast apply` — register 3 feature views vào registry
3. Cell 3: `feast materialize-incremental` — load offline (Parquet/Postgres) → online (Redis)
4. Cell 4: Single online lookup + đo latency
5. Cell 5: 100 lookups P50/P95/P99 (Redis thường < 5ms)
6. Cell 6: PIT join (offline) — 3 entity rows

**Nếu `feast apply` lỗi:**
```powershell
Remove-Item app/feast_repo/registry.db -ErrorAction SilentlyContinue
# Chạy lại
```

**Deliverable cần chụp màn hình:**
- `feast apply` STDOUT: "Created feature view X" × 3
- `materialize` log: rows materialized to Redis
- Online lookup result + P99 < 10ms
- PIT join DataFrame (3 rows)

---

## Phase 6 — Final Benchmark & Cleanup

### 6.1 Chạy full benchmark
```powershell
make benchmark
```
Output: bảng Precision@10 + P99 latency tổng hợp

### 6.2 Chạy test suite
```powershell
make test
```

### 6.3 Kiểm tra deliverables
| Item | Pass condition |
|---|---|
| NB1 | `n_indexed == 1000`, top-5 đúng cluster |
| NB2 | Hybrid P@10 > kw và sem |
| NB3 | API P99 < 50ms |
| NB4 | materialize OK, online P99 < 10ms |

---

## Phase 7 — Submission

1. Add screenshots vào `submission/screenshots/`:
   - `nb1_indexed.png` — 1000 vectors + top-5
   - `nb2_precision.png` — bảng Precision@10
   - `nb3_latency.png` — API response + P99 table
   - `nb4_feast.png` — feast apply + materialize + PIT join

2. Điền `submission/REFLECTION.md` (≤ 200 chữ)

3. Commit và push:
   ```powershell
   git add -A
   git commit -m "Lab 19 submission — Quan Dao Anh"
   git push -u origin main
   ```

4. Submit GitHub URL công khai vào VinUni LMS

---

## Lệnh quản lý Docker services

```powershell
make docker-up      # Khởi động (giữ data)
make docker-down    # Dừng (giữ data)
make docker-clean   # Dừng + xóa volumes (reset hoàn toàn)
```

---

## Rủi ro và xử lý

| Rủi ro | Xử lý |
|---|---|
| Port conflict 6333/6379/5432 | `docker compose down` → kill process conflict → `docker compose up -d` |
| bge-m3 download chậm (~500MB) | Dùng `fastembed` (bge-small) trước, đổi sang bge-m3 sau |
| Qdrant timeout sau `docker compose up` | Chờ 60s, kiểm tra `docker compose ps` Health = healthy |
| `feast apply` lỗi | Xóa `registry.db` + chạy lại |
| NB3 P99 > 50ms | Chạy 10 warmup queries trước khi benchmark |
| Redis connection refused | Kiểm tra `.env` có `FEAST_ONLINE_STORE=redis` và `REDIS_URL` đúng |
