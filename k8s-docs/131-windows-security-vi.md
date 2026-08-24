# Bảo mật cho các node Windows (Security For Windows Nodes)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/security/windows-security/>
>
> Trang này mô tả các cân nhắc về bảo mật và các thực hành tốt nhất dành riêng cho hệ điều hành Windows.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 15](00-ALO-TRINH-ADMIN.md#giai-đoạn-15--windows-nếu-môi-trường-có-node-windows),
bài 7/7 · Kiểm chứng ở Lab 15 (tùy chọn, chưa viết, xem
[bản đồ lab](labs/README.md#4-bản-đồ-lab)).

**Lộ trình ghi rõ: bỏ qua hoàn toàn giai đoạn 15 nếu cluster của bạn chỉ có Linux.** Cluster lab
ba VM Ubuntu của bạn không có node Windows.

Bài chỉ hơn 40 dòng và là bài khép lại giai đoạn. Nó trả lời đúng một câu hỏi: **những gì bạn đã
dựng ở giai đoạn 9 còn tác dụng bao nhiêu trên node Windows?** Câu trả lời ngắn là: phần
RBAC/ServiceAccount thì nguyên vẹn, còn **toàn bộ tầng cô lập dựa vào kernel Linux thì không**.

**Phải hiểu ở lần đọc này:**

- Khác biệt nguy hiểm nhất: trên Windows, dữ liệu Secret được **ghi ra dưới dạng văn bản thuần
  trên bộ lưu trữ cục bộ của node**, khác với tmpfs / filesystem trong bộ nhớ của Linux. Người
  vận hành phải bù bằng **cả hai** biện pháp: **file ACL** bảo vệ vị trí file và **mã hóa cấp
  volume bằng BitLocker**.
- Danh tính container: dùng **`RunAsUsername`** thay cho `RunAsUser`. Có hai tài khoản mặc định
  **ContainerUser** và **ContainerAdministrator**; image **Nano Server** mặc định chạy dưới
  `ContainerUser`, image **Server Core** mặc định chạy dưới `ContainerAdministrator`.
- Container Windows có thể chạy dưới **danh tính Active Directory** bằng Group Managed Service
  Accounts (GMSA) — thứ không có tương đương trên Linux.
- Cô lập ở mức Pod: **SELinux, AppArmor, Seccomp và POSIX capability tùy chỉnh không được hỗ
  trợ** trên node Windows; **container đặc quyền cũng không**, thay bằng **HostProcess container**
  — thứ làm được *nhiều* việc mà container đặc quyền làm trên Linux, không phải *mọi* việc.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Cách cấu hình `RunAsUsername`, GMSA, HostProcess container | là thao tác trên cluster có node Windows | khi môi trường thực sự có node Windows |
| Cách chọn giữa ContainerAdmin và ContainerUser theo tài liệu Microsoft | phụ thuộc từng image cụ thể | khi môi trường thực sự có node Windows |
| Cách thêm local user vào image lúc build | là việc của người build image | khi môi trường thực sự có node Windows |
| Nền tảng SELinux, AppArmor, seccomp, capability và các profile bảo mật Pod | đã học ở phần bảo mật | giai đoạn 9 — bài [127](127-linux-kernel-security-vi.md) và [115](115-pod-security-standards-vi.md) |

---

Trang này mô tả các cân nhắc về bảo mật và các thực hành tốt nhất dành riêng cho hệ điều hành Windows.

## Bảo vệ dữ liệu Secret trên node (Protection for Secret data on nodes)

Trên Windows, dữ liệu từ Secret được ghi ra dưới dạng văn bản thuần (clear text) trên bộ lưu trữ
cục bộ của node (khác với việc dùng tmpfs / filesystem trong bộ nhớ trên Linux). Với tư cách là
người vận hành cluster, bạn nên thực hiện cả hai biện pháp bổ sung sau:

1. Sử dụng file ACL để bảo vệ vị trí file của các Secret.
1. Áp dụng mã hóa cấp volume bằng
   [BitLocker](https://docs.microsoft.com/windows/security/information-protection/bitlocker/bitlocker-how-to-deploy-on-windows-server).

## Người dùng trong container (Container users)

[RunAsUsername](278-configure-runasusername-vi.md)
có thể được chỉ định cho các Pod hoặc container Windows để thực thi các tiến trình
của container dưới một người dùng cụ thể. Điều này gần tương đương với
[RunAsUser](https://kubernetes.io/docs/concepts/security/pod-security-policy#users-and-groups).

Các container Windows cung cấp hai tài khoản người dùng mặc định là ContainerUser và ContainerAdministrator.
Sự khác biệt giữa hai tài khoản người dùng này được trình bày trong
[When to use ContainerAdmin and ContainerUser user accounts](https://docs.microsoft.com/virtualization/windowscontainers/manage-containers/container-security#when-to-use-containeradmin-and-containeruser-user-accounts)
thuộc tài liệu _Secure Windows containers_ của Microsoft.

Người dùng cục bộ (local user) có thể được thêm vào container image trong quá trình build container.

> **Ghi chú:**
>
> * Các image dựa trên [Nano Server](https://hub.docker.com/_/microsoft-windows-nanoserver) chạy dưới
>   `ContainerUser` theo mặc định
> * Các image dựa trên [Server Core](https://hub.docker.com/_/microsoft-windows-servercore) chạy dưới
>   `ContainerAdministrator` theo mặc định

Các container Windows cũng có thể chạy dưới danh tính Active Directory bằng cách sử dụng
[Group Managed Service Accounts](273-configure-gmsa-vi.md)

## Cô lập bảo mật cấp Pod (Pod-level security isolation)

Các cơ chế security context của pod dành riêng cho Linux (như SELinux, AppArmor, Seccomp, hay
POSIX capability tùy chỉnh) không được hỗ trợ trên các node Windows.

Container đặc quyền (privileged container) [không được hỗ trợ](175-windows-intro-vi.md#compatibility-v1-pod-spec-containers-securitycontext)
trên Windows.
Thay vào đó, có thể dùng [HostProcess container](281-create-hostprocess-pod-vi.md)
trên Windows để thực hiện nhiều tác vụ mà container đặc quyền thực hiện trên Linux.

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 15:

1. Câu bẫy: trên node Ubuntu, Secret mount vào Pod nằm trên tmpfs — filesystem trong bộ nhớ, không
   chạm đĩa. Trên node Windows nó nằm ở đâu, dưới dạng gì, và bài bắt bạn bù bằng hai biện pháp
   nào?
2. Bài liệt kê những cơ chế cô lập nào của Linux **không** dùng được trên node Windows? Nếu bạn
   đã quen dựa vào chúng ở giai đoạn 9 thì trên Windows còn lại gì?
3. Hai tài khoản mặc định của container Windows là gì, và image dựa trên Nano Server với image
   dựa trên Server Core mặc định chạy dưới tài khoản nào? Vì sao khác biệt đó đáng quan tâm về
   mặt bảo mật?
4. Container đặc quyền trên Windows: bài trả lời thế nào và đưa ra thay thế nào? Thay thế đó có
   tương đương hoàn toàn không — hãy đọc kỹ chữ bài dùng.

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Trên Windows, "dữ liệu từ Secret được ghi ra dưới dạng **văn bản thuần (clear text)** trên
   **bộ lưu trữ cục bộ của node**", bài ghi rõ là "khác với việc dùng tmpfs / filesystem trong bộ
   nhớ trên Linux". Nghĩa là Secret **chạm đĩa** và tồn tại ở đó. Bài yêu cầu người vận hành
   cluster thực hiện **cả hai** biện pháp bổ sung: **dùng file ACL để bảo vệ vị trí file của các
   Secret**, và **áp dụng mã hóa cấp volume bằng BitLocker**. Đây là chỗ dễ chủ quan nhất: cùng
   một object Secret trong API, nhưng mức bảo vệ trên node hoàn toàn khác nhau giữa hai hệ điều
   hành.
2. **SELinux, AppArmor, Seccomp và POSIX capability tùy chỉnh** — bài gọi chung là "các cơ chế
   security context của pod dành riêng cho Linux" — **không được hỗ trợ trên các node Windows**;
   **container đặc quyền cũng không được hỗ trợ**. Thứ còn lại là các tầng không phụ thuộc kernel
   Linux: xác thực và phân quyền của API server, RBAC, ServiceAccount, namespace, cộng với các cơ
   chế riêng của Windows nêu trong bài — `RunAsUsername`, GMSA, file ACL và BitLocker.
3. **ContainerUser** và **ContainerAdministrator**. Image dựa trên **Nano Server** mặc định chạy
   dưới **`ContainerUser`**, image dựa trên **Server Core** mặc định chạy dưới
   **`ContainerAdministrator`**. Khác biệt đáng quan tâm vì chỉ cần đổi base image là **quyền mặc
   định của tiến trình trong container đổi theo**, mà manifest thì không hề thay đổi. Muốn ghi
   đè, bạn chỉ định `RunAsUsername` cho Pod hoặc container — thứ bài mô tả là "gần tương đương"
   `RunAsUser` của Linux.
4. Bài trả lời dứt khoát: container đặc quyền **không được hỗ trợ** trên Windows. Thay thế là
   **HostProcess container**, dùng "để thực hiện **nhiều** tác vụ mà container đặc quyền thực
   hiện trên Linux". Chữ **nhiều**, không phải *mọi* — bài [175](175-windows-intro-vi.md) cũng
   chỉ nói HostProcess "cung cấp chức năng **tương tự**". Đừng coi đây là ánh xạ một–một; khi
   chuyển một workload đặc quyền từ Linux sang, phải kiểm tra lại từng khả năng nó cần.

</details>

Đây là bài cuối của **Giai đoạn 15**, và cũng là bài cuối của cả 15 giai đoạn lý thuyết. Nếu môi
trường của bạn thực sự có node Windows thì tiếp tục với Lab 15 (tùy chọn, chưa viết, xem
[bản đồ lab](labs/README.md#4-bản-đồ-lab)); nếu không thì chuyển thẳng sang phần thực hành vận
hành ở [Checkpoint tiếp nối](00-ALO-TRINH-ADMIN.md#checkpoint-tiếp-nối--nhánh-docstasks).
