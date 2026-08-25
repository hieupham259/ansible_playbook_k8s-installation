# Định nghĩa command và argument cho container (Define a Command and Arguments for a Container)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/inject-data-application/define-command-argument-container/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 3 — Pod và cấu hình](00-ALO-TRINH-ADMIN.md#giai-đoạn-3--pod-và-cấu-hình)
→ nhóm [3b. Cấu hình ứng dụng](00-ALO-TRINH-ADMIN.md#3b-cấu-hình-ứng-dụng-configmap-secret-và-dữ-liệu-cho-pod),
bài 8/12 · Kiểm chứng ở [Lab 3b](labs/LAB-3B-CAU-HINH-UNG-DUNG.md) phần B4.1 (`command`/`args` ghi
đè image) và B4.2 (`args` lấy từ biến môi trường).

Bài ngắn và chỉ có ba mục nội dung, nhưng nó là chỗ nối: cú pháp `$(VAR)` học ở đây được dùng lại
cho mọi nguồn biến môi trường của nhóm 3b — ConfigMap ở [275](275-configure-pod-configmap-vi.md),
Secret ở [334](334-distribute-credentials-secure-vi.md).

**Phải hiểu ở lần đọc này:**

- `command` tương ứng với `ENTRYPOINT` và `args` tương ứng với `CMD` của image. Giá trị khai trong
  Pod **ghi đè** giá trị mặc định của image, và **không đổi được sau khi Pod đã được tạo** — mục
  *Định nghĩa command và argument khi bạn tạo Pod*.
- Quy tắc ghép ở cùng mục đó: khai `args` mà **không** khai `command` thì **command mặc định của
  image vẫn được dùng**, chỉ argument bị thay. Đây là chỗ quyết định khi bạn chỉ muốn đổi tham số
  mà không muốn đụng tới entrypoint.
- Đưa biến môi trường vào argument: `args: ["$(MESSAGE)"]` — mục *Dùng biến môi trường để định
  nghĩa argument*. Ghi chú của bài: **cặp ngoặc đơn `"$(VAR)"` là bắt buộc** để biến được mở rộng
  trong `command` hoặc `args`. Nhờ vậy argument lấy được từ bất kỳ cách định nghĩa biến môi trường
  nào, kể cả ConfigMap và Secret.
- Muốn dùng pipe, chuỗi lệnh hay vòng lặp thì phải **tự gọi shell**: `command: ["/bin/sh"]` cộng
  `args: ["-c", "..."]` — mục *Chạy command trong shell*. Container không tự bọc lệnh của bạn trong
  một shell.
- Cách quan sát kết quả trong ví dụ đầu bài: container chạy `printenv` rồi kết thúc, Pod dùng
  `restartPolicy: OnFailure`; `kubectl get pods` chỉ cho thấy container **đã hoàn thành**, còn nội
  dung in ra phải xem bằng `kubectl logs <tên Pod>`.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Biến `KUBERNETES_PORT` trong output ví dụ (`tcp://10.3.240.1:443`) | là biến Kubernetes tự đặt vào container, gắn với Service — chưa học ở giai đoạn 3 | [giai đoạn 5 — Mạng nền tảng](00-ALO-TRINH-ADMIN.md#giai-đoạn-5--mạng-nền-tảng) |
| Link *chạy lệnh trong container* ở mục *Tiếp theo* | là thao tác `exec` vào container đang chạy, khác hẳn việc khai `command` lúc tạo Pod | bài [304](304-get-shell-running-container-vi.md), [giai đoạn 11](00-ALO-TRINH-ADMIN.md#giai-đoạn-11--observability) |
| Link *cấu hình Pod và container* ở mục *Tiếp theo* | là trang mục gốc của cả nhánh `/docs/tasks/`, không phải bài học | trang [367](367-tasks-index-vi.md), giới thiệu ở [Phần II — Vận hành cluster](00-ALO-TRINH-ADMIN.md#phần-ii--vận-hành-cluster) |

---

Trang này hướng dẫn cách định nghĩa command (lệnh) và argument (đối số) khi bạn chạy một container
trong một Pod.

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình
để giao tiếp với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít
nhất hai node không đóng vai trò control plane host. Nếu bạn chưa có cluster, bạn có
thể tạo một cluster bằng [minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/)
hoặc dùng một trong các sân chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

Để kiểm tra phiên bản, hãy chạy `kubectl version`.

## Định nghĩa command và argument khi bạn tạo Pod (Define a command and arguments when you create a Pod)

Khi tạo một Pod, bạn có thể định nghĩa command và argument cho các container
chạy trong Pod đó. Để định nghĩa một command, hãy đưa trường `command`
vào file cấu hình. Để định nghĩa argument cho command, hãy đưa
trường `args` vào file cấu hình. Command và argument mà bạn
định nghĩa không thể thay đổi sau khi Pod đã được tạo.

Command và argument mà bạn định nghĩa trong file cấu hình
sẽ ghi đè command và argument mặc định do container image cung cấp.
Nếu bạn định nghĩa args nhưng không định nghĩa command, command mặc định sẽ được dùng
cùng với các argument mới của bạn.

> **Ghi chú:** Trường `command` tương ứng với `ENTRYPOINT`, và trường `args` tương ứng với `CMD` trong một số container runtime.

Trong bài thực hành này, bạn tạo một Pod chạy một container. File cấu hình
của Pod định nghĩa một command và hai argument:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: command-demo
  labels:
    purpose: demonstrate-command
spec:
  containers:
  - name: command-demo-container
    image: debian
    command: ["printenv"]
    args: ["HOSTNAME", "KUBERNETES_PORT"]
  restartPolicy: OnFailure
```

1. Tạo một Pod dựa trên file cấu hình YAML:

   ```shell
   kubectl apply -f https://k8s.io/examples/pods/commands.yaml
   ```

1. Liệt kê các Pod đang chạy:

   ```shell
   kubectl get pods
   ```

   Kết quả xuất ra cho thấy container đã chạy trong Pod command-demo
   đã hoàn thành.

1. Để xem kết quả của command đã chạy trong container, hãy xem log
của Pod:

   ```shell
   kubectl logs command-demo
   ```

   Kết quả xuất ra hiển thị giá trị của các biến môi trường HOSTNAME và
   KUBERNETES_PORT:

   ```
   command-demo
   tcp://10.3.240.1:443
   ```

## Dùng biến môi trường để định nghĩa argument (Use environment variables to define arguments)

Trong ví dụ trước, bạn đã định nghĩa các argument một cách trực tiếp bằng cách
cung cấp các chuỗi ký tự. Thay vì cung cấp chuỗi trực tiếp,
bạn có thể định nghĩa argument bằng cách dùng các biến môi trường:

```yaml
env:
- name: MESSAGE
  value: "hello world"
command: ["/bin/echo"]
args: ["$(MESSAGE)"]
```

Điều này có nghĩa là bạn có thể định nghĩa một argument cho Pod bằng bất kỳ
kỹ thuật nào dùng để định nghĩa biến môi trường, bao gồm
[ConfigMap](275-configure-pod-configmap-vi.md)
và
[Secret](109-secret-vi.md).

> **Ghi chú:** Biến môi trường xuất hiện trong cặp ngoặc đơn, `"$(VAR)"`. Cách viết này
> là bắt buộc để biến được mở rộng (expand) trong trường `command` hoặc `args`.

## Chạy command trong shell (Run a command in a shell)

Trong một số trường hợp, bạn cần command của mình chạy trong một shell. Ví dụ,
command của bạn có thể gồm nhiều lệnh nối với nhau qua pipe, hoặc nó có thể là một
shell script. Để chạy command trong một shell, hãy bọc nó lại như sau:

```shell
command: ["/bin/sh"]
args: ["-c", "while true; do echo hello; sleep 10;done"]
```

## Tiếp theo (What's next)

* Tìm hiểu thêm về [cấu hình Pod và container](367-tasks-index-vi.md).
* Tìm hiểu thêm về [chạy lệnh trong container](304-get-shell-running-container-vi.md).
* Xem [Container](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.36/#container-v1-core).

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 3:

1. `command` và `args` trong Pod tương ứng với thứ gì trong container image, và chúng làm gì với
   giá trị mặc định của image?
2. **Câu bẫy.** Bạn chỉ khai `args` và bỏ trống `command`. Container chạy cái gì — không chạy gì
   cả, chạy `args` như một lệnh, hay chạy thứ khác?
3. Bạn viết `args: ["$MESSAGE"]` thay vì `args: ["$(MESSAGE)"]`. Chuyện gì xảy ra và vì sao?
4. Trên `lab-k8s-worker2`, bạn cần một container in ra `hello` mỗi 10 giây bằng một vòng lặp
   `while`. Viết thẳng vòng lặp đó vào `command` có chạy không, và bài bảo làm thế nào?
5. Pod ví dụ đầu bài chạy `printenv` rồi kết thúc. Bạn xem kết quả của nó ở đâu, và vì sao
   `kubectl get pods` không đủ?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **`command` tương ứng với `ENTRYPOINT`, `args` tương ứng với `CMD`.** Giá trị bạn khai trong file
   cấu hình **ghi đè** command và argument mặc định do container image cung cấp. Thêm một ràng buộc
   dễ quên: **không đổi được sau khi Pod đã được tạo** — muốn khác thì phải tạo Pod mới.
2. **Chạy command mặc định của image, kèm argument mới của bạn.** Bài viết đúng một câu cho trường
   hợp này: "Nếu bạn định nghĩa args nhưng không định nghĩa command, command mặc định sẽ được dùng
   cùng với các argument mới của bạn". Trực giác "khai `args` thì `args` là thứ được chạy" nhầm vì
   `args` không phải là lệnh — nó là phần `CMD`, tức đối số truyền cho entrypoint.
3. Biến **không được mở rộng**: chuỗi `$MESSAGE` được truyền nguyên văn làm argument. Ghi chú của
   bài nói rõ **cặp ngoặc đơn `"$(VAR)"` là bắt buộc** để biến được mở rộng trong trường `command`
   hoặc `args`. Đây là cú pháp thay thế của Kubernetes, không phải của shell — mà ở đây cũng không
   có shell nào chạy để diễn giải `$MESSAGE`.
4. **Không chạy được** nếu viết thẳng, vì `while ... ; do ... ; done` là cú pháp của shell chứ không
   phải một chương trình. Bài đưa đúng cách bọc lại: **`command: ["/bin/sh"]` và `args: ["-c",
   "while true; do echo hello; sleep 10;done"]`** — tức tự gọi shell rồi đưa cả chuỗi lệnh vào `-c`.
   Cùng lý do đó áp dụng cho pipe và cho script nhiều lệnh.
5. Xem bằng **`kubectl logs command-demo`** — output của container nằm trong log của Pod.
   `kubectl get pods` không đủ vì nó chỉ cho biết **container đã chạy xong**, không cho biết nó in
   ra gì; container chạy `printenv` rồi kết thúc ngay nên không có gì để nhìn ở trạng thái.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
