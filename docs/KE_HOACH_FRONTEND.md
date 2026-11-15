# Kế hoạch frontend: Dashboard Vite + React đọc JSON

Mục tiêu: một dashboard web tĩnh (không cần backend) hiển thị toàn bộ KPI của lớp `mart`, chạy được bằng `npm run dev` và build thành file tĩnh để nộp hoặc deploy. Dashboard trả lời cùng câu hỏi với Power BI: *tháng tới cần bao nhiêu xe, ở hãng/vùng/đại lý nào, và dự báo đáng tin đến mức nào.*

## 1. Kiến trúc

```
PostgreSQL (schema mart)
   │  python src/export_json.py         (bước mới "json" trong run_pipeline)
   ▼
frontend/public/data/*.json            (1 file / bảng mart + manifest.json)
   │  fetch() lúc chạy, cache trong bộ nhớ
   ▼
Vite + React + TypeScript  →  npm run build  →  frontend/dist/ (HTML/JS tĩnh)
```

Nguyên tắc:
- **Không tính lại KPI ở frontend.** Mọi KPI đã có trong `mart.*`; frontend chỉ lọc, gộp và vẽ. Khi gộp nhiều dòng thì tính lại tỷ lệ từ cột thành phần (`units`/`units_py`, `abs_error_total`/`actual_total`), không lấy trung bình phần trăm (quy ước số 1 trong `CLAUDE.md`).
- **JSON là hợp đồng duy nhất** giữa Python và React. Đổi cột trong `kpi_marts.sql` thì cập nhật `export_json.py` và `src/types/mart.ts` cùng lúc.
- Dữ liệu tĩnh 2022–2023 + dự báo 01–06/2024; không có "hôm nay".

## 2. Xuất JSON: `src/export_json.py`

- Đọc `DATABASE_URL` qua `env_config.get_database_url()`.
- Mỗi bảng mart → `frontend/public/data/<tên_bảng>.json`, dạng **mảng object** (dễ đọc, dễ debug):
  ```json
  [{"month_start": "2023-12-01", "units": 1675, "revenue": 47235200, "units_yoy_pct": 0.1822, ...}]
  ```
- Quy ước kiểu: ngày → chuỗi `YYYY-MM-DD`; `NUMERIC` → số (làm tròn 6 chữ số thập phân); `NULL` → `null`; boolean giữ nguyên.
- `manifest.json`: `generated_at`, `main_model`, các ngưỡng từ `kpi_parameters`, và số dòng + kích thước từng file để frontend kiểm tra dữ liệu đủ.
- Bảng lớn nhất `kpi_monthly_by_dimension` (~5.700 dòng, ~1 MB): tách thành `kpi_monthly_by_dimension/<dimension_name>.json` (10 file nhỏ), trang chỉ tải chiều đang chọn.
- Thêm bước `json` vào `run_pipeline.py` sau `marts`: `--only marts json` để làm mới dữ liệu dashboard.
- **Commit cả `frontend/public/data/`** để người khác chạy dashboard mà không cần PostgreSQL (dữ liệu tĩnh, nhỏ).

## 3. Công nghệ

| Hạng mục | Chọn | Lý do |
| --- | --- | --- |
| Build | Vite 5 + React 18 + TypeScript | Theo yêu cầu; TS bắt lỗi sai tên cột |
| Biểu đồ | Recharts | Đủ line/bar/area/composed; vẽ dải dự báo 80% bằng `Area` với `[lower, upper]` |
| Định tuyến | React Router, `HashRouter` | Build tĩnh mở được trực tiếp, deploy GitHub Pages không cần cấu hình server |
| Trạng thái lọc | React Context + URL query (`?year=2023&dim=company`) | Đủ cho 5 trang; chia sẻ link giữ nguyên bộ lọc |
| Style | CSS Modules + biến CSS (sáng/tối) | Không thêm thư viện; dễ đồng bộ màu với Power BI |
| Kiểm tra dữ liệu | Zod (schema runtime) + Vitest | Báo lỗi rõ khi JSON thiếu cột; test hàm gộp |
| Định dạng số | `Intl.NumberFormat('vi-VN')` | `1.675 xe`, `47,2 tr USD`, `18,2%` |

## 4. Cấu trúc thư mục

