# Kiến trúc EKS + ALB cho ứng dụng 100–1000 CCU: thiết kế, hạ tầng và chi phí

> **Tài liệu này trả lời:** một ứng dụng web chạy trên Amazon EKS, nhận traffic qua ALB do AWS Load Balancer
> Controller quản lý, ở quy mô 100–1000 người dùng đồng thời, thì nên dựng hạ tầng thế nào và mỗi tháng tốn bao nhiêu.
>
> **Ví dụ xuyên suốt:** một sàn thương mại điện tử nhỏ, có các đợt flash sale.
>
> **Đối tượng:** kỹ sư backend và DevOps cần hiểu kiến trúc và lập ngân sách hạ tầng.
>
> **Giá:** lấy ngày 2026-10-12 từ AWS Price List API, vùng Asia Pacific (Singapore) `ap-southeast-1`, giá on-demand,
> một tháng tính 730 giờ, chưa gồm thuế. Giá AWS có thể đổi; kiểm lại bằng AWS Pricing Calculator trước khi chốt.
>
> **Cách viết số:** kiểu Việt, dấu chấm ngăn hàng nghìn, dấu phẩy ngăn phần thập phân. Ví dụ `$0,102` là 0.102 USD,
> `$2.277` là hai nghìn hai trăm bảy mươi bảy USD. Khối code giữ cú pháp gốc.

---

## Mục lục

