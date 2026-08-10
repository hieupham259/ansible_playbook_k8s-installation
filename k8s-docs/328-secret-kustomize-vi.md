# Quản lý Secret bằng Kustomize (Managing Secrets using Kustomize)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/configmap-secret/managing-secret-using-kustomize/>
>
> Tạo các đối tượng Secret bằng file kustomization.yaml.

`kubectl` hỗ trợ sử dụng [công cụ quản lý đối tượng Kustomize](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/) để quản lý Secret
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
[Quản lý Secret bằng kubectl](https://kubernetes.io/docs/tasks/configmap-secret/managing-secret-using-kubectl/#verify-the-secret) và
[Quản lý đối tượng Kubernetes theo kiểu khai báo bằng Kustomize](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/).

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

- Đọc thêm về [khái niệm Secret](https://kubernetes.io/docs/concepts/configuration/secret/)
- Tìm hiểu cách [quản lý Secret bằng kubectl](https://kubernetes.io/docs/tasks/configmap-secret/managing-secret-using-kubectl/)
- Tìm hiểu cách [quản lý Secret bằng file cấu hình](https://kubernetes.io/docs/tasks/configmap-secret/managing-secret-using-config-file/)