```
frontend/
  package.json, vite.config.ts, tsconfig.json, index.html
  public/data/                  # do export_json.py sinh, có commit
  src/
    main.tsx, App.tsx           # HashRouter, layout, thanh điều hướng 5 trang
    types/mart.ts               # kiểu TS cho từng bảng mart (khớp MART_TABLES)
    data/
      load.ts                   # fetch + cache + kiểm tra Zod
      schemas.ts                # Zod schema
      useMart.ts                # hook: useMart('kpi_monthly_overview')
    lib/
      aggregate.ts              # sumBy, ratio, yoy(), wape(), share()
      format.ts                 # số, %, tiền, tháng
      whatif.ts                 # recommendedOrder(), projectedRevenue()
    state/filters.tsx           # năm, chiều, thành viên, cấp/mô hình dự báo, what-if
    components/
      KpiCard.tsx, StatusBadge.tsx, Section.tsx, DataTable.tsx
      charts/ LineTrend.tsx, BarRank.tsx, ForecastBand.tsx, Heatmap.tsx, DonutShare.tsx
      filters/ YearSelect.tsx, DimensionSelect.tsx, ForecastFilters.tsx, WhatIfSliders.tsx
    pages/
      Overview.tsx, Dimensions.tsx, Forecast.tsx, Decision.tsx, Quality.tsx
    styles/ tokens.css, global.css
  tests/ aggregate.test.ts, whatif.test.ts, data.test.ts
```

## 5. Các trang dashboard

Bộ lọc chung ở thanh trên: **Năm** (2022 / 2023 / Tất cả). Mỗi trang ghi rõ nguồn bảng mart.

### 5.1 Tổng quan (`/`)
| Thành phần | Dữ liệu | KPI |
| --- | --- | --- |
| 6 thẻ KPI: Sản lượng, Doanh thu, ASP, YoY sản lượng, YoY doanh thu, % so mục tiêu (xanh/đỏ) | `kpi_annual_summary`, `kpi_monthly_overview` | D1–D4, mục tiêu +5% |
| Line sản lượng theo tháng 2022 vs 2023 + đường mục tiêu | `kpi_monthly_overview` | D1, D4 |
| Column doanh thu theo quý, nhãn QoQ | `kpi_quarterly` | D4 |
| Heatmap tháng × năm theo chỉ số mùa vụ | `kpi_monthly_overview.seasonality_index` | D7 |
| Bar chỉ số ngày trong tuần | `kpi_weekday` | D8 |
| Dòng chú thích: tháng đỉnh/đáy, Top 10 mẫu, HHI | `kpi_annual_summary` | D10 |

### 5.2 Phân tích theo chiều (`/dimensions`)
| Thành phần | Dữ liệu | KPI |
| --- | --- | --- |
| Chọn chiều: Hãng, Mẫu xe, Hộp số, Động cơ, Kiểu thân, Màu, Vùng, Đại lý, Giới tính, Nhóm thu nhập | `dim_member` | — |
| Bar Top N theo sản lượng, kèm tỷ trọng | `kpi_dimension_summary` | D5, D6, D12–D17 |
| Scatter tỷ trọng × YoY (bong bóng = doanh thu) | `kpi_dimension_summary` | D12–D17 |
| Line theo tháng của các thành viên đã chọn (tối đa 5) | `kpi_monthly_by_dimension/<dim>.json` | D12–D17 |
| Bảng xếp hạng: hạng, sản lượng, doanh thu, ASP, YoY, Δ tỷ trọng, CV, nhóm "Top 5 tăng/giảm" | `kpi_dimension_summary` | D9, D13, D16 |
| Phân khúc khách hàng: bar nhóm thu nhập × giới tính, giá/thu nhập | `kpi_customer_segment` | D11, D17 |

### 5.3 Dự báo (`/forecast`)
| Thành phần | Dữ liệu | KPI |
| --- | --- | --- |
| Bộ lọc: Cấp (Tổng/Hãng/Vùng) → Thành viên; Mô hình (mặc định mô hình chính) | `dim_forecast_member`, `dim_forecast_model` | — |
| Biểu đồ chính: thực tế 24 tháng → dự báo 6 tháng, dải 80%, vùng backtest tô nền | `forecast_series` | F1, F8 |
| 4 thẻ: WAPE, Accuracy, Bias, Độ phủ khoảng, tô màu theo ngưỡng | `forecast_accuracy` | F2–F8 |
| Bảng so sánh 3 mô hình (WAPE, MAE, RMSE, Bias, Tracking signal, FVA, Đạt/Không đạt) | `forecast_accuracy` | F2–F7 |
| Bar WAPE theo hãng (mô hình chính), vạch ngưỡng 20% | `forecast_accuracy` | F3 |

### 5.4 Hỗ trợ quyết định (`/decision`)
| Thành phần | Dữ liệu | KPI |
| --- | --- | --- |
| Thanh trượt What-if: thay đổi nhu cầu −30%…+30%, hệ số đệm k 0–2 | tính ở `lib/whatif.ts` | S6 |
| Thẻ: dự báo 6 tháng, đề xuất nhập, doanh thu dự kiến, tăng trưởng dự báo | `dss_forecast_plan` | S1, S3, S4 |
| Bar đề xuất nhập theo hãng (dự báo vs đề xuất có đệm) | `dss_forecast_plan` (level = company) | S3 |
| Bảng phân bổ theo vùng → đại lý (mở rộng được), chọn tháng | `dss_dealer_allocation` | S2 |
| Bảng xếp hạng ưu tiên hãng: điểm 4 thành phần (stacked bar), hạng, nhóm ưu tiên | `dss_company_priority` | S5 |

