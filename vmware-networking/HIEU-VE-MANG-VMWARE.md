# Hiểu về mạng VMware Workstation — Bridged, NAT, Host-only và VMnet

> **Mục đích:** trang bị đủ kiến thức để tự quyết định VM nên dùng Bridged, NAT, Host-only hay
> LAN segment, và hiểu các VMnet trong Virtual Network Editor thực chất là gì. Đọc xong phần
> §7 phải tự trả lời được "lab này cần mode nào" mà không phải hỏi lại.
>
> Áp dụng cho **VMware Workstation Pro trên Windows host**. Ngày đối chiếu: **18/08/2026**.
> Hai ví dụ thật trong repo: [runbook VMware](../runbook-k8s-vmware.md) +
> [Lab 00](../k8s-docs/labs/LAB-00-MOI-TRUONG-1.35.7.md) (Bridged) và
> [Lab M1](../rke2-multi-cluster-labs/LAB-M1-BA-CUM-RKE2-RANCHER-GITLAB.md)
> (host-only nhiều dải + router NAT).

---

## 1. Mô hình nền tảng: mọi thứ đều là VMnet

Đừng học thuộc "3 mode". Học một mô hình duy nhất:

- **VMnet = một switch ảo** (một broadcast domain, một dải L2). VMware Workstation trên
  Windows cho tối đa 20 switch: `VMnet0` … `VMnet19`.
- **Network adapter của VM là một sợi cáp** cắm vào đúng một VMnet. VM nhiều adapter thì cắm
  được nhiều VMnet cùng lúc (đây là cách dựng router VM ở §8.2).
- Mỗi VMnet có thể gắn thêm tối đa 4 "thiết bị" sau, bật tắt độc lập:

| Thiết bị | Vai trò | Thấy ở đâu |
| --- | --- | --- |
| **Bridge tới NIC vật lý** | Nối switch ảo thẳng vào LAN thật ở layer 2 | Cột Type = *Bridged* |
| **Host virtual adapter** | Card mạng ảo cắm vào Windows host, để host tham gia dải đó | `ipconfig` thấy `VMware Network Adapter VMnetX` |
| **DHCP server ảo** | Cấp IP động cho VM trong dải | Tick *Use local DHCP service* |
| **NAT device** | Dịch địa chỉ để cả dải ra Internet bằng IP của host | Chỉ tồn tại trên đúng một VMnet kiểu NAT |

Ba "mode" trong VM Settings chỉ là ba tổ hợp dựng sẵn của các thiết bị trên:

| Mode trong VM Settings | VMnet mặc định | Bridge | Host adapter | DHCP ảo | NAT device |
| --- | --- | --- | --- | --- | --- |
| Bridged | `VMnet0` | ✅ | ❌ | ❌ (dùng DHCP của LAN thật) | ❌ |
| NAT | `VMnet8` | ❌ | ✅ | ✅ | ✅ |
| Host-only | `VMnet1` | ❌ | ✅ | ✅ (tắt được) | ❌ |
| Custom → VMnet2..7, 9..19 | tự tạo | ❌ | tùy chọn | tùy chọn | ❌ |
| LAN segment | không phải VMnet | ❌ | ❌ | ❌ | ❌ |

Nắm bảng này là nắm 80% vấn đề: chọn mode thực chất là chọn **VM cắm vào switch nào, và
switch đó có những thiết bị gì**.

## 2. Bridged (VMnet0)

### 2.1. Cơ chế

`VMnet0` được bridge ở **layer 2** vào một NIC vật lý của host. VM xuất hiện trên LAN thật
như một máy tính độc lập: MAC riêng, xin IP từ **DHCP của router nhà**, cùng dải với host và
mọi thiết bị khác trong LAN.

```text
Router nhà (192.168.1.1, DHCP)
   │
LAN thật 192.168.1.0/24 ──── host Windows (192.168.1.10)
   │                              │ NIC vật lý ⇄ bridge ⇄ VMnet0
   ├── điện thoại, máy in…        ├── VM1 (192.168.1.21)
   │                              └── VM2 (192.168.1.22)
```

### 2.2. Hệ quả

- **Mọi chiều đều thông**: LAN → VM, host → VM, VM → VM, VM → Internet. Không cần cấu hình gì.
- VM dùng **gateway và DNS của LAN thật**. Dải IP do router nhà quyết định — mỗi nhà một khác.
- Các VM bridged nằm **chung một broadcast domain** với nhau và với cả LAN. Không tách dải,
  không đặt firewall giữa các VM được.

### 2.3. Khi nào chọn, khi nào tránh

