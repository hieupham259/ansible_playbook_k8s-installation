# Rà soát dự án Bookstore và Lab 3a → Lab 14

Kiểm lại: **14/10/2026**, đối chiếu các chỉnh sửa với review ngày 13/10/2026. Phạm vi: README, DU-AN-BOOKSTORE và 21 lab dự án. Báo cáo này chỉ giữ các vấn đề còn tồn tại; bảy phát hiện cũ đã được xác nhận sửa thành công đã được xóa, cùng bảng trạng thái và nhận xét cũ không còn đúng.

**Còn 4 vấn đề: 1 P1 và 3 P2.** Thiết kế dự án vẫn phù hợp mục 7.1. Các vấn đề dưới đây nằm ở độ tin cậy của gate, nhịp bàn giao và việc giữ kiểm thử hồi quy qua các sprint.

| Mã | Mức | Vấn đề còn lại |
| --- | --- | --- |
| F02 | P1 | Một số gate kiểm kê vẫn biến lỗi đọc thư mục/API thành kết quả sạch |
| F07 | P2 | Phép đo request mới chưa chứng minh thao tác rollback/phát hành đã thành công |
| F08 | P2 | Gate giữa phiên chung không chạy được tại các mốc chưa có Kubernetes hoặc chưa triển khai Bookstore |
| F11 | P2 | Smoke 9b → 12 bỏ mất kiểm retention/file tạm mới có ở 9a |

P1: cần sửa trước pilot/nghiệm thu. P2: cần chỉnh trước công bố chuỗi lab đã được kiểm chứng.

## Trạng thái xử lý — 14/10/2026, lần 2

Cả bốn vấn đề đều được đối chiếu lại với source và xác nhận **đúng**; đã sửa. Ký hiệu: ✅ đúng và đã
sửa. Chi tiết ở dòng *Trạng thái* dưới tiêu đề từng vấn đề.

| Mã | Đánh giá | Trạng thái | Phạm vi sửa |
| --- | --- | --- | --- |
| F02 | Đúng, và rộng hơn báo cáo | ✅ Đã sửa | 11 bản `teardown.sh` (3a → 14, cả bản HA của 8b); gate node ở 6a, 6b, 8c, 9b, 12; 121 phép đếm `kubectl` trong gate; gate đóng sprint dạng "không thấy thì PASS" |
| F07 | Đúng | ✅ Đã sửa | 5a B3, 12 B6 |
| F08 | Đúng | ✅ Đã sửa | README 7.5, mục 1.2 của 5b, 8b, 8c, CLAUDE.md |
| F11 | Đúng, và rộng hơn báo cáo | ✅ Đã sửa | smoke 6a, 6b, 7a, 7b, 8a, 9b, 11a, 11b, 12 |

Kiểm tra sau khi sửa — tĩnh, không chạy cluster, không cài gì:

- `bash -n`: 868 khối Bash của mọi file lab và 55 thân script trong heredoc. Không có lỗi mới; năm khối
  báo lỗi là mẫu `kubeadm join`/đường dẫn dump có placeholder `<...>`, giống hệt ở HEAD.
- Link tương đối trong `labs/*.md`, `00-ALO-TRINH-ADMIN.md`, `CLAUDE.md`: 0 lỗi file, 0 lỗi anchor.
- Đối chứng bằng hàm giả, đúng các kịch bản của báo cáo:
  - Teardown 3a, 4b, 5a, 6a, 7a, 9b, 11a, 14: một truy vấn ở giữa khối trả Forbidden → `STOP`, exit 1;
    mọi truy vấn thành công và rỗng → `teardown: sach`.
  - Gate node 6b B0: SSH thành công nhưng `find` lỗi → `STOP`; SSH 255 → `STOP`; còn thư mục → `FAIL`.
  - Helper `sl`: `kubectl` lỗi → in `loi`, mọi phép so sánh số đều sai, không có PASS.
  - 5a B3: undo lỗi, deploy lỗi, hoặc cả hai, trong khi HTTP vẫn 200 → `FAIL`; hai chặng thật sự
    xảy ra → PASS với revision tăng 5 → 6 → 7.
  - 12 B6: restart lỗi, status lỗi, hoặc Pod cũ không đi → `FAIL`.
  - Khối đọc PVC: 11 bản → `FAIL`; một file tạm cũ → `FAIL`; Pod đọc không xong → `FAIL`.

Một phép thử ban đầu chạy lệnh remote thật với `sudo` giả qua `bash -c` bị công cụ chặn vì chạy nội
dung động; phần đó được thay bằng `ssh` giả trả đúng mã thoát và output mà lệnh remote sinh ra.

