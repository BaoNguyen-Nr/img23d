# img23d

Web app nhận một ảnh, trả về một mesh 3D xem được ngay trong trình duyệt và tải về dạng `.glb`.

Mục tiêu học tập chính của dự án là **hạ tầng triển khai** — Git, FastAPI, hàng đợi, Docker, CI/CD, monitoring. Mô hình dùng checkpoint công khai, không train lại; phần code tay là xử lý hình học và toàn bộ lớp triển khai.

> **Trạng thái: giai đoạn 0/8 — khởi tạo repo.**
> Cây thư mục và công cụ chất lượng mã đã dựng xong. Chưa có dòng code chạy được nào: `pytest` sẽ báo không tìm thấy test, và chưa có gì để `docker compose up`. Phần "Tính năng" bên dưới mô tả đích đến, không phải hiện trạng.

## Tính năng (đích đến)

Một ảnh vào, một mesh ra. Xử lý mất từ 15 giây đến vài phút tuỳ engine, nên job chạy nền và giao diện hiển thị tiến độ theo bước.

Hai engine nằm sau cùng một interface:

| Engine | Cách làm | Hợp với | Giới hạn |
| --- | --- | --- | --- |
| **depth** | Depth Anything V2 Small sinh bản đồ độ sâu → point cloud → mesh | Cảnh, phòng, phong cảnh | Kết quả là 2.5D: chỉ có mặt nhìn thấy |
| **triposr** | Ảnh đơn → mesh kín, có cả mặt sau | Vật thể đơn đã tách nền | Phần khuất là suy đoán, không có đơn vị mét |

VLM đóng vai phụ, không phải xương sống: chọn engine theo ảnh, sinh mô tả asset, và chấm chất lượng mesh bằng cách render 4 góc rồi hỏi lại. Hệ thống chạy được khi không có nó.

Ngoài phạm vi: multi-view, texture PBR, đo đạc theo mét, chỉnh sửa mesh, tài khoản người dùng.

Ràng buộc nền tảng: **chỉ có CPU**. Suy luận chạy được, chỉ là chậm.

## Kiến trúc

Nguyên tắc chi phối toàn bộ thiết kế: **API không bao giờ chạy model.** Nó nhận ảnh, ghi xuống volume, đẩy job vào Redis và trả `job_id` trong vài chục mili giây. Worker mới là nơi tốn vài phút CPU.

Với dự án này, tách như vậy không còn là lựa chọn kiến trúc mà là điều kiện cần — TripoSR mất tới một phút cho một ảnh, không request/response đồng bộ nào sống nổi qua đó.

| Service | Vì sao cần |
| --- | --- |
| `nginx` | Phục vụ file tĩnh và viewer, giới hạn kích thước upload, trả `.glb` đúng MIME type |
| `api` | Biên giới HTTP, xác thực đầu vào, phát tiến độ qua SSE |
| `worker` | Giữ session ONNX trong RAM, chạy inference và dựng mesh, cập nhật tiến độ |
| `redis` | Hàng đợi, trạng thái job, cache theo hash ảnh |
| `prometheus` | Thu metric từ `/metrics` của api và worker |
| `grafana` | Dashboard đọc từ prometheus |

Khác biệt lớn nhất so với một API thông thường: **worker phải có trần RAM.** TripoSR có thể ngốn vài GB; ba worker không giới hạn trên máy 16 GB sẽ làm treo cả máy chứ không phải chết gọn một container. `mem_limit` đặt từ số đo thật ở giai đoạn 3, cộng 30% dự phòng.

Chi tiết trong `docs/architecture.md`, quyết định kiến trúc trong `docs/adr/`.

## Yêu cầu

- Python **3.11** hoặc **3.12** (máy phát triển dùng 3.11, container dùng 3.12-slim — CI chạy cả hai từ giai đoạn 6)
- Git
- Docker Desktop (từ giai đoạn 5 trở đi)
- RAM: tối thiểu 8 GB cho engine depth, 16 GB nếu chạy TripoSR

## Cài đặt cho phát triển

PowerShell (Windows):

```powershell
git clone https://github.com/BaoNguyen-Nr/img23d.git
cd img23d
python -m venv .venv
.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
pip install -e ".[dev]"
pre-commit install
```

bash (Linux/macOS):