Chọn khi: cần thiết bị khác trong LAN (laptop thứ hai, điện thoại) truy cập VM trực tiếp;
hoặc cần đường host ⇄ VM đơn giản nhất mà không quan tâm topology — đúng trường hợp cluster
phẳng của [Lab 00](../k8s-docs/labs/LAB-00-MOI-TRUONG-1.35.7.md).

Tránh khi:

- Mạng không cho bridge: **Wi-Fi công cộng/công ty có AP isolation**, mạng 802.1X chỉ chấp
  nhận một MAC trên một port, một số adapter Wi-Fi bridge không ổn định → triệu chứng là VM
  không xin được DHCP.
- Cần **IP cố định tái lập** trên mọi máy host — bridged phụ thuộc dải LAN từng nơi; đổi
  Wi-Fi là VM đổi dải.
- Cần nhiều hơn một dải mạng — bridged chỉ có một.

Mẹo vận hành: trên laptop, tick *Replicate physical network connection state* để VM tự xin
lại DHCP khi host đổi mạng. Và **không để "Bridged to: Automatic"** — xem bẫy ở §9.

## 3. NAT (VMnet8)

### 3.1. Cơ chế

`VMnet8` là một switch **cô lập khỏi LAN thật**, có đủ ba thiết bị: host adapter, DHCP ảo,
và NAT device. VM ra Internet bằng cách NAT device dịch địa chỉ, đi ra ngoài **dưới IP của
host** — giống hệt cách router nhà NAT cho cả nhà ra Internet.

```text
Internet ⇄ router nhà ⇄ host Windows (192.168.1.10)
                              │
                              ├─ NAT device  (gateway .2 của dải VMnet8)
                              ├─ host adapter VMnet8 (mặc định .1)
                              │
                        VMnet8 172.16.90.0/24 (dải tự chọn, độc lập LAN)
                              ├── VM1 (.11)
                              └── VM2 (.12)
```

### 3.2. Hệ quả

- **VM → Internet, VM → LAN thật, VM → host: thông.** VM cùng VMnet8 thấy nhau bình thường.
- **Host → VM: thông**, nhờ host adapter cắm sẵn vào dải — SSH từ host vào VM không cần gì
  thêm. Đây là điểm hay bị hiểu nhầm: NAT **không** chặn host truy cập VM.
- **LAN thật → VM: không thông** (VM "vô hình" với thiết bị khác trong nhà), trừ khi khai
  báo port forwarding trong *Virtual Network Editor → chọn VMnet8 → NAT Settings*.
- Subnet của VMnet8 **tự chọn và giữ nguyên trên mọi máy host** → lab tái lập được, IP tĩnh
  viết vào tài liệu không phải kèm câu "đổi theo dải LAN của bạn".
- Quy ước địa chỉ của VMware trong dải NAT: NAT device chiếm **`.2`** (đây là gateway và DNS
  forwarder cho VM), host adapter mặc định **`.1`**, DHCP ảo mặc định cấp từ `.128`. Đặt IP
  tĩnh cho VM thì trỏ gateway về `.2`, không phải `.1`.

Chỉ có **một** VMnet kiểu NAT trong toàn hệ thống (mặc định VMnet8). Cần "nhiều dải đều ra
được Internet" thì không nhân bản NAT được — phải dựng router VM như §8.2.

## 4. Host-only (VMnet1 và VMnet2–19 tự tạo)

Switch + host adapter, **không có NAT device**: VM và host thấy nhau, VM cùng dải thấy nhau,
nhưng **không có đường ra Internet và không thấy LAN thật**. DHCP ảo bật tắt tùy ý — lab đặt
IP tĩnh thì tắt để khỏi nhiễu.

Giá trị thật của host-only nằm ở chỗ **tạo được nhiều dải**: VMnet2, VMnet3… mỗi cái một
subnet tự đặt, cô lập nhau hoàn toàn ở L2. Muốn các dải nói chuyện với nhau hay ra Internet
thì phải tự dựng **router VM** đứng giữa — nghe như nhược điểm nhưng chính là tính năng: mọi
traffic liên dải buộc phải đi qua một điểm mà ta kiểm soát được (routing, firewall, DNS),
giống hệt mạng doanh nghiệp thật. [Lab M1](../rke2-multi-cluster-labs/LAB-M1-BA-CUM-RKE2-RANCHER-GITLAB.md)
dùng đúng pattern này với 4 dải VMnet2–5.