Công thức What-if (giống measure DAX):
`đề xuất nhập = forecast × (1 + Δ) × (1 + k × demand_cv)`, `doanh thu dự kiến = forecast × (1 + Δ) × asp_hist`, `phân bổ = allocated_units × (1 + Δ)`.

### 5.5 Chất lượng dữ liệu (`/quality`)
Bảng Q1–Q6 từ `dq_checks` (giá trị, ngưỡng, Đạt/Không đạt), thẻ tỷ lệ đạt, và `manifest.json`: thời điểm xuất, số dòng từng bảng, tham số đang dùng.

## 6. Giao diện

- Lưới 12 cột, tối thiểu 1280 px cho máy chiếu; xuống 1 cột trên màn hình hẹp.
- Màu: xanh `#2E7D32` = đạt, đỏ `#C62828` = không đạt (giống Power BI); một màu nhấn cho thực tế, một màu cho dự báo, dải 80% cùng màu dự báo độ mờ 20%.
- Mọi biểu đồ có tooltip giá trị đầy đủ và tiêu đề nói kết luận (ví dụ "Sản lượng 2023 tăng 24,6% so với 2022").
- Trạng thái tải và lỗi: khung xám khi đang tải; khi thiếu file JSON, báo rõ "Chạy `python src/run_pipeline.py --only marts json`".

## 7. Các bước triển khai

| # | Việc | Kết quả kiểm tra | Ước lượng |
| --- | --- | --- | --- |
| 1 | `src/export_json.py` + bước `json` trong `run_pipeline.py` | `frontend/public/data/` có đủ file, `manifest.json` khớp số dòng trong PostgreSQL | 1–2 giờ |
| 2 | Khởi tạo `npm create vite@latest frontend -- --template react-ts`, cài `react-router-dom recharts zod`, `vitest` | `npm run dev` chạy, có layout + 5 trang trống | 1 giờ |
| 3 | `types/mart.ts`, `schemas.ts`, `load.ts`, `useMart`, `lib/format.ts`, `lib/aggregate.ts` + test | `npm test` qua; tải được mọi file JSON không lỗi Zod | 2 giờ |
| 4 | Trang Tổng quan | Số trên thẻ khớp `kpi_output.txt` (sản lượng 2023 = 13.261, YoY 24,57%) | 2–3 giờ |
| 5 | Trang Phân tích theo chiều | Đổi chiều cập nhật mọi biểu đồ; tỷ trọng cộng đủ 100% | 3 giờ |
| 6 | Trang Dự báo | Đường thực tế nối liền dự báo; WAPE cấp tổng khớp `mart.forecast_accuracy` | 3 giờ |
| 7 | Trang Hỗ trợ quyết định + What-if | Δ = 0, k = giá trị trong `kpi_parameters` cho ra đúng `recommended_order` trong SQL | 3 giờ |
| 8 | Trang Chất lượng dữ liệu | Hiện đủ Q1–Q6 | 1 giờ |
| 9 | Hoàn thiện: responsive, sáng/tối, xử lý lỗi, `npm run build` | `dist/` mở được bằng `npx vite preview` | 2 giờ |
| 10 | Kiểm thử đối chiếu với Power BI + cập nhật `CLAUDE.md`, hướng dẫn | Bảng đối chiếu 10 số chính giữa web, Power BI, SQL | 1–2 giờ |

Tổng: khoảng 20 giờ làm việc. Nếu thiếu thời gian, ưu tiên theo thứ tự: bước 1–4, 6, 7 (Tổng quan, Dự báo, What-if là phần chấm điểm DSS); trang 5.2 và 5.5 làm sau.

## 8. Lệnh chạy (sau khi hoàn thành)

```powershell
python src/run_pipeline.py --only marts json   # làm mới dữ liệu dashboard
cd frontend
npm install
npm run dev        # http://localhost:5173
npm run build      # tạo frontend/dist để nộp hoặc deploy
npm test
```

## 9. Rủi ro và cách xử lý

| Rủi ro | Cách xử lý |
| --- | --- |
| Đổi cột trong `kpi_marts.sql` làm vỡ dashboard | Zod báo lỗi rõ tên bảng, cột; test `data.test.ts` chạy trên JSON đã commit |
| Số trên web lệch Power BI | Chỉ dùng hàm trong `lib/aggregate.ts` để gộp; bước 10 đối chiếu từng số |
| JSON lớn làm chậm | Tách `kpi_monthly_by_dimension` theo chiều, tải theo trang (lazy) |
| Recharts khó vẽ dải dự báo | `ComposedChart` + `Area dataKey="band"` với giá trị `[lower_80, upper_80]` |
| Người dùng không có PostgreSQL | JSON đã commit trong `frontend/public/data/` |
