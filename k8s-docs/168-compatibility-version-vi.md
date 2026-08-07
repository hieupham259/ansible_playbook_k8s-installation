# Phiên bản tương thích cho các thành phần Control Plane của Kubernetes (Compatibility Version For Kubernetes Control Plane Components)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/concepts/cluster-administration/compatibility-version/

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](LO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 12](LO-TRINH-ADMIN.md#giai-đoạn-12--quản-trị-cluster-nâng-cao), bài 6/8 ·
Kiểm chứng ở Lab 12 (chưa viết, xem [bản đồ lab](labs/README.md#4-bản-đồ-lab)).

Bài chỉ dài mười mấy dòng nhưng đáng đọc kỹ, vì nó giới thiệu một ý tưởng thay đổi cách nghĩ về
nâng cấp: **đổi binary và đổi hành vi là hai việc tách rời được**. Ghi nhớ ý đó, vì bài
[167](167-coordinated-leader-election-vi.md) ngay sau đây dựa hẳn vào emulation version để chọn
leader.

**Phải hiểu ở lần đọc này:**

- Từ **v1.32**, các thành phần control plane nhận flag **`--emulated-version`** để giả lập hành
  vi — API, tính năng — của một phiên bản Kubernetes cũ hơn.
- Quy tắc xác định khả năng khả dụng, đối xứng nhau: thứ được **giới thiệu sau** emulation version
  thì **không** khả dụng; thứ bị **loại bỏ sau** emulation version thì **vẫn** khả dụng.
- Ràng buộc cứng: `--emulated-version` phải **≤ `binaryVersion`**. Bạn giả lập được phiên bản cũ
  hơn binary đang chạy, không bao giờ giả lập được phiên bản mới hơn.
- Mục đích thực dụng: **tách bước thay binary khỏi bước đổi hành vi**, để việc nâng cấp có nhiều
  quyền kiểm soát hơn và chia được thành các bước nhỏ hơn — nếu bước sau hỏng thì lùi lại chỉ là
  đổi một flag, không phải hạ cấp binary.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Ý "khả năng tương tác giữa các thành phần có thể được định nghĩa theo phiên bản được giả lập" | chỉ có nghĩa khi vận hành nhiều thành phần lệch phiên bản trong một lần nâng cấp thật | CP2 nâng cấp cluster |
| Việc tra *thông điệp trợ giúp* của flag để biết dải phiên bản giả lập được hỗ trợ | là thao tác tại chỗ ngay trước khi nâng cấp | CP2 nâng cấp cluster |

---

Kể từ bản phát hành v1.32, chúng tôi đã đưa vào các tùy chọn có thể cấu hình về tương thích phiên bản (version compatibility) và giả lập (emulation) cho các thành phần control plane của Kubernetes, nhằm giúp việc nâng cấp an toàn hơn bằng cách cung cấp nhiều quyền kiểm soát hơn và tăng độ chi tiết của các bước mà quản trị viên cluster có thể thực hiện.

## Phiên bản giả lập (Emulated Version)

Tùy chọn giả lập được đặt bằng flag `--emulated-version` của các thành phần control plane. Nó cho phép thành phần đó giả lập hành vi (các API, tính năng, ...) của một phiên bản Kubernetes cũ hơn.

Khi được sử dụng, các khả năng (capability) sẵn có sẽ khớp với phiên bản được giả lập:
* Bất kỳ khả năng nào có trong binary version nhưng được giới thiệu sau emulation version sẽ không khả dụng.
* Bất kỳ khả năng nào bị loại bỏ sau emulation version sẽ vẫn khả dụng.

Điều này cho phép một binary của một bản phát hành Kubernetes cụ thể giả lập hành vi của một phiên bản trước đó với độ trung thực đủ cao, đến mức khả năng tương tác (interoperability) với các thành phần hệ thống khác có thể được định nghĩa theo phiên bản được giả lập.

Giá trị `--emulated-version` phải <= `binaryVersion`. Xem thông điệp trợ giúp (help message) của flag `--emulated-version` để biết dải phiên bản giả lập được hỗ trợ.

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 12:

1. Cluster lab đang chạy phiên bản đã khóa ở
   [bảng A1.3 của Lab 00](labs/LAB-00-MOI-TRUONG.md#a13-phiên-bản-được-khóa). Bạn đặt
   `--emulated-version` cao hơn phiên bản binary đó được không? Vì sao?
2. **Câu bẫy.** Bạn nâng binary kube-apiserver lên một phiên bản mới nhưng đặt `--emulated-version`
   giữ ở phiên bản cũ. Một API đã bị xóa ở phiên bản mới — còn dùng được không? Một tính năng chỉ
   có ở phiên bản mới — có dùng được không?
3. Tùy chọn này làm việc nâng cấp an toàn hơn bằng cách tách hai việc gì ra khỏi nhau?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Không.** Bài nêu ràng buộc thẳng: **`--emulated-version` phải ≤ `binaryVersion`**. Giả lập
   là làm cho binary **cư xử như một phiên bản cũ hơn**; một binary không thể tự nghĩ ra hành vi
   của phiên bản chưa tồn tại trong chính nó. Dải giả lập được hỗ trợ tra ở thông điệp trợ giúp
   của flag.
2. API bị xóa ở phiên bản mới thì **vẫn dùng được**, còn tính năng mới thì **không dùng được**.
   Đây đúng là cặp quy tắc của bài: khả năng **được giới thiệu sau** emulation version thì không
   khả dụng, khả năng **bị loại bỏ sau** emulation version thì vẫn khả dụng. Chỗ dễ ngã là nghĩ
   "binary mới thì có mọi thứ mới, thêm emulation chỉ là giữ tương thích ngược" — thực ra emulation
   **cắt cả hai chiều**: bạn được giữ cái cũ nhưng đồng thời mất quyền dùng cái mới, cho tới khi
   nâng emulation version lên.
3. Tách **việc thay binary** khỏi **việc đổi hành vi**. Nhờ đó quản trị viên có "nhiều quyền kiểm
   soát hơn và tăng độ chi tiết của các bước": trước hết triển khai binary mới trong khi hành vi
   vẫn y như cũ, xác nhận cluster ổn, rồi mới nâng emulation version thành một bước riêng — và nếu
   bước đó gây sự cố thì lùi lại chỉ là đổi một flag, không phải hạ cấp binary.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
