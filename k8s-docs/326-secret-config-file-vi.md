# Quản lý Secret bằng file cấu hình (Managing Secrets using Configuration File)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/configmap-secret/managing-secret-using-config-file/>
>
> Tạo các đối tượng Secret bằng file cấu hình tài nguyên (resource configuration file).

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 3 — Pod và cấu hình](00-ALO-TRINH-ADMIN.md#giai-đoạn-3--pod-và-cấu-hình)
→ nhóm [3b. Cấu hình ứng dụng](00-ALO-TRINH-ADMIN.md#3b-cấu-hình-ứng-dụng-configmap-secret-và-dữ-liệu-cho-pod),
bài 4/12 · Kiểm chứng ở [Lab 3b](labs/LAB-3B-CAU-HINH-UNG-DUNG.md) phần B6.2 — `data`, `stringData`
và ai thắng khi trùng key.

Đây là bài **thứ nhất trong ba bài** làm cùng một việc bằng ba đường khác nhau: 326 (file cấu
hình), [327](327-secret-kubectl-vi.md) (`kubectl`), [328](328-secret-kustomize-vi.md) (Kustomize).
Đường của bài này là **manifest**: bạn viết YAML rồi `kubectl apply -f`. Hợp khi Secret cần nằm
cùng chỗ với các manifest khác và được apply lại nhiều lần; đổi lại bạn phải tự lo phần base64
(hoặc dùng `stringData`), và chính file manifest trở thành thứ phải bảo vệ.

**Phải hiểu ở lần đọc này:**

- Object `Secret` có **hai map**: `data` nhận giá trị **đã mã hóa base64**, còn `stringData` nhận
  chuỗi thuần và được mã hóa giúp bạn khi Secret được tạo hoặc cập nhật — mục *Tạo Secret*. Key của
  cả hai chỉ được gồm chữ, số, `-`, `_`, `.`.
- Chuỗi base64 **không được chứa ký tự xuống dòng**: bài bắt dùng `echo -n ... | base64`, và trên
  Linux khuyên thêm `-w 0` hoặc `base64 | tr -d '\n'` — ghi chú nằm ngay ở bước 1 của mục *Tạo
  Secret*. Đây là nguyên nhân kinh điển của Secret sai một cách âm thầm.
- Khi một key có mặt ở **cả `data` lẫn `stringData`** thì **`stringData` thắng** — mục *Chỉ định cả
  `data` lẫn `stringData`*: ví dụ trong bài cho ra `username: YWRtaW5pc3RyYXRvcg==`, giải ra
  `administrator`.
- Đọc lại Secret **luôn thấy giá trị đã base64 trong `data`**, không bao giờ thấy `stringData` —
  mục *Chỉ định dữ liệu chưa mã hóa khi tạo Secret* nói rõ: lệnh trả về các giá trị đã mã hóa, chứ
  không phải plaintext bạn đã cung cấp.
- Sửa Secret = sửa `data`/`stringData` trong manifest rồi `kubectl apply` lại; `kubectl` lấy object
  hiện có, lên kế hoạch thay đổi rồi gửi lên control plane — mục *Sửa một Secret*. Secret đã đặt
  `immutable` thì không sửa được.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Ghi chú lặp ba lần *`stringData` không hoạt động tốt với server-side apply*, và cờ `kubectl apply --server-side` | ở giai đoạn này bạn apply theo kiểu client-side mặc định | bài [61](61-management-vi.md), [giai đoạn 4](00-ALO-TRINH-ADMIN.md#giai-đoạn-4--workload-controller) — nơi lộ trình bàn `kubectl apply`, và cũng xếp server-side apply vào phần chưa cần |
| Link *bất biến (immutable)* ở mục *Sửa một Secret* | là một tính chất của Secret chứ không phải của cách tạo | bài [109](109-secret-vi.md) đã đọc ở đầu nhóm 3b |
| Ràng buộc tên Secret phải là *tên miền con DNS hợp lệ* | đã học rồi, ở đây chỉ là nhắc lại | bài [17](17-names-vi.md), [giai đoạn 1](00-ALO-TRINH-ADMIN.md#giai-đoạn-1--mô-hình-kubernetes) |

---

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao tiếp
với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node không đóng
vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster bằng
[minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong các sân
chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

## Tạo Secret (Create the Secret) {#create-the-config-file}

Bạn có thể định nghĩa đối tượng `Secret` trong một manifest trước, ở định dạng JSON hoặc YAML,
rồi sau đó tạo đối tượng đó. Tài nguyên
[Secret](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.36/#secret-v1-core)
chứa hai map: `data` và `stringData`.
Trường `data` được dùng để lưu dữ liệu tùy ý, mã hóa (encode) bằng base64. Trường
`stringData` được cung cấp cho thuận tiện, và nó cho phép bạn cung cấp
cùng dữ liệu đó dưới dạng chuỗi chưa mã hóa.
Các khóa (key) của `data` và `stringData` phải chỉ gồm các ký tự chữ và số,
`-`, `_` hoặc `.`.

Ví dụ sau lưu hai chuỗi vào một Secret bằng trường `data`.

1. Chuyển các chuỗi sang base64:

   ```shell
   echo -n 'admin' | base64
   echo -n '1f2d1e2e67df' | base64
   ```

   > **Ghi chú:** Các giá trị JSON và YAML đã tuần tự hóa (serialized) của dữ liệu Secret được
   > mã hóa dưới dạng chuỗi base64. Ký tự xuống dòng không hợp lệ bên trong các chuỗi này và
   > phải được loại bỏ. Khi dùng tiện ích `base64` trên Darwin/macOS, người dùng nên tránh dùng
   > tùy chọn `-b` để ngắt các dòng dài. Ngược lại, người dùng Linux *nên* thêm tùy chọn `-w 0`
   > vào các lệnh `base64`, hoặc dùng pipeline `base64 | tr -d '\n'` nếu tùy chọn `-w` không có
   > sẵn.

   Output tương tự như sau:

   ```
   YWRtaW4=
   MWYyZDFlMmU2N2Rm
   ```

1. Tạo manifest:

   ```yaml
   apiVersion: v1
   kind: Secret
   metadata:
     name: mysecret
   type: Opaque
   data:
     username: YWRtaW4=
     password: MWYyZDFlMmU2N2Rm
   ```

   Lưu ý rằng tên của một đối tượng Secret phải là một
   [tên miền con DNS hợp lệ (DNS subdomain name)](17-names-vi.md#dns-subdomain-names).

1. Tạo Secret bằng [`kubectl apply`](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#apply):

   ```shell
   kubectl apply -f ./secret.yaml
   ```

   Output tương tự như sau:

   ```
   secret/mysecret created
   ```

Để xác minh rằng Secret đã được tạo và để giải mã (decode) dữ liệu của Secret, hãy tham khảo
[Quản lý Secret bằng kubectl](327-secret-kubectl-vi.md#verify-the-secret).

### Chỉ định dữ liệu chưa mã hóa khi tạo Secret (Specify unencoded data when creating a Secret)

Trong một số tình huống nhất định, bạn có thể muốn dùng trường `stringData` thay thế. Trường
này cho phép bạn đưa một chuỗi chưa mã hóa base64 trực tiếp vào Secret,
và chuỗi đó sẽ được mã hóa giúp bạn khi Secret được tạo hoặc cập nhật.

Một ví dụ thực tế cho việc này là khi bạn triển khai một ứng dụng
dùng Secret để lưu một file cấu hình, và bạn muốn điền một số
phần của file cấu hình đó trong quá trình triển khai của mình.

Ví dụ, nếu ứng dụng của bạn dùng file cấu hình sau:

```yaml
apiUrl: "https://my.api.com/api/v1"
username: "<user>"
password: "<password>"
```

Bạn có thể lưu nó vào một Secret bằng định nghĩa sau:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
type: Opaque
stringData:
  config.yaml: |
    apiUrl: "https://my.api.com/api/v1"
    username: <user>
    password: <password>
```

> **Ghi chú:** Trường `stringData` của Secret không hoạt động tốt với server-side apply.

Khi bạn truy xuất dữ liệu của Secret, lệnh trả về các giá trị đã mã hóa,
chứ không phải các giá trị thuần văn bản (plaintext) mà bạn đã cung cấp trong `stringData`.

Ví dụ, nếu bạn chạy lệnh sau:

```shell
kubectl get secret mysecret -o yaml
```

Output tương tự như sau:

```yaml
apiVersion: v1
data:
  config.yaml: YXBpVXJsOiAiaHR0cHM6Ly9teS5hcGkuY29tL2FwaS92MSIKdXNlcm5hbWU6IHt7dXNlcm5hbWV9fQpwYXNzd29yZDoge3twYXNzd29yZH19
kind: Secret
metadata:
  creationTimestamp: 2018-11-15T20:40:59Z
  name: mysecret
  namespace: default
  resourceVersion: "7225"
  uid: c280ad2e-e916-11e8-98f2-025000000001
type: Opaque
```

### Chỉ định cả `data` lẫn `stringData` (Specify both `data` and `stringData`)

Nếu bạn chỉ định một trường trong cả `data` lẫn `stringData`, giá trị từ `stringData` sẽ được dùng.

Ví dụ, nếu bạn định nghĩa Secret sau:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
type: Opaque
data:
  username: YWRtaW4=
stringData:
  username: administrator
```

> **Ghi chú:** Trường `stringData` của Secret không hoạt động tốt với server-side apply.

Đối tượng `Secret` được tạo ra như sau:

```yaml
apiVersion: v1
data:
  username: YWRtaW5pc3RyYXRvcg==
kind: Secret
metadata:
  creationTimestamp: 2018-11-15T20:46:46Z
  name: mysecret
  namespace: default
  resourceVersion: "7579"
  uid: 91460ecb-e917-11e8-98f2-025000000001
type: Opaque
```

`YWRtaW5pc3RyYXRvcg==` giải mã ra thành `administrator`.

## Sửa một Secret (Edit a Secret) {#edit-secret}

Để sửa dữ liệu trong Secret mà bạn đã tạo bằng manifest, hãy chỉnh sửa trường `data`
hoặc `stringData` trong manifest của bạn và apply file đó vào
cluster. Bạn có thể sửa một đối tượng `Secret` hiện có, trừ khi nó là
[bất biến (immutable)](109-secret-vi.md#secret-immutable).

Ví dụ, nếu bạn muốn đổi mật khẩu ở ví dụ trước thành
`birdsarentreal`, hãy làm như sau:

1. Mã hóa chuỗi mật khẩu mới:

   ```shell
   echo -n 'birdsarentreal' | base64
   ```

   Output tương tự như sau:

   ```
   YmlyZHNhcmVudHJlYWw=
   ```

1. Cập nhật trường `data` với chuỗi mật khẩu mới của bạn:

   ```yaml
   apiVersion: v1
   kind: Secret
   metadata:
     name: mysecret
   type: Opaque
   data:
     username: YWRtaW4=
     password: YmlyZHNhcmVudHJlYWw=
   ```

1. Apply manifest vào cluster của bạn:

   ```shell
   kubectl apply -f ./secret.yaml
   ```

   Output tương tự như sau:

   ```
   secret/mysecret configured
   ```

Kubernetes cập nhật đối tượng `Secret` hiện có. Cụ thể, công cụ `kubectl`
nhận thấy đã có một đối tượng `Secret` hiện hữu với cùng tên. `kubectl`
lấy về đối tượng hiện có, lên kế hoạch các thay đổi cho nó, và gửi đối tượng
`Secret` đã thay đổi lên control plane của cluster của bạn.

Nếu bạn chỉ định `kubectl apply --server-side` thay thế, `kubectl` sẽ dùng
[Server Side Apply](https://kubernetes.io/docs/reference/using-api/server-side-apply/).

## Dọn dẹp (Clean up)

Để xóa Secret bạn đã tạo:

```shell
kubectl delete secret mysecret
```

## Tiếp theo (What's next)

- Đọc thêm về [khái niệm Secret](109-secret-vi.md)
- Tìm hiểu cách [quản lý Secret bằng kubectl](327-secret-kubectl-vi.md)
- Tìm hiểu cách [quản lý Secret bằng kustomize](328-secret-kustomize-vi.md)

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 3:

1. `data` và `stringData` nhận đầu vào khác nhau thế nào, và sau khi Secret được tạo thì
   `kubectl get secret -o yaml` hiển thị trường nào?
2. **Câu bẫy.** Manifest của bạn khai `data.username: YWRtaW4=` và `stringData.username:
   administrator`. Object tạo ra mang giá trị nào, và vì sao trực giác "`data` là trường thật nên
   `data` thắng" lại sai?
3. Trên `lab-k8s-master` bạn tạo file base64 bằng `echo 'demo-password' | base64` rồi dán vào
   `data`. Secret tạo ra sai chỗ nào, sai bao nhiêu, và lệnh đúng phải viết thế nào?
4. Bạn đã có Secret tạo từ `secret.yaml` và muốn đổi mật khẩu. Quy trình theo bài gồm những bước
   nào, và `kubectl` xử lý ra sao khi thấy đã có object cùng tên trong cluster?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **`data` bắt buộc nhận chuỗi đã mã hóa base64; `stringData` nhận chuỗi thuần** và được mã hóa
   giúp khi Secret được tạo hoặc cập nhật. Sau khi tạo, bài nói rõ: "khi bạn truy xuất dữ liệu của
   Secret, lệnh trả về các giá trị đã mã hóa, chứ không phải các giá trị thuần văn bản mà bạn đã
   cung cấp trong `stringData`" — nên bạn **luôn chỉ thấy `data`**, và `stringData` biến mất khỏi
   object.
2. Giá trị là **`administrator`** — object tạo ra mang `username: YWRtaW5pc3RyYXRvcg==`, đúng như
   ví dụ trong bài. Quy tắc là **`stringData` thắng**: "nếu bạn chỉ định một trường trong cả `data`
   lẫn `stringData`, giá trị từ `stringData` sẽ được dùng". Trực giác kia sai vì `stringData` không
   phải một trường lưu trữ song song — nó là lối vào tiện lợi, được gộp vào `data` khi Secret được
   tạo, nên nó ghi đè lên cái đã có ở đó.
3. Sai vì thiếu `-n`: `echo` thêm một **ký tự xuống dòng** vào cuối chuỗi, và ký tự đó **cũng bị mã
   hóa vào giá trị**. Mật khẩu trong Secret dài hơn mật khẩu thật đúng một ký tự, ứng dụng xác thực
   sẽ hỏng mà nhìn Secret không thấy gì bất thường. Viết đúng là **`echo -n 'demo-password' |
   base64`**; trên Linux, bài còn khuyên thêm `-w 0` hoặc `base64 | tr -d '\n'` để chắc chắn chuỗi
   base64 không bị ngắt dòng.
4. Ba bước: **mã hóa mật khẩu mới** (`echo -n 'birdsarentreal' | base64`), **cập nhật trường `data`
   trong manifest**, rồi **`kubectl apply -f ./secret.yaml`** — output là `secret/mysecret
   configured`. `kubectl` **nhận thấy đã có object cùng tên**, lấy về object hiện có, lên kế hoạch
   các thay đổi cho nó, rồi gửi object đã thay đổi lên control plane; tức là **cập nhật object cũ**
   chứ không tạo object mới. Trường hợp duy nhất không làm được là Secret đã đặt `immutable`.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
