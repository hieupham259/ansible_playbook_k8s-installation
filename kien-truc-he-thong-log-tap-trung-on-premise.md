# Kiến trúc hệ thống log tập trung on-premise

- **Mục tiêu:** Nhiều máy chủ cùng ghi log về một nơi; tra cứu, tìm kiếm và tổng hợp tương tự AWS Athena; có dashboard; **toàn bộ chạy on-premise**, không phụ thuộc cloud.
- **Tiêu chí lựa chọn:** Ưu tiên **mức độ phổ biến và độ an toàn đã được kiểm chứng**, đứng trên hiệu năng trên mỗi máy chủ và chi phí phần cứng.
- **Kết luận:** **Layered log pipeline trên nền Elasticsearch/OpenSearch** (§2). Đây là kiến trúc được triển khai nhiều nhất trong on-prem tự vận hành hơn 10 năm qua.
- **Phạm vi:** Tài liệu kiến trúc — topology, vai trò từng lớp, sizing, chế độ hỏng. Không chứa file cấu hình hay mã triển khai.
- **Ngày:** 2026-08-04.

---

## 1. Nguyên tắc chọn: tại sao "phổ biến" chính là "an toàn"

Đây là luận điểm nền của toàn bộ tài liệu, và nó không hiển nhiên.

Một kiến trúc an toàn không phải là kiến trúc có thiết kế đẹp nhất trên giấy. Nó là kiến trúc mà **mọi cách hỏng đều đã có người khác gặp trước** — và đã viết lại thành runbook, thành câu trả lời trên diễn đàn, thành mục trong sách vận hành, thành kinh nghiệm của người sắp được tuyển vào đội.

Khi cluster chuyển trạng thái vàng lúc 2 giờ sáng, với Elasticsearch/OpenSearch câu trả lời đã tồn tại sẵn ở đâu đó. Với một kiến trúc mới hơn, đội vận hành là người **phát hiện ra** sự cố đó — và phát hiện đúng lúc production đang cháy.

Hệ quả: rủi ro đã biết tên thì quản lý được. Toàn bộ §7 (cái giá phải trả) là các rủi ro đã biết tên, đều có quy tắc rõ ràng.

---

## 2. Kiến trúc tham chiếu

```
     Máy 1..N          Lớp đệm bền        Lớp xử lý           Lớp lưu + tìm kiếm        Lớp hiển thị
   ┌──────────┐      ┌─────────────┐    ┌───────────┐    ┌──────────────────────┐    ┌────────────┐
   │ Filebeat │      │             │    │           │    │  3 × master node     │    │            │
   │    hoặc  │─────▶│    Kafka    │───▶│ Logstash  │───▶│  N × data hot (NVMe) │◀───│ Dashboards │
   │Fluent Bit│      │  3 broker   │    │  Fluentd  │    │  M × data warm (HDD) │    │  + Alert   │
   └──────────┘      │   RF = 3    │    │  (2 node) │    │  2 × coordinating    │    └────────────┘
    file + disk      └─────────────┘    └───────────┘    └──────────┬───────────┘
      buffer          giữ 3–7 ngày        parse,           mỗi shard 2 bản
                        replay được       enrich                    │ snapshot
                                                                    ▼
                                                          ┌──────────────────┐
                                                          │  MinIO / NFS     │
                                                          │  backup dài hạn  │
                                                          └──────────────────┘
```

### 2.1 Vai trò từng lớp — và lý do lớp đó tồn tại

| Lớp | Vai trò | Nếu bỏ lớp này thì sao |
|---|---|---|
| **Agent** (Filebeat / Fluent Bit) | Đọc file log trên máy nguồn, buffer trên đĩa cục bộ | Trung tâm chết 10 phút = mất log 10 phút |
| **Kafka** | Lớp đệm bền, tách rời nguồn khỏi đích | Không thể nâng cấp/bảo trì cluster mà không dừng ingest |
| **Logstash / Fluentd** | Parse, chuẩn hoá, làm giàu dữ liệu | Parse lỗi làm nghẽn ingest; không thể sửa rồi replay |
| **Master node (3)** | Bầu cử quorum, quản lý metadata cluster | 2 node = split-brain. **Luôn là số lẻ, luôn là 3** |
| **Data hot / warm** | Lưu trữ và tìm kiếm; mỗi shard có 2 bản | Một ổ đĩa hỏng = mất dữ liệu vĩnh viễn |
| **Coordinating node (2)** | Nhận toàn bộ query, gom kết quả từ data node | Query nặng đánh thẳng vào data node đang ghi |
| **Snapshot ra MinIO/NFS** | Backup thật sự | Replication **không phải** backup — nó nhân bản cả lệnh xoá nhầm |

