# Expose thông tin Pod cho container thông qua biến môi trường (Expose Pod Information to Containers Through Environment Variables)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/inject-data-application/environment-variable-expose-pod-information/>
>
> Trang này hướng dẫn cách một Pod có thể dùng biến môi trường để expose thông tin về chính nó cho các container đang chạy trong Pod, thông qua downward API.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 3 — Pod và cấu hình](00-ALO-TRINH-ADMIN.md#giai-đoạn-3--pod-và-cấu-hình)
→ nhóm [3a. Pod và vòng đời](00-ALO-TRINH-ADMIN.md#3a-pod-và-vòng-đời), bài thực hành 10/11 ·
Kiểm chứng ở [Lab 3a](labs/LAB-3A-POD-VA-VONG-DOI.md) phần B10.1.

Đây là **nửa còn lại** của downward API, song song với bài
[335](335-downward-api-volume-vi.md) vừa đọc: cùng hai nguồn giá trị, chỉ khác nơi giá trị đi
tới — biến môi trường thay vì file. Đọc để so hai bài với nhau, đừng đọc như một bài rời.

**Phải hiểu ở lần đọc này:**

- Điểm neo cú pháp là **`env[].valueFrom`**: đặt `valueFrom` thay cho `value` thì giá trị của biến
  môi trường được lấy từ chính object Pod chứ không viết cứng trong manifest.
- Hai nguồn giá trị, đúng ranh giới của bài [56](56-downward-api-vi.md):
  **`fieldRef.fieldPath`** cho field **cấp Pod**, **`resourceFieldRef`** cho field **cấp
  container** — và vì thế `resourceFieldRef` phải nêu **`containerName`** cùng **`resource`**.
  Ghi chú của bài nhấn đúng chỗ này ở ví dụ đầu.
- Năm field cấp Pod trong ví dụ đầu trải trên **ba nhánh khác nhau** của object: `spec.nodeName`
  và `spec.serviceAccountName` từ `spec`; `metadata.name` và `metadata.namespace` từ `metadata`;
  `status.podIP` từ `status`. Downward API không chỉ đọc phần bạn viết ra, nó đọc cả phần
  Kubernetes điền vào.
- Giá trị mà biến môi trường nhận là giá trị **đã chuẩn hóa**, không phải chuỗi bạn gõ trong
  manifest: `memory: "32Mi"` ra `33554432` và `"64Mi"` ra `67108864` — tức **byte**, khớp với
  mặc định `divisor: 1` mà bài [335](335-downward-api-volume-vi.md) vừa nêu; tương tự `125m` và
  `250m` của CPU đều in ra `1` vì mặc định đơn vị là **core**.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Đoạn mở đầu về Service và "khám phá Service lúc runtime", cùng link *Accessing the Service* | Service chưa học | [giai đoạn 5](00-ALO-TRINH-ADMIN.md#giai-đoạn-5--mạng-nền-tảng), thực hành ở Lab 5a |
| Biến `MY_POD_SERVICE_ACCOUNT` lấy từ `spec.serviceAccountName` | ở đây chỉ cần biết đó là một field của Pod đọc được; ServiceAccount là danh tính của Pod khi gọi API | bài [118](118-service-accounts-vi.md) ở [giai đoạn 9](00-ALO-TRINH-ADMIN.md#giai-đoạn-9--bảo-mật-và-multi-tenancy), thực hành ở Lab 9a |
| Ý nghĩa của `resources.requests` và `resources.limits` trong manifest thứ hai | ở đây chỉ cần biết downward API **đọc lại** hai giá trị đó | bài [110](110-manage-resources-containers-vi.md) ở nhóm [3c](00-ALO-TRINH-ADMIN.md#3c-tài-nguyên-qos-và-gián-đoạn), thực hành ở Lab 3c |
| Mục *Tiếp theo* trỏ sang "Định nghĩa biến môi trường cho container" | đó là biến môi trường thông thường, không đi qua downward API | bài [331](331-define-environment-variable-vi.md) ở nhóm [3b](00-ALO-TRINH-ADMIN.md#3b-cấu-hình-ứng-dụng-configmap-secret-và-dữ-liệu-cho-pod), thực hành ở Lab 3b |

---

Trang này hướng dẫn cách một Pod có thể dùng biến môi trường để expose thông tin
về chính nó cho các container đang chạy trong Pod, bằng cách sử dụng _downward API_.
Bạn có thể dùng biến môi trường để expose các field của Pod, các field của container, hoặc cả hai.

Trong Kubernetes, có hai cách để expose các field của Pod và container cho một container đang chạy:

* _Biến môi trường_, như được trình bày trong tác vụ này
* [File trong volume](335-downward-api-volume-vi.md)

Gộp chung lại, hai cách expose các field của Pod và container này được gọi là
downward API.

Vì Service là phương thức giao tiếp chính giữa các ứng dụng chạy trong container do Kubernetes quản lý,
việc có thể khám phá (discover) chúng lúc chạy (runtime) là rất hữu ích.

Đọc thêm về cách truy cập Service [tại đây](https://kubernetes.io/docs/tutorials/services/connect-applications-service/#accessing-the-service).

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao tiếp
với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node không đóng
vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster bằng
[minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong các sân
chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

## Dùng field của Pod làm giá trị cho biến môi trường (Use Pod fields as values for environment variables)

Trong phần này của bài thực hành, bạn tạo một Pod có một container, và bạn
chiếu (project) các field ở cấp Pod vào container đang chạy dưới dạng các biến môi trường.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: dapi-envars-fieldref
spec:
  containers:
    - name: test-container
      image: registry.k8s.io/busybox:1.27.2
      command: [ "sh", "-c"]
      args:
      - while true; do
          echo -en '\n';
          printenv MY_NODE_NAME MY_POD_NAME MY_POD_NAMESPACE;
          printenv MY_POD_IP MY_POD_SERVICE_ACCOUNT;
          sleep 10;
        done;
      env:
        - name: MY_NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        - name: MY_POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: MY_POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: MY_POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
        - name: MY_POD_SERVICE_ACCOUNT
          valueFrom:
            fieldRef:
              fieldPath: spec.serviceAccountName
  restartPolicy: Never
```

Trong manifest đó, bạn có thể thấy năm biến môi trường. Field `env`
là một mảng các định nghĩa biến môi trường.
Phần tử đầu tiên trong mảng chỉ định rằng biến môi trường `MY_NODE_NAME`
lấy giá trị từ field `spec.nodeName` của Pod. Tương tự, các
biến môi trường còn lại lấy giá trị từ các field của Pod.

> **Ghi chú:**
> Các field trong ví dụ này là các field của Pod. Chúng không phải là field của
> container trong Pod.

Tạo Pod:

```shell
kubectl apply -f https://k8s.io/examples/pods/inject/dapi-envars-pod.yaml
```

Xác minh rằng container trong Pod đang chạy:

```shell
# Nếu Pod mới chưa healthy, hãy chạy lại lệnh này vài lần.
kubectl get pods
```

Xem log của container:

```shell
kubectl logs dapi-envars-fieldref
```

Kết quả hiển thị giá trị của các biến môi trường được chọn:

```
minikube
dapi-envars-fieldref
default
172.17.0.4
default
```

Để hiểu vì sao các giá trị này xuất hiện trong log, hãy xem các field `command` và `args`
trong file cấu hình. Khi container khởi động, nó ghi giá trị của
năm biến môi trường ra stdout. Nó lặp lại việc này mỗi mười giây.

Tiếp theo, mở một shell vào container đang chạy trong Pod của bạn:

```shell
kubectl exec -it dapi-envars-fieldref -- sh
```

Trong shell của bạn, xem các biến môi trường:

```shell
# Chạy lệnh này trong shell bên trong container
printenv
```

Kết quả cho thấy một số biến môi trường đã được gán
giá trị của các field của Pod:

```
MY_POD_SERVICE_ACCOUNT=default
...
MY_POD_NAMESPACE=default
MY_POD_IP=172.17.0.4
...
MY_NODE_NAME=minikube
...
MY_POD_NAME=dapi-envars-fieldref
```

## Dùng field của container làm giá trị cho biến môi trường (Use container fields as values for environment variables)

Ở bài thực hành trước, bạn đã dùng thông tin từ các field ở cấp Pod làm giá trị
cho biến môi trường.
Trong bài thực hành tiếp theo này, bạn sẽ truyền các field vốn là một phần của định nghĩa
Pod, nhưng được lấy từ một
[container](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-v1/#Container)
cụ thể thay vì từ Pod nói chung.

Dưới đây là manifest cho một Pod khác cũng chỉ có một container:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: dapi-envars-resourcefieldref
spec:
  containers:
    - name: test-container
      image: registry.k8s.io/busybox:1.27.2
      command: [ "sh", "-c"]
      args:
      - while true; do
          echo -en '\n';
          printenv MY_CPU_REQUEST MY_CPU_LIMIT;
          printenv MY_MEM_REQUEST MY_MEM_LIMIT;
          sleep 10;
        done;
      resources:
        requests:
          memory: "32Mi"
          cpu: "125m"
        limits:
          memory: "64Mi"
          cpu: "250m"
      env:
        - name: MY_CPU_REQUEST
          valueFrom:
            resourceFieldRef:
              containerName: test-container
              resource: requests.cpu
        - name: MY_CPU_LIMIT
          valueFrom:
            resourceFieldRef:
              containerName: test-container
              resource: limits.cpu
        - name: MY_MEM_REQUEST
          valueFrom:
            resourceFieldRef:
              containerName: test-container
              resource: requests.memory
        - name: MY_MEM_LIMIT
          valueFrom:
            resourceFieldRef:
              containerName: test-container
              resource: limits.memory
  restartPolicy: Never
```

Trong manifest này, bạn có thể thấy bốn biến môi trường. Field `env`
là một mảng các định nghĩa biến môi trường.
Phần tử đầu tiên trong mảng chỉ định rằng biến môi trường `MY_CPU_REQUEST`
lấy giá trị từ field `requests.cpu` của container có tên
`test-container`. Tương tự, các biến môi trường còn lại lấy giá trị
từ các field riêng của container này.

Tạo Pod:

```shell
kubectl apply -f https://k8s.io/examples/pods/inject/dapi-envars-container.yaml
```

Xác minh rằng container trong Pod đang chạy:

```shell
# Nếu Pod mới chưa healthy, hãy chạy lại lệnh này vài lần.
kubectl get pods
```

Xem log của container:

```shell
kubectl logs dapi-envars-resourcefieldref
```

Kết quả hiển thị giá trị của các biến môi trường được chọn:

```
1
1
33554432
67108864
```

## Tiếp theo (What's next)

* Đọc [Định nghĩa biến môi trường cho container](331-define-environment-variable-vi.md)
* Đọc định nghĩa API [`spec`](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-v1/#PodSpec)
  của Pod. Định nghĩa này bao gồm cả định nghĩa của Container (một phần của Pod).
* Đọc danh sách [các field khả dụng](56-downward-api-vi.md#available-fields) mà bạn
  có thể expose bằng downward API.

Đọc về Pod, container và biến môi trường trong tài liệu tham khảo API cũ (legacy):

* [PodSpec](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.36/#podspec-v1-core)
* [Container](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.36/#container-v1-core)
* [EnvVar](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.36/#envvar-v1-core)
* [EnvVarSource](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.36/#envvarsource-v1-core)
* [ObjectFieldSelector](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.36/#objectfieldselector-v1-core)
* [ResourceFieldSelector](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.36/#resourcefieldselector-v1-core)

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 3:

1. Trong khối `env`, cái gì báo cho Kubernetes biết giá trị phải lấy từ object Pod chứ không phải
   từ chuỗi bạn gõ sẵn? Hai nguồn giá trị là gì, và nguồn nào bắt buộc nêu `containerName`?
2. Năm biến của ví dụ đầu lấy từ ba nhánh khác nhau của object Pod. Kể tên ba nhánh đó, mỗi nhánh
   một field ví dụ. Vì sao có nhánh mà bạn không hề viết ra trong manifest?
3. **Câu bẫy.** Manifest ghi `memory: "32Mi"`, nhưng `kubectl logs` in ra `33554432`. Kubernetes
   đọc sai, hay bạn hiểu sai? Còn `125m` và `250m` của CPU thì in ra gì?
4. Bạn tạo Pod `dapi-envars-fieldref` trên cluster lab và nó được lập lịch lên `lab-k8s-worker2`.
   So với đầu ra mẫu trong bài (`minikube`, `dapi-envars-fieldref`, `default`, `172.17.0.4`,
   `default`), năm dòng log của bạn khác ở dòng nào và giống ở dòng nào?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **`valueFrom`** — có `valueFrom` thì giá trị được lấy từ chính object Pod, thay vì `value` với
   một chuỗi cố định. Hai nguồn: **`fieldRef`** (kèm `fieldPath`) cho field **cấp Pod**, và
   **`resourceFieldRef`** cho field **cấp container**. Nguồn bắt buộc nêu **`containerName`** là
   `resourceFieldRef` — trong ví dụ là `containerName: test-container` cùng
   `resource: requests.cpu`.
2. **`spec`** — `spec.nodeName`, `spec.serviceAccountName`; **`metadata`** — `metadata.name`,
   `metadata.namespace`; **`status`** — `status.podIP`. Nhánh bạn không viết ra là **`status`**
   (và một phần của `spec` như `nodeName`): đó là phần **Kubernetes điền vào** sau khi Pod được
   tạo và được lập lịch. Downward API đọc object Pod như nó đang tồn tại trên cluster, không phải
   file YAML bạn gõ.
3. **Bạn hiểu sai, Kubernetes không sai.** Giá trị mà biến môi trường nhận là giá trị **đã được
   chuẩn hóa về đơn vị cơ sở**, không phải nguyên văn chuỗi trong manifest: `32Mi` ra
   **`33554432`** và `64Mi` ra **`67108864`**, tức **byte**. Đây là hệ quả của mặc định
   `divisor: 1` mà bài [335](335-downward-api-volume-vi.md) đã nêu — divisor 1 nghĩa là byte cho
   `memory` và **core** cho `cpu`. Vì thế `125m` và `250m` của CPU **đều in ra `1`**. Chỗ dễ sai
   là chờ đợi chuỗi `32Mi` xuất hiện nguyên vẹn trong `printenv`.
4. **Khác ở dòng 1 và dòng 4.** Dòng 1 là `MY_NODE_NAME` từ `spec.nodeName` — của bạn sẽ là
   `lab-k8s-worker2` chứ không phải `minikube`. Dòng 4 là `MY_POD_IP` từ `status.podIP` — địa chỉ
   do cluster của bạn cấp, không phải `172.17.0.4`. **Giống ở dòng 2, 3 và 5**: `MY_POD_NAME` là
   `dapi-envars-fieldref` vì tên nằm trong manifest; `MY_POD_NAMESPACE` và
   `MY_POD_SERVICE_ACCOUNT` đều là `default` nếu bạn tạo Pod trong namespace `default` và không
   chỉ định service account.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
