# Cấu hình lại ServiceCIDR mặc định của Kubernetes (Kubernetes Default ServiceCIDR Reconfiguration)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/network/reconfigure-default-service-ip-ranges/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:**
[Phần II — Vận hành cluster](00-ALO-TRINH-ADMIN.md#phần-ii--vận-hành-cluster)
→ [Giai đoạn 21 — DNS, CNI và kube-proxy](00-ALO-TRINH-ADMIN.md#giai-đoạn-21--dns-cni-và-kube-proxy),
bài 13/14 · Phần II không có lab: thực hành thẳng trên cluster VM của
[Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md) và tự chấm bằng **Checkpoint** ở cuối
[mục giai đoạn 21](00-ALO-TRINH-ADMIN.md#giai-đoạn-21--dns-cni-và-kube-proxy). Bài này **đọc để
hiểu quy trình, không chạy trên cluster lab**: thay ServiceCIDR mặc định là thao tác một chiều,
có downtime, và sẽ phá trạng thái cluster mà các giai đoạn sau dựa vào.

Đọc ngay sau bài [393](393-extend-service-ip-ranges-vi.md) và luôn giữ ranh giới giữa hai bài:
393 **thêm** dải cho cluster đang chạy — động, không đụng kube-apiserver, hoàn tác được; bài này
**thay** dải mặc định — phải sửa cờ apiserver, cấu hình lại các thành phần mạng và đánh số lại
Service.

**Phải hiểu ở lần đọc này:**

- Bốn nhóm ở mục *Các nhóm tình huống cấu hình lại ServiceCIDR*, và đâu là ranh giới: **chỉ nhóm
  đầu — mở rộng ServiceCIDR hiện có — làm được động** bằng bài
  [393](393-extend-service-ip-ranges-vi.md); ba nhóm còn lại đều đòi cập nhật cấu hình
  kube-apiserver và chỉnh nhiều thành phần khác của cluster.
- Vai trò của cờ `--service-cluster-ip-range`: nó quyết định các **họ địa chỉ IP** khả dụng cho
  ClusterIP. ServiceCIDR `kubernetes` do **instance kube-apiserver khởi động đầu tiên** tạo ra từ
  cờ này, nên **mọi instance apiserver phải được cấu hình cùng giá trị** và giá trị đó phải khớp
  với đối tượng ServiceCIDR mặc định.
- Bốn bước ở mục *Các thao tác thủ công để thay thế ServiceCIDR mặc định*, theo đúng thứ tự: sửa
  cờ apiserver → cấu hình lại các thành phần mạng (kube-proxy, network plugin, DNS, service mesh)
  → xử lý Service có IP thuộc CIDR cũ → **xóa và tạo lại Service `kubernetes.default`**.
- Cái giá phải trả, nêu ngay trong bước 3: các Service không nằm trong dải mới phải **tạo lại**,
  nghĩa là **có thời gian ngừng hoạt động và được gán IP mới**. Đây không phải thao tác chạy tùy
  hứng trên cluster đang phục vụ.
- Mẹo của trình tự ở mục *Các bước cấu hình lại minh họa*: dựng một **ServiceCIDR tạm** làm đích
  trung gian, rồi **đánh dấu ServiceCIDR mặc định để xóa** — nó ở trạng thái chờ vì còn finalizer,
  nhưng tác dụng tức thì là **chặn mọi lần cấp phát mới từ dải cũ**.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Mục *Trước khi bạn bắt đầu* — minikube và các playground | lộ trình không dùng minikube hay cluster dùng chung | cluster VM ba node của [Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md) là môi trường thực hành duy nhất |
| Hai nhóm chuyển đổi single-stack ↔ dual-stack | cluster lab chạy thuần IPv4 nên không có họ địa chỉ thứ cấp để thêm hay bớt | bài [395](395-validate-dual-stack-vi.md) — bài 14/14; khái niệm nền đã đọc ở bài [85](85-dual-stack-vi.md) |
| Quy trình cụ thể để cấu hình lại kube-proxy, network plugin, DNS cho họ địa chỉ mới | mỗi thành phần một quy trình riêng, phụ thuộc plugin đang chạy | CNI của cluster lab đã chốt ở [Lab 5b](labs/LAB-5B-NETWORKPOLICY-INGRESS-VA-CNI.md); phần DNS ở bài [204](204-dns-custom-nameservers-vi.md) của chính giai đoạn 21 |
| Mục *Tiếp theo* dẫn sang bài [157](157-networking-vi.md) và [85](85-dual-stack-vi.md) | cả hai đã đọc ở mạch chính | bài [157](157-networking-vi.md) và [85](85-dual-stack-vi.md), đọc lại khi cần ôn |

---

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.33 [stable]` (được bật mặc định)

Tài liệu này trình bày cách cấu hình lại (các) dải Service IP mặc định đã được gán cho một
cluster.

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao
tiếp với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node
không đóng vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster
bằng [minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong
các sân chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

Kubernetes server của bạn phải ở phiên bản v1.33 hoặc mới hơn. Để kiểm tra phiên bản, hãy
chạy `kubectl version`.

## Cấu hình lại ServiceCIDR mặc định của Kubernetes (Kubernetes Default ServiceCIDR Reconfiguration)

Tài liệu này giải thích cách quản lý dải địa chỉ IP dành cho Service bên trong một cluster
Kubernetes, thứ cũng ảnh hưởng tới các họ địa chỉ IP (IP family) mà cluster hỗ trợ cho
Service.

Các họ địa chỉ IP khả dụng cho ClusterIP của Service được quyết định bởi cờ (flag)
`--service-cluster-ip-range` của kube-apiserver. Để hiểu rõ hơn về việc cấp phát địa chỉ IP
cho Service, hãy tham khảo tài liệu
[Theo dõi việc cấp phát địa chỉ IP cho Service](https://kubernetes.io/docs/reference/networking/virtual-ips/#ip-address-objects).

Kể từ Kubernetes 1.33, các họ địa chỉ IP dành cho Service được cấu hình cho cluster sẽ được
phản ánh qua đối tượng ServiceCIDR có tên là `kubernetes`. Đối tượng ServiceCIDR `kubernetes`
được tạo bởi instance kube-apiserver khởi động đầu tiên, dựa trên cờ
`--service-cluster-ip-range` mà instance đó được cấu hình. Để đảm bảo hành vi của cluster là
nhất quán, tất cả các instance kube-apiserver đều phải được cấu hình với cùng những giá trị
`--service-cluster-ip-range`, và các giá trị này phải khớp với đối tượng ServiceCIDR mặc định
`kubernetes`.

### Các nhóm tình huống cấu hình lại ServiceCIDR (Kubernetes ServiceCIDR Reconfiguration Categories)

Việc cấu hình lại ServiceCIDR thường rơi vào một trong các nhóm sau:

* **Mở rộng các ServiceCIDR hiện có:** Việc này có thể được thực hiện một cách động bằng cách
  thêm các đối tượng ServiceCIDR mới mà không cần cấu hình lại kube-apiserver. Vui lòng tham
  khảo tài liệu riêng về
  [Mở rộng dải IP của Service](393-extend-service-ip-ranges-vi.md).

* **Chuyển đổi từ single-stack sang dual-stack nhưng giữ nguyên ServiceCIDR chính:** Việc này
  bao gồm đưa thêm một họ địa chỉ IP thứ cấp (thêm IPv6 vào cluster chỉ có IPv4, hoặc thêm
  IPv4 vào cluster chỉ có IPv6) trong khi vẫn giữ họ địa chỉ IP ban đầu làm họ chính. Điều đó
  đòi hỏi phải cập nhật cấu hình của kube-apiserver và chỉnh sửa tương ứng nhiều thành phần
  khác nhau của cluster — những thành phần cần xử lý họ địa chỉ IP bổ sung này. Các thành phần
  đó bao gồm, nhưng không giới hạn ở, kube-proxy, CNI hoặc network plugin, các bản triển khai
  service mesh, và các dịch vụ DNS.

* **Chuyển đổi từ dual-stack sang single-stack nhưng giữ nguyên ServiceCIDR chính:** Việc này
  bao gồm loại bỏ họ địa chỉ IP thứ cấp khỏi một cluster dual-stack, quay về một họ địa chỉ IP
  duy nhất trong khi vẫn giữ lại họ địa chỉ IP chính ban đầu. Ngoài việc cấu hình lại các
  thành phần cho khớp với họ địa chỉ IP mới, bạn có thể còn phải xử lý các Service vốn được
  cấu hình một cách tường minh để dùng họ địa chỉ IP vừa bị loại bỏ.

* **Bất kỳ thay đổi nào dẫn tới việc đổi ServiceCIDR chính:** Thay thế hoàn toàn ServiceCIDR
  mặc định là một thao tác phức tạp. Nếu ServiceCIDR mới không chồng lấn (overlap) với
  ServiceCIDR hiện có, việc đó sẽ đòi hỏi phải
  [đánh số lại toàn bộ các Service hiện có và thay đổi Service `kubernetes.default`](#illustrative-reconfiguration-steps).
  Trường hợp họ địa chỉ IP chính cũng thay đổi thì còn phức tạp hơn nữa, và có thể đòi hỏi
  phải thay đổi nhiều thành phần của cluster (kubelet, các network plugin, v.v.) cho khớp với
  họ địa chỉ IP chính mới.

### Các thao tác thủ công để thay thế ServiceCIDR mặc định (Manual Operations for Replacing the Default ServiceCIDR)

Việc cấu hình lại ServiceCIDR mặc định đòi hỏi những bước thủ công do người vận hành cluster,
quản trị viên, hoặc phần mềm quản lý vòng đời cluster thực hiện. Các bước đó thường bao gồm:

1. **Cập nhật** cấu hình của kube-apiserver: Sửa cờ `--service-cluster-ip-range` sang (các)
   dải IP mới.
1. **Cấu hình lại** các thành phần mạng: Đây là bước then chốt và quy trình cụ thể phụ thuộc
   vào các thành phần mạng khác nhau đang được sử dụng. Việc này có thể bao gồm cập nhật các
   file cấu hình, khởi động lại các agent pod, hoặc cập nhật các thành phần để chúng quản lý
   được (các) ServiceCIDR mới cùng cấu hình họ địa chỉ IP mong muốn cho Pod. Các thành phần
   điển hình có thể là bản triển khai (implementation) của Kubernetes Service, chẳng hạn
   kube-proxy, và networking plugin đã được cấu hình, cũng như có thể là các thành phần mạng
   khác như service mesh controller và DNS server, nhằm đảm bảo chúng xử lý lưu lượng đúng
   cách và thực hiện được service discovery với cấu hình họ địa chỉ IP mới.
1. **Quản lý các Service hiện có:** Các Service có IP thuộc CIDR cũ cần được xử lý nếu chúng
   không nằm trong các dải mới được cấu hình. Các lựa chọn bao gồm tạo lại (dẫn tới thời gian
   ngừng hoạt động và được gán IP mới) hoặc các chiến lược cấu hình lại phức tạp hơn.
1. **Tạo lại các Service nội bộ của Kubernetes:** Service `kubernetes.default` phải bị xóa và
   tạo lại để lấy được một địa chỉ IP từ ServiceCIDR mới, nếu họ địa chỉ IP chính bị thay đổi
   hoặc bị thay thế bằng một mạng khác.

### Các bước cấu hình lại minh họa (Illustrative Reconfiguration Steps) {#illustrative-reconfiguration-steps}

Các bước sau mô tả một quy trình cấu hình lại có kiểm soát, tập trung vào việc thay thế hoàn
toàn ServiceCIDR mặc định và tạo lại Service `kubernetes.default`:

1. Khởi động kube-apiserver với giá trị `--service-cluster-ip-range` ban đầu.
1. Tạo các Service ban đầu, chúng sẽ lấy IP từ dải này.
1. Đưa vào một ServiceCIDR mới để làm đích tạm thời cho việc cấu hình lại.
1. Đánh dấu ServiceCIDR mặc định `kubernetes` để xóa (nó sẽ ở trạng thái chờ vì vẫn còn các IP
   đang dùng và còn finalizer). Điều này ngăn việc cấp phát mới từ dải cũ.
1. Tạo lại các Service hiện có. Lúc này chúng sẽ được cấp phát IP từ ServiceCIDR mới, tạm thời.
1. Khởi động lại kube-apiserver với (các) ServiceCIDR mới đã được cấu hình và tắt instance cũ.
1. Xóa Service `kubernetes.default`. kube-apiserver mới sẽ tạo lại Service này bên trong
   ServiceCIDR mới.

## Tiếp theo (What's next)

* [Các khái niệm về mạng trong Kubernetes](157-networking-vi.md)
* [Dual-stack Service trong Kubernetes](85-dual-stack-vi.md)
* [Mở rộng dải IP của Service trong Kubernetes](393-extend-service-ip-ranges-vi.md)

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 21:

1. Bài chia việc cấu hình lại ServiceCIDR thành bốn nhóm. Nhóm nào làm được **động**, không phải
   cấu hình lại kube-apiserver? Ba nhóm còn lại đòi thêm những gì?
2. Trong trình tự minh họa, vì sao phải đưa vào một ServiceCIDR **tạm** trước, rồi mới đánh dấu
   ServiceCIDR mặc định để xóa? Việc đánh dấu đó đạt được điều gì ngay lập tức, dù object chưa
   biến mất?
3. **Câu bẫy.** Bạn muốn đổi Service CIDR của cluster lab từ `10.96.0.0/12` sang `172.30.0.0/16`.
   Sửa cờ `--service-cluster-ip-range` trong manifest kube-apiserver trên `lab-k8s-master` rồi để
   nó khởi động lại — như vậy đã xong chưa? Nếu chưa thì còn thiếu những gì?
4. Service `kubernetes.default` phải được xử lý ra sao trong quy trình này, và trong tình huống
   nào thì việc đó là bắt buộc?
5. Cluster lab hiện chỉ có một control plane. Nếu bạn dựng HA nhiều kube-apiserver theo
   [Lab 8b](labs/LAB-8B-HA-VOI-STACKED-ETCD.md) hoặc
   [Lab 8c](labs/LAB-8C-HA-VOI-EXTERNAL-ETCD.md), bài đòi hỏi gì về cờ
   `--service-cluster-ip-range` trên các instance, và ai là bên tạo ra đối tượng ServiceCIDR
   `kubernetes`?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Chỉ nhóm **"mở rộng các ServiceCIDR hiện có"** — làm động bằng cách **thêm đối tượng
   ServiceCIDR mới**, đúng quy trình của bài [393](393-extend-service-ip-ranges-vi.md). Ba nhóm
   còn lại (single-stack → dual-stack, dual-stack → single-stack, và đổi ServiceCIDR chính) đều
   đòi **cập nhật cấu hình kube-apiserver** cộng với **chỉnh các thành phần cluster khác** —
   kube-proxy, CNI hoặc network plugin, service mesh, DNS.
2. ServiceCIDR tạm là **chỗ để các Service hiện có được cấp IP mới** trong lúc dải cũ đang bị rút
   đi. Còn việc đánh dấu ServiceCIDR mặc định để xóa, dù nó chỉ chuyển sang trạng thái chờ vì còn
   IP đang dùng và còn finalizer, **có tác dụng ngay là chặn mọi lần cấp phát mới từ dải cũ** —
   nếu không, vừa dọn xong lại có Service mới nhận IP cũ.
3. **Chưa xong.** Sửa cờ mới là **bước 1/4**. Còn phải: **cấu hình lại các thành phần mạng** để
   chúng biết dải mới (kube-proxy, network plugin, DNS); **xử lý các Service có IP thuộc dải cũ**
   — tạo lại chúng, chấp nhận **downtime và IP mới**; và **xóa rồi tạo lại Service
   `kubernetes.default`** để nó lấy địa chỉ trong dải mới. Bỏ qua bất kỳ bước nào là để cluster ở
   trạng thái không nhất quán.
4. Nó phải **bị xóa và được tạo lại** để lấy một địa chỉ từ ServiceCIDR mới. Bắt buộc khi **họ địa
   chỉ IP chính bị thay đổi hoặc bị thay bằng một mạng khác** — vì lúc đó IP cũ của nó không còn
   thuộc dải hợp lệ nào. kube-apiserver mới sẽ tự tạo lại Service này.
5. **Tất cả các instance kube-apiserver phải được cấu hình cùng những giá trị
   `--service-cluster-ip-range`**, và giá trị đó phải **khớp với đối tượng ServiceCIDR mặc định
   `kubernetes`** — nếu lệch, hành vi cluster không còn nhất quán. Đối tượng ServiceCIDR
   `kubernetes` do **instance kube-apiserver khởi động đầu tiên** tạo ra, dựa trên cờ mà chính nó
   được cấu hình.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