Dòng cuối cùng đáng nhấn mạnh: replication bảo vệ khỏi hỏng phần cứng, **không** bảo vệ khỏi con người và ransomware. Nó chép lệnh xoá sang mọi bản sao trong vài mili giây.

---

## 3. Ba nguyên tắc quyết định độ an toàn

Ba nguyên tắc này đúng bất kể chọn engine nào, và quyết định hệ thống có sập hay không nhiều hơn hẳn việc chọn tên sản phẩm.

### 3.1 Ba lớp đệm độc lập

```
file trên máy nguồn  ──▶  Kafka  ──▶  cluster có replica
   (giữ hàng giờ)      (giữ hàng ngày)   (giữ theo retention)
```

Bất kỳ lớp nào chết, hai lớp còn lại vẫn giữ dữ liệu. Đây là điều khiến kiến trúc này sống sót qua sự cố thật.

Hệ quả bắt buộc: **ứng dụng phải ghi log ra file**, không ghi qua UDP syslog. UDP không có backpressure — khi tắc, dữ liệu biến mất im lặng, và lớp đệm thứ nhất không tồn tại.

### 3.2 Đường đọc tách khỏi đường ghi

Trong kiến trúc trên, đó là **coordinating node**. Người dùng chạy query hỏng thì coordinating node chịu trận; data node vẫn tiếp tục nhận log.

Nguyên tắc phía sau: **việc ghi log là thứ không thể làm lại, việc đọc log thì luôn có thể thử lại.** Khi tài nguyên khan hiếm, ghi phải thắng.

### 3.3 Mọi thành phần có trạng thái đều N+1, và quorum luôn lẻ

3 master, 3 Kafka broker, tối thiểu 2 bản mỗi shard. Con số 3 không tuỳ tiện: nó là số nhỏ nhất cho phép mất một node mà vẫn giữ được đa số.

---

## 4. Các điểm mất dữ liệu và cách bịt

| # | Điểm gãy | Nguyên nhân | Cách bịt |
|---|---|---|---|
| 1 | App → file log | App ghi qua UDP syslog, hoặc ghi stdout mà agent chết | Luôn ghi ra **file** — file chính là lớp đệm bền đầu tiên (§3.1) |
| 2 | Log rotation | Agent tắc 2 giờ, logrotate đã xoá mất các file cũ | Giữ số bản rotate tương ứng **ít nhất bằng thời gian buffer dự kiến** |
| 3 | Agent → Kafka | Mạng đứt, trung tâm down | Disk buffer tại agent + chỉ đánh dấu đã đọc **sau khi** đích xác nhận |
| 4 | Kafka nhận nhưng chưa bền | Ghi mới ở một broker thì broker đó chết | Ghi phải được xác nhận bởi **đa số bản sao**, không phải một bản |
| 5 | Đĩa hỏng / đĩa đầy | Một bản duy nhất; vòng đời dữ liệu không chạy | Tối thiểu 2 bản mỗi shard + cảnh báo dung lượng ở 70% |

Điểm 3 và 4 cùng chung một nguyên tắc: **không bao giờ coi dữ liệu là đã gửi xong khi mới có một bên biết về nó.**

---

## 5. Các điểm làm sập server và cách chặn

| Điểm gãy | Cơ chế | Cách chặn |
|---|---|---|
| **Shard sprawl** | Quá nhiều shard nhỏ → master quá tải → cluster mất ổn định | Giữ mỗi shard trong khoảng **10–50 GB** (§6.2) |
| **Query nặng giết data node** | Dashboard auto-refresh × nhiều panel × nhiều người xem = tự DDoS | Coordinating node riêng + giới hạn bộ nhớ/thời gian cho user đọc |
| **Heap áp lực → GC pause** | Heap vượt ngưỡng, JVM dừng thế giới liên tục | Heap **≤ 31 GB** và **≤ 50% RAM** (§6.2) |
| **Đĩa đầy** | Vòng đời dữ liệu không chạy, hoặc đột biến lưu lượng | Ngưỡng bảo vệ 85 / 90 / 95% — đến 95% index tự chuyển read-only |
| **Bão IO khi node chết** | Cluster tự cân bằng lại shard, tạo đợt IO lớn | Đủ headroom IO; trì hoãn tái phân bổ vài phút để tránh phản ứng với reboot ngắn |

Dòng cuối là nghịch lý đáng lưu ý: **cơ chế tự phục hồi của cluster cũng chính là thứ có thể làm nó quá tải.** Một node reboot 2 phút không nên kích hoạt việc chép lại hàng terabyte.

---

## 6. Sizing — ví dụ tính cụ thể

Tài liệu này dùng một ví dụ chạy xuyên suốt để mọi con số đều có thể kiểm chứng lại.

### 6.1 Giả thiết đầu vào

