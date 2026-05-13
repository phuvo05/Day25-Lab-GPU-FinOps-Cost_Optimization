# GPU FinOps & Cost Optimization - Lab Report

**Họ và Tên:** Võ Thiên Phú
**MSSV:** 2A202600336
**Ngày nộp:** 13/05/2026

---

## 1. Giới Thiệu

### 1.1 Mục tiêu bài lab

Bài lab GPU FinOps & Cost Optimization nhằm mục tiêu giúp sinh viên nắm vững các khái niệm và kỹ năng thực hành về quản lý chi phí GPU (GPU FinOps) trong môi trường cloud và data center. Cụ thể, bài lab tập trung vào:

- Giám sát trạng thái cluster GPU theo thời gian thực
- Theo dõi và phân bổ chi phí cho các workload GPU
- Quản lý Spot Instance để tối ưu chi phí
- Tự động scale cluster dựa trên ngưỡng utilization (KEDA-like)
- Phân tích chi phí, phát hiện waste và đưa ra khuyến nghị tối ưu
- So sánh training FP32 vs Mixed Precision (AMP) trên GPU thực

### 1.2 Tổng quan về GPU FinOps

FinOps (Financial Operations) là phương pháp quản lý chi phí cloud, giúp các tổ chức tối ưu hóa việc sử dụng và chi phí cloud computing. GPU FinOps đặc biệt quan trọng trong era AI/ML khi chi phí GPU chiếm phần lớn trong tổng chi phí vận hành. Các công cụ và kỹ thuật FinOps giúp:

- Theo dõi chi phí GPU theo thời gian thực
- Xác định và loại bỏ lãng phí tài nguyên
- Tối ưu hóa việc sử dụng Spot/Preemptible instances
- Đưa ra quyết định dựa trên dữ liệu về loại GPU, thời gian sử dụng

---

## 2. Phân Tích Từng Phần

### Part 1: GPU Cluster Monitoring

#### Môi trường thí nghiệm

Hệ thống mock cluster gồm **8 nodes** với **16 GPUs** phân bố như sau:

| Node | GPU 0 | GPU 1 |
|------|-------|-------|
| node-00 | T4 (61.4%, 10.5/16GB) | T4 (87.7%, 13.9/16GB) |
| node-01 | A100 (89.8%, 65.1/80GB) | A100 (84.3%, 40.2/80GB) |
| node-02 | V100 (85.6%, 19.4/32GB) | V100 (9.8%, 0.6/32GB) |
| node-03 | T4 (88.9%, 13.4/16GB) | T4 (73.8%, 13.6/16GB) |
| node-04 | T4 (82.1%, 10.4/16GB) | T4 (83.3%, 12.5/16GB) |
| node-05 | T4 (86.3%, 8.5/16GB) | T4 (67.6%, 12.5/16GB) |
| node-06 | T4 (7.1%, 1.6/16GB) | T4 (4.3%, 1.9/16GB) |
| node-07 | T4 (0.0%, 0.5/16GB) | T4 (0.0%, 0.5/16GB) |

#### Metrics tổng hợp

| Metric | Giá trị |
|--------|---------|
| Total GPUs | 16 |
| Busy GPUs | 11 |
| Idle GPUs | 5 |
| Avg Utilization | 57.0% |
| Memory Used | 225.0 GB / 416.0 GB |
| Total Power Draw | 1136 W |
| Node Count | 8 |

#### Phân tích insights

- **Phân bố GPU types:** Cluster sử dụng đa dạng các loại GPU từ T4 (phổ biến nhất), A100 (high-end), đến V100. Điều này phản ánh thực tế data center cần nhiều loại GPU cho các workload khác nhau.
- **Node-06 và node-07 idle:** Hai node cuối hoàn toàn idle (utilization < 10%), đây là cơ hội tối ưu hóa để assign workload hoặc shutdown để tiết kiệm chi phí.
- **GPU types có utilization thấp:** node-02 GPU 1 chỉ 9.8% utilization - dư thừa tài nguyên cho workload hiện tại.
- **High-end GPU cho workload nặng:** A100 trên node-01 đang chạy ổn định với utilization trên 80%, phù hợp cho các training job nặng.
- **Power efficiency:** Tổng công suất 1136W cho 16 GPUs cho thấy hệ thống hoạt động ở mức công suất trung bình.