## Phạm vi kiểm chứng lần này

Đã đọc diff và các đoạn source liên quan tới từng phát hiện cũ, rà các phiên bản script kế thừa, kiểm cú pháp và chạy đối chứng cục bộ bằng công cụ đã cài. `kubectl`, `ssh`, `curl`, `pg_dump` và `pg_restore` trong phép thử được thay bằng hàm giả; file thử nằm trong thư mục tạm riêng. Không kết nối cluster, chạy image, cài/tải phần mềm hay thực hiện các lệnh phá cluster.

| Kiểm tra | Kết quả |
| --- | --- |
| Bash | 672 khối Bash và 42 thân script dùng chung; không có lỗi cú pháp trong 42 thân script. Bốn khối mẫu vẫn cần điền placeholder `<...>` trước khi chạy |
| YAML | 227 khối; ba mảnh minh họa không phải YAML độc lập như ở lần rà đầu; chưa thay thế kiểm schema phía API server |
| Guard Pod Security | 15 đối chứng trên smoke 9b, 11a, 11b, 12, 14: thành công sạch → PASS; cảnh báo vi phạm và Forbidden → FAIL |
| Ràng buộc tên CRD | 12 kiểm tra schema trên các version trong hai mẫu CRD: tên ở biên 54 ký tự được nhận; 55 ký tự và dấu chấm bị từ chối. Quy tắc ràng buộc metadata.name được đối chiếu với [tài liệu CRD Kubernetes v1.35](https://v1-35.docs.kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/#specifying-a-structural-schema) |
| Logic operator | Job không hợp lệ → ghi Failed; quota từ chối → trả lỗi để thử lại |
| Logic ghi backup | Bốn trường hợp: 6a, 9a, operator plain và custom. Mỗi trường hợp chạy 12 lần thành công bằng hàm dump giả còn 10 file hoàn chỉnh; lần dump lỗi không sinh thêm file hoàn chỉnh |
| Gate kiểm thư mục/API | Tái hiện hai kết quả báo sạch sai; chi tiết F02 |
| Gate đo rollout | Tái hiện PASS khi undo và deploy đều thất bại nhưng HTTP đối chứng vẫn 200; chi tiết F07 |

Các phép thử backup xác nhận logic shell; chúng không chứng nhận nội dung dump PostgreSQL, quyền filesystem trong container hay khả năng restore. Chưa chạy trọn chuỗi VM, đo tải HPA hoặc xác nhận thời lượng người học. Phần kiểm tra này chỉ cập nhật file review, không chỉnh source lab.

## F02 — P1: Gate kiểm kê vẫn có thể báo sạch khi không đọc được dữ liệu

> **Trạng thái 14/10/2026: ✅ Đúng — đã sửa, phạm vi rộng hơn báo cáo.** (1) Ba gate thư mục node ở 6a, 6b: lệnh remote là `sudo -n test ! -e … || sudo -n find …`, không còn `| wc -l` phía remote; thư mục gốc chưa có → rỗng, mã 0; `find` hay `sudo` lỗi → mã khác 0 → `STOP`. Cùng lỗi được tìm và sửa ở 8c B2 (kiểm sau `kubeadm reset`, `/var/lib/etcd`, CNI/kubeconfig/iptables, sandbox `crictl`), 9b (swap ba node) và 12 (inhibitor lock). (2) Mọi bản `teardown.sh` — không chỉ 6a, 7a, 9b, 11a, 14 mà cả 3b, 4a, 4b, 5a, 5b và `teardown-ha-lab.sh` của 8b — đếm bằng hàm `dem <biến> <lệnh>`: mỗi truy vấn chạy riêng trong shell chính, lỗi thì `STOP` và exit 1. `project_pvs` và danh sách PV của 9b tách hai truy vấn, lỗi thì trả lỗi thay vì danh sách thiếu. (3) Mẫu `[ "$(kubectl … 2>/dev/null | wc -l)" -eq 0 ]` trong gate — 121 chỗ ở 19 lab — chuyển sang helper `sl`, in `loi` khi lệnh lỗi để phép so sánh sai. (4) Gate đóng sprint dạng `kubectl get X && FAIL || PASS` ở 3c, 5a, 6b, 7a, 8a, 8c, 14 đọc trước rồi mới xét có/không. Còn lại, chưa sửa: các phép thử "phải bị từ chối" giữa ticket dùng `&& FAIL || PASS` — không phải gate kiểm kê trước snapshot.

**Trạng thái:** chưa thể đóng. Trường hợp SSH trả 255 đã được xử lý ở các vòng kiểm thư mục của teardown; nhưng còn hai đường lỗi khác.

**1. Lỗi lệnh đọc thư mục phía remote vẫn bị pipeline che.**

Nguồn: [6b B0](E:/courses/Ansible/ansible_playbook_k8s-installation/k8s-docs/labs/LAB-6B-SNAPSHOT-VA-VOLUME-NANG-CAO.md:190), [6b gate cuối](E:/courses/Ansible/ansible_playbook_k8s-installation/k8s-docs/labs/LAB-6B-SNAPSHOT-VA-VOLUME-NANG-CAO.md:2481), [6a gate cuối](E:/courses/Ansible/ansible_playbook_k8s-installation/k8s-docs/labs/LAB-6A-PV-PVC-VA-STORAGECLASS.md:2854).

Các đoạn này vẫn đếm bằng `ls ... 2>/dev/null | wc -l` trong shell remote không kiểm lỗi của `ls`. `sudo -n true` thành công chỉ chứng minh quyền sudo tại thời điểm đó; không chứng minh lệnh đọc thư mục tiếp theo thành công. Nếu `ls` lỗi thì `wc -l` vẫn in 0, và case 0 báo PASS.

Đối chứng lấy đúng đoạn B0 của 6b: SSH được giả lập thành công, `sudo -n true` thành công, lệnh đọc thư mục trả lỗi. Kết quả:

```text
exit=0
PASS: worker khong con thu muc du lieu cu
```

**2. Lỗi một truy vấn API trong khối đếm của teardown vẫn bị bỏ qua.**

Nguồn: [9b khối LEFT](E:/courses/Ansible/ansible_playbook_k8s-installation/k8s-docs/labs/LAB-9B-POD-SECURITY-VA-HARDENING.md:2010). Các khối tương tự cần rà ở teardown 6a, 7a, 11a và 14.

Đưa lệnh `kubectl` xuống cuối khối chưa bảo đảm mọi truy vấn trước đó đã thành công. Với Bash mặc định, lỗi trong một lệnh sớm của command substitution có thể bị che khi khối tiếp tục và lệnh cuối thành công. Đối chứng dùng đúng khối LEFT của 9b với `set -euo pipefail`: truy vấn PV trả Forbidden, các truy vấn còn lại trả thành công và rỗng. Kết quả vẫn `exit=0`, `LEFT=0`.

**Cần sửa:** kiểm mã thoát của từng thao tác đọc trước khi đếm. Chỉ coi thư mục không tồn tại là sạch khi đã xác nhận được tình trạng đó; lỗi đọc phải STOP. Khối kiểm kê API phải thất bại nếu bất kỳ truy vấn bắt buộc nào thất bại. Thêm đối chứng lỗi `ls/find` sau khi SSH thành công và lỗi một truy vấn API ở giữa khối, ngoài đối chứng SSH 255.

## F07 — P2: Gate rollout có thể PASS dù không thực hiện được rollback/phát hành

> **Trạng thái 14/10/2026: ✅ Đúng — đã sửa.** 5a B3: `rollout undo` và `deploy.sh` phải trả mã 0; sau chặng 1 template phải trỏ ConfigMap cũ, sau chặng 2 trỏ lại ConfigMap mới, revision tăng sau mỗi chặng; probe luôn được dừng (`wait "$PROBE"`) kể cả khi một chặng hỏng; PASS tách hai dòng — thao tác đã xảy ra, rồi mới tới số mẫu và 0 lỗi. Dòng kiểm lý do phát hành trong lịch sử rollout cũng đọc trước rồi mới xét. 12 B6: `rollout restart` và `rollout status` phải thành công, chờ tới khi không còn UID Pod `api` nào của trước khi thay, revision tăng; số mẫu tối thiểu đọc từ dòng tóm tắt. Nghiệm thu của hai ticket ghi lại theo đúng hai bước này.

**Trạng thái:** đã bổ sung phép đo request, nhưng gate chưa đủ để đóng phát hiện.

Nguồn: [5a thao tác undo và deploy](E:/courses/Ansible/ansible_playbook_k8s-installation/k8s-docs/labs/LAB-5A-SERVICE-ENDPOINTSLICE-VA-DNS.md:997), [5a điều kiện PASS](E:/courses/Ansible/ansible_playbook_k8s-installation/k8s-docs/labs/LAB-5A-SERVICE-ENDPOINTSLICE-VA-DNS.md:1004), [12 B6](E:/courses/Ansible/ansible_playbook_k8s-installation/k8s-docs/labs/LAB-12-VAN-HANH-VONG-DOI-NODE.md:1415).

Ở 5a, mã thoát của `rollout undo` và `deploy.sh` không được đưa vào điều kiện nghiệm thu. `rollout status` có thể thành công cho Deployment hiện đang ổn định, dù thao tác trước đó thất bại. Kiểm ConfigMap cuối bằng NEW cũng không phân biệt được hệ thống chưa từng rollback với hệ thống rollback rồi phát hành lại thành công.

Đối chứng giữ nguyên logic gate, thay đường ghi file bằng thư mục tạm và các tác vụ ngoài bằng hàm giả: undo trả 1, deploy trả 1, rollout status vẫn trả 0 cho trạng thái ổn định, mọi HTTP trả 200. Có đủ 34 mẫu và gate vẫn in:

```text
PASS: rollback va phat hanh lai web duoi phep do lien tuc: 34 mau, 0 loi tren /
PASS: web tro lai dung cau hinh trong repo
```

12 B6 có cùng khoảng trống: không chặn khi `rollout restart`/`rollout status` lỗi và chỉ đọc số request lỗi để kết luận đã thay toàn bộ Pod API.

**Cần sửa:** yêu cầu từng thao tác undo/deploy/restart/status thành công trước khi kết luận; kiểm revision hoặc Pod UID/template thực sự thay đổi theo từng chặng. Probe phải được dừng và dọn khi thao tác lỗi. Thêm đối chứng thao tác phát hành thất bại trong lúc hệ thống cũ vẫn trả HTTP 200: gate phải FAIL. Sau đó mới dùng số mẫu, lỗi và thời gian để chứng minh R4.

## F08 — P2: Gate giữa phiên chung không phù hợp mọi trạng thái bàn giao

> **Trạng thái 14/10/2026: ✅ Đúng — đã sửa.** README 7.5: gate tiếp tục ghi ở mục 1.2 của chính lab là chuẩn; bảng theo mốc — mặc định A5.5 + `deploy.sh` (bỏ smoke của tag trước vì ticket phiên 1 có thể đã đổi hợp đồng), 12/13/14 có kiểm riêng, 5b (nền mạng mới, Bookstore đã gỡ), 8a, 8b (bộ VM HA), 8c (chưa có Kubernetes). Chỉ chạy lại nghiệm thu của ticket cuối khi đó là *Tính năng* — chạy lại *Diễn tập* là gây lỗi lần nữa. 5b có khối gate chỉ đọc (Calico, Traefik, không còn Flannel, chưa có namespace dự án); 8b, 8c chạy lại nghiệm thu chỉ đọc của B6 — 8c kiểm repo trên `lab-ha-4`, nơi giữ repo. CLAUDE.md ghi quy tắc: mục 1.2 phải có điểm dừng và gate tiếp tục khớp trạng thái.

**Trạng thái:** quy định thời lượng đã được thống nhất theo phiên; còn lỗi khả thi của gate mới.

Nguồn: [README gate đầu phiên 2](E:/courses/Ansible/ansible_playbook_k8s-installation/k8s-docs/labs/README.md:312), [5b điểm dừng giữa phiên](E:/courses/Ansible/ansible_playbook_k8s-installation/k8s-docs/labs/LAB-5B-NETWORKPOLICY-INGRESS-VA-CNI.md:116), [8c điểm dừng giữa phiên](E:/courses/Ansible/ansible_playbook_k8s-installation/k8s-docs/labs/LAB-8C-HA-VOI-EXTERNAL-ETCD.md:119).

README yêu cầu mọi đầu phiên 2 đều PASS A5.5 và `tests/smoke.sh` hiện có. Nhưng hai điểm dừng có chủ đích không đáp ứng được điều kiện đó:

- 5b kết thúc phiên 1 sau khi đã gỡ Bookstore và cài Calico/Traefik; dự án chưa triển khai lại. Smoke của 5a sẽ fail vì workload ứng dụng chưa tồn tại.
- 8c kết thúc phiên 1 với etcd, **chưa có Kubernetes**. A5.5 và smoke Bookstore chưa thể chạy. Lab 8c đã có hướng dẫn đúng cho mốc này: đầu phiên 2 kiểm lại bằng script B6.

Đây là mâu thuẫn giữa quy tắc chung và trạng thái được chính lab thiết kế, không phải lý do phải triển khai workload sớm để vượt gate.

**Cần sửa:** ghi ngoại lệ và gate tiếp tục theo mốc của từng lab. A5.5/smoke ứng dụng chỉ áp dụng khi cluster và Bookstore đã được dựng tới mức đó; nhánh HA dùng đúng kubeconfig, topology và script phù hợp. Đồng bộ README với hướng dẫn giữa phiên hiện có, thay vì yêu cầu mọi lab chạy cùng bốn gate vô điều kiện. Thời lượng vẫn cần số đo pilot, nhưng không còn giữ lại nhận xét cũ về ngưỡng 2–4 giờ cho toàn lab.

## F11 — P2: Smoke kế thừa làm mất kiểm retention/file tạm mới bổ sung

> **Trạng thái 14/10/2026: ✅ Đúng — đã sửa, phạm vi rộng hơn báo cáo.** Khối đọc PVC của smoke 9a (Pod đạt PSS `restricted`, có `resources`) được chép nguyên văn vào smoke 9b, 11a, 11b, 12. Rà thêm cho thấy 7a, 7b, 8a cũng mất kiểm số bản đã có ở 6a/6b — đã chèn cùng khối. Smoke 6a, 6b có kiểm số bản nhưng chưa kiểm file tạm cũ — đã bổ sung, và Pod đọc không xong là FAIL thay vì "0 bản". Từ 6a tới 14, mọi bản smoke của chuỗi chính đều kiểm hợp đồng ghi file backup; smoke HA của 8b dùng overlay `emptyDir`, không có PVC `backup-store`.

**Phát hiện mới trong lần kiểm lại.**

Nguồn: [smoke 9a có kiểm mới](E:/courses/Ansible/ansible_playbook_k8s-installation/k8s-docs/labs/LAB-9A-SERVICEACCOUNT-AUTHN-AUTHZ-VA-RBAC.md:2489), [smoke 9b](E:/courses/Ansible/ansible_playbook_k8s-installation/k8s-docs/labs/LAB-9B-POD-SECURITY-VA-HARDENING.md:3041), [smoke 11a](E:/courses/Ansible/ansible_playbook_k8s-installation/k8s-docs/labs/LAB-11A-OBSERVABILITY.md:2273), [smoke 11b](E:/courses/Ansible/ansible_playbook_k8s-installation/k8s-docs/labs/LAB-11B-HPA-VA-VPA.md:2110), [smoke 12](E:/courses/Ansible/ansible_playbook_k8s-installation/k8s-docs/labs/LAB-12-VAN-HANH-VONG-DOI-NODE.md:2046).

9a đã thêm Pod đọc PVC để kiểm tối đa 10 bản hoàn chỉnh và không có file tạm cũ. Khi thay `tests/smoke.sh` bằng các bản 9b, 11a, 11b và 12, các phép kiểm này biến mất. 14 mới thêm chúng trở lại. Đọc trực tiếp phần thân từng smoke xác nhận bốn bản giữa chuỗi không có kiểm số bản hoặc tuổi file tạm.

Như vậy logic backup đã sửa nhưng hồi quy retention có thể lọt qua nhiều sprint, trái với [README 7.3, nguyên tắc smoke tích lũy](E:/courses/Ansible/ansible_playbook_k8s-installation/k8s-docs/labs/README.md:241) và [hợp đồng backup](E:/courses/Ansible/ansible_playbook_k8s-installation/k8s-docs/labs/DU-AN-BOOKSTORE.md:390).

**Cần sửa:** giữ phép kiểm đọc PVC trong mọi bản smoke kế thừa từ 9a, hoặc dùng helper chung mà từng bản smoke gọi. Kiểm Pod đọc phù hợp PSS/quota của sprint hiện tại. Đối chứng hơn 10 bản hoặc một file tạm cũ hơn một giờ phải làm smoke fail ở 9b, 11a, 11b và 12, không đợi tới Lab 14.

## Thứ tự xử lý còn lại

1. F02: mọi lỗi kiểm kê phải STOP trước khi xác nhận sạch/chụp snapshot.
2. F07: gate phải chứng minh đã thực hiện thay đổi, rồi mới đánh giá số đo request.
3. F08: chốt gate tiếp tục cho các mốc chưa có cluster/app, giữ đúng thứ tự học.
4. F11: bảo toàn phép kiểm backup qua mọi bản smoke kế thừa.

Sau các chỉnh sửa này, pilot vẫn cần kiểm hành vi trên VM, quyền filesystem, thời gian hội tụ và thời lượng người học. Việc chưa pilot là giới hạn kiểm chứng chung; không dùng nó để giữ lại các phát hiện về source đã được sửa thành công.
