# Sử dụng user namespace với Pod (Use a User Namespace With a Pod)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/configure-pod-container/user-namespaces/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 3 — Pod và cấu hình](00-ALO-TRINH-ADMIN.md#giai-đoạn-3--pod-và-cấu-hình)
→ nhóm [3a. Pod và vòng đời](00-ALO-TRINH-ADMIN.md#3a-pod-và-vòng-đời), bài thực hành 8/11 ·
Kiểm chứng ở [Lab 3a](labs/LAB-3A-POD-VA-VONG-DOI.md) phần B9.2.

Bài là mặt thi công của [55](55-user-namespaces-vi.md): một trường trong manifest, và **đúng hai
lệnh** để tự chứng minh nó có tác dụng. Đọc để nắm hai lệnh đó và biết đọc kết quả của chúng.

**Phải hiểu ở lần đọc này:**

- Bật bằng field **`hostUsers: false`** trong `.spec` của Pod — chỉ vậy, không cần gì thêm trong
  manifest.
- Ý nghĩa bảo mật, theo đúng lập luận của bài: tiến trình chạy với quyền root **trong** container
  có đầy đủ đặc quyền với thao tác **bên trong** user namespace, nhưng **không có đặc quyền với
  thao tác bên ngoài**. Nếu có container breakout, nó **không** thành root trên node; và
  capability được cấp cho container **cũng không có hiệu lực trên host**.
- Cách tự chứng minh, chạy cả trong Pod lẫn trên host rồi so sánh: `readlink /proc/self/ns/user`
  — kết quả **phải khác nhau** giữa hai nơi; và `cat /proc/self/uid_map` — con số cuối **phải là
  65536** bên trong container, còn trên host thì **lớn hơn**.
- Điều kiện môi trường: **node phải là Linux**, và cluster **bắt buộc** có ít nhất một node đáp
  ứng các yêu cầu nêu ở bài [55](55-user-namespaces-vi.md). Cluster có nhiều loại node lẫn lộn
  thì phải tự lo cho Pod được lập lịch vào đúng node hỗ trợ.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Link "lập lịch (schedule) đến các node phù hợp" | ràng buộc Pod vào node là một chủ đề riêng | bài [138](138-assign-pod-node-vi.md) ở [giai đoạn 7](00-ALO-TRINH-ADMIN.md#giai-đoạn-7--lập-lịch-và-chính-sách-tài-nguyên); `nodeSelector` có làm sơ ở [Lab 3a](labs/LAB-3A-POD-VA-VONG-DOI.md) phần B11.2 |
| Danh sách yêu cầu chi tiết mà node phải đáp ứng | nằm ở bài khái niệm, không nằm ở bài này | bài [55](55-user-namespaces-vi.md) của chính nhóm 3a, thực hành ở [Lab 3a](labs/LAB-3A-POD-VA-VONG-DOI.md) phần B9.1 |
| Đoạn cuối "nếu bạn đang chạy kubelet bên trong một user namespace" và `readlink /proc/$pid/ns/user` | đó là rootless mode, chuyện của thành phần Node chứ không phải của Pod | bài [226](226-kubelet-in-userns-vi.md) của chính nhóm 3a |

---

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.36 [stable]`

Trang này hướng dẫn cách cấu hình user namespace cho các Pod. Điều này cho phép bạn cô lập
người dùng chạy bên trong container khỏi người dùng trên host.

Một tiến trình chạy với quyền root trong container có thể chạy dưới một người dùng khác
(không phải root) trên host; nói cách khác, tiến trình đó có đầy đủ đặc quyền đối với các thao
tác bên trong user namespace, nhưng không có đặc quyền đối với các thao tác bên ngoài
namespace đó.

Bạn có thể dùng tính năng này để giảm thiệt hại mà một container bị xâm nhập có thể gây ra cho
host hoặc cho các Pod khác trên cùng node. Có [một số lỗ hổng bảo mật][KEP-vulns] được đánh
giá mức **HIGH** hoặc **CRITICAL** đã không thể khai thác được khi user namespace được kích
hoạt. Người ta cũng kỳ vọng user namespace sẽ giảm nhẹ một số lỗ hổng trong tương lai.

Khi không dùng user namespace, một container chạy với quyền root — trong trường hợp thoát khỏi
container (container breakout) — sẽ có đặc quyền root trên node. Và nếu container được cấp một
capability nào đó, capability đó cũng có hiệu lực trên host. Không điều nào trong số này còn
đúng khi user namespace được sử dụng.

[KEP-vulns]: https://github.com/kubernetes/enhancements/tree/217d790720c5aef09b8bd4d6ca96284a0affe6c2/keps/sig-node/127-user-namespaces#motivation

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao tiếp
với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node không đóng
vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster bằng
[minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong các sân
chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

Server Kubernetes của bạn phải ở phiên bản v1.25 hoặc mới hơn. Để kiểm tra phiên bản, hãy nhập
`kubectl version`.

> **Ghi chú:** Mục này đề cập đến một sản phẩm hoặc dự án bên thứ ba cung cấp chức năng mà
> Kubernetes cần. Các tác giả của dự án Kubernetes không chịu trách nhiệm về sản phẩm hay dự
> án bên thứ ba đó. Xem
> [hướng dẫn website của CNCF](https://github.com/cncf/foundation/blob/master/website-guidelines.md)
> để biết thêm chi tiết.

* Hệ điều hành của node cần là Linux
* Bạn cần chạy được các lệnh trên host
* Bạn cần có khả năng exec vào các Pod

Cluster mà bạn đang dùng **bắt buộc** phải có ít nhất một node đáp ứng các
[yêu cầu](./55-user-namespaces-vi.md#trước-khi-bạn-bắt-đầu-before-you-begin)
để dùng user namespace với Pod.

Nếu bạn có nhiều loại node lẫn lộn và chỉ một số node hỗ trợ user namespace cho Pod, bạn cũng
cần đảm bảo rằng các Pod dùng user namespace được
[lập lịch (schedule)](./138-assign-pod-node-vi.md) đến các node phù hợp.

## Chạy một Pod dùng user namespace (Run a Pod that uses a user namespace) {#create-pod}

User namespace cho một Pod được bật bằng cách đặt trường `hostUsers` trong `.spec` thành
`false`. Ví dụ:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: userns
spec:
  hostUsers: false
  containers:
  - name: shell
    command: ["sleep", "infinity"]
    image: debian
```

1. Tạo Pod trên cluster của bạn:

   ```shell
   kubectl apply -f https://k8s.io/examples/pods/user-namespaces-stateless.yaml
   ```

1. Exec vào Pod và chạy `readlink /proc/self/ns/user`:

   ```shell
   kubectl exec -ti userns -- bash
   ```

Chạy lệnh này:

```shell
readlink /proc/self/ns/user
```

Output tương tự như sau:

```shell
user:[4026531837]
```

Chạy thêm:

```shell
cat /proc/self/uid_map
```

Output tương tự như sau:

```shell
0  833617920      65536
```

Sau đó, mở một shell trên host và chạy các lệnh tương tự.

Lệnh `readlink` cho biết user namespace mà tiến trình đang chạy trong đó. Kết quả này phải
khác nhau khi chạy trên host và khi chạy bên trong container.

Con số cuối cùng trong file `uid_map` bên trong container phải là 65536, còn trên host thì nó
phải là một con số lớn hơn.

Nếu bạn đang chạy kubelet bên trong một user namespace, bạn cần so sánh output khi chạy lệnh
trong Pod với output khi chạy trên host:

```shell
readlink /proc/$pid/ns/user
```

trong đó thay `$pid` bằng PID của kubelet.

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 3:

1. Field nào bật user namespace cho một Pod, và bài mô tả mức thiệt hại đổi thế nào khi container
   bị xâm nhập?
2. **Câu bẫy.** Bạn exec vào Pod `userns` và thấy mình vẫn là `root`, uid 0. Vậy `hostUsers:
   false` có tác dụng gì? Lệnh nào cho bạn thấy sự thật, và đọc kết quả ra sao?
3. Bạn apply Pod `userns` lên cluster lab và nó chạy được trên `lab-k8s-worker1`. Điều đó có bảo
   đảm nó chạy được trên `lab-k8s-worker2` không? Bài đòi cluster phải thỏa điều kiện gì, và bảo
   làm gì khi cluster có nhiều loại node lẫn lộn?
4. Ngoài `uid_map`, bài còn một lệnh nữa để đối chiếu trong Pod với trên host. Lệnh đó là gì, và
   kết quả **phải** như thế nào thì mới đúng?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **`hostUsers: false`**, đặt trong `.spec` của Pod. Bài nói mức thiệt hại đổi như sau: **không**
   dùng user namespace, container chạy root mà thoát ra được (container breakout) thì **có đặc
   quyền root trên node**, và capability cấp cho container **cũng có hiệu lực trên host**. **Có**
   user namespace thì **không điều nào trong hai điều đó còn đúng** — tiến trình chỉ đầy đủ đặc
   quyền với thao tác **bên trong** namespace. Bài dẫn thêm: một số lỗ hổng mức HIGH/CRITICAL đã
   không khai thác được khi user namespace được bật.
2. Có tác dụng, và chính đây là chỗ trực giác đánh lừa: **root bên trong container không phải là
   root trên host**. Bài mở đầu bằng đúng câu đó — "một tiến trình chạy với quyền root trong
   container có thể chạy dưới một người dùng khác (không phải root) trên host". Lệnh cho thấy sự
   thật là **`cat /proc/self/uid_map`**: trong container nó ra dạng `0  833617920  65536`, tức uid
   0 bên trong được ánh xạ sang một dải uid **không phải 0** trên host. **Con số cuối trong
   container phải là 65536**, còn chạy trên host thì con số đó **lớn hơn**.
3. **Không bảo đảm.** Bài đặt điều kiện: cluster **bắt buộc** phải có ít nhất một node đáp ứng
   các yêu cầu để dùng user namespace với Pod, và **hệ điều hành của node phải là Linux**. Khi
   cluster có nhiều loại node lẫn lộn mà chỉ một số node hỗ trợ, bài bảo bạn phải **đảm bảo Pod
   dùng user namespace được lập lịch đến đúng các node phù hợp**.
4. **`readlink /proc/self/ns/user`** — nó cho biết tiến trình đang chạy trong user namespace nào.
   Chạy trong Pod và chạy trên host phải cho **kết quả khác nhau**; giống nhau nghĩa là Pod vẫn
   đang dùng chung user namespace với host, tức tính năng chưa có hiệu lực.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