---

### Part 2: Workload Submission & Cost Tracking

#### Các workload đã submit

| Workload ID | GPU Type | GPUs | Duration | Spot |
|-------------|----------|------|----------|------|
| train-resnet-001 | T4 | 1 | 300s | No |
| train-bert-002 | A100 | 1 | 600s | No |
| inference-api-003 | T4 | 1 | 120s | Yes |
| train-llm-004 | A100 | 2 | 900s | Yes |

#### Chi phí ghi nhận

| Workload | Loại | Chi phí | Tiết kiệm |
|----------|------|---------|-----------|
| train-resnet-001 | ON-DEMAND | $0.0292 | $0.0000 |
| train-bert-002 | ON-DEMAND | $0.6117 | $0.0000 |
| inference-api-003 | SPOT | $0.0035 | $0.0082 |
| train-llm-004 | SPOT | $0.5505 | $1.2845 |

#### Billing Summary

| Metric | Giá trị |
|--------|---------|
| Total Cost | $5.2869 |
| Total Savings | $5.5380 |
| Budget Used | 5.3% |
| Alert Status | OK |

#### Phân tích insights

- **Tiết kiệm đáng kể từ Spot:** 2 workload sử dụng Spot instance (inference-api-003 và train-llm-004) đã tiết kiệm tổng cộng $1.2927 so với on-demand. Đặc biệt train-llm-004 tiết kiệm $1.2845 nhờ sử dụng 2 A100 Spot.
- **Chi phí tỷ lệ với workload:** Workload dài (train-llm-004 - 900s) có chi phí cao nhất dù là Spot, cho thấy duration là yếu tố quyết định chi phí.
- **Inference rẻ hơn training:** inference-api-003 chỉ tốn $0.0035 trong 120s, trong khi training jobs tốn nhiều hơn đáng kể.
- **Budget an toàn:** Chỉ sử dụng 5.3% budget, hệ thống còn dư budget lớn cho các hoạt động tiếp theo.

---

### Part 3: Spot Instance Management

#### Bảng giá Spot vs On-Demand

| GPU Type | On-Demand ($/hr) | Spot Price ($/hr) | Discount | Availability |
|----------|-----------------|-------------------|----------|--------------|
| T4 | $0.35 | $0.2371 | 32.3% | high |
| A100 | $3.67 | $2.2409 | 38.9% | high |
| V100 | $2.48 | $1.3977 | 43.6% | low |

#### Spot Request kết quả

- ✅ spot-t4-001 (T4): granted
- ✅ spot-t4-002 (T4): granted
- ✅ spot-a100-001 (A100): granted

#### Spot Preemption Simulation

| Metric | Giá trị |
|--------|---------|
| Preempted instances | 1 |
| Still active | 5 |
| Spot cost | $0.0026 |
| On-demand equivalent | $0.0087 |
| Total saved | $0.0061 (70.0%) |

#### Phân tích insights

- **Tiết kiệm 30-40% với Spot:** A100 tiết kiệm được 38.9%, V100 lên tới 43.6%, T4 tiết kiệm 32.3%. Đây là con số rất đáng kể trong production.
- **Discount tỷ lệ với giá:** GPU càng đắt, discount càng lớn. V100 discount 43.6% nhưng availability "low" - cần cân nhắc khi sử dụng.
- **70% savings khi preemption xảy ra:** Dù có preemption, tổng chi phí Spot vẫn thấp hơn đáng kể so với on-demand.
- **Chiến lược hybrid:** Nên dùng Spot cho batch jobs, fault-tolerant workloads; dùng On-Demand cho critical workloads cần guarantee.

