# Thêm entry vào file /etc/hosts của Pod bằng HostAliases (Adding entries to Pod /etc/hosts with HostAliases)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/network/customize-hosts-file-for-pods/>
>
> Thêm entry vào file `/etc/hosts` của một Pod cho phép ghi đè việc phân giải hostname ở cấp Pod,
> dùng khi DNS và các phương án khác không áp dụng được.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:**
[Phần II — Vận hành cluster](00-ALO-TRINH-ADMIN.md#phần-ii--vận-hành-cluster)
→ [Giai đoạn 21 — DNS, CNI và kube-proxy](00-ALO-TRINH-ADMIN.md#giai-đoạn-21--dns-cni-và-kube-proxy),
bài 11/14 · Phần II không có lab: thực hành thẳng trên cluster VM của
[Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md) và tự chấm bằng **Checkpoint** ở cuối
[mục giai đoạn 21](00-ALO-TRINH-ADMIN.md#giai-đoạn-21--dns-cni-và-kube-proxy). Bài này chạy được
trọn vẹn trên cluster lab: tạo Pod, `kubectl exec ... -- cat /etc/hosts`, so trước và sau khi
thêm `hostAliases`.

Bài ngắn và làm được ngay. Nó nối tiếp bài [10](10-dns-pod-service-vi.md) và bài
[57](57-pod-hostname-vi.md) đã đọc ở mạch chính.

**Phải hiểu ở lần đọc này:**

- `hostAliases` là một trường trong `.spec` của Pod, ghi đè việc phân giải hostname **ở cấp Pod**,
  dùng cho trường hợp DNS và các lựa chọn khác không phù hợp — không phải để thay DNS.
- Khuyến nghị ở đầu bài: đổi phân giải tên **qua `hostAliases`**, không dùng init container hay
  cách nào khác sửa trực tiếp `/etc/hosts`; thay đổi làm theo cách khác **có thể bị kubelet ghi
  đè** trong lúc tạo hoặc khởi động lại Pod.
- Nội dung mặc định ở mục *Nội dung mặc định của file hosts*: chỉ có các dòng khuôn mẫu IPv4/IPv6
  cho `localhost`, cộng đúng một dòng ánh xạ **Pod IP → hostname của chính Pod**.
- Hình dạng của trường ở mục *Thêm entry bổ sung bằng hostAliases*: mỗi mục là một cặp `ip` và
  danh sách `hostnames`; kết quả được nối vào **cuối file**, sau dòng đánh dấu
  `# Entries added by HostAliases.`
- Lý do ở mục *Vì sao kubelet quản lý file hosts?*: kubelet giữ quyền quản lý để container runtime
  không sửa file sau khi container khởi động, nhờ đó kết quả như nhau với mọi runtime. Hệ quả
  thực tế nằm ở ô *Thận trọng*: **sửa tay file hosts trong container thì thay đổi mất khi
  container thoát**.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Các dòng khuôn mẫu IPv6 (`::1`, `fe00::0`, `ip6-allnodes`…) trong output mẫu | cluster lab chạy thuần IPv4 nên các dòng này không dùng tới | bài [395](395-validate-dual-stack-vi.md) — bài 14/14 của giai đoạn 21 |
| Đoạn lịch sử "Docker Engine sửa `/etc/hosts` sau khi container khởi động" | thuộc thời dockershim, không phải runtime của cluster lab | [Giai đoạn 27 — Di chuyển khỏi dockershim](00-ALO-TRINH-ADMIN.md#giai-đoạn-27--di-chuyển-khỏi-dockershim-cluster-cũ) |
| Lệnh `kubectl apply -f https://k8s.io/examples/...` | tải manifest từ internet; bản YAML đã in nguyên trong bài | dán thẳng khối YAML in ở mục *Thêm entry bổ sung* vào file cục bộ trên `lab-k8s-master` theo [A1.2 của Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md#a12-ba-vm) |
| Pod IP `10.200.0.x` và tên node `worker0` trong output mẫu | là output của cluster trong tài liệu gốc | Pod IP trên cluster lab nằm trong `10.244.0.0/16` theo [A1.3 của Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa) |

---

Việc thêm entry vào file `/etc/hosts` của một Pod cho phép ghi đè việc phân giải hostname
(hostname resolution) ở cấp Pod, dùng cho trường hợp DNS và các lựa chọn khác không phù hợp.
Bạn có thể thêm các entry tùy chỉnh này bằng trường HostAliases trong PodSpec.

Dự án Kubernetes khuyến nghị bạn thay đổi cấu hình DNS thông qua trường `hostAliases`
(một phần của `.spec` của Pod), chứ không dùng init container hay cách nào khác để sửa trực tiếp
`/etc/hosts`.
Thay đổi thực hiện theo những cách khác có thể bị kubelet ghi đè trong quá trình tạo hoặc khởi
động lại Pod.

## Nội dung mặc định của file hosts (Default hosts file content)

Khởi chạy một Pod Nginx được cấp một Pod IP:

```shell
kubectl run nginx --image nginx
```

```
pod/nginx created
```

Kiểm tra Pod IP:

```shell
kubectl get pods --output=wide
```

```
NAME     READY     STATUS    RESTARTS   AGE    IP           NODE
nginx    1/1       Running   0          13s    10.200.0.4   worker0
```

Nội dung file hosts sẽ trông như sau:

```shell
kubectl exec nginx -- cat /etc/hosts
```

```
# Kubernetes-managed hosts file.
127.0.0.1	localhost
::1	localhost ip6-localhost ip6-loopback
fe00::0	ip6-localnet
fe00::0	ip6-mcastprefix
fe00::1	ip6-allnodes
fe00::2	ip6-allrouters
10.200.0.4	nginx
```

Theo mặc định, file `hosts` chỉ chứa các dòng khuôn mẫu (boilerplate) cho IPv4 và IPv6 như
`localhost` cùng với hostname của chính Pod đó.

## Thêm entry bổ sung bằng hostAliases (Adding additional entries with hostAliases)

Ngoài phần khuôn mẫu mặc định, bạn có thể thêm các entry khác vào file `hosts`.
Ví dụ: để phân giải `foo.local`, `bar.local` thành `127.0.0.1` và `foo.remote`,
`bar.remote` thành `10.1.2.3`, bạn có thể cấu hình HostAliases cho một Pod tại
`.spec.hostAliases`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hostaliases-pod
spec:
  restartPolicy: Never
  hostAliases:
  - ip: "127.0.0.1"
    hostnames:
    - "foo.local"
    - "bar.local"
  - ip: "10.1.2.3"
    hostnames:
    - "foo.remote"
    - "bar.remote"
  containers:
  - name: cat-hosts
    image: busybox:1.28
    command:
    - cat
    args:
    - "/etc/hosts"
```

Bạn có thể khởi chạy một Pod với cấu hình đó bằng cách chạy:

```shell
kubectl apply -f https://k8s.io/examples/service/networking/hostaliases-pod.yaml
```

```
pod/hostaliases-pod created
```

Xem chi tiết của Pod để biết địa chỉ IPv4 và trạng thái của nó:

```shell
kubectl get pod --output=wide
```

```
NAME                           READY     STATUS      RESTARTS   AGE       IP              NODE
hostaliases-pod                0/1       Completed   0          6s        10.200.0.5      worker0
```

Nội dung file `hosts` trông như sau:

```shell
kubectl logs hostaliases-pod
```

```
# Kubernetes-managed hosts file.
127.0.0.1	localhost
::1	localhost ip6-localhost ip6-loopback
fe00::0	ip6-localnet
fe00::0	ip6-mcastprefix
fe00::1	ip6-allnodes
fe00::2	ip6-allrouters
10.200.0.5	hostaliases-pod

# Entries added by HostAliases.
127.0.0.1	foo.local	bar.local
10.1.2.3	foo.remote	bar.remote
```

với các entry bổ sung được ghi ở cuối file.

## Vì sao kubelet quản lý file hosts? (Why does the kubelet manage the hosts file?) {#why-does-kubelet-manage-the-hosts-file}

kubelet quản lý file `hosts` cho từng container của Pod nhằm ngăn container runtime sửa file này
sau khi các container đã khởi động.
Về mặt lịch sử, Kubernetes luôn dùng Docker Engine làm container runtime, và Docker Engine sẽ
sửa file `/etc/hosts` sau khi mỗi container khởi động.

Kubernetes hiện nay có thể dùng nhiều container runtime khác nhau; dù vậy, kubelet vẫn quản lý
file hosts bên trong từng container để kết quả luôn đúng như mong đợi, bất kể bạn dùng container
runtime nào.

> **Thận trọng:** Tránh sửa thủ công file hosts bên trong một container.
>
> Nếu bạn sửa thủ công file hosts, những thay đổi đó sẽ mất khi container thoát.

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 21:

1. Vì sao dự án Kubernetes khuyên đổi phân giải tên bằng `hostAliases` thay vì dùng init container
   sửa `/etc/hosts`? Chuyện gì xảy ra với thay đổi làm theo cách kia?
2. Trước khi bạn thêm bất kỳ `hostAliases` nào, file `/etc/hosts` trong Pod đã có sẵn những gì?
   Trong đó dòng nào nói về chính Pod đó?
3. Bạn chạy `kubectl run nginx --image nginx` trên cluster lab và Pod được xếp lên
   `lab-k8s-worker2`. Dòng cuối cùng của `/etc/hosts` trong Pod đó sẽ ánh xạ cái gì sang cái gì,
   và địa chỉ ở đó thuộc dải nào của cluster lab?
4. **Câu bẫy.** Bạn `kubectl exec` vào container rồi thêm tay một dòng vào `/etc/hosts`, kiểm tra
   lại thấy dòng đó có thật. Dòng đó sống được bao lâu, và ai là bên quyết định điều đó?
5. Một mục trong `hostAliases` gồm những trường nào, và Kubernetes ghi kết quả vào chỗ nào trong
   file `hosts`?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Vì `hostAliases` là **cách duy nhất được kubelet tôn trọng**. Bài nói rõ: thay đổi thực hiện
   theo những cách khác **có thể bị kubelet ghi đè trong quá trình tạo hoặc khởi động lại Pod** —
   nghĩa là bản vá bằng init container có thể biến mất đúng lúc bạn không mong.
2. Chỉ có **các dòng khuôn mẫu IPv4 và IPv6 cho `localhost`**, cộng đúng **một dòng ánh xạ Pod IP
   sang hostname của chính Pod**. Đó là dòng cuối trong output mẫu của bài.
3. Nó ánh xạ **Pod IP của chính Pod đó sang tên `nginx`**. Trên cluster lab, Pod IP nằm trong
   **`10.244.0.0/16`** — cụ thể là dải `/24` mà control plane cấp riêng cho `lab-k8s-worker2`.
4. **Chỉ tới khi container thoát.** kubelet là bên quản lý file `hosts` của từng container, giữ
   quyền đó để container runtime không sửa file sau khi container khởi động. Vì vậy chỉnh sửa thủ
   công **không phải là cấu hình** — nó là một thay đổi tạm thời sẽ mất, và bài đặt hẳn ô *Thận
   trọng* để cảnh báo.
5. Mỗi mục gồm **`ip`** và danh sách **`hostnames`** — một IP có thể gắn nhiều tên. Kubernetes ghi
   chúng **vào cuối file**, dưới dòng đánh dấu **`# Entries added by HostAliases.`**, sau phần
   khuôn mẫu mặc định.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
