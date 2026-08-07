# Lưu trữ trên Windows (Windows Storage)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/storage/windows-storage/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](LO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 15](LO-TRINH-ADMIN.md#giai-đoạn-15--windows-nếu-môi-trường-có-node-windows),
bài 5/7 · Kiểm chứng ở Lab 15 (tùy chọn, chưa viết, xem
[bản đồ lab](labs/README.md#4-bản-đồ-lab)).

**Lộ trình ghi rõ: bỏ qua hoàn toàn giai đoạn 15 nếu cluster của bạn chỉ có Linux.** Cluster lab
ba VM Ubuntu của bạn không có node Windows, nên bài này chỉ để đối chiếu với những gì bạn đã làm
ở giai đoạn 6.

Bài rất ngắn và gần như **toàn bộ là một danh sách phủ định**: những gì lưu trữ trên Windows
không làm được. Đọc theo hướng "cái gì tôi quen dùng trên Ubuntu mà ở đây mất", và tìm **một
nguyên nhân gốc chung** cho phần lớn các gạch đầu dòng đó.

**Phải hiểu ở lần đọc này:**

- Cơ chế nền: Windows dùng **driver hệ thống file phân lớp** để mount các layer của container và
  tạo hệ thống file sao chép **dựa trên NTFS**. **Mọi đường dẫn file chỉ được phân giải trong
  ngữ cảnh của chính container** — không có chuyện chiếu ngược ra host.
- Nguyên nhân gốc của phần lớn hạn chế: **SAM không được chia sẻ giữa host và container**, nên
  **không tồn tại ánh xạ quyền nào** giữa hai bên; và **registry Windows cùng cơ sở dữ liệu SAM
  luôn cần quyền ghi**.
- Phân biệt hai thứ dễ lẫn: **hệ thống file gốc chỉ đọc không được hỗ trợ**, nhưng **volume chỉ
  đọc thì vẫn được** (các volume được ánh xạ vẫn hỗ trợ `readOnly`).
- Danh sách không hỗ trợ trên node Windows: **mount subpath của volume** (chỉ mount được toàn bộ
  volume), subpath cho Secret, chiếu mount ngược về host, root filesystem chỉ đọc, **ánh xạ thiết
  bị block**, **`emptyDir.medium: Memory`**, quyền hệ thống file theo **uid/gid**, `defaultMode`
  cho Secret, **lưu trữ dựa trên NFS**, và **mở rộng volume đã mount (resizefs)**.
- Plugin dùng được: nhóm **CSI** và nhóm [**FlexVolume**](91-volumes-vi.md#flexvolume) (đã
  deprecated từ 1.23); in-tree chỉ còn **`azureFile`** và **`vsphereVolume`**.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Khác biệt giữa Docker và containerd khi mount file đơn lẻ | phụ thuộc container runtime cụ thể trên node Windows | khi môi trường thực sự có node Windows |
| Chi tiết `azureFile`, `vsphereVolume` | gắn với nền tảng của một nhà cung cấp cụ thể | khi môi trường thực sự có node Windows |
| Khái niệm PV/PVC, provisioning, attach/detach, mount/unmount nói chung | đã học ở phần lưu trữ | giai đoạn 6 — bài [92](92-persistent-volumes-vi.md) |

---

Trang này cung cấp cái nhìn tổng quan về lưu trữ dành riêng cho hệ điều hành Windows.

## Lưu trữ bền vững (Persistent storage) {#storage}

Windows có driver hệ thống file phân lớp (layered filesystem driver) để mount các lớp
(layer) của container và tạo một hệ thống file sao chép dựa trên NTFS. Mọi đường dẫn
file trong container chỉ được phân giải trong ngữ cảnh của chính container đó.

* Với Docker, volume mount chỉ có thể trỏ tới một thư mục trong container, không thể
  trỏ tới một file riêng lẻ. Hạn chế này không áp dụng cho containerd.
* Volume mount không thể chiếu (project) file hoặc thư mục ngược trở lại hệ thống file
  của host.
* Hệ thống file chỉ đọc (read-only) không được hỗ trợ vì quyền ghi luôn được yêu cầu
  đối với registry của Windows và cơ sở dữ liệu SAM. Tuy nhiên, volume chỉ đọc vẫn
  được hỗ trợ.
* User-mask và quyền (permission) trên volume không khả dụng. Vì SAM không được chia sẻ
  giữa host và container, không tồn tại ánh xạ nào giữa hai bên. Mọi quyền đều được
  phân giải trong ngữ cảnh của container.

Do đó, các chức năng lưu trữ sau không được hỗ trợ trên node Windows:

* Mount subpath của volume: chỉ có thể mount toàn bộ volume vào một container Windows
* Mount volume theo subpath cho Secret
* Chiếu mount ngược về host (host mount projection)
* Hệ thống file gốc (root filesystem) chỉ đọc (các volume được ánh xạ vẫn hỗ trợ `readOnly`)
* Ánh xạ thiết bị block (block device mapping)
* Dùng bộ nhớ (memory) làm phương tiện lưu trữ (ví dụ, `emptyDir.medium` đặt thành `Memory`)
* Các tính năng hệ thống file như uid/gid; quyền hệ thống file Linux theo từng người dùng
* Thiết lập [quyền cho secret bằng DefaultMode](https://kubernetes.io/docs/tasks/inject-data-application/distribute-credentials-secure/#set-posix-permissions-for-secret-keys) (do phụ thuộc vào UID/GID)
* Hỗ trợ lưu trữ/volume dựa trên NFS
* Mở rộng volume đã mount (resizefs)

Các volume của Kubernetes cho phép triển khai trên Kubernetes những ứng dụng phức tạp
có yêu cầu lưu dữ liệu bền vững và chia sẻ volume giữa các Pod. Việc quản lý các
persistent volume gắn với một backend hoặc giao thức lưu trữ cụ thể bao gồm các thao
tác như: cấp phát/thu hồi cấp phát/thay đổi kích thước (provisioning/de-provisioning/resizing)
volume, gắn (attach) một volume vào / tháo (detach) một volume khỏi một node Kubernetes,
và mount/unmount một volume vào/khỏi từng container trong một Pod cần lưu dữ liệu bền vững.

Các thành phần quản lý volume được phát hành dưới dạng
[plugin](./91-volumes-vi.md#volume-types) volume của Kubernetes.
Các nhóm plugin volume Kubernetes lớn sau đây được hỗ trợ trên Windows:

* [`FlexVolume plugins`](./91-volumes-vi.md#flexvolume)
  * Lưu ý rằng FlexVolume đã bị loại bỏ dần (deprecated) kể từ phiên bản 1.23
* [`CSI Plugins`](https://kubernetes.io/docs/concepts/storage/volumes/#csi)

##### Các plugin volume in-tree (In-tree volume plugins)

Các plugin in-tree sau hỗ trợ lưu trữ bền vững trên node Windows:

* [`azureFile`](https://kubernetes.io/docs/concepts/storage/volumes/#azurefile)
* [`vsphereVolume`](https://kubernetes.io/docs/concepts/storage/volumes/#vspherevolume)

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 15:

1. Câu bẫy: bài viết "hệ thống file chỉ đọc không được hỗ trợ" nhưng ngay sau đó "volume chỉ đọc
   vẫn được hỗ trợ". Hai câu này nói về hai thứ khác nhau — là gì? Vì sao hệ thống file gốc bắt
   buộc phải ghi được?
2. Trên node Ubuntu bạn mount đúng một key của Secret vào một file bằng `subPath`, và đặt
   `defaultMode: 0400` cho nó. Hai thao tác đó trên node Windows ra sao?
3. Vì sao "quyền hệ thống file theo uid/gid" không có nghĩa gì trên Windows? Bài quy về nguyên
   nhân nào, và nguyên nhân đó còn giải thích thêm hạn chế nào nữa?
4. Bạn có một Pod dùng `emptyDir` với `medium: Memory` để làm scratch space nhanh, và một Pod
   khác mount NFS. Chuyển cả hai sang node Windows thì sao?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Câu thứ nhất nói về **hệ thống file gốc (root filesystem) của container**, câu thứ hai nói về
   **volume được mount vào**. Bài giải thích lý do: "Hệ thống file chỉ đọc không được hỗ trợ vì
   **quyền ghi luôn được yêu cầu đối với registry của Windows và cơ sở dữ liệu SAM**." Tức chính
   hệ điều hành trong container cần ghi để chạy — đây cũng là lý do bài
   [175](175-windows-intro-vi.md) xếp `securityContext.readOnlyRootFilesystem` vào nhóm "không
   khả thi trên Windows". Còn `readOnly` trên **các volume được ánh xạ** thì vẫn có tác dụng bình
   thường.
2. **Cả hai đều không dùng được.** Bài liệt kê "**mount volume theo subpath cho Secret**" và
   "thiết lập **quyền cho secret bằng DefaultMode** (do phụ thuộc vào UID/GID)" trong danh sách
   chức năng không hỗ trợ trên node Windows. Với volume nói chung cũng vậy: "**mount subpath của
   volume: chỉ có thể mount toàn bộ volume vào một container Windows**".
3. Vì **SAM không được chia sẻ giữa host và container**, nên **không tồn tại ánh xạ nào giữa hai
   bên** và "mọi quyền đều được phân giải trong ngữ cảnh của container" — user-mask và quyền trên
   volume do đó không khả dụng. Cùng nguyên nhân đó giải thích luôn vì sao **`defaultMode` cho
   Secret** không dùng được (nó phụ thuộc UID/GID) và vì sao **chiếu mount ngược về host** không
   khả thi: "Volume mount không thể chiếu file hoặc thư mục ngược trở lại hệ thống file của
   host."
4. **Cả hai đều hỏng.** "Dùng bộ nhớ (memory) làm phương tiện lưu trữ (ví dụ, `emptyDir.medium`
   đặt thành `Memory`)" và "**hỗ trợ lưu trữ/volume dựa trên NFS**" đều nằm trong danh sách chức
   năng lưu trữ **không được hỗ trợ trên node Windows**. `emptyDir` thường vẫn dùng được — bài
   [175](175-windows-intro-vi.md) liệt kê volume `emptyDir` trong nhóm được hỗ trợ; chỉ riêng
   nguồn `memory` là không.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