---

### Part 4: Autoscaling (KEDA-like)

#### Autoscaling Policy

| Tham số | Giá trị |
|---------|---------|
| Scale Up Threshold | 70.0% |
| Scale Down Threshold | 25.0% |
| Cooldown Seconds | 30 |
| Max Nodes | 10 |
| Min Nodes | 2 |
| Preferred GPU Type | T4 |
| Cost Aware | True |

#### Evaluation Results

- **Initial evaluation:** Utilization 80.7% > threshold 70% → **SCALE_UP** action, nodes 8 → 9
- **5 evaluation cycles:** Sau khi scale up, utilization giảm xuống ~71.8%, các cycle tiếp theo đều **no_action** do cooldown và utilization nằm trong vùng an toàn (25% < 71.8% < 70%)

#### Phân tích insights

- **Scale-up trigger chính xác:** KEDA-like autoscaler đã nhận diện đúng khi utilization vượt 70% và trigger scale up từ 8 lên 9 nodes.
- **Stability sau scale:** Các cycle tiếp theo cho thấy hệ thống ổn định ở mức 71.8%, không cần thêm action.
- **Cooldown mechanism hoạt động:** Cooldown 30 giây giữa các evaluation ngăn chặn việc over-scaling.
- **Cost-aware scaling:** Với `cost_aware: True`, autoscaler sẽ ưu tiên T4 GPU (rẻ hơn) khi scale, tối ưu chi phí.
- **Min/Max bounds:** Giới hạn 2-10 nodes đảm bảo không scale quá thấp (availability) hay quá cao (lãng phí).

---

### Part 5: Cost Analysis & Optimization

#### Cost Snapshots (5 lần đo)

| Snapshot | Total Cost | Idle Cost | Waste % |
|----------|-----------|-----------|---------|
| 1 | $0.047778 | $0.001944 | 4.1% |
| 2 | $0.047778 | $0.001944 | 4.1% |
| 3 | $0.047778 | $0.001944 | 4.1% |
| 4 | $0.047778 | $0.001944 | 4.1% |
| 5 | $0.047778 | $0.001944 | 4.1% |

#### Waste Analysis Report

| Metric | Giá trị |
|--------|---------|
| Average Waste | 4.2% |
| Total Idle Cost | $0.019440 |
| Total Cost | $0.460279 |
| Potential Monthly Savings | $503.88 |
| Severity | LOW |

#### Optimization Recommendations

| Priority | Type | Description | Est. Savings |
|----------|------|-------------|-------------|
| MEDIUM | USE_SPOT | Switch fault-tolerant workloads to spot instances | 65.0% |
| LOW | SCHEDULING | Schedule non-urgent training during off-peak hours | 20.0% |

#### Phân tích insights

- **Waste ở mức LOW:** 4.1-4.2% waste được đánh giá là LOW severity - cluster đang hoạt động tương đối hiệu quả.
- **Idle cost ổn định:** $0.001944 idle cost mỗi snapshot, chủ yếu đến từ node-06 và node-07.
- **Tiết kiệm tiềm năng $503.88/tháng:** Con số này cho thấy nếu triển khai đầy đủ recommendations, có thể tiết kiệm đáng kể hàng tháng.
- **USE_SPOT là ưu tiên cao nhất:** Với 65% estimated savings, việc chuyển fault-tolerant workloads sang Spot là action quan trọng nhất.
- **Off-peak scheduling bổ sung:** Scheduling giúp giảm thêm 20% chi phí khi kết hợp với Spot.

---

### Part 6: Visualization

Đã tạo các biểu đồ visualization:

1. **Cost Breakdown Chart:** Phân bổ chi phí theo GPU type và workload
2. **Time-series Chart:** Theo dõi chi phí theo thời gian với waste percentage overlay

Các biểu đồ giúp trực quan hóa dữ liệu cost, hỗ trợ việc đưa ra quyết định tối ưu hóa nhanh chóng.

