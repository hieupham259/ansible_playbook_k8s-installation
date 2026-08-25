# Giao tiếp giữa các Container trong cùng Pod bằng Volume dùng chung (Communicate Between Containers in the Same Pod Using a Shared Volume)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/access-application-cluster/communicate-containers-same-pod-shared-volume/>
>
> Trang này hướng dẫn cách dùng một Volume để giao tiếp giữa hai Container chạy trong cùng một Pod.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 3 — Pod và cấu hình](00-ALO-TRINH-ADMIN.md#giai-đoạn-3--pod-và-cấu-hình)
→ nhóm [3a. Pod và vòng đời](00-ALO-TRINH-ADMIN.md#3a-pod-và-vòng-đời), bài thực hành 11/11 —
**bài cuối của nhóm** · Kiểm chứng ở [Lab 3a](labs/LAB-3A-POD-VA-VONG-DOI.md) phần B1.2.

Bài đóng lại nhóm 3a bằng câu hỏi nền nhất: **vì sao một Pod lại có nhiều container**. Nó là cặp
song sinh của bài [292](292-share-process-namespace-vi.md) — cùng một mục tiêu "hai container nói
chuyện với nhau", nhưng qua **filesystem dùng chung** thay vì qua process namespace.

**Phải hiểu ở lần đọc này:**

- Cơ chế: một Volume khai báo ở **`.spec.volumes`** (ở đây là `emptyDir` tên `shared-data`), rồi
  **cả hai container cùng mount nó** ở hai `mountPath` khác nhau — `/usr/share/nginx/html` cho
  nginx và `/pod-data` cho debian. Thứ nối hai đường dẫn lại là **tên volume**.
- Hai container trong một Pod **không buộc phải cùng vòng đời**: container debian ghi
  `index.html` rồi **kết thúc**, container nginx vẫn chạy và phục vụ đúng file đó.
  `kubectl get pod two-containers --output=yaml` cho thấy một container ở trạng thái `terminated`
  còn một ở `running` **trong cùng một Pod**.
- Giới hạn về dữ liệu, bài nói thẳng ở mục *Thảo luận*: Volume này chỉ cho phép giao tiếp **trong
  suốt vòng đời của Pod**. **Pod bị xóa và được tạo lại thì mọi dữ liệu trong Volume dùng chung
  bị mất.**
- Lý do Pod có nhiều container: hỗ trợ **ứng dụng phụ trợ (helper application)** cho một ứng dụng
  chính — bài kể data puller, data pusher, proxy. Hai đường giao tiếp điển hình là **hệ thống
  file dùng chung** (bài này) hoặc **giao diện mạng loopback, localhost**.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| `restartPolicy: Never` trong manifest | `restartPolicy` là một chủ đề riêng của chính nhóm này | bài [47](47-pod-lifecycle-vi.md), thực hành ở [Lab 3a](labs/LAB-3A-POD-VA-VONG-DOI.md) phần B3 |
| `containerID: docker://...` trong đầu ra ví dụ | trang gốc còn dùng cluster chạy Docker; runtime của cluster lab khác | [giai đoạn 2](00-ALO-TRINH-ADMIN.md#giai-đoạn-2--container-và-runtime) đã học, đối chiếu ở [Lab 2](labs/LAB-2-CONTAINER-IMAGE-CRI-VA-CGROUP.md) |
| `apt-get update && apt-get install curl procps` bên trong container nginx | là cách trang gốc xem kết quả, không phải cơ chế cần học | [Lab 3a](labs/LAB-3A-POD-VA-VONG-DOI.md) phần B1.2 kiểm bằng gate so sánh nội dung |
| Mục *Tiếp theo*: các bài blog/slide về composite container, bài *Cấu hình Pod dùng Volume để lưu trữ*, API reference `Volume` | Volume là chủ đề riêng, chưa học | bài [91](91-volumes-vi.md) ở [giai đoạn 6](00-ALO-TRINH-ADMIN.md#giai-đoạn-6--lưu-trữ) |

---

Trang này hướng dẫn cách dùng một Volume để giao tiếp giữa hai Container chạy
trong cùng một Pod. Xem thêm cách cho phép các tiến trình giao tiếp bằng việc
[chia sẻ process namespace](292-share-process-namespace-vi.md)
giữa các container.

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình
để giao tiếp với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít
nhất hai node không đóng vai trò control plane host. Nếu bạn chưa có cluster, bạn có
thể tạo một cluster bằng [minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/)
hoặc dùng một trong các sân chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

Để kiểm tra phiên bản, hãy chạy `kubectl version`.

## Tạo một Pod chạy hai Container (Creating a Pod that runs two Containers)

Trong bài thực hành này, bạn tạo một Pod chạy hai Container. Hai container
chia sẻ một Volume mà chúng có thể dùng để giao tiếp với nhau. Đây là file cấu hình
cho Pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: two-containers
spec:

  restartPolicy: Never

  volumes:
  - name: shared-data
    emptyDir: {}

  containers:

  - name: nginx-container
    image: nginx
    volumeMounts:
    - name: shared-data
      mountPath: /usr/share/nginx/html

  - name: debian-container
    image: debian
    volumeMounts:
    - name: shared-data
      mountPath: /pod-data
    command: ["/bin/sh"]
    args: ["-c", "echo Hello from the debian container > /pod-data/index.html"]
```

Trong file cấu hình, bạn có thể thấy Pod có một Volume tên là
`shared-data`.

Container đầu tiên được liệt kê trong file cấu hình chạy một nginx server.
Đường dẫn mount (mount path) cho Volume dùng chung là `/usr/share/nginx/html`.
Container thứ hai dựa trên image debian, và có đường dẫn mount là
`/pod-data`. Container thứ hai chạy lệnh sau rồi kết thúc.

    echo Hello from the debian container > /pod-data/index.html

Lưu ý rằng container thứ hai ghi file `index.html` vào thư mục gốc
của nginx server.

Tạo Pod cùng hai Container:

    kubectl apply -f https://k8s.io/examples/pods/two-container-pod.yaml

Xem thông tin về Pod và các Container:

    kubectl get pod two-containers --output=yaml

Đây là một phần của kết quả:

    apiVersion: v1
    kind: Pod
    metadata:
      ...
      name: two-containers
      namespace: default
      ...
    spec:
      ...
      containerStatuses:

      - containerID: docker://c1d8abd1 ...
        image: debian
        ...
        lastState:
          terminated:
            ...
        name: debian-container
        ...

      - containerID: docker://96c1ff2c5bb ...
        image: nginx
        ...
        name: nginx-container
        ...
        state:
          running:
        ...

Bạn có thể thấy Container debian đã kết thúc (terminated), còn Container nginx
vẫn đang chạy.

Mở một shell vào Container nginx:

    kubectl exec -it two-containers -c nginx-container -- /bin/bash

Trong shell của bạn, xác minh rằng nginx đang chạy:

    root@two-containers:/# apt-get update
    root@two-containers:/# apt-get install curl procps
    root@two-containers:/# ps aux

Kết quả tương tự như sau:

    USER       PID  ...  STAT START   TIME COMMAND
    root         1  ...  Ss   21:12   0:00 nginx: master process nginx -g daemon off;

Hãy nhớ rằng Container debian đã tạo file `index.html` trong thư mục gốc
của nginx. Dùng `curl` để gửi một request GET tới nginx server:

```
root@two-containers:/# curl localhost
```

Kết quả cho thấy nginx phục vụ trang web do container debian viết ra:

```
Hello from the debian container
```

## Thảo luận (Discussion)

Lý do chính khiến Pod có thể có nhiều container là để hỗ trợ
các ứng dụng phụ trợ (helper application) trợ giúp cho một ứng dụng chính. Các ví dụ điển hình của
ứng dụng phụ trợ là bộ kéo dữ liệu (data puller), bộ đẩy dữ liệu (data pusher), và proxy.
Ứng dụng phụ trợ và ứng dụng chính thường cần giao tiếp với nhau.
Thông thường, việc này được thực hiện qua một hệ thống file dùng chung, như minh họa
trong bài thực hành này, hoặc qua giao diện mạng loopback, localhost. Một ví dụ của mẫu hình này là
một web server đi cùng một chương trình phụ trợ định kỳ kiểm tra một Git repository để lấy các cập nhật mới.

Volume trong bài thực hành này cung cấp một cách để các Container giao tiếp trong
suốt vòng đời của Pod. Nếu Pod bị xóa và được tạo lại, mọi dữ liệu lưu trong
Volume dùng chung sẽ bị mất.

## Tiếp theo (What's next)

* Tìm hiểu thêm về [các mẫu hình cho composite container](https://kubernetes.io/blog/2015/06/the-distributed-system-toolkit-patterns/).

* Tìm hiểu về [composite container cho kiến trúc dạng module](https://www.slideshare.net/Docker/slideshare-burns).

* Xem [Cấu hình một Pod dùng Volume để lưu trữ](280-configure-volume-storage-vi.md).

* Xem [Cấu hình một Pod chia sẻ process namespace giữa các container trong Pod](292-share-process-namespace-vi.md)

* Xem [Volume](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.36/#volume-v1-core).

* Xem [Pod](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.36/#pod-v1-core).

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 3:

1. Container debian ghi vào `/pod-data`, container nginx đọc ở `/usr/share/nginx/html`. Hai đường
   dẫn khác hẳn nhau — cái gì trong manifest nối chúng lại?
2. **Câu bẫy.** `kubectl get pod two-containers --output=yaml` cho thấy container debian đã
   `terminated` trong khi container nginx vẫn `running`. Đó có phải dấu hiệu Pod hỏng không? Và
   vì sao file `index.html` không mất theo container đã chết?
3. Trên cluster lab, Pod `two-containers` chạy trên `lab-k8s-worker1`. Bạn xóa Pod rồi apply lại
   đúng manifest cũ, `curl localhost` trong nginx vẫn trả `Hello from the debian container`. Điều
   đó có chứng minh Volume dùng chung giữ được dữ liệu qua các lần tạo lại Pod không?
4. Bài nêu vì sao một Pod lại có nhiều container. Lý do đó là gì, và bài kể hai đường giao tiếp
   nào giữa ứng dụng chính với ứng dụng phụ trợ?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Tên volume `shared-data`.** Nó được khai báo đúng một lần ở `.spec.volumes` dưới dạng
   `emptyDir`, rồi mỗi container tham chiếu tới nó bằng `volumeMounts` với `name: shared-data`.
   `mountPath` chỉ nói volume đó **xuất hiện ở đâu bên trong từng container** — nên hai container
   nhìn cùng một nội dung ở hai đường dẫn khác nhau là chuyện bình thường, và đó chính là cách
   debian ghi `index.html` vào đúng thư mục gốc của nginx.
2. **Không phải Pod hỏng.** Container debian chạy đúng một lệnh — `echo ... > /pod-data/index.html`
   — rồi kết thúc theo thiết kế; bài nói thẳng "Bạn có thể thấy Container debian đã kết thúc, còn
   Container nginx vẫn đang chạy". Chỗ trực giác sai là nghĩ mọi container trong Pod phải cùng
   sống cùng chết. File không mất vì nó nằm trên **Volume của Pod**, không nằm trong lớp ghi của
   container debian — Volume là thứ tồn tại độc lập với từng container và được cả hai cùng mount.
3. **Không.** Kết quả giống nhau là do container debian **chạy lại và ghi lại** file mỗi lần Pod
   khởi động, chứ không phải do dữ liệu cũ còn đó. Mục *Thảo luận* nói rõ: Volume trong bài này
   chỉ cung cấp cách giao tiếp **trong suốt vòng đời của Pod**, và **nếu Pod bị xóa rồi được tạo
   lại, mọi dữ liệu lưu trong Volume dùng chung sẽ bị mất**.
4. Lý do chính là **hỗ trợ các ứng dụng phụ trợ (helper application) trợ giúp cho một ứng dụng
   chính** — bài lấy ví dụ data puller, data pusher và proxy, cùng mẫu hình web server đi kèm một
   chương trình định kỳ kiểm tra Git repository để lấy cập nhật. Hai đường giao tiếp: qua **một
   hệ thống file dùng chung** như bài này minh họa, hoặc qua **giao diện mạng loopback,
   localhost**.

</details>

Bạn vừa đọc xong bài cuối của nhóm 3a. Câu nào chưa trả lời được thì quay lại đúng mục tương ứng,
rồi mở [Lab 3a — Pod và vòng đời](labs/LAB-3A-POD-VA-VONG-DOI.md) và bắt đầu từ phần B0.
