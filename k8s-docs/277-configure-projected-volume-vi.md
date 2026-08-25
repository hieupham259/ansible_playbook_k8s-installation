# Cấu hình Pod sử dụng projected Volume cho lưu trữ (Configure a Pod to Use a Projected Volume for Storage)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/configure-pod-container/configure-projected-volume-storage/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:**
[Phần I — Nền tảng Kubernetes](00-ALO-TRINH-ADMIN.md#phần-i--nền-tảng-kubernetes)
→ [Giai đoạn 6 — Lưu trữ](00-ALO-TRINH-ADMIN.md#giai-đoạn-6--lưu-trữ), dòng **Thực hành**, bài 1/4 ·
Kiểm chứng ở [Lab 6a — PV, PVC và StorageClass](labs/LAB-6A-PV-PVC-VA-STORAGECLASS.md), phần B7.1–B7.2.

Đây là bài thực hành ngắn nhất của nhóm và là bản chạy tay của bài
[93](93-projected-volumes-vi.md) bạn vừa đọc. Nó không thêm khái niệm mới, chỉ cho bạn thấy một
Pod thật có `projected` trông ra sao.

**Phải hiểu ở lần đọc này:**

- Trong manifest ở mục *Cấu hình một projected volume cho Pod*, Pod chỉ có **một** mục `volumes`
  và **một** `volumeMounts` trỏ tới `/projected-volume`, dù đứng sau là hai Secret khác nhau. Đó
  là toàn bộ ý nghĩa của `projected`: gộp nhiều **nguồn** thành một volume, một điểm mount.
- Bên trong `projected.sources`, mỗi nguồn Secret khai bằng trường `name` — manifest của bài viết
  `- secret: name: user`.
- Danh sách bốn thứ chiếu được ở đầu bài (`secret`, `configMap`, `downwardAPI`,
  `serviceAccountToken`) đi kèm ghi chú ngay sau đó rằng `serviceAccountToken` **không phải** một
  loại volume. Chiếu được không đồng nghĩa với đứng riêng được.
- Trình tự của bài: `echo -n ... > ./username.txt` rồi
  `kubectl create secret generic user --from-file=./username.txt`. Hai file cục bộ chỉ là
  **nguyên liệu**; sau lệnh đó nội dung nằm trong object Secret, và Pod chiếu Secret chứ không
  chiếu file.
- `readOnly: true` đặt trên `volumeMounts`, và mục *Dọn dẹp* xóa cả Pod lẫn hai Secret — xóa Pod
  không xóa Secret.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Nguồn `serviceAccountToken` và ghi chú "không phải là một loại volume" | bài chỉ nêu tên, không có ví dụ; hiểu nó cần biết ServiceAccount và bên nhận token | [giai đoạn 9](00-ALO-TRINH-ADMIN.md#giai-đoạn-9--bảo-mật-và-multi-tenancy); [Lab 6a](labs/LAB-6A-PV-PVC-VA-STORAGECLASS.md) B7.3 chỉ cho bạn thấy Pod nào cũng đang mang sẵn một projection như vậy |
| Nguồn `downwardAPI` — có trong danh sách nhưng bài không dùng | ví dụ của bài chỉ gộp hai Secret | bài [93](93-projected-volumes-vi.md) vừa đọc ở đầu [giai đoạn 6](00-ALO-TRINH-ADMIN.md#giai-đoạn-6--lưu-trữ) |
| Mục *Trước khi bạn bắt đầu* gợi ý minikube và các playground | lộ trình chạy trên cluster ba VM riêng, không dùng cluster dùng chung | [Lab 00 — Môi trường](labs/LAB-00-MOI-TRUONG-1.35.7.md), đã dựng từ giai đoạn 1 |

---

Trang này hướng dẫn cách sử dụng Volume kiểu
[`projected`](91-volumes-vi.md#projected) để mount nhiều
nguồn volume có sẵn vào cùng một thư mục. Hiện tại, các volume `secret`, `configMap`,
`downwardAPI` và `serviceAccountToken` có thể được chiếu (project).

> **Ghi chú:**
>
> `serviceAccountToken` không phải là một loại volume.

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

## Cấu hình một projected volume cho Pod (Configure a projected volume for a pod)

Trong bài thực hành này, bạn tạo các Secret chứa tên người dùng (username) và mật khẩu
(password) từ các file cục bộ. Sau đó bạn tạo một Pod chạy một container, sử dụng Volume kiểu
[`projected`](91-volumes-vi.md#projected) để mount các
Secret vào cùng một thư mục dùng chung.

Đây là file cấu hình của Pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-projected-volume
spec:
  containers:
  - name: test-projected-volume
    image: busybox:1.28
    args:
    - sleep
    - "86400"
    volumeMounts:
    - name: all-in-one
      mountPath: "/projected-volume"
      readOnly: true
  volumes:
  - name: all-in-one
    projected:
      sources:
      - secret:
          name: user
      - secret:
          name: pass
```

1. Tạo các Secret:

    ```shell
    # Tạo các file chứa username và password:
    echo -n "admin" > ./username.txt
    echo -n "1f2d1e2e67df" > ./password.txt

    # Đóng gói các file này thành các secret:
    kubectl create secret generic user --from-file=./username.txt
    kubectl create secret generic pass --from-file=./password.txt
    ```

1. Tạo Pod:

    ```shell
    kubectl apply -f https://k8s.io/examples/pods/storage/projected.yaml
    ```

1. Xác nhận rằng container của Pod đang chạy, sau đó theo dõi (watch) các thay đổi của Pod:

    ```shell
    kubectl get --watch pod test-projected-volume
    ```

    Kết quả trông như sau:

    ```
    NAME                    READY     STATUS    RESTARTS   AGE
    test-projected-volume   1/1       Running   0          14s
    ```

1. Trong một terminal khác, mở một shell tới container đang chạy:

    ```shell
    kubectl exec -it test-projected-volume -- /bin/sh
    ```

1. Trong shell của bạn, xác nhận rằng thư mục `projected-volume` chứa các nguồn đã được chiếu:

    ```shell
    ls /projected-volume/
    ```

## Dọn dẹp (Clean up)

Xóa Pod và các Secret:

```shell
kubectl delete pod test-projected-volume
kubectl delete secret user pass
```

## Tiếp theo (What's next)

* Tìm hiểu thêm về volume
  [`projected`](91-volumes-vi.md#projected).
* Đọc tài liệu thiết kế
  [all-in-one volume](https://git.k8s.io/design-proposals-archive/node/all-in-one-volume.md).

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 6:

1. Pod `test-projected-volume` lấy dữ liệu từ hai Secret khác nhau. Trong Pod spec có bao nhiêu
   mục `volumes` và bao nhiêu `volumeMounts`? Con số đó nói lên tính chất gì của kiểu `projected`?
2. **Câu bẫy.** Bên trong `projected.sources`, một nguồn Secret được khai bằng trường tên gì?
3. Bài xếp `serviceAccountToken` vào nhóm chiếu được, rồi ghi chú ngay rằng nó "không phải là một
   loại volume". Hai câu đó có mâu thuẫn nhau không?
4. Bạn tạo hai Secret trên `lab-k8s-master`, còn Pod có thể được lập lịch lên `lab-k8s-worker1`
   hoặc `lab-k8s-worker2`. Bài không hề bảo bạn chép `username.txt` và `password.txt` sang worker.
   Vì sao Pod vẫn chạy được?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Một `volumes` và một `volumeMounts`.** `projected` gộp nhiều **nguồn** thành **một** volume,
   nên container chỉ thấy đúng một thư mục `/projected-volume`. Đó là điều bài muốn minh họa:
   "mount nhiều nguồn volume có sẵn vào cùng một thư mục".
2. **`name`.** Manifest của bài viết `- secret: name: user`, không phải `secretName`. Đây là chỗ
   dễ gõ nhầm; [Lab 6a](labs/LAB-6A-PV-PVC-VA-STORAGECLASS.md) B7.4 cho bạn thấy kết quả thật khi
   viết nhầm.
3. **Không mâu thuẫn.** Bài phân biệt hai chuyện khác nhau: thứ *chiếu được* vào một `projected`
   volume, và thứ *là một loại volume*. `serviceAccountToken` thuộc nhóm đầu chứ không thuộc nhóm
   sau — ghi chú đó có để bạn khỏi đi tìm một volume tên `serviceAccountToken` đứng riêng.
4. **Vì hai file chỉ là nguyên liệu để tạo Secret.** `--from-file` đọc file ngay lúc chạy
   `kubectl create secret`; từ đó nội dung nằm trong object Secret. Manifest Pod trỏ tới
   `- secret: name: user`, tức trỏ tới **object**, không trỏ tới đường dẫn trên đĩa. Node nào chạy
   Pod cũng không cần hai file `.txt` đó.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
