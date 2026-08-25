# Debug các node Kubernetes bằng Kubectl (Debugging Kubernetes Nodes With Kubectl)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/debug/debug-cluster/kubectl-node-debug/>
>
> Trang này hướng dẫn cách debug một node đang chạy trong cluster Kubernetes bằng lệnh
> `kubectl debug`.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Checkpoint tiếp nối, giai đoạn 24 — Xử lý sự cố](00-ALO-TRINH-ADMIN.md#giai-đoạn-24--xử-lý-sự-cố),
bài 3/10 · nối tiếp bài [305 — Troubleshooting Clusters](305-debug-cluster-vi.md) và bài
[307 — crictl](307-crictl-vi.md); giai đoạn này không có lab riêng, thực hành trực tiếp trên
cluster lab (xem [bản đồ lab](labs/README.md#4-bản-đồ-lab)).

Bài rất ngắn: một lệnh duy nhất `kubectl debug node/...`. Giá trị nằm ở việc biết rõ Pod debug
này làm được gì và **không** làm được gì so với SSH thẳng vào node.

**Phải hiểu ở lần đọc này:**

- `kubectl debug node/<tên-node>` triển khai một Pod lên đúng node cần xem và mở shell tương
  tác trên node — dùng cho tình huống không SSH vào node được.
- Root filesystem của node được mount tại `/host` trong Pod debug; log của node đọc qua các
  đường dẫn `/host/var/log/...` (kubelet.log, kube-proxy.log, containerd.log, syslog, kern.log).
- Container debug chạy trong host IPC, Network và PID namespace nhưng Pod **không privileged**:
  một số thao tác cần quyền superuser sẽ thất bại (ví dụ `chroot /host`); khi cần Pod
  privileged thì tự tạo thủ công hoặc dùng cờ `--profile=sysadmin`.
- Pod debug không tự xóa sau khi dùng xong (nằm lại ở trạng thái `Completed`) — phải tự
  `kubectl delete pod ... --now`.
- Giới hạn của lệnh: node phải còn liên lạc được với control plane; node đã down thì quay về
  quy trình trong bài [305 — Troubleshooting Clusters](305-debug-cluster-vi.md).

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Debugging Profiles và `securityContext` cho Pod debug | cơ chế profile được trình bày ở bài khác trong cùng nhóm | bài [300 — Debug Running Pods](300-debug-running-pod-vi.md), cùng giai đoạn 24 |
| Ghi chú "kubelet chạy trong filesystem namespace" | trường hợp hiếm, chỉ gặp với cách triển khai kubelet đặc thù | tra cứu khi cần |

---

Trang này hướng dẫn cách debug một [node](23-nodes-vi.md) đang chạy trong cluster Kubernetes
bằng lệnh `kubectl debug`.

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao tiếp
với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node không đóng
vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster bằng
[minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong các sân
chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

Kubernetes server của bạn phải ở phiên bản v1.20 hoặc mới hơn. Để kiểm tra phiên bản, nhập
`kubectl version`.

Bạn cần có quyền tạo Pod và gán các Pod mới đó vào node bất kỳ. Bạn cũng cần được phép tạo
các Pod truy cập filesystem từ host.

## Debug một Node bằng `kubectl debug node` (Debugging a Node using `kubectl debug node`)

Dùng lệnh `kubectl debug node` để triển khai một Pod lên Node mà bạn muốn xử lý sự cố
(troubleshoot). Lệnh này hữu ích trong những tình huống bạn không thể truy cập Node của mình
qua kết nối SSH. Khi Pod được tạo, Pod sẽ mở một shell tương tác trên Node. Để tạo một shell
tương tác trên Node tên là "mynode", chạy:

```shell
kubectl debug node/mynode -it --image=ubuntu
```

```console
Creating debugging pod node-debugger-mynode-pdx84 with container debugger on node mynode.
If you don't see a command prompt, try pressing enter.
root@mynode:/#
```

Lệnh debug giúp thu thập thông tin và xử lý sự cố. Các lệnh bạn có thể dùng bao gồm `ip`,
`ifconfig`, `nc`, `ping`, `ps` và những lệnh tương tự. Bạn cũng có thể cài thêm các công cụ
khác, chẳng hạn `mtr`, `tcpdump` và `curl`, từ trình quản lý gói tương ứng.

> **Ghi chú:** Các lệnh debug có thể khác nhau tùy theo image mà Pod debug đang dùng, và những
> lệnh này có thể cần được cài đặt thêm.

Pod debug có thể truy cập root filesystem của Node, được mount tại `/host` trong Pod. Nếu bạn
chạy kubelet trong một filesystem namespace, Pod debug sẽ thấy root của namespace đó, không
phải của toàn bộ node. Với một node Linux điển hình, bạn có thể xem các đường dẫn sau để tìm
những log liên quan:

`/host/var/log/kubelet.log`
: Log của `kubelet`, thành phần chịu trách nhiệm chạy các container trên node.

`/host/var/log/kube-proxy.log`
: Log của `kube-proxy`, thành phần chịu trách nhiệm điều hướng lưu lượng đến các endpoint của
  Service.

`/host/var/log/containerd.log`
: Log của tiến trình `containerd` chạy trên node.

`/host/var/log/syslog`
: Hiển thị các thông điệp chung và thông tin về hệ thống.

`/host/var/log/kern.log`
: Hiển thị log của kernel.

Khi tạo một phiên debug trên Node, hãy ghi nhớ rằng:

* `kubectl debug` tự động sinh tên cho Pod mới, dựa trên tên của node.
* Root filesystem của Node sẽ được mount tại `/host`.
* Mặc dù container chạy trong host IPC, Network và PID namespace, Pod này không phải là Pod
  privileged. Điều đó có nghĩa là việc đọc một số thông tin tiến trình có thể thất bại, vì
  quyền truy cập những thông tin đó bị giới hạn cho superuser. Ví dụ, `chroot /host` sẽ thất
  bại. Nếu bạn cần một Pod privileged, hãy tạo nó thủ công hoặc dùng cờ `--profile=sysadmin`.
* Bằng cách áp dụng [Debugging Profiles](300-debug-running-pod-vi.md#debugging-profiles), bạn
  có thể đặt các thuộc tính cụ thể, chẳng hạn
  [securityContext](291-security-context-vi.md),
  cho Pod debug.

## Dọn dẹp (Cleaning up)

Khi dùng xong Pod debug, hãy xóa nó:

```shell
kubectl get pods
```

```none
NAME                          READY   STATUS       RESTARTS   AGE
node-debugger-mynode-pdx84    0/1     Completed    0          8m1s
```

```shell
# Thay tên Pod cho phù hợp
kubectl delete pod node-debugger-mynode-pdx84 --now
```

```none
pod "node-debugger-mynode-pdx84" deleted
```

> **Ghi chú:** Lệnh `kubectl debug node` sẽ không hoạt động nếu Node đã down (bị ngắt khỏi
> mạng, hoặc kubelet chết và không khởi động lại được, v.v.). Trong trường hợp đó, xem mục
> "Ví dụ: gỡ lỗi một node bị down hoặc không thể kết nối" trong bài
> [Khắc phục sự cố cluster](305-debug-cluster-vi.md).

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn này.

1. Bạn không SSH được vào `k8s-worker2` nhưng node vẫn `Ready`. Lệnh nào cho bạn một shell
   trên node đó, và Pod mà lệnh này tạo ra được đặt tên theo quy tắc nào?
2. Trong shell của Pod debug, muốn đọc log kubelet của node thì bạn mở đường dẫn nào? Vì sao
   không phải là `/var/log/kubelet.log`?
3. Câu bẫy: Pod debug chạy trong host PID namespace, vậy `chroot /host` bên trong nó có chạy
   được không? Nếu không, bạn có những cách nào để có quyền cao hơn?
4. Sau khi thoát shell debug, cluster của bạn có còn "sạch" không? Bạn phải làm gì tiếp và
   kiểm chứng bằng lệnh nào?
5. `kubectl debug node` có giúp gì được khi node hiện `NotReady` vì kubelet đã chết hẳn không?
   Vì sao?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **`kubectl debug node/k8s-worker2 -it --image=ubuntu`.** Pod được `kubectl debug` **tự
   sinh tên dựa trên tên node**, dạng `node-debugger-k8s-worker2-<hậu tố ngẫu nhiên>`.
2. **`/host/var/log/kubelet.log`** — vì root filesystem của node được mount vào Pod debug tại
   **`/host`**, nên mọi đường dẫn trên node phải thêm tiền tố `/host`; `/var/log/` không có
   tiền tố chỉ là filesystem của chính container debug.
3. **Không chạy được.** Dù container nằm trong host IPC/Network/PID namespace, Pod này
   **không privileged**, nên thao tác đòi quyền superuser như `chroot /host` bị từ chối. Muốn
   quyền cao hơn: **tự tạo Pod privileged thủ công** hoặc chạy lại với cờ
   **`--profile=sysadmin`**. Trực giác "vào được host namespace nghĩa là toàn quyền trên
   host" là sai — namespace và privilege là hai chuyện khác nhau.
4. **Chưa sạch:** Pod debug không tự xóa mà nằm lại ở trạng thái `Completed`. Phải
   `kubectl delete pod node-debugger-... --now`, rồi kiểm chứng bằng `kubectl get pods` thấy
   Pod đã biến mất.
5. **Không.** Lệnh này hoạt động bằng cách tạo một Pod và để kubelet trên node đó chạy Pod —
   node mất liên lạc hoặc kubelet chết thì Pod không bao giờ chạy được. Khi đó phải theo
   hướng debug node down trong bài [305 — Troubleshooting Clusters](305-debug-cluster-vi.md).

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
