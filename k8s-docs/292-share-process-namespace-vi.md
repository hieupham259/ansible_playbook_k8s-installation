# Chia sẻ Process Namespace giữa các Container trong một Pod (Share Process Namespace between Containers in a Pod)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/configure-pod-container/share-process-namespace/

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 3 — Pod và cấu hình](00-ALO-TRINH-ADMIN.md#giai-đoạn-3--pod-và-cấu-hình)
→ nhóm [3a. Pod và vòng đời](00-ALO-TRINH-ADMIN.md#3a-pod-và-vòng-đời), bài thực hành 7/11 ·
Kiểm chứng ở [Lab 3a](labs/LAB-3A-POD-VA-VONG-DOI.md) phần B8.1.

Bài bật một trường duy nhất, nhưng mục cuối — *Hiểu về chia sẻ process namespace* — mới là phần
phải đọc kỹ: nó liệt kê ba thứ **đổi hành vi** khi bạn bật trường đó, và cả ba đều là bẫy vận
hành.

**Phải hiểu ở lần đọc này:**

- Bật bằng field **`shareProcessNamespace: true`** trong `.spec` của Pod. Hệ quả trực tiếp: tiến
  trình trong một container **hiển thị với tất cả container khác trong cùng Pod** — `ps ax` trong
  container `shell` thấy cả tiến trình nginx.
- **Tiến trình của container không còn mang PID 1.** PID 1 là **pod sandbox** (`/pause` trong ví
  dụ). Container nào đòi PID 1 mới khởi động được — ví dụ container dùng `systemd` — sẽ hỏng; và
  `kill -HUP 1` bắn tín hiệu vào sandbox chứ không vào ứng dụng.
- Gửi tín hiệu sang tiến trình của container khác thì được, nhưng **cần capability `SYS_PTRACE`**
  — trong manifest nó nằm ở `securityContext.capabilities.add` của container `shell`. Ví dụ:
  `kill -HUP 8` để nginx nạp lại worker process.
- Đọc được **hệ thống tập tin của container khác** qua liên kết **`/proc/$pid/root`**, ví dụ
  `head /proc/8/root/etc/nginx/nginx.conf`.
- Cái giá phải trả, chính là hai điểm cuối của mục *Hiểu về chia sẻ process namespace*: mọi thứ
  nhìn thấy trong `/proc` — kể cả **mật khẩu truyền qua argument hoặc biến môi trường** — và cả
  secret nằm trên filesystem, **chỉ còn được bảo vệ bằng quyền Unix và quyền filesystem thông
  thường**. Ranh giới giữa các container trong Pod không còn.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Ý "cấu hình một sidecar container xử lý log" ở đoạn mở đầu | sidecar là một bài riêng của chính nhóm này | bài [51](51-sidecar-containers-vi.md), thực hành ở [Lab 3a](labs/LAB-3A-POD-VA-VONG-DOI.md) phần B6 |
| Ý "khắc phục sự cố các container image không kèm shell" | đó là việc của ephemeral container và `kubectl debug` | bài [52](52-ephemeral-containers-vi.md), thực hành ở [Lab 3a](labs/LAB-3A-POD-VA-VONG-DOI.md) phần B8.2 |
| Capability của Linux kernel **là gì** (ở đây chỉ cần biết `SYS_PTRACE` là thứ phải thêm) | capability, seccomp, AppArmor, SELinux là một chủ đề bảo mật riêng | bài [127](127-linux-kernel-security-vi.md) ở [giai đoạn 9](00-ALO-TRINH-ADMIN.md#giai-đoạn-9--bảo-mật-và-multi-tenancy), thực hành ở Lab 9b |
| `stdin: true` và `tty: true` trong manifest | chỉ để `kubectl exec -it` có shell tương tác | dùng ngay ở [Lab 3a](labs/LAB-3A-POD-VA-VONG-DOI.md) phần B8.1 |

---

Trang này hướng dẫn cách cấu hình chia sẻ process namespace cho một pod. Khi tính năng chia sẻ
process namespace được bật, các process (tiến trình) trong một container sẽ hiển thị với tất cả
các container khác trong cùng pod.

Bạn có thể dùng tính năng này để cấu hình các container phối hợp với nhau, chẳng hạn một
sidecar container xử lý log, hoặc để khắc phục sự cố (troubleshoot) các container image không
kèm theo các tiện ích gỡ lỗi như shell.

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao tiếp
với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node không đóng
vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster bằng
[minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong các sân
chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

## Cấu hình một Pod (Configure a Pod)

Tính năng chia sẻ process namespace được bật bằng field `shareProcessNamespace` trong `.spec`
của một Pod. Ví dụ:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  shareProcessNamespace: true
  containers:
  - name: nginx
    image: nginx
  - name: shell
    image: busybox:1.28
    command: ["sleep", "3600"]
    securityContext:
      capabilities:
        add:
        - SYS_PTRACE
    stdin: true
    tty: true
```

1. Tạo pod `nginx` trên cluster của bạn:

   ```shell
   kubectl apply -f https://k8s.io/examples/pods/share-process-namespace.yaml
   ```

1. Attach vào container `shell` và chạy `ps`:

   ```shell
   kubectl exec -it nginx -c shell -- /bin/sh
   ```

   Nếu bạn không thấy dấu nhắc lệnh, hãy thử nhấn Enter. Bên trong shell của container:

   ```shell
   # chạy lệnh này bên trong container "shell"
   ps ax
   ```

   Kết quả xuất ra tương tự như sau:

   ```none
   PID   USER     TIME  COMMAND
       1 root      0:00 /pause
       8 root      0:00 nginx: master process nginx -g daemon off;
      14 101       0:00 nginx: worker process
      15 root      0:00 sh
      21 root      0:00 ps ax
   ```

Bạn có thể gửi tín hiệu (signal) tới các process trong container khác. Ví dụ, gửi `SIGHUP` tới
`nginx` để khởi động lại worker process. Việc này yêu cầu capability `SYS_PTRACE`.

```shell
# chạy lệnh này bên trong container "shell"
kill -HUP 8   # đổi "8" cho khớp với PID của process nginx chính, nếu cần
ps ax
```

Kết quả xuất ra tương tự như sau:

```none
PID   USER     TIME  COMMAND
    1 root      0:00 /pause
    8 root      0:00 nginx: master process nginx -g daemon off;
   15 root      0:00 sh
   22 101       0:00 nginx: worker process
   23 root      0:00 ps ax
```

Thậm chí bạn còn có thể truy cập hệ thống tập tin (file system) của một container khác thông
qua liên kết `/proc/$pid/root`.

```shell
# chạy lệnh này bên trong container "shell"
# đổi "8" thành PID của process Nginx, nếu cần
head /proc/8/root/etc/nginx/nginx.conf
```

Kết quả xuất ra tương tự như sau:

```none
user  nginx;
worker_processes  1;

error_log  /var/log/nginx/error.log warn;
pid        /var/run/nginx.pid;


events {
    worker_connections  1024;
```

## Hiểu về chia sẻ process namespace (Understanding process namespace sharing)

Các Pod vốn đã chia sẻ nhiều tài nguyên với nhau, nên việc chúng cũng chia sẻ một process
namespace là điều hợp lý. Tuy nhiên, một số container có thể kỳ vọng được cô lập khỏi các
container khác, vì vậy điều quan trọng là phải hiểu những khác biệt sau:

1. **Process của container không còn mang PID 1 nữa.** Một số container từ chối khởi động nếu
   không có PID 1 (ví dụ: các container dùng `systemd`) hoặc chạy các lệnh như `kill -HUP 1`
   để gửi tín hiệu tới process của container. Trong các pod có process namespace được chia sẻ,
   `kill -HUP 1` sẽ gửi tín hiệu tới pod sandbox (`/pause` trong ví dụ ở trên).

1. **Các process hiển thị với các container khác trong pod.** Điều này bao gồm toàn bộ thông
   tin nhìn thấy được trong `/proc`, chẳng hạn các mật khẩu được truyền dưới dạng đối số
   (argument) hoặc biến môi trường. Những thông tin này chỉ được bảo vệ bởi các quyền
   (permission) Unix thông thường.

1. **Hệ thống tập tin của container hiển thị với các container khác trong pod thông qua liên
   kết `/proc/$pid/root`.** Điều này giúp việc gỡ lỗi dễ dàng hơn, nhưng cũng có nghĩa là các
   bí mật (secret) trên hệ thống tập tin chỉ được bảo vệ bởi các quyền của hệ thống tập tin.

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 3:

1. Field nào bật tính năng này, nó nằm ở đâu trong manifest, và hệ quả trực tiếp quan sát được là
   gì?
2. **Câu bẫy.** `ps ax` trong container `shell` cho thấy PID 1 là `/pause`, còn nginx mang PID 8.
   Một container quen chạy `kill -HUP 1` để tự nạp lại cấu hình sẽ làm gì trong Pod này? Và loại
   container nào bài cảnh báo là có thể **từ chối khởi động**?
3. Bạn dựng Pod hai container này trên `lab-k8s-worker2` và muốn từ container `shell` gửi
   `SIGHUP` cho nginx, rồi đọc `/etc/nginx/nginx.conf` của container nginx. Manifest phải có thêm
   gì, và hai thao tác đó gõ như thế nào?
4. Mục *Hiểu về chia sẻ process namespace* liệt kê ba khác biệt. Hai khác biệt sau cùng dẫn tới
   một cảnh báo bảo mật chung — cảnh báo đó là gì?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **`shareProcessNamespace: true`**, đặt trong **`.spec` của Pod** (ngang cấp với `containers`,
   không nằm trong từng container). Hệ quả: **các process trong một container hiển thị với tất cả
   các container khác trong cùng Pod** — chạy `ps ax` trong container `shell` thấy cả
   `nginx: master process` lẫn `nginx: worker process`.
2. Nó sẽ **gửi tín hiệu tới pod sandbox** (`/pause`), **không** tới nginx — vì trong Pod chia sẻ
   process namespace, **process của container không còn mang PID 1 nữa**. Đây là chỗ trực giác
   sai: `kill -HUP 1` là thói quen đúng trong container thường, nhưng ở đây PID 1 đã đổi chủ.
   Muốn trúng nginx thì phải nhắm đúng PID thật của nó (`kill -HUP 8` trong ví dụ). Loại container
   bài cảnh báo có thể **từ chối khởi động** là container **cần có PID 1**, ví dụ container dùng
   `systemd`.
3. Manifest phải cấp cho container `shell` capability **`SYS_PTRACE`**, qua
   `securityContext.capabilities.add`. Sau đó, trong container `shell`: `kill -HUP 8` (đổi `8`
   cho khớp PID thật của nginx) để nginx khởi động lại worker process — `ps ax` sẽ cho thấy worker
   mang PID mới. Còn đọc file của container kia thì đi qua liên kết `/proc/$pid/root`:
   `head /proc/8/root/etc/nginx/nginx.conf`.
4. Rằng **ranh giới cô lập giữa các container trong Pod biến mất**, và những gì trước đây được
   ngăn bằng container giờ **chỉ còn được bảo vệ bằng quyền Unix và quyền filesystem thông
   thường**. Cụ thể: toàn bộ thông tin nhìn thấy trong `/proc` — kể cả **mật khẩu truyền dưới dạng
   argument hoặc biến môi trường** — lộ ra với container khác; và **secret nằm trên hệ thống tập
   tin** cũng đọc được qua `/proc/$pid/root`.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
