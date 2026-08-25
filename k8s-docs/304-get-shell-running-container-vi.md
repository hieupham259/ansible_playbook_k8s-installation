# Truy cập shell của một container đang chạy (Get a Shell to a Running Container)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/debug/debug-application/get-shell-running-container/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 11 — Observability](00-ALO-TRINH-ADMIN.md#giai-đoạn-11--observability)
→ dòng **Thực hành**, bài 3/7 · Kiểm chứng ở
[Lab 11a — Observability](labs/LAB-11A-OBSERVABILITY.md) phần B11.1 (mở shell), B11.2 (ghi trang
gốc rồi tự gọi từ bên trong), B11.3 (chạy lệnh rời và vai trò của dấu `--`) và B11.4 (Pod nhiều
container).

Đây là bài thao tác duy nhất của nhánh *Gỡ lỗi ứng dụng* mà giai đoạn 11 làm trọn vẹn. Nó ngắn
và đi thẳng vào lệnh; phần đáng nhớ là **ranh giới** giữa shell tương tác và lệnh rời, chứ không
phải danh sách lệnh gõ thử bên trong container.

**Phải hiểu ở lần đọc này:**

- Lệnh mở shell tương tác là `kubectl exec --stdin --tty <pod> -- /bin/bash`; `-i` và `-t` là
  dạng ngắn của `--stdin` và `--tty` (ghi chú ở mục *Mở shell khi Pod có nhiều hơn một
  container*). Thoát bằng `exit`.
- Vai trò của dấu `--`: nó **phân tách các đối số dành cho lệnh chạy trong container** khỏi các
  đối số của chính `kubectl`.
- Mục *Ghi trang gốc cho nginx* chứng minh shell đứng **bên trong** container: file ghi vào
  `/usr/share/nginx/html` — chính là chỗ volume `emptyDir` `shared-data` được mount — trở thành
  trang gốc mà nginx phục vụ, kiểm lại bằng `curl http://localhost/` chạy từ trong container.
- Mục *Chạy từng lệnh riêng lẻ trong container*: không phải lúc nào cũng cần shell tương tác —
  `kubectl exec shell-demo -- env`, `-- ps aux`, `-- ls /` chạy một lệnh rồi trả kết quả về
  terminal thường của bạn.
- Mục *Mở shell khi Pod có nhiều hơn một container*: Pod nhiều container thì phải chỉ đích danh
  bằng `--container` hoặc `-c`, ví dụ `kubectl exec -i -t my-pod --container main-app -- /bin/bash`.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Mục *Trước khi bạn bắt đầu* — minikube và các playground | lộ trình không dùng minikube; cluster để chạy bài này đã có sẵn | [Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md) đã dựng cluster ba VM |
