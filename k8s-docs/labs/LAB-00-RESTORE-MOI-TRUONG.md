# Lab 00 — Restore môi trường: hồ sơ sự cố etcd mất dữ liệu

> **Loại tài liệu:** hồ sơ một lần điều tra và khôi phục có thật. Không phải bài học trong lộ
> trình: không có checkpoint vấn đáp, không tạo snapshot mới.
> **Cluster:** cluster VM dựng theo [Lab 00](LAB-00-MOI-TRUONG-1.35.7.md) — `lab-k8s-master`,
> `lab-k8s-worker1`, `lab-k8s-worker2`.
> **Phát hiện và khôi phục:** 31/10/2026.
> **Kết quả:** cả ba VM về snapshot `01-cluster-ready`, gate A5.5 PASS. Mọi thay đổi trong ba VM
> sau ngày chụp snapshot (16/08/2026) bị mất.

Giờ trong VM là UTC: log ghi hậu tố `Z` hoặc `+0000`. Giờ hiển thị trong VMware Workstation là
giờ của máy host Windows.

---

## 1. Tóm tắt

Ba VM bị tắt không rõ nguyên nhân. Sau khi bật lại, `kubectl` trên master báo
`connection refused`. Nguyên nhân là file dữ liệu `db` của etcd đã mất nội dung. etcd dừng hẳn
mỗi lần khởi động, và kube-apiserver cần etcd nên chết theo. containerd và kubelet vẫn chạy, nên
nhìn từ ngoài tưởng máy không lỗi.