| Tham số | Giá trị |
|---|---|
| Log thô mỗi ngày | **100 GB** (JSON, ~500 byte/dòng → ~200 triệu dòng/ngày) |
| Retention nóng (tra cứu tương tác) | 7 ngày |
| Retention ấm (tra cứu chậm hơn) | 23 ngày (tổng online 30 ngày) |
| Retention backup | 90 ngày |
| Số bản sao mỗi shard | 2 (1 primary + 1 replica) |

### 6.2 Tính dung lượng

Với `best_compression`, dữ liệu đã lập chỉ mục của OpenSearch thường **xấp xỉ 1,0–1,3 lần log thô** — phần nén bù lại phần index nghịch đảo. Lấy hệ số 1,1:

```
Primary mỗi ngày   = 100 GB × 1,1                    = 110 GB
Trên cluster/ngày  = 110 GB × 2 bản                  = 220 GB

Vùng hot  (7 ngày)  = 220 GB × 7                     = 1,54 TB
Vùng warm (23 ngày) = 220 GB × 23                    = 5,06 TB
Tổng online                                          = 6,60 TB
+ 30% headroom (merge + ngưỡng 85%)                  = 8,58 TB đĩa thô

Snapshot 90 ngày (chỉ primary, đã nén) = 110 GB × 90 = 9,90 TB trên MinIO
```

Số shard:

```
Shard mục tiêu 30 GB  →  110 GB/ngày ÷ 30 GB  = 4 primary shard mỗi index ngày
Kèm replica                                    = 8 shard mỗi ngày
30 ngày online                                 = 240 shard

Sức chứa: 3 hot node × 31 GB heap = 93 GB heap
Quy tắc ~20 shard mỗi GB heap     ≈ 1.860 shard
→ 240 / 1.860 = 13% sức chứa. Rất thoải mái.
```

Kafka:

```
Log nén trong Kafka (~4:1)  = 100 GB ÷ 4        = 25 GB/ngày
Retention 3 ngày × RF 3     = 25 × 3 × 3        = 225 GB toàn cluster
Chia cho 3 broker                               = 75 GB/broker
→ Cấp 500 GB SSD/broker: đủ để nâng retention lên 7 ngày mà không đổi phần cứng.
```

### 6.3 Topology kết quả

| Vai trò | Số node | Cấu hình mỗi node | Ghi chú |
|---|---|---|---|
| Master | 3 | 4 vCPU / 8 GB / 100 GB SSD | Số lẻ, chỉ giữ metadata (§3.3) |
| Data hot | 3 | 16 vCPU / 64 GB / 2 TB NVMe | Heap 31 GB; tối thiểu 3 để chịu mất 1 node |
| Data warm | 2 | 8 vCPU / 32 GB / 6 TB SATA | Không còn nhận ghi, chỉ đọc |
| Coordinating | 2 | 8 vCPU / 16 GB | Toàn bộ query vào đây (§3.2) |
| Kafka broker | 3 | 8 vCPU / 32 GB / 500 GB SSD | RF = 3 |
| Logstash | 2 | 8 vCPU / 16 GB | Không trạng thái, scale ngang tự do |
| Dashboards | 2 | 4 vCPU / 8 GB | Sau load balancer |
| MinIO | 4 | 4 vCPU / 16 GB / 4 TB | Erasure coding cho 9,9 TB snapshot |

Tổng 21 node. Ở quy mô nhỏ hơn có thể gộp vai trò (§8.2), nhưng **không bao giờ gộp xuống dưới 3 master và 2 bản mỗi shard**.

### 6.4 Vòng đời dữ liệu

```
ngày 0 ─── hot (NVMe) ─── ngày 7 ─── warm (SATA) ─── ngày 30 ─── snapshot ─── ngày 90 ─── xoá
           ghi + đọc                  chỉ đọc                     MinIO
```

Snapshot phải được tạo **trước** khi index bị xoá khỏi cluster, và phải được thử khôi phục định kỳ (§9, bài 7).

---

## 7. Cái giá phải trả — nói cho công bằng

OpenSearch không miễn phí về mặt vận hành:

- **Tốn phần cứng gấp 3–5 lần** so với một hệ cột như ClickHouse cho cùng lượng log, vì index nghịch đảo chiếm chỗ.
- **Shard sprawl là kẻ giết cluster số một** (§5).
- **Heap tối đa 31 GB** — trên ngưỡng này JVM mất cơ chế nén con trỏ và hiệu năng tụt. Đây là ràng buộc cứng, không phải khuyến nghị.
- **Bão IO khi tự cân bằng** (§5).

Cả bốn đều đã có quy tắc rõ ràng và tài liệu đầy đủ. Đó lại chính là minh chứng cho §1.

---

## 8. Các biến thể thực tế

### 8.1 So sánh