| Các lệnh `apt-get update` và `apt-get install -y tcpdump / lsof / procps` trong ví dụ | cài công cụ vào container ứng dụng đang chạy là thao tác chẩn đoán nâng cao và cần mạng ra ngoài từ trong Pod | [giai đoạn 24](00-ALO-TRINH-ADMIN.md#giai-đoạn-24--xử-lý-sự-cố), bài [300](300-debug-running-pod-vi.md) — nơi dùng ephemeral container thay cho việc cài thêm vào container ứng dụng |
| `hostNetwork: true` và `dnsPolicy: Default` trong manifest `shell-demo` | bài không giải thích vì sao ví dụ chọn hai giá trị này; chúng không liên quan tới cơ chế `exec` | đã học ở [giai đoạn 5](00-ALO-TRINH-ADMIN.md#giai-đoạn-5--mạng-nền-tảng), bài [10](10-dns-pod-service-vi.md) cho `dnsPolicy` |

---

Trang này hướng dẫn cách dùng `kubectl exec` để truy cập shell của một container đang chạy.

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao tiếp
với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node không đóng
vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster bằng
[minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong các sân
chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

## Truy cập shell của một container (Getting a shell to a container)

Trong bài thực hành này, bạn tạo một Pod có một container. Container chạy image nginx. Đây là
file cấu hình của Pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: shell-demo
spec:
  volumes:
  - name: shared-data
    emptyDir: {}
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - name: shared-data
      mountPath: /usr/share/nginx/html
  hostNetwork: true
  dnsPolicy: Default
```

Tạo Pod:

```shell
kubectl apply -f https://k8s.io/examples/application/shell-demo.yaml
```

Xác nhận rằng container đang chạy:

```shell
kubectl get pod shell-demo
```

Truy cập shell của container đang chạy:

```shell
kubectl exec --stdin --tty shell-demo -- /bin/bash
```

> **Ghi chú:**
> Dấu gạch ngang kép (`--`) phân tách các đối số bạn muốn truyền cho lệnh khỏi các đối số của
> kubectl.

Trong shell của bạn, liệt kê thư mục gốc:

```shell
# Chạy lệnh này bên trong container
ls /
```

Trong shell của bạn, hãy thử nghiệm với các lệnh khác. Dưới đây là một vài ví dụ:

```shell
# Bạn có thể chạy các lệnh ví dụ này bên trong container
ls /
cat /proc/mounts
cat /proc/1/maps
apt-get update
apt-get install -y tcpdump
tcpdump
apt-get install -y lsof
lsof
apt-get install -y procps
ps aux
ps aux | grep nginx
```

## Ghi trang gốc cho nginx (Writing the root page for nginx)

Hãy xem lại file cấu hình của Pod. Pod có một volume `emptyDir`, và container mount volume này
tại `/usr/share/nginx/html`.

Trong shell của bạn, tạo một file `index.html` trong thư mục `/usr/share/nginx/html`:

```shell
# Chạy lệnh này bên trong container
echo 'Hello shell demo' > /usr/share/nginx/html/index.html
```

Trong shell của bạn, gửi một request GET đến nginx server:

```shell
# Chạy lệnh này trong shell bên trong container của bạn
apt-get update
apt-get install curl
curl http://localhost/
```

Kết quả đầu ra hiển thị đoạn văn bản mà bạn đã ghi vào file `index.html`:

```
Hello shell demo
```

Khi bạn dùng xong shell, hãy nhập `exit`.

```shell
exit # Để thoát khỏi shell trong container
```

## Chạy từng lệnh riêng lẻ trong container (Running individual commands in a container)

Trong một cửa sổ lệnh thông thường (không phải shell của bạn trong container), liệt kê các
biến môi trường trong container đang chạy:

```shell
kubectl exec shell-demo -- env
```

Hãy thử nghiệm chạy các lệnh khác. Dưới đây là một vài ví dụ:

```shell
kubectl exec shell-demo -- ps aux
kubectl exec shell-demo -- ls /
kubectl exec shell-demo -- cat /proc/1/mounts
```

## Mở shell khi Pod có nhiều hơn một container (Opening a shell when a Pod has more than one container)

Nếu một Pod có nhiều hơn một container, hãy dùng `--container` hoặc `-c` để chỉ định container
trong lệnh `kubectl exec`. Ví dụ, giả sử bạn có một Pod tên là my-pod, và Pod này có hai
container tên là _main-app_ và _helper-app_. Lệnh sau sẽ mở một shell tới container _main-app_.

```shell
kubectl exec -i -t my-pod --container main-app -- /bin/bash
```

> **Ghi chú:**
> Các tùy chọn ngắn `-i` và `-t` tương đương với các tùy chọn dài `--stdin` và `--tty`.

## Tiếp theo (What's next)

* Đọc về [kubectl exec](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands/#exec)

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 11:

1. Trong `kubectl exec shell-demo -- env`, dấu `--` phân tách cái gì với cái gì?
2. **Câu bẫy.** So `kubectl exec --stdin --tty shell-demo -- /bin/bash` với
   `kubectl exec shell-demo -- ps aux`. Có phải lệnh nào cũng cần `--stdin --tty` không, và hai
   cờ đó thật ra làm gì?
3. Pod `shell-demo` đang chạy trên `lab-k8s-worker2`. Bạn vào shell, `echo 'Hello shell demo' >
   /usr/share/nginx/html/index.html`, rồi `curl http://localhost/` **từ bên trong container** và
   thấy đúng chuỗi đó. Kết quả này chứng minh điều gì về quan hệ giữa shell của bạn và container
   nginx? Vì sao đúng thư mục đó?
4. Pod `my-pod` có hai container `main-app` và `helper-app`. Viết lệnh mở shell vào `main-app`,
   và nói vì sao ở đây không thể bỏ qua cờ chỉ định container.

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Nó phân tách **các đối số bạn muốn truyền cho lệnh chạy trong container** khỏi **các đối số
   của chính `kubectl`**. Trong ví dụ, `env` và mọi thứ đứng sau `--` thuộc về lệnh chạy bên
   trong container, không phải cờ của `kubectl exec`.
2. **Không.** Đây là chỗ dễ nhầm vì `-it` bị gõ theo thói quen. `--stdin` (`-i`) và `--tty`
   (`-t`) chỉ cần khi bạn muốn một **phiên tương tác** — nối stdin của bạn vào container và cấp
   một tty, tức đúng trường hợp mở `/bin/bash`. Với **lệnh rời** như `-- ps aux`, `-- env`,
   `-- ls /` thì không cần: bài chạy chúng "trong một cửa sổ lệnh thông thường, không phải shell
   trong container", kubectl chạy lệnh rồi trả kết quả ra terminal của bạn.
3. Nó chứng minh **shell của bạn đang chạy ngay bên trong container đó, nhìn thấy đúng filesystem
   của nó**. Đúng thư mục là vì Pod khai một volume `emptyDir` tên `shared-data` và container
   nginx **mount volume đó tại `/usr/share/nginx/html`** — chính là thư mục nginx lấy trang gốc.
   Ghi file vào đó từ shell là ghi vào đúng chỗ nginx đọc, nên `curl` trả về nội dung bạn vừa
   viết.
4. `kubectl exec -i -t my-pod --container main-app -- /bin/bash` (hoặc `-c main-app`). **Không bỏ
   qua được** vì Pod có nhiều hơn một container: bài nói rõ khi đó phải dùng `--container` hoặc
   `-c` để chỉ định container trong lệnh `kubectl exec` — không chỉ định thì lệnh không xác định
   được đích là `main-app` hay `helper-app`.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