Dữ liệu không cứu được từ chính VM. Cluster được khôi phục bằng cách revert cả ba VM về snapshot
`01-cluster-ready`, đúng đường lùi mà
[mục 4 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#4-troubleshooting-môi-trường) quy định.

Chuỗi nhân quả:

1. Lúc 17:23:32 UTC ngày 20/09/2026, etcd ghi lần cuối. Sau đó VM dừng, và file `db` mất dữ liệu.
2. Khi boot lại, `db` báo nó ở mục 0, trong khi ghi chép của etcd đã tới mục 3.210.324. etcd tìm
   một bản dữ liệu đầy đủ để chép đè lên `db`, không thấy, và `panic`.
3. kube-apiserver không kết nối được etcd nên chết lặp lại. Không có gì nghe ở cổng `6443`.
4. `kubectl` gọi `192.168.100.221:6443` và bị từ chối.

---

## 2. Hiện trường

### 2.1. Triệu chứng

Trên `lab-k8s-master`, sau khi bật lại ba VM:

```text
ubuntu@lab-k8s-master:~$ kubectl get nodes
E1031 15:45:36.376497    2227 memcache.go:265] "Unhandled Error" err="couldn't get current server API group list: Get \"https://lab-k8s-master:6443/api?timeout=32s\": dial tcp 192.168.100.221:6443: connect: connection refused"
… (dòng trên lặp lại thêm bốn lần)
The connection to the server lab-k8s-master:6443 was refused - did you specify the right host or port?
```

### 2.2. Thông tin có lúc bắt đầu

- Ba VM đã bị tắt không rõ nguyên nhân, sau đó được bật lại bằng tay.
- Trên master, `containerd` và `kubelet` đang chạy (người vận hành tự kiểm).
- Cluster từng chạy tốt sau khi dựng theo Lab 00.
- Cả ba VM có snapshot `01-cluster-ready`.

---

## 3. Hướng điều tra

### 3.1. Đọc lỗi

`connection refused` nghĩa là máy ở `192.168.100.221` có trả lời, nhưng không có chương trình nào
nghe ở cổng `6443`. Tên `lab-k8s-master` vẫn phân giải đúng IP. Vậy hướng đầu tiên không phải mạng
hay DNS, mà là vì sao kube-apiserver không chạy.

**Dễ hiểu nhầm:** `containerd` và `kubelet` `active` không có nghĩa là control plane đang chạy.
etcd và kube-apiserver là static Pod. kubelet đọc file trong `/etc/kubernetes/manifests` rồi nhờ
containerd chạy chúng. Hai container này có thể chết và khởi động lại liên tục trong khi kubelet
vẫn báo `active`.

### 3.2. Nghi phạm ban đầu

Xếp theo khả năng khi VM bị tắt đột ngột:

| # | Nghi phạm | Cơ chế gây lỗi |
| --- | --- | --- |
| 1 | Dữ liệu etcd hỏng | Tắt ngang lúc đang ghi → etcd không khởi động → apiserver chết theo |
| 2 | Môi trường đổi sau khi boot | Swap bật lại, đồng hồ lệch, đĩa đầy, ổ `/` chuyển read-only |
| 3 | IP của master đổi | etcd và apiserver không bind được `192.168.100.221` |
| 4 | Certificate hết hạn | Kết nối TLS giữa các thành phần bị từ chối |

### 3.3. Nguyên tắc trong lúc điều tra

- Chỉ thu bằng chứng, chưa sửa gì.
- Không chạy `kubeadm reset`. Không xóa hay sửa gì trong `/var/lib/etcd`.
- Đi từ ngoài vào trong: cổng → container → log → file dữ liệu.
- Container nào phụ thuộc container khác thì đọc log của thành phần gốc trước.

---

## 4. Diễn biến điều tra

Mọi lệnh trong mục này chạy trên `lab-k8s-master`.

### 4.1. Bước 1 — Cổng 6443 và 2379 có ai nghe không

```bash
sudo ss -tlnp | grep -E ':6443|:2379'
```

**Kết quả:** không có output.

**Kết luận:** cả kube-apiserver (cổng `6443`) lẫn etcd (cổng `2379`) đều không chạy. Câu hỏi tiếp
theo: chúng đang chết lặp lại, hay kubelet chưa từng dựng chúng.

### 4.2. Bước 2 — Trạng thái container control plane

```bash
sudo crictl ps -a | grep -E 'CONTAINER|etcd|kube-apiserver|kube-controller|kube-scheduler'
```

**Kết quả:**

```text
CONTAINER       IMAGE           CREATED          STATE     NAME                      ATTEMPT   POD ID          POD                                      NAMESPACE
2759495373312   0a108f7189562   24 seconds ago   Exited    etcd                      78        3c05321fbd025   etcd-lab-k8s-master                      kube-system
6d72f7907f991   3b05d87ef3ca5   3 minutes ago    Exited    kube-apiserver            74        c01d217c34ea3   kube-apiserver-lab-k8s-master            kube-system
7f6f75da36a22   af4ba4e4da63f   16 minutes ago   Running   kube-scheduler            8         42f63890cfa15   kube-scheduler-lab-k8s-master            kube-system
f85e97a595450   9b18ff7513a51   16 minutes ago   Running   kube-controller-manager   8         811fc719652bb   kube-controller-manager-lab-k8s-master   kube-system
68ef88878d770   9b18ff7513a51   12 hours ago     Exited    kube-controller-manager   7         82d7f87e97dc4   kube-controller-manager-lab-k8s-master   kube-system
c84c99d620b22   af4ba4e4da63f   12 hours ago     Exited    kube-scheduler            7         9c7585f0d6943   kube-scheduler-lab-k8s-master            kube-system
```

**Đọc kết quả:**

- etcd `Exited`, `ATTEMPT` 78. Lần chạy lệnh trước đó khoảng hai phút, `ATTEMPT` của etcd là 77.
  etcd đang chết và được kubelet khởi động lại liên tục.
- kube-apiserver `Exited`, `ATTEMPT` 74.
- kube-scheduler và kube-controller-manager `Running`, nhưng không làm được gì khi thiếu
  apiserver.
- Hai container `Exited` tạo "12 hours ago" thuộc một lần boot trước. Xem
  [mục 8.2](#82-lần-boot-khoảng-12-giờ-trước-đó).

**Kết luận:** kubelet vẫn dựng static Pod đều đặn, nên lỗi không nằm ở kubelet. apiserver phụ
thuộc etcd, nên đọc log etcd trước.

### 4.3. Bước 3 — Log của etcd

```bash
sudo crictl logs --tail 50 2759495373312
```

**Kết quả** — trích các dòng quyết định; trường `caller` và các trường không liên quan thay bằng
`…`:

```text
{"level":"info","ts":"2026-10-31T15:59:19.622467Z",…,"msg":"server has already been initialized","data-dir":"/var/lib/etcd","dir-type":"member"}
{"level":"info","ts":"2026-10-31T15:59:19.622492Z",…,"msg":"configuring peer listeners","listen-peer-urls":["https://192.168.100.221:2380"]}
{"level":"info","ts":"2026-10-31T15:59:19.624240Z",…,"msg":"configuring client listeners","listen-client-urls":["https://127.0.0.1:2379","https://192.168.100.221:2379"]}
{"level":"info","ts":"2026-10-31T15:59:19.625884Z",…,"msg":"opened backend db","path":"/var/lib/etcd/member/snap/db","took":"831.43µs"}
{"level":"info","ts":"2026-10-31T15:59:19.625900Z",…,"msg":"restore consistentIndex","index":0}
{"level":"info","ts":"2026-10-31T15:59:20.117865Z",…,"msg":"recovered v2 store from snapshot","snapshot-index":3210324,"snapshot-size":"7.2 kB"}
{"level":"info","ts":"2026-10-31T15:59:20.117915Z",…,"msg":"Recovering from snapshot backend","consistent-index":0,"snapshot-index":3210324}
{"level":"warn","ts":"2026-10-31T15:59:20.118643Z",…,"msg":"failed to find [SNAPSHOT-INDEX].snap.db","snapshot-index":3210324,"snapshot-file-path":"/var/lib/etcd/member/snap/000000000030fc54.snap.db","error":"snap: snapshot file doesn't exist"}
{"level":"panic","ts":"2026-10-31T15:59:20.118680Z",…,"msg":"failed to recover v3 backend from snapshot","error":"failed to find database snapshot file (snap: snapshot file doesn't exist)",…}
panic: failed to recover v3 backend from snapshot
```

**Đọc kết quả:**

- etcd dựng listener trên `192.168.100.221:2380` và `192.168.100.221:2379` mà không báo lỗi. Master
  vẫn giữ đúng IP. Loại nghi phạm 3.
- `opened backend db` thành công: file `db` có tồn tại và mở được.
- `restore consistentIndex` với `index: 0`: file `db` báo nó mới ở mục 0.
- `recovered v2 store from snapshot` với `snapshot-index: 3210324`: ghi chép `.snap` của etcd báo
  dữ liệu đã tới mục 3.210.324.
- Vì `db` tụt lại so với ghi chép, etcd tìm file `000000000030fc54.snap.db` để chép đè lên `db`.
  Không có file đó nên etcd `panic`. `30fc54` chính là 3210324 viết theo hệ 16.

**Dễ hiểu nhầm:** `snapshot file doesn't exist` không nói tới snapshot của VMware hay file backup
etcd. File `<index>.snap.db` chỉ xuất hiện khi etcd nhận dữ liệu từ một etcd khác qua mạng. Cluster
một etcd như lab này gần như không bao giờ có file đó. Lỗi thật là `consistent-index: 0`: `db` đã
mất dữ liệu.

**Kết luận:** nghi phạm 1 đúng. Cần xác nhận `db` thật sự rỗng trước khi chọn cách khôi phục.

### 4.4. Bước 4 — File dữ liệu etcd và nhật ký kernel

```bash
sudo ls -la --time-style=full-iso /var/lib/etcd/member/snap/ /var/lib/etcd/member/wal/
sudo du -sh /var/lib/etcd
sudo journalctl -k -b | grep -iE 'EXT4-fs|I/O error'
```

**Kết quả:**

```text
/var/lib/etcd/member/snap/:
total 2632
drwx------ 2 root root     4096 2026-10-31 16:04:30.637285644 +0000 .
drwx------ 4 root root     4096 2026-10-31 16:04:30.637285644 +0000 ..
-rw-r--r-- 1 root root     7165 2026-09-20 09:57:21.973471247 +0000 0000000000000007-0000000000306010.snap
-rw-r--r-- 1 root root     7165 2026-09-20 11:35:02.041712838 +0000 0000000000000007-0000000000308721.snap
-rw-r--r-- 1 root root     7165 2026-09-20 13:12:45.446315389 +0000 0000000000000007-000000000030ae32.snap
-rw-r--r-- 1 root root     7165 2026-09-20 14:50:26.763621455 +0000 0000000000000007-000000000030d543.snap
-rw-r--r-- 1 root root     7165 2026-09-20 16:28:07.654828418 +0000 0000000000000007-000000000030fc54.snap
-rw------- 1 root root 16801792 2026-10-31 16:04:31.132310497 +0000 db

/var/lib/etcd/member/wal/:
total 375024
drwx------ 2 root root     4096 2026-09-20 06:32:05.161023536 +0000 .
drwx------ 4 root root     4096 2026-10-31 16:04:30.637285644 +0000 ..
-rw------- 1 root root 64000152 2026-09-18 07:58:13.078649648 +0000 000000000000001d-00000000002a3c7e.wal
-rw------- 1 root root 64000552 2026-09-18 23:29:44.586807312 +0000 000000000000001e-00000000002bb0a9.wal
-rw------- 1 root root 64000896 2026-09-19 15:00:08.661899458 +0000 000000000000001f-00000000002d251c.wal
-rw------- 1 root root 64000680 2026-09-20 06:31:54.678565740 +0000 0000000000000020-00000000002e993f.wal
-rw------- 1 root root 64000000 2026-09-20 17:23:32.710887531 +0000 0000000000000021-0000000000300de4.wal
-rw------- 1 root root 64000000 2026-09-20 06:31:54.680565827 +0000 0.tmp
369M    /var/lib/etcd
Oct 31 15:43:31 lab-k8s-master kernel: EXT4-fs (dm-0): mounted filesystem b22d59e6-30e3-4557-a36b-b184a96f9e44 ro with ordered data mode. Quota mode: none.
Oct 31 15:43:31 lab-k8s-master kernel: EXT4-fs (dm-0): re-mounted b22d59e6-30e3-4557-a36b-b184a96f9e44 r/w.
Oct 31 15:43:32 lab-k8s-master kernel: EXT4-fs (sda2): mounted filesystem 38d39f5c-1672-4982-bd11-66a9c2b0c84b r/w with ordered data mode. Quota mode: none.
```

**Đọc kết quả:**

- **`db` gần như rỗng.** File nặng 16.801.792 byte, bằng đúng 16 MiB (16.777.216) cộng 24 KiB
  (24.576). bbolt, thư viện etcd dùng để lưu `db`, mỗi lần nới file sẽ cộng thêm một khoảng trống
  dự phòng 16 MiB. Phần dữ liệu thật chỉ còn 24 KiB, tức sáu trang 4 KiB. Con số này khớp với
  `consistent-index: 0` ở bước 3. Giờ sửa đổi 16:04:31 ngày 31/10 là lần etcd chạy thử gần nhất,
  không cho biết lúc dữ liệu bị mất.
- **WAL không cứu được.** `du` báo 369M, nghe như dữ liệu còn, nhưng gần hết là WAL: năm file
  `.wal` và một file `0.tmp`, mỗi file được dành sẵn khoảng 64 MB. WAL chỉ ghi các thay đổi từ mục
  `0x2a3c7e` (2.767.998) trở đi, không chứa toàn bộ trạng thái từ khi dựng cluster. Không dựng lại
  được `db` từ WAL.
- **Mốc thời gian.** Mọi file `.snap` và `.wal` dừng ở ngày 20/09/2026. File `.snap` cuối lúc
  16:28:07 UTC, đúng mục 3.210.324 trong log bước 3. Lần ghi WAL cuối lúc 17:23:32 UTC. Đó là lần
  cuối etcd chạy khỏe, vì etcd đang chạy thì ghi WAL liên tục. Sự cố xảy ra quanh thời điểm này,
  không phải ngày 31/10. Nếu máy host đặt giờ Việt Nam (UTC+7), đó là khoảng 00:23 ngày 21/09.
- **Ổ đĩa.** Lần boot 31/10 mount ổ `/` (`dm-0`) rồi chuyển sang `r/w`, không có dòng `recovery`.
  Ổ không bị read-only, và đã được unmount sạch ở lần tắt máy ngay trước lần boot này. Nếu sự cố
  20/09 là tắt ngang, việc sửa ổ đã diễn ra ở một lần boot khác — xem
  [mục 8.2](#82-lần-boot-khoảng-12-giờ-trước-đó).

**Kết luận:** dữ liệu cluster không còn trong VM. Chỉ khôi phục được từ bản sao lưu.

---

## 5. Kết luận nguyên nhân

### 5.1. Bằng chứng và suy luận

| Khẳng định | Loại | Dựa vào |
| --- | --- | --- |
| kube-apiserver và etcd không chạy | Bằng chứng | Bước 1: không có gì nghe ở `6443`, `2379` |
| etcd chết lặp lại, apiserver chết theo | Bằng chứng | Bước 2: etcd `Exited`, `ATTEMPT` 77 → 78; apiserver `Exited`, `ATTEMPT` 74 |
| etcd dừng vì `db` ở mục 0 trong khi `.snap` ở mục 3.210.324 | Bằng chứng | Bước 3: `restore consistentIndex` `index: 0`; `panic: failed to recover v3 backend from snapshot` |
| IP của master không đổi | Bằng chứng | Bước 3: etcd dựng listener trên `192.168.100.221` không lỗi |
| Ổ `/` không read-only | Bằng chứng | Bước 4: `re-mounted … r/w` |
| `db` gần như rỗng | Suy luận mạnh | `consistent-index: 0` cộng kích thước đúng 16 MiB + 24 KiB |
| Sự cố xảy ra quanh 17:23 UTC ngày 20/09 | Suy luận mạnh | Lần ghi WAL cuối; etcd chạy thì ghi WAL liên tục |
| VM bị tắt ngang làm mất nội dung `db` | Giả thuyết | Chưa kiểm chứng; bằng chứng trong VM đã mất khi revert |

### 5.2. Nghi phạm không kiểm trực tiếp

- **Swap:** loại gián tiếp. kubelet vẫn tạo container liên tục (`ATTEMPT` tăng), tức kubelet không
  bị swap chặn lúc khởi động.
- **Đồng hồ và certificate:** không kiểm. Log etcd đã chỉ thẳng vào nguyên nhân, và sau khi khôi
  phục, `readyz` trả `ok`.
- **Đĩa đầy:** không kiểm bằng `df`. Log etcd không có `no space left`.

---

## 6. Quyết định khôi phục

### 6.1. Các phương án

Kubernetes lưu mọi object trong etcd. Khi `db` rỗng, dữ liệu chỉ quay lại được từ bản sao lưu:

| Phương án | Điều kiện | Mất gì |
| --- | --- | --- |
| Revert cả ba VM về `01-cluster-ready` | Có snapshot trên cả ba VM | Mọi thay đổi trong VM sau 16/08/2026 |
| Khôi phục etcd từ file backup | Có file backup etcd | Thay đổi sau lúc backup. Quy trình thuộc [giai đoạn 19](../00-ALO-TRINH-ADMIN.md#giai-đoạn-19--etcd-backup-và-khôi-phục-thảm-họa), [nợ lab #8](README.md#5-sổ-nợ-lab) |
| Dựng lại cluster | Không có cả hai | Toàn bộ cluster |

### 6.2. Snapshot hiện có

Đọc trong menu **Snapshot** của từng VM trên VMware Workstation:

| VM | Snapshot | Thời điểm chụp (giờ máy host) |
| --- | --- | --- |
| `lab-k8s-master` | `01-cluster-ready` | 16/08/2026 18:50:01 |
| `lab-k8s-worker1` | `01-cluster-ready` | 16/08/2026 18:50:15 |
| `lab-k8s-worker2` | `01-cluster-ready` | 16/08/2026 18:51:00 |

Ba snapshot cùng tên và được chụp cách nhau chưa tới một phút. Đây là một bộ nhất quán, revert
cùng nhau được.

### 6.3. Lựa chọn

Chọn revert cả ba VM về `01-cluster-ready`, theo
[mục 4 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#4-troubleshooting-môi-trường): không restore riêng
master, không debug một cluster đã lệch state.

Bước dựng lại dòng thời gian trong VM ([mục 8.1](#81-vì-sao-vm-dừng-quanh-2009)) được bỏ qua để
khôi phục ngay. Đánh đổi: nhật ký trong VM mất theo khi revert, nên nguyên nhân VM tự tắt chỉ còn
tìm được từ phía máy host.

---

## 7. Khôi phục

### 7.1. Quy trình

Theo [mục 4](LAB-00-MOI-TRUONG-1.35.7.md#4-troubleshooting-môi-trường) và
[A5.5](LAB-00-MOI-TRUONG-1.35.7.md#a55-quy-trình-mở-đầu-mỗi-lab) của Lab 00.

1. **Chép ra ngoài những file cần giữ.** Revert đưa toàn bộ ổ đĩa VM về ngày 16/08, không chỉ sửa
   etcd. Master vẫn SSH được vì chỉ etcd hỏng. Kiểm `~/bookstore`, `~/lab-evidence/`,
   `~/lab-work/` trên cả ba node. Cần giữ thì chép về máy host, ví dụ từ PowerShell:

   ```powershell
   scp -r ubuntu@192.168.100.221:~/lab-evidence .\lab-evidence-master
   ```

2. **Tắt sạch ba VM** theo thứ tự worker 2 → worker 1 → master. Trên từng node:

   ```bash
   sudo shutdown -h now
   ```

   Chờ VMware Workstation hiện cả ba VM ở trạng thái *Powered off*.

3. **Revert từng VM:** chuột phải VM → **Snapshot → Revert to Snapshot: 01-cluster-ready** → xác
   nhận. Làm đủ cả ba VM. Không cần chờ một khoảng thời gian cố định: revert xong khi thanh tiến
   trình biến mất, và trong **Snapshot Manager** mốc *You Are Here* nằm ngay dưới
   `01-cluster-ready`. Snapshot được chụp lúc VM tắt, nên sau khi revert VM vẫn tắt.

4. **Bật máy và chạy gate A5.5.** Bật master → worker 1 → worker 2. Đợi một hai phút cho control
   plane lên, rồi chạy trên `lab-k8s-master`:

   ```bash
   kubectl wait --for=condition=Ready node --all --timeout=180s
   test "$(kubectl get --raw='/readyz')" = 'ok' && echo 'PASS: readyz ok'
   kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded
   kubectl -n kube-system get deployment coredns
   kubectl get pods -n default
   ```

   **PASS:** ba node `Ready`; dòng `PASS: readyz ok` xuất hiện; lệnh `--field-selector` trả
   `No resources found`; CoreDNS đủ replica `READY`; namespace `default` không còn Pod nào.

   Nếu `kubectl` báo `connection refused` ngay sau boot, chờ thêm rồi chạy lại. Nếu gate không
   PASS, chạy đủ bảy tầng của [A5.4](LAB-00-MOI-TRUONG-1.35.7.md#a54-gate-01-cluster-ready).

### 7.2. Kết quả

```text
node/lab-k8s-master condition met
node/lab-k8s-worker1 condition met
node/lab-k8s-worker2 condition met
PASS: readyz ok
No resources found
NAME      READY   UP-TO-DATE   AVAILABLE   AGE
coredns   2/2     2            2           76d
No resources found in default namespace.
```

Đạt đủ năm điều kiện PASS. CoreDNS có `AGE 76d`, khớp với khoảng từ 16/08 tới 31/10: cluster đang
chạy đúng dữ liệu của snapshot `01-cluster-ready`.

---

## 8. Câu hỏi còn mở

### 8.1. Vì sao VM dừng quanh 20/09

Chưa rõ. Nhật ký trong VM đã mất khi revert. Phía máy host Windows vẫn kiểm được vì log nằm ngoài
VM. Chạy trong PowerShell trên máy host:

```powershell
Get-WinEvent -FilterHashtable @{LogName='System'; Id=41,1074,6006,6008; StartTime='2026-09-20'; EndTime='2026-09-22'} | Format-List TimeCreated, Id, Message
```

- `1074`: có người hoặc chương trình ra lệnh restart; `Message` ghi rõ là ai, ví dụ Windows Update.
- `41` hoặc `6008`: chính Windows bị tắt ngang.
- `6006`: Windows tắt êm.
- `No events were found`: Windows không tắt trong khoảng đó; nguyên nhân nằm ở VMware hoặc trong
  VM.

Nếu sự cố lặp lại, chạy hai lệnh sau trên VM **trước** khi revert để dựng dòng thời gian:

```bash
journalctl --list-boots --no-pager
sudo journalctl -b <số ở cột IDX của lần boot cần xem> -n 30 --no-pager
```

Lệnh đầu liệt kê các lần boot kèm giờ bắt đầu và kết thúc. Lệnh sau in cuối nhật ký của một lần
boot. Nhật ký kết thúc bằng các dòng shutdown của systemd là tắt êm. Nhật ký dừng giữa chừng, giữa
các dòng log của kubelet hay etcd, là tắt ngang.

### 8.2. Lần boot khoảng 12 giờ trước đó

Ở bước 2, `crictl` còn hai container `Exited` tạo "12 hours ago", với `ATTEMPT` 7. Theo cột
`CREATED`, chúng được tạo khoảng 04:00 UTC ngày 31/10, trước lần boot 15:43. Vậy giữa 20/09 và lần
boot 15:43 ngày 31/10 có ít nhất một lần boot khác. Điều này có thể giải thích vì sao lần boot
15:43 không phải sửa ổ đĩa. Chưa rõ ai hoặc cái gì đã bật VM lúc đó.

### 8.3. Vì sao `db` mất dữ liệu trong khi WAL và `.snap` còn

Chưa chứng minh được. Giả thuyết là VM bị tắt ngang khiến phần dữ liệu của `db` chưa kịp xuống đĩa.
Bằng chứng để kiểm giả thuyết này nằm trong nhật ký và file của VM, đã mất khi revert.

---

## 9. Bài học

1. **`kubelet` `active` không có nghĩa là control plane chạy.** Control plane là static Pod. Kiểm
   bằng `crictl ps -a`, nhìn cột `STATE` và `ATTEMPT`.
2. **Đọc log của thành phần gốc trước nạn nhân.** apiserver chết vì etcd chết; log apiserver chỉ
   lặp lại lỗi không kết nối được etcd.
3. **`snapshot file doesn't exist` là file nội bộ của etcd**, không phải snapshot VMware hay file
   backup.
4. **Thư mục etcd nặng không có nghĩa là dữ liệu còn.** WAL được dành sẵn chỗ và chỉ ghi thay đổi
   gần đây; dữ liệu đầy đủ nằm trong `db`.
5. **Revert snapshot xóa cả bằng chứng.** Muốn biết nguyên nhân thì thu nhật ký trong VM trước,
   rồi mới revert.
6. **Cluster một control plane chỉ có một bản etcd.** VM snapshot là đường lùi duy nhất của lab
   hiện tại, và nó làm mất mọi thứ sau ngày chụp. Sao lưu etcd riêng là nội dung của
   [giai đoạn 19](../00-ALO-TRINH-ADMIN.md#giai-đoạn-19--etcd-backup-và-khôi-phục-thảm-họa).

---

## 10. Tra nhanh khi gặp lại

Triệu chứng: `kubectl` báo `connection refused` tới `:6443` sau khi VM bị tắt rồi bật lại. Chạy
trên `lab-k8s-master`:

```bash
# 1. Có gì nghe ở cổng apiserver và etcd không
sudo ss -tlnp | grep -E ':6443|:2379'
# 2. Container control plane đang ở trạng thái nào
sudo crictl ps -a | grep -E 'CONTAINER|etcd|kube-apiserver|kube-controller|kube-scheduler'
# 3. Log của container gốc đang chết: etcd trước, apiserver sau
sudo crictl logs --tail 50 <CONTAINER_ID>
# 4. File dữ liệu etcd và tình trạng ổ đĩa
sudo ls -la --time-style=full-iso /var/lib/etcd/member/snap/ /var/lib/etcd/member/wal/
sudo journalctl -k -b | grep -iE 'EXT4-fs|I/O error'
```

| Thấy | Nghĩa là | Làm tiếp |
| --- | --- | --- |
| Không có gì ở `6443` lẫn `2379` | apiserver và etcd không chạy | Lệnh 2 |
| etcd `Exited`, `ATTEMPT` tăng | etcd là gốc | Lệnh 3 với container etcd |
| Không có container control plane nào | kubelet không dựng được static Pod | `sudo journalctl -u kubelet -b --no-pager \| tail -50` |
| Log etcd có `restore consistentIndex` `index: 0` và `failed to find database snapshot file` | `db` mất dữ liệu | Thu nhật ký ([mục 8.1](#81-vì-sao-vm-dừng-quanh-2009)), rồi revert ([mục 7](#7-khôi-phục)) |
| Log etcd có `cannot assign requested address` | IP của master đổi | Kiểm IP và netplan theo [A3 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a3-đặt-ip-tĩnh-và-phân-giải-tên) |
| Log có `x509` | Đồng hồ lệch hoặc certificate hết hạn | `timedatectl`; `sudo openssl x509 -noout -enddate -in /etc/kubernetes/pki/apiserver.crt` |
| Log có `no space left` hoặc `read-only file system` | Lỗi đĩa | `df -h / /var/lib`; `findmnt -no OPTIONS /` |

---

## 11. Nguồn chính thức

Bản dịch trong repo:

- [307 — Debug node Kubernetes bằng crictl](../307-crictl-vi.md)
- [58 — Pod tĩnh](../58-static-pods-vi.md)
- [197 — Vận hành cluster etcd cho Kubernetes](../197-configure-upgrade-etcd-vi.md)
- [09 — Troubleshooting kubeadm](../09-troubleshooting-kubeadm-vi.md)

Trang gốc trên kubernetes.io:

- [Debugging Kubernetes nodes with crictl](https://kubernetes.io/docs/tasks/debug/debug-cluster/crictl/)
- [Static Pods](https://kubernetes.io/docs/concepts/workloads/pods/static-pods/)
- [Operating etcd clusters for Kubernetes](https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/)
- [Troubleshooting kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/troubleshooting-kubeadm/)