---

### Part 7: Complete FinOps Workflow

Workflow tổng hợp 7 bước đã được thực thi thành công, kết nối toàn bộ các phần từ monitoring → workload submission → cost tracking → spot management → autoscaling → waste analysis → recommendations vào một pipeline hoàn chỉnh. Điều này mô phỏng quy trình FinOps thực tế trong production.

---

### Part 8: Real GPU Workload (Kaggle - FP32 vs AMP)

#### GPU Detection

GPU được detect thành công trên Kaggle environment với thông số kỹ thuật đầy đủ.

#### GPU Metrics Diagnostic

Các diagnostic tests về memory, compute capability, và performance metrics đã chạy thành công.

#### Training Results

| Metric | FP32 | AMP |
|--------|------|-----|
| Training Time | Dài hơn | Ngắn hơn |
| Memory Usage | Cao hơn | Thấp hơn (~50%) |
| Accuracy | Baseline | Tương đương |
| Cost/Epoch | Cao hơn | Thấp hơn |

#### Phân tích insights

- **AMP tiết kiệm memory ~50%:** Mixed Precision (AMP) giảm đáng kể memory usage so với FP32 full precision, cho phép training với batch size lớn hơn hoặc GPU nhỏ hơn.
- **Training time cải thiện:** Với fewer memory bandwidth requirements, AMP training thường nhanh hơn trên các GPU hỗ trợ Tensor Cores (A100, V100).
- **Accuracy tương đương:** Việc sử dụng FP16 với AMP loss scaling không ảnh hưởng đáng kể đến model accuracy, đây là điểm mấu chốt khiến AMP trở thành standard practice.
- **Cost efficiency:** Thời gian training ngắn hơn + memory thấp hơn = chi phí tính trên per-epoch và per-sample thấp hơn đáng kể.
- **Real GPU telemetry:** GPU telemetry cho thấy utilization và power draw thay đổi theo workload, giúp xác định bottleneck.

---

### Part 8.5: Advanced GPU Cost Optimization

#### Multi-GPU Cost Analysis

| GPUs | Cost/GPU | Speedup | Scaling Efficiency |
|------|----------|---------|-------------------|
| 1 | $1.00 | 1.0x | 100% |
| 2 | $0.95 | 1.85x | 92.5% |
| 4 | $0.85 | 3.4x | 85.0% |
| 8 | $0.80 | 6.0x | 75.0% |

#### Project Cost Forecasting

Dự báo chi phí dựa trên historical data với confidence intervals, giúp lập kế hoạch budget dài hạn.

#### Optimization Opportunity Analysis

Các recommendations được sắp xếp theo priority và estimated savings impact.

#### Challenge Exercise

Đã thiết kế chiến lược optimization tổng hợp dựa trên tất cả phân tích từ Parts 1-8.5.

#### Phân tích insights

- **Scaling efficiency giảm theo số GPU:** Khi scale từ 1 lên 8 GPUs, efficiency giảm từ 100% xuống 75%. Nguyên nhân chính là communication overhead và không phải mọi workload đều parallelize hiệu quả.
- **Cost per GPU giảm khi scale:** Nhờ amortization của fixed costs (management, networking), cost per GPU giảm khi scale - kinh tế theo quy mô.
- **Forecasting giúp proactive management:** Project cost forecast với confidence intervals giúp anticipate future expenses thay vì chỉ reactive.
- **Chiến lược optimization cần holistic approach:** Kết hợp Spot usage, AMP training, autoscaling, và off-peak scheduling tạo ra compound effect tiết kiệm lớn nhất.

---

## 3. Kết Luận và Học Hỏi

### 3.1 Những kỹ năng FinOps đã học được

Qua bài lab, các kỹ năng FinOps quan trọng đã được thực hành:

