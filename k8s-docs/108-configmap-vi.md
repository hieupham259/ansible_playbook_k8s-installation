# ConfigMap (ConfigMaps)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/configuration/configmap/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](LO-TRINH-ADMIN.md).

**Vị trí:** Giai đoạn 3 → nhóm [3b](LO-TRINH-ADMIN.md#3b-cấu-hình-và-tài-nguyên), bài 2/7 ·
Kiểm chứng ở Lab 3b (chưa viết, xem [bản đồ lab](labs/README.md#4-bản-đồ-lab)).

Bài vừa đủ ngắn để đọc hết trong một lượt, nhưng phần dễ bỏ sót lại là phần quan trọng nhất
khi vận hành: **ranh giới cập nhật**. Ba cách tiêu thụ ConfigMap có ba hành vi cập nhật khác
nhau, và đó là nguồn của loại sự cố "tôi đã sửa cấu hình rồi mà ứng dụng không đổi".

**Phải hiểu ở lần đọc này:**

- ConfigMap là dữ liệu **không bí mật** dạng key-value, và khác hầu hết object Kubernetes, nó
  không có `spec` mà có `data` (chuỗi UTF-8) và `binaryData` (base64) — mục *Đối tượng
  ConfigMap*. Trần dữ liệu là 1 MiB, nêu ở mục *Động lực*.
- Bốn cách một Pod tiêu thụ ConfigMap (mục *ConfigMap và Pod*), và với ba cách đầu thì
  **kubelet nạp dữ liệu lúc khởi chạy container**.
- Ranh giới cập nhật, mục *ConfigMap đã mount được cập nhật tự động*: mount làm volume thì
  được cập nhật (trễ bằng chu kỳ đồng bộ của kubelet cộng độ trễ lan truyền cache); dùng làm
  biến môi trường thì **không**, phải khởi động lại Pod; mount kiểu `subPath` cũng không.
- Pod và ConfigMap phải **cùng namespace**; static Pod không tham chiếu được ConfigMap. Cách
  duy nhất đọc ConfigMap ở namespace khác là cách thứ tư — tự gọi Kubernetes API từ trong Pod.
- Mảng `items` trong `.spec.volumes[].configMap` quyết định key nào thành file; bỏ hẳn `items`
  thì mọi key đều thành file. `immutable: true` là thay đổi một chiều, muốn sửa phải xóa và
  tạo lại.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Cách thứ tư — viết mã trong Pod gọi Kubernetes API để đọc và subscribe ConfigMap | cần ServiceAccount và RBAC để Pod có quyền đọc | giai đoạn 9, bài [118](118-service-accounts-vi.md) |
| `configMapAndSecretChangeDetectionStrategy` và các loại cache của kubelet | là tham số cấu hình kubelet | giai đoạn 8, bài [04](04-kubelet-integration-vi.md) |
| Volume mount kiểu [subPath](./91-volumes-vi.md#using-subpath) | chưa học volume | giai đoạn 6, bài [91](91-volumes-vi.md) |
| Lợi ích hiệu năng của ConfigMap bất biến (đóng watch, giảm tải kube-apiserver) | chỉ có nghĩa ở cluster hàng chục nghìn lượt mount | không cần |

---

ConfigMap là một API object dùng để lưu dữ liệu không bí mật (non-confidential) dưới
dạng các cặp key-value. Các Pod có thể tiêu thụ ConfigMap dưới dạng biến môi trường
(environment variable), đối số dòng lệnh (command-line argument), hoặc dưới dạng các
file cấu hình trong một volume.

ConfigMap cho phép bạn tách cấu hình phụ thuộc vào môi trường ra khỏi container image,
nhờ đó ứng dụng của bạn dễ dàng di chuyển giữa các môi trường.

> **Thận trọng:** ConfigMap không cung cấp khả năng giữ bí mật hay mã hóa.
> Nếu dữ liệu bạn muốn lưu là dữ liệu mật, hãy dùng Secret thay vì ConfigMap,
> hoặc dùng thêm các công cụ (bên thứ ba) để giữ dữ liệu của bạn ở chế độ riêng tư.

## Động lực (Motivation)

Hãy dùng ConfigMap để thiết lập dữ liệu cấu hình tách biệt khỏi mã của ứng dụng.

Ví dụ, hãy tưởng tượng bạn đang phát triển một ứng dụng có thể chạy trên máy tính của
chính bạn (để phát triển) và trên đám mây (để xử lý lưu lượng thật). Bạn viết mã đọc
biến môi trường có tên `DATABASE_HOST`. Ở máy cục bộ, bạn đặt biến đó thành `localhost`.
Trên đám mây, bạn đặt nó tham chiếu tới một Service của Kubernetes — Service này expose
thành phần cơ sở dữ liệu cho cluster của bạn. Điều này cho phép bạn lấy về một container
image đang chạy trên đám mây và debug chính xác cùng một đoạn mã đó ở máy cục bộ khi cần.

> **Ghi chú:** ConfigMap không được thiết kế để chứa những khối dữ liệu lớn. Dữ liệu
> lưu trong một ConfigMap không được vượt quá 1 MiB. Nếu bạn cần lưu các thiết lập lớn
> hơn giới hạn này, bạn có thể cân nhắc mount một volume hoặc dùng một cơ sở dữ liệu
> hay dịch vụ file riêng.

## Đối tượng ConfigMap (ConfigMap object)

ConfigMap là một API object cho phép bạn lưu cấu hình để các object khác sử dụng.
Khác với hầu hết các object Kubernetes vốn có `spec`, ConfigMap có các trường `data`
và `binaryData`. Các trường này nhận các cặp key-value làm giá trị. Cả trường `data`
lẫn trường `binaryData` đều không bắt buộc. Trường `data` được thiết kế để chứa các
chuỗi UTF-8, còn trường `binaryData` được thiết kế để chứa dữ liệu nhị phân dưới dạng
các chuỗi mã hóa base64.

Tên của một ConfigMap phải là một
[tên DNS subdomain](https://kubernetes.io/docs/concepts/overview/working-with-objects/names#dns-subdomain-names) hợp lệ.

Mỗi key dưới trường `data` hoặc `binaryData` phải bao gồm các ký tự chữ-số, `-`, `_`
hoặc `.`. Các key lưu trong `data` không được trùng lặp với các key trong trường
`binaryData`.

Kể từ v1.19, bạn có thể thêm trường `immutable` vào định nghĩa ConfigMap để tạo một
[ConfigMap bất biến](#configmap-immutable).

## ConfigMap và Pod (ConfigMaps and Pods)

Bạn có thể viết `spec` của Pod tham chiếu tới một ConfigMap và cấu hình (các) container
trong Pod đó dựa trên dữ liệu trong ConfigMap. Pod và ConfigMap phải nằm trong cùng một
namespace.

> **Ghi chú:** `spec` của một static Pod không thể tham chiếu tới ConfigMap hay bất kỳ
> API object nào khác.

Đây là một ví dụ ConfigMap có một số key với giá trị đơn, và một số key khác có giá trị
trông giống như một đoạn của một định dạng cấu hình.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: game-demo
data:
  # các key kiểu thuộc tính (property); mỗi key ánh xạ tới một giá trị đơn giản
  player_initial_lives: "3"
  ui_properties_file_name: "user-interface.properties"

  # các key kiểu file
  game.properties: |
    enemy.types=aliens,monsters
    player.maximum-lives=5
  user-interface.properties: |
    color.good=purple
    color.bad=yellow
    allow.textmode=true
```

Có bốn cách khác nhau để bạn có thể dùng một ConfigMap nhằm cấu hình một container bên
trong Pod:

1. Bên trong lệnh (command) và các đối số (args) của container
1. Biến môi trường cho một container
1. Thêm một file trong volume chỉ đọc, để ứng dụng đọc
1. Viết mã chạy bên trong Pod, dùng Kubernetes API để đọc ConfigMap

Các phương thức khác nhau này phù hợp với những cách mô hình hóa khác nhau đối với dữ
liệu được tiêu thụ. Với ba phương thức đầu tiên, kubelet dùng dữ liệu từ ConfigMap khi
khởi chạy (các) container cho một Pod.

Phương thức thứ tư nghĩa là bạn phải viết mã để đọc ConfigMap và dữ liệu của nó. Tuy
nhiên, vì bạn dùng trực tiếp Kubernetes API, ứng dụng của bạn có thể đăng ký (subscribe)
để nhận cập nhật mỗi khi ConfigMap thay đổi, và phản ứng khi điều đó xảy ra. Bằng cách
truy cập trực tiếp Kubernetes API, kỹ thuật này cũng cho phép bạn truy cập một ConfigMap
ở một namespace khác.

Đây là một ví dụ Pod dùng các giá trị từ `game-demo` để cấu hình Pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: configmap-demo-pod
spec:
  containers:
    - name: demo
      image: alpine
      command: ["sleep", "3600"]
      env:
        # Định nghĩa biến môi trường
        - name: PLAYER_INITIAL_LIVES # Chú ý rằng cách viết hoa/thường ở đây khác
                                     # với tên key trong ConfigMap.
          valueFrom:
            configMapKeyRef:
              name: game-demo           # ConfigMap mà giá trị này lấy từ đó.
              key: player_initial_lives # Key cần lấy.
        - name: UI_PROPERTIES_FILE_NAME
          valueFrom:
            configMapKeyRef:
              name: game-demo
              key: ui_properties_file_name
      volumeMounts:
      - name: config
        mountPath: "/config"
        readOnly: true
  volumes:
  # Bạn khai báo volume ở cấp Pod, sau đó mount chúng vào các container bên trong Pod đó
  - name: config
    configMap:
      # Cung cấp tên của ConfigMap bạn muốn mount.
      name: game-demo
      # Mảng các key từ ConfigMap sẽ được tạo thành file
      items:
      - key: "game.properties"
        path: "game.properties"
      - key: "user-interface.properties"
        path: "user-interface.properties"
```

ConfigMap không phân biệt giữa giá trị thuộc tính một dòng và giá trị nhiều dòng kiểu
file. Điều quan trọng là cách các Pod và các object khác tiêu thụ những giá trị đó.

Trong ví dụ này, việc định nghĩa một volume và mount nó vào bên trong container `demo`
tại `/config` tạo ra hai file, `/config/game.properties` và
`/config/user-interface.properties`, mặc dù trong ConfigMap có bốn key. Đó là vì định
nghĩa Pod chỉ định một mảng `items` trong phần `volumes`. Nếu bạn bỏ hẳn mảng `items`,
mỗi key trong ConfigMap sẽ trở thành một file có tên trùng với key, và bạn nhận được 4 file.

## Sử dụng ConfigMap (Using ConfigMaps)

ConfigMap có thể được mount làm volume dữ liệu. ConfigMap cũng có thể được các phần khác
của hệ thống sử dụng mà không cần expose trực tiếp cho Pod. Ví dụ, ConfigMap có thể chứa
dữ liệu mà các phần khác của hệ thống nên dùng để cấu hình.

Cách phổ biến nhất để dùng ConfigMap là cấu hình các thiết lập cho những container chạy
trong một Pod ở cùng namespace. Bạn cũng có thể dùng một ConfigMap một cách độc lập.

Ví dụ, bạn có thể gặp các addon hoặc operator điều chỉnh hành vi của chúng dựa trên một
ConfigMap.

### Dùng ConfigMap làm file trong một Pod (Using ConfigMaps as files from a Pod)

Để tiêu thụ một ConfigMap trong một volume của Pod:

1. Tạo một ConfigMap hoặc dùng một ConfigMap có sẵn. Nhiều Pod có thể tham chiếu cùng
   một ConfigMap.
1. Sửa định nghĩa Pod của bạn để thêm một volume dưới `.spec.volumes[]`. Đặt tên volume
   tùy ý, và đặt trường `.spec.volumes[].configMap.name` tham chiếu tới object ConfigMap
   của bạn.
1. Thêm `.spec.containers[].volumeMounts[]` vào mỗi container cần ConfigMap. Chỉ định
   `.spec.containers[].volumeMounts[].readOnly = true` và
   `.spec.containers[].volumeMounts[].mountPath` trỏ tới một tên thư mục chưa được dùng,
   nơi bạn muốn ConfigMap xuất hiện.
1. Sửa image hoặc dòng lệnh của bạn sao cho chương trình tìm các file trong thư mục đó.
   Mỗi key trong map `data` của ConfigMap trở thành tên file dưới `mountPath`.

Đây là một ví dụ về Pod mount một ConfigMap trong một volume:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
  - name: mypod
    image: redis
    volumeMounts:
    - name: foo
      mountPath: "/etc/foo"
      readOnly: true
  volumes:
  - name: foo
    configMap:
      name: myconfigmap
```

Mỗi ConfigMap bạn muốn dùng cần được tham chiếu trong `.spec.volumes`.

Nếu Pod có nhiều container, mỗi container cần khối `volumeMounts` riêng của nó, nhưng
với mỗi ConfigMap chỉ cần một `.spec.volumes`.

#### ConfigMap đã mount được cập nhật tự động (Mounted ConfigMaps are updated automatically)

Khi một ConfigMap đang được tiêu thụ trong một volume được cập nhật, các key được chiếu
(projected) rồi cũng sẽ được cập nhật theo. Kubelet kiểm tra xem ConfigMap đã mount có
còn mới hay không trong mỗi chu kỳ đồng bộ định kỳ. Tuy nhiên, kubelet dùng cache cục bộ
của nó để lấy giá trị hiện tại của ConfigMap. Loại cache có thể cấu hình được thông qua
trường `configMapAndSecretChangeDetectionStrategy` trong
[struct KubeletConfiguration](https://kubernetes.io/docs/reference/config-api/kubelet-config.v1beta1/).
Một ConfigMap có thể được lan truyền bằng watch (mặc định), dựa trên ttl, hoặc bằng cách
chuyển hướng mọi request trực tiếp tới API server. Kết quả là, tổng độ trễ từ thời điểm
ConfigMap được cập nhật tới thời điểm các key mới được chiếu vào Pod có thể dài bằng chu
kỳ đồng bộ của kubelet + độ trễ lan truyền cache, trong đó độ trễ lan truyền cache phụ
thuộc vào loại cache được chọn (tương ứng bằng độ trễ lan truyền watch, ttl của cache,
hoặc bằng không).

ConfigMap được tiêu thụ dưới dạng biến môi trường không được cập nhật tự động và đòi hỏi
phải khởi động lại pod.

> **Ghi chú:** Một container dùng ConfigMap qua volume mount kiểu
> [subPath](./91-volumes-vi.md#using-subpath) sẽ không nhận được các cập nhật của ConfigMap.

### Dùng ConfigMap làm biến môi trường (Using Configmaps as environment variables)

Để dùng một Configmap trong một biến môi trường của Pod:

1. Với mỗi container trong đặc tả Pod của bạn, thêm một biến môi trường cho từng key của
   Configmap mà bạn muốn dùng vào trường `env[].valueFrom.configMapKeyRef`.
1. Sửa image và/hoặc dòng lệnh của bạn sao cho chương trình tìm giá trị trong các biến
   môi trường đã chỉ định.

Đây là một ví dụ về việc định nghĩa một ConfigMap làm biến môi trường của pod:

ConfigMap sau đây (myconfigmap.yaml) lưu hai thuộc tính: username và access_level:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: myconfigmap
data:
  username: k8s-admin
  access_level: "1"
```

Lệnh sau sẽ tạo object ConfigMap:

```shell
kubectl apply -f myconfigmap.yaml
```

Pod sau đây tiêu thụ nội dung của ConfigMap dưới dạng các biến môi trường:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: env-configmap
spec:
  containers:
    - name: app
      command: ["/bin/sh", "-c", "printenv"]
      image: busybox:latest
      envFrom:
        - configMapRef:
            name: myconfigmap
```

Trường `envFrom` chỉ thị cho Kubernetes tạo các biến môi trường từ các nguồn được lồng
bên trong nó. `configMapRef` bên trong tham chiếu tới một ConfigMap theo tên và chọn tất
cả các cặp key-value của nó. Thêm Pod vào cluster của bạn, rồi lấy log của Pod để xem
output từ lệnh printenv. Điều này sẽ xác nhận rằng hai cặp key-value từ ConfigMap đã
được đặt làm biến môi trường:

```shell
kubectl apply -f env-configmap.yaml
```
```shell
kubectl logs pod/env-configmap
```
Output tương tự như sau:
```console
...
username: "k8s-admin"
access_level: "1"
...
```

Đôi khi một Pod không cần truy cập tất cả các giá trị trong một ConfigMap. Ví dụ, bạn
có thể có một Pod khác chỉ dùng giá trị username từ ConfigMap. Với trường hợp này, bạn
có thể dùng cú pháp `env.valueFrom` thay thế — cú pháp này cho phép bạn chọn từng key
riêng lẻ trong một ConfigMap. Tên của biến môi trường cũng có thể khác với key bên trong
ConfigMap. Ví dụ:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: env-configmap
spec:
  containers:
  - name: envars-test-container
    image: nginx
    env:
    - name: CONFIGMAP_USERNAME
      valueFrom:
        configMapKeyRef:
          name: myconfigmap
          key: username
```

Trong Pod được tạo từ manifest này, bạn sẽ thấy biến môi trường `CONFIGMAP_USERNAME`
được đặt bằng giá trị của `username` trong ConfigMap. Các key khác trong dữ liệu của
ConfigMap không được sao chép vào môi trường.

Điều quan trọng cần lưu ý là phạm vi ký tự được phép dùng cho tên biến môi trường trong
pod bị [hạn chế](https://kubernetes.io/docs/tasks/inject-data-application/define-environment-variable-container/#using-environment-variables-inside-of-your-config).
Nếu có key nào không đáp ứng các quy tắc đó, những key ấy sẽ không được cung cấp cho
container của bạn, mặc dù Pod vẫn được phép khởi động.

## ConfigMap bất biến (Immutable ConfigMaps) {#configmap-immutable}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.21 [stable]`

Tính năng _Immutable Secrets and ConfigMaps_ (Secret và ConfigMap bất biến) của
Kubernetes cung cấp tùy chọn đặt từng Secret và ConfigMap riêng lẻ thành bất biến. Với
các cluster sử dụng ConfigMap rộng rãi (ít nhất hàng chục nghìn lượt mount ConfigMap vào
Pod khác nhau), việc ngăn thay đổi dữ liệu của chúng mang lại các lợi ích sau:

- bảo vệ bạn khỏi các cập nhật vô tình (hoặc không mong muốn) có thể gây gián đoạn ứng dụng
- cải thiện hiệu năng của cluster bằng cách giảm đáng kể tải lên kube-apiserver, nhờ
  đóng các watch đối với những ConfigMap được đánh dấu là bất biến.

Bạn có thể tạo một ConfigMap bất biến bằng cách đặt trường `immutable` thành `true`.
Ví dụ:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  ...
data:
  ...
immutable: true
```

Một khi ConfigMap đã được đánh dấu là bất biến, _không_ thể hoàn tác thay đổi này, cũng
không thể thay đổi nội dung của trường `data` hay `binaryData`. Bạn chỉ có thể xóa và
tạo lại ConfigMap. Vì các Pod hiện có vẫn giữ mount point trỏ tới ConfigMap đã bị xóa,
nên khuyến nghị tạo lại các pod này.

## Tiếp theo (What's next)

* Đọc về [Secret](https://kubernetes.io/docs/concepts/configuration/secret/).
* Đọc [Cấu hình một Pod để sử dụng ConfigMap](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/).
* Đọc về [thay đổi một ConfigMap (hoặc bất kỳ object Kubernetes nào khác)](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/update-api-object-kubectl-patch/).
* Đọc [The Twelve-Factor App](https://12factor.net/) để hiểu động lực của việc tách mã
  nguồn khỏi cấu hình.

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 3:

1. Bạn sửa giá trị một key trong ConfigMap `game-demo`. Pod A mount ConfigMap đó làm volume,
   Pod B nạp nó bằng `envFrom`. Sau vài phút, Pod nào thấy giá trị mới, Pod nào không, và
   phải làm gì với Pod không thấy?
2. Trong ví dụ `game-demo`, ConfigMap có bốn key nhưng container `demo` chỉ thấy hai file
   trong `/config`. Vì sao? Sửa gì để có đủ bốn file?
3. Trên cluster lab, bạn tạo ConfigMap trong namespace `default` rồi tạo một Pod ở namespace
   khác, Pod này chạy trên `k8s-worker2` và mount ConfigMap đó theo tên. Kết quả thế nào?
   Có cách nào để một Pod đọc được ConfigMap ở namespace khác không?
4. `binaryData` lưu giá trị dưới dạng chuỗi base64. Vậy đặt mật khẩu vào `binaryData` thì
   ConfigMap có giấu được mật khẩu không?
5. Bạn đã đặt `immutable: true` cho một ConfigMap, giờ cần đổi một giá trị trong `data`.
   Làm thế nào, và phải lưu ý gì với các Pod đang mount nó?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Pod A thấy, Pod B không.** ConfigMap được tiêu thụ trong volume thì các key đã chiếu
   (project) cũng được cập nhật theo — kubelet kiểm tra độ mới trong mỗi chu kỳ đồng bộ định
   kỳ, nên có độ trễ bằng chu kỳ đồng bộ của kubelet cộng độ trễ lan truyền cache. Còn
   "ConfigMap được tiêu thụ dưới dạng biến môi trường **không được cập nhật tự động và đòi
   hỏi phải khởi động lại pod**". Với Pod B phải khởi động lại Pod. (Ngoại lệ cùng chiều:
   container mount kiểu `subPath` cũng không nhận cập nhật.)
2. Vì định nghĩa Pod chỉ định mảng **`items`** trong phần `volumes`, và mảng đó chỉ liệt kê
   hai key `game.properties` và `user-interface.properties`. **Bỏ hẳn mảng `items`** thì mỗi
   key trong ConfigMap trở thành một file có tên trùng với key, và bạn nhận được đủ bốn file.
3. **Không mount được: Pod và ConfigMap phải nằm trong cùng một namespace.** Tham chiếu
   `.spec.volumes[].configMap.name` chỉ tìm trong namespace của chính Pod, nên kubelet trên
   `k8s-worker2` không có gì để chiếu vào container. Cách duy nhất bài nêu để đọc ConfigMap ở
   namespace khác là **cách thứ tư**: viết mã trong Pod dùng trực tiếp Kubernetes API — và khi
   đó việc Pod có được phép đọc hay không lại là chuyện phân quyền, thuộc giai đoạn 9.
4. **Không.** base64 chỉ là cách biểu diễn dữ liệu nhị phân dưới dạng chuỗi, không phải mã
   hóa. Bài nói thẳng ngay đầu: "ConfigMap không cung cấp khả năng giữ bí mật hay mã hóa. Nếu
   dữ liệu bạn muốn lưu là dữ liệu mật, hãy dùng Secret thay vì ConfigMap." Trực giác "nhìn
   không đọc được thì là bí mật" sai ở đây — và sẽ còn sai đúng y như vậy ở bài Secret.
5. **Không sửa được — phải xóa và tạo lại ConfigMap.** Một khi đã đánh dấu bất biến thì không
   thể hoàn tác thay đổi đó, cũng không thể sửa `data` hay `binaryData`. Lưu ý: các Pod hiện
   có vẫn giữ mount point trỏ tới ConfigMap đã bị xóa, nên **khuyến nghị tạo lại các Pod đó**.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
