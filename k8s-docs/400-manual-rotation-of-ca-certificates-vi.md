# Xoay CA certificate thủ công (Manual Rotation of CA Certificates)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/tls/manual-rotation-of-ca-certificates/>
>
> Trang này hướng dẫn cách xoay (rotate) thủ công các certificate của certificate authority (CA).

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:**
[Checkpoint tiếp nối — nhánh `/docs/tasks/`](00-ALO-TRINH-ADMIN.md#phần-ii--vận-hành-cluster)
→ [Giai đoạn 18 — Vòng đời chứng chỉ](00-ALO-TRINH-ADMIN.md#giai-đoạn-18--vòng-đời-chứng-chỉ), bài 7/7 — **bài cuối và
nguy hiểm nhất nhóm** · Chỉ làm trên cluster lab, sau khi đã snapshot cả ba VM.

Đây là quy trình đụng vào **gốc tin cậy của toàn cluster**. Bài giả định control plane chạy HA
nhiều API server; cluster lab một control plane sẽ **mất khả năng phục vụ** trong lúc API server
restart — bài nói rõ điều đó. Đọc trước bài [219](219-kubeadm-certs-vi.md) và
[191](191-certificates-manual-vi.md).

**Phải hiểu ở lần đọc này:**

- **Nguyên tắc xuyên suốt: tin cả hai CA trước, bỏ CA cũ sau.** Mọi thành phần được nạp bundle
  chứa **CA cũ + CA mới**, chạy ổn định, kiểm chứng xong rồi mới gỡ CA cũ. Nhờ vậy chỉ có một
  khoảnh khắc mất kết nối chứ không đứt hẳn.
- Thứ tự bắt buộc: phân phối CA mới → `--root-ca-file` của controller manager → chờ Secret của
  ServiceAccount cập nhật → **restart Pod dùng cấu hình in-cluster** (kube-proxy, CoreDNS) →
  `--client-ca-file` của apiserver và scheduler → cập nhật kubeconfig người dùng → cuốn chiếu
  kubelet và API server.
- **Bẫy đã biết:** `--client-ca-file` và `--cluster-signing-cert-file` của controller manager
  **không nhận CA bundle**. Nếu chúng cùng trỏ tới `ca.crt` vốn đã thành bundle, cluster sẽ lỗi.
  Cách xử lý: tách CA mới ra file riêng cho hai cờ đó (kubeadm issue 1350).
- Vì sao phải **gắn annotation cho Deployment và DaemonSet**: để buộc Pod được thay thế theo kiểu
  cuốn chiếu, nhờ đó chúng nạp lại dữ liệu CA từ Secret của ServiceAccount.
- Bước dễ quên: nếu cluster dùng **bootstrap token** để join node, phải cập nhật ConfigMap
  `cluster-info` trong namespace `kube-public`, nếu không node mới sẽ không join được.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Restart aggregated API server và webhook handler | cluster lab chưa có thành phần nào loại này | [giai đoạn 28](00-ALO-TRINH-ADMIN.md#giai-đoạn-28--mở-rộng-kubernetes), sau khi dựng extension API server |
| Bước cloud-controller-manager | cluster lab là on-premise, không có thành phần này | bài [34](34-cloud-controller-vi.md) đã đọc ở giai đoạn 1c |
| Quy trình thay thế cuốn chiếu cho StatefulSet | phụ thuộc cách ứng dụng của bạn dùng StatefulSet | bài [65](65-statefulset-vi.md) và [339](339-configure-pdb-vi.md) |

---

Trang này hướng dẫn cách xoay (rotate) thủ công các certificate của certificate authority (CA).

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao tiếp
với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node không đóng
vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster bằng
[minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong các sân
chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

- Để biết thêm về xác thực (authentication) trong Kubernetes, xem
  [Xác thực (Authenticating)](https://kubernetes.io/docs/reference/access-authn-authz/authentication/).
- Để biết thêm về các thực hành tốt nhất (best practices) cho CA certificate, xem
  [CA gốc duy nhất (Single root CA)](https://kubernetes.io/docs/setup/best-practices/certificates/#single-root-ca).

## Xoay CA certificate thủ công (Rotate the CA certificates manually)

> **Thận trọng:** Hãy chắc chắn rằng bạn đã sao lưu thư mục certificate cùng với các file cấu
> hình và mọi file cần thiết khác.
>
> Cách tiếp cận này giả định control plane Kubernetes đang vận hành ở cấu hình HA (tính sẵn sàng
> cao) với nhiều API server. Nó cũng giả định API server được kết thúc một cách êm ái (graceful
> termination), để client có thể ngắt kết nối sạch sẽ khỏi một API server và kết nối lại tới một
> API server khác.
>
> Các cấu hình chỉ có một API server sẽ bị mất khả năng phục vụ (unavailability) trong lúc API
> server đang được khởi động lại.

1. Phân phối các CA certificate và private key mới (ví dụ: `ca.crt`, `ca.key`, `front-proxy-ca.crt`
   và `front-proxy-ca.key`) tới tất cả các control plane node của bạn, đặt vào thư mục certificate
   của Kubernetes.

1. Cập nhật flag `--root-ca-file` của kube-controller-manager để bao gồm cả CA cũ lẫn CA mới, sau
   đó khởi động lại kube-controller-manager.

   Mọi ServiceAccount được tạo sau thời điểm này sẽ nhận được các Secret chứa cả CA cũ lẫn CA mới.

   > **Ghi chú:** Các file được chỉ định bởi các flag `--client-ca-file` và
   > `--cluster-signing-cert-file` của kube-controller-manager không thể là CA bundle. Nếu các
   > flag này và `--root-ca-file` cùng trỏ tới một file `ca.crt`, mà file này giờ đã là một bundle
   > (chứa cả CA cũ lẫn CA mới), bạn sẽ gặp lỗi. Để khắc phục vấn đề này, bạn có thể sao chép CA
   > mới sang một file riêng và cho các flag `--client-ca-file` và `--cluster-signing-cert-file`
   > trỏ tới bản sao đó. Khi `ca.crt` không còn là một bundle nữa, bạn có thể trả các flag gây
   > vấn đề này về trỏ tới `ca.crt` và xóa bản sao đi.
   >
   > [Issue 1350](https://github.com/kubernetes/kubeadm/issues/1350) của kubeadm theo dõi lỗi
   > kube-controller-manager không chấp nhận được CA bundle.

1. Chờ controller manager cập nhật `ca.crt` trong các Secret của service account để chứa cả CA cũ
   lẫn CA mới.

   Nếu có Pod nào được khởi động trước khi CA mới được các API server sử dụng, các Pod mới đó sẽ
   nhận được bản cập nhật này và sẽ tin tưởng cả CA cũ lẫn CA mới.

1. Khởi động lại toàn bộ Pod đang dùng cấu hình bên trong cluster (in-cluster configuration) (ví
   dụ: kube-proxy, CoreDNS, v.v.) để chúng có thể dùng dữ liệu certificate authority đã được cập
   nhật từ các Secret liên kết tới ServiceAccount.

   * Hãy chắc chắn rằng CoreDNS, kube-proxy và các Pod khác dùng cấu hình in-cluster đang hoạt
     động như mong đợi.

1. Nối (append) cả CA cũ lẫn CA mới vào file được trỏ tới bởi các flag `--client-ca-file` và
   `--kubelet-certificate-authority` trong cấu hình `kube-apiserver`.

1. Nối cả CA cũ lẫn CA mới vào file được trỏ tới bởi flag `--client-ca-file` trong cấu hình
   `kube-scheduler`.

1. Cập nhật certificate cho các user account bằng cách thay thế nội dung của
   `client-certificate-data` và `client-key-data` tương ứng.

   Để biết cách tạo certificate cho từng user account riêng lẻ, xem
   [Cấu hình certificate cho user account](https://kubernetes.io/docs/setup/best-practices/certificates/#configure-certificates-for-user-accounts).

   Ngoài ra, hãy cập nhật phần `certificate-authority-data` trong các file kubeconfig, tương ứng
   bằng dữ liệu certificate authority cũ và mới đã được mã hóa Base64.

1. Cập nhật flag `--root-ca-file` của cloud-controller-manager để bao gồm cả CA cũ lẫn CA mới, sau
   đó khởi động lại cloud-controller-manager.

   > **Ghi chú:** Nếu cluster của bạn không có cloud-controller-manager, bạn có thể bỏ qua bước
   > này.

1. Thực hiện các bước dưới đây theo kiểu cuốn chiếu (rolling).

   1. Khởi động lại mọi
      [aggregated API server](180-apiserver-aggregation-vi.md) khác hoặc các webhook handler để
      chúng tin tưởng các CA certificate mới.

   1. Khởi động lại kubelet bằng cách cập nhật file được trỏ tới bởi `clientCAFile` trong cấu hình
      kubelet và `certificate-authority-data` trong `kubelet.conf` để dùng cả CA cũ lẫn CA mới
      trên tất cả các node.

      Nếu kubelet của bạn không dùng cơ chế xoay vòng client certificate (client certificate
      rotation), hãy cập nhật `client-certificate-data` và `client-key-data` trong `kubelet.conf`
      trên tất cả các node, cùng với file client certificate của kubelet thường nằm ở
      `/var/lib/kubelet/pki`.

   1. Khởi động lại các API server với các certificate (`apiserver.crt`,
      `apiserver-kubelet-client.crt` và `front-proxy-client.crt`) đã được ký bởi CA mới.
      Bạn có thể dùng lại các private key hiện có hoặc dùng private key mới.
      Nếu bạn đổi private key, hãy cập nhật chúng trong thư mục certificate của Kubernetes nữa.

      Vì các Pod trong cluster của bạn tin tưởng cả CA cũ lẫn CA mới, sẽ chỉ có một khoảnh khắc
      mất kết nối, sau đó các Kubernetes client của Pod sẽ kết nối lại tới API server mới.
      API server mới dùng certificate được ký bởi CA mới.

      * Khởi động lại kube-scheduler để nó dùng và tin tưởng các CA mới.
      * Hãy chắc chắn rằng log của các thành phần control plane không báo lỗi TLS nào.

      > **Ghi chú:** Để tạo certificate và private key cho cluster của bạn bằng công cụ dòng lệnh
      > `openssl`, xem [Certificate (`openssl`)](191-certificates-manual-vi.md#openssl).
      > Bạn cũng có thể dùng [`cfssl`](191-certificates-manual-vi.md#cfssl).

   1. Gắn annotation cho các DaemonSet và Deployment để kích hoạt việc thay thế Pod theo kiểu cuốn
      chiếu an toàn hơn.

      ```shell
      for namespace in $(kubectl get namespace -o jsonpath='{.items[*].metadata.name}'); do
          for name in $(kubectl get deployments -n $namespace -o jsonpath='{.items[*].metadata.name}'); do
              kubectl patch deployment -n ${namespace} ${name} -p '{"spec":{"template":{"metadata":{"annotations":{"ca-rotation": "1"}}}}}';
          done
          for name in $(kubectl get daemonset -n $namespace -o jsonpath='{.items[*].metadata.name}'); do
              kubectl patch daemonset -n ${namespace} ${name} -p '{"spec":{"template":{"metadata":{"annotations":{"ca-rotation": "1"}}}}}';
          done
      done
      ```

      > **Ghi chú:** Để giới hạn số lượng gián đoạn (disruption) đồng thời mà ứng dụng của bạn phải
      > chịu, xem [cấu hình pod disruption budget](339-configure-pdb-vi.md).

      Tùy vào cách bạn dùng StatefulSet, bạn cũng có thể cần thực hiện một quy trình thay thế cuốn
      chiếu tương tự.

1. Nếu cluster của bạn dùng bootstrap token để cho node tham gia (join) cluster, hãy cập nhật
   ConfigMap `cluster-info` trong namespace `kube-public` với CA mới.

   ```shell
   base64_encoded_ca="$(base64 -w0 /etc/kubernetes/pki/ca.crt)"

   kubectl get cm/cluster-info --namespace kube-public -o yaml | \
       /bin/sed "s/\(certificate-authority-data:\).*/\1 ${base64_encoded_ca}/" | \
       kubectl apply -f -
   ```

1. Kiểm chứng lại chức năng của cluster.

   1. Kiểm tra log của các thành phần control plane, cùng với log của kubelet và kube-proxy.
      Hãy chắc chắn rằng các thành phần đó không báo lỗi TLS nào; xem
      [xem log](305-debug-cluster-vi.md#xem-log-looking-at-logs) để biết thêm chi tiết.

   1. Kiểm tra log của mọi aggregated API server và của các Pod dùng cấu hình in-cluster.

1. Sau khi đã kiểm chứng thành công chức năng của cluster:

   1. Cập nhật tất cả các service account token để chúng chỉ còn chứa CA certificate mới.

      * Mọi Pod đang dùng kubeconfig in-cluster rốt cuộc đều sẽ cần được khởi động lại để nhận
        Secret mới, sao cho không còn Pod nào phụ thuộc vào CA cũ của cluster.

   1. Khởi động lại các thành phần control plane sau khi loại bỏ CA cũ khỏi các file kubeconfig và
      khỏi các file được trỏ tới bởi các flag `--client-ca-file`, `--root-ca-file` tương ứng.

   1. Trên từng node, khởi động lại kubelet sau khi loại bỏ CA cũ khỏi file được trỏ tới bởi flag
      `clientCAFile` và khỏi file kubeconfig của kubelet. Bạn nên thực hiện việc này theo kiểu
      cuốn chiếu (rolling update).

      Nếu cluster của bạn cho phép bạn thực hiện thay đổi này, bạn cũng có thể triển khai nó bằng
      cách thay thế node thay vì cấu hình lại chúng.

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 18:

1. Vì sao toàn bộ quy trình phải đi qua giai đoạn "mọi thành phần tin **cả hai** CA" thay vì thay
   thẳng CA cũ bằng CA mới?
2. **Câu bẫy.** Bạn nối CA cũ và CA mới vào `ca.crt`, rồi để cả `--root-ca-file`,
   `--client-ca-file` và `--cluster-signing-cert-file` của kube-controller-manager cùng trỏ tới
   file đó. Chuyện gì xảy ra?
3. Cluster lab của bạn chỉ có **một** control plane. Bài giả định điều gì về control plane, và hệ
   quả với cluster của bạn là gì?
4. Vì sao phải gắn annotation cho Deployment và DaemonSet ở gần cuối quy trình?
5. Nếu quên bước cập nhật ConfigMap `cluster-info` trong namespace `kube-public`, triệu chứng sẽ
   xuất hiện lúc nào?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Vì các thành phần được cập nhật **không đồng thời**. Trong lúc chuyển tiếp, sẽ có client vẫn
   giữ CA cũ nói chuyện với server đã dùng CA mới và ngược lại. Khi mọi bên tin **cả hai** CA,
   những cặp giao tiếp lệch pha đó vẫn thành công; chỉ còn một khoảnh khắc mất kết nối khi API
   server restart. Thay thẳng CA sẽ làm đứt tin cậy toàn cluster ngay lập tức.
2. **Cluster lỗi.** Hai cờ `--client-ca-file` và `--cluster-signing-cert-file` **không chấp nhận
   CA bundle**; chỉ `--root-ca-file` chấp nhận. Khi `ca.crt` đã thành bundle mà cả ba cùng trỏ
   vào đó, hai cờ kia sẽ hỏng. Cách khắc phục: chép **riêng CA mới** ra một file khác cho hai cờ
   đó, và trả chúng về `ca.crt` sau khi file này không còn là bundle. Đây là lỗi đã biết, theo dõi
   ở kubeadm issue 1350 — dễ mắc vì trực giác bảo "cứ trỏ hết vào một file cho gọn".
3. Bài giả định control plane chạy **HA với nhiều API server** và API server được **kết thúc êm
   ái**, để client ngắt khỏi một API server rồi kết nối sang API server khác. Cluster lab một
   control plane **không có** điều đó, nên sẽ **mất khả năng phục vụ** trong suốt thời gian API
   server khởi động lại — bài nói thẳng điều này.
4. Để **buộc Pod bị thay thế theo kiểu cuốn chiếu**. Pod chỉ nạp dữ liệu CA mới từ Secret của
   ServiceAccount khi nó được tạo lại; annotation làm thay đổi pod template nên kích hoạt việc
   thay thế một cách an toàn và có kiểm soát, thay vì xóa Pod hàng loạt.
5. Triệu chứng chỉ xuất hiện khi bạn **join một node mới** bằng bootstrap token — node sẽ không
   tin được API server vì `cluster-info` vẫn chứa CA cũ. Cluster đang chạy vẫn bình thường, nên
   lỗi này thường bị phát hiện rất muộn.

</details>

Đây là bài cuối của [Giai đoạn 18 — Vòng đời chứng chỉ](00-ALO-TRINH-ADMIN.md#giai-đoạn-18--vòng-đời-chứng-chỉ). Trả lời trôi cả năm câu thì chuyển sang
[Giai đoạn 19 — etcd, backup và khôi phục thảm họa](00-ALO-TRINH-ADMIN.md#giai-đoạn-19--etcd-backup-và-khôi-phục-thảm-họa).
