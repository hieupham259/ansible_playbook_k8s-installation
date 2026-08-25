# Tạo static Pod (Create static Pods)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/configure-pod-container/static-pod/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** Giai đoạn 3 → nhóm [3c](00-ALO-TRINH-ADMIN.md#3c-tài-nguyên-qos-và-gián-đoạn), bài 9/9 ·
Kiểm chứng ở [Lab 3c](labs/LAB-3C-TAI-NGUYEN-QOS-VA-GIAN-DOAN.md) phần B8: B8.1 kiểm kê static Pod
của control plane, B8.2 xóa mirror Pod bằng `kubectl`, B8.3 và B8.4 dựng rồi xóa một static Pod
thử nghiệm trên `lab-k8s-worker2`.

Đây là **bài cuối của nhóm 3c** và là vế thực hành của bài [58](58-static-pods-vi.md). Bài viết cho
CRI-O trên Fedora, còn cluster lab chạy containerd trên Ubuntu theo
[Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md) — đọc lấy **cơ chế**, đừng chép nguyên lệnh. Cơ chế này
không phải kiến thức bên lề: bốn thành phần control plane trên `lab-k8s-master` chạy đúng theo cách
bài mô tả, nên nhầm lẫn ở đây là nhầm lẫn trên chính cluster của bạn.

**Phải hiểu ở lần đọc này:**

- Hai nguồn manifest ở mục *Tạo một static Pod*: một thư mục trên hệ thống file khai bằng
  `staticPodPath`, và một URL khai bằng `staticPodURL`. Cả hai đều được kubelet **quét lại theo
  lịch định kỳ**; mục *Thêm và xóa static Pod một cách động* cho thấy hệ quả trực tiếp — file xuất
  hiện thì Pod được tạo, file biến mất thì Pod bị xóa.
- Khối *Thận trọng*: kubelet xử lý **tất cả các file không bắt đầu bằng dấu chấm** trong thư mục
  đó, **không** lọc theo phần mở rộng. Chạy `cp kube-apiserver.yaml kube-apiserver.yaml.backup`
  ngay tại chỗ tạo ra hai file cùng định nghĩa một Pod trùng tên; hành vi thu được là **không xác
  định** và spec lỗi thời của bản backup có thể âm thầm có hiệu lực. Backup phải lưu **bên ngoài**
  thư mục static Pod.
- Mục *Quan sát hành vi của static Pod*: kubelet tự tạo một **mirror Pod** trên API server để bạn
  thấy Pod đó bằng `kubectl get pods` — manifest tên `static-web` chạy trên node `my-node1` hiện ra
  thành `static-web-my-node1`. Label của static Pod được **lan truyền** sang mirror Pod nên selector
  vẫn dùng được bình thường. Kubelet phải có quyền tạo mirror Pod, nếu không API server từ chối.
- Cũng ở mục đó: `kubectl delete pod <mirror-pod>` in ra `deleted` nhưng kubelet **không** xóa
  static Pod — Pod xuất hiện lại ngay. Cách xóa thật nằm ở mục *Thêm và xóa static Pod một cách
  động*: gỡ file manifest khỏi thư mục trên chính node đó.
