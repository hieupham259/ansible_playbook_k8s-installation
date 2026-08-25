# Cấu hình Pod sử dụng Volume để lưu trữ (Configure a Pod to Use a Volume for Storage)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/configure-pod-container/configure-volume-storage/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:**
[Phần I — Nền tảng Kubernetes](00-ALO-TRINH-ADMIN.md#phần-i--nền-tảng-kubernetes)
→ [Giai đoạn 6 — Lưu trữ](00-ALO-TRINH-ADMIN.md#giai-đoạn-6--lưu-trữ), dòng **Thực hành**, bài 2/4 ·
Kiểm chứng ở [Lab 6a — PV, PVC và StorageClass](labs/LAB-6A-PV-PVC-VA-STORAGECLASS.md), phần B1.

Bài này chỉ dùng một kiểu volume duy nhất — `emptyDir` — và toàn bộ giá trị của nó nằm ở **một
phép thử**: giết tiến trình trong container rồi xem file còn hay mất. Đọc để nắm đúng ranh giới
mà phép thử đó chứng minh được, vì phần còn lại của giai đoạn 6 xây trên đúng ranh giới ấy.

**Phải hiểu ở lần đọc này:**

- Vấn đề bài đặt ra ngay ở đoạn mở đầu: filesystem của Container **chỉ tồn tại chừng nào Container
  còn tồn tại**; Container kết thúc rồi khởi động lại thì thay đổi trên filesystem mất. Volume là
  chỗ ghi tách khỏi Container.
- Câu định nghĩa phạm vi ở mục *Cấu hình một volume cho Pod*: `emptyDir` "tồn tại trong suốt vòng
  đời của **Pod**, ngay cả khi Container kết thúc và khởi động lại". Từ khóa là **Pod**, không phải
  Container và cũng không phải node.
- Cách phép thử chứng minh điều đó: ghi `test-file` vào `/data/redis`, `kill` tiến trình
  `redis-server`, rồi đọc output watch — Pod đổi qua `Completed` và trở lại `Running` với cột
  `RESTARTS` thành `1`, trong khi tên Pod không đổi; `exec` lại vào container thì `test-file` vẫn
  còn.
- Vì sao container quay lại: bài chỉ rõ Pod Redis có `restartPolicy` là `Always`, nên container
  được dựng lại **trong cùng Pod** thay vì Pod bị thay thế.
- Ranh giới mà bài hứa dừng ở vòng đời Pod: bước cuối là `kubectl delete pod redis`, và mục *Tiếp
  theo* chỉ sang lưu trữ gắn qua mạng cho "dữ liệu quan trọng".

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Bước `apt-get update` và `apt-get install procps` để chạy `ps aux` | chỉ là cách tìm PID bên trong image `redis`, không phải cơ chế Kubernetes | [Lab 6a](labs/LAB-6A-PV-PVC-VA-STORAGECLASS.md) B1.2 làm container chết mà không phải cài thêm gói nào |
| Chi tiết các giá trị của `restartPolicy` và link API reference của nó | ở đây chỉ cần biết `Always` khiến container quay lại | bài [47](47-pod-lifecycle-vi.md) đã đọc ở [3a](00-ALO-TRINH-ADMIN.md#3a-pod-và-vòng-đời) |
| Gợi ý lưu trữ gắn qua mạng (PD trên GCE, EBS trên EC2) ở mục *Tiếp theo* | cluster lab là ba VM bare-metal, không có provisioner của cloud nào | phần còn lại của [giai đoạn 6](00-ALO-TRINH-ADMIN.md#giai-đoạn-6--lưu-trữ) — PV, PVC và StorageClass; [Lab 6a](labs/LAB-6A-PV-PVC-VA-STORAGECLASS.md) B2–B5 |

---

Trang này hướng dẫn cách cấu hình một Pod sử dụng Volume để lưu trữ.

Hệ thống file (filesystem) của một Container chỉ tồn tại chừng nào Container còn tồn tại. Vì vậy,
khi một Container kết thúc và khởi động lại, các thay đổi trên filesystem sẽ bị mất. Để có nơi
lưu trữ nhất quán hơn và độc lập với Container, bạn có thể dùng một
[Volume](./91-volumes-vi.md). Điều này đặc biệt quan trọng đối với các ứng dụng có trạng thái
(stateful), chẳng hạn như các kho lưu trữ key-value (ví dụ Redis) và các cơ sở dữ liệu.

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao tiếp
với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node không đóng
vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster bằng
[minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong các sân
chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

Để kiểm tra phiên bản, hãy nhập `kubectl version`.

## Cấu hình một volume cho Pod (Configure a volume for a Pod)

Trong bài thực hành này, bạn tạo một Pod chạy một Container. Pod này có một Volume kiểu
[emptyDir](./91-volumes-vi.md#emptydir) tồn tại trong suốt vòng đời của Pod, ngay cả khi
Container kết thúc và khởi động lại. Đây là file cấu hình cho Pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: redis
spec:
  containers:
  - name: redis
    image: redis
    volumeMounts:
    - name: redis-storage
      mountPath: /data/redis
  volumes:
  - name: redis-storage
    emptyDir: {}
```

1. Tạo Pod:

   ```shell
   kubectl apply -f https://k8s.io/examples/pods/storage/redis.yaml
   ```

1. Xác minh rằng Container của Pod đang chạy, sau đó theo dõi (watch) các thay đổi của Pod:

   ```shell
   kubectl get pod redis --watch
   ```

   Output trông như sau:

   ```console
   NAME      READY     STATUS    RESTARTS   AGE
   redis     1/1       Running   0          13s
   ```

1. Trong một terminal khác, mở một shell vào Container đang chạy:

   ```shell
   kubectl exec -it redis -- /bin/bash
   ```

1. Trong shell của bạn, đi tới `/data/redis`, sau đó tạo một file:

   ```shell
   root@redis:/data# cd /data/redis/
   root@redis:/data/redis# echo Hello > test-file
   ```

1. Trong shell của bạn, liệt kê các tiến trình (process) đang chạy:

   ```shell
   root@redis:/data/redis# apt-get update
   root@redis:/data/redis# apt-get install procps
   root@redis:/data/redis# ps aux
   ```

   Output tương tự như sau:

   ```console
   USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
   redis        1  0.1  0.1  33308  3828 ?        Ssl  00:46   0:00 redis-server *:6379
   root        12  0.0  0.0  20228  3020 ?        Ss   00:47   0:00 /bin/bash
   root        15  0.0  0.0  17500  2072 ?        R+   00:48   0:00 ps aux
   ```

1. Trong shell của bạn, kill tiến trình Redis:

   ```shell
   root@redis:/data/redis# kill <pid>
   ```

   trong đó `<pid>` là ID tiến trình (PID) của Redis.

1. Trong terminal ban đầu của bạn, theo dõi các thay đổi của Pod Redis. Cuối cùng, bạn sẽ thấy
   kết quả tương tự như sau:

   ```console
   NAME      READY     STATUS     RESTARTS   AGE
   redis     1/1       Running    0          13s
   redis     0/1       Completed  0         6m
   redis     1/1       Running    1         6m
   ```

Đến thời điểm này, Container đã kết thúc và khởi động lại. Đó là vì Pod Redis có
[restartPolicy](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.36/#podspec-v1-core)
là `Always`.

1. Mở một shell vào Container đã được khởi động lại:

   ```shell
   kubectl exec -it redis -- /bin/bash
   ```

1. Trong shell của bạn, đi tới `/data/redis` và xác minh rằng `test-file` vẫn còn ở đó.

   ```shell
   root@redis:/data/redis# cd /data/redis/
   root@redis:/data/redis# ls
   test-file
   ```

1. Xóa Pod mà bạn đã tạo cho bài thực hành này:

   ```shell
   kubectl delete pod redis
   ```

## Tiếp theo (What's next)

- Xem [Volume](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.36/#volume-v1-core).

- Xem [Pod](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.36/#pod-v1-core).

- Ngoài kho lưu trữ trên đĩa cục bộ do `emptyDir` cung cấp, Kubernetes còn hỗ trợ nhiều giải pháp
  lưu trữ gắn qua mạng (network-attached storage) khác nhau, bao gồm PD trên GCE và EBS trên EC2;
  các giải pháp này được ưu tiên cho dữ liệu quan trọng và sẽ xử lý các chi tiết như mount và
  unmount thiết bị trên các node. Xem [Volumes](./91-volumes-vi.md) để biết thêm chi tiết.

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 6:

1. Phép thử của bài — kill tiến trình Redis rồi thấy `test-file` vẫn còn — chứng minh `emptyDir`
   sống lâu hơn **cái gì**, và **không** chứng minh nó sống lâu hơn cái gì?
2. **Câu bẫy.** Sau khi kill, cột `RESTARTS` nhảy từ `0` lên `1` nhưng tên Pod vẫn là `redis` và
   `AGE` vẫn tiếp tục đếm. Đó là Pod cũ hay Pod mới? Trường nào trong manifest quyết định chuyện đó?
3. Trên cluster lab, Pod `redis` có thể rơi vào `lab-k8s-worker1` hoặc `lab-k8s-worker2`. Bạn chạy
   `kubectl delete pod redis`, apply lại đúng manifest đó, và lần này Pod lên node còn lại.
   `test-file` có ở đó không? Câu trả lời có phụ thuộc vào việc nó lên node nào không?
4. Nếu Redis ghi thẳng vào `/data/redis` mà Pod **không** khai volume nào, phép thử ở bước cuối
   cho kết quả gì? Câu nào ở đầu bài đã nói trước điều đó?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Nó chứng minh `emptyDir` sống lâu hơn **container**. Nó **không** chứng minh gì về việc sống
   lâu hơn **Pod** — bài chỉ hứa "tồn tại trong suốt vòng đời của Pod", và toàn bộ phép thử diễn
   ra bên trong một Pod duy nhất.
2. **Pod cũ.** Container bị dựng lại bên trong chính Pod đó, nên tên Pod và `AGE` giữ nguyên còn
   `RESTARTS` tăng lên; bài chỉ đích danh nguyên nhân là Pod Redis có `restartPolicy` là `Always`.
   Nếu là Pod mới thì volume `emptyDir` đã được tạo lại rỗng và `test-file` không còn.
3. **Không có `test-file`, và câu trả lời không phụ thuộc vào node.** `kubectl delete pod` kết
   thúc vòng đời của Pod cũ, mà bảo đảm của `emptyDir` chỉ kéo dài đúng bằng vòng đời Pod. Pod
   mới nhận một `emptyDir` mới rỗng, kể cả khi nó tình cờ lên lại đúng node cũ.
4. **`test-file` mất.** Đoạn mở đầu đã nói trước: "Hệ thống file của một Container chỉ tồn tại
   chừng nào Container còn tồn tại. Vì vậy, khi một Container kết thúc và khởi động lại, các thay
   đổi trên filesystem sẽ bị mất." Volume tồn tại chính là để phá vỡ ràng buộc đó.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
