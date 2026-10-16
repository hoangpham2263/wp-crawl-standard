# WP Crawl Standard — quy chuẩn dựng tool crawl → WordPress / WooCommerce

Tài liệu này mô tả khung chung mà mọi tool trong thư mục `crawls/` đang dùng (đúc kết từ wp-post-chatuy-crawl, wp-product-crawler, haravan-product-crawl, các tool dethitracnghiem…), kèm những chỗ cần làm chuẩn hơn. Khi cần tool mới: **đưa file này cho AI/dev đọc, mô tả nguồn dữ liệu + đích, rồi dựng theo đúng khung dưới đây.**

---

## Quy tắc bắt buộc (tóm tắt)

1. **Mỗi bước một file** `fN_<việc>.py`, có hàm `run() -> bool`; `main.py` chỉ là menu gọi các bước.
2. **Bước sau đọc output của bước trước từ `data/`**, không gọi thẳng code của nhau.
3. **Chạy lại không làm hỏng / không tạo trùng**: bỏ qua item đã xong; chống trùng bằng `slug` hoặc `sku`.
4. **Cấu hình nằm trong `config.json`** (nguồn, selector, WordPress, tốc độ crawl); không viết cứng trong code.
5. **Không commit mật khẩu và dữ liệu**: `config.json`, `data/`, file CSV/Excel nằm trong `.gitignore`; `config.example.json` để trống mật khẩu.
6. **Chạy thử trước**: `test_limit = 5`, nhập bài ở trạng thái `draft`.
7. **Crawl lịch sự**: nghỉ 1–3 giây giữa các request, tối đa 3 luồng, có retry, dừng khi bị chặn.
8. **Lỗi không làm dừng cả pipeline**: item lỗi ghi vào `data/logs/`, chạy lại riêng sau.
9. **Chỉ crawl dữ liệu có quyền dùng**; nguồn có API/feed thì dùng API.
10. **Mỗi tool có README**: nguồn, đích, cách chạy, lưu ý quyền dữ liệu.

## Quy trình dựng tool mới — từng bước

| Bước | Việc làm | Kết quả cần có |
|---|---|---|
| **1. Khảo sát nguồn** | Mở site nguồn: có sitemap? có API trong tab Network? trang render bằng JS không? Kiểm tra `robots.txt`, quyền dùng dữ liệu | Biết lấy danh sách URL từ đâu, dùng `requests` hay Playwright |
| **2. Xác định đích** | Đích là post, custom post type hay WooCommerce product? Cần chuyên mục, tag, ACF/meta nào? | Danh sách trường cần lấy (schema JSON mục 7) |
| **3. Tạo khung** | Tạo thư mục theo mục 2: `main.py`, `start`, `common.py`, `config.example.json`, `.gitignore`, `requirements.txt`, `README.md` | Chạy `./start` hiện được menu |
| **4. Viết f1 — danh sách URL** | Đọc sitemap / file txt / CSV, lọc theo `url_include` / `url_exclude` | `data/f1_urls.json` |
| **5. Viết f2 — tải HTML** | Tải từng URL theo mục 6, bỏ qua file đã có | `data/f2_html/*.html` |
| **6. Lấy selector** | Mở 2–3 trang mẫu, dùng Inspect tìm selector tiêu đề, nội dung, ảnh, chuyên mục, ngày → điền vào `config.json` | Phần `selectors` trong config |
| **7. Viết f3 — trích dữ liệu** | Đọc HTML, trích theo selector, làm sạch, chuẩn hóa | `data/f3_extracted/*.json` — mở vài file kiểm tra bằng mắt |
| **8. Viết f4 — tải ảnh** | Tải ảnh đại diện + ảnh trong nội dung, đặt tên theo slug | `data/f4_images/` + map url → file |
| **9. Chuẩn bị WordPress** | Tạo Application Password; nếu có ACF/meta/endpoint riêng thì viết `WORDPRESS_UPDATE.md` + đoạn PHP thêm vào site | Đăng nhập REST API được |
| **10. Viết f5 — nhập dữ liệu** | Theo mục 9: kiểm tra trùng, upload ảnh, thay URL ảnh, tạo chuyên mục, tạo bài | `data/f5_import_state.json` |
| **11. Chạy thử** | `test_limit = 5`, status `draft` → kiểm tra 5 bài trên WordPress (nội dung, ảnh, chuyên mục, trường tùy biến) | 5 bài nháp đúng |
| **12. Chạy thật** | `test_limit = 0`; theo dõi log; chạy lại phần lỗi | Toàn bộ dữ liệu đã nhập |
| **13. Bàn giao / lưu trữ** | Cập nhật README, xóa dữ liệu nhạy cảm nếu đưa lên Git, thu hồi Application Password nếu không dùng nữa | Repo sạch, có hướng dẫn |

