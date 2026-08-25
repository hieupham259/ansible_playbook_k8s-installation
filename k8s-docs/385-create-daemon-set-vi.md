# Xây dựng một DaemonSet cơ bản (Building a Basic DaemonSet)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/manage-daemon/create-daemon-set/>
>
> Trang này minh họa cách xây dựng một DaemonSet cơ bản để chạy một Pod trên mọi node
> trong cluster Kubernetes.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:**
[Phần II — Vận hành cluster](00-ALO-TRINH-ADMIN.md#phần-ii--vận-hành-cluster)
→ [Giai đoạn 29 — DaemonSet, Job nâng cao và thiết bị chuyên dụng](00-ALO-TRINH-ADMIN.md#giai-đoạn-29--daemonset-job-nâng-cao-và-thiết-bị-chuyên-dụng),
bài 2/8 · Kiểm chứng trực tiếp trên cluster lab: apply `example-daemonset` từ `lab-k8s-master`, đếm
Pod bằng `kubectl get pods -o wide`, rồi xóa theo đúng lệnh ở mục *Dọn dẹp*.

Đây là DaemonSet bạn sẽ dùng lại suốt giai đoạn 29 — bài [388](388-update-daemon-set-vi.md) rolling
update nó, bài [387](387-rollback-daemon-set-vi.md) rollback nó, bài
[386](386-pods-some-nodes-vi.md) giới hạn nó xuống một nhóm node. Bạn đã dựng một DaemonSet đơn giản
hơn ở [Lab 4b](labs/LAB-4B-STATEFULSET-DAEMONSET-VA-JOB.md) phần B4; bài này thêm init container và
volume `hostPath`. Chép manifest ra file local thay vì `apply` thẳng URL — ba bài sau đều cần sửa
chính manifest đó. Lưu ý một chi tiết thực tế: container chính là `registry.k8s.io/pause`, một image
tối giản, nên bước `kubectl exec ... -- cat` có thể không chạy được; `/var/log` được mount từ host
nên bạn luôn đọc được file đó bằng cách vào thẳng node.

**Phải hiểu ở lần đọc này:**

- Hình dạng tối thiểu của một DaemonSet ở mục *Định nghĩa DaemonSet*: có `selector.matchLabels` phải
  khớp `template.metadata.labels`, và **không có** `replicas` — số Pod do số node đủ điều kiện quyết
  định, không do bạn đặt.
- Vai trò từng container: init container `log-machine-id` chạy trước rồi kết thúc; container chính
  `pause` không làm gì ngoài việc **giữ cho Pod tiếp tục chạy** — đúng như đoạn mở đầu mục *Định
  nghĩa DaemonSet* nói.
- Hai volume `hostPath` là cách Pod chạm vào máy host: `/etc/machine-id` mount `readOnly` với
  `type: File`, và `/var/log` mount cả thư mục để init container ghi kết quả xuống đĩa của node.
- Cách xác minh mà bài đưa ra: `kubectl get pods -o wide` phải cho **mỗi node một Pod** — cột `NODE`
  là bằng chứng, không phải số lượng Pod.
- Lệnh dọn dẹp ở mục *Dọn dẹp* xóa DaemonSet kèm ba cờ `--cascade=foreground`, `--ignore-not-found`
  và `--now`; nhận ra rằng xóa DaemonSet là xóa luôn Pod nó tạo ra.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Mục *Trước khi bạn bắt đầu* — minikube và ba playground | Lộ trình cấm minikube, kind và cluster dùng chung; điều kiện "ít nhất hai node không phải control plane host" thì cluster lab đã thỏa | Bỏ hẳn — dùng ba VM của [Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md) |
| Câu "mở rộng chúng cho các tình huống sử dụng nâng cao hơn" ở cuối mục *Dọn dẹp* | Bài không nói mở rộng thế nào | Bài [388](388-update-daemon-set-vi.md) (đổi image đang chạy) và bài [386](386-pods-some-nodes-vi.md) (thu hẹp tập node) — bài 3/8 và 5/8 |
| Link *Tạo một DaemonSet để nhận (adopt) các Pod DaemonSet đã tồn tại* ở mục *Tiếp theo* | Việc nhận Pod trần là chuyện của controller nói chung | Bài [66](66-daemonset-vi.md) đã đọc ở [giai đoạn 4](00-ALO-TRINH-ADMIN.md#giai-đoạn-4--workload-controller) |

---

Trang này minh họa cách xây dựng một DaemonSet cơ bản chạy một Pod trên mọi node trong một
cluster Kubernetes. Bài viết trình bày một tình huống sử dụng đơn giản: mount một file từ
host, ghi log nội dung của file đó bằng [init container](50-init-containers-vi.md), và tận
dụng một pause container.

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao
tiếp với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node
không đóng vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster
bằng [minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong
các sân chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

Một cluster Kubernetes có ít nhất hai node (một control plane node và một worker node) để
minh họa hành vi của DaemonSet.

## Định nghĩa DaemonSet (Define the DaemonSet)

Trong bài thực hành này, bạn tạo một DaemonSet cơ bản để đảm bảo rằng bản sao của một Pod
được lập lịch (schedule) trên mọi node. Pod này sẽ dùng một init container để đọc và ghi log
nội dung của `/etc/machine-id` từ host, còn container chính sẽ là một container `pause`, có
nhiệm vụ giữ cho Pod tiếp tục chạy.

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: example-daemonset
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: example
  template:
    metadata:
      labels:
        app.kubernetes.io/name: example
    spec:
      containers:
      - name: pause
        image: registry.k8s.io/pause
      initContainers:
      - name: log-machine-id
        image: busybox:1.37
        command: ['sh', '-c', 'cat /etc/machine-id > /var/log/machine-id.log']
        volumeMounts:
        - name: machine-id
          mountPath: /etc/machine-id
          readOnly: true
        - name: log-dir
          mountPath: /var/log
      volumes:
      - name: machine-id
        hostPath:
          path: /etc/machine-id
          type: File
      - name: log-dir
        hostPath:
          path: /var/log
```

1. Tạo một DaemonSet dựa trên manifest (YAML) ở trên:

   ```shell
   kubectl apply -f https://k8s.io/examples/application/basic-daemonset.yaml
   ```

1. Sau khi áp dụng, bạn có thể xác minh rằng DaemonSet đang chạy một Pod trên mọi node trong
   cluster:

   ```shell
   kubectl get pods -o wide
   ```

   Kết quả sẽ liệt kê mỗi node một Pod, tương tự như sau:

   ```
   NAME                                READY   STATUS    RESTARTS   AGE    IP       NODE
   example-daemonset-xxxxx             1/1     Running   0          5m     x.x.x.x  node-1
   example-daemonset-yyyyy             1/1     Running   0          5m     x.x.x.x  node-2
   ```

1. Bạn có thể kiểm tra nội dung của file `/etc/machine-id` đã được ghi log bằng cách xem thư
   mục log được mount từ host:

   ```shell
   kubectl exec <pod-name> -- cat /var/log/machine-id.log
   ```

   Trong đó `<pod-name>` là tên của một trong các Pod của bạn.

## Dọn dẹp (Cleaning up)

Để xóa DaemonSet, hãy chạy lệnh sau:

```shell
kubectl delete --cascade=foreground --ignore-not-found --now daemonsets/example-daemonset
```

Ví dụ DaemonSet đơn giản này giới thiệu những thành phần chủ chốt như init container và
volume dạng host path, và bạn có thể mở rộng chúng cho các tình huống sử dụng nâng cao hơn.
Để biết thêm chi tiết, hãy tham khảo [DaemonSet](66-daemonset-vi.md).

## Tiếp theo (What's next)

* Xem [Thực hiện rolling update trên một DaemonSet](388-update-daemon-set-vi.md)
* Xem [Tạo một DaemonSet để nhận (adopt) các Pod DaemonSet đã tồn tại](66-daemonset-vi.md)

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 29:

1. Manifest `example-daemonset` không có trường `replicas`. Vậy cái gì quyết định số Pod được tạo, và
   quan hệ giữa `spec.selector.matchLabels` với `spec.template.metadata.labels` phải thế nào?
2. **Câu bẫy.** Cluster lab có ba node, nhưng `lab-k8s-master` mang taint của control plane còn
   manifest trong bài **không có** trường `tolerations`. Sau khi apply, `kubectl get pods -o wide`
   cho mấy dòng? Điều đó có mâu thuẫn với câu "chạy một Pod trên mọi node" ở đầu bài không?
3. Init container `log-machine-id` chạy xong rồi kết thúc. Vì sao Pod vẫn cần container `pause` bên
   cạnh nó, và cột `READY` hiện `1/1` là đang đếm container nào?
4. Pod trên `lab-k8s-worker1` và Pod trên `lab-k8s-worker2` cùng chạy đúng một manifest, nhưng nội
   dung `/var/log/machine-id.log` của hai máy khác nhau. Vì sao — và file đó thực sự nằm trên đĩa của
   ai?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Số node đủ điều kiện quyết định số Pod** — DaemonSet bảo đảm mỗi node một bản sao, nên nó không
   có núm `replicas` để bạn vặn. `selector.matchLabels` **phải khớp** với
   `template.metadata.labels`; trong bài cả hai đều là `app.kubernetes.io/name: example`. Đó là cách
   DaemonSet nhận ra Pod nào là của mình.
2. **Hai dòng, không phải ba** — Pod chỉ lên `lab-k8s-worker1` và `lab-k8s-worker2`. Không mâu thuẫn:
   "mọi node" luôn có nghĩa là **mọi node đủ điều kiện**, và một node mang taint mà Pod không
   toleration thì không đủ điều kiện. Manifest trong bài không khai `tolerations` nào, nên
   `lab-k8s-master` bị loại. Chỗ dễ sai là đếm node rồi kỳ vọng đúng bằng đó Pod; muốn phủ cả control
   plane thì phải tự thêm toleration vào `template`.
3. Vì **init container kết thúc là xong việc của nó**, mà một Pod chỉ ở trạng thái `Running` khi còn
   container thường đang chạy. `pause` chính là container thường duy nhất, và bài nói rõ nhiệm vụ của
   nó là **giữ cho Pod tiếp tục chạy**. `READY 1/1` đếm **container thường**, tức chỉ mình `pause`;
   init container đã chạy xong không nằm trong con số này.
4. Vì cả hai volume đều là `hostPath`: init container đọc `/etc/machine-id` **của chính máy host** —
   mỗi máy một giá trị — rồi ghi kết quả vào `/var/log`, cũng là `hostPath`. Nên **file
   `/var/log/machine-id.log` nằm trên đĩa của node, không nằm trong container**: nó sống sót khi Pod
   bị xóa, và mỗi node giữ một bản riêng với nội dung riêng.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
