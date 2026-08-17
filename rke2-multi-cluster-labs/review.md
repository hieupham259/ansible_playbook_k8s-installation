# Review các lab RKE2 multi-cluster — phần còn mở

> Cập nhật sau đợt sửa thứ hai (17/08/2026). Theo quy ước: mục đã sửa trong file LAB thì xóa
> khỏi đây; file này chỉ giữ vấn đề còn tồn đọng. Đợt sửa này đã đóng — với các claim
> version được kiểm chứng lại trên docs chính thức: **M1-01** (PostgreSQL 17 từ PGDG — GitLab
> 19.x chỉ hỗ trợ 17.x), **M1-02** (`ingressendpoint.ip` thay `publishedService` — ServiceLB
> của RKE2 tắt mặc định), **M1-03** (transcript Git→pipeline→ArgoCD đầy đủ, namespace
> idempotent), **M1-04** (khối CICD viết trọn, thêm Helm), **M1-05** (resolv.conf của
> router), **M1-06** (đường dẫn CA/cert chuẩn hóa + gate), **M1-09** (giới hạn rotation
> xuyên cụm ghi tường minh), phần lớn **M1-10** (gate storage/egress thành lệnh xác định);
> **M2-01** (đổi định vị: capstone assignment, chấm bằng gate), **M2-02** (18 VM, app 6
> node, quy ước IP), **M2-03** (keepalived đầy đủ + `ip_nonlocal_bind` + gate đếm đúng VIP),
> **M2-04** (ruleset nftables hoàn chỉnh, mc2-fw kiêm bastion), **M2-05** (Traefik DaemonSet
> theo cụm, Longhorn `createDefaultDiskLabeledNodes` + label 3 node), **M2-06** (gate canary
> thay `--check`, pin CI deps, vault password cho CI), **M2-07** (bootstrap ArgoCD bằng
> Ansible; secrets từ vault → Kubernetes Secret, escrow), **M2-08** (`antiAffinity=required`),
> **M2-10** (manifest Ingress sink + AlertmanagerConfig + gate config sinh ra), **M2-11**
> (MinIO chạy HTTPS bằng CA lab; Backup CR có `resourceSetName` + Secret), **M2-12** (toolbox
> objectStorage config), **M2-14** (uncordon, snapshot filename thật, lệnh theo từng host,
> biến thể restore từ S3), **M2-15** (bỏ placeholder version, vòng nâng cấp agent riêng).
> Đợt hardening M1 kế tiếp đã đóng thêm các blocker tĩnh: cấu hình `.254` cho VMnet2–5,
> Netplan/nftables/dnsmasq/PostgreSQL ghi file bằng command đầy đủ, disk Longhorn retry-safe,
> gate readiness RKE2/GitLab, lifecycle secret, GitOps v1→v2 và snapshot powered-off nhất
> quán. Helm, MinIO/`mc`, gitlab-runner và Reflector cũng đã pin version cụ thể; trạng thái
> runtime của các pin này vẫn thuộc gate pilot cuối.

## Các mục còn mở

### 1. (từ M1-07, thu hẹp) GitLab trên node 8 GB chưa có bộ values đã kiểm bằng chạy thật

M1 đã ghi rõ 8 GB là mức memory-constrained (baseline chính thức 16 GB) và thêm gate
OOMKilled + headroom sau cài. Phần còn thiếu chỉ đóng được bằng pilot: bộ values chạy ổn
thực tế trên 8 GB (hoặc kết luận phải tăng RAM). Ghi kết quả vào bảng version M1 sau pilot.

### 2. (từ M1-10 + M2-16) Logging tập trung vẫn nằm ngoài phạm vi bắt buộc

M1 nói rõ không phủ logging, M2 để logging là phần mở rộng — chưa khớp trọn mô tả "cụm Admin
chứa monitoring/logging" của bài viết gốc. Nâng thành mục bắt buộc (rancher-logging → Loki
trên Admin, gate query log xuyên cụm) là quyết định phạm vi của chủ repo, chưa thực hiện.

### 3. (từ M2-09, thu hẹp) Negative test của §8 còn ở dạng mô tả

Phạm vi PSA đã sửa (label namespace, không phải project). Còn thiếu manifest cụ thể cho hai
phép thử âm tính: pod xin vượt quota và pod privileged; hiện là câu hướng dẫn, người học tự
viết. Bổ sung hai manifest ngắn hoặc chấp nhận là bài tập — cần một quyết định.

### 4. (từ M2-13, thu hẹp) Hai restore còn ở mức thao tác hướng dẫn

PostgreSQL đã có script + cron + restore-verify từ S3; etcd đã command-by-command. Còn lại:
Longhorn restore vẫn qua UI ("restore thành volume mới, tạo PV/PVC") chưa có manifest PV/PVC
mẫu, và GitLab restore chưa có maintenance plan (scale down webservice/sidekiq, thông báo
downtime, rollback nếu restore fail). Chốt command-by-command sau lần diễn tập đầu.

### 5. (từ M2-16) Ngưỡng capacity mới là ngưỡng tham chiếu

Gate capacity đã thêm vào M2 §9 (headroom node, p99 fsync etcd < 25ms, thời gian rebuild
Longhorn) nhưng các con số ngưỡng chỉ được xác nhận là phù hợp với hạ tầng thuê cụ thể sau
khi đo thật. Sizing §2 vẫn là ước lượng cho tới lúc đó.

## Gate cuối cùng: pilot thật

Mọi sửa đổi ở trên là sửa tĩnh trên văn bản, đối chiếu docs chính thức — chưa có lượt chạy
thật nào. Điều kiện xóa mục này:

1. Chạy M1 từ host trống đến `m1-complete`, không dùng kiến thức ngầm ngoài prerequisite đã
   link; lưu transcript (command, version thật, output gate, thời gian, mọi điểm lệch).
2. Chạy M2 trên topology 18 VM đã chốt, từ inventory đến đủ gate §13.
3. Cập nhật hai file LAB theo mọi điểm lệch và chỉ giữ version đã PASS runtime.
4. Reset rồi để một người học khác chạy lại bằng đúng tài liệu đã sửa; lượt thứ hai không
   cần "mentor tự điền phần thiếu" thì bộ lab mới được coi là runbook đã kiểm chứng.