---

## 1. Ý tưởng chung: pipeline nhiều bước, mỗi bước một file

```
f1 Lấy danh sách URL  →  f2 Tải HTML  →  f3 Trích dữ liệu  →  f4 Tải ảnh  →  f5 Nhập vào WordPress
   (sitemap / txt / csv)    (lưu file .html)   (lưu .json)         (lưu file ảnh)    (REST API, có resume)
```

- Mỗi bước **đọc output của bước trước từ ổ đĩa** và **ghi output của mình ra ổ đĩa** → chạy lại một bước không cần làm lại các bước trước, dễ kiểm tra dữ liệu giữa chừng.
- Bước nào không cần thì bỏ (vd. nguồn là CSV Haravan → không có f1/f2, bắt đầu từ trích dữ liệu CSV).

## 2. Cấu trúc thư mục

```
<ten-nguon>-<loai>-crawl/          # vd: chatuy-post-crawl, haravan-product-crawl
├── main.py                 # menu chọn bước, "0 = chạy tất cả"
├── start                   # #!/bin/bash → python3 main.py (chmod +x)
├── config.example.json     # mẫu cấu hình, KHÔNG chứa mật khẩu thật
├── config.json             # cấu hình thật — nằm trong .gitignore
├── .gitignore
├── requirements.txt
├── README.md               # nguồn, đích, cách chạy, lưu ý quyền dữ liệu
├── common.py               # load_config, session HTTP, logger, tiện ích dùng chung
├── f1_<ten_buoc>.py        # mỗi bước có hàm run() -> bool
├── f2_crawl_html.py
├── f3_extract_data.py
├── f4_download_images.py
├── f5_import_posts.py      # hoặc f5_import_products.py
└── data/                   # toàn bộ output — nằm trong .gitignore
    ├── f1_urls.json
    ├── f2_html/            # 1 file .html / URL
    ├── f3_extracted/       # 1 file .json / item
    ├── f4_images/
    ├── f5_import_state.json
    └── logs/
```

**Quy tắc đặt tên:** file bước `fN_<việc>.py`; output của bước N luôn có tiền tố `fN_` trong `data/`. Tên thư mục tool: `<nguồn>-<post|product|quiz>-crawl`.

## 3. Hợp đồng của mỗi bước

Mỗi file `fN_*.py` có đúng một hàm công khai:

```python
def run() -> bool:
    """Trả True nếu xong, False nếu lỗi nghiêm trọng (main dừng pipeline)."""
```

- Bước phải **chạy lại được nhiều lần** (idempotent): bỏ qua item đã có output, chỉ làm phần thiếu.
- In tiến độ ngắn gọn: `[f2] 120/540 ok, 3 lỗi`.
- Item lỗi ghi vào `data/logs/fN_errors.json` (url + lý do) để chạy lại riêng.

## 4. main.py (khung chuẩn)