Host adapter của mỗi dải là "cửa hậu quản trị": host SSH thẳng vào VM của dải đó **không đi
qua router VM**, nên firewall trên router có chặt đến đâu cũng không khóa mất đường quản trị.
Mặc định host adapter lấy `.1`; nếu `.1` dành cho router VM thì đổi IP host adapter trong
Windows (Lab M1 dùng `.254`).

## 5. LAN segment — switch "trần"

Cấu hình trong **VM Settings** (không phải Virtual Network Editor): một switch thuần túy,
không host adapter, không DHCP, không NAT. Chỉ các VM cùng segment thấy nhau; **host cũng
không thấy VM**.

Khác host-only đúng một điểm: không có host adapter. Dùng khi muốn cô lập tuyệt đối (lab
malware, lab thi mô phỏng không có đường tắt từ host). Nhược điểm vận hành: mất SSH từ host,
mọi thao tác qua console VMware — với lab dài dòng lệnh thì rất khổ, nên các lab trong repo
này không dùng.

## 6. Bảng so sánh tổng hợp

| Trục so sánh | Bridged | NAT | Host-only | LAN segment |
| --- | --- | --- | --- | --- |
| VM lấy IP từ | DHCP của LAN thật | DHCP ảo / tĩnh, dải tự chọn | DHCP ảo / tĩnh, dải tự chọn | tự đặt tĩnh |
| VM → Internet | ✅ | ✅ (qua NAT device) | ❌ (trừ khi có router VM) | ❌ |
| Host → VM | ✅ | ✅ (qua host adapter) | ✅ (qua host adapter) | ❌ |
| Thiết bị LAN khác → VM | ✅ | ❌ (trừ port forwarding) | ❌ | ❌ |
| VM ↔ VM cùng dải | ✅ | ✅ | ✅ | ✅ |
| Phụ thuộc dải LAN vật lý | **Có** — đổi mạng là đổi IP | Không | Không | Không |
| Số dải tạo được | 1 | 1 | nhiều (VMnet2–19) | nhiều |
| Kiểm soát traffic giữa các VM | ❌ chung broadcast domain | ❌ trong cùng dải | ✅ nếu tách dải + router VM | ✅ như host-only |
| Dùng cho | cluster phẳng cần LAN thấy VM | cluster phẳng, LAN cấm bridge, cần tái lập | topology nhiều segment, firewall giữa dải | cô lập tuyệt đối |

## 7. Cách chọn — trả lời lần lượt bốn câu hỏi

Đi từ trên xuống, dừng ở câu đầu tiên quyết định được:

1. **Có thiết bị nào NGOÀI host cần chủ động mở kết nối tới VM không?** (laptop khác, điện
   thoại, máy đồng nghiệp)
   - Có, nhiều dịch vụ/cổng → **Bridged**.
   - Có, chỉ 1–2 cổng cụ thể → **NAT + port forwarding** cũng đủ, đỡ phụ thuộc LAN.
   - Không, chỉ host cần SSH vào VM → NAT và host-only đều thỏa, xuống câu 2.
2. **Cần bao nhiêu dải mạng, và có cần chặn/lọc traffic giữa các VM không?**
   - Một dải phẳng, VM nào cũng ngang hàng → xuống câu 3.
   - Nhiều dải, hoặc cần firewall giữa các nhóm VM → **host-only nhiều VMnet + router VM**
     (§8.2). Bridged và NAT về nguyên lý không làm được việc này: mọi VM chung một switch,
     traffic đi thẳng VM → VM, không có điểm nào để đặt firewall.
3. **VM có cần Internet không?**
   - Có → **NAT** (một dải, có sẵn egress) hoặc Bridged nếu câu 1 đã chọn.
   - Không → **host-only** là đủ và an toàn nhất.
4. **Môi trường LAN có cho bridge không, và có cần IP tái lập không?**
   - LAN nhà, chấp nhận IP theo dải nhà → Bridged dùng được.
   - Wi-Fi công ty/quán, AP isolation, 802.1X, hay cần bộ IP cố định viết vào tài liệu chạy
     được ở mọi nơi → **NAT**.

Quy tắc nhớ nhanh: **Bridged là "VM như một máy thật trong nhà"; NAT là "VM nấp sau host";
host-only là "mạng riêng tự quản, muốn gì tự dựng"**.

## 8. Hai pattern thật trong repo này

### 8.1. Cluster phẳng — Bridged ([Lab 00](../k8s-docs/labs/LAB-00-MOI-TRUONG-1.35.7.md), [runbook VMware](../runbook-k8s-vmware.md))

