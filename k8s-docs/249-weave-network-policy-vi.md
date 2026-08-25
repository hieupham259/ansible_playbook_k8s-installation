# Weave Net cho NetworkPolicy (Weave Net for NetworkPolicy)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/administer-cluster/network-policy-provider/weave-network-policy/

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:**
[Phần II — Vận hành cluster](00-ALO-TRINH-ADMIN.md#phần-ii--vận-hành-cluster)
→ [Giai đoạn 21 — DNS, CNI và kube-proxy](00-ALO-TRINH-ADMIN.md#giai-đoạn-21--dns-cni-và-kube-proxy)
— **trang con của bài 7/14**, mục [Cài đặt một Network Policy
Provider](243-network-policy-provider-vi.md); nó không có dòng riêng trong lộ trình. Phần II không
có lab: thực hành thẳng trên cluster VM của [Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md) và tự chấm
bằng **Checkpoint** ở cuối [mục giai đoạn
21](00-ALO-TRINH-ADMIN.md#giai-đoạn-21--dns-cni-và-kube-proxy).

Đây là trang provider **duy nhất trong nhóm có mục kiểm tra sau cài đặt kèm output thật**, nên
đọc nó bù được phần mà bốn trang kia bỏ trống. Cluster lab đã chạy Calico từ
[Lab 5b](labs/LAB-5B-NETWORKPOLICY-INGRESS-VA-CNI.md), nên chỉ bài
[245](245-calico-network-policy-vi.md) khớp cluster của bạn. **Không cài Weave Net** vào cluster
lab: đổi CNI sẽ phá [chuỗi snapshot](labs/README.md#3-chuỗi-snapshot) mà các lab sau dựa vào. Đọc
để biết Weave Net khác các provider kia ở đâu, không để làm theo.

**Phải hiểu ở lần đọc này:**

- Mục *Kiểm tra việc cài đặt* dạy đúng một phép đọc: `kubectl get pods -n kube-system -o wide`,
  **mỗi Node có một Pod weave**, tất cả ở trạng thái `Running` và cột READY hiện **`2/2`**.
- Bài giải thích luôn `2/2` nghĩa là gì: mỗi Pod chứa **hai container**, `weave` và `weave-npc`.
  Đây là điều riêng của trang này — nó cho thấy phần mạng và phần **thực thi network policy** là
  hai tiến trình tách rời nằm chung một Pod, thay vì một khối duy nhất.
- Cơ chế mà mục *Cài đặt addon Weave Net* mô tả: addon đi kèm một **Network Policy Controller**
  tự động giám sát Kubernetes để phát hiện **mọi annotation NetworkPolicy trên tất cả các
  namespace**, rồi cấu hình các quy tắc **`iptables`** để cho phép hoặc chặn lưu lượng. Ghi nhận
  đúng chữ *annotation* mà bài dùng và đối chiếu với thứ bạn thật sự khai ở bài
  [201](201-declare-network-policy-vi.md).

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Hướng dẫn *Tích hợp Kubernetes qua Addon* mà mục cài đặt trỏ tới | đó là quy trình cài một CNI thật | **không chạy trên cluster lab** — cluster đã dùng Calico từ [Lab 5b](labs/LAB-5B-NETWORKPOLICY-INGRESS-VA-CNI.md) và đổi CNI phá [chuỗi snapshot](labs/README.md#3-chuỗi-snapshot). Chỉ mở khi bạn tiếp quản một cluster đã chạy Weave Net |
| Tên node và địa chỉ IP trong output mẫu — `worknode3`, `masternode`, `192.168.2.10`… | là output của một cluster khác, không phải cluster của bạn | tên ba VM và dải IP của cluster lab nằm ở [Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md); cách đọc chúng đã làm ở [Lab 5b](labs/LAB-5B-NETWORKPOLICY-INGRESS-VA-CNI.md) phần B1 |
| Kênh Slack `#weave-community` và Weave User Group ở mục *Tiếp theo* | là kênh hỗ trợ của một dự án bên ngoài | không có trong lộ trình |

---

Trang này hướng dẫn cách sử dụng Weave Net cho NetworkPolicy.

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes. Hãy làm theo
[hướng dẫn bắt đầu với kubeadm](https://kubernetes.io/docs/reference/setup-tools/kubeadm/) để dựng (bootstrap) một cluster.

## Cài đặt addon Weave Net (Install the Weave Net addon)

Làm theo hướng dẫn [Tích hợp Kubernetes qua Addon](https://github.com/weaveworks/weave/blob/master/site/kubernetes/kube-addon.md#-installation).

Addon Weave Net cho Kubernetes đi kèm một
[Network Policy Controller](https://github.com/weaveworks/weave/blob/master/site/kubernetes/kube-addon.md#network-policy)
tự động giám sát Kubernetes để phát hiện mọi annotation NetworkPolicy trên tất cả các
namespace và cấu hình các quy tắc `iptables` nhằm cho phép hoặc chặn lưu lượng theo đúng chỉ dẫn của các policy.

## Kiểm tra việc cài đặt (Test the installation)

Xác minh rằng weave hoạt động.

Nhập lệnh sau:

```shell
kubectl get pods -n kube-system -o wide
```

Kết quả xuất ra tương tự như sau:

```
NAME                                    READY     STATUS    RESTARTS   AGE       IP              NODE
weave-net-1t1qg                         2/2       Running   0          9d        192.168.2.10    worknode3
weave-net-231d7                         2/2       Running   1          7d        10.2.0.17       worknodegpu
weave-net-7nmwt                         2/2       Running   3          9d        192.168.2.131   masternode
weave-net-pmw8w                         2/2       Running   0          9d        192.168.2.216   worknode2
```

Mỗi Node có một Pod weave, và tất cả các Pod đều ở trạng thái `Running` và `2/2 READY`. (`2/2` nghĩa là mỗi Pod có `weave` và `weave-npc`.)

## Tiếp theo (What's next)

Sau khi đã cài đặt addon Weave Net, bạn có thể làm theo bài
[Khai báo Network Policy](201-declare-network-policy-vi.md)
để thử nghiệm NetworkPolicy của Kubernetes. Nếu bạn có bất kỳ câu hỏi nào, hãy liên hệ với chúng tôi tại
[#weave-community trên Slack hoặc Weave User Group](https://github.com/weaveworks/weave#getting-help).

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 21:

1. Trong output mẫu, cột READY hiện `2/2`. Con số đó nói lên điều gì về cấu tạo của một Pod weave,
   và hai thành phần bên trong chia nhau việc gì?
2. **Câu bẫy.** Bài nói Network Policy Controller giám sát để phát hiện *mọi **annotation**
   NetworkPolicy trên tất cả các namespace*. Nhưng thứ bạn viết ở
   [Lab 5b](labs/LAB-5B-NETWORKPOLICY-INGRESS-VA-CNI.md) là một **object** `NetworkPolicy`, không
   phải annotation. Đọc câu đó thế nào cho đúng, và mục *Tiếp theo* của chính bài chốt lại bằng gì?
3. Cluster lab của bạn có ba node: `lab-k8s-master`, `lab-k8s-worker1`, `lab-k8s-worker2`. Nếu
   Weave Net chạy trên đó, theo mục *Kiểm tra việc cài đặt* bạn trông đợi thấy bao nhiêu Pod weave
   và ở trạng thái nào? Và vì sao bạn vẫn không cài nó lên cluster này?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. `2/2` nghĩa là Pod đó có **hai container và cả hai đều sẵn sàng**. Bài nói rõ hai container ấy
   là **`weave` và `weave-npc`**: phần mạng và phần thực thi network policy **tách làm hai tiến
   trình** trong cùng một Pod. Vì vậy một Pod weave hiện `1/2` là tín hiệu hỏng — một trong hai
   nửa chưa lên, và nửa thiếu quyết định bạn mất mạng hay mất phần cưỡng chế policy.
2. Đọc nó như **mô tả của trang về controller**, không phải chỉ dẫn cho bạn khai báo. Chính bài
   khép lại ở mục *Tiếp theo*: sau khi cài addon, bạn làm theo bài
   [201](201-declare-network-policy-vi.md) để thử **NetworkPolicy của Kubernetes** — tức vẫn là
   object chuẩn, giống hệt bốn trang provider còn lại trong nhóm. Chỗ dễ sai là đọc chữ
   *annotation* thành một cú pháp khai báo khác và đi tìm cách viết annotation.
3. **Ba Pod — một trên mỗi Node** — tất cả ở trạng thái `Running` và cột READY hiện `2/2`, đúng
   theo câu "Mỗi Node có một Pod weave" của bài. Lý do không cài: cluster lab **đã có provider
   thực thi NetworkPolicy** là Calico từ [Lab 5b](labs/LAB-5B-NETWORKPOLICY-INGRESS-VA-CNI.md);
   thêm Weave Net là **đổi CNI**, phá [chuỗi snapshot](labs/README.md#3-chuỗi-snapshot) mà các lab
   sau dựa vào. Ở giai đoạn 21, năm trang provider đọc để **biết lựa chọn và biết chúng khác nhau
   ở đâu**, không phải để thi công.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