1. [Bài toán và giả định tải](#1-bài-toán-và-giả-định-tải)
2. [Sơ đồ kiến trúc](#2-sơ-đồ-kiến-trúc)
3. [Đi theo một request](#3-đi-theo-một-request)
4. [Tính số Pod và node](#4-tính-số-pod-và-node)
5. [AWS Load Balancer Controller và ALB](#5-aws-load-balancer-controller-và-alb)
6. [Mạng](#6-mạng)
7. [Compute và autoscaling](#7-compute-và-autoscaling)
8. [Deploy không downtime](#8-deploy-không-downtime)
9. [Tầng dữ liệu](#9-tầng-dữ-liệu)
10. [Bảo mật, quan sát, CI/CD, dự phòng](#10-bảo-mật-quan-sát-cicd-dự-phòng)
11. [Chi phí vận hành](#11-chi-phí-vận-hành)
12. [Khi vượt 1000 CCU](#12-khi-vượt-1000-ccu)
13. [Tóm tắt quyết định thiết kế](#13-tóm-tắt-quyết-định-thiết-kế)
14. [Nguồn](#14-nguồn)

---

## 1. Bài toán và giả định tải

### 1.1. Từ CCU ra request

CCU (concurrent users) là số người đang mở app và thao tác **cùng một lúc**. Hạ tầng không nhìn thấy người dùng, nó
chỉ nhìn thấy request và byte. Vì vậy việc đầu tiên là đổi CCU ra số request mỗi giây (RPS) và băng thông.

Dưới đây là giả định về hành vi của **một** người dùng đang hoạt động. Đây là số điển hình cho app mua sắm. Khi có số
đo thật từ app của bạn thì thay vào, mọi phép tính bên dưới làm lại được y như vậy.

- Cứ khoảng 20 giây người dùng mở một màn hình mới. Mỗi màn hình gọi 4 API và tải 6 ảnh hoặc file JS mới (phần còn
  lại đã nằm trong cache của app hoặc trình duyệt). Suy ra:
  - API: 4 ÷ 20 = **0,2 request/giây**.
  - Ảnh, JS: 6 ÷ 20 = **0,3 request/giây**.
  - Tổng request đi qua CDN: **0,5 request/giây**.
- Mỗi màn hình tải khoảng 200 KB (ảnh đã nén WebP cộng JSON), tức 200 ÷ 20 = **10 KB/giây** mỗi người.
- Lúc tranh mua trong flash sale, người dùng bấm liên tục: API tăng lên **1 request/giây** mỗi người.
- Mỗi API request gửi lên khoảng 1 KB và nhận về khoảng 15 KB, tức ALB xử lý **16 KB** mỗi request.

Tính tay với 1000 CCU:

- API lúc bình thường: 1000 × 0,2 = **200 RPS**.
- API lúc flash sale: 1000 × 1 = **1000 RPS**.
- Băng thông ra người dùng: 1000 × 10 KB/s = **10 MB/s**.

Để tính tiền theo tháng cần số **trung bình**, không phải số đỉnh. Người dùng đông buổi tối và gần như vắng ban đêm,
nên giả định **CCU trung bình cả tháng bằng 25% CCU đỉnh**. Đỉnh 1000 CCU thì trung bình 250 CCU. Một tháng có
730 × 3.600 = 2.628.000 giây, nên tổng "giây-người dùng" là 250 × 2.628.000 = 657 triệu. Từ đó:

- API request mỗi tháng: 657 triệu × 0,2 = 131,4 triệu, trung bình 50 RPS.
- Request qua CloudFront mỗi tháng: 657 triệu × 0,5 = 328,5 triệu.
- Dữ liệu ra internet mỗi tháng: 657 triệu × 10 KB = 6.570 GB.

> **Nếu "1000 CCU/s" nghĩa là mỗi giây có 1000 người *mới vào*** thì quy mô lớn hơn rất nhiều. Số người đồng thời
> bằng tốc độ vào nhân thời gian ở lại (định luật Little). Phiên trung bình 10 phút (600 giây) thì
> 1000 × 600 = **600.000 CCU**, gấp 600 lần mức M dưới đây. Tài liệu này hiểu CCU là số người dùng đồng thời.

### 1.2. Hai mức thiết kế

Tài liệu tính cho hai đầu của khoảng mục tiêu: **mức S = 100 CCU** và **mức M = 1000 CCU**. Ứng dụng nằm giữa hai
mức thì nội suy, hoặc bắt đầu ở S rồi nâng dần lên M.

| Chỉ số | Mức S | Mức M | Cách tính |
| --- | --- | --- | --- |
| CCU đỉnh | 100 | 1000 | mục tiêu |
| API lúc đỉnh bình thường | 20 RPS | 200 RPS | CCU × 0,2 |
| API lúc flash sale | 100 RPS | 1000 RPS | CCU × 1 |
| CCU trung bình cả tháng | 25 | 250 | 25% đỉnh |
| API request mỗi tháng | 13,1 triệu | 131,4 triệu | CCU trung bình × 0,2 × 2.628.000 |
| Request qua CloudFront mỗi tháng | 32,9 triệu | 328,5 triệu | CCU trung bình × 0,5 × 2.628.000 |
| Dữ liệu ra internet mỗi tháng | 657 GB | 6.570 GB | CCU trung bình × 10 KB × 2.628.000 |
| Giờ flash sale mỗi tháng | 30 | 30 | 10 đợt × 3 giờ (gồm 30 phút scale trước) |

Yêu cầu chung cho cả hai mức:

- p99 latency của API dưới 300 ms.
- Deploy không gây downtime.
- Không bán quá số hàng tồn.
- Mức S chịu được mất 1 node. Mức M chịu được mất trọn 1 Availability Zone (AZ, một cụm trung tâm dữ liệu độc lập
  trong region).

---

## 2. Sơ đồ kiến trúc

```text
 Người dùng (web / app)
        │
        ▼
 Route 53  (shop.example.com → CloudFront)
        │
        ▼
 CloudFront + AWS WAF ─────────────────────────► S3 riêng tư (ảnh, JS, CSS), đọc qua OAC
        │  /api/* không cache, đi qua VPC origin
        ▼
┌─────────────────────────── VPC 10.0.0.0/16 ───────────────────────────┐
│                                                                       │
│  Public subnet    NAT Gateway (S: 1 cái · M: mỗi AZ 1 cái)            │
│                                                                       │
│  Private subnet   ALB internal (Terraform tạo)                        │
│                     │  target-type ip: gửi thẳng tới IP của Pod       │
│                     ▼                                                 │
│                   ┌──────────────── EKS cluster ────────────────┐     │
│                   │ Addon: AWS Load Balancer Controller,        │     │
│                   │        CoreDNS, metrics-server,             │     │
│                   │        External Secrets                     │     │
│                   │        (M thêm: Karpenter, KEDA, Prometheus)│     │
│                   │ App:   api (Deployment), worker (đọc SQS)   │     │
│                   └─────────────────────────────────────────────┘     │
│                                                                       │
│  Data subnet      RDS PostgreSQL Multi-AZ · ElastiCache Valkey        │
│                                                                       │
└───────────────────────────────────────────────────────────────────────┘
  Dịch vụ cấp region: SQS · ECR · Secrets Manager · S3 (qua gateway endpoint)
```

Hai mức dùng cùng một khung, chỉ khác số lượng và kích cỡ:

| Thành phần | Mức S (100 CCU) | Mức M (1000 CCU) |
| --- | --- | --- |
| Số AZ | 2 | 3 |
| Node EKS | 2 × `m7g.large`, chạy chung addon và app | 3 node hệ thống + 3 node app; thêm 3 node khi flash sale |
| Cấp node tự động | Managed node group | Karpenter |
| Pod `api` | 2 (HPA tối đa 4) | 3 lúc thường, 15 lúc flash sale |
| Database | RDS PostgreSQL `db.t4g.medium` Multi-AZ | RDS PostgreSQL `db.m7g.large` Multi-AZ |
| Cache | Valkey 2 × `cache.t4g.small` | Valkey 2 × `cache.t4g.medium` |
| NAT Gateway | 1 | 3 (mỗi AZ một cái) |
| Edge | CloudFront + WAF trả theo dùng | CloudFront gói phẳng Premium (đã gồm WAF) |
| Quan sát | CloudWatch | CloudWatch + Prometheus/Grafana tự host |

`m7g` là dòng máy dùng chip Graviton (ARM) của AWS, rẻ hơn dòng Intel cùng cỡ: `m7g.large` giá $0,102/giờ, còn
`m7i.large` giá $0,126/giờ, đắt hơn 23,5%. Đổi lại, image container phải build cho `arm64`.

---

## 3. Đi theo một request

**Route 53** trả địa chỉ CloudFront cho `shop.example.com`.

**CloudFront + WAF:**

- Ảnh, JS, CSS lấy từ một bucket S3 riêng tư. CloudFront đọc bucket qua Origin Access Control (OAC), người ngoài
  không đọc thẳng được bucket.
- WAF giới hạn tốc độ theo IP và áp các bộ luật có sẵn (managed rules), chặn traffic rác trước khi nó tốn tiền chạy
  Pod.
- Request động (`/api/*`) không cache, đi tiếp vào ALB qua **VPC origin**. VPC origin cho CloudFront nói chuyện thẳng
  với một ALB **internal** nằm trong subnet private, không tốn thêm phí. Vì ALB không có IP public:
  - Không ai đi vòng qua CloudFront và WAF để đánh thẳng vào ALB được.
  - Không phải trả $0,005/giờ cho mỗi IP public. Một ALB public trải 3 AZ tốn 3 × $3,65 = $10,95/tháng chỉ riêng
    tiền IP.
- Dữ liệu từ ALB hoặc S3 sang CloudFront được miễn phí. Chỉ chiều từ CloudFront ra người dùng là tính tiền.

**ALB** chia request theo path về target group của từng service. Với **target-type ip**, mỗi Pod có một IP thật
trong VPC (do VPC CNI cấp), nên ALB gửi thẳng tới Pod, không vòng qua NodePort. Health check của ALB vì thế phản ánh
đúng từng Pod. Đây cũng là điều kiện để readiness gate hoạt động (mục 8).

**Pod `api`** không giữ trạng thái. Nó đọc cache Valkey trước, trượt cache mới đọc PostgreSQL. Lệnh đặt hàng đi qua
hàng đợi SQS (mục 9).

**Valkey** là bản mã nguồn mở tách ra từ Redis, dùng y nguyên lệnh Redis. Trên ElastiCache nó rẻ hơn Redis khoảng
20%: `cache.t4g.small` giá $0,0384/giờ so với $0,048/giờ của Redis OSS.

---

## 4. Tính số Pod và node

Mọi phép tính dựa trên một số đo từ load test. Ở đây là số giả định, **phải đo lại cho app thật**: một Pod `api` xin
0,5 vCPU và 1 GiB RAM, giữ được p99 dưới 300 ms tới **100 RPS**, khi đó CPU khoảng 70%.

Một node `m7g.large` có 2 vCPU và 8 GiB. Sau phần dành cho kubelet và hệ điều hành, còn khoảng 1,9 vCPU cho Pod. Mỗi
node còn chạy các DaemonSet (VPC CNI, kube-proxy, Pod Identity agent, Fluent Bit), tốn khoảng 0,3 vCPU.

### 4.1. Mức S

1. Lúc flash sale cần 100 ÷ 100 = 1 Pod.
2. Để chịu mất 1 node, chạy 2 Pod trên 2 node ở 2 AZ. Mất 1 node vẫn còn 1 Pod gánh đủ 100 RPS.
3. Tổng CPU cần:
   - `api`: 2 × 0,5 = 1 vCPU.
   - `worker`: 0,25 vCPU.
   - Addon (Load Balancer Controller, CoreDNS, metrics-server, External Secrets): khoảng 0,6 vCPU.
   - DaemonSet: 2 × 0,3 = 0,6 vCPU.

   Cộng lại 2,45 vCPU, trong khi 2 node cho 3,8 vCPU.
4. Khi mất 1 node, phần tối thiểu (1 `api`, 1 `worker`, mỗi addon 1 bản, DaemonSet) cần khoảng 1,45 vCPU, vừa với
   1,9 vCPU của node còn lại. Managed node group tự dựng node thay thế trong vài phút.

### 4.2. Mức M

1. Lúc thường cần 200 ÷ 100 = 2 Pod. Để chịu mất 1 AZ, hai AZ còn lại phải tự gánh 2 Pod, tức mỗi AZ 1 Pod. Với 3
   AZ: **3 Pod**.
2. Lúc flash sale cần 1000 ÷ 100 = 10 Pod. Chịu mất 1 AZ: mỗi AZ 10 ÷ 2 = 5 Pod. Với 3 AZ: **15 Pod**, tức 7,5 vCPU.
3. Node cho app lúc thường: 3 × `m7g.large`, mỗi AZ một node. Mỗi node chạy 1 `api` (0,5) + `worker` (0,25) +
   DaemonSet (0,3), khoảng 1,05 vCPU. Phần trống còn lại đủ để HPA tăng lên 6 Pod mà chưa cần node mới.
4. Lúc flash sale cần 15 `api` + 4 `worker` = 8,5 vCPU. Mỗi node dùng được khoảng 1,6 vCPU (1,9 trừ DaemonSet), 6
   node cho 9,6 vCPU. Vậy Karpenter thêm **3 node**, mỗi AZ một.
5. Node hệ thống: 3 × `m7g.large`, mỗi AZ một node. Karpenter không được chạy trên node do chính nó tạo ra, nên cần
   nhóm node riêng. Nhóm này chạy Karpenter, Load Balancer Controller, CoreDNS, KEDA, metrics-server, External
   Secrets, Prometheus (khoảng 1 vCPU, 4 GiB) và Grafana, tổng khoảng 3,1 vCPU và 8,3 GiB.

---

## 5. AWS Load Balancer Controller và ALB

Controller chạy như một Deployment 2 bản trong cluster, nhận quyền IAM qua **EKS Pod Identity**. Việc của nó:

| Nó theo dõi | Nó làm trên AWS |
| --- | --- |
| `TargetGroupBinding` (hoặc `Ingress`, `Service type: LoadBalancer`) | Gắn target group với Service, mở security group cho ALB vào Pod |
| EndpointSlice của Service | Đăng ký và huỷ đăng ký IP Pod trong target group |
| Trạng thái target trên ALB | Đặt readiness gate của Pod thành True khi target đã healthy |

### 5.1. Ai sở hữu ALB

**Khuyến nghị: Terraform tạo ALB, listener và target group. Controller chỉ gắn Pod vào target group qua
`TargetGroupBinding`.**

ALB là cửa chính mà CloudFront (qua VPC origin) trỏ vào. Nếu để `Ingress` tạo ALB, chỉ cần ai đó xoá hoặc đổi tên
Ingress là ALB bị xoá rồi dựng lại với địa chỉ mới, và CloudFront mất origin. Khi ALB nằm ngoài vòng đời của K8s:

- Thay cả cluster (blue/green lúc nâng version EKS) chỉ cần đổi trọng số giữa hai target group.
- Hạ tầng (Terraform) và app (manifest K8s) có ranh giới rõ.

Đổi lại, thêm route mới phải sửa Terraform thay vì sửa Ingress.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: shop
  labels:
    elbv2.k8s.aws/pod-readiness-gate-inject: enabled
---
apiVersion: elbv2.k8s.aws/v1beta1
kind: TargetGroupBinding
metadata:
  name: api
  namespace: shop
spec:
  serviceRef:
    name: api
    port: 8080
  targetGroupARN: arn:aws:elasticloadbalancing:ap-southeast-1:111122223333:targetgroup/shop-api/0123456789abcdef
  targetType: ip
  networking:
    ingress:
    - from:
      - securityGroup:
          groupID: sg-0123456789abcdef0
      ports:
      - protocol: TCP
        port: 8080
```

- Nhãn trên namespace bật webhook chèn readiness gate. Webhook chỉ chèn **lúc Pod được tạo** và khi
  `TargetGroupBinding` đã tồn tại. Pod có từ trước phải `kubectl rollout restart` mới được chèn.
- Phần `networking` cho controller tự mở cổng 8080 từ security group của ALB (`sg-0123...`) vào Pod, không phải viết
  rule tay.

### 5.2. Cấu hình target group (trong Terraform)

| Thuộc tính | Giá trị | Vì sao |
| --- | --- | --- |
| Kiểu target | `ip` | Gửi thẳng tới Pod, cần cho readiness gate |
| Health check | `/healthz`, mỗi 10 giây, 2 lần đạt là healthy | Pod mới được nhận traffic sau khoảng 20–30 giây |
| `deregistration_delay` | 30 giây | Mặc định 300 giây làm rollout rất chậm; 30 khớp với mục 8 |
| Thuật toán chia tải | `least_outstanding_requests` | Gửi request cho Pod đang ít việc nhất |
| `slow_start` | 30 giây, nếu app khởi động chậm (JVM) | Pod mới nhận tải tăng dần |

---

## 6. Mạng

### 6.1. Chia subnet

VPC `10.0.0.0/16`, mỗi AZ ba subnet:

| Subnet | Cỡ mỗi AZ | Chứa |
| --- | --- | --- |
| Public | /24 | NAT Gateway |
| Private | /19 (8.190 IP) | ALB internal, node EKS, mọi Pod |
| Data | /24 | RDS, ElastiCache |

App nhỏ mà subnet private vẫn chia lớn, vì VPC CNI cấp cho mỗi Pod một IP thật. Khi bật prefix delegation, IP được
giữ theo khối 16 cái: mức M lúc flash sale có 9 node, mỗi node giữ ít nhất một khối, đã chiếm từ 144 IP trở lên. IP
private không tốn tiền, còn đổi dải IP khi hệ thống đã chạy thì rất vất vả, nên chia rộng ngay từ đầu.

### 6.2. NAT Gateway

Pod trong subnet private cần đường ra internet để gọi cổng thanh toán, kéo image từ ECR, gọi API của AWS. Đường ra đó
là NAT Gateway: $0,059/giờ ($43,07/tháng), cộng $0,059 cho mỗi GB đi qua, cộng $3,65/tháng cho IP public của nó.

- **Mức S: 1 NAT.** Đánh đổi: nếu AZ chứa NAT sập, Pod ở AZ còn lại cũng mất đường ra internet cho tới khi AZ hồi
  phục.
- **Mức M: 3 NAT, mỗi AZ một cái.** Luồng thanh toán phụ thuộc đường ra internet, nên không để một AZ kéo sập cả hệ
  thống. Hai NAT thêm tốn 2 × ($43,07 + $3,65) = $93,44/tháng.

### 6.3. VPC endpoint: cái nào đáng tiền

- **Gateway endpoint cho S3** (và DynamoDB nếu dùng): miễn phí. Luôn bật, để traffic tới S3 không đi qua NAT.
- **Interface endpoint** (ECR, STS, CloudWatch Logs, Secrets Manager...): $0,013/giờ cho mỗi AZ, cộng $0,01/GB. Một
  endpoint trải 3 AZ tốn 3 × $0,013 × 730 = $28,47/tháng.

Tính tay cho ECR. Kéo image cần 2 endpoint (`ecr.api` và `ecr.dkr`), tốn 2 × $28,47 = $56,94/tháng. Không có
endpoint thì kéo 50 GB image mỗi tháng qua NAT tốn 50 × $0,059 = $2,95. Mỗi GB đi qua endpoint thay vì qua NAT chỉ
tiết kiệm $0,059 − $0,01 = $0,049. Muốn bù được $56,94 thì phải kéo 56,94 ÷ 0,049 ≈ **1.162 GB/tháng**.

Ở quy mô 100–1000 CCU còn xa mức đó, nên **chưa dùng interface endpoint**, trừ khi yêu cầu bảo mật cấm traffic tới
dịch vụ AWS đi ra internet.

---

## 7. Compute và autoscaling

### 7.1. Mức S

- Một managed node group, 2 node `m7g.large` (tối đa 3), chạy chung addon và app.
- HPA cho `api`: mục tiêu CPU 60%, tối thiểu 2, tối đa 4 Pod.
- Không cần Karpenter và KEDA: đỉnh flash sale 100 RPS vừa với 2 Pod sẵn có.

### 7.2. Mức M

- **Node hệ thống:** managed node group 3 × `m7g.large`, có taint để chỉ addon chạy trên đó.
- **Node app:** Karpenter, cho dùng các dòng Graviton (`m7g`, `c7g`). `api` chạy on-demand, `worker` cho phép chạy
  spot.
- **`worker`:** KEDA scale theo số message đang chờ trong SQS.
- **`api`:** KEDA kết hợp hai điều kiện, theo lịch (scale trước flash sale) và theo CPU.

Phải scale trước vì thời gian từ lúc tải tăng tới lúc Pod mới thật sự nhận traffic khá dài. Tính tay:

| Bước | Ước lượng |
| --- | --- |
| metrics-server lấy số liệu, HPA quyết định tăng Pod | 15–30 giây |
| Karpenter thấy Pod chờ, gọi EC2, node mới vào cluster | 60–90 giây |
| Kéo image (node mới chưa có sẵn) | 10–30 giây |
| App khởi động, readiness probe qua | 10–30 giây |
| ALB health check qua 2 lần, readiness gate chuyển True | 30–60 giây |
| **Tổng** | **2–4 phút** |

Flash sale đẩy tải từ 200 lên 1000 RPS trong vài giây. Nếu chỉ chờ autoscaling, trong 2–4 phút đầu, đúng lúc đông
nhất, 3 Pod phải gánh 1000 RPS trong khi sức chịu chỉ 300. Vì vậy nâng sẵn lên 15 Pod từ 30 phút trước:

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: api
  namespace: shop
spec:
  scaleTargetRef:
    name: api
  minReplicaCount: 3
  maxReplicaCount: 15
  triggers:
  - type: cron
    metadata:
      timezone: Asia/Ho_Chi_Minh
      start: 30 19 * * 5
      end: 30 22 * * 5
      desiredReplicas: "15"
  - type: cpu
    metricType: Utilization
    metadata:
      value: "60"
```

Đọc cấu hình: flash sale lúc 20:00 thứ Sáu. Từ 19:30 tới 22:30, trigger `cron` giữ 15 Pod. KEDA lấy số Pod lớn nhất
mà các trigger yêu cầu, nên trong khung giờ này số Pod không xuống dưới 15. Ngoài khung giờ, trigger `cpu` quyết định
trong khoảng 3–15 Pod.

---

## 8. Deploy không downtime

Cấu hình rollout: `maxSurge: 25%`, `maxUnavailable: 0`. Namespace đã bật chèn readiness gate (mục 5.1).

### 8.1. Pod mới lên: readiness gate lo phần này

| Thời điểm | Sự kiện |
| --- | --- |
| t = 0 | Pod v2 được tạo |
| t ≈ 20 giây | Readiness probe qua, `ContainersReady=True` |
| t ≈ 21 giây | Controller đăng ký IP Pod vào target group, target ở trạng thái `initial` |
| t ≈ 45 giây | ALB health check qua 2 lần, target chuyển `healthy` |
| t ≈ 50 giây | Controller đặt gate thành True, Pod có `Ready=True`, Deployment mới xoá một Pod v1 |

Không có gate, Deployment coi v2 sẵn sàng ngay ở t ≈ 20 giây và xoá v1, trong khi ALB chưa gửi request nào tới v2. Ở
mức S chỉ có 2 Pod, rollout nhanh rất dễ rơi vào khoảng target group không còn target healthy nào, và người dùng nhận
lỗi 502/503.

### 8.2. Pod cũ tắt: cần cấu hình riêng

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
  namespace: shop
spec:
  selector:
    matchLabels:
      app: api
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: api
    spec:
      terminationGracePeriodSeconds: 60
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: api
      containers:
      - name: api
        image: 111122223333.dkr.ecr.ap-southeast-1.amazonaws.com/shop-api:1.4.2
        resources:
          requests:
            cpu: 500m
            memory: 1Gi
        readinessProbe:
          httpGet:
            path: /healthz
            port: 8080
          periodSeconds: 5
        lifecycle:
          preStop:
            sleep:
              seconds: 20
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: api
  namespace: shop
spec:
  maxUnavailable: 1
  selector:
    matchLabels:
      app: api
```

`preStop.sleep` là lệnh chờ có sẵn của Kubernetes từ bản 1.30, image không cần có chương trình `sleep`. Cùng với
`deregistration_delay` 30 giây của target group (mục 5.2), Pod cũ tắt theo trình tự:

| Thời điểm | Sự kiện |
| --- | --- |
| t = 0 | Pod v1 bị xoá. **Hai việc chạy song song:** Endpoint đánh dấu Pod không còn ready, và kubelet bắt đầu `preStop` chờ 20 giây |
| t ≈ 1–5 giây | Controller huỷ đăng ký target, ALB ngừng gửi request mới (target `draining`). Trong vài giây ALB chưa áp dụng xong, request vẫn tới và Pod vẫn phục vụ vì chưa nhận SIGTERM |
| t = 20 giây | `preStop` xong, app nhận SIGTERM, ngừng nhận kết nối mới và xử lý nốt request đang dở (≤ 10 giây) |
| t ≈ 30 giây | App thoát. Cũng là lúc hết `deregistration_delay`, ALB đóng hẳn kết nối |

Ba quy tắc rút ra từ timeline:

- `preStop` phải dài hơn thời gian ALB cần để thật sự ngừng gửi request mới.
- `deregistration_delay` bằng khoảng `preStop` cộng thời gian xử lý nốt request.
- `terminationGracePeriodSeconds` lớn hơn `preStop` cộng thời gian xử lý nốt request.

Các cấu hình bổ trợ trong YAML trên:

- **PodDisruptionBudget:** PDB đếm Pod theo condition `Ready`, mà `Ready` đã gồm readiness gate. Khi Karpenter gom
  node hoặc khi nâng cấp node, việc rút Pod cũng phải chờ ALB xác nhận Pod thay thế đã healthy.
- **`topologySpreadConstraints`:** chia đều Pod qua các AZ, khớp với phép tính ở mục 4.
- **Health check và readiness probe chỉ kiểm tra bản thân Pod, không kiểm tra DB.** Nếu kiểm tra DB, chỉ cần DB chậm
  là mọi Pod cùng bị coi là hỏng một lúc, và cả hệ thống sập theo DB.

---

## 9. Tầng dữ liệu

### 9.1. PostgreSQL trên RDS Multi-AZ

Multi-AZ nghĩa là RDS giữ một bản standby ở AZ khác và ghi đồng bộ sang đó. Khi máy chính hỏng, RDS tự chuyển sang
standby trong khoảng 1–2 phút, endpoint giữ nguyên. Standby không nhận truy vấn đọc.

- **Mức S: `db.t4g.medium`** (2 vCPU, 4 GiB). Dòng `t` là loại "burstable": chạy nhẹ thì tích điểm CPU, chạy nặng thì
  tiêu điểm, CPU cao kéo dài thì RDS tính thêm tiền cho phần vượt. Theo dõi `CPUCreditBalance`. Nếu số này cứ giảm
  dần qua các ngày, chuyển sang `db.m7g.large`.
- **Mức M: `db.m7g.large`** (2 vCPU, 8 GiB), CPU ổn định.

Số kết nối: mức M lúc flash sale có 15 Pod × pool 10 + 4 `worker` × 5 = 170 kết nối, `db.m7g.large` chịu được. Vì
vậy **chưa cần RDS Proxy**. Khi cần (nhiều kết nối hơn, hoặc muốn failover nhanh hơn), RDS Proxy tính $0,018 cho mỗi
vCPU của DB mỗi giờ: 2 × $0,018 × 730 = $26,28/tháng.

### 9.2. Valkey trên ElastiCache

Hai node (chính và bản sao) ở hai AZ, tự chuyển khi node chính hỏng. Dùng cho: cache dữ liệu sản phẩm, session, bộ
đếm rate-limit, và **bộ đếm tồn kho của hàng flash sale**.

### 9.3. Đặt hàng flash sale: vì sao tồn kho nằm ở Valkey

Cách đơn giản nhất là trừ tồn kho thẳng trong PostgreSQL:

```sql
UPDATE products SET stock = stock - 1 WHERE id = 42 AND stock > 0;
```

Câu lệnh này đúng, không bán quá tồn, vì PostgreSQL khoá dòng trong lúc cập nhật. Vấn đề nằm ở chính cái khoá: mọi
lệnh trên cùng sản phẩm phải xếp hàng. Tính tay ở mức M: 1000 người cùng bấm "Mua" một sản phẩm hot trong cùng một
giây. Mỗi lệnh giữ khoá khoảng 1 ms (gồm cả ghi log giao dịch), nên người cuối hàng chờ khoảng 1000 × 1 ms = 1 giây,
vượt mục tiêu p99 300 ms.

Với Valkey, lệnh `DECR` chạy trong bộ nhớ, mỗi lệnh khoảng 0,1 ms, nên 1000 lệnh xong trong khoảng 0,1 giây. Luồng
xử lý:

1. Trước flash sale, nạp số tồn của hàng flash sale vào Valkey: `SET stock:42 500`.
2. API chạy `DECR stock:42`. Kết quả từ 0 trở lên: giữ được hàng. Kết quả âm: chạy `INCR stock:42` để trả lại, rồi
   báo "hết hàng".
3. API đẩy message "đơn mới" vào SQS, kèm idempotency key (mã duy nhất của lần bấm) để bấm lại không tạo đơn trùng,
   rồi trả `202 Đang xử lý`.
4. `worker` đọc SQS, ghi đơn vào PostgreSQL, gọi cổng thanh toán.

Ở mức M, PostgreSQL thừa sức ghi 100–200 đơn mỗi giây. Hàng đợi ở đây không phải để cứu DB, mà để tách việc gọi cổng
thanh toán (chậm, có thể lỗi và phải thử lại) ra khỏi request của người dùng. Hàng thường, không thuộc flash sale, vẫn
trừ tồn bằng câu `UPDATE` ở trên.

---

## 10. Bảo mật, quan sát, CI/CD, dự phòng

| Lớp | Mức S | Mức M |
| --- | --- | --- |
| Bảo mật | WAF (rate-limit, managed rules); EKS API endpoint private; Pod Identity riêng cho từng service; Secrets Manager + External Secrets Operator; security group riêng cho RDS và Valkey | Như S, thêm NetworkPolicy (VPC CNI hỗ trợ sẵn) giữa các service |
| Quan sát | Metric có sẵn của ALB, RDS, ElastiCache, SQS trên CloudWatch (miễn phí); log app qua Fluent Bit vào CloudWatch Logs; khoảng 10 alarm | Như S, thêm Prometheus + Grafana tự host trên node hệ thống cho metric của Pod; khoảng 20 alarm |
| CI/CD | GitHub Actions build image `arm64`, đẩy lên ECR, `helm upgrade`; Terraform cho hạ tầng | Như S, thay `helm upgrade` bằng Argo CD (GitOps) |
| Sao lưu, dự phòng | RDS backup tự động 7 ngày, khôi phục về thời điểm bất kỳ (PITR); một region | Như S, thêm copy snapshot sang region khác nếu cần |
| Kiểm chứng | Load test 1,5 lần mức đỉnh trước mỗi đợt mở bán lớn | Như S, thêm diễn tập tắt một AZ bằng AWS FIS |

Các alarm nên có ngay từ đầu:

- ALB: `HTTPCode_Target_5XX_Count` tăng, `TargetResponseTime` p99 vượt 300 ms, `HealthyHostCount` dưới 2.
- RDS: `CPUUtilization`, `FreeStorageSpace`, `DatabaseConnections`, và `CPUCreditBalance` với mức S.
- ElastiCache: `EngineCPUUtilization`, `DatabaseMemoryUsagePercentage`.
- SQS: `ApproximateAgeOfOldestMessage`, tức đơn hàng đang chờ quá lâu.

---

## 11. Chi phí vận hành

### 11.1. Bảng giá đơn vị đã dùng

Tất cả lấy từ AWS Price List API ngày 2026-10-12, vùng Singapore, trừ dòng KMS lấy theo giá niêm yết chung.

| Hạng mục | Đơn giá |
| --- | --- |
| EKS cluster | $0,10/giờ ($0,60/giờ nếu version hết hạn hỗ trợ chuẩn) |
| EC2 `m7g.large` (2 vCPU, 8 GiB) | $0,102/giờ; Compute Savings Plan 1 năm không trả trước: $0,076/giờ |
| EBS gp3 | $0,096/GB-tháng |
| ALB | $0,0252/giờ + $0,008/LCU-giờ |
| NAT Gateway | $0,059/giờ + $0,059/GB |
| IP public (IPv4) | $0,005/giờ |
| VPC interface endpoint | $0,013/giờ mỗi AZ + $0,01/GB |
| RDS PostgreSQL `db.t4g.medium` Multi-AZ | $0,203/giờ; Reserved 1 năm không trả trước: $0,1605/giờ |
| RDS PostgreSQL `db.m7g.large` Multi-AZ | $0,468/giờ; Reserved 1 năm không trả trước: $0,3607/giờ |
| Ổ RDS gp3 Multi-AZ | $0,276/GB-tháng |
| ElastiCache Valkey `cache.t4g.small` / `cache.t4g.medium` | $0,0384/giờ / $0,076/giờ |
| CloudFront trả theo dùng (châu Á) | $0,120/GB cho 10 TB đầu, $0,100/GB 40 TB tiếp; $0,012 cho 10.000 request HTTPS; miễn phí mỗi tháng 1 TB và 10 triệu request |
| CloudFront gói phẳng | Pro $15 · Business $200 · Premium $1.000 mỗi tháng (hạn mức ở mục 11.4) |
| WAF | $5/web ACL + $1/rule mỗi tháng + $0,60/triệu request |
| CloudWatch | Log: $0,70/GB nạp + $0,03/GB-tháng lưu; alarm $0,10/tháng |
| Truyền dữ liệu giữa hai AZ | $0,01/GB mỗi chiều |
| Secrets Manager | $0,40/secret-tháng |
| SQS | $0,40/triệu request, 1 triệu đầu mỗi tháng miễn phí |
| S3 Standard / ECR | $0,025/GB-tháng / $0,10/GB-tháng |
| Route 53 | $0,50/hosted zone-tháng |
| KMS | $1/key-tháng |

### 11.2. Mức S: khoảng $587/tháng

Trước khi đọc dòng ALB: **LCU** là đơn vị đo tải của ALB. Traffic đi qua CloudFront dùng lại kết nối, nên chiều quyết
định số LCU là dữ liệu xử lý: 1 LCU = 1 GB mỗi giờ.

- Mức S trung bình 5 RPS × 16 KB = 80 KB/giây = 0,288 GB/giờ, tức 0,288 LCU.
- Lúc flash sale: 100 RPS × 16 KB × 3.600 giây = 5,76 GB/giờ, tức 5,76 LCU, trong 30 giờ mỗi tháng.

| Hạng mục | Cách tính | $/tháng |
| --- | --- | --- |
| EKS control plane | $0,10 × 730 | 73,00 |
| EC2: 2 × `m7g.large` | 2 × $0,102 × 730 | 148,92 |
| EBS gốc của node | 2 × 30 GB × $0,096 | 5,76 |
| ALB | $0,0252 × 730 = 18,40; LCU trung bình 0,288 × 730 × $0,008 = 1,68; thêm lúc flash sale (5,76 − 0,288) × 30 × $0,008 = 1,31 | 21,39 |
| NAT: 1 cái + IP + 30 GB (kéo image, gọi cổng thanh toán) | 43,07 + 3,65 + 30 × $0,059 | 48,49 |
| RDS `db.t4g.medium` Multi-AZ + 50 GB | $0,203 × 730 = 148,19; 50 × $0,276 = 13,80 | 161,99 |
| Valkey 2 × `cache.t4g.small` | 2 × $0,0384 × 730 | 56,06 |
| CloudFront trả theo dùng | 657 GB nằm trong 1 TB miễn phí; (32,85 − 10) triệu request × $1,20/triệu | 27,42 |
| WAF | $5 + 6 rule × $1 + 32,85 triệu × $0,60/triệu | 30,71 |
| CloudWatch | 5 GB log × $0,70; lưu 3 tháng 15 GB × $0,03; 10 alarm × $0,10 | 4,95 |
| Lặt vặt | 10 secret 4,00; 1 KMS key 1,00; ECR 5 GB 0,50; Route 53 0,50; S3 50 GB 1,25; SQS dưới 1 triệu request 0; 50 GB giữa hai AZ × $0,02 = 1,00 | 8,25 |
| **Tổng** | | **≈ 587** |

### 11.3. Mức M: khoảng $2.277/tháng

LCU mức M: trung bình 50 RPS × 16 KB = 2,88 GB/giờ, tức 2,88 LCU. Lúc flash sale 1000 RPS × 16 KB × 3.600 = 57,6
GB/giờ, tức 57,6 LCU.

| Hạng mục | Cách tính | $/tháng |
| --- | --- | --- |
| EKS control plane | $0,10 × 730 | 73,00 |
| EC2: node hệ thống 3 × `m7g.large` | 3 × $0,102 × 730 | 223,38 |
| EC2: node app 3 × `m7g.large` | 3 × $0,102 × 730 | 223,38 |
| EC2: thêm 3 node lúc flash sale | 3 × $0,102 × 30 giờ | 9,18 |
| EBS: 6 node × 30 GB + 50 GB cho Prometheus | 230 GB × $0,096 | 22,08 |
| ALB | 18,40; LCU trung bình 2,88 × 730 × $0,008 = 16,82; thêm lúc flash sale (57,6 − 2,88) × 30 × $0,008 = 13,13 | 48,35 |
| NAT: 3 cái + 3 IP + 100 GB | 3 × 43,07 = 129,21; 3 × 3,65 = 10,95; 100 × $0,059 = 5,90 | 146,06 |
| RDS `db.m7g.large` Multi-AZ + 100 GB | $0,468 × 730 = 341,64; 100 × $0,276 = 27,60 | 369,24 |
| Valkey 2 × `cache.t4g.medium` | 2 × $0,076 × 730 | 110,96 |
| Edge: CloudFront gói Premium (đã gồm WAF, Route 53 DNS, nạp log CloudFront) | giá phẳng, so sánh ở mục 11.4 | 1.000,00 |
| CloudWatch | 30 GB log × $0,70 = 21,00; lưu 3 tháng 90 GB × $0,03 = 2,70; 20 alarm = 2,00 | 25,70 |
| Lặt vặt | 10 secret 4,00; 2 KMS key 2,00; ECR 10 GB 1,00; Route 53 0,50; S3 200 GB 5,00; SQS (10 − 1) triệu × $0,40 = 3,60; 500 GB giữa hai AZ × $0,02 = 10,00 | 26,10 |
| **Tổng** | | **≈ 2.277** |

### 11.4. Edge: trả theo dùng hay gói phẳng

Ngoài cách trả theo dùng, CloudFront có **gói phẳng** (flat-rate): trả một số tiền cố định mỗi tháng, gộp cả
CloudFront, WAF, chống DDoS, Route 53 DNS và nạp log. Mỗi gói có hạn mức:

| Gói | Giá/tháng | Request | Dữ liệu | VPC origin |
| --- | --- | --- | --- | --- |
| Free | $0 | 1 triệu | 100 GB | không |
| Pro | $15 | 10 triệu | 50 TB | không |
| Business | $200 | 125 triệu | 50 TB | có |
| Premium | $1.000 | 500 triệu (nâng được tới 6 tỉ với giá cao hơn) | 50 TB | có |

Vượt hạn mức **không bị tính thêm tiền**. Lần đầu vượt tới 3 lần hạn mức trong một tháng thì không ảnh hưởng gì. Nhưng
nếu vượt nhiều và kéo dài nhiều tháng, AWS có thể điều chỉnh chất lượng phân phối, ví dụ phục vụ từ ít edge hơn. Vì
vậy chỉ chọn gói mà mức dùng thường xuyên nằm trong hạn mức.

So sánh cho hai mức:

| | Mức S | Mức M |
| --- | --- | --- |
| Trả theo dùng: CloudFront | $27,42 | (6.570 − 1.024) GB × $0,12 = 665,52 + (328,5 − 10) triệu × $1,20 = 382,20 → $1.047,72 |
| Trả theo dùng: WAF | $30,71 | $5 + $6 + 328,5 × $0,60 = $208,10 |
| **Trả theo dùng: tổng** | **$58,13** | **$1.255,82** |
| Gói Pro ($15) | 32,9 triệu request, gấp 3,3 lần hạn mức; không có VPC origin | không hợp |
| Gói Business ($200) | đắt hơn trả theo dùng | 328,5 triệu request, gấp 2,6 lần hạn mức |
| Gói Premium ($1.000) | đắt hơn nhiều | nằm trong hạn mức, rẻ hơn trả theo dùng $255,82 |
| **Chọn** | **Trả theo dùng** | **Premium** |

Ngoài khoản rẻ hơn, gói phẳng ở mức M còn chặn được rủi ro hoá đơn tăng đột biến khi bị tấn công DDoS: request bị WAF
chặn không tính vào hạn mức.

### 11.5. Tiền đi đâu

| Mức S ($587) | Tỉ lệ | | Mức M ($2.277) | Tỉ lệ |
| --- | --- | --- | --- | --- |
| RDS | 27,6% | | Edge (CloudFront Premium) | 43,9% |
| EC2 + EBS | 26,4% | | EC2 + EBS | 21,0% |
| EKS control plane | 12,4% | | RDS | 16,2% |
| CloudFront + WAF | 9,9% | | NAT | 6,4% |
| Valkey | 9,6% | | Valkey | 4,9% |
| NAT | 8,3% | | EKS control plane | 3,2% |
| ALB | 3,6% | | ALB | 2,1% |
| CloudWatch + lặt vặt | 2,2% | | CloudWatch + lặt vặt | 2,3% |

Hai mức có hai câu chuyện khác nhau:

- **Mức S bị chi phối bởi phần cố định.** Bỏ hết khoản tăng theo tải (CloudFront $27,42, request WAF $19,71, LCU
  $2,99, data qua NAT $1,77) thì vẫn còn khoảng **$535/tháng dù không có ai dùng**. Muốn rẻ hơn nữa phải bỏ bớt thành
  phần, không phải tối ưu máy (xem mục 11.8).
- **Mức M bị chi phối bởi băng thông và số request.** Edge chiếm 44%. Giảm số request và dung lượng ảnh mỗi người dùng
  tiết kiệm nhiều hơn mọi cách tối ưu máy chủ.

### 11.6. Cách giảm chi phí

**Cam kết 1 năm, không trả trước**, áp cho phần chạy cả tháng (node chạy thêm lúc flash sale vẫn trả on-demand):

| | Mức S | Mức M |
| --- | --- | --- |
| EC2 Compute Savings Plan ($0,102 → $0,076, giảm 25,5%) | 2 node × $0,026 × 730 = $37,96 | 6 node × $0,026 × 730 = $113,88 |
| RDS Reserved Instance (giảm 20,9% với `t4g.medium`, 22,9% với `m7g.large`) | ($0,203 − $0,1605) × 730 = $31,03 | ($0,468 − $0,3607) × 730 = $78,33 |
| **Tổng tiết kiệm** | **$68,99** | **$192,21** |
| **Còn lại** | **≈ $518/tháng** | **≈ $2.085/tháng** |

**Giảm số request mỗi người dùng (mức M).** Gói Business $200 có hạn mức 125 triệu request. Muốn vừa, cần
125 triệu ÷ 657 triệu giây-người dùng ≈ 0,19 request/giây mỗi người, so với 0,5 hiện tại. Hai việc giúp đạt mức đó:

- Gộp 4 API mỗi màn hình thành 1–2 lần gọi qua một lớp BFF (backend for frontend).
- Cho app giữ ảnh lâu bằng `Cache-Control: max-age=31536000, immutable`, đổi tên file mỗi khi ảnh đổi.

Nếu đạt 0,15 request/giây mỗi người: 657 triệu × 0,15 = 98,6 triệu request, nằm trong gói Business, **tiết kiệm
$800/tháng**.

**Môi trường staging:** tắt node và dừng RDS ngoài giờ làm việc (xem mục 11.8).

### 11.7. Giả định thay đổi thì edge mức M đổi thế nào

| Thay đổi | Trả theo dùng | Gói phẳng phù hợp |
| --- | --- | --- |
| Như giả định (10 KB/s, 0,5 request/s mỗi người) | $1.255,82 | Premium $1.000 |
| Ảnh nặng gấp đôi (20 KB/s) | 9.216 GB × $0,12 + 2.900 GB × $0,10 = 1.395,92 + 382,20 + 208,10 = $1.986,22 | Premium $1.000 (13,1 TB, dưới 50 TB) |
| Request gấp đôi (1 request/s mỗi người) | 665,52 + 647 triệu × $1,20 = 776,40 + ($11 + 657 × $0,60 = 405,20) = $1.847,12 | Premium mức 750 triệu: $1.450 |
| Request còn 0,15/s mỗi người | 665,52 + 88,6 triệu × $1,20 = 106,26 + ($11 + 98,6 × $0,60 = 70,13) = $841,91 | Business $200 |

Bảng này cho thấy vì sao phải đo hành vi người dùng thật trước khi chốt ngân sách: cùng 1000 CCU, chi phí edge chênh
nhau từ $200 tới $1.986 tuỳ app tải bao nhiêu ảnh và gọi bao nhiêu request.

### 11.8. Chưa tính trong bảng

- **Môi trường staging.** Dựng riêng, thu nhỏ, tốn khoảng **$377/tháng**: EKS $73; 2 × `t4g.large` $123,81; RDS
  `db.t4g.medium` Single-AZ + 20 GB $77,22; Valkey 1 node $28,03; NAT + IP $46,72; ALB $18,40; lặt vặt khoảng $10.
  Tắt node và dừng RDS ngoài giờ làm việc giảm được khoảng một nửa phần máy.
- **EKS hết hạn hỗ trợ chuẩn.** Mỗi bản Kubernetes trên EKS được hỗ trợ chuẩn 14 tháng. Quá hạn mà chưa nâng version
  thì phí cluster cộng thêm $0,50/giờ, từ $73 lên $438/tháng. Nghĩa là phải nâng version đều đặn, và đó là công của
  người vận hành.
- **Nhân sự.** EKS cần người biết vận hành Kubernetes. Ở mức S, nếu team chưa quen Kubernetes, ECS là lựa chọn rẻ hơn
  cả về tiền (không có phí control plane $73, không tốn CPU chạy addon) lẫn công vận hành.
- Gói hỗ trợ trả phí của AWS, thuế, tên miền, dịch vụ ngoài AWS (cổng thanh toán, email, SMS).

### 11.9. Quy ra tiền Việt

Tỷ giá giả định 26.000 đ/USD:

| | On-demand | Có cam kết 1 năm |
| --- | --- | --- |
| Mức S | ≈ 15,3 triệu đ/tháng | ≈ 13,5 triệu đ/tháng |
| Mức M | ≈ 59,2 triệu đ/tháng | ≈ 54,2 triệu đ/tháng |

---

## 12. Khi vượt 1000 CCU

Thêm thành phần khi có tín hiệu đo được, không thêm trước:

| Tín hiệu | Việc làm |
| --- | --- |
| CPU của RDS trên 60% giờ cao điểm, phần lớn là truy vấn đọc | Thêm read replica, hoặc chuyển sang Aurora PostgreSQL (`db.r7g.large` $0,333/giờ mỗi instance) có instance đọc riêng |
| Kết nối DB lên vài trăm, hoặc có Lambda gọi DB | RDS Proxy ($0,018 mỗi vCPU của DB mỗi giờ) |
| Dữ liệu qua NAT tới một dịch vụ AWS trên khoảng 600 GB/tháng | Interface endpoint cho dịch vụ đó; hoà vốn ở $28,47 ÷ $0,049 ≈ 581 GB/tháng cho mỗi endpoint |
| Request CloudFront trên 500 triệu/tháng | Premium mức cao hơn (750 triệu request: $1.450) |
| Tải nhảy hơn 10 lần trong vài giây ở mức hàng chục nghìn RPS | Đặt trước capacity của ALB (LCU Reservation, $0,008/LCU-giờ) |
| Cần khôi phục trong vài phút khi cả region sập | Region thứ hai, Aurora Global Database, Route 53 failover |

---

## 13. Tóm tắt quyết định thiết kế

1. **Đổi CCU ra RPS và GB trước.** Mọi con số về máy và tiền đều suy ra từ đó.
2. **Chặn tải ở xa nhất.** CloudFront và WAF đứng trước; ALB là internal, chỉ nhận traffic qua VPC origin.
3. **ALB do Terraform sở hữu.** Controller chỉ gắn Pod qua `TargetGroupBinding`.
4. **target-type ip + readiness gate**, cùng `preStop`, `deregistration_delay` và grace period khớp nhau, cho deploy
   không downtime.
5. **Dự trù đủ để mất 1 node (S) hoặc 1 AZ (M)**, và trải Pod đều qua các AZ.
6. **Scale trước cho flash sale.** Autoscaling chỉ dùng cho dao động thường ngày.
7. **Tồn kho hàng flash sale trừ trong Valkey, đơn hàng đi qua SQS.**
8. **Không thêm thứ chưa cần ở quy mô này:** RDS Proxy, interface endpoint, Aurora, multi-region.
9. **Chi phí:** mức S bị phần cố định chi phối, mức M bị băng thông và request chi phối. Tối ưu request và ảnh trước
   khi tối ưu máy.
10. **Thay mọi giả định bằng số đo thật**: hành vi người dùng (mục 1.1) và sức chịu của một Pod (mục 4).

---

## 14. Nguồn

- AWS Price List API, vùng `ap-southeast-1`, lấy ngày 2026-10-12:
  `https://pricing.us-east-1.amazonaws.com/offers/v1.0/aws/<dịch-vụ>/current/ap-southeast-1/index.json` với các dịch
  vụ AmazonEC2, AmazonEKS, AWSELB, AmazonVPC, AmazonRDS, AmazonElastiCache, AmazonCloudWatch, awswaf,
  AWSQueueService, AWSSecretsManager, AmazonECR, AmazonS3; AmazonCloudFront và AmazonRoute53 dùng file toàn cầu.
  Giá Savings Plan lấy từ
  `https://pricing.us-east-1.amazonaws.com/savingsPlan/v1.0/aws/AWSComputeSavingsPlan/current/region_index.json`.
- [CloudFront flat-rate pricing plans](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/flat-rate-pricing-plan.html)
- [CloudFront FAQ: free tier khi trả theo dùng](https://aws.amazon.com/cloudfront/faqs/)
- [CloudFront VPC origins](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/private-content-vpc-origins.html)
- [AWS Load Balancer Controller: TargetGroupBinding](https://kubernetes-sigs.github.io/aws-load-balancer-controller/latest/guide/targetgroupbinding/targetgroupbinding/)
- [AWS Load Balancer Controller: Pod readiness gate](https://kubernetes-sigs.github.io/aws-load-balancer-controller/latest/deploy/pod_readiness_gate/)
- [Kubernetes: Pod lifecycle](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/)
- [KEDA: Cron scaler](https://keda.sh/docs/latest/scalers/cron/)
- [Amazon EKS: Kubernetes version lifecycle](https://docs.aws.amazon.com/eks/latest/userguide/kubernetes-versions.html)