```bash
git clone https://github.com/BaoNguyen-Nr/img23d.git
cd img23d
python -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
pip install -e ".[dev]"
pre-commit install
```

Một lệnh `pre-commit install` là đủ: file cấu hình khai `default_install_hook_types`, nên cả hook `pre-commit` lẫn hook `commit-msg` đều được cài. Kiểm tra bằng cách xem hai file `.git/hooks/pre-commit` và `.git/hooks/commit-msg` có tồn tại không.

**Luôn kích hoạt `.venv` trước khi commit.** Hook ruff và mypy gọi thẳng công cụ trong venv của dự án để phiên bản ở máy, ở hook và ở CI luôn trùng nhau; không kích hoạt venv thì hook báo không tìm thấy lệnh.

Cài thêm theo giai đoạn:

```bash
pip install -e ".[dev,nb]"       # giai đoạn 1 — notebook thử ngưỡng lọc điểm
pip install -e ".[dev,worker]"   # giai đoạn 2 — depth, hình học, dựng mesh
pip install -e ".[dev,api]"      # giai đoạn 4 — FastAPI
```

## Lệnh thường dùng

| Lệnh | Việc |
| --- | --- |
| `ruff check .` | Lint |
| `ruff format .` | Format |
| `mypy src` | Kiểm kiểu |
| `pytest` | Test nhanh (bỏ qua test gắn marker `slow`) |
| `pytest -m slow` | Test chậm — inference thật, dựng mesh đầy đủ |
| `pytest --cov` | Test kèm báo cáo coverage |
| `pre-commit run --all-files` | Chạy toàn bộ hook trên cả repo |
| `pre-commit autoupdate` | Nâng phiên bản hook |

## Cấu trúc thư mục

```text
src/img23d/          package Python cài được — mọi thứ khác là hạ tầng quanh nó
  api/               FastAPI: routes, schemas, dependency injection
  worker/            RQ worker, nạp session ONNX một lần rồi warm-up
  engines/           base.py định nghĩa interface; depth, triposr, router
  geometry/          unproject, lọc điểm, dựng mesh, giảm mặt, xuất .glb
  preprocess/        chuẩn hoá EXIF/resize/hash, tách nền
  quality.py         chấm mesh: số mặt, kín/hở, VLM
models/              checkpoint .onnx — KHÔNG commit
scripts/             tải checkpoint, export ONNX, benchmark latency và RAM
notebooks/           thử ngưỡng lọc điểm, so hai engine
frontend/            upload, tiến độ, viewer Three.js
tests/               unit (hình học test bằng dữ liệu giả), integration, fixtures
docker/              Dockerfile ba tầng + nginx
deploy/              prometheus, dashboard grafana
load/                kịch bản Locust
docs/                architecture, runbook, ADR
```

Bốn lựa chọn đáng giải thích:

- **`geometry/` tách khỏi `engines/`.** Model trả về tensor; biến tensor thành mesh là việc khác hẳn. Tách ra thì toàn bộ phần hình học test được bằng dữ liệu giả, không cần nạp model nào.
- **`engines/base.py` là một interface duy nhất.** Thêm engine thứ ba chỉ là thêm một file, không sửa worker.
- **`src/` layout.** Buộc phải cài package mới import được, nên test chạy trên đúng thứ sẽ đóng gói.
- **`quality.py` đứng riêng.** Chấm chất lượng áp dụng cho mesh bất kể nó từ engine nào ra.

## Quy ước Git

| Nhánh | Vai trò | Ai được đẩy thẳng |
| --- | --- | --- |
| `main` | Chỉ chứa mã đã phát hành, mỗi commit một tag | Không ai |
| `develop` | Nhánh tích hợp, luôn phải xanh | Không ai |
| `feature/<mô-tả>` | Một việc, một nhánh, sống dưới 3 ngày | Có |
| `fix/<mô-tả>` | Sửa lỗi trên develop | Có |
| `hotfix/<mô-tả>` | Sửa khẩn trên main, merge ngược về develop | Có |

Nhánh đặt tên theo việc, không theo người: `feature/onnx-export`, không phải `feature/bao-test`.