1. **GPU Cluster Monitoring:** Sử dụng các công cụ giám sát (tương tự OpenCost) để theo dõi utilization, memory, power theo thời gian thực.
2. **Cost Allocation:** Phân bổ chi phí theo workload, node, và GPU type để hiểu rõ chi phí đến từ đâu.
3. **Spot Instance Strategy:** Đánh giá trade-off giữa savings và reliability của Spot instances; thiết kế workload phù hợp (fault-tolerant).
4. **Autoscaling:** Cấu hình KEDA-like autoscaler với thresholds, cooldown, và cost-awareness để balance giữa performance và cost.
5. **Waste Detection:** Xác định idle resources và tính toán potential savings.
6. **Mixed Precision Training:** Áp dụng AMP để giảm memory và tăng throughput mà không hy sinh accuracy.

### 3.2 Các chiến lược cost optimization hiệu quả

| Chiến lược | Savings tiềm năng | Effort | Priority |
|-----------|------------------|--------|----------|
| Sử dụng Spot instances | 30-70% | Thấp | Rất cao |
| AMP Mixed Precision | 20-40% | Thấp | Cao |
| Off-peak scheduling | 10-25% | Trung bình | Trung bình |
| Autoscaling | 15-30% | Trung bình | Cao |
| Right-sizing GPU type | 10-30% | Cao | Trung bình |
| Shutdown idle resources | 100% of idle cost | Thấp | Rất cao |

### 3.3 Ứng dụng thực tế trong projects

- **AI/ML Training Projects:** Luôn sử dụng AMP cho training để giảm chi phí. Với các training jobs dài ngày, chuyển sang Spot với checkpointing để tiết kiệm 60-70%.
- **Batch Inference:** Sử dụng T4 Spot cho inference workloads không yêu cầu low-latency tuyệt đối.
- **Experiment Management:** Thiết lập autoscaling policy phù hợp để cluster tự điều chỉnh theo nhu cầu, tránh over-provisioning.
- **Budget Tracking:** Thiết lập alerts ở ngưỡng 70-80% budget để chủ động quản lý chi phí.
- **Multi-GPU Training:** Cân nhắc scaling efficiency khi chọn số GPU - đôi khi 2 GPU hiệu quả hơn 1 GPU gấp đôi vì communication overhead.

### 3.4 Tổng kết

Bài lab GPU FinOps đã cung cấp một nền tảng thực hành toàn diện, từ mock cluster monitoring đến real GPU training. Các kết quả chính:

- Cluster với 11/16 GPUs busy (57% avg utilization) - còn dư capacity
- Spot instances tiết kiệm 30-40% so với On-Demand
- Waste chỉ 4.1% - mức LOW, cluster hoạt động hiệu quả
- Tiết kiệm tiềm năng $503.88/tháng nếu triển khai đầy đủ recommendations
- AMP training giảm memory ~50% với accuracy tương đương FP32
- Chiến lược kết hợp multi-pronged (Spot + AMP + Autoscaling + Scheduling) mang lại compound savings tối ưu

FinOps không chỉ là công cụ tiết kiệm chi phí mà còn là mindset giúp tổ chức đưa ra quyết định dựa trên data, tối ưu hóa resource allocation, và đạt được mục tiêu kinh doanh với chi phí thấp nhất có thể.

---

## 4. Thông Tin Phụ Lục

### Kiến trúc hệ thống

- **Gateway:** cloudflared tunnel kết nối Docker Compose services
- **Services:** cluster, billing, spot, autoscaler, cost (5 microservices)
- **Protocol:** REST API qua HTTP
- **Monitoring:** Real-time metrics collection

### File báo cáo

- **Notebook:** `gpu_finops_lab.ipynb` - hoàn thành đầy đủ 31 cells
- **Screenshots:** 27 screenshots minh họa tất cả các parts
- **Generated Charts:** 9 biểu đồ (cost breakdown, timeseries, GPU comparison, etc.)

---

*Report generated on 13/05/2026 by Võ Thiên Phú (2A202600336)*
