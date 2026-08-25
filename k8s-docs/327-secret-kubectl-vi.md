# Quản lý Secret bằng kubectl (Managing Secrets using kubectl)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/configmap-secret/managing-secret-using-kubectl/>
>
> Tạo các đối tượng Secret bằng công cụ dòng lệnh kubectl.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 3 — Pod và cấu hình](00-ALO-TRINH-ADMIN.md#giai-đoạn-3--pod-và-cấu-hình)
→ nhóm [3b. Cấu hình ứng dụng](00-ALO-TRINH-ADMIN.md#3b-cấu-hình-ứng-dụng-configmap-secret-và-dữ-liệu-cho-pod),
bài 5/12 · Kiểm chứng ở [Lab 3b](labs/LAB-3B-CAU-HINH-UNG-DUNG.md) phần B6.1 (`--from-literal` và
`--from-file`) và phần B7.1 (`describe` giấu giá trị, `jsonpath` thì không).

Đây là bài **thứ hai trong ba bài** làm cùng một việc: 326 (file cấu hình),
[327](327-secret-kubectl-vi.md) (`kubectl`), [328](328-secret-kustomize-vi.md) (Kustomize). Đường
của bài này là **dòng lệnh**: nhanh nhất, hợp lúc đang vận hành và chỉ cần một Secret dùng ngay.
Đổi lại nó không để lại manifest nào để apply lại, và bài cảnh báo hai lần rằng giá trị gõ ra dòng
lệnh sẽ nằm lại trong lịch sử shell.

**Phải hiểu ở lần đọc này:**

- Hai cách nạp dữ liệu, mục *Tạo một Secret*: `--from-literal` gõ thẳng giá trị — phải bọc **nháy
  đơn** để shell không diễn giải `$`, `\`, `*`, `=`, `!`; còn `--from-file` đọc từ file — **không
  cần escape gì**, key mặc định là tên file và đổi được bằng `--from-file=<key>=<nguồn>`.
- Khi ghi file nguồn phải dùng `echo -n`: thiếu `-n` thì ký tự xuống dòng thừa **cũng bị mã hóa vào
  giá trị** — lý do bài giải thích ngay trong mục *Dùng file nguồn*.
- `kubectl get` và `kubectl describe` **cố ý không in nội dung Secret**; `describe` chỉ hiện số byte
  của mỗi key (`password: 12 bytes`). Mục *Xác minh Secret* nói rõ mục đích: tránh lộ Secret một
  cách vô tình và tránh để nó nằm lại trong log của terminal.
- Muốn thấy giá trị thật thì phải tự lấy: `kubectl get secret <tên> -o jsonpath='{.data}'` rồi
  `base64 --decode`. Khối *Thận trọng* ở mục *Giải mã Secret* bảo **ghép hai lệnh thành một
  pipeline** (`-o jsonpath='{.data.password}' | base64 --decode`) thay vì dán chuỗi base64 ra dòng
  lệnh, vì chuỗi đó sẽ nằm lại trong lịch sử shell cho bất kỳ ai đọc được máy bạn.
- `kubectl edit secrets <tên>` mở editor trên object sống và bạn sửa **giá trị đã base64** trong
  `data` — không phải plaintext. Secret đã đặt `immutable` thì không sửa được.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Ghi chú *`stringData` không hoạt động tốt với server-side apply* đặt trong mục *Dùng dữ liệu thô* | lệnh `kubectl create secret` ở đây không dùng `stringData`; ghi chú này thuộc về đường manifest | trường `stringData` ở bài [326](326-secret-config-file-vi.md) cùng nhóm; server-side apply thì xem bài [61](61-management-vi.md), [giai đoạn 4](00-ALO-TRINH-ADMIN.md#giai-đoạn-4--workload-controller) |
| Link *bất biến (immutable)* ở mục *Sửa một Secret* | là tính chất của Secret, không phải của cách tạo | bài [109](109-secret-vi.md) đã đọc ở đầu nhóm 3b |

---

Trang này chỉ cho bạn cách tạo, sửa, quản lý và xóa các
Secret của Kubernetes bằng công cụ dòng lệnh `kubectl`.

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao tiếp
với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node không đóng
vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster bằng
[minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong các sân
chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

## Tạo một Secret (Create a Secret)

Một đối tượng `Secret` lưu trữ dữ liệu nhạy cảm, chẳng hạn thông tin xác thực (credentials)
mà các Pod dùng để truy cập các dịch vụ. Ví dụ, bạn có thể cần một Secret để lưu
tên người dùng và mật khẩu cần thiết để truy cập một cơ sở dữ liệu.

Bạn có thể tạo Secret bằng cách truyền dữ liệu thô (raw data) ngay trong lệnh, hoặc bằng cách
lưu thông tin xác thực vào các file rồi truyền các file đó trong lệnh. Các lệnh sau
tạo một Secret lưu tên người dùng `admin` và mật khẩu `S!B\*d$zDsb=`.

### Dùng dữ liệu thô (Use raw data)

Chạy lệnh sau:

```shell
kubectl create secret generic db-user-pass \
    --from-literal=username=admin \
    --from-literal=password='S!B\*d$zDsb='
```

Bạn phải dùng dấu nháy đơn `''` để escape các ký tự đặc biệt như `$`, `\`,
`*`, `=` và `!` trong chuỗi của bạn. Nếu không, shell của bạn sẽ diễn giải các
ký tự này.

> **Ghi chú:** Trường `stringData` của Secret không hoạt động tốt với server-side apply.

### Dùng file nguồn (Use source files)

1. Lưu thông tin xác thực vào các file:

   ```shell
   echo -n 'admin' > ./username.txt
   echo -n 'S!B\*d$zDsb=' > ./password.txt
   ```

   Cờ `-n` đảm bảo các file được tạo ra không có thêm một ký tự xuống dòng
   thừa ở cuối văn bản. Điều này quan trọng vì khi `kubectl`
   đọc một file và mã hóa (encode) nội dung thành chuỗi base64, ký tự xuống dòng
   thừa cũng sẽ bị mã hóa theo. Bạn không cần escape các ký tự đặc biệt
   trong các chuỗi mà bạn đưa vào file.

1. Truyền đường dẫn file trong lệnh `kubectl`:

   ```shell
   kubectl create secret generic db-user-pass \
       --from-file=./username.txt \
       --from-file=./password.txt
   ```

   Tên khóa (key) mặc định là tên file. Bạn có thể tùy chọn đặt tên khóa
   bằng `--from-file=[key=]source`. Ví dụ:

   ```shell
   kubectl create secret generic db-user-pass \
       --from-file=username=./username.txt \
       --from-file=password=./password.txt
   ```

Với cả hai cách, output tương tự như sau:

```
secret/db-user-pass created
```

### Xác minh Secret (Verify the Secret) {#verify-the-secret}

Kiểm tra rằng Secret đã được tạo:

```shell
kubectl get secrets
```

Output tương tự như sau:

```
NAME              TYPE       DATA      AGE
db-user-pass      Opaque     2         51s
```

Xem chi tiết của Secret:

```shell
kubectl describe secret db-user-pass
```

Output tương tự như sau:

```
Name:            db-user-pass
Namespace:       default
Labels:          <none>
Annotations:     <none>

Type:            Opaque

Data
====
password:    12 bytes
username:    5 bytes
```

Các lệnh `kubectl get` và `kubectl describe` mặc định tránh hiển thị nội dung
của một `Secret`. Điều này nhằm bảo vệ `Secret` khỏi bị lộ
một cách vô tình, hoặc khỏi bị lưu lại trong log của terminal.

### Giải mã Secret (Decode the Secret) {#decoding-secret}

1. Xem nội dung của Secret bạn đã tạo:

   ```shell
   kubectl get secret db-user-pass -o jsonpath='{.data}'
   ```

   Output tương tự như sau:

   ```json
   { "password": "UyFCXCpkJHpEc2I9", "username": "YWRtaW4=" }
   ```

1. Giải mã (decode) dữ liệu `password`:

   ```shell
   echo 'UyFCXCpkJHpEc2I9' | base64 --decode
   ```

   Output tương tự như sau:

   ```
   S!B\*d$zDsb=
   ```

   > **Thận trọng:** Đây là một ví dụ cho mục đích minh họa trong tài liệu. Trong thực tế,
   > cách này có thể khiến lệnh chứa dữ liệu đã mã hóa bị lưu lại trong
   > lịch sử shell của bạn. Bất kỳ ai có quyền truy cập máy tính của bạn đều có thể tìm thấy
   > lệnh đó và giải mã secret. Cách tốt hơn là kết hợp lệnh xem và
   > lệnh giải mã với nhau.

   ```shell
   kubectl get secret db-user-pass -o jsonpath='{.data.password}' | base64 --decode
   ```

## Sửa một Secret (Edit a Secret) {#edit-secret}

Bạn có thể sửa một đối tượng `Secret` hiện có, trừ khi nó là
[bất biến (immutable)](109-secret-vi.md#secret-immutable). Để sửa một
Secret, hãy chạy lệnh sau:

```shell
kubectl edit secrets <secret-name>
```

Lệnh này mở trình soạn thảo mặc định của bạn và cho phép bạn cập nhật các giá trị
Secret đã mã hóa base64 trong trường `data`, như trong ví dụ sau:

```yaml
# Please edit the object below. Lines beginning with a '#' will be ignored,
# and an empty file will abort the edit. If an error occurs while saving this file, it will be
# reopened with the relevant failures.
#
apiVersion: v1
data:
  password: UyFCXCpkJHpEc2I9
  username: YWRtaW4=
kind: Secret
metadata:
  creationTimestamp: "2022-06-28T17:44:13Z"
  name: db-user-pass
  namespace: default
  resourceVersion: "12708504"
  uid: 91becd59-78fa-4c85-823f-6d44436242ac
type: Opaque
```

## Dọn dẹp (Clean up)

Để xóa một Secret, hãy chạy lệnh sau:

```shell
kubectl delete secret db-user-pass
```

## Tiếp theo (What's next)

- Đọc thêm về [khái niệm Secret](109-secret-vi.md)
- Tìm hiểu cách [quản lý Secret bằng file cấu hình](326-secret-config-file-vi.md)
- Tìm hiểu cách [quản lý Secret bằng kustomize](328-secret-kustomize-vi.md)

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 3:

1. `--from-literal` và `--from-file` khác nhau ở hai chỗ: chuyện escape ký tự đặc biệt, và chuyện
   đặt tên key. Kể ra cả hai.
2. **Câu bẫy.** `kubectl describe secret db-user-pass` in ra `password: 12 bytes` chứ không in giá
   trị. Điều đó chứng tỏ Secret đã được bảo vệ chứ?
3. Trên `lab-k8s-master`, bạn cần xem mật khẩu thật trong Secret `db-cred`. Bài khuyên gõ lệnh
   dạng nào, và vì sao cách "lấy chuỗi base64 ra rồi `echo '...' | base64 --decode`" bị xếp vào
   khối *Thận trọng*?
4. Bạn chạy `echo 'demo-password' > ./password.txt` rồi `kubectl create secret generic ...
   --from-file=./password.txt`. Secret sinh ra sai ở đâu, và tên key của nó là gì?
5. `kubectl edit secrets db-user-pass` mở ra nội dung gì trong editor — plaintext hay base64 — và
   trường hợp nào lệnh này không dùng được?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Thứ nhất, **escape**: với `--from-literal` bạn phải dùng nháy đơn `''` để escape các ký tự đặc
   biệt như `$`, `\`, `*`, `=`, `!`, nếu không shell sẽ diễn giải chúng; còn khi đưa giá trị vào
   file rồi dùng `--from-file` thì **không cần escape gì cả**. Thứ hai, **key**: `--from-literal`
   đặt key ngay trong lệnh (`username=admin`), còn `--from-file` lấy **tên file làm key mặc định**,
   muốn khác thì viết `--from-file=<key>=<nguồn>`.
2. **Không.** `get` và `describe` giấu nội dung chỉ để Secret **khỏi bị lộ vô tình hoặc bị lưu lại
   trong log của terminal** — đó là biện pháp che màn hình, không phải biện pháp bảo vệ dữ liệu.
   Bằng chứng nằm ngay mục sau: chỉ cần `-o jsonpath='{.data}'` rồi `base64 --decode` là ra
   plaintext, không cần quyền gì thêm ngoài quyền đọc chính Secret đó.
3. Gõ **một pipeline duy nhất**: `kubectl get secret db-cred -o jsonpath='{.data.password}' |
   base64 --decode`. Cách hai bước bị cảnh báo vì nó **để chuỗi base64 nằm lại trong lịch sử
   shell** — bài viết: "bất kỳ ai có quyền truy cập máy tính của bạn đều có thể tìm thấy lệnh đó và
   giải mã secret". Ghép hai lệnh lại thì chuỗi bí mật chỉ đi qua ống, không thành đối số.
4. Sai vì thiếu `-n`: `echo` thêm **một ký tự xuống dòng** vào cuối file, và khi `kubectl` đọc file
   rồi mã hóa nội dung thành base64 thì **ký tự thừa đó cũng bị mã hóa theo** — mật khẩu trong
   Secret dài hơn mật khẩu thật một ký tự. **Key là `password.txt`**, vì key mặc định là tên file;
   muốn key là `password` thì viết `--from-file=password=./password.txt`.
5. Editor mở ra object với **các giá trị đã mã hóa base64 trong trường `data`** — bạn sửa chuỗi
   base64, không sửa plaintext. Lệnh **không dùng được với Secret đã đặt `immutable`**: bài mở đầu
   mục đó bằng đúng điều kiện "bạn có thể sửa một đối tượng `Secret` hiện có, **trừ khi** nó là bất
   biến".

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