```python
#!/usr/bin/env python3
"""<Tên tool> — menu chạy từng bước của pipeline crawl."""
from f1_urls import run as f1
from f2_crawl_html import run as f2
from f3_extract_data import run as f3
from f4_download_images import run as f4
from f5_import_posts import run as f5

STEPS = [("Lấy danh sách URL", f1), ("Tải HTML", f2), ("Trích dữ liệu", f3),
         ("Tải ảnh", f4), ("Nhập vào WordPress", f5)]

def main():
    while True:
        print("\n0. Chạy tất cả")
        for i, (name, _) in enumerate(STEPS, 1):
            print(f"{i}. {name}")
        print("9. Thoát")
        choice = input("Chọn: ").strip()
        if choice == "9":
            break
        if choice == "0":
            for name, step in STEPS:
                print(f"\n--- {name} ---")
                if not step():
                    print(f"Dừng ở bước: {name}")
                    break
        elif choice.isdigit() and 1 <= int(choice) <= len(STEPS):
            STEPS[int(choice) - 1][1]()

if __name__ == "__main__":
    try:
        main()
    except KeyboardInterrupt:
        print("\nĐã dừng.")
```

## 5. Cấu hình

`config.example.json` (commit) — `config.json` (không commit):

```json
{
  "source": {
    "sitemap_url": "https://example.com/sitemap.xml",
    "url_include": ["/bai-viet/"],
    "url_exclude": ["/tag/", "/page/"]
  },
  "selectors": {
    "title": "h1",
    "content": "article .entry-content",
    "image": "meta[property='og:image']",
    "category": ".breadcrumb a",
    "date": "time[datetime]"
  },
  "crawl": { "delay_min": 1, "delay_max": 3, "max_retries": 3, "workers": 3, "user_agent": "" },
  "wordpress": {
    "url": "https://your-site.com",
    "username": "",
    "app_password": "",
    "post_type": "posts",
    "status": "draft"
  },
  "test_limit": 5
}
```

- **Selector để trong config, không viết cứng trong code** → đổi nguồn chỉ cần sửa config.
- Cho phép đọc mật khẩu từ biến môi trường để không phải ghi vào file: `WP_APP_PASSWORD` ưu tiên hơn `config.json`.
- `test_limit` > 0: mỗi bước chỉ xử lý N item đầu để chạy thử; 0 = chạy hết.
- Mặc định nhập bài ở trạng thái **draft** để kiểm tra trước khi đăng.

## 6. Tải trang lịch sự (f2)

- Một `requests.Session` dùng chung; `User-Agent` rõ ràng; timeout 20–30s.
- Nghỉ ngẫu nhiên `delay_min`–`delay_max` giây giữa các request; `workers` ≤ 3.
- Retry có backoff khi lỗi mạng / 429 / 5xx; **dừng hẳn** nếu bị chặn liên tục.
- Phát hiện encoding (chardet) trước khi lưu, lưu UTF-8.
- Trang render bằng JavaScript: ưu tiên tìm API JSON trong tab Network; chỉ dùng Playwright khi bắt buộc.

## 7. Trích dữ liệu (f3)

- Mỗi item ra 1 file JSON với schema thống nhất:

```json
{
  "source_url": "",
  "slug": "",
  "title": "",
  "excerpt": "",
  "content_html": "",
  "categories": ["Cha", "Con"],
  "tags": [],
  "featured_image": "",
  "images": [],
  "published_at": "",
  "meta": {}
}
```

- Sản phẩm thêm: `sku`, `price`, `sale_price`, `stock`, `attributes`, `variations`.
- **Làm sạch nội dung:** bỏ `script/style/iframe` lạ, bỏ link về site nguồn, bỏ thuộc tính inline thừa, giữ thẻ HTML cơ bản.
- Chuẩn hóa: giá → số nguyên, ngày → ISO 8601, khoảng trắng, slug không dấu.

## 8. Tải ảnh (f4)

- Tên file theo slug item + số thứ tự; bỏ qua ảnh đã tải; ghi lại map `url gốc → file`.
- Có thể nén/đổi WebP trước khi upload để nhẹ media library.

