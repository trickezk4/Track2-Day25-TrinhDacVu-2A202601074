### 1. Baseline vs. Optimized: Hiệu quả Chi phí & Đơn vị `$/1M-token`

Đo lường chi phí AI theo **`$/1M-token` (Unit Economics)** phản ánh chính xác hiệu suất sinh token thay vì chỉ nhìn vào chi phí thuê máy thô `$/GPU-giờ`.

| Hạng mục                               |          Baseline          |         Optimized         | Mức tiết kiệm | Ý nghĩa Vận hành & FinOps                    |
| ---------------------------------------- | :------------------------: | :------------------------: | :--------------: | ------------------------------------------------ |
| **Inference (2,400 reqs/ngày)**   |       $48.87 / ngày       |       $8.48 / ngày       | **-82.6%** | Tối ưu hóa nhờ Cascade, Cache và Batch.     |
| **Chi phí $/1M-Token**            |      **$6.488**      |      **$1.126**      | **-82.6%** | Đơn giá giảm sâu trên 7.53M token/ngày.   |
| **Workloads Mua GPU (M3)**         |      $25,667 / tháng      |      $15,627 / tháng      | **-39.1%** | Phân bổ Spot + Checkpoint và Reserved.        |
| **Tổng Chi phí Vận hành (M5)** | **$27,133** / tháng | **$14,626** / tháng | **-46.1%** | Vượt mục tiêu cắt giảm ban đầu (>= 40%). |

---

### 2. Phân tích 4 Đòn bẩy Chi phí

* **Đòn bẩy Inference (-82.6% — *Đóng góp lớn nhất*):** Đạt ROI cao nhất nhờ nguyên lý **Discount Stacking (Nhân dồn chiết khấu)**. Định tuyến Cascade chuyển 70% request sang model nhỏ (rẻ hơn 15x), kết hợp Prompt Caching (giảm 90% input lặp lại) và Batch API (giảm 50% non-realtime). Khi kết hợp Batch và 100% Cache, chi phí chỉ còn $0.50 \times 0.10 = \mathbf{0.05}$ (**giảm 95%**).
* **Đòn bẩy Purchasing (-39.1%):** Dùng Spot cho tác vụ gián đoạn được (`interruptible=1`); dùng Reserved cho dịch vụ inference có duty cycle >= 55% (điểm hòa vốn).
* **Khắc phục GPU-Util Lie (Right-sizing):** Hạ cấp GPU nghẽn I/O sang GPU tối ưu băng thông chi phí thấp.
* **Thu hồi GPU Idle:** Tắt các instance utilization < 10%, tránh thất thoát $20.00/ngày.

---

### 3. Hiện tượng "GPU-Util Lie" & Tác động Tài chính

* **Bản chất kỹ thuật:** Lệnh `nvidia-smi` chỉ đo thời gian xung nhịp hoạt động, không phản ánh khối lượng tính toán hữu ích (FLOPs). GPU có thể đạt 98% GPU-Util nhưng MFU chỉ đạt ~20% do nghẽn bộ nhớ (memory stall) hoặc chờ nạp I/O.
* **Phát hiện:** **`gpu-h100-4`** (Util 98.0%, MFU 20.2%, MBU 45.0%) và **`gpu-a10g-1`**.
* **Tác động tài chính:** NimbusAI trả **$2.50/giờ** cho H100 nhưng chỉ nhận được 1/5 năng lực thực tế. Với LLM Decode (memory-bound, arithmetic intensity chỉ 1–2 FLOP/byte < 295 FLOP/byte của H100), chuyển sang L4/A10G giúp giữ nguyên throughput và giảm >60% chi phí thuê GPU.

---

### 4. Báo cáo Thực nghiệm 2 Extensions ("Your Turn")

**Extension 1: Cải tiến `recommend_tier` theo Rủi ro & Vòng đời Job**

* **Mô tả:** Bổ sung `gpu_type` (xác suất thu hồi phần cứng) và `job_days` (thời lượng tác vụ).
* **Kết quả đo lường:**
  * `train-a10g` (A10G, 20h/ngày, 10 ngày): Đổi từ `spot` -> `on_demand` do A10G có tỷ lệ thu hồi cao (>15%), tránh lãng phí checkpoint rework.
  * `train-h100` (H100, 20h/ngày, 10 ngày): Giữ `spot` vì H100 ít bị ngắt quãng (<5%), tiết kiệm an toàn 40.9%.
  * `short-eval` (A100, 24h/ngày, 7 ngày): Đổi từ `reserved` -> `on_demand` để tránh cam kết 1–3 năm cho job 7 ngày.
* **Insight:** Mua phần cứng tối ưu phải kết hợp giữa điểm hòa vốn lý thuyết và độ ổn định thực tế của từng dòng GPU.

**Extension 3: Đánh giá Điểm Hòa Vốn Caching (`cache_is_worth_it`) (AI Support)**

* **Toán học:** Với giảm giá đọc 90% ($P_{read} = 0.10 \times P_{write}$), điểm hòa vốn là:

  $$
  N_{reads} \times (0.90 \times P_{write}) > P_{write} \iff N_{reads} > \frac{1}{1 - 0.10} \approx \mathbf{1.111 \text{ lượt đọc}}
  $$
* **Kết quả đo lường:** $N_{reads} \le 1.0 \rightarrow$ `False`; $N_{reads} \ge 1.12 \rightarrow$ `True` (bắt đầu sinh lời).

  ```
  === ĐO LƯỜNG EXTENSION 3: BREAK-EVEN CACHE ===

  Model: Small Model (Claude Instant/Haiku)

  - Avg Reads: 0.5  -> Có lợi tài chính? False
  - Avg Reads: 1.0  -> Có lợi tài chính? False
  - Avg Reads: 1.12 -> Có lợi tài chính? True
  - Avg Reads: 2.0  -> Có lợi tài chính? True
  - Avg Reads: 5.0  -> Có lợi tài chính? True

  Model: Large Model (Claude Opus/GPT-4)

  - Avg Reads: 0.5  -> Có lợi tài chính? False
  - Avg Reads: 1.0  -> Có lợi tài chính? False
  - Avg Reads: 1.12 -> Có lợi tài chính? True
  - Avg Reads: 2.0  -> Có lợi tài chính? True
  - Avg Reads: 5.0  -> Có lợi tài chính? True
  ```
* **Insight:** Chỉ bật caching cho System Prompt/RAG dùng chung lặp lại nhiều lần; không bật cho prompt dùng 1 lần.

---

### 5. Ba Khuyến nghị Chiến lược Cho Ban Lãnh đạo NimbusAI

1. **Mở Cổng Chargeback & Chuẩn hóa FOCUS:** Tag coverage đạt **92%** (vượt chuẩn 80%). Chuyển ngay từ *Showback* sang *Chargeback* để trừ chi phí trực tiếp vào ngân sách từng team (`ml-research`, `platform`, `product`) theo chuẩn FOCUS.
2. **Triển khai AI Gateway Trung tâm:** Bắt buộc toàn bộ request đi qua Gateway tự động: Router phân tầng Cascade, Cache tập trung và gom Batch API để giữ mức giảm 82.6% chi phí inference.
3. **Điều phối Tác vụ Theo Carbon (Carbon-Aware Scheduling):** Di chuyển các batch training gián đoạn sang vùng **`europe-north1`** (Na Uy — dùng 100% thủy điện, giá rẻ nhất và carbon thấp nhất), giảm hơn 70% lượng phát thải gCO2e.
