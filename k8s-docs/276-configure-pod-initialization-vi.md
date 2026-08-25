# Cấu hình khởi tạo Pod (Configure Pod Initialization)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-initialization/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 3 — Pod và cấu hình](00-ALO-TRINH-ADMIN.md#giai-đoạn-3--pod-và-cấu-hình)
→ nhóm [3a. Pod và vòng đời](00-ALO-TRINH-ADMIN.md#3a-pod-và-vòng-đời), bài thực hành 5/11 ·
Kiểm chứng ở [Lab 3a](labs/LAB-3A-POD-VA-VONG-DOI.md) phần B5.1.

Bài rất ngắn và chỉ trình bày **một mẫu hình duy nhất**: init container chuẩn bị dữ liệu, app
container dùng dữ liệu đó. Lý thuyết về init container nằm ở bài [50](50-init-containers-vi.md);
ở đây chỉ cần bám lấy cơ chế bàn giao.

**Phải hiểu ở lần đọc này:**

- Init container **chạy đến khi hoàn tất trước khi container ứng dụng khởi động** — đó là toàn bộ
  ý của bài.
- `initContainers` là một mảng **riêng, ngang cấp với `containers`** trong `.spec`, không phải một
  mục con của `containers`.
- Cơ chế bàn giao dữ liệu là **một Volume dùng chung**: manifest khai báo `emptyDir` tên
  `workdir`; init container mount nó ở `/work-dir`, app container mount **cùng volume đó** ở
  `/usr/share/nginx/html`. Init container `wget` file `index.html` vào volume rồi kết thúc, nginx
  phục vụ đúng file đó.
- Điểm dễ nhầm nằm ngay trong ví dụ: hai container mount **cùng một volume ở hai đường dẫn khác
  nhau**. Thứ nối chúng lại là **tên volume** trong `volumeMounts`, không phải `mountPath`.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| `dnsPolicy: Default` trong manifest | chính sách DNS của Pod là chủ đề riêng, không liên quan tới init container | bài [10](10-dns-pod-service-vi.md) ở [giai đoạn 5](00-ALO-TRINH-ADMIN.md#giai-đoạn-5--mạng-nền-tảng) |
| Việc init container `wget http://info.cern.ch` — tức cần internet | môi trường lab có thể không ra được internet, và điểm cần học không phải là `wget` | [Lab 3a](labs/LAB-3A-POD-VA-VONG-DOI.md) phần B5.1 dựng lại cùng mẫu hình bằng lệnh cục bộ |
| `apt-get update && apt-get install curl` bên trong container nginx | là cách trang gốc xem kết quả, không phải cơ chế | [Lab 3a](labs/LAB-3A-POD-VA-VONG-DOI.md) phần B5.1 kiểm bằng gate so sánh nội dung file |
| Mục *Tiếp theo* trỏ sang bài Volume và bài Gỡ lỗi Init Container | Volume là chủ đề riêng; gỡ lỗi thuộc giai đoạn xử lý sự cố | bài [91](91-volumes-vi.md) ở [giai đoạn 6](00-ALO-TRINH-ADMIN.md#giai-đoạn-6--lưu-trữ), bài [298](298-debug-init-containers-vi.md) ở [giai đoạn 24](00-ALO-TRINH-ADMIN.md#giai-đoạn-24--xử-lý-sự-cố) |

---

Trang này hướng dẫn cách sử dụng một Init Container để khởi tạo một Pod trước khi container
ứng dụng chạy.

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao tiếp
với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node không đóng
vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster bằng
[minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong các sân
chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

Để kiểm tra phiên bản, hãy nhập `kubectl version`.

## Tạo một Pod có Init Container (Create a Pod that has an Init Container) {#create-a-pod-that-has-an-init-container}

Trong bài thực hành này, bạn tạo một Pod có một container ứng dụng và một Init Container.
Init container chạy đến khi hoàn tất trước khi container ứng dụng khởi động.

Đây là file cấu hình của Pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: init-demo
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
    volumeMounts:
    - name: workdir
      mountPath: /usr/share/nginx/html
  # Các container này được chạy trong quá trình khởi tạo Pod
  initContainers:
  - name: install
    image: busybox:1.28
    command:
    - wget
    - "-O"
    - "/work-dir/index.html"
    - http://info.cern.ch
    volumeMounts:
    - name: workdir
      mountPath: "/work-dir"
  dnsPolicy: Default
  volumes:
  - name: workdir
    emptyDir: {}
```

Trong file cấu hình, bạn có thể thấy Pod có một Volume mà init container và container ứng dụng
cùng chia sẻ.

Init container mount Volume dùng chung này tại `/work-dir`, còn container ứng dụng mount Volume
dùng chung này tại `/usr/share/nginx/html`. Init container chạy lệnh sau rồi kết thúc:

```shell
wget -O /work-dir/index.html http://info.cern.ch
```

Lưu ý rằng init container ghi file `index.html` vào thư mục gốc của nginx server.

Tạo Pod:

```shell
kubectl apply -f https://k8s.io/examples/pods/init-containers.yaml
```

Xác minh rằng container nginx đang chạy:

```shell
kubectl get pod init-demo
```

Kết quả cho thấy container nginx đang chạy:

```
NAME        READY     STATUS    RESTARTS   AGE
init-demo   1/1       Running   0          1m
```

Mở một shell vào container nginx đang chạy trong Pod init-demo:

```shell
kubectl exec -it init-demo -- /bin/bash
```

Trong shell của bạn, gửi một GET request đến nginx server:

```
root@nginx:~# apt-get update
root@nginx:~# apt-get install curl
root@nginx:~# curl localhost
```

Kết quả cho thấy nginx đang phục vụ trang web đã được init container ghi vào:

```html
<html><head></head><body><header>
<title>http://info.cern.ch</title>
</header>

<h1>http://info.cern.ch - home of the first website</h1>
  ...
  <li><a href="http://info.cern.ch/hypertext/WWW/TheProject.html">Browse the first website</a></li>
  ...
```

## Tiếp theo (What's next)

* Tìm hiểu thêm về
  [giao tiếp giữa các container chạy trong cùng một Pod](./360-containers-shared-volume-vi.md).
* Tìm hiểu thêm về [Init Container](./50-init-containers-vi.md).
* Tìm hiểu thêm về [Volume](./91-volumes-vi.md).
* Tìm hiểu thêm về [Gỡ lỗi Init Container](./298-debug-init-containers-vi.md)

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 3:

1. Init container ghi file vào `/work-dir`, còn nginx đọc file ở `/usr/share/nginx/html`. Hai
   đường dẫn khác nhau — cái gì nối chúng lại với nhau trong manifest?
2. **Câu bẫy.** Init container `install` đã chạy xong và kết thúc **trước khi** nginx khởi động.
   Vậy file `index.html` nó ghi ra có biến mất theo nó không? Dựa vào đâu trong bài mà bạn kết
   luận được?
3. Bạn apply manifest này lên cluster lab và Pod được lập lịch lên `lab-k8s-worker1`. Trong lúc
   init container còn đang chạy, `kubectl get pod init-demo` **chưa** hiện `1/1 Running`. Con số
   `1/1` khi Pod đã sẵn sàng đang đếm cái gì?
4. `initContainers` nằm ở đâu trong `.spec`, và nếu bạn đặt nhầm nó thành một mục con của
   `containers` thì mẫu hình trong bài còn đúng không?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Tên volume.** Cả hai container đều có một mục `volumeMounts` trỏ tới cùng một volume tên
   `workdir` — thứ được khai báo một lần duy nhất ở `.spec.volumes` dưới dạng `emptyDir`.
   `mountPath` chỉ nói **volume đó xuất hiện ở đâu bên trong từng container**, nên hai container
   thấy cùng một nội dung ở hai đường dẫn khác nhau là chuyện bình thường. Bài mô tả đúng điều
   này: "Pod có một Volume mà init container và container ứng dụng cùng chia sẻ".
2. **Không biến mất.** Volume `workdir` là **Volume của Pod**, được "init container và container
   ứng dụng cùng chia sẻ" như bài mô tả — nó không thuộc về init container, nên nội dung ở lại sau
   khi init container kết thúc. Chỗ dễ sai là gắn tuổi thọ dữ liệu với tuổi thọ của container đã
   ghi ra nó. Bằng chứng nằm ngay ở kết quả cuối bài: `curl localhost` trong container nginx trả
   về đúng trang `info.cern.ch` mà init container đã `wget` về — nếu file mất theo init container
   thì nginx đã không có gì để phục vụ.
3. `1/1` đếm **container ứng dụng**, không đếm init container. Bài viết rõ init container "chạy
   đến khi hoàn tất **trước khi** container ứng dụng khởi động", nên trong lúc init còn chạy thì
   container nginx **chưa được tạo**; chỉ khi init xong, nginx mới khởi động và `kubectl get pod`
   mới hiện `1/1 Running`.
4. `initContainers` là **một mảng riêng nằm thẳng trong `.spec`, ngang cấp với `containers`** —
   trong manifest nó nằm ngay sau khối `containers`, cùng mức thụt lề. Nếu đặt nó vào trong
   `containers` thì đó không còn là init container nữa: mất luôn cái bảo đảm "chạy đến khi hoàn
   tất trước khi container ứng dụng khởi động", tức mất đúng thứ mà bài này dạy.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
