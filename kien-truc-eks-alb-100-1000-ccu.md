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
> `$2.298` là hai nghìn hai trăm chín mươi tám USD. Khối code giữ cú pháp gốc.
>
> **Cập nhật 2026-10-13** theo [review](review-kien-truc-eks-alb-va-ngan-sach-lab.md): sửa cấu hình target group, tính
> lại capacity cho rollout, làm lại luồng đặt hàng flash sale để PostgreSQL quyết định tồn kho, cộng phần traffic flash
> sale thống nhất cho mọi dịch vụ, bổ sung điều kiện VPC origin và đường quản trị cluster, thêm môi trường lab ở mục 12.
> Bản trước khi cập nhật: [kien-truc-eks-alb-100-1000-ccu.truoc-review.md](kien-truc-eks-alb-100-1000-ccu.truoc-review.md).

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
12. [Môi trường lab giữ kết cấu M](#12-môi-trường-lab-giữ-kết-cấu-m)
13. [Khi vượt 1000 CCU](#13-khi-vượt-1000-ccu)
14. [Tóm tắt quyết định thiết kế](#14-tóm-tắt-quyết-định-thiết-kế)
15. [Nguồn](#15-nguồn)

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
nên giả định **CCU trung bình lúc thường (chưa tính flash sale) bằng 25% CCU đỉnh**. Đỉnh 1000 CCU thì trung bình
250 CCU. Một tháng có
730 × 3.600 = 2.628.000 giây, nên tổng "giây-người dùng" là 250 × 2.628.000 = 657 triệu. Từ đó:

- API request mỗi tháng: 657 triệu × 0,2 = 131,4 triệu, trung bình 50 RPS.
- Request qua CloudFront mỗi tháng: 657 triệu × 0,5 = 328,5 triệu.
- Dữ liệu ra internet mỗi tháng: 657 triệu × 10 KB = 6.570 GB.

Các số trên là traffic **lúc thường**. Flash sale được **cộng riêng**, không nằm trong mức trung bình. Mỗi tháng có 10
đợt, mỗi đợt giữ capacity 3 giờ (30 phút scale trước + 2,5 giờ bán), tức 30 giờ giữ capacity nhưng chỉ 25 giờ có tải,
bằng 90.000 giây. Trong lúc bán, giả định CCU ở mức đỉnh, mỗi người gọi API 1 request/giây và vẫn tải ảnh, JS
0,3 request/giây. Phần tăng thêm so với lúc thường, tính cho 1000 CCU:

- API: (1000 × 1 − 50) × 90.000 = **85,5 triệu** request.
- Ảnh, JS: (1000 − 250) × 0,3 × 90.000 = **20,25 triệu** request.
- Dữ liệu ra: 85,5 triệu × 15 KB + 20,25 triệu × 23,3 KB ≈ **1.755 GB**. Mỗi request ảnh, JS trung bình
  (200 − 4 × 15) ÷ 6 ≈ 23,3 KB.

Mọi bảng chi phí ở mục 11 cộng phần này **thống nhất** cho API, ALB, CloudFront, WAF, log và dữ liệu ra.

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
| CCU trung bình lúc thường | 25 | 250 | 25% đỉnh |
| Giờ flash sale mỗi tháng | 30 giờ giữ capacity, 25 giờ có tải | như S | 10 đợt × (30 phút scale trước + 2,5 giờ bán) |
| API request lúc thường | 13,14 triệu | 131,4 triệu | CCU trung bình × 0,2 × 2.628.000 |
| API request tăng thêm do flash sale | 8,55 triệu | 85,5 triệu | (CCU đỉnh × 1 − CCU trung bình × 0,2) × 90.000 |
| **API request cả tháng** | **21,69 triệu** | **216,9 triệu** | hai dòng trên |
| **Request qua CloudFront cả tháng** | **43,43 triệu** | **434,25 triệu** | lúc thường × 0,5 req/s + phần flash sale (API và ảnh) |
| **Dữ liệu ra internet cả tháng** | **832,5 GB** | **8.325 GB** | lúc thường 10 KB/s + phần flash sale |

Yêu cầu chung cho cả hai mức:

- p99 latency của API dưới 300 ms, tức 99% request API phản hồi trong 300 ms. Đo cả ở client lẫn `TargetResponseTime`
  của ALB, mỗi lần đo đủ vài nghìn request. Với đặt hàng, chỉ tiêu này áp cho lúc API trả `202`; thời gian để đơn được
  xử lý xong qua SQS đo riêng.
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
│                   │        (M thêm: Karpenter, KEDA, Prometheus,│     │
│                   │        Grafana, Argo CD, EBS CSI driver)    │     │
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
| Cấp node | Managed node group; không tự tăng node, đổi desired size thủ công | Karpenter |
| Pod `api` | 2 (HPA tối đa 3) | 3 lúc thường, 15 lúc flash sale |
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
- VPC origin có bốn điều kiện mạng phải chuẩn bị trước:
  - VPC phải gắn Internet Gateway. IGW chỉ để đánh dấu VPC nhận traffic từ internet, không dùng để định tuyến vào
    ALB, không cần sửa route table.
  - Subnet private còn ít nhất một địa chỉ IPv4 trống để CloudFront tạo ENI.
  - Security group của ALB cho phép inbound từ managed prefix list của CloudFront, hoặc từ security group
    `CloudFront-VPCOrigins-Service-SG` mà CloudFront tạo ra (chặt hơn). Rule ALB → Pod do controller mở (mục 5.1)
    chưa đủ cho cả luồng.
  - Inbound NACL không được đánh giá cho traffic từ CloudFront, nhưng outbound NACL thì có: phải cho phép cổng tạm
    1024–65535 trên đường trả về.

  VPC origin không hỗ trợ gRPC. App phải phục vụ HTTP/1.1 hoặc HTTP/2 thường.
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
0,5 vCPU và 1 GiB RAM, giữ được p99 dưới 300 ms tới **100 RPS**, khi đó CPU khoảng 70%. AWS không bảo đảm con số
này; p99 phụ thuộc query và index, tỉ lệ trúng cache, TLS, mạng, connection pool, kích thước payload và tỉ lệ thao
tác ghi.

Capacity phải đủ cho cả lúc **rollout**: với `maxSurge: 25%`, `maxUnavailable: 0` (mục 8), Deployment tạo thêm Pod
mới trước khi xoá Pod cũ, nên cần chừa chỗ cho các Pod tăng thêm này.

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
4. HPA cho `api` tối đa **3 Pod**. Lúc 3 Pod mà đang rollout thì có thêm 1 Pod mới: 4 × 0,5 + 0,25 + 0,6 + 0,6 =
   3,45 vCPU, vừa 3,8. Nếu cho HPA lên 4 Pod, rollout cần 3,95 vCPU, vượt 3,8; Pod mới nằm chờ và rollout đứng lại vì
   managed node group không tự thêm node.
5. Khi mất 1 node, phần tối thiểu (1 `api`, 1 `worker`, mỗi addon 1 bản, DaemonSet) cần khoảng 1,45 vCPU, vừa với
   1,9 vCPU của node còn lại. Managed node group tự dựng node thay node hỏng trong vài phút để giữ đúng số node
   mong muốn, nhưng không tự thêm node khi Pod nằm chờ (mục 7.1).

### 4.2. Mức M

1. Lúc thường cần 200 ÷ 100 = 2 Pod. Để chịu mất 1 AZ, hai AZ còn lại phải tự gánh 2 Pod, tức mỗi AZ 1 Pod. Với 3
   AZ: **3 Pod**.
2. Lúc flash sale cần 1000 ÷ 100 = 10 Pod. Chịu mất 1 AZ: mỗi AZ 10 ÷ 2 = 5 Pod. Với 3 AZ: **15 Pod**, tức 7,5 vCPU.
3. Node cho app lúc thường: 3 × `m7g.large`, mỗi AZ một node. Mỗi node chạy 1 `api` (0,5) + `worker` (0,25) +
   DaemonSet (0,3), khoảng 1,05 vCPU. Phần trống còn lại đủ để HPA tăng lên 6 Pod mà chưa cần node mới.
4. Lúc flash sale cần 15 `api` + 4 `worker` = 8,5 vCPU. Mỗi node dùng được khoảng 1,6 vCPU (1,9 trừ DaemonSet), 6
   node cho 9,6 vCPU. Vậy Karpenter thêm **3 node**, mỗi AZ một.
5. Deploy trong lúc flash sale: `maxSurge: 25%` của 15 Pod làm tròn lên thành 4 Pod mới, đưa nhu cầu lên 10,5 vCPU,
   vượt 9,6. Karpenter sẽ cấp thêm một node nên rollout không đứng, nhưng chậm thêm 1–2 phút và tốn thêm giờ máy.
   `limits` của NodePool phải chừa chỗ cho node này. Tốt nhất **không deploy trong khung flash sale**.
6. Node hệ thống: 3 × `m7g.large`, mỗi AZ một node. Karpenter không được chạy trên node do chính nó tạo ra, nên cần
   nhóm node riêng. Nhóm này chạy Karpenter, Load Balancer Controller, CoreDNS, KEDA, metrics-server, External
   Secrets, EBS CSI driver, Prometheus (khoảng 1 vCPU, 4 GiB), Grafana và Argo CD (ước lượng 0,5 vCPU, 1,5 GiB, phải
   đo lại), tổng khoảng 3,6 vCPU và 9,8 GiB. 3 node trừ DaemonSet còn khoảng 4,8 vCPU nên vẫn vừa.
7. Prometheus lưu dữ liệu trên EBS gp3 50 GB qua EBS CSI driver. Ổ EBS gắn với một AZ, nên Pod Prometheus chỉ chạy
   được ở AZ của ổ; mất AZ đó thì Prometheus ngừng cho tới khi AZ hồi phục, còn app vẫn chạy.

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

- Thay cả cluster (blue/green lúc nâng version EKS) chỉ cần đổi trọng số giữa hai target group. Cách đổi này vẫn
  phải kiểm thử trước khi dùng thật.
- Hạ tầng (Terraform) và app (manifest K8s) có ranh giới rõ.

Đổi lại, thêm route mới phải sửa Terraform thay vì sửa Ingress. Đặt `lifecycle { prevent_destroy = true }` cho ALB,
listener và target group để một lần `terraform apply` nhầm không xoá cửa chính.

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
| Health check | `/healthz`, mỗi 10 giây; healthy threshold 2 | Target mới đăng ký chỉ cần **1 lần** đạt là healthy, nên nhận traffic trong khoảng 10 giây sau khi được đăng ký. Threshold 2 chỉ áp khi target phục hồi từ unhealthy |
| `deregistration_delay` | 30 giây | Mặc định 300 giây làm rollout rất chậm; 30 khớp với mục 8 |
| Thuật toán chia tải | `least_outstanding_requests` | Gửi request cho Pod đang ít việc nhất |
| `slow_start` | **Không bật** | ALB không cho bật slow start cùng `least_outstanding_requests`. Nếu app khởi động chậm (JVM) và cần tăng tải dần, chuyển sang `round_robin` + `slow_start` 30 giây, hoặc cho app tự làm nóng trước khi readiness probe đạt |

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

Tính tay cho ECR. Kéo image cần 2 endpoint (`ecr.api` và `ecr.dkr`), tốn 2 × $28,47 = $56,94/tháng. Mỗi GB đi qua
endpoint thay vì qua NAT chỉ tiết kiệm $0,059 − $0,01 = $0,049. Muốn bù được $56,94 thì phải có
56,94 ÷ 0,049 ≈ **1.162 GB/tháng** đi qua hai endpoint đó.

Thực tế còn xa hơn: ECR lưu các layer của image trên S3, nên khi đã có gateway endpoint cho S3 thì phần lớn dung lượng
image (các layer) không đi qua NAT. Qua NAT chỉ còn lệnh gọi API ECR và manifest, rất nhỏ. Vì vậy **chưa dùng interface
endpoint**, trừ khi yêu cầu bảo mật cấm traffic tới dịch vụ AWS đi ra internet. Image lấy từ registry ngoài AWS (Docker
Hub, quay.io, ghcr.io) vẫn đi qua NAT; khoản này đã nằm trong phần GB qua NAT ở mục 11.

---

## 7. Compute và autoscaling

### 7.1. Mức S

- Một managed node group, 2 node `m7g.large`, chạy chung addon và app. Đặt `maxSize: 3` chỉ là trần: managed node
  group thay node hỏng để giữ đúng `desiredSize`, nhưng **không tự thêm node khi Pod nằm chờ**. Muốn tự động phải cài
  Cluster Autoscaler (chạy trong cluster, nhận quyền IAM qua Pod Identity). Nếu không cài, trước đợt bán lớn hãy tăng
  `desiredSize` bằng tay qua AWS API.
- HPA cho `api`: mục tiêu CPU 60%, tối thiểu 2, tối đa 3 Pod, để rollout vẫn vừa 2 node (mục 4.1).
- Không cần Karpenter và KEDA: đỉnh flash sale 100 RPS vừa với 2 Pod sẵn có.

### 7.2. Mức M

- **Node hệ thống:** managed node group 3 × `m7g.large`, có taint để chỉ addon chạy trên đó.
- **Node app:** Karpenter, cho dùng các dòng Graviton (`m7g`, `c7g`). `api` chạy on-demand, `worker` cho phép chạy
  spot. `limits` của NodePool phải chừa đủ cho 6 node lúc flash sale cộng 1 node cho Pod tăng thêm khi rollout
  (mục 4.2).
- **`worker`:** KEDA scale theo số message đang chờ trong SQS.
- **`api`:** KEDA kết hợp hai điều kiện, theo lịch (scale trước flash sale) và theo CPU.

Phải scale trước vì thời gian từ lúc tải tăng tới lúc Pod mới thật sự nhận traffic khá dài. Tính tay:

| Bước | Ước lượng |
| --- | --- |
| metrics-server lấy số liệu, HPA quyết định tăng Pod | 15–30 giây |
| Karpenter thấy Pod chờ, gọi EC2, node mới vào cluster | 60–90 giây |
| Kéo image (node mới chưa có sẵn) | 10–30 giây |
| App khởi động, readiness probe qua | 10–30 giây |
| Lần health check đầu tiên của ALB đạt (target mới chỉ cần 1 lần), readiness gate chuyển True | 5–15 giây |
| **Tổng** | **khoảng 2–3 phút** |

Flash sale đẩy tải từ 200 lên 1000 RPS trong vài giây. Nếu chỉ chờ autoscaling, trong 2–3 phút đầu, đúng lúc đông
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
trong khoảng 3–15 Pod. Ví dụ chỉ có lịch thứ Sáu hàng tuần; mục 1.2 giả định 10 đợt mỗi tháng, nên các đợt khác cần
thêm trigger `cron` tương ứng.

---

## 8. Deploy không downtime

Cấu hình rollout: `maxSurge: 25%`, `maxUnavailable: 0`. Namespace đã bật chèn readiness gate (mục 5.1). Capacity cho
các Pod tăng thêm khi rollout đã tính ở mục 4.

Các timeline dưới đây là **ước lượng** để chọn tham số, không phải bảo đảm zero downtime. Xác nhận bằng cách rollout
khi đang chạy tải và đếm lỗi `HTTPCode_ELB_5XX_Count`, `HTTPCode_Target_5XX_Count` của ALB.

### 8.1. Pod mới lên: readiness gate lo phần này

| Thời điểm | Sự kiện |
| --- | --- |
| t = 0 | Pod v2 được tạo |
| t ≈ 20 giây | Readiness probe qua, `ContainersReady=True` |
| t ≈ 21 giây | Controller đăng ký IP Pod vào target group, target ở trạng thái `initial` |
| t ≈ 21–31 giây | Lần health check đầu tiên đạt (chu kỳ 10 giây), target chuyển `healthy`. Target mới chỉ cần 1 lần đạt |
| t ≈ 25–35 giây | Controller đặt gate thành True, Pod có `Ready=True`, Deployment mới xoá một Pod v1 |

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
      - maxSkew: 1
        topologyKey: kubernetes.io/hostname
        whenUnsatisfiable: DoNotSchedule
        nodeTaintsPolicy: Honor
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
| t ≈ 1–5 giây | Controller huỷ đăng ký target, ALB ngừng gửi request mới (target `draining`). `deregistration_delay` **bắt đầu tính từ lúc này**, không phải từ t = 0. Trong vài giây ALB chưa áp dụng xong, request vẫn tới và Pod vẫn phục vụ vì chưa nhận SIGTERM |
| t = 20 giây | `preStop` xong, app nhận SIGTERM, ngừng nhận kết nối mới và xử lý nốt request đang dở (≤ 10 giây) |
| t ≈ 30 giây | App thoát |
| t ≈ 31–35 giây | Hết `deregistration_delay`, target chuyển `unused`. ALB kết thúc sớm hơn nếu target không còn kết nối nào |

Ba quy tắc rút ra từ timeline:

- `preStop` phải dài hơn thời gian ALB cần để thật sự ngừng gửi request mới.
- `deregistration_delay`, tính từ lúc huỷ đăng ký, không ngắn hơn `preStop` cộng thời gian xử lý nốt request. Đây là
  giới hạn trên chứ không phải thời gian chờ cố định. Nếu app đóng kết nối khi request còn dở, client nhận lỗi 5xx.
- `terminationGracePeriodSeconds` lớn hơn `preStop` cộng thời gian xử lý nốt request, có biên an toàn. App phải xử lý
  SIGTERM đúng: ngừng nhận kết nối mới, trả nốt request đang dở rồi mới thoát.

Các cấu hình bổ trợ trong YAML trên:

- **PodDisruptionBudget:** chỉ điều tiết việc rút Pod chủ động qua eviction, ví dụ khi Karpenter gom node hoặc khi nâng
  cấp node. PDB **không** điều khiển rollout của Deployment (rollout do `maxSurge`/`maxUnavailable` quyết định) và
  **không** ngăn được node hay AZ hỏng. PDB đếm Pod theo condition `Ready`, mà `Ready` đã gồm readiness gate. Với
  `maxUnavailable: 1`, khi mọi Pod đang healthy thì Pod đầu tiên bị rút ngay, chưa chờ Pod thay thế; các lần rút sau mới
  phải chờ Pod thay thế được ALB xác nhận healthy.
- **`topologySpreadConstraints`:** ràng buộc thứ nhất chia đều Pod qua các AZ, khớp với phép tính ở mục 4. Trải theo AZ
  chưa bảo đảm trải theo node, vì một AZ có thể có nhiều node và managed node group không cam kết luôn có đúng một node
  mỗi AZ. Ràng buộc thứ hai chia đều theo node để chịu được mất một node. `nodeTaintsPolicy: Honor` loại các node hệ
  thống có taint khỏi phép đếm. Mặc định Kubernetes bỏ qua taint khi đếm, nên nếu Pod không có nodeAffinity chọn riêng
  node app, các node hệ thống (luôn 0 Pod `api`) sẽ kéo mức tối thiểu về 0 và Pod mới không xếp được.
- **Health check và readiness probe chỉ kiểm tra bản thân Pod, không kiểm tra DB.** Nếu kiểm tra DB, chỉ cần DB chậm
  là mọi Pod cùng bị coi là hỏng một lúc, và cả hệ thống sập theo DB. Đổi lại, lỗi DB và các dependency phải có alarm
  riêng (mục 10).

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
đếm rate-limit, và **cổng chặn nhanh cho hàng flash sale**.

Valkey sao chép từ node chính sang bản sao **bất đồng bộ**. Khi chuyển sang bản sao, các lệnh ghi vừa được node chính
xác nhận nhưng chưa kịp sao chép sẽ mất. ElastiCache chỉ bảo đảm không mất dữ liệu đã commit khi bật durability
(Valkey 9.0 trở lên), mà thiết kế này không bật. Vì vậy Valkey **không phải nơi quyết định tồn kho**; nơi quyết định là
PostgreSQL (mục 9.3).

### 9.3. Đặt hàng flash sale: Valkey chặn nhanh, PostgreSQL quyết định

Cách đơn giản nhất là trừ tồn kho thẳng trong PostgreSQL:

```sql
UPDATE products SET stock = stock - 1 WHERE id = 42 AND stock > 0;
```

Câu lệnh này đúng, không bán quá tồn, vì PostgreSQL khoá dòng trong lúc cập nhật. Vấn đề nằm ở chính cái khoá: mọi
lệnh trên cùng sản phẩm phải xếp hàng. Tính tay ở mức M: 1000 người cùng bấm "Mua" một sản phẩm hot trong cùng một
giây. Mỗi lệnh giữ khoá khoảng 1 ms (gồm cả ghi log giao dịch), nên người cuối hàng chờ khoảng 1000 × 1 ms = 1 giây,
vượt mục tiêu p99 300 ms nếu lệnh này nằm trên đường request.

Với Valkey, một lệnh chạy trong bộ nhớ khoảng 0,1 ms, nên 1000 lệnh xong trong khoảng 0,1 giây. Hai con số 1 ms và
0,1 ms là giả định, phải đo lại. Thiết kế dùng Valkey để **chặn nhanh** trên đường request, còn câu `UPDATE` có điều
kiện ở trên vẫn chạy, nhưng trong `worker`, ngoài đường request:

1. Trước flash sale, nạp số tồn từ PostgreSQL vào Valkey: `SET stock:42 500`.
2. API chạy một Lua script kiểm tra và trừ trong cùng một lệnh: còn hàng thì trừ 1 và cho đi tiếp, hết hàng thì trả
   "hết hàng" ngay. Script thay cho cặp `DECR` rồi `INCR` khi âm.
3. API gửi message "đơn mới" vào SQS, kèm idempotency key (mã duy nhất của lần bấm), rồi trả `202 Đang xử lý`.
   `202` chỉ có nghĩa **đã nhận yêu cầu**, chưa xác nhận giữ được hàng; app hỏi lại trạng thái đơn hoặc nhận thông
   báo.
4. `worker` đọc SQS và làm trong **một transaction PostgreSQL**:
   - `INSERT` đơn với cột `idempotency_key` có ràng buộc `UNIQUE`. Nếu trùng, tức message được giao lại, thì bỏ qua.
   - Nếu là đơn mới, chạy `UPDATE ... WHERE id = 42 AND stock > 0`. Không trúng dòng nào thì đánh dấu đơn
     "hết hàng".
   - Commit, rồi mới gọi cổng thanh toán, kèm idempotency key của cổng thanh toán để thử lại không trừ tiền hai lần.

**Vì sao không bán quá tồn:** quyết định cuối cùng là câu `UPDATE` có điều kiện trong PostgreSQL. Valkey chỉ lọc bớt
yêu cầu. SQS Standard có thể giao một message nhiều lần, nhưng ràng buộc `UNIQUE` chặn đơn trùng.

**Các chỗ hở phải xử lý:**

- API chết giữa bước 2 và 3, hoặc `SendMessage` lỗi hay hết thời gian chờ: Valkey đã trừ nhưng không có message.
- Valkey chuyển sang bản sao: có thể mất vài lần trừ vừa xong.

Hai trường hợp này làm số trong Valkey lệch với PostgreSQL theo cả hai hướng. Lệch thấp thì hàng còn mà báo hết; lệch
cao thì nhận thêm yêu cầu, nhưng `worker` sẽ từ chối ở PostgreSQL nên vẫn không bán quá tồn. Cần một job đối soát đặt
lại `stock:42` theo PostgreSQL trong và sau đợt bán, cùng quy định thời hạn giữ chỗ, hoàn tồn khi thanh toán lỗi hoặc
quá hạn.

Dòng nóng trong PostgreSQL vẫn bị khoá, nhưng ở `worker`: thông lượng khoảng 1.000 lệnh mỗi giây theo giả định 1 ms,
đủ cho 500 suất hàng, và không ảnh hưởng p99 của API. Hàng đợi tách việc gọi cổng thanh toán (chậm, có thể lỗi và phải
thử lại) ra khỏi request của người dùng.

Nếu nghiệp vụ cần **xác nhận giữ hàng trước khi trả `202`**, ghi bản giữ chỗ và một bản ghi outbox trong cùng một
transaction PostgreSQL, rồi `worker` chuyển outbox sang SQS. Khi đó khoá dòng nóng nằm lại trên đường request, nên phải
load test riêng với một sản phẩm hot trước khi dùng ở mức M.

Hàng thường, không thuộc flash sale, vẫn trừ tồn bằng câu `UPDATE` ở trên.

---

## 10. Bảo mật, quan sát, CI/CD, dự phòng

| Lớp | Mức S | Mức M |
| --- | --- | --- |
| Bảo mật | WAF (rate-limit, managed rules); EKS API endpoint private; Pod Identity riêng cho từng service; Secrets Manager + External Secrets Operator; security group riêng cho RDS và Valkey | Như S, thêm NetworkPolicy (VPC CNI hỗ trợ sẵn) giữa các service |
| Truy cập quản trị | API endpoint private chỉ vào được từ mạng đã nối với VPC (VPN, Direct Connect hoặc máy trong VPC). Phải xác định đường này trước khi dựng; nếu phải dựng mới thì cộng vào ngân sách | Như S. Thao tác thường ngày đi qua Git (Argo CD) và AWS API, nên ít khi cần `kubectl` (mục 12.3) |
| Quan sát | Metric có sẵn của ALB, RDS, ElastiCache, SQS trên CloudWatch (miễn phí); log app qua Fluent Bit vào CloudWatch Logs, mỗi API request khoảng 300 byte log (JSON gọn, không ghi body); khoảng 10 alarm | Như S, thêm Prometheus + Grafana tự host trên node hệ thống cho metric của Pod (ổ EBS qua EBS CSI driver); khoảng 20 alarm |
| CI/CD | GitHub Actions build image `arm64`, đẩy lên ECR, `helm upgrade` từ runner tự host trong VPC; runner GitHub-hosted không vào được API private. Terraform cho hạ tầng | Như S, nhưng thay `helm upgrade` bằng Argo CD (GitOps, kéo từ Git): CI chỉ build, đẩy image và commit, không cần vào API EKS |
| Sao lưu, dự phòng | RDS backup tự động 7 ngày, khôi phục về thời điểm bất kỳ (PITR); một region | Như S, thêm copy snapshot sang region khác nếu cần |
| Kiểm chứng | Load test 1,5 lần mức đỉnh trước mỗi đợt mở bán lớn; rollout trong lúc chạy tải để đếm 5xx; thử giao trùng message, lỗi gửi SQS và failover Valkey để chắc không có đơn trùng hay âm tồn | Như S, thêm diễn tập tắt một AZ bằng AWS FIS |

Chưa tính trong log: log control plane EKS, access log của ALB và WAF, query log của DB, custom metric và phí truy vấn
log. Bật thêm loại nào thì cộng vào ngân sách CloudWatch.

**Đo p99:** đo ở client (gồm cả mạng và CloudFront) và ở `TargetResponseTime` của ALB (chỉ từ ALB tới Pod), mỗi lần đo
đủ vài nghìn request. Đo riêng thời gian từ lúc nhận `202` tới khi đơn xử lý xong. Lúc RDS hoặc Valkey failover có gián
đoạn, nên không yêu cầu giữ p99 300 ms trong khoảng đó; đo thời gian phục hồi và độ đúng của dữ liệu.

Các alarm nên có ngay từ đầu:

- ALB: `HTTPCode_Target_5XX_Count` và `HTTPCode_ELB_5XX_Count` tăng, `TargetResponseTime` p99 vượt 300 ms,
  `HealthyHostCount` dưới 2.
- RDS: `CPUUtilization`, `FreeStorageSpace`, `DatabaseConnections`, và `CPUCreditBalance` với mức S.
- ElastiCache: `EngineCPUUtilization`, `DatabaseMemoryUsagePercentage`.
- SQS: `ApproximateAgeOfOldestMessage`, tức đơn hàng đang chờ quá lâu.
- Lỗi kết nối DB và cổng thanh toán từ log app (health check không bắt được các lỗi này, mục 8.2), và độ lệch tồn kho
  giữa Valkey và PostgreSQL do job đối soát đẩy lên (mục 9.3).

---

## 11. Chi phí vận hành

### 11.1. Bảng giá đơn vị đã dùng

Tất cả lấy từ AWS Price List API ngày 2026-10-12, vùng Singapore, trừ dòng KMS lấy theo giá niêm yết chung. Giá
on-demand của EC2, RDS, ElastiCache, ALB, NAT, EBS và IPv4 đã được đối chiếu lại theo SKU ngày 2026-10-13. Giá
Savings Plan và Reserved Instance chưa được đối chiếu lại theo offer và kỳ hạn cụ thể; kiểm tra trước khi mua.

| Hạng mục | Đơn giá |
| --- | --- |
| EKS cluster | $0,10/giờ ($0,60/giờ nếu version hết hạn hỗ trợ chuẩn) |
| EC2 `m7g.large` (2 vCPU, 8 GiB) | $0,102/giờ; Compute Savings Plan 1 năm không trả trước: $0,076/giờ |
| EBS gp3 | $0,096/GB-tháng |
| ALB | $0,0252/giờ + $0,008/LCU-giờ |
| NAT Gateway | $0,059/giờ + $0,059/GB |
| IP public (IPv4) | $0,005/giờ |
| VPC interface endpoint | $0,013/giờ mỗi AZ + $0,01/GB |
| RDS PostgreSQL `db.t4g.small` Multi-AZ (lab, mục 12) | $0,102/giờ |
| RDS PostgreSQL `db.t4g.medium` Multi-AZ | $0,203/giờ; Reserved 1 năm không trả trước: $0,1605/giờ |
| RDS PostgreSQL `db.m7g.large` Multi-AZ | $0,468/giờ; Reserved 1 năm không trả trước: $0,3607/giờ |
| Ổ RDS gp3 Multi-AZ | $0,276/GB-tháng |
| ElastiCache Valkey `cache.t4g.micro` / `cache.t4g.small` / `cache.t4g.medium` | $0,0192 / $0,0384 / $0,076 mỗi giờ |
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

### 11.2. Mức S: khoảng $607/tháng

Trước khi đọc dòng ALB: **LCU** là đơn vị đo tải của ALB. Mỗi giờ, số LCU là **giá trị lớn nhất của bốn chiều**: 25
kết nối mới mỗi giây, 3.000 kết nối đang mở mỗi phút, 1 GB dữ liệu xử lý mỗi giờ, 1.000 lần đánh giá rule mỗi giây
(10 rule đầu miễn phí). Traffic đi qua CloudFront dùng lại kết nối và ALB có ít rule, nên giả định chiều quyết định là
dữ liệu xử lý: 1 LCU = 1 GB mỗi giờ. Khi chạy thật, kiểm tra giả định này bằng metric `ConsumedLCUs` của ALB.

Chiều dữ liệu tính tuyến tính, nên tổng LCU-giờ bằng tổng GB mà ALB xử lý trong tháng:

- Lúc thường: 13,14 triệu request × 16 KB = 210,2 GB, tức 210,2 LCU-giờ.
- Phần tăng thêm do flash sale (mục 1.1): 8,55 triệu × 16 KB = 136,8 GB, tức 136,8 LCU-giờ.

| Hạng mục | Cách tính | $/tháng |
| --- | --- | --- |
| EKS control plane | $0,10 × 730 | 73,00 |
| EC2: 2 × `m7g.large` | 2 × $0,102 × 730 | 148,92 |
| EBS gốc của node | 2 × 30 GB × $0,096 | 5,76 |
| ALB | $0,0252 × 730 = 18,396; 347,0 LCU-giờ × $0,008 = 2,776 | 21,17 |
| NAT: 1 cái + IP + 30 GB (image từ registry ngoài AWS, cổng thanh toán, API AWS) | 43,07 + 3,65 + 30 × $0,059 | 48,49 |
| RDS `db.t4g.medium` Multi-AZ + 50 GB | $0,203 × 730 = 148,19; 50 × $0,276 = 13,80 | 161,99 |
| Valkey 2 × `cache.t4g.small` | 2 × $0,0384 × 730 | 56,06 |
| CloudFront trả theo dùng | 832,5 GB nằm trong 1 TB miễn phí; (43,43 − 10) triệu request × $1,20/triệu | 40,11 |
| WAF | $5 + 6 rule × $1 + 43,43 triệu × $0,60/triệu | 37,06 |
| CloudWatch | 21,69 triệu request × 300 byte = 6,51 GB log × $0,70 = 4,55; lưu 3 tháng 19,5 GB × $0,03 = 0,59; 10 alarm × $0,10 = 1,00 | 6,14 |
| Lặt vặt | 10 secret 4,00; 1 KMS key 1,00; ECR 5 GB 0,50; Route 53 0,50; S3 50 GB 1,25; SQS dưới 1 triệu request 0; 50 GB giữa hai AZ × $0,02 = 1,00 | 8,25 |
| **Tổng** | | **≈ 607** |

SQS tính theo **mọi lệnh gọi API**, không theo số đơn: gửi, nhận, xoá, cả lần nhận rỗng, và mỗi khối 64 KB payload
tính là một request. Mức S có 1 `worker` long polling 20 giây, khi rảnh tốn khoảng 131.000 request/tháng, cộng khoảng
3 request mỗi đơn (gửi, nhận, xoá). Vậy vẫn nằm trong 1 triệu request miễn phí khi dưới khoảng 290.000 đơn/tháng. Gom
lô khi nhận và xoá giúp giảm số request; con số thật phải đo.

### 11.3. Mức M: khoảng $2.298/tháng

LCU mức M: lúc thường 131,4 triệu × 16 KB = 2.102,4 GB; phần tăng thêm do flash sale 85,5 triệu × 16 KB = 1.368 GB;
tổng 3.470,4 LCU-giờ.

| Hạng mục | Cách tính | $/tháng |
| --- | --- | --- |
| EKS control plane | $0,10 × 730 | 73,00 |
| EC2: node hệ thống 3 × `m7g.large` | 3 × $0,102 × 730 | 223,38 |
| EC2: node app 3 × `m7g.large` | 3 × $0,102 × 730 | 223,38 |
| EC2: thêm 3 node lúc flash sale | 3 × $0,102 × 30 giờ | 9,18 |
| EBS: 6 node × 30 GB + 50 GB cho Prometheus; 3 node flash sale × 30 GB trong 30 giờ | 230 GB × $0,096 = 22,08; 90 GB × $0,096 × 30 ÷ 730 = 0,36 | 22,44 |
| ALB | 18,396; 3.470,4 LCU-giờ × $0,008 = 27,763 | 46,16 |
| NAT: 3 cái + 3 IP + 100 GB | 3 × 43,07 = 129,21; 3 × 3,65 = 10,95; 100 × $0,059 = 5,90 | 146,06 |
| RDS `db.m7g.large` Multi-AZ + 100 GB | $0,468 × 730 = 341,64; 100 × $0,276 = 27,60 | 369,24 |
| Valkey 2 × `cache.t4g.medium` | 2 × $0,076 × 730 | 110,96 |
| Edge: CloudFront gói Premium (đã gồm WAF, Route 53 DNS, nạp log CloudFront và WAF, 5 TB credit S3) | 434,25 triệu request và 8,3 TB, trong hạn mức 500 triệu và 50 TB; so sánh ở mục 11.4 | 1.000,00 |
| CloudWatch | 216,9 triệu request × 300 byte = 65,07 GB log × $0,70 = 45,55; lưu 3 tháng 195,2 GB × $0,03 = 5,86; 20 alarm = 2,00 | 53,41 |
| Lặt vặt | 10 secret 4,00; 2 KMS key 2,00; ECR 10 GB 1,00; Route 53 và S3 200 GB: 0, vì gói Premium đã gồm hosted zone (khi gắn vào gói) và 5 TB credit S3; SQS (10 − 1) triệu × $0,40 = 3,60; 500 GB giữa hai AZ × $0,02 = 10,00 | 20,60 |
| **Tổng** | | **≈ 2.298** |

SQS 10 triệu request là mức dự trù: KEDA hỏi số message mỗi 30 giây (khoảng 88.000 request/tháng), tối đa 4 `worker`
long polling (khoảng 0,5 triệu request khi rảnh), phần còn lại đủ cho khoảng 3 triệu đơn với 3 request mỗi đơn. Log
chiếm $53: nếu muốn giảm, chỉ ghi mẫu một phần request thành công và ghi toàn bộ request lỗi.

### 11.4. Edge: trả theo dùng hay gói phẳng

Ngoài cách trả theo dùng, CloudFront có **gói phẳng** (flat-rate): trả một số tiền cố định mỗi tháng, gộp cả
CloudFront, WAF, chống DDoS, Route 53 DNS (khi gắn hosted zone vào gói), nạp log CloudFront và WAF, và credit lưu trữ
S3 (Free 5 GB, Pro 50 GB, Business 1 TB, Premium 5 TB). Mỗi gói có hạn mức:

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
| Trả theo dùng: CloudFront | $40,11 | (8.325 − 1.024) GB × $0,12 = 876,12 + (434,25 − 10) triệu × $1,20 = 509,10 → $1.385,22 |
| Trả theo dùng: WAF | $37,06 | $5 + $6 + 434,25 × $0,60 = $271,55 |
| **Trả theo dùng: tổng** | **$77,17** | **$1.656,77** |
| Gói Pro ($15) | 43,43 triệu request, gấp 4,3 lần hạn mức; không có VPC origin | không hợp |
| Gói Business ($200) | đắt hơn trả theo dùng | 434,25 triệu request, gấp 3,5 lần hạn mức |
| Gói Premium ($1.000) | đắt hơn nhiều | nằm trong hạn mức, rẻ hơn trả theo dùng $656,77 |
| **Chọn** | **Trả theo dùng** | **Premium** |

Ngoài khoản rẻ hơn, gói phẳng ở mức M còn chặn được rủi ro hoá đơn tăng đột biến khi bị tấn công DDoS: request bị WAF
chặn không tính vào hạn mức.

### 11.5. Tiền đi đâu

| Mức S ($607) | Tỉ lệ | | Mức M ($2.298) | Tỉ lệ |
| --- | --- | --- | --- | --- |
| RDS | 26,7% | | Edge (CloudFront Premium) | 43,5% |
| EC2 + EBS | 25,5% | | EC2 + EBS | 20,8% |
| CloudFront + WAF | 12,7% | | RDS | 16,1% |
| EKS control plane | 12,0% | | NAT | 6,4% |
| Valkey | 9,2% | | Valkey | 4,8% |
| NAT | 8,0% | | EKS control plane | 3,2% |
| ALB | 3,5% | | CloudWatch + lặt vặt | 3,2% |
| CloudWatch + lặt vặt | 2,4% | | ALB | 2,0% |

Hai mức có hai câu chuyện khác nhau:

- **Mức S bị chi phối bởi phần cố định.** Bỏ hết khoản tăng theo tải (CloudFront $40,11, request WAF $26,06, LCU
  $2,78, data qua NAT $1,77) thì vẫn còn khoảng **$536/tháng dù không có ai dùng**. Muốn rẻ hơn nữa phải bỏ bớt thành
  phần, không phải tối ưu máy (xem mục 11.8).
- **Mức M bị chi phối bởi băng thông và số request.** Edge chiếm 43,5%. Giảm số request và dung lượng ảnh mỗi người
  dùng tiết kiệm nhiều hơn mọi cách tối ưu máy chủ.

### 11.6. Cách giảm chi phí

**Cam kết 1 năm, không trả trước**, áp cho phần chạy cả tháng (node chạy thêm lúc flash sale vẫn trả on-demand):

| | Mức S | Mức M |
| --- | --- | --- |
| EC2 Compute Savings Plan ($0,102 → $0,076, giảm 25,5%) | 2 node × $0,026 × 730 = $37,96 | 6 node × $0,026 × 730 = $113,88 |
| RDS Reserved Instance (giảm 20,9% với `t4g.medium`, 22,9% với `m7g.large`) | ($0,203 − $0,1605) × 730 = $31,03 | ($0,468 − $0,3607) × 730 = $78,33 |
| **Tổng tiết kiệm** | **$68,99** | **$192,21** |
| **Còn lại** | **≈ $538/tháng** | **≈ $2.106/tháng** |

Đơn giá cam kết ở bảng trên chưa được đối chiếu lại (mục 11.1).

**Giảm số request mỗi người dùng (mức M).** Gói Business $200 có hạn mức 125 triệu request. Riêng phần tăng thêm của
flash sale đã khoảng 105,75 triệu request (mục 1.1), nên chỉ giảm request lúc thường là không đủ: còn 0,15 request/giây
mỗi người lúc thường thì tổng vẫn khoảng 193 triệu (mục 11.7). Phải giảm cả hai phía:

- Lúc thường: gộp 4 API mỗi màn hình thành 1–2 lần gọi qua một lớp BFF (backend for frontend); cho app giữ ảnh lâu
  bằng `Cache-Control: max-age=31536000, immutable`, đổi tên file mỗi khi ảnh đổi.
- Lúc flash sale: chặn bấm liên tục ở client (tối đa 1 lần mỗi 3 giây) hoặc dùng hàng chờ, đưa API xuống khoảng
  0,3 request/giây mỗi người.

Nếu đạt 0,12 request/giây mỗi người lúc thường và 0,3 request API/giây lúc flash sale: tổng khoảng 109,6 triệu
request, nằm trong gói Business, **tiết kiệm $800/tháng**.

**Môi trường staging:** tắt node và dừng RDS ngoài giờ làm việc (xem mục 11.8).

### 11.7. Giả định thay đổi thì edge mức M đổi thế nào

Mỗi dòng tính lại cả phần lúc thường lẫn phần tăng thêm của flash sale (mục 1.1). Lúc flash sale mỗi người gọi API
1 request/giây và tải ảnh theo đúng tần suất lúc thường của dòng đó.

| Thay đổi | Request cả tháng | Trả theo dùng (CloudFront dữ liệu + request, WAF) | Gói phẳng phù hợp |
| --- | ---: | --- | --- |
| Như giả định (10 KB/s, 0,5 request/s mỗi người) | 434,25 triệu | 876,12 + 509,10 + 271,55 = $1.656,77 | Premium $1.000 |
| Dữ liệu gấp đôi (20 KB/s) | 434,25 triệu | 9.216 GB × $0,12 + 6.410 GB × $0,10 = 1.746,92 + 509,10 + 271,55 = $2.527,57 | Premium $1.000 (16,7 TB, dưới 50 TB) |
| Request gấp đôi (1 request/s mỗi người lúc thường) | 778,5 triệu | 843,72 + 922,20 + 478,10 = $2.244,02 | Premium mức 750 triệu ($1.450), vượt nhẹ 3,8%; nếu vượt kéo dài, mức 1,25 tỉ ($2.250) đắt ngang trả theo dùng |
| Request còn 0,15/s mỗi người lúc thường | 193,3 triệu | 898,80 + 219,93 + 126,97 = $1.245,70 | Premium $1.000; gấp 1,5 lần hạn mức Business |
| 0,12/s lúc thường và flash sale 0,3 request API/s | 109,6 triệu | 787,34 + 119,54 + 76,77 = $983,65 | Business $200 |

Bảng này cho thấy vì sao phải đo hành vi người dùng thật trước khi chốt ngân sách: cùng 1000 CCU, chi phí edge chênh
nhau từ $200 tới $2.528 tuỳ app tải bao nhiêu ảnh, gọi bao nhiêu request, và người dùng bấm dồn thế nào lúc flash
sale.

### 11.8. Chưa tính trong bảng

- **Môi trường staging.** Dựng riêng, thu nhỏ, tốn khoảng **$377/tháng**: EKS $73; 2 × `t4g.large` $123,81; RDS
  `db.t4g.medium` Single-AZ + 20 GB $77,22; Valkey 1 node $28,03; NAT + IP $46,72; ALB $18,40; lặt vặt khoảng $10.
  Tắt node và dừng RDS ngoài giờ làm việc giảm được khoảng một nửa phần máy.
- **Môi trường lab giữ nguyên kết cấu M**, tắt ngoài giờ: khoảng $327/tháng, xem mục 12.
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
| Mức S | ≈ 15,8 triệu đ/tháng | ≈ 14,0 triệu đ/tháng |
| Mức M | ≈ 59,7 triệu đ/tháng | ≈ 54,7 triệu đ/tháng |

Chưa gồm thuế. Nếu hoá đơn AWS chịu VAT hoặc doanh nghiệp phải nộp thuế nhà thầu nước ngoài, cần cộng thêm theo xác
nhận của kế toán.

---

## 12. Môi trường lab giữ kết cấu M

Mục này dành cho lab chỉ chạy vài chục giờ mỗi tháng nhưng phải **giữ nguyên kết cấu M**: 3 AZ, 3 NAT, nhóm node hệ
thống và nhóm node app riêng, đủ các addon. Thứ được giảm là kích cỡ DB, cache, dữ liệu, số Pod tối đa và số giờ chạy
máy; không gộp nhóm node, không giảm AZ hay NAT, không bỏ addon. Số liệu là dự toán, chưa phải hoá đơn.

### 12.1. Cấu hình

| Thành phần | Cấu hình lab | So với M |
| --- | --- | --- |
| VPC, AZ | Singapore, 3 AZ, subnet public, private, data mỗi AZ | Giữ nguyên |
| NAT | 3 NAT + 3 IP public, chạy cả tháng | Giữ nguyên |
| Edge | CloudFront trả theo dùng + WAF, VPC origin tới ALB internal | Đổi gói Premium sang trả theo dùng, kết cấu giữ nguyên |
| EKS | 1 cluster, chạy cả tháng | Giữ nguyên |
| Node hệ thống | Managed node group 3 × `m7g.large` khi mở lab, mỗi AZ một node | Giữ nguyên, chỉ chạy trong giờ lab |
| Node app | Karpenter cấp 3 × `m7g.large` khi mở lab, mỗi AZ một node; NodePool chỉ cho `m7g.large` | Không có 3 node thêm lúc flash sale |
| `api` | KEDA theo CPU, 3–4 Pod; trigger cron (nếu thử scale theo lịch) `desiredReplicas: 4` | M: 3–15 Pod |
| `worker` | KEDA theo SQS, tối đa 3 Pod | M: tối đa 4 |
| PostgreSQL | `db.t4g.small` Multi-AZ, gp3 20 GB; dừng ngoài giờ | M: `db.m7g.large`, 100 GB |
| Valkey | 2 × `cache.t4g.micro` (0,5 GiB mỗi node), primary + replica khác AZ, Multi-AZ | M: 2 × `cache.t4g.medium` |
| Quan sát | Khoảng 20 alarm CloudWatch; Prometheus và Grafana tự host, PVC 50 GB | Giữ nguyên, giảm log, số series và thời gian lưu |
| Dịch vụ khác | S3 10 GB, ECR 2 GB, 3 secret, 2 KMS key, SQS, Route 53, Argo CD | Giữ đủ dịch vụ, dung lượng nhỏ |

Capacity: 4 `api` + 3 `worker` cần 2,75 vCPU; rollout thêm 1 Pod thì 3,25 vCPU, dưới 4,8 vCPU của 3 node app. Vẫn
phải kiểm tra RAM, cách xếp Pod và Pod đang tắt. Không dùng lịch cron 15 Pod của M; muốn thử mức đó phải thêm node và
tính lại ngân sách.

Giữ `m7g.large` cho cả hai nhóm. Đổi sang `t4g.large` cho nhóm hệ thống và `t4g.medium` cho nhóm app chỉ bớt khoảng
$7/tháng ở 32 giờ, nhưng máy burstable làm số đo p99 dao động. Máy 1 vCPU (`m7g.medium`) không đủ: sau DaemonSet chỉ
còn khoảng 0,64 vCPU và tối đa 8 Pod mỗi node.

Valkey `cache.t4g.micro` chỉ có 0,5 GiB mỗi node; chỉ đặt dữ liệu lab nhỏ vào cache. Không có node cache nào dừng được
như RDS, nên Valkey tính đủ tháng.

### 12.2. Chi phí

H là số giờ chạy của **mỗi** node và của RDS trong tháng. Ở H = 32, hai nhóm 3 node có tổng 192 giờ máy. Bảng dưới
tính H = 32, tháng 730 giờ, tỷ giá 26.000 đ/USD:

| Hạng mục | Cách tính | $/tháng |
| --- | --- | ---: |
| EKS | $0,10 × 730 | 73,00 |
| 6 node `m7g.large` | 6 × $0,102 × 32 giờ | 19,58 |
| EBS | Ổ gốc 6 × 20 GB × $0,096 = 11,52 (tính cả tháng cho an toàn); Prometheus 50 GB = 4,80 | 16,32 |
| ALB | $0,0252 × 730 + 0,10 dự trù LCU | 18,50 |
| 3 NAT + 3 IP + 10 GB | 3 × ($0,059 + $0,005) × 730 + 10 × $0,059 | 140,75 |
| RDS | $0,102 × 32 giờ = 3,26; gp3 Multi-AZ 20 × $0,276 = 5,52 | 8,78 |
| Valkey | 2 × $0,0192 × 730 | 28,03 |
| CloudFront | Dự trù cho tải lab nhỏ | 2,00 |
| WAF | $5 + 6 rule × $1 + 0,288 triệu request × $0,60 | 11,17 |
| CloudWatch | 20 alarm 2,00; log và lưu trữ 1,00 | 3,00 |
| Lặt vặt | Secrets 1,20; KMS 2,00; ECR 0,20; Route 53 0,50; S3 0,55; SQS 0,50; giữa AZ 0,20; khác 1,00 | 6,15 |
| **Tổng** (cộng số chưa làm tròn) | | **327,29** |
| **Quy ra tiền Việt** | | **8,51 triệu** |
| **Cộng 10% dự phòng** | | **9,36 triệu** |

Ghi chú:

- Tải giả định: trung bình 5 CCU trong 32 giờ, tức 288.000 request CloudFront, 115.200 request API và 5,76 GB dữ liệu.
- Ổ gốc thực tế bị xoá khi node tắt, nên chỉ tốn khoảng $0,50; dòng $11,52 cao hơn thực tế khoảng 286.000 đ, giữ làm
  biên an toàn. Nếu Grafana dùng PVC gp3 10 GB thì cộng $0,96.
- Nếu tài khoản đang ở gói Premium, hạ gói có hiệu lực từ chu kỳ thanh toán sau, nên hạ trước cuối tháng.

Công thức theo H:

```text
USD(H) = 304,268 + 0,7194 × H            (tháng 730 giờ)
0,7194 = 6 × 0,102 (node) + 0,102 (RDS) + 0,0054 (request WAF)
Tháng T giờ: cộng (T − 730) × 0,3556 USD  (EKS, 3 NAT + IP, ALB, Valkey tính theo giờ)
```

| Giờ chạy H/tháng | Trước thuế | Cộng 10% dự phòng |
| ---: | ---: | ---: |
| 0 | 7,91 triệu | 8,70 triệu |
| **24** | **8,36 triệu** | **9,20 triệu** |
| **32** | **8,51 triệu** | **9,36 triệu** |
| 40 | 8,66 triệu | 9,53 triệu |
| 80 | 9,41 triệu | 10,35 triệu |

Tháng 31 ngày (744 giờ) cộng khoảng $4,98: H = 32 thành 8,64 triệu, 9,50 triệu có dự phòng. Nếu phải giữ dung lượng
của M (RDS 100 GB, ổ gốc 30 GB, 10 secret, ECR 10 GB, S3 200 GB) thì cộng $36,19/tháng: H = 24–32 khoảng
**10,2–10,5 triệu có dự phòng**.

**Ngân sách đề xuất: khoảng 9,5 triệu đ/tháng** cho lab dựng mới, ít dữ liệu, 24–32 giờ chạy máy. Khoản 10% dự phòng
không gồm thuế; nếu có VAT hoặc thuế nhà thầu nước ngoài thì dự phòng có thể bị dùng hết. Mỗi 1.000 đ/USD chênh tỷ giá
làm tổng đổi khoảng 0,33 triệu.

**Không thể xuống 5–6 triệu khi giữ kết cấu M.** Dù không chạy giờ máy nào, riêng EKS $73 + 3 NAT và IP $140,16 + ALB
$18,40 + 2 Valkey micro $28,03 + WAF cơ bản $11 đã là $270,59, khoảng 7,04 triệu. Giảm CCU hay giảm giờ máy không
xoá được các khoản này; giảm thêm cỡ node cũng chỉ bớt vài USD.

### 12.3. Tắt và mở lab

Quy trình chỉ dùng **commit Git** (Argo CD kéo về) và **AWS API**, chạy từ máy quản trị nằm ngoài cluster, nên không
cần `kubectl` hay đường vào API private.

Chuẩn bị một lần trong Git:

- Manifest Deployment `api` và `worker` không khai báo `replicas`, để KEDA và Argo CD không ghi đè lẫn nhau.
- Trạng thái "lab-off": thêm annotation `autoscaling.keda.sh/paused-replicas: "0"` vào ScaledObject của `api` và
  `worker`, đặt `spec.limits.cpu: "0"` cho NodePool app. Trạng thái "lab-on" bỏ hai thay đổi này.
- NodePool app thu hồi node trống (`consolidationPolicy: WhenEmptyOrUnderutilized` hoặc `WhenEmpty`, `consolidateAfter`
  ngắn), chỉ cho `m7g.large` và trải theo zone.

Tắt lab:

1. Dừng load test. Chờ hàng đợi SQS rỗng, kể cả message đang xử lý (`ApproximateNumberOfMessagesVisible` và
   `ApproximateNumberOfMessagesNotVisible` bằng 0).
2. Commit "lab-off". Argo CD áp dụng: KEDA hạ `api` và `worker` về 0, NodePool không cấp node mới, Karpenter thu hồi node
   app đã trống. KEDA hạ replica không qua eviction nên PDB không chặn.
3. Dùng EC2 API xác nhận không còn instance có tag `karpenter.sh/nodepool` ở trạng thái `pending`, `running`,
   `stopping` hay `shutting-down`. Quá thời gian chờ mà vẫn còn, chỉ terminate trực tiếp khi chắc không còn Pod app.
4. Sau bước 3 mới hạ managed node group hệ thống về `minSize = 0`, `desiredSize = 0` (giữ `maxSize` ≥ 1) bằng
   `aws eks update-nodegroup-config`, rồi `aws rds stop-db-instance`. Tắt Karpenter trước thì node app còn sót sẽ chạy
   và tính tiền mãi.

Mở lab:

1. Commit "lab-on" (cluster đang tắt nên chưa áp dụng ngay).
2. `aws rds start-db-instance`; đưa nhóm hệ thống về min/desired 3. Karpenter cần node hệ thống để chạy, nên không tự
   khởi động từ 0.
3. Argo CD chạy lại, áp dụng "lab-on": KEDA khôi phục 3 `api`, Karpenter cấp 3 node app.
4. Chờ RDS `available` và target group có 3 target `healthy` (`aws elbv2 describe-target-health`); kiểm tra mỗi AZ có
   một node hệ thống và một node app. Không sleep cố định rồi coi là sẵn sàng.
5. Ghi lại giờ chạy thực tế từng nhóm để đối chiếu hoá đơn. RDS khởi động có thể mất từ vài phút tới vài giờ.

Lưu ý thêm:

- RDS Multi-AZ dừng được, nhưng **tự bật lại sau 7 ngày dừng liên tục**, và vẫn tính tiền ổ đĩa, backup khi dừng. Kỳ
  nghỉ dài phải kiểm tra việc tự bật.
- ElastiCache không có chức năng dừng; muốn ngừng tính tiền phải xoá cluster, nằm ngoài phạm vi giữ nguyên dịch vụ.
- Dữ liệu cần giữ qua các lần tắt phải nằm trên PVC hoặc dịch vụ ngoài, không nằm trên ổ gốc hay `emptyDir`.
- Không mua Savings Plan hay Reserved Instance cho node chỉ chạy vài chục giờ mỗi tháng. Valkey chạy cả tháng có thể
  cân nhắc Reserved sau khi chốt cỡ node, nhưng khoản tiết kiệm nhỏ.

### 12.4. Tiêu chí chấp nhận

Mức **10–20 CCU** là mục tiêu thử ban đầu, chưa phải kết quả đo:

| CCU đỉnh | API lúc thường | API lúc tranh mua |
| ---: | ---: | ---: |
| 10 | 2 RPS | 10 RPS |
| 20 | 4 RPS | 20 RPS |
| 50, thử sau | 10 RPS | 50 RPS |

Trước khi chấp nhận 20 CCU:

- Chạy tải 20 CCU đúng tỉ lệ đọc ghi, thử đỉnh ngắn 30 CCU; thử flash sale với 1 request API/giây mỗi người. Thời gian
  chạy tải nằm trong H.
- Đo p99 như mục 10. Ở 20 CCU chỉ có khoảng 4 request API mỗi giây, nên mỗi lần đo cần chạy khoảng 20–30 phút mới đủ
  vài nghìn request. Chưa đo thì không cam kết p99 dưới 300 ms.
- Đo CPU (requests và thực dùng), RAM, số kết nối DB, CPU credit của RDS, bộ nhớ và số key bị đẩy ra của Valkey, tuổi
  message SQS.
- Rollout trong lúc chạy tải; diễn tập mất node hoặc AZ trong một buổi riêng; đo cả thời gian phục hồi và độ đúng dữ
  liệu khi RDS hoặc Valkey failover.
- Thử bấm lại, giao trùng message, lỗi gửi SQS và failover Valkey: không được có đơn trùng, thanh toán trùng hay tồn
  kho âm trong PostgreSQL.
- Đối chiếu hoá đơn tuần đầu để cập nhật các khoản dự trù. WAF tính cả request bị chặn, nên lab tắt không có nghĩa
  traffic edge bằng 0.

Không đạt thì hạ về 10 CCU hoặc giảm dữ liệu, sửa query, index, connection pool và luồng đặt hàng trước. Chỉ tăng cấu
hình sau khi tính lại ngân sách; không bỏ dịch vụ, không chuyển RDS sang Single-AZ, không bỏ bản sao Valkey.

---

## 13. Khi vượt 1000 CCU

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

## 14. Tóm tắt quyết định thiết kế

1. **Đổi CCU ra RPS và GB trước**, gồm cả phần tăng thêm của flash sale. Mọi con số về máy và tiền đều suy ra từ đó.
2. **Chặn tải ở xa nhất.** CloudFront và WAF đứng trước; ALB là internal, chỉ nhận traffic qua VPC origin.
3. **ALB do Terraform sở hữu.** Controller chỉ gắn Pod qua `TargetGroupBinding`.
4. **target-type ip + readiness gate**, cùng `preStop`, `deregistration_delay` và grace period khớp nhau, cho deploy
   không downtime; xác nhận bằng cách rollout khi đang chạy tải và đếm 5xx.
5. **Dự trù đủ để mất 1 node (S) hoặc 1 AZ (M), kể cả lúc rollout**, và trải Pod đều qua các AZ lẫn các node.
6. **Scale trước cho flash sale.** Autoscaling chỉ dùng cho dao động thường ngày.
7. **Valkey chặn nhanh, PostgreSQL quyết định tồn kho.** Đơn đi qua SQS với idempotency key `UNIQUE`; `202` chỉ là đã
   nhận yêu cầu.
8. **Không thêm thứ chưa cần ở quy mô này:** RDS Proxy, interface endpoint, Aurora, multi-region.
9. **Chi phí:** mức S bị phần cố định chi phối, mức M bị băng thông và request chi phối. Tối ưu request và ảnh trước
   khi tối ưu máy.
10. **Thay mọi giả định bằng số đo thật**: hành vi người dùng (mục 1.1) và sức chịu của một Pod (mục 4).
11. **Lab giữ kết cấu M tốn khoảng 9,5 triệu đ/tháng có dự phòng.** Phần cố định (EKS, 3 NAT, ALB, Valkey) chiếm gần
    hết, nên giảm giờ máy hay CCU không đưa được về 5–6 triệu.

---

## 15. Nguồn

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
- [Review kiến trúc và ngân sách lab, 2026-10-13](review-kien-truc-eks-alb-va-ngan-sach-lab.md)
- [ALB: target group attributes (slow start, routing algorithm, deregistration delay)](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/edit-target-group-attributes.html)
- [ALB: health checks](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/target-group-health-checks.html)
- [Elastic Load Balancing pricing (LCU)](https://aws.amazon.com/elasticloadbalancing/pricing/)
- [ElastiCache: Multi-AZ và replication bất đồng bộ](https://docs.aws.amazon.com/AmazonElastiCache/latest/dg/AutoFailover.html)
- [SQS: at-least-once delivery](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/standard-queues-at-least-once-delivery.html)
- [Amazon ECR: VPC endpoints](https://docs.aws.amazon.com/AmazonECR/latest/userguide/vpc-endpoints.html)
- [Amazon EKS: cluster endpoint access](https://docs.aws.amazon.com/eks/latest/userguide/cluster-endpoint.html)
- [Amazon RDS: stopping a DB instance](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_StopInstance.html)
- [Kubernetes: Disruptions và PodDisruptionBudget](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/)
- [Kubernetes: Pod topology spread constraints](https://kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints/)
- [KEDA: Pausing autoscaling](https://keda.sh/docs/latest/concepts/scaling-deployments/#pausing-autoscaling)
- [Karpenter: NodePools](https://karpenter.sh/docs/concepts/nodepools/)
