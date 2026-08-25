# Quản lý Secret bằng Kustomize (Managing Secrets using Kustomize)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/configmap-secret/managing-secret-using-kustomize/>
>
> Tạo các đối tượng Secret bằng file kustomization.yaml.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 3 — Pod và cấu hình](00-ALO-TRINH-ADMIN.md#giai-đoạn-3--pod-và-cấu-hình)
→ nhóm [3b. Cấu hình ứng dụng](00-ALO-TRINH-ADMIN.md#3b-cấu-hình-ứng-dụng-configmap-secret-và-dữ-liệu-cho-pod),
bài 6/12 · Kiểm chứng ở [Lab 3b](labs/LAB-3B-CAU-HINH-UNG-DUNG.md) phần B6.3 — `secretGenerator`,
hậu tố hash, và bằng chứng rằng sửa dữ liệu sinh ra object mới.

Đây là bài **thứ ba trong ba bài** làm cùng một việc: [326](326-secret-config-file-vi.md) (file cấu
hình), [327](327-secret-kubectl-vi.md) (`kubectl`), 328 (Kustomize). Đường của bài này khác hai
đường kia ở một điểm quyết định: **tên Secret mang hậu tố hash của dữ liệu**, nên đổi dữ liệu là
sinh ra một object mới chứ không sửa object cũ. Hợp khi Secret được sinh cùng bộ manifest và bạn
muốn mọi thay đổi dữ liệu đều lộ ra thành một tên mới; đổi lại bạn phải tự cập nhật tham chiếu
trong Pod và tự dọn Secret cũ.

Bài giả định bạn đã biết Kustomize. Ở lần đọc này chỉ cần nắm đúng một trường `secretGenerator`.

**Phải hiểu ở lần đọc này:**

- `secretGenerator` trong `kustomization.yaml` nhận ba dạng nguồn — `literals`, `files` (tên file
  thành key), và `envs` (file `.env`) — và ở **mọi dạng bạn đều không phải tự mã hóa base64**; mục
  *Tạo file kustomization* nói thẳng điều đó.
- File **bắt buộc** mang tên `kustomization.yaml` hoặc `kustomization.yml`, và được áp dụng bằng
  `kubectl apply -k <đường-dẫn-thư-mục>` — mục *Apply file kustomization*.
- Tên Secret sinh ra là **tên khai báo cộng hậu tố băm từ dữ liệu** (`database-creds-5hdh7hhgfk`).
  Bài nêu rõ mục đích: "đảm bảo rằng một Secret mới sẽ được sinh ra mỗi khi dữ liệu bị thay đổi".
- Hệ quả ở mục *Sửa một Secret*: sửa dữ liệu rồi apply lại thì **Secret được tạo thành một object
  mới, thay vì cập nhật object hiện có** — và bài cảnh báo "bạn có thể cần cập nhật các tham chiếu
  tới Secret trong các Pod của mình".
- Xem và giải mã cũng đi qua thư mục kustomization: `kubectl get -k <đường-dẫn> -o
  jsonpath='{.data}'` rồi `base64 --decode`. Dữ liệu ra vẫn chỉ là **base64**, y hệt hai đường kia.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Bản thân công cụ Kustomize — cấu trúc đầy đủ của `kustomization.yaml`, overlay, patch | ở đây bạn chỉ dùng đúng một trường `secretGenerator` và một lệnh `apply -k` | bài [322](322-kustomization-vi.md), [giai đoạn 5](00-ALO-TRINH-ADMIN.md#giai-đoạn-5--mạng-nền-tảng) |
| Ghi chú *`stringData` không hoạt động tốt với server-side apply* | `secretGenerator` không bắt bạn viết `stringData`, và ở giai đoạn này bạn apply kiểu client-side | trường `stringData` ở bài [326](326-secret-config-file-vi.md) cùng nhóm; server-side apply ở bài [61](61-management-vi.md), [giai đoạn 4](00-ALO-TRINH-ADMIN.md#giai-đoạn-4--workload-controller) |

---

`kubectl` hỗ trợ sử dụng [công cụ quản lý đối tượng Kustomize](322-kustomization-vi.md) để quản lý Secret
và ConfigMap. Bạn tạo một *bộ sinh tài nguyên* (resource generator) bằng Kustomize, bộ sinh này
sẽ tạo ra một Secret mà bạn có thể apply lên API server bằng `kubectl`.

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình
để giao tiếp với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít
nhất hai node không đóng vai trò control plane host. Nếu bạn chưa có cluster, bạn có
thể tạo một cluster bằng [minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/)
hoặc dùng một trong các sân chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

## Tạo một Secret (Create a Secret)

Bạn có thể sinh ra một Secret bằng cách định nghĩa một `secretGenerator` trong file
`kustomization.yaml` tham chiếu tới các file khác đã tồn tại, các file `.env`, hoặc
các giá trị trực tiếp (literal). Ví dụ, các hướng dẫn sau đây tạo một file kustomization
cho username `admin` và password `1f2d1e2e67df`.

> **Ghi chú:** Trường `stringData` của Secret không hoạt động tốt với server-side apply.

### Tạo file kustomization (Create the kustomization file)

#### Giá trị trực tiếp (Literals)

```yaml
secretGenerator:
- name: database-creds
  literals:
  - username=admin
  - password=1f2d1e2e67df
```

#### File (Files)

1.  Lưu thông tin xác thực (credentials) vào các file. Tên file chính là các key của Secret:

    ```shell
    echo -n 'admin' > ./username.txt
    echo -n '1f2d1e2e67df' > ./password.txt
    ```
    Flag `-n` đảm bảo không có ký tự xuống dòng (newline) ở cuối các file của bạn.

1.  Tạo file `kustomization.yaml`:

    ```yaml
    secretGenerator:
    - name: database-creds
      files:
      - username.txt
      - password.txt
    ```

#### File .env (.env files)

Bạn cũng có thể định nghĩa secretGenerator trong file `kustomization.yaml` bằng cách
cung cấp các file `.env`. Ví dụ, file `kustomization.yaml` sau đây lấy dữ liệu
từ một file `.env.secret`:

```yaml
secretGenerator:
- name: db-user-pass
  envs:
  - .env.secret
```

Trong mọi trường hợp, bạn không cần mã hóa các giá trị sang base64. Tên của file YAML
**bắt buộc** phải là `kustomization.yaml` hoặc `kustomization.yml`.

### Apply file kustomization (Apply the kustomization file)

Để tạo Secret, hãy apply thư mục chứa file kustomization:

```shell
kubectl apply -k <directory-path>
```

Kết quả xuất ra tương tự như:

```
secret/database-creds-5hdh7hhgfk created
```

Khi một Secret được sinh ra, tên của Secret được tạo bằng cách băm (hash)
dữ liệu của Secret rồi nối giá trị băm vào tên. Điều này đảm bảo rằng
một Secret mới sẽ được sinh ra mỗi khi dữ liệu bị thay đổi.

Để xác nhận rằng Secret đã được tạo và để giải mã dữ liệu của Secret,

```shell
kubectl get -k <directory-path> -o jsonpath='{.data}' 
```

Kết quả xuất ra tương tự như:

```
{ "password": "MWYyZDFlMmU2N2Rm", "username": "YWRtaW4=" }
```

```
echo 'MWYyZDFlMmU2N2Rm' | base64 --decode
```

Kết quả xuất ra tương tự như:

```
1f2d1e2e67df
```

Để biết thêm thông tin, hãy tham khảo
[Quản lý Secret bằng kubectl](327-secret-kubectl-vi.md#verify-the-secret) và
[Quản lý đối tượng Kubernetes theo kiểu khai báo bằng Kustomize](322-kustomization-vi.md).

## Sửa một Secret (Edit a Secret) {#edit-secret}

1.  Trong file `kustomization.yaml` của bạn, sửa dữ liệu, chẳng hạn như `password`.
1.  Apply thư mục chứa file kustomization:

    ```shell
    kubectl apply -k <directory-path>
    ```

    Kết quả xuất ra tương tự như:

    ```
    secret/db-user-pass-6f24b56cc8 created
    ```

Secret sau khi sửa được tạo thành một đối tượng `Secret` mới, thay vì cập nhật
đối tượng `Secret` hiện có. Bạn có thể cần cập nhật các tham chiếu tới Secret
trong các Pod của mình.

## Dọn dẹp (Clean up)

Để xóa một Secret, dùng `kubectl`:

```shell
kubectl delete secret db-user-pass
```

## Tiếp theo (What's next)

- Đọc thêm về [khái niệm Secret](109-secret-vi.md)
- Tìm hiểu cách [quản lý Secret bằng kubectl](327-secret-kubectl-vi.md)
- Tìm hiểu cách [quản lý Secret bằng file cấu hình](326-secret-config-file-vi.md)

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 3:

1. `secretGenerator` nhận ba dạng nguồn nào, và ở dạng nào thì bạn phải tự chạy `base64` trước?
2. **Câu bẫy.** Trên `lab-k8s-master`, bạn sửa `password` trong `~/lab-work/3b/kust/kustomization.yaml`
   rồi chạy lại `kubectl apply -k ~/lab-work/3b/kust`. Secret cũ có bị thay giá trị không? Có bị
   xóa không? Và một Pod đang mount Secret cũ có thấy giá trị mới không?
3. Tên Secret sinh ra có dạng `database-creds-5hdh7hhgfk`. Hậu tố đó từ đâu ra và để làm gì?
4. So với [327](327-secret-kubectl-vi.md), đường Kustomize giúp bạn tránh được thao tác nào — và
   nó có làm dữ liệu Secret an toàn hơn không?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Ba dạng: **`literals`** (viết thẳng `username=admin`), **`files`** (tên file trở thành key), và
   **`envs`** (file `.env`). Câu sau là bẫy nhỏ: **không dạng nào cả** — bài viết "trong mọi trường
   hợp, bạn không cần mã hóa các giá trị sang base64"; Kustomize làm việc đó khi sinh object.
2. **Không bị thay, không bị xóa.** Bài nói thẳng: "Secret sau khi sửa được tạo thành một đối tượng
   `Secret` **mới**, thay vì cập nhật đối tượng `Secret` hiện có" — vì dữ liệu đổi thì hash đổi,
   hash đổi thì tên đổi, và tên mới là một object khác. Secret cũ **vẫn nằm đó** cho tới khi bạn tự
   xóa. Pod đang mount Secret cũ tham chiếu theo **tên cũ**, nên nó vẫn thấy dữ liệu cũ; đúng như
   cảnh báo của bài, "bạn có thể cần cập nhật các tham chiếu tới Secret trong các Pod của mình".
3. Hậu tố là **giá trị băm (hash) của dữ liệu Secret**, được nối vào tên khi Secret được sinh ra.
   Mục đích bài nêu rõ: **đảm bảo một Secret mới được sinh ra mỗi khi dữ liệu bị thay đổi** — tức
   biến "dữ liệu đã đổi" thành một sự kiện nhìn thấy được ở cấp tên object, thay vì một thay đổi
   ngầm bên trong object cũ.
4. Nó giúp bạn khỏi phải **tự mã hóa base64** và khỏi gõ giá trị bí mật thành đối số dòng lệnh —
   giá trị nằm trong `kustomization.yaml` hoặc trong file nguồn. Nhưng **không**, dữ liệu không an
   toàn hơn chút nào: object sinh ra vẫn là một `Secret` bình thường, và bài kết thúc phần xem dữ
   liệu bằng đúng thao tác của hai đường kia — `jsonpath='{.data}'` rồi `base64 --decode` ra
   plaintext. Cái đổi là quy trình quản lý, không phải mức bảo vệ.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