Commit theo [Conventional Commits](https://www.conventionalcommits.org/), kiểm tự động bằng hook `commit-msg`. Scope trùng tên thư mục trong `src/`:

```text
feat(geometry): loc diem bien theo gradient depth
fix(worker): chia nho chunk diem truy van de khong vo RAM
chore(docker): dat mem_limit cho worker tu so do benchmark
docs(adr): ghi lai nguong loc diem da chon
```

Scope hợp lệ: `api`, `worker`, `engines`, `geometry`, `preprocess`, `quality`, `storage`, `frontend`, `docker`, `ci`, `docs`, `deps`, `repo`.

Quy trình: nhánh từ `develop` → commit nhỏ → PR vào `develop` → CI xanh → được duyệt → squash merge → xoá nhánh. **Không bao giờ merge khi CI đỏ.**

## Quyền riêng tư và dữ liệu

Ảnh người dùng tải lên là dữ liệu cá nhân. Dự án cam kết:

- **Ảnh và mesh tự xoá sau 24 giờ** — không có lưu trữ lâu dài
- **Không ghi nội dung ảnh vào log** — log chỉ chứa `job_id`, `request_id`, engine, kích thước và thời gian xử lý
- **Không dùng ảnh người dùng để huấn luyện** bất kỳ mô hình nào
- Không có tài khoản người dùng, không thu thập thông tin định danh

## Mô hình và giấy phép

Tất cả đều là checkpoint công khai, không train lại. Phiên bản cụ thể và sha256 sẽ được chốt ở giai đoạn 1 và ghi vào `scripts/download_models.py` để kiểm tra khi tải.

| Vai trò | Model | License | Ghi chú |
| --- | --- | --- | --- |
| Ước lượng độ sâu | Depth Anything V2 **Small** (24.8M) | Apache-2.0 | **Chỉ dùng bản Small** |
| Ảnh đơn → mesh kín | TripoSR | MIT (cả mã lẫn trọng số) | Chạy ở resolution 128 |
| Tách nền | rembg (U²-Net) hoặc BiRefNet-lite | MIT | Bắt buộc trước TripoSR |
| Định tuyến, chấm chất lượng | SmolVLM-500M hoặc Qwen2-VL-2B | Apache-2.0 | Tuỳ chọn, thêm sau cùng |

> **Cảnh báo license.** Bản **Base** và **Large** của Depth Anything V2 là CC-BY-NC — không dùng được nếu công khai demo. Chỉ tải bản Small, và kiểm license trước mỗi lần thêm checkpoint mới.

Giấy phép mã nguồn của dự án: **chưa chọn** — cần quyết trước khi công khai repo hoặc demo.

## Lộ trình

Thứ tự quan trọng nhất của cả kế hoạch: **làm trọn vòng deploy bằng engine depth trước, rồi mới thêm TripoSR.** Engine depth chắc chắn chạy, nhẹ, và đủ để dựng toàn bộ hệ thống. Làm ngược lại thì sẽ mắc kẹt ở tối ưu bộ nhớ nhiều tuần và không bao giờ chạm tới phần hạ tầng.

Mỗi giai đoạn kết thúc bằng một PR được merge vào `develop`, không phải bằng cảm giác "xong rồi".

- [ ] **0 — Git và quy tắc làm việc.** Cây thư mục, pyproject, pre-commit, CI lint, branch protection
- [ ] **1 — Bộ ảnh thử và tiêu chí chấm.** 30 ảnh cảnh + 30 ảnh vật thể, hàm chấm trong `quality.py`
- [ ] **2 — depth → mesh.** Unproject, lọc điểm biên, Poisson, giảm mặt, xuất `.glb`
- [ ] **3 — ONNX, latency và RAM.** Cố định số thread, warm-up, đo RAM đỉnh để lấy `mem_limit`
- [ ] **4 — API và worker.** FastAPI + RQ, SSE tiến độ, idempotency, timeout job
- [ ] **5 — Docker.** Build ba tầng, non-root, `mem_limit`, và cố tình gây OOM một lần
- [ ] **6 — CI/CD.** lint + test + build + security, release tự động lên GHCR
- [ ] **7 — Monitoring, load test, engine thứ hai.** Grafana tách theo engine, Locust, rồi thêm TripoSR

Quy tắc tự bảo vệ: **nếu hết tuần thứ ba mà chưa có `docker compose up` chạy được, hãy cắt tính năng chứ đừng kéo dài thời gian.** Một engine được triển khai đầy đủ có giá trị hơn ba engine nằm trong notebook.

Tài liệu thiết kế đầy đủ: `Ảnh sang 3D — tài liệu dự án.md`.