Yêu cầu chỉ có ba: 3 VM liên lạc hai chiều, host SSH được vào cả 3, cả 3 có Internet egress.
Bridged thỏa cả ba với zero cấu hình thêm nên được chọn làm mặc định. Lab 00 nói rõ đây
**không phải nguyên tắc**: LAN không cho bridge thì chuyển sang một VMnet NAT riêng, miễn ba
điều kiện trên còn đúng. Theo bảng §6, NAT quả thật vẫn thỏa cả ba — chỉ khác là thiết bị LAN
ngoài không thấy VM, điều mà lab đó không cần.

### 8.2. Nhiều segment + router VM — host-only ([Lab M1](../rke2-multi-cluster-labs/LAB-M1-BA-CUM-RKE2-RANCHER-GITLAB.md))

Yêu cầu của Lab M1 là thứ Bridged/NAT không đáp ứng được: 4 dải riêng (ADMIN/CICD/APP/DATA),
firewall giữa dải (chặn mọi dải vào DATA, chỉ mở CICD → 5432), DNS nội bộ `*.mc.lab`, và bộ
IP `10.20.x.0/24` tái lập trên mọi máy host. Lời giải là pattern **router VM**:

```text
Internet ⇄ NAT VMnet8 ⇄ [mc-router] ⇄ VMnet2 (ADMIN) · VMnet3 (CICD) · VMnet4 (APP) · VMnet5 (DATA)
                        5 NIC: 1 NAT + 4 host-only, làm gateway .1 của cả 4 dải
```

- 4 VMnet host-only, tắt DHCP ảo, mỗi dải một subnet.
- Router VM có 1 NIC cắm VMnet8 (NAT — chỉ để lấy egress Internet) + 4 NIC cắm 4 dải, bật
  `ip_forward`, chạy nftables (masquerade ra NAT, lọc liên dải) và dnsmasq.
- Host adapter mỗi dải giữ đường SSH quản trị không qua router.

Vậy câu "vì sao M1 dùng NAT còn Lab 00 dùng Bridged" có đáp án chính xác là: **M1 không dùng
NAT cho VM cluster** — VM nằm trên host-only; NAT chỉ là chân uplink Internet của router VM.
Chân này dùng NAT thay vì Bridged để router lab không chiếm IP trên LAN nhà và không phụ
thuộc LAN có cho bridge hay không.

Pattern này tổng quát hóa được cho mọi lab cần topology: cứ thêm dải là thêm một VMnet
host-only và một NIC trên router.

## 9. Virtual Network Editor — thao tác và bẫy

Mở từ **menu cửa sổ chính VMware → Edit → Virtual Network Editor** — đây là cửa sổ **khác**
với VM Settings. Các ô bị mờ thì bấm **Change Settings** (cần quyền admin/UAC).

Những bẫy đã gặp thật trong các runbook của repo:

1. **"Bridged to: Automatic"** — VMware tự đoán NIC vật lý và dễ bind nhầm sau reboot. Luôn
   chọn đích danh card đang có mạng. Không chắc card nào: `ipconfig /all` trên host, tìm
   adapter có IPv4 `192.168.x.x` **và có Default Gateway**, lấy tên ở dòng *Description*.
2. **Không bridge vào card ảo**: `Hyper-V Virtual Ethernet Adapter`, `TAP-Windows Adapter
   (OpenVPN)`… không phải card thật; bridge vào đó là VM mất mạng.