- Kubelet giám sát container của static Pod trực tiếp: dừng container bằng tay trên node thì sau
  một khoảng thời gian kubelet nhận ra và tự khởi động lại Pod, `ATTEMPT` tăng lên trong cùng một
  POD ID. Nguồn sự thật của static Pod là **file trên node**, không phải object trên API server.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Bước "Cấu hình kubelet để đặt `staticPodPath`" và đối số deprecated `--pod-manifest-path`, cùng link tới bài [224](224-kubelet-config-file-vi.md) | sửa file cấu hình kubelet là làm lệch baseline của Lab 00; Lab 3c chỉ **đọc** `staticPodPath` có sẵn | bài [224](224-kubelet-config-file-vi.md) ở [giai đoạn 20](00-ALO-TRINH-ADMIN.md#giai-đoạn-20--cấu-hình-lại-cluster-đang-chạy) |
| Toàn mục *Manifest static Pod đặt trên web* — `staticPodURL` và `--manifest-url` | cần dựng thêm một web server và vẫn phải sửa cấu hình kubelet; ở đây chỉ cần biết đường này tồn tại | [giai đoạn 20](00-ALO-TRINH-ADMIN.md#giai-đoạn-20--cấu-hình-lại-cluster-đang-chạy) |
| `crictl logs`, cách đọc URI kèm checksum SHA-256 của image trong output `crictl ps`, và link *Debug các node Kubernetes bằng `crictl`* | debug ở tầng runtime là chủ đề riêng; Lab 3c kiểm chứng static Pod bằng `kubectl` chứ không qua `crictl` | bài [307](307-crictl-vi.md) ở [giai đoạn 24](00-ALO-TRINH-ADMIN.md#giai-đoạn-24--xử-lý-sự-cố) |
| Các mục *Tiếp theo* về sinh manifest static Pod cho thành phần control plane và cho etcd cục bộ, cùng bài [07](07-setup-ha-etcd-with-kubeadm-vi.md) | `kubeadm` sinh ra đúng bốn manifest bạn sẽ đọc ở B8.1, nhưng cách nó sinh ra thì chưa học | [giai đoạn 8](00-ALO-TRINH-ADMIN.md#giai-đoạn-8--dựng-cluster-bằng-kubeadm) |

---

Trang này hướng dẫn bạn cách tạo _static Pod_ trên một node. Để có cái nhìn tổng quan về static
Pod là gì và khi nào nên dùng chúng, xem [Static Pod](./58-static-pods-vi.md).

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

Trang này giả định rằng bạn đang dùng CRI-O để chạy Pod, và các node của bạn chạy hệ điều hành
Fedora. Hướng dẫn cho các bản phân phối khác hoặc các cách cài đặt Kubernetes khác có thể
khác đi.

## Tạo một static Pod (Create a static pod) {#static-pod-creation}

Bạn có thể cấu hình một static Pod bằng [file cấu hình đặt trên hệ thống file](#configuration-files)
hoặc [file cấu hình đặt trên web](#pods-created-via-http).

### Manifest static Pod đặt trên hệ thống file (Filesystem-hosted static Pod manifest) {#configuration-files}

Manifest là các định nghĩa Pod tiêu chuẩn ở định dạng JSON hoặc YAML nằm trong một thư mục cụ
thể. Hãy dùng trường `staticPodPath: <thư mục>` trong
[file cấu hình kubelet](https://kubernetes.io/docs/reference/config-api/kubelet-config.v1beta1/);
kubelet sẽ định kỳ quét thư mục này và tạo/xóa static Pod khi các file YAML/JSON xuất hiện/biến
mất trong đó. Lưu ý rằng kubelet sẽ bỏ qua các file bắt đầu bằng dấu chấm khi quét thư mục
được chỉ định.

> **Thận trọng:** Kubelet xử lý **tất cả các file không bắt đầu bằng dấu chấm** trong thư mục
> static Pod — không hề có việc lọc theo phần mở rộng của file. Ví dụ, nếu bạn tạo một bản sao
> lưu (backup) của manifest bằng cách chạy `cp kube-apiserver.yaml kube-apiserver.yaml.backup`,
> kubelet sẽ đọc **cả hai** file và cố gắng tạo một static Pod từ mỗi file. Khi hai file định
> nghĩa một Pod có cùng tên, hành vi thu được là không xác định (undefined) và có thể khiến
> spec lỗi thời của bản backup âm thầm có hiệu lực thay cho manifest hiện tại. Nếu bạn có tạo
> bản backup, hãy lưu nó **bên ngoài** thư mục static Pod (ví dụ, trong
> `/etc/kubernetes/backup/`).

Ví dụ, đây là cách khởi động một web server đơn giản dưới dạng static Pod:

1. Chọn một node mà bạn muốn chạy static Pod. Trong ví dụ này, đó là `my-node1`.

   ```shell
   ssh my-node1
   ```

1. Chọn một thư mục, chẳng hạn `/etc/kubernetes/manifests`, và đặt một định nghĩa Pod web
   server vào đó, ví dụ `/etc/kubernetes/manifests/static-web.yaml`:

   ```shell
   # Chạy lệnh này trên node nơi kubelet đang chạy
   mkdir -p /etc/kubernetes/manifests/
   cat <<EOF >/etc/kubernetes/manifests/static-web.yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: static-web
     labels:
       role: myrole
   spec:
     containers:
       - name: web
         image: nginx
         ports:
           - name: web
             containerPort: 80
             protocol: TCP
   EOF
   ```

1. Cấu hình kubelet trên node đó để đặt giá trị `staticPodPath` trong
   [file cấu hình kubelet](https://kubernetes.io/docs/reference/config-api/kubelet-config.v1beta1/).
   Xem [Thiết lập tham số kubelet qua file cấu hình](./224-kubelet-config-file-vi.md)
   để biết thêm thông tin.

   Một cách thay thế đã bị loại bỏ dần (deprecated) là cấu hình kubelet trên node đó để tìm
   manifest static Pod cục bộ bằng một đối số dòng lệnh. Để dùng cách tiếp cận đã deprecated
   này, hãy khởi động kubelet với đối số `--pod-manifest-path=/etc/kubernetes/manifests/`.

1. Khởi động lại kubelet. Trên Fedora, bạn sẽ chạy:

   ```shell
   # Chạy lệnh này trên node nơi kubelet đang chạy
   systemctl restart kubelet
   ```

### Manifest static Pod đặt trên web (Web-hosted static pod manifest) {#pods-created-via-http}

Kubelet định kỳ tải xuống file được chỉ định bởi đối số `--manifest-url=<URL>` và diễn giải nó
như một file JSON/YAML chứa các định nghĩa Pod. Tương tự cách
[manifest đặt trên hệ thống file](#configuration-files) hoạt động, kubelet tải lại manifest
theo lịch định kỳ. Nếu có thay đổi trong danh sách static Pod, kubelet sẽ áp dụng chúng.

Để dùng cách tiếp cận này:

1. Tạo một file YAML và lưu nó trên một web server sao cho bạn có thể truyền URL của file đó
   cho kubelet.

   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: static-web
     labels:
       role: myrole
   spec:
     containers:
       - name: web
         image: nginx
         ports:
           - name: web
             containerPort: 80
             protocol: TCP
   ```

1. Cấu hình kubelet trên node bạn đã chọn để dùng manifest trên web này bằng cách cập nhật
   file cấu hình kubelet để bao gồm trường `staticPodURL`:

   ```yaml
   apiVersion: kubelet.config.k8s.io/v1beta1
   kind: KubeletConfiguration
   staticPodURL: "<manifest-url>"
   ```

1. Khởi động lại kubelet. Trên Fedora, bạn sẽ chạy:

   ```shell
   # Chạy lệnh này trên node nơi kubelet đang chạy
   systemctl restart kubelet
   ```

## Quan sát hành vi của static Pod (Observe static pod behavior) {#behavior-of-static-pods}

Khi kubelet khởi động, nó tự động khởi động tất cả các static Pod đã được định nghĩa. Vì bạn
đã định nghĩa một static Pod và khởi động lại kubelet, static Pod mới hẳn đã đang chạy rồi.

Bạn có thể xem các container đang chạy (bao gồm cả static Pod) bằng cách chạy (trên node):

```shell
# Chạy lệnh này trên node nơi kubelet đang chạy
crictl ps
```

Output có thể trông giống như sau:

```console
CONTAINER       IMAGE                                 CREATED           STATE      NAME    ATTEMPT    POD ID
129fd7d382018   docker.io/library/nginx@sha256:...    11 minutes ago    Running    web     0          34533c6729106
```

> **Ghi chú:** `crictl` in ra URI của image và checksum SHA-256. `NAME` sẽ trông giống như:
> `docker.io/library/nginx@sha256:0d17b565c37bcbd895e9d92315a05c1c3c9a29f762b011a10c54a66cd53c9b31`.

Bạn có thể thấy mirror Pod trên API server:

```shell
kubectl get pods
```
```console
NAME                  READY   STATUS    RESTARTS        AGE
static-web-my-node1   1/1     Running   0               2m
```

> **Ghi chú:** Hãy đảm bảo kubelet có quyền tạo mirror Pod trong API server. Nếu không, yêu
> cầu tạo sẽ bị API server từ chối.

Các Label từ static Pod được lan truyền (propagate) sang mirror Pod. Bạn có thể dùng các label
đó một cách bình thường thông qua các selector, v.v.

Nếu bạn thử dùng `kubectl` để xóa mirror Pod khỏi API server, kubelet sẽ _không_ xóa static
Pod:

```shell
kubectl delete pod static-web-my-node1
```
```console
pod "static-web-my-node1" deleted
```

Bạn có thể thấy rằng Pod vẫn đang chạy:

```shell
kubectl get pods
```
```console
NAME                  READY   STATUS    RESTARTS   AGE
static-web-my-node1   1/1     Running   0          4s
```

Quay lại node nơi kubelet đang chạy, bạn có thể thử dừng container theo cách thủ công. Bạn sẽ
thấy rằng, sau một khoảng thời gian, kubelet sẽ nhận ra và tự động khởi động lại Pod:

```shell
# Chạy các lệnh này trên node nơi kubelet đang chạy
crictl stop 129fd7d382018 # thay bằng ID container của bạn
sleep 20
crictl ps
```

```console
CONTAINER       IMAGE                                 CREATED           STATE      NAME    ATTEMPT    POD ID
89db4553e1eeb   docker.io/library/nginx@sha256:...    19 seconds ago    Running    web     1          34533c6729106
```

Khi bạn đã xác định được đúng container, bạn có thể lấy log của container đó bằng `crictl`:

```shell
# Chạy các lệnh này trên node nơi container đang chạy
crictl logs <container_id>
```

```console
10.240.0.48 - - [16/Nov/2022:12:45:49 +0000] "GET / HTTP/1.1" 200 612 "-" "curl/7.47.0" "-"
10.240.0.48 - - [16/Nov/2022:12:45:50 +0000] "GET / HTTP/1.1" 200 612 "-" "curl/7.47.0" "-"
10.240.0.48 - - [16/Nove/2022:12:45:51 +0000] "GET / HTTP/1.1" 200 612 "-" "curl/7.47.0" "-"
```

Để tìm hiểu thêm về cách debug bằng `crictl`, hãy xem
[_Debug các node Kubernetes bằng crictl_](307-crictl-vi.md).

## Thêm và xóa static Pod một cách động (Dynamic addition and removal of static pods)

Kubelet đang chạy sẽ định kỳ quét thư mục đã cấu hình (`/etc/kubernetes/manifests` trong ví dụ
của chúng ta) để tìm các thay đổi, và thêm/xóa Pod khi các file xuất hiện/biến mất trong thư
mục này.

```shell
# Phần này giả định bạn đang dùng cấu hình static Pod đặt trên hệ thống file
# Chạy các lệnh này trên node nơi container đang chạy
mv /etc/kubernetes/manifests/static-web.yaml /tmp
sleep 20
crictl ps
# Bạn thấy rằng không có container nginx nào đang chạy
mv /tmp/static-web.yaml  /etc/kubernetes/manifests/
sleep 20
crictl ps
```
```console
CONTAINER       IMAGE                                 CREATED           STATE      NAME    ATTEMPT    POD ID
f427638871c35   docker.io/library/nginx@sha256:...    19 seconds ago    Running    web     1          34533c6729106
```

## Tiếp theo (What's next)

* [Static Pod](./58-static-pods-vi.md)
* [Sinh manifest static Pod cho các thành phần control plane](https://kubernetes.io/docs/reference/setup-tools/kubeadm/implementation-details/#generate-static-pod-manifests-for-control-plane-components)
* [Sinh manifest static Pod cho etcd cục bộ](https://kubernetes.io/docs/reference/setup-tools/kubeadm/implementation-details/#generate-static-pod-manifest-for-local-etcd)
* [Debug các node Kubernetes bằng `crictl`](307-crictl-vi.md)
* [Tìm hiểu thêm về `crictl`](https://github.com/kubernetes-sigs/cri-tools)
* [Thiết lập các instance etcd dưới dạng static Pod do kubelet quản lý](./07-setup-ha-etcd-with-kubeadm-vi.md)

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 3.

1. Trên `lab-k8s-worker2` bạn ghi file `/etc/kubernetes/manifests/lab3c-static.yaml` định nghĩa một
   Pod tên `lab3c-static`, và **không** chạy `kubectl apply` lần nào. Ít lâu sau, `kubectl get pods`
   chạy trên `lab-k8s-master` vẫn thấy một Pod mới. Pod đó tên gì, ai tạo ra nó, và vì sao `kubectl`
   nhìn thấy được nó?
2. **Câu bẫy.** Bạn chạy `kubectl delete pod lab3c-static-lab-k8s-worker2` và kubectl in ra
   `pod "lab3c-static-lab-k8s-worker2" deleted`. Static Pod đã bị xóa chưa? Cách duy nhất để xóa
   thật là gì?
3. Bạn muốn sao lưu manifest apiserver trên `lab-k8s-master` bằng
   `cp kube-apiserver.yaml kube-apiserver.yaml.backup`, thực hiện ngay trong
   `/etc/kubernetes/manifests`. Chuyện gì xảy ra, và bài bảo lưu bản backup ở đâu?
4. Trên node, bạn dừng thủ công container của một static Pod. Vài chục giây sau bạn liệt kê
   container thì thấy gì, và điều đó chứng minh thành phần nào đang giám sát Pod này?
5. Bạn `mv` file manifest ra `/tmp`, lát sau `mv` nó trở lại thư mục cũ. Kubelet phản ứng ra sao ở
   mỗi lần, và có phải khởi động lại kubelet để nó nhận ra hai thay đổi đó không?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Pod tên **`lab3c-static-lab-k8s-worker2`** — tên trong manifest cộng hostname của node, đúng
   theo ví dụ của bài: manifest `static-web` đặt trên `my-node1` hiện ra thành
   `static-web-my-node1`.
   Người tạo là **kubelet của chính `lab-k8s-worker2`**: nó quét thư mục `staticPodPath` theo định
   kỳ, thấy file YAML mới thì khởi động Pod. `kubectl` thấy được vì kubelet còn tạo thêm một
   **mirror Pod** trên API server để phản chiếu Pod đó — bạn đang nhìn bản phản chiếu, không phải
   static Pod. (Kubelet phải có quyền tạo mirror Pod, nếu không API server từ chối yêu cầu.)
2. **Chưa.** Bài nói rõ: nếu bạn dùng `kubectl` xóa mirror Pod khỏi API server thì kubelet **không**
   xóa static Pod — chạy `kubectl get pods` ngay sau đó vẫn thấy Pod `Running`, chỉ khác là `AGE`
   đếm lại từ đầu vì mirror Pod vừa được dựng lại. Thứ bạn xóa được chỉ là bản phản chiếu. Cách duy
   nhất là **xóa file manifest khỏi thư mục static Pod trên chính node đó**, đúng như mục *Thêm và
   xóa static Pod một cách động* làm.
3. Kubelet xử lý **tất cả các file không bắt đầu bằng dấu chấm** và **không** lọc theo phần mở
   rộng, nên nó đọc **cả hai** file và cố tạo một static Pod từ mỗi file. Hai file lại định nghĩa
   một Pod **cùng tên**, nên hành vi là **không xác định** — spec lỗi thời trong bản backup có thể
   âm thầm có hiệu lực thay cho manifest hiện tại. Bài yêu cầu lưu backup **bên ngoài** thư mục
   static Pod, ví dụ `/etc/kubernetes/backup/`.
4. Bạn thấy container **đã chạy lại**: cùng POD ID, nhưng `ATTEMPT` tăng thêm 1 và thời gian tạo
   mới tinh. Thành phần làm việc đó là **kubelet** — nó giám sát trực tiếp static Pod trên node và
   tự khởi động lại, không cần yêu cầu nào đi qua API server.
5. Kubelet **quét thư mục đã cấu hình theo định kỳ** để tìm thay đổi, rồi thêm/xóa Pod khi file
   xuất hiện/biến mất. Lúc `mv` đi: một lát sau liệt kê container không còn container nginx nào.
   Lúc `mv` về: container chạy lại. **Không** phải khởi động lại kubelet — bài chỉ restart kubelet ở
   bước đầu, khi thay đổi *cấu hình* (`staticPodPath` / `staticPodURL`), chứ không phải khi thay đổi
   nội dung thư mục.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng. Đây là bài cuối của nhóm
[3c](00-ALO-TRINH-ADMIN.md#3c-tài-nguyên-qos-và-gián-đoạn): trả lời trôi chảy cả năm câu rồi thì mở
[Lab 3c — Tài nguyên, QoS và gián đoạn](labs/LAB-3C-TAI-NGUYEN-QOS-VA-GIAN-DOAN.md) và chạy từ B0
tới B9.
