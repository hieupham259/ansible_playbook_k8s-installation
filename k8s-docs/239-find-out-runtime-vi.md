# Tìm hiểu Container Runtime nào đang được dùng trên Node (Find Out What Container Runtime is Used on a Node)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/administer-cluster/migrating-from-dockershim/find-out-runtime-you-use/

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:**
[Phần II — Vận hành cluster](00-ALO-TRINH-ADMIN.md#phần-ii--vận-hành-cluster)
→ [Giai đoạn 27 — Di chuyển khỏi dockershim (cluster cũ)](00-ALO-TRINH-ADMIN.md#giai-đoạn-27--di-chuyển-khỏi-dockershim-cluster-cũ),
bài 2/6 · Phần II không có lab riêng: tự chấm bằng **Checkpoint** ghi ở cuối mục giai đoạn 27 trong
lộ trình. Đây là **bài duy nhất của giai đoạn 27 chạy được nguyên vẹn trên cluster lab** — chính nó
là vế "xác định được container runtime đang dùng bằng lệnh của bài" trong Checkpoint đó.

**Đọc bài này với tư thế người đi tiếp quản cluster lạ.** Lộ trình cho phép *bỏ qua toàn bộ giai
đoạn 27 nếu cluster của bạn đã dùng containerd*, và cluster lab thuộc nhóm đó. Nhưng riêng bài này
vẫn đáng chạy tay: nó là quy trình nhận diện, và kết quả "không phải dockershim" cũng là một kết
luận có giá trị. Chạy `kubectl get nodes -o wide` trên `lab-k8s-master` là bạn đã làm xong nửa đầu;
nửa sau — đọc runtime endpoint — bạn từng làm ở
[Lab 2 phần B1.2](labs/LAB-2-CONTAINER-IMAGE-CRI-VA-CGROUP.md#b12-endpoint-grpc-nằm-ở-đâu).

**Phải hiểu ở lần đọc này:**

- Cột `CONTAINER-RUNTIME` của `kubectl get nodes -o wide` cho biết **runtime và phiên bản của nó**,
  dạng `docker://…` hoặc `containerd://…`. Cách này dùng được ở bất cứ đâu chạy được `kubectl`,
  không phụ thuộc công cụ riêng của nhà cung cấp.
- Cột đó **chưa phải kết luận cuối**: bài nói rõ thấy Docker Engine vẫn **có thể không bị ảnh
  hưởng**, phải kiểm tiếp runtime endpoint mới biết có đang qua dockershim hay không.
- Vì sao đọc socket là đủ để trả lời: runtime nói chuyện với kubelet qua một **Unix socket theo
  giao thức CRI trên nền gRPC**, trong đó **kubelet là client, runtime là server**. Biết node đang
  nối tới socket nào tức là biết ai đang phục vụ.
- Quy trình đọc: lấy dòng lệnh kubelet bằng `tr \0 ' ' < /proc/"$(pgrep kubelet)"/cmdline`, rồi tìm
  hai cờ `--container-runtime` và `--container-runtime-endpoint`.
- Hai nhánh kết luận, kèm ràng buộc phiên bản: node ở **v1.23 trở về trước** mà thiếu hai cờ này,
  hoặc `--container-runtime` không phải `remote`, thì đang dùng **dockershim socket với Docker
  Engine**; còn nếu có `--container-runtime-endpoint` thì **tên socket** chỉ ra runtime, ví dụ
  `unix:///run/containerd/containerd.sock` là containerd. Cờ `--container-runtime` không còn tồn tại
  từ **v1.27 trở về sau**.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Mục *Trước khi bạn bắt đầu* — cài và cấu hình `kubectl` | ba node lab đã có `kubectl` cấu hình xong từ Lab 00 | không cần — bài [185](185-tools-vi.md#kubectl) là chỗ tra khi dựng máy mới |
| Nhánh "dịch vụ Kubernetes được quản lý có cách kiểm tra riêng của từng nhà cung cấp" | cluster lab là kubeadm tự dựng, không có lớp nhà cung cấp nào chen vào | chỉ áp dụng khi tiếp quản cluster của một dịch vụ managed |
| Ghi chú về `cri-dockerd` và câu cuối trang về chuyển sang containerd | là hành động **sau** khi đã kết luận, không thuộc bước nhận diện | bài [237](237-change-runtime-containerd-vi.md) của chính giai đoạn 27 |

---

Trang này trình bày các bước để tìm hiểu xem các node trong cluster của bạn đang dùng
[container runtime](00-container-runtimes-vi.md) nào.

Tùy theo cách bạn vận hành cluster, container runtime cho các node có thể
đã được cấu hình sẵn hoặc bạn phải tự cấu hình. Nếu bạn đang dùng dịch vụ
Kubernetes được quản lý (managed), có thể có cách riêng của từng nhà cung cấp để kiểm tra container runtime nào
đang được cấu hình cho các node. Phương pháp mô tả trên trang này hoạt động bất cứ khi nào
việc thực thi `kubectl` được cho phép.

## Trước khi bạn bắt đầu (Before you begin)

Cài đặt và cấu hình `kubectl`. Xem mục [Install Tools](185-tools-vi.md#kubectl) để biết chi tiết.

## Tìm container runtime đang được dùng trên một Node (Find out the container runtime used on a Node)

Dùng `kubectl` để lấy và hiển thị thông tin node:

```shell
kubectl get nodes -o wide
```

Kết quả sẽ tương tự như dưới đây. Cột `CONTAINER-RUNTIME` hiển thị
runtime và phiên bản của nó.

Với Docker Engine, kết quả tương tự như sau:

```none
NAME         STATUS   VERSION    CONTAINER-RUNTIME
node-1       Ready    v1.16.15   docker://19.3.1
node-2       Ready    v1.16.15   docker://19.3.1
node-3       Ready    v1.16.15   docker://19.3.1
```

Nếu runtime của bạn hiển thị là Docker Engine, bạn vẫn có thể không bị ảnh hưởng bởi
việc loại bỏ dockershim trong Kubernetes v1.24.
[Kiểm tra runtime endpoint](#which-endpoint) để xem bạn có đang dùng dockershim hay không.
Nếu bạn không dùng dockershim, bạn không bị ảnh hưởng.

Với containerd, kết quả tương tự như sau:

```none
NAME         STATUS   VERSION   CONTAINER-RUNTIME
node-1       Ready    v1.19.6   containerd://1.4.1
node-2       Ready    v1.19.6   containerd://1.4.1
node-3       Ready    v1.19.6   containerd://1.4.1
```

Tìm hiểu thêm thông tin về các container runtime
tại trang [Container Runtimes](00-container-runtimes-vi.md).

## Tìm container runtime endpoint mà bạn đang dùng (Find out what container runtime endpoint you use) {#which-endpoint}

Container runtime giao tiếp với kubelet qua một Unix socket bằng [giao thức CRI](https://kubernetes.io/docs/concepts/architecture/cri/),
vốn dựa trên framework gRPC. Kubelet đóng vai trò client, còn runtime đóng vai trò server.
Trong một số trường hợp, bạn có thể thấy hữu ích khi biết node của mình đang dùng socket nào. Ví dụ,
với việc loại bỏ dockershim trong Kubernetes v1.24 trở về sau, bạn có thể
muốn biết liệu mình có đang dùng Docker Engine với dockershim hay không.

> **Ghi chú:** Nếu bạn hiện đang dùng Docker Engine trên các node của mình với `cri-dockerd`, bạn không
> bị ảnh hưởng bởi việc loại bỏ dockershim.

Bạn có thể kiểm tra socket đang dùng bằng cách kiểm tra cấu hình kubelet trên
các node của mình.

1.  Đọc các lệnh khởi động của tiến trình kubelet:

    ```
    tr \\0 ' ' < /proc/"$(pgrep kubelet)"/cmdline
    ```
    Nếu bạn không có `tr` hoặc `pgrep`, hãy kiểm tra thủ công dòng lệnh của tiến trình
    kubelet.

1.  Trong kết quả, tìm flag `--container-runtime` và flag
    `--container-runtime-endpoint`.

    *   Nếu node của bạn dùng Kubernetes v1.23 trở về trước và các flag này không
        xuất hiện, hoặc flag `--container-runtime` không phải là `remote`,
        thì bạn đang dùng dockershim socket với Docker Engine. Đối số dòng lệnh
        `--container-runtime` không còn khả dụng trong Kubernetes v1.27 trở về sau.
    *   Nếu flag `--container-runtime-endpoint` xuất hiện, hãy kiểm tra tên socket
        để tìm ra runtime bạn đang dùng. Ví dụ,
        `unix:///run/containerd/containerd.sock` là endpoint của containerd.

Nếu bạn muốn thay đổi Container Runtime trên một Node từ Docker Engine sang containerd,
bạn có thể tìm thêm thông tin tại [di chuyển từ Docker Engine sang containerd](237-change-runtime-containerd-vi.md),
hoặc, nếu bạn muốn tiếp tục dùng Docker Engine trong Kubernetes v1.24 trở về sau, hãy chuyển sang một
adapter tương thích CRI như [`cri-dockerd`](https://github.com/Mirantis/cri-dockerd).

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 27:

1. Trên `lab-k8s-master`, `kubectl get nodes -o wide` in `containerd://…` ở cột
   `CONTAINER-RUNTIME` cho cả ba node. Theo bài, bạn còn phải kiểm runtime endpoint nữa không, hay
   đã đủ kết luận rằng ba node không dùng dockershim?
2. **Câu bẫy.** Trên một cluster lạ, cột `CONTAINER-RUNTIME` in `docker://19.3.1`. Kết luận ngay
   rằng node đó đi qua dockershim và bị ảnh hưởng bởi việc loại bỏ dockershim — sai ở đâu?
3. Bạn đọc dòng lệnh kubelet của một node cũ và **không thấy** cả `--container-runtime` lẫn
   `--container-runtime-endpoint`. Bài cho phép kết luận gì, và điều kiện phiên bản nào phải đúng
   thì kết luận đó mới có giá trị?
4. Vì sao chỉ cần biết **tên socket** là đã trả lời được câu hỏi "runtime nào đang chạy", trong khi
   socket chỉ là một file trên đĩa? Ai là client, ai là server trên đường đó?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Đã đủ.** Bài chỉ bắt kiểm tiếp runtime endpoint trong **một** trường hợp: khi cột hiển thị
   Docker Engine, vì lúc đó còn mập mờ giữa dockershim và một adapter tương thích CRI. Giá trị
   `containerd://…` đã tự nó nói ra runtime, không có nhánh mập mờ nào để phân giải. Kiểm endpoint
   thêm thì không sai, nhưng bài không đòi.
2. Sai ở chỗ **coi "Docker Engine" đồng nghĩa với "dockershim"**. Bài nói thẳng: nếu runtime hiển
   thị là Docker Engine, bạn **vẫn có thể không bị ảnh hưởng** — phải
   [kiểm tra runtime endpoint](#which-endpoint) mới biết. Trường hợp cụ thể bài nêu: node dùng
   Docker Engine với **`cri-dockerd`** thì **không** bị ảnh hưởng bởi việc loại bỏ dockershim, vì
   `cri-dockerd` là adapter tương thích CRI riêng, không phải dockershim tích hợp trong kubelet.
3. Kết luận được rằng node đó **đang dùng dockershim socket với Docker Engine** — nhưng **chỉ khi
   node ở Kubernetes v1.23 trở về trước**. Ràng buộc phiên bản là bắt buộc chứ không phải chi tiết
   phụ: cờ `--container-runtime` **không còn khả dụng từ v1.27 trở về sau**, nên trên một node đời
   mới, việc thiếu cờ đó **không nói lên điều gì** về dockershim. Cùng một quan sát, hai phiên bản
   khác nhau, hai kết luận khác nhau.
4. Vì đường giữa kubelet và runtime là **giao thức CRI chạy trên gRPC qua một Unix socket**, và
   **kubelet là client còn container runtime là server**. Socket không phải dữ liệu, nó là **đầu
   mối kết nối tới đúng một tiến trình server**; kubelet nối vào socket nào thì runtime phục vụ
   socket đó chính là runtime đang chạy container của node. Vì vậy giá trị của
   `--container-runtime-endpoint` — ví dụ `unix:///run/containerd/containerd.sock` — là một câu trả
   lời trực tiếp, không phải suy đoán.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
