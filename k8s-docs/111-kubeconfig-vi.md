# Tổ chức quyền truy cập cluster bằng file kubeconfig (Organizing Cluster Access Using kubeconfig Files)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](LO-TRINH-ADMIN.md).

**Vị trí:** Giai đoạn 1 → nhóm [1b](LO-TRINH-ADMIN.md#1b-làm-việc-với-object-và-kubectl),
bài 7/9 · Kiểm chứng ở Lab 1b (chưa viết, xem [bản đồ lab](labs/README.md#4-bản-đồ-lab)).

Ở [Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md) bạn đã copy `/etc/kubernetes/admin.conf` thành
`~/.kube/config` mà chưa biết bên trong có gì. Đây là bài mở file đó ra.

**Phải hiểu ở lần đọc này:**

- Kubeconfig có **ba nhóm mục**: `clusters` (địa chỉ và CA), `users` (thông tin xác thực),
  `contexts`. Một **context** buộc ba thứ lại: cluster + user + namespace.
- `current-context` quyết định lệnh không có flag sẽ đi tới đâu.
- Thứ tự tìm file: `--kubeconfig` (chỉ một file, không merge) → `KUBECONFIG` (danh sách, có
  merge) → `$HOME/.kube/config`.
- Quy tắc merge quan trọng nhất: **file đầu tiên đặt một giá trị sẽ thắng**, và file sau không
  bổ sung được vào mục đã bị chiếm.
- Đường dẫn file bên trong kubeconfig là **tương đối so với vị trí file kubeconfig**, còn
  đường dẫn trên dòng lệnh thì tương đối so với thư mục hiện tại.
- Cảnh báo bảo mật: kubeconfig lạ phải soi như soi một shell script lạ trước khi dùng.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Chuỗi sáu bước xác định cluster và user | là đặc tả để tra khi debug, không phải để nhớ | quay lại khi kubectl trỏ sai chỗ |
| Các cơ chế xác thực khác nhau (certificate, token) | chưa học authentication | giai đoạn 9 |
| `proxy-url` | trường hợp đặc thù | khi cần qua proxy |

---

Hãy dùng các file kubeconfig để tổ chức thông tin về cluster, người dùng (user), namespace và
các cơ chế xác thực (authentication). Công cụ dòng lệnh `kubectl` dùng các file kubeconfig để
tìm thông tin cần thiết nhằm chọn một cluster và giao tiếp với API server
của cluster đó.

> **Ghi chú:**
> Một file được dùng để cấu hình quyền truy cập tới các cluster được gọi là
> *file kubeconfig*. Đây là cách gọi chung cho loại file cấu hình này.
> Nó không có nghĩa là tồn tại một file có tên `kubeconfig`.

> **Cảnh báo:**
> Chỉ sử dụng file kubeconfig từ những nguồn đáng tin cậy. Việc dùng một file kubeconfig được chế tác đặc biệt có thể dẫn đến thực thi mã độc hoặc làm lộ file.
> Nếu bắt buộc phải dùng một file kubeconfig không đáng tin cậy, hãy kiểm tra nó cẩn thận trước, giống như cách bạn kiểm tra một shell script.

Theo mặc định, `kubectl` tìm một file tên là `config` trong thư mục `$HOME/.kube`.
Bạn có thể chỉ định các file kubeconfig khác bằng cách đặt biến môi trường `KUBECONFIG`
hoặc bằng cách dùng flag
[`--kubeconfig`](https://kubernetes.io/docs/reference/generated/kubectl/kubectl/).

Để xem hướng dẫn từng bước về việc tạo và chỉ định file kubeconfig, hãy xem
[Cấu hình quyền truy cập tới nhiều cluster](https://kubernetes.io/docs/tasks/access-application-cluster/configure-access-multiple-clusters).

## Hỗ trợ nhiều cluster, người dùng và cơ chế xác thực (Supporting multiple clusters, users, and authentication mechanisms)

Giả sử bạn có nhiều cluster, và người dùng cùng các thành phần của bạn xác thực
theo nhiều cách khác nhau. Ví dụ:

- Một kubelet đang chạy có thể xác thực bằng certificate.
- Một người dùng có thể xác thực bằng token.
- Quản trị viên có thể có các bộ certificate mà họ cấp cho từng người dùng riêng lẻ.

Với các file kubeconfig, bạn có thể tổ chức các cluster, người dùng và namespace của mình.
Bạn cũng có thể định nghĩa các context để chuyển đổi nhanh chóng và dễ dàng giữa
các cluster và namespace.

## Context

Một phần tử *context* trong file kubeconfig được dùng để gom nhóm các tham số truy cập
dưới một cái tên thuận tiện. Mỗi context có ba tham số: cluster, namespace và user.
Theo mặc định, công cụ dòng lệnh `kubectl` dùng các tham số từ
*context hiện tại (current context)* để giao tiếp với cluster.

Để chọn context hiện tại:
```
kubectl config use-context
```

## Biến môi trường KUBECONFIG (The KUBECONFIG environment variable)

Biến môi trường `KUBECONFIG` chứa một danh sách các file kubeconfig.
Với Linux và Mac, danh sách được phân tách bằng dấu hai chấm. Với Windows, danh sách
được phân tách bằng dấu chấm phẩy. Biến môi trường `KUBECONFIG` không
bắt buộc. Nếu biến môi trường `KUBECONFIG` không tồn tại,
`kubectl` dùng file kubeconfig mặc định, `$HOME/.kube/config`.

Nếu biến môi trường `KUBECONFIG` tồn tại, `kubectl` dùng
một cấu hình hiệu lực (effective configuration) là kết quả của việc hợp nhất (merge) các file
được liệt kê trong biến môi trường `KUBECONFIG`.

## Hợp nhất các file kubeconfig (Merging kubeconfig files)

Để xem cấu hình của bạn, hãy nhập lệnh sau:

```shell
kubectl config view
```

Như đã mô tả ở trên, kết quả xuất ra có thể đến từ một file kubeconfig duy nhất,
hoặc có thể là kết quả của việc hợp nhất nhiều file kubeconfig.

Dưới đây là các quy tắc mà `kubectl` áp dụng khi hợp nhất các file kubeconfig:

1. Nếu flag `--kubeconfig` được đặt, chỉ dùng file được chỉ định. Không hợp nhất.
   Chỉ được phép dùng flag này một lần.

   Ngược lại, nếu biến môi trường `KUBECONFIG` được đặt, dùng nó như một
   danh sách các file cần được hợp nhất.
   Hợp nhất các file được liệt kê trong biến môi trường `KUBECONFIG`
   theo các quy tắc sau:

   * Bỏ qua các tên file rỗng.
   * Báo lỗi với các file có nội dung không thể giải tuần tự hóa (deserialize) được.
   * File đầu tiên đặt một giá trị hoặc một khóa map cụ thể sẽ thắng.
   * Không bao giờ thay đổi giá trị hoặc khóa map đó.
     Ví dụ: Giữ nguyên context của file đầu tiên đặt `current-context`.
     Ví dụ: Nếu hai file cùng khai báo một `red-user`, chỉ dùng các giá trị từ `red-user` của file đầu tiên.
     Kể cả khi file thứ hai có các mục không xung đột dưới `red-user`, vẫn loại bỏ chúng.

   Để xem ví dụ về cách đặt biến môi trường `KUBECONFIG`, hãy xem
   [Đặt biến môi trường KUBECONFIG](https://kubernetes.io/docs/tasks/access-application-cluster/configure-access-multiple-clusters/#set-the-kubeconfig-environment-variable).

   Nếu không thuộc các trường hợp trên, dùng file kubeconfig mặc định, `$HOME/.kube/config`, và không hợp nhất gì cả.

1. Xác định context sẽ dùng dựa trên kết quả khớp đầu tiên trong chuỗi sau:

    1. Dùng flag dòng lệnh `--context` nếu nó tồn tại.
    1. Dùng `current-context` từ các file kubeconfig đã hợp nhất.

   Tại bước này, context rỗng vẫn được chấp nhận.

1. Xác định cluster và user. Tại thời điểm này, có thể có hoặc không có context.
   Xác định cluster và user dựa trên kết quả khớp đầu tiên trong chuỗi sau,
   chuỗi này được chạy hai lần: một lần cho user và một lần cho cluster:

   1. Dùng flag dòng lệnh nếu nó tồn tại: `--user` hoặc `--cluster`.
   1. Nếu context không rỗng, lấy user hoặc cluster từ context đó.

   Tại bước này, user và cluster có thể rỗng.

1. Xác định thông tin cluster thực sự sẽ dùng. Tại thời điểm này, có thể có hoặc
   không có thông tin cluster.
   Xây dựng từng phần của thông tin cluster dựa trên chuỗi sau; kết quả khớp đầu tiên sẽ thắng:

   1. Dùng các flag dòng lệnh nếu chúng tồn tại: `--server`, `--certificate-authority`, `--insecure-skip-tls-verify`.
   1. Nếu có bất kỳ thuộc tính thông tin cluster nào từ các file kubeconfig đã hợp nhất, dùng chúng.
   1. Nếu không có vị trí server nào, thất bại.

1. Xác định thông tin user thực sự sẽ dùng. Xây dựng thông tin user theo cùng các
   quy tắc như thông tin cluster, ngoại trừ việc chỉ cho phép một kỹ thuật
   xác thực cho mỗi user:

   1. Dùng các flag dòng lệnh nếu chúng tồn tại: `--client-certificate`, `--client-key`, `--username`, `--password`, `--token`.
   1. Dùng các trường `user` từ các file kubeconfig đã hợp nhất.
   1. Nếu có hai kỹ thuật xung đột nhau, thất bại.

1. Với bất kỳ thông tin nào vẫn còn thiếu, dùng các giá trị mặc định và có thể
   nhắc người dùng nhập thông tin xác thực.

## Tham chiếu file (File references)

Các tham chiếu file và đường dẫn trong một file kubeconfig là tương đối so với vị trí của file kubeconfig đó.
Các tham chiếu file trên dòng lệnh là tương đối so với thư mục làm việc hiện tại.
Trong `$HOME/.kube/config`, đường dẫn tương đối được lưu ở dạng tương đối, và đường dẫn tuyệt đối
được lưu ở dạng tuyệt đối.

## Proxy

Bạn có thể cấu hình `kubectl` dùng một proxy cho từng cluster bằng `proxy-url` trong file kubeconfig của bạn, như sau:

```yaml
apiVersion: v1
kind: Config

clusters:
- cluster:
    proxy-url: http://proxy.example.org:3128
    server: https://k8s.example.org/k8s/clusters/c-xxyyzz
  name: development

users:
- name: developer

contexts:
- context:
  name: development
```

## Tiếp theo (What's next)

* [Cấu hình quyền truy cập tới nhiều cluster](https://kubernetes.io/docs/tasks/access-application-cluster/configure-access-multiple-clusters/)
* [`kubectl config`](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#config)

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

1. Một context gồm mấy tham số, là những gì?
2. Ở Lab 00 bạn chạy `sudo cp /etc/kubernetes/admin.conf ~/.kube/config` rồi `chmod 600`. File
   đó chứa gì khiến quyền 600 là bắt buộc?
3. `KUBECONFIG` trỏ tới hai file, cả hai đều khai báo một user tên `red-user` với các trường
   khác nhau. Kết quả hợp nhất ra sao?
4. Bạn muốn mọi lệnh `kubectl` mặc định chạy trong namespace `lab-1b` mà không phải gõ `-n`.
   Làm thế nào, và thay đổi đó nằm ở đâu?
5. Có `--kubeconfig` **và** `KUBECONFIG` cùng lúc thì `kubectl` dùng cái nào?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Ba**: cluster, user và namespace.
2. Nó chứa **thông tin xác thực quản trị cluster** — client certificate và key của admin. Ai
   đọc được file đó thì có toàn quyền trên cluster, nên không được để người dùng khác trên máy
   đọc.
3. Chỉ các giá trị từ **file đầu tiên** khai báo `red-user` được dùng. Các mục của file thứ
   hai bị **loại bỏ hoàn toàn**, kể cả những mục không xung đột. Đây là chỗ hay gây bất ngờ khi
   ghép nhiều kubeconfig.
4. `kubectl config set-context --current --namespace=lab-1b`. Thay đổi được ghi vào **context
   hiện tại trong chính file kubeconfig**, nên nó bền qua các phiên terminal.
5. Dùng `--kubeconfig`, và **không hợp nhất gì cả**. Flag thắng, `KUBECONFIG` bị bỏ qua.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