| Kiến trúc | Vị thế thực tế | Chọn khi |
|---|---|---|
| **OpenSearch stack** (§2) | Phổ biến nhất trong on-prem tự vận hành | **Mặc định.** Cần hệ sinh thái chín và dễ tuyển người |
| **Splunk** | Thống trị enterprise lớn, đặc biệt mảng SIEM | Có ngân sách; cần compliance/audit sẵn sàng |
| **Graylog** (đóng gói sẵn OpenSearch) | Rất phổ biến ở on-prem quy mô vừa | **Đội vận hành dưới 3 người** |
| **Grafana Loki** | Chuẩn de-facto trong môi trường Kubernetes | Log chủ yếu tra theo nhãn; muốn nhẹ |
| **ClickHouse / SigNoz** | Đang lên nhanh, hiệu năng tốt nhất nhóm | Ưu tiên chi phí phần cứng và truy vấn SQL |

### 8.2 Hai điều chỉnh theo hoàn cảnh

**Đội vận hành mỏng → dùng Graylog thay vì tự lắp ELK.** Nó dựng sẵn đúng kiến trúc §2, kèm phân quyền, alerting và định tuyến stream. Nhận cùng nền tảng nhưng bớt phần lắp ráp — mà lắp ráp sai chính là nơi rủi ro nằm.

**Dưới ~50 GB/ngày → có thể bỏ Kafka giai đoạn đầu**, chỉ dựa vào disk buffer của agent. Nhưng phải thiết kế sao cho chèn Kafka vào sau được mà không phải sửa agent, vì đó là thứ sẽ cần đúng vào ngày phải nâng cấp cluster mà không được dừng ingest.

### 8.3 Khi nào ClickHouse là lựa chọn đúng hơn

Nếu tiêu chí đổi từ "phổ biến và an toàn nhất" sang **"truy vấn SQL giống Athena nhất, chi phí phần cứng thấp nhất"**, thì ClickHouse thắng rõ: nén tốt hơn khoảng 3–5 lần, SQL đầy đủ, tổng hợp nhanh hơn nhiều.

Đánh đổi cần biết trước: MergeTree **mặc định không fsync sau khi ghi**, nên một `INSERT` đã trả về OK vẫn có thể mất khi server mất điện đột ngột. Cách phòng thủ đúng là ghi có quorum trên tối thiểu 2 replica ở hai máy vật lý khác nhau — không phải bật fsync, vì fsync làm sụt throughput mạnh.

Ba nguyên tắc ở §3 áp dụng nguyên vẹn cho phương án này.

---

## 9. Kiểm chứng — đừng tin cấu hình, hãy thử phá

Chạy trên staging trước khi lên production. Mỗi bài kết thúc bằng cùng một câu hỏi: **đếm số dòng log trước và sau, có khớp không?**

| # | Bài kiểm tra | Kiểm chứng điều gì |
|---|---|---|
| 1 | Kill tiến trình cluster giữa lúc ingest | Replay từ Kafka có đủ không (§3.1) |
| 2 | **Rút điện vật lý một node** | Độ bền thật sự — khác hẳn kill tiến trình (§4.4) |
| 3 | Chặn firewall giữa agent và trung tâm 30 phút | Disk buffer giữ và gửi lại đủ không (§4.3) |
| 4 | Đổ đầy đĩa đến 100% | Hệ thống dừng sạch hay hỏng dữ liệu (§5) |
| 5 | Chạy query cố tình quét toàn bộ dữ liệu | Ingest có bị ảnh hưởng không (§3.2) |
| 6 | Cho một máy client lệch giờ 2 tiếng | Log của nó có tìm được không — NTP là bắt buộc |
| 7 | **Khôi phục snapshot vào cluster trống** | Backup có thật sự dùng được không (§2.1) |

Bài 2, 4 và 7 là ba bài hay bị bỏ qua nhất, và cũng là ba bài hay trượt nhất. Bài 7 đặc biệt quan trọng: một backup chưa từng được khôi phục thử thì chưa phải là backup.

---

## 10. Tóm tắt quyết định

1. **Kiến trúc:** Layered log pipeline trên OpenSearch (§2) — agent → Kafka → xử lý → cluster hot/warm → dashboard, kèm snapshot ra MinIO.
2. **Lý do:** Ưu tiên độ chín của hệ sinh thái hơn hiệu năng trên mỗi máy chủ (§1).
3. **Điều kiện tiên quyết:** Ba nguyên tắc ở §3. Thiếu bất kỳ nguyên tắc nào thì việc chọn engine nào cũng không cứu được.
4. **Điều chỉnh:** Graylog nếu đội vận hành mỏng; hoãn Kafka nếu dưới 50 GB/ngày nhưng phải chừa đường chèn vào sau (§8.2).
5. **Nghiệm thu:** Bảy bài ở §9, đặc biệt bài khôi phục snapshot.