## 9. Nhập vào WordPress (f5)

- Xác thực **Application Password** (Basic auth) qua REST API `/wp-json/wp/v2/...`; WooCommerce dùng `/wp-json/wc/v3/products` với consumer key/secret.
- **Chống trùng bằng khóa tự nhiên**, không dựa vào số thứ tự: tìm theo `slug` (bài viết) hoặc `sku` (sản phẩm) trước khi tạo; có rồi thì cập nhật hoặc bỏ qua.
- **Resume bằng danh sách đã xong**: `data/f5_import_state.json` lưu `{ "<slug>": <wp_id> }` — an toàn hơn lưu "đang ở item thứ N".
- Upload ảnh → thay URL ảnh trong `content_html` bằng URL media mới → tạo bài/sản phẩm.
- Chuyên mục: cache `tên → id`, tạo phân cấp cha/con nếu chưa có.
- Trường tùy biến (ACF / post meta / endpoint riêng): mô tả trong `WORDPRESS_UPDATE.md` kèm đoạn PHP đăng ký meta / route cần thêm vào site.
- Giới hạn song song (semaphore ~3) để không làm quá tải host.

## 10. .gitignore bắt buộc

```
config.json
.env
data/
venv/
__pycache__/
*.pyc
*.csv
*.xlsx
*.zip
.DS_Store
```

## 11. Bảo mật & quyền dữ liệu (đọc trước khi crawl)

- **Không bao giờ** commit `config.json`, mật khẩu ứng dụng, consumer key; `config.example.json` chỉ để chuỗi rỗng. Nếu lỡ commit mật khẩu: thu hồi Application Password đó trong WordPress ngay.
- Chỉ crawl dữ liệu **có quyền dùng**: site cũ của chính khách hàng, dữ liệu xuất (CSV) của chủ shop, nguồn có giấy phép. Nguồn có API/feed thì dùng API.
- Tôn trọng `robots.txt` và điều khoản của site nguồn; không crawl dữ liệu cá nhân.
- Repo công khai: chỉ đưa **khung tool + config mẫu**, không kèm dữ liệu khách, không ghi tên khách hàng nếu chưa được phép.

## 12. README mỗi tool (mẫu ngắn)

```md
# <Tên tool>
Nguồn: <site/CSV>  →  Đích: <WordPress post | WooCommerce product>

## Chạy
pip install -r requirements.txt
cp config.example.json config.json   # điền URL, user, app password, selector
./start                               # chọn 0 để chạy tất cả, hoặc từng bước

## Các bước
f1 … f5 (mỗi bước làm gì, output ở đâu)

## Lưu ý
Chỉ dùng cho dữ liệu bạn có quyền sử dụng.
```

## 13. Checklist khi dựng tool mới

- [ ] Đúng cấu trúc thư mục + tên file `fN_*.py`, mỗi bước có `run() -> bool`
- [ ] Selector và nguồn nằm trong config; có `config.example.json` không chứa mật khẩu
- [ ] `.gitignore` đủ (config.json, data/, venv/…)
- [ ] `test_limit` chạy thử 5 item, import ở trạng thái draft
- [ ] Chạy lại không tạo trùng (kiểm tra slug/SKU), resume bằng state file
- [ ] Item lỗi ghi log riêng, có cách chạy lại phần lỗi
- [ ] README ghi nguồn, đích, cách chạy, lưu ý quyền dữ liệu

---

### Prompt mẫu khi cần tool mới

> Đọc `CRAWL-STANDARD.md`. Dựng tool `<ten>-crawl` theo đúng quy chuẩn: nguồn là `<sitemap / file CSV / danh sách URL>`, lấy các trường `<…>`, nhập vào `<WordPress post type / WooCommerce>` với chuyên mục `<…>`. Selector: `<…>`. Chạy thử `test_limit = 5` trước.