3. **Host bật Hyper-V/WSL2** có thể chiếm NIC vật lý → bridge đúng card mà VM vẫn không xin
   được DHCP. Xử lý trong [runbook VMware §15](../runbook-k8s-vmware.md#15-vận-hành--troubleshooting).
4. **Đặt IP tĩnh thì bỏ tick "Use local DHCP service"** trên VMnet đó — để DHCP ảo chạy song
   song chỉ gây nhiễu (VM lúc nhận IP tĩnh lúc nhận IP DHCP tùy thứ tự boot).
5. Dải NAT: gateway là **`.2`** (NAT device), không phải `.1` (host adapter). Netplan trỏ
   `via .1` là mất Internet dù ping host vẫn thông.
6. Mọi thiết bị ảo của VMware trên Windows là service: `VMware DHCP Service`, `VMware NAT
   Service`. Dải NAT/host-only "chết" toàn bộ thì kiểm tra hai service này trước khi nghi VM.

## 10. Sự cố thường gặp và cách nhận diện

| Triệu chứng | Nghi vấn theo thứ tự | Kiểm chứng |
| --- | --- | --- |
| VM bridged không xin được DHCP | Bridge sai card (§9.1–2) → Hyper-V chiếm card (§9.3) → Wi-Fi/AP isolation/802.1X không cho bridge (§2.3) | đổi "Bridged to" sang đích danh card thật; thử NAT — NAT chạy được thì thủ phạm là LAN |
| VM NAT ping được host nhưng không ra Internet | gateway trỏ `.1` thay vì `.2` (§9.5) → `VMware NAT Service` chết (§9.6) → VPN full-tunnel trên host nuốt traffic | `ip route` trong VM; `services.msc` trên host; tắt VPN thử lại |
| Hai VM clone tranh nhau một IP | full clone chưa chuẩn hóa identity — trùng MAC hoặc machine-id nên DHCP coi là một máy | quy trình chuẩn hóa ở [runbook VMware §4](../runbook-k8s-vmware.md#4-tạo-và-nhân-bản-3-server-theo-serversmd) |
| Host không SSH được VM host-only | VMnet đó chưa tick host adapter → Windows Firewall chặn dải mới → IP host adapter không cùng dải | `ipconfig` phải thấy `VMware Network Adapter VMnetX` đúng dải |
| VM khác dải host-only không thấy nhau | đúng thiết kế — liên dải phải qua router VM; router chưa bật `ip_forward` hoặc nftables chặn | `sysctl net.ipv4.ip_forward` và `nft list ruleset` trên router |
| VM bridged đổi IP sau khi host đổi Wi-Fi | bản chất của bridged (§2.3) — IP theo dải LAN hiện tại | chấp nhận, hoặc chuyển NAT/host-only nếu cần IP cố định |

## 11. Tự kiểm tra

> Trả lời được không nhìn lại bài là đủ để tự chọn cấu hình mạng cho lab sau.

1. Một VM cắm NAT (VMnet8). Windows host có SSH thẳng vào VM được không, và điện thoại cùng
   Wi-Fi nhà có mở được web server trên VM đó không? Vì sao mỗi bên một khác?
2. Bạn cần 3 nhóm VM mà nhóm A bị cấm nói chuyện với nhóm C. Vì sao xếp cả 3 nhóm vào Bridged
   rồi "đặt firewall" là bất khả thi về nguyên lý, chứ không phải chỉ khó cấu hình?
3. Trong dải NAT của VMware, `.1` và `.2` là ai? Đặt IP tĩnh cho VM thì default route trỏ về
   đâu?
4. Lab M1 được mô tả là "dùng NAT". Phát biểu đó đúng ở mức nào — VM cluster của M1 thực chất
   cắm vào đâu, và NAT xuất hiện ở đúng chỗ nào?
5. Host-only và LAN segment cùng cô lập VM khỏi Internet. Khác biệt duy nhất giữa chúng là
   gì, và khác biệt đó làm mất khả năng vận hành nào?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Host SSH được; điện thoại thì không** (trừ khi khai báo port forwarding). Host có host
   adapter cắm thẳng vào dải VMnet8 nên tới VM không cần NAT; điện thoại nằm trên LAN thật,
   mà NAT chỉ dịch chiều đi ra — chiều đi vào không có ánh xạ thì bị chặn.
2. Vì bridged đặt mọi VM lên **chung một broadcast domain với LAN thật**: frame đi thẳng
   VM → VM qua switch, **không có hop trung gian nào** để đặt firewall. Muốn có điểm lọc thì
   phải tách dải để traffic buộc đi qua một router — tức là rời bỏ bridged, không phải cấu
   hình thêm trên bridged.
3. `.1` là **host adapter** (đường host ⇄ VM), `.2` là **NAT device** — gateway kiêm DNS
   forwarder. Default route của VM phải trỏ **`.2`**; trỏ `.1` thì ping host vẫn thông nhưng
   mất Internet — đây là câu bẫy vì mọi mạng thông thường gateway đều là `.1`.
4. Chỉ đúng một phần nhỏ: **VM cluster của M1 cắm vào 4 VMnet host-only** (VMnet2–5). NAT
   chỉ xuất hiện trên **một NIC uplink của router VM** (VMnet8) để cả hệ ra Internet qua
   masquerade. Gọi M1 là "lab dùng NAT" dễ dẫn tới chọn nhầm NAT cho mọi VM — khi đó mất
   luôn khả năng tách dải và firewall, tức mất chính nội dung bài.
5. LAN segment **không có host virtual adapter** — host không tham gia dải. Hệ quả vận hành:
   mất SSH từ host vào VM, mọi thao tác phải qua console VMware.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng (§3, §2.2/§8.2, §3.2, §8.2, §5)
trước khi chọn cấu hình cho lab kế tiếp.
