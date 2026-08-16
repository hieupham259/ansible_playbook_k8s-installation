# Review các lab RKE2 multi-cluster — phần còn mở

> Bản gốc của review này liệt kê ~20 vấn đề của [LAB-M1](LAB-M1-BA-CUM-RKE2-RANCHER-GITLAB.md)
> và [LAB-M2](LAB-M2-CAPSTONE-PRODUCTION-HA.md). Các mục đã xử lý (đợt sửa 17/08/2026) đã
> được loại khỏi file theo quy ước: **file này chỉ giữ những gì chưa xong**. Trong số đã xử
> lý có toàn bộ 7 lỗi chặn của M1 (StorageClass CICD, GitLab chart 10.x với external
> Redis/object storage, sizing PostgreSQL, CA cho runner job pod, pull credential, ArgoCD
> repo credential, manifest workload + mâu thuẫn RWO) và 4 lỗi chặn của M2 (firewall matrix
> cho join qua VIP, tách 3 VIP ingress, guard version-aware cho IaC, alert receiver liên
> cụm + failure injection). Đối chiếu chi tiết: lịch sử git của hai file lab.

## Các mục còn mở

### 1. Ba thành phần vẫn không pin cứng version

`gitlab-runner` chart, `reflector`, binary MinIO — là quyết định có khai báo tại bảng
[M1 §2.1](LAB-M1-BA-CUM-RKE2-RANCHER-GITLAB.md#21-phiên-bản-được-khóa) (verify lúc cài + ghi
version vào hồ sơ), không phải thiếu sót ngầm. Muốn reproducibility tuyệt đối thì pin nốt
sau lần pilot đầu tiên, dùng chính version mà pilot đã chạy PASS.

### 2. M2: repo IaC hoàn chỉnh vẫn là bài tập của người học

M2 §4 đã có các mảnh chịu lực (`config.yaml.j2`, handlers, `site.yml` với `serial: 1`,
guard version-aware, template haproxy/keepalived, `.gitlab-ci.yml` validate) và ghi rõ phạm
vi: ráp thành repo chạy được — gồm cả role `fw` và role `lb` hoàn chỉnh — là một phần của
capstone. Đây là lựa chọn thiết kế được chấp nhận, nhưng vẫn là khối việc thật người học
phải làm trước khi §5–§6 chạy được bằng playbook.

### 3. Logging tập trung vẫn là tùy chọn

M1 nói rõ không phủ logging; M2 §9 để logging (rancher-logging → Loki) là phần mở rộng.
Nếu mục tiêu là khớp 100% bài viết gốc ("cụm Admin chứa monitor/logging"), cần nâng logging
thành mục bắt buộc của M2 với gate riêng (query được log của app demo từ Loki trên cụm
Admin).

### 4. Chưa có performance/capacity gate

M2 §11 đã bắt buộc đo RTO/RPO cho từng kịch bản DR, nhưng chưa có gate hiệu năng/capacity:
tải thử ingress qua VIP, độ trễ etcd (`etcdctl check perf` hoặc metric
`etcd_disk_wal_fsync_duration`), headroom RAM/CPU từng node sau khi đủ stack. Thiếu các con
số này thì sizing của §2 là ước lượng chưa được kiểm chứng.

## Việc thứ 4 — pilot run, gate cuối cùng còn thiếu

End-to-end có một caveat nền cần nói thẳng: toàn bộ lệnh và values được viết và pin theo
docs chính thức (đã kiểm từng điểm quan trọng), nhưng **chưa từng được chạy thật một lượt**.
Các chỗ rủi ro nhất nếu có sai lệch: cấu trúc values của GitLab chart 10.2.2 (key
`appConfig.*` đúng theo docs nhưng chưa xác nhận bằng thực thi), key `ports.web.hostPort`
trong HelmChartConfig của `rke2-traefik`, và các bước phụ thuộc UI (Rancher import, tạo
deploy token). Độ tin hiện tại là "đúng theo tài liệu", chưa phải "đã chứng minh bằng một
lần chạy pilot" — với tài liệu dạy học, một lần pilot chính là gate cuối cùng còn thiếu.

Cách đóng mục này:

1. Chạy pilot M1 trọn vẹn trên một máy host thật, theo đúng thứ tự và gate trong file.
2. Ghi lại mọi điểm lệch giữa tài liệu và thực tế (gate fail, key values sai, version repo
   đã trôi) vào hồ sơ pilot.
3. Cập nhật hai file lab theo kết quả; pin nốt ba thành phần ở mục 1 bằng version pilot đã
   dùng.
4. Xóa mục tương ứng khỏi file này. File rỗng phần "còn mở" = bộ lab được coi là đã kiểm
   chứng.
