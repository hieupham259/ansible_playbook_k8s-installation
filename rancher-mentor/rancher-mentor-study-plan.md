# Giáo án mentor — học toàn bộ cơ chế setup Rancher

Mục tiêu là giúp người học hiểu toàn bộ cơ chế được trình bày trong
[setup-rancher.md](../cloudflare-rancher-docs/setup-rancher.md), tự giải thích và vận dụng qua
sáu câu hỏi tình huống. **M01–M06 là chặng nền:** bốn bài nền tảng và hai bài tổng hợp học ngay
cùng Câu 2/6–3/6. **Sau M06 tiếp tục Câu 4/6–6/6 trực tiếp trong tài liệu**, rồi đối chiếu những
phần còn thiếu; hoàn tất M06 chưa phải hoàn tất toàn bộ tài liệu. Giáo án không thay thế thứ tự
thao tác hoặc gate trong [runbook](../runbook-k8s-vmware.md).

## Mục lục

1. [Điểm xuất phát và đích học](#điểm-xuất-phát-và-đích-học)
2. [Cấu hình lab dùng xuyên suốt](#cấu-hình-lab-dùng-xuyên-suốt)
3. [Cách mentor dạy từng lượt](#cách-mentor-dạy-từng-lượt) — [đối chiếu diagram](#đối-chiếu-diagram-sau-mỗi-phần-đã-hiểu)
4. [Bản đồ 6 bài học](#bản-đồ-6-bài-học)
5. [Nội dung và mức độ cần đạt từng bài](#nội-dung-và-mức-độ-cần-đạt-từng-bài)
6. [Câu 2/6–3/6 của chặng nền](#hai-câu-hỏi-đích)
7. [Lộ trình áp dụng sau M06](#lộ-trình-áp-dụng-sau-m06)
8. [Đối chiếu toàn bộ setup-rancher.md](#đối-chiếu-toàn-bộ-setup-ranchermd)
9. [Khi nào hoàn tất toàn bộ tài liệu?](#khi-nào-hoàn-tất-toàn-bộ-tài-liệu)
10. [Tiến độ và checkpoint](#tiến-độ-và-checkpoint)
11. [Prompt bắt đầu chat mới](#prompt-bắt-đầu-chat-mới)
12. [Nguồn và giới hạn phạm vi](#nguồn-và-giới-hạn-phạm-vi)

## Điểm xuất phát và đích học

- Người học đã học hai đường truy cập Rancher và đã làm câu 1/6; không bắt làm lại toàn bộ.
- Giáo án hiện có **6 bài**: M01–M04 là nền tảng đã hoàn tất; M05 và M06 là hai bài tổng hợp
  học song song với câu hỏi tình huống trong `setup-rancher.md`.
- **M05 — Helm, cấp certificate và gate:** dùng Câu 2/6 ngay từ đầu bài. Nội dung tương ứng
  M05–M07 của bản giáo án cũ được gộp thành các mục kiến thức bên trong M05 mới.
- **M06 — TLS, origin và khoanh vùng lỗi:** dùng Câu 3/6 ngay từ đầu bài. Nội dung tương ứng
  M08–M12 của bản giáo án cũ được gộp thành các mục kiến thức bên trong M06 mới.
- Đã có lượt trả lời cũ cho câu 2/6; giữ nguyên lịch sử và điểm cũ. Câu 3/6 chưa có lượt trả lời
  tại thời điểm chuyển giáo án; trạng thái cập nhật được theo dõi trong [README](README.md).
- Không đòi học xong nhiều bài lý thuyết rồi mới mở tình huống. Người học trả lời phần đã hiểu
  hoặc hỏi chỗ chưa rõ; mentor giải thích đúng phần cần dùng ngay trong tình huống đó.
- Sau M06, tiếp tục Câu 4/6–6/6 và dùng bảng đối chiếu toàn bộ tài liệu để bổ sung phần chưa học.
  Sáu bài M01–M06 không phải toàn bộ đích học; không mở thêm M07–M12.
- Không yêu cầu tự đọc hết tài liệu trước khi học cùng mentor, cài lại hoặc đổi cấu hình cluster
  để học lý thuyết.

[Bảng thứ tự đọc trong setup-rancher.md](../cloudflare-rancher-docs/setup-rancher.md#thứ-tự-đọc-phù-hợp--chuẩn-bị-cho-câu-26-và-36)
là nguồn tra cứu theo chỗ đang vướng. Mentor đọc phần cần dùng và hướng dẫn cụ thể; các mục
tra cứu không phải những bài độc lập cần hoàn tất trước khi được trả lời tình huống.

## Cấu hình lab dùng xuyên suốt

| Thành phần | Giá trị / phạm vi |
| --- | --- |
| Cụm Kubernetes | kubeadm; Rancher được cài vào chính cụm này và quản nó dưới tên `local` |
| Rancher | `rancher.hieupn.site`, namespace `cattle-system` |
| Chart trong runbook hiện tại | Rancher `2.14.3`; đọc lại runbook nếu phiên bản đã thay đổi |
| cert-manager | Cài ở §14.1, trước Rancher ở §14.3 |
| CA của Issuer Rancher | Secret `tls-rancher`; phân biệt với Secret certificate endpoint |
| Certificate và Secret endpoint | Cùng tên `tls-rancher-ingress`, khác kind và khác ý nghĩa |
| Origin URL của route Rancher | `https://traefik.traefik.svc.cluster.local:443` |
| Origin Server Name | `rancher.hieupn.site` |
| HTTP Host Header | `rancher.hieupn.site` |
| No TLS Verify | ON, tương ứng `noTLSVerify: true`, áp dụng cho connector tới origin HTTPS |
| Public / nội bộ | Public DNS trả IP Edge; split DNS cho client local trả ClusterIP Traefik |
| Backend sau Traefik | HTTP tới Rancher theo `ingress.servicePort: 80` |

Không đưa một Rancher agent giả định vào luồng browser. Nếu cần nhắc, nói đúng loại:
`cattle-cluster-agent` là tình huống downstream khi đăng ký/import; `fleet-agent` có thể
được tạo tự động cho `local`; `rancher-system-agent` thuộc luồng provision RKE2/K3s.
Không suy đoán inventory, request hay certificate mà một agent đang thực tế nhận.

## Cách mentor dạy từng lượt

1. Đọc README và checkpoint để xác định bài/câu hỏi đang học. Với M05, mở tình huống Câu 2/6
   ngay từ đầu; với M06, mở tình huống Câu 3/6 ngay từ đầu. Sau M06 nối tiếp Câu 4/6–6/6
   trong setup-rancher.md và bảng đối chiếu. Không giảng lại M01–M04 đã đạt.
2. Dùng tình huống làm trục của bài: nêu rõ dữ kiện, client, namespace, trạng thái giả định
   và một mục tiêu suy luận cần giải quyết. Người học có thể trả lời từng ý hoặc hỏi chỗ chưa rõ.
3. Khi câu trả lời bộc lộ khoảng trống, giải thích kiến thức cần dùng ngay tại ý đó rồi quay lại
   tình huống. Không giao một chuỗi bài lý thuyết phải học xong trước khi được tiếp tục trả lời.
4. Dùng tiếng Việt đơn giản, định nghĩa thuật ngữ mới và nói rõ chủ thể đang làm gì. Có thể dùng
   ví dụ, nhưng chỉ rõ đâu là ví dụ và đâu là object, controller hoặc dữ liệu thật trong Kubernetes.
5. Đối chiếu đúng đoạn tài liệu và vẽ Mermaid theo quy tắc bên dưới khi nhắc diagram; dùng nhãn
   ngắn và tối đa năm thành phần nằm ngang. Mỗi lượt tập trung một phần có mục tiêu của tình huống;
   đặt câu hỏi rồi dừng để người học trả lời. Tránh chuỗi câu hỏi vụn chỉ yêu cầu nhớ tên object
   hoặc trường YAML, nhất là những phần người học đã giải thích đúng.
6. Nhận xét cụ thể: đúng gì, thiếu gì, sai gì; giải thích lại và cho diễn đạt/sửa câu trả lời.
   Khi người học đang hỏi lại hoặc review tài liệu, giải quyết chỗ đó, không chen câu kiểm tra
   hoặc tự chuyển nội dung. Không lặp câu xin phép “có muốn học tiếp không” sau mỗi đoạn.
7. Sau khi người học đã trả lời đủ các ý của tình huống, chấm từng ý 0–1, tổng 0–3, rồi cho sửa.
   Không đưa đáp án mẫu của cả câu trước lượt tự trả lời; vẫn giải thích đầy đủ phần chưa hiểu.
   Sau Câu 4/6–6/6, kiểm thêm các mục liên quan trong bảng đối chiếu, không coi điểm câu hỏi là
   bằng chứng đã học hết tài liệu.
8. Kết thúc bài bằng checkpoint có căn cứ. Khi bài hoàn tất, tự động lưu `Mxx.md` và cập nhật
   [README](README.md) theo quy ước lưu đầy đủ ở dưới. Câu 4/6–6/6 lưu theo mã C04–C06.
   Không tự đánh dấu M05/M06 hoàn tất chỉ
   vì đã gộp chủ đề hoặc đã đọc đáp án. Các chỉnh sửa tài liệu khác chỉ làm khi được yêu cầu.

Nếu chưa có checkpoint hoặc bằng chứng đã học nền, xác định điểm xuất phát trước khi bắt đầu
M01. Với tiến độ hiện tại, M01–M04 đã hoàn tất; bước học tiếp là M05 cùng Câu 2/6.

### Đối chiếu diagram sau mỗi phần đã hiểu

Theo yêu cầu của người học, sau mỗi câu trả lời được đánh giá là đúng, hoặc khi kết thúc một
phần/bài đã đạt mục tiêu, mentor thêm một đoạn ngắn **“Bạn đang ở đâu trong sơ đồ”** trước khi
chuyển sang kiến thức mới. Dùng các diagram hiện có trong `setup-rancher.md` làm điểm đối chiếu:

- [Nền tảng — Helm, cert-manager và Rancher thực chất là gì](../cloudflare-rancher-docs/setup-rancher.md#nền-tảng--helm-cert-manager-và-rancher-thực-chất-là-gì).
- [Sơ đồ tuần tự — chuỗi cấp certificate chạy ngầm sau helm install](../cloudflare-rancher-docs/setup-rancher.md#sơ-đồ-tuần-tự--chuỗi-cấp-certificate-chạy-ngầm-sau-helm-install).

Mỗi lần nhắc một diagram, mentor phải **vẽ ngay diagram đó trong câu trả lời bằng Mermaid**,
nêu tên phần/tên diagram và chỉ đúng **nhãn node hoặc lời gọi/mũi tên** đang xét. Không gửi link
thay cho diagram và không yêu cầu người học mở trang khác để xem. Các link trong bảng dưới
là nguồn cho mentor đối chiếu, không phải cách trình bày thay cho hình trong lượt dạy.
Giải thích trong một hoặc hai câu vì sao câu trả lời vừa đúng tương ứng với đoạn đã vẽ.
Nếu chỉ vẽ phần liên quan, ghi rõ là **trích đoạn/rút gọn**; nếu đổi vai client/server để áp dụng
vào lab, nói rõ đây là bản áp dụng cho lab, không nhận là toàn bộ sơ đồ gốc. Giữ tách biệt chương
trình chạy, object cấu hình, dữ liệu và hai giai đoạn cấp certificate / dùng certificate trong TLS.

| Nội dung vừa học | Nguồn nên mở | Đoạn cần chỉ rõ |
| --- | --- | --- |
| Chart, values, render, release — M01 | [Nền tảng → Helm](../cloudflare-rancher-docs/setup-rancher.md#helm--trình-quản-lý-gói-của-kubernetes); [Sơ đồ 1](../cloudflare-rancher-docs/setup-rancher.md#sơ-đồ-1--ai-tạo-gì-trong-chuỗi-cấp-certificate) | Phần Helm định nghĩa đầu vào/render/release. Trong Sơ đồ 1, chỉ `Helm gửi cấu hình Rancher → Kubernetes lưu các tài nguyên` là bước gửi bản khai; bước render không được vẽ thành node riêng. |
| Object, CRD, controller; webhook và cainjector — M02 | [Nền tảng → cert-manager](../cloudflare-rancher-docs/setup-rancher.md#cert-manager--crd-và-chuỗi-cấp-phát) | Chuỗi minh họa Issuer/Certificate/CertificateRequest/Secret và đoạn mô tả ba Deployment. CRD, webhook và cainjector không có node riêng trong Sơ đồ 1–3; nói rõ giới hạn này. |
| Ingress-shim tạo Certificate — M02/M04/M05 | [Sơ đồ 1](../cloudflare-rancher-docs/setup-rancher.md#sơ-đồ-1--ai-tạo-gì-trong-chuỗi-cấp-certificate); [Sơ đồ 3](../cloudflare-rancher-docs/setup-rancher.md#sơ-đồ-3--theo-dõi-từng-bước-qua-kubernetes-api) | Sơ đồ 1: `Ingress khai hostname và Secret TLS → ingress-shim tạo Certificate`. Sơ đồ 3: `Kubernetes API → cert-manager: ingress-shim quan sát Ingress Rancher`, rồi `cert-manager → Kubernetes API: ingress-shim tạo Certificate tls-rancher-ingress`. |
| Certificate, CA và key — M03 | [Sơ đồ 1](../cloudflare-rancher-docs/setup-rancher.md#sơ-đồ-1--ai-tạo-gì-trong-chuỗi-cấp-certificate); [Sơ đồ 3](../cloudflare-rancher-docs/setup-rancher.md#sơ-đồ-3--theo-dõi-từng-bước-qua-kubernetes-api) | Tùy phần nhỏ, chỉ các node `Rancher chuẩn bị CA trong tls-rancher`, `Controller dùng CA để ký`, `Ghi certificate và key vào tls-rancher-ingress`, `Traefik nạp certificate và private key`; Sơ đồ 3 có thao tác `Ghi tls.crt và tls.key vào tls-rancher-ingress`. |
| Vai trò từng object và chuỗi cấp phát — M04/M05 | [Nền tảng → cert-manager](../cloudflare-rancher-docs/setup-rancher.md#cert-manager--crd-và-chuỗi-cấp-phát); [Sơ đồ 1](../cloudflare-rancher-docs/setup-rancher.md#sơ-đồ-1--ai-tạo-gì-trong-chuỗi-cấp-certificate); [Sơ đồ 3](../cloudflare-rancher-docs/setup-rancher.md#sơ-đồ-3--theo-dõi-từng-bước-qua-kubernetes-api) | Chọn đúng đoạn vừa hiểu: Ingress/shim, Issuer/CA, CertificateRequest/ký, ghi Secret hoặc Traefik nạp; không gom toàn bộ vào một bước. |
| Helm --wait và công việc bất đồng bộ — M05 | [Sơ đồ 2 — Helm và cert-manager chạy song song](../cloudflare-rancher-docs/setup-rancher.md#sơ-đồ-2--helm-và-cert-manager-chạy-song-song) | Chỉ nhánh `Nhánh chờ của Helm` hoặc `Nhánh cấp certificate`, và ghi chú `Hai nhánh không được bảo đảm hoàn tất cùng lúc` theo ý vừa trả lời. |
| Ready, Secret và gate — M05 | [Sơ đồ 1](../cloudflare-rancher-docs/setup-rancher.md#sơ-đồ-1--ai-tạo-gì-trong-chuỗi-cấp-certificate); [Sơ đồ 3](../cloudflare-rancher-docs/setup-rancher.md#sơ-đồ-3--theo-dõi-từng-bước-qua-kubernetes-api); [Đọc gate §14.4](../cloudflare-rancher-docs/setup-rancher.md#đọc-gate-144-theo-từng-câu-xác-nhận) | Phân biệt `Certificate đạt Ready=True` với `Traefik nạp certificate và private key` và việc kiểm endpoint bằng gate. Đây không phải bằng chứng các trạng thái thực luôn cập nhật theo thứ tự tuyệt đối của sơ đồ. |

Cấu trúc phản hồi: nêu tên phần/diagram → vẽ khối Mermaid ngay trong câu trả lời → chỉ nhãn
node/mũi tên liên quan → nhận xét phần đã hiểu và phần còn nhầm. Không dùng mẫu chỉ có link.

Nếu diagram hiện có không vẽ chi tiết đang học, nêu rõ giới hạn và giải thích ngay nội dung
cần thiết. Có thể vẽ minh họa bổ sung nhưng phải ghi rõ đó là minh họa mới, không tự đặt tên
một node rồi khẳng định node đó có trong sơ đồ gốc. Với M06, các sơ đồ cấp certificate này
chỉ dùng để liên hệ phần chuẩn bị/đọc Secret; khi học handshake, request hoặc khoanh vùng lỗi,
dùng đúng diagram TLS/đường truy cập ở phần tương ứng, nêu tên nguồn và vẽ lại ngay trong câu trả lời.

“Đã hiểu đoạn nào” là tiến độ học, không phải xác nhận cluster thực tế đã chạy tới bước ấy.
Phân biệt mũi tên quan hệ tạo/sử dụng tài nguyên ở Sơ đồ 1 với lời gọi qua API ở Sơ đồ 3;
không diễn giải chúng thành đường HTTP request của browser. Không coi webhook/cainjector
đã được vẽ thành node, hoặc private key đã được gửi cho client, chỉ từ nhãn tổng quát của diagram.

Khi lưu bài `Mxx.md`, giữ cả các lần đối chiếu diagram thực sự đã đưa trong hội thoại.
Checkpoint ghi thêm diagram/đoạn vừa đối chiếu và đoạn tiếp theo. Chỉ ghi phần được người học
trả lời đúng; nội dung chưa trả lời hoặc mới giảng chưa được đánh dấu đã hiểu.

## Bản đồ 6 bài học

| Bài | Nội dung | Học cùng câu hỏi / tài liệu | Tiêu chí đạt |
| --- | --- | --- | --- |
| [M01](#m01-helm-làm-gì) | Helm: chart, values, manifest, release | Setup → Nền tảng → Helm | Phân biệt đầu vào, bản khai và bản cài; đã hoàn tất |
| [M02](#m02-object-controller-và-crd) | Object, controller, CRD và các thành phần cert-manager | Setup → Nền tảng → cert-manager | Phân biệt cấu hình với chương trình thực hiện; đã hoàn tất |
| [M03](#m03-certificate-ca-và-key-ở-mức-cần-thiết) | Certificate, CA, public/private key | Setup → Certificate, CA và “ký”; tài liệu CA/TLS | Phân biệt được mã hóa, sở hữu khóa và xác minh danh tính; đã hoàn tất |
| [M04](#m04-ingress-issuer-certificate-và-hai-secret) | Ingress, Issuer, Certificate, CertificateRequest và hai Secret | Setup → chuỗi cấp certificate; [M04.md](M04.md) | Giải thích vai trò object/dữ liệu, lần xin cấp khác kết nối TLS; đã hoàn tất |
| [M05](#m05-helm-cấp-certificate-và-gate) | Chuỗi cấp phát, phạm vi `--wait`, readiness và gate §14.4 | [Câu 2/6](../cloudflare-rancher-docs/setup-rancher.md#câu-26--helm-đã-xong-thì-certificate-đã-sẵn-sàng-chưa) **ngay từ đầu bài** | Tự giải thích ba ý của tình huống, được nhận xét, chấm 0–3 và sửa |
| [M06](#m06-tls-origin-và-khoanh-vùng-lỗi) | TLS, hai đường truy cập, origin parameters, Access và khoanh vùng `502` | [Câu 3/6](../cloudflare-rancher-docs/setup-rancher.md#câu-36--tunnel-khỏe-nhưng-browser-vẫn-nhận-502-khoanh-vùng-lỗi-ở-đâu) **ngay từ đầu bài** | Tự giải thích các chặng, cấu hình và giới hạn bằng chứng, được nhận xét, chấm 0–3 và sửa |

M05 mới gộp M05–M07 cũ; M06 mới gộp M08–M12 cũ. Các nội dung được giữ thành mục con để
tra cứu/bổ sung trong tình huống, không còn là tám bài riêng phải lần lượt vượt qua. M01–M04
và các bản lưu đã có được giữ nguyên. Các mã/mốc cũ trong transcript là lịch sử của giáo án cũ.

Các liên kết “Setup” trỏ tới [setup-rancher.md](../cloudflare-rancher-docs/setup-rancher.md);
tài liệu TLS là [ca-tls-https-request-response.md](../cloudflare-rancher-docs/ca-tls-https-request-response.md).
Thứ tự thao tác và gate triển khai thực tế vẫn lấy từ runbook. Sau bản đồ sáu bài là
[chặng áp dụng Câu 4/6–6/6](#lộ-trình-áp-dụng-sau-m06); đây là phần tiếp tục để bao phủ toàn bộ file.

## Nội dung và mức độ cần đạt từng bài

### M01. Helm làm gì?

- **Dạy:** chart là bộ mẫu; values là lựa chọn; render tạo manifest; install gửi tài nguyên tới
  Kubernetes; release là lần cài có tên. Phân biệt `helm template` và `helm install` ở mức mục đích.
- **Ví dụ:** đổi `hostname` trong values làm host trong Ingress thay đổi; Kubernetes dùng
  manifest đó để tạo tài nguyên. Helm không phải Pod Rancher.
- **Chưa cần:** tự viết chart, helper phức tạp, hooks, OCI, lịch sử rollback hoặc lỗi version-skew.
- **Đạt khi:** người học chỉ được đầu vào, đầu ra và ai chạy Pod từ ví dụ chart Rancher + values.

### M02. Object, controller và CRD

- **Dạy:** object là bản khai/trạng thái lưu trong Kubernetes; controller là chương trình quan sát
  và hành động; CRD thêm loại object. `Certificate` không phải một Pod chạy cấp chứng chỉ.
- **Ví dụ:** bản khai “cần hai replica” và controller làm xuất hiện Pod; nối sang yêu cầu certificate.
- **Thành phần:** `cert-manager` xử lý cấp phát, `cert-manager-webhook` kiểm tra object,
  `cert-manager-cainjector` quản lý CA bundle cho các cấu hình được hỗ trợ; ingress-shim là
  thành phần tạo Certificate từ Ingress, không tự đại diện toàn bộ chuỗi ký/ghi Secret.
- **Đạt khi:** phân loại đúng các tên này thành object, controller hoặc thành phần của controller.

### M03. Certificate, CA và key ở mức cần thiết

- **Dạy:** certificate gắn tên server với public key; CA ký xác nhận; private key tương ứng dùng
  để chứng minh server sở hữu khóa. CA key và key của endpoint có vai trò khác nhau.
- **Ví dụ:** certificate cho `rancher.hieupn.site`; client cần vừa nhận đúng tên vừa tin CA khi verify.
- **Chưa cần:** thuật toán ký, tính khóa phiên, ASN.1 hoặc dựng PKI tổng quát.
- **Đạt khi:** phân biệt “có certificate”, “client tin certificate” và “kết nối được mã hóa”.

### M04. Ingress, Issuer, Certificate và hai Secret

- **Dạy:** Ingress khai hostname/Secret và annotation Issuer; Issuer là cấu hình cách cấp;
  Certificate là nhu cầu; CertificateRequest là một lần xin ký; Secret chứa kết quả.
- **Ví dụ:** `Certificate/tls-rancher-ingress` và `Secret/tls-rancher-ingress` cùng tên nhưng khác
  kind. `tls-rancher` giữ CA mà Issuer dùng; `tls-rancher-ingress` giữ cert/key endpoint.
- **Sơ đồ:** mở “Sơ đồ 1 — Ai tạo gì”; nói rõ mũi tên tạo tài nguyên khác mũi tên HTTP request.
- **Đạt khi:** kể đúng vai trò từng object, không coi Issuer là một server mạng độc lập.

### M05. Helm, cấp certificate và gate

**Tình huống học xuyên suốt:** [Câu 2/6 — Helm đã xong thì certificate đã sẵn sàng chưa?](../cloudflare-rancher-docs/setup-rancher.md#câu-26--helm-đã-xong-thì-certificate-đã-sẵn-sàng-chưa)

Đưa tình huống ngay khi bắt đầu M05. Người học trả lời phần đã hiểu hoặc hỏi điểm chưa rõ;
mentor dùng các mục bên dưới để bổ sung đúng chỗ rồi quay lại ba ý của câu hỏi. Không yêu cầu
học xong các mục con rồi mới được trả lời. Kiến thức từ M01–M04 được vận dụng, không mặc định dạy lại.

<a id="m05-chuỗi-xin-và-cấp-certificate"></a>

#### Chuỗi xin và cấp certificate

- **Dạy:** Helm tạo Ingress/Issuer; Rancher chuẩn bị CA; ingress-shim tạo Certificate; controller
  cấp phát tạo key/yêu cầu ký, dùng CA theo Issuer, ghi Secret và cập nhật status; Traefik nạp dữ liệu.
- **Ví dụ:** Certificate đã xuất hiện nhưng CA/Issuer chưa dùng được thì bước cấp vẫn chưa xong.
- **Sơ đồ:** học từng nhánh của Sơ đồ 1, sau đó Sơ đồ 3 để thấy đọc/ghi object qua Kubernetes API.
  Không ép các controller vào một thứ tự lịch chạy tuyệt đối.
- **Điểm cần làm rõ trong tình huống:** tự kể ai tạo gì đến lúc Traefik có cert; hiểu cấp/gia hạn không được kích hoạt cho mỗi request.

<a id="m06-helm---wait-và-công-việc-bất-đồng-bộ"></a>

#### Helm --wait và công việc bất đồng bộ

- **Dạy:** hai nhánh có thể chạy cùng lúc; Helm thành công chỉ xác nhận phạm vi chờ của release,
  không chứng minh mọi tài nguyên runtime hoặc chức năng Rancher đã sẵn sàng.
- **Ví dụ:** Deployment Rancher Ready, nhưng Certificate do ingress-shim tạo vẫn đang xử lý.
- **Sơ đồ:** “Sơ đồ 2 — Helm và cert-manager chạy song song”; cho người học thử đổi thứ tự hoàn tất.
- **Chính xác theo phiên bản:** chi tiết readiness/`--wait` khác giữa Helm 3 và Helm 4/strategy.
  Không khẳng định mọi custom resource đều bị bỏ qua trong mọi phiên bản. Trong lab, Certificate
  do ingress-shim tạo ngoài manifest chart Ingress nên phải có gate riêng.
- **Điểm cần làm rõ trong tình huống:** giải thích được “Helm xong” khác “certificate xong” mà không nói cert-manager
  chỉ bắt đầu sau lúc lệnh Helm kết thúc.

<a id="m07-khi-nào-sẵn-sàng-và-đọc-gate-144"></a>

#### Readiness và gate §14.4

- **Dạy:** object xuất hiện → Certificate Ready → Secret kết quả → Traefik nạp → endpoint trả lời.
  Đây là các mốc cần phân biệt, không phải một chuỗi đồng bộ mà mọi status luôn đổi đúng thứ tự.
- **Ví dụ:** `wait --for=create` khác `wait --for=condition=Ready`; Secret tồn tại chưa tự chứng
  minh Traefik đã trình đúng cert; `curl -k` bỏ verification; `--resolve` giữ hostname nhưng ép IP.
- **Thực hành đọc:** lấy nguyên văn gate §14.4 hiện tại, giải thích một lệnh mỗi lần, nơi chạy và
  kết quả mong đợi. Không mặc định yêu cầu thực thi.
- **Điểm cần làm rõ trong tình huống:** nói được mỗi bằng chứng xác nhận điều gì và chưa xác nhận điều gì.

**Hoàn tất M05 khi:** người học giải thích được ai tạo/cấp/ghi certificate, phạm vi chờ
của Helm và bằng chứng cần có trước publish. Chấm ba ý của Câu 2/6 theo thang 0–3, cho sửa,
lưu đầy đủ lượt mới vào `M05.md`; giữ nguyên lịch sử/điểm cũ trong `setup-rancher.md`.

### M06. TLS, origin và khoanh vùng lỗi

**Tình huống học xuyên suốt:** [Câu 3/6 — Tunnel khỏe nhưng browser vẫn nhận 502: khoanh vùng lỗi ở đâu?](../cloudflare-rancher-docs/setup-rancher.md#câu-36--tunnel-khỏe-nhưng-browser-vẫn-nhận-502-khoanh-vùng-lỗi-ở-đâu)

Đưa tình huống ngay khi bắt đầu M06. Hai phiên TLS, DNS, SNI/Host, origin parameters và Access
được giải thích ngay khi cần để lập luận về tình huống. Nội dung đã hiểu ở M03–M04 hoặc Câu 1/6
chỉ được ôn khi cần. Các chi tiết handshake là nguồn tra cứu trong bài, không phải một bài riêng
bắt buộc phải hoàn tất trước khi được trả lời Câu 3/6.

<a id="m08-một-phiên-tls-ai-là-client-ai-là-server"></a>

#### Một phiên TLS: client, server và điểm kết thúc

- **Dạy:** client chủ động kết nối; server trình certificate; client quyết định verification
  theo cấu hình/trust; terminate là nơi xử lý và giải mã TLS.
- **Ví dụ:** chỉ xét `cloudflared` gọi Traefik trước khi đưa thêm browser vào.
- **Liên hệ:** cert-manager cấp cert trước; nó không ngồi giữa request TLS. Không kéo thêm Rancher agent.
- **Điểm cần làm rõ trong tình huống:** chỉ được bên trình cert, bên kiểm cert và điểm kết thúc phiên.

<a id="m09-handshake-tls-13-vừa-đủ-để-đọc-diagram"></a>

#### Handshake TLS 1.3 khi cần đọc luồng

- **Dạy:** ClientHello/ServerHello có key_share; tính khóa handshake rồi mới gửi phần certificate
  được mã hóa; CertificateVerify/Finished phục vụ xác thực handshake.
- **Ví dụ:** đọc một handshake mới đầy đủ; chỉ sau đó mới nói việc tái sử dụng kết nối/session.
- **Giới hạn:** đây là ví dụ TLS 1.3; không suy ra mọi kết nối thực tế đều thương lượng TLS 1.3.
  Không biến bài thành giải toán mật mã hoặc so sánh sâu các cipher suite.
- **Điểm cần làm rõ trong tình huống:** không nhầm trình tự “verify certificate xong mới trao đổi key_share” và phân biệt
  ký certificate lúc cấp với xác thực handshake lúc dùng.

<a id="m10-hai-phiên-tls-và-hai-hệ-dns-trong-lab"></a>

#### Hai phiên TLS và hai hệ DNS trong lab

- **Dạy:** browser–Edge dùng cert public; connector–Traefik dùng cert origin; tunnel có transport
  mã hóa riêng. Public DNS và CoreDNS trả lời cho các tên/đường tương ứng.
- **Ví dụ:** browser resolve `rancher.hieupn.site`; connector resolve
  `traefik.traefik.svc.cluster.local`. Client API local giả định là một luồng khác có split DNS.
- **Sơ đồ:** tách hai handshake trước, rồi ghép; kết thúc ở Traefik nếu chưa cần backend.
- **Điểm cần làm rõ trong tình huống:** không cho browser nhìn thấy certificate trong Secret origin và không kéo TLS xuyên
  từ browser qua Edge tới Traefik như một phiên duy nhất.

<a id="m11-origin-url-origin-server-name-và-http-host"></a>

#### Origin URL, Origin Server Name và HTTP Host

- **Dạy:** URL origin chỉ đích/giao thức kết nối; Origin Server Name chọn tên TLS;
  HTTP Host Header chọn danh tính HTTP cho routing; No TLS Verify điều khiển verification origin.
- **Ví dụ:** kết nối Service DNS Traefik nhưng giữ danh tính website Rancher. So sánh app demo
  `http://…:80`: không có origin TLS, nên default `noTLSVerify: false` không làm phát sinh TLS.
- **Sơ đồ/bảng:** ba cột “đích mạng / tên TLS / HTTP Host”, gắn từng tham số vào đúng bước.
- **Điểm cần làm rõ trong tình huống:** điền đúng ba setting của route Rancher và giải thích từng vai trò, không gộp SNI với Host.

<a id="m12-access-302-502-và-khoanh-vùng-theo-bằng-chứng"></a>

#### Access, 302, 502 và khoanh vùng bằng bằng chứng

- **Dạy:** Access cho người truy cập là lớp kiểm soát ở Edge; response từ Access có thể xuất hiện
  trước khi request đi vào tunnel. Healthy tunnel chưa kiểm chứng chặng connector–origin.
- **Ví dụ:** dùng đúng tình huống câu 3/6 phía dưới; kiểm tra từng chặng theo bằng chứng, không đoán
  Pod Rancher hỏng chỉ vì nhìn thấy `502`. Lỗi `502` có nhiều nguyên nhân.
- **Sơ đồ:** đánh dấu chặng đã có bằng chứng và chặng chưa được kiểm; lấy log/test phù hợp với nơi phát lỗi.
- **Điểm cần làm rõ trong tình huống:** giải thích được `302`, Healthy tunnel và `curl -k` nội bộ giới hạn kết luận ra sao.

**Hoàn tất M06 khi:** người học phân biệt được bằng chứng từ Access, tunnel và gate nội bộ;
xác định chặng ưu tiên kiểm tra và vai trò từng origin parameter, không suy đoán Pod Rancher
hỏng chỉ từ `502`. Chấm ba ý của Câu 3/6 theo thang 0–3, cho sửa và lưu toàn bộ vào `M06.md`.
Sau đó chuyển Câu 4/6–6/6; trạng thái hoàn tất M06 chỉ xác nhận chặng nền.

## Hai câu hỏi đích

**Hai câu hỏi đích dưới đây thuộc chặng nền, không phải đích cuối của toàn bộ tài liệu.**
Câu 2/6 được dùng ngay từ đầu M05; Câu 3/6 được dùng ngay từ đầu M06. Hai câu là trục học
song song với `setup-rancher.md`, không phải bài thi chỉ được mở sau khi học xong một dãy bài.
Đọc [phần Tự kiểm tra](../cloudflare-rancher-docs/setup-rancher.md#tự-kiểm-tra) để lấy tình huống
và đối chiếu đáp án sau lượt tự trả lời. Nội dung câu hỏi được giữ dưới đây để chat mới có đủ ngữ cảnh.

### Câu 2/6 — Helm đã xong thì certificate đã sẵn sàng chưa?

Giả sử bạn cài Rancher theo §14.3, dùng mode Ingress và cấu hình:

```yaml
ingress:
  tls:
    source: rancher
    secretName: tls-rancher-ingress
```

Kết quả hiện tại:

- `helm install ... --wait` báo thành công.
- Deployment Rancher đã Ready.
- Trong namespace `cattle-system`, **Issuer và Ingress đã tồn tại**.
- Nhưng **chưa thấy Certificate và Secret `tls-rancher-ingress`**.

Đây là tình huống kiểm tra ranh giới readiness, không phải khẳng định cluster thực tế đang lỗi.
Hãy trả lời:

1. Thành phần nào có nhiệm vụ tạo **Certificate**, rồi thành phần nào cấp certificate và
   ghi vào **Secret**?
2. Vì sao `helm --wait` thành công mà các tài nguyên đó vẫn có thể chưa sẵn sàng?
3. Bạn sẽ kiểm tra hoặc chờ những tài nguyên, điều kiện nào trước khi publish route Rancher
   ra Cloudflare?

### Câu 3/6 — Tunnel khỏe nhưng browser vẫn nhận 502: khoanh vùng lỗi ở đâu?

Giả sử dùng đúng cấu hình Rancher trong runbook:

- Certificate `tls-rancher-ingress` có `Ready=True`, Secret TLS đã tồn tại.
- Gate §14.4: `curl -k --resolve` gọi `https://rancher.hieupn.site/healthz` qua ClusterIP
  Traefik từ master trả HTTP `200`.
- Tunnel `homelab-k8s` báo Healthy, connector `cloudflared` đang Connected.
- Laptop ngoài cluster, chưa đăng nhập Access: gọi URL Rancher nhận `302` tới trang đăng nhập Access.
- Browser đăng nhập Access thành công rồi mở Rancher thì nhận `502 Bad Gateway`.

Trả lời ba ý:

1. Response `302` từ Access chứng minh những chặng nào đã hoạt động? Nó đã chứng minh request
   qua tunnel tới Traefik và Rancher chưa? Giải thích.
2. Ưu tiên kiểm tra chặng nào khi nhận `502` sau khi đăng nhập Access? Bằng chứng trên đã đủ
   kết luận Pod Rancher bị lỗi chưa? Vì sao?
3. Route dùng `https://traefik.traefik.svc.cluster.local:443`. Nêu giá trị theo runbook và vai
   trò riêng của TLS → Origin Server Name, TLS → No TLS Verify, HTTP Settings → HTTP Host Header.

Thang chấm cả hai câu: mỗi ý 0–1, tổng 0–3. Nêu rõ đúng, thiếu và sai; phân biệt trả lời thiếu
chi tiết với sai bản chất. Không sửa điểm của lượt trả lời cũ trong setup-rancher.md nếu người
học chưa yêu cầu ghi kết quả lượt mới.

## Lộ trình áp dụng sau M06

**Hoàn tất M06 là hoàn tất chặng nền, chưa phải hoàn tất toàn bộ `setup-rancher.md`.**
Sau đó tiếp tục ngay trong phần Tự kiểm tra của tài liệu bằng Câu 4/6–6/6. Không mở thêm
M07–M12 và không yêu cầu học một loạt bài lý thuyết trước các tình huống này.

| Mốc | Tình huống và phần kiến thức cần bổ sung | Tiêu chí đạt |
| --- | --- | --- |
| [Câu 4/6](../cloudflare-rancher-docs/setup-rancher.md#câu-46--vì-sao-phải-cài-và-kiểm-tra-theo-thứ-tự-này) | Kiểm tra trước cài, backup etcd/cấu hình, thứ tự §14.0–§14.7, render/values, bootstrap password, inventory và gate | Giải thích được vì sao mỗi bước đứng ở vị trí đó; phân biệt manifest, tài nguyên runtime và bằng chứng hoàn thành |
| [Câu 5/6](../cloudflare-rancher-docs/setup-rancher.md#câu-56--browser-dùng-được-thì-client-và-cluster-khác-đã-dùng-được-chưa) | Server URL, DNS/trust/quyền của từng client, Rancher quản `local`, các loại agent, `strict` và giới hạn import downstream | Theo được đường gọi của từng chủ thể; chỉ ra điều kiện còn thiếu cho client máy và downstream, không suy từ browser hoặc setting |
| [Câu 6/6](../cloudflare-rancher-docs/setup-rancher.md#câu-66--trang-web-và-node-đều-khỏe-vì-sao-rancher-vẫn-lỗi) | Dashboard/API, Steve, SelfUser, Aggregation Layer, extension API, startup race, timeline và management condition của `local` | Phân biệt các nguồn trạng thái; giải thích suy luận của case đã lưu và giới hạn kết luận; biết cần bằng chứng nào trước khi cân nhắc thao tác phục hồi |

Mỗi câu vẫn có ba ý, chấm 0–3 và cho sửa. Mentor mở tình huống trước, giải thích các phần còn
thiếu ngay trong lúc học; không đưa nguyên đáp án mẫu trước lượt tự trả lời. Những khái niệm
như etcd, service account, RBAC, Aggregation Layer, APIService, schema, cache, leader và
reconcile cần được định nghĩa khi dùng. Mỗi lượt chỉ giải quyết một mục tiêu suy luận rõ.

Câu 1/6 là nền đã học; Câu 2/6 và 3/6 học ngay trong M05/M06. Tiếp theo là Câu 4/6, 5/6 và 6/6.
Phần còn thiếu trong bảng đối chiếu bên dưới được học ngay tại tình huống tương ứng, kể cả khi
người học đã trả lời đúng một phần của câu. Điểm câu hỏi và mức bao phủ tài liệu là hai dữ kiện
tiến độ riêng: đạt một câu chưa tự chứng minh đã giải thích mọi mục liên quan.

## Đối chiếu toàn bộ setup-rancher.md

Bảng này là bản đồ chương trình, **không đánh dấu người học đã hiểu các mục chỉ vì có tên trong
giáo án**. Mentor dùng nó để tìm phần cần bổ sung và ghi checkpoint thực sự đã đạt trong README.
Các phần trình bày lặp cùng cơ chế (lời kể, định nghĩa, sơ đồ, lệnh và mục đào sâu) được đối chiếu
trong cùng tình huống; không bắt trả lời lại cả bài chỉ vì cách trình bày khác.

### Các phần chính và từng bước §14

| Phần trong tài liệu | Học ở đâu | Điều cần tự giải thích |
| --- | --- | --- |
| [Nhìn nhanh và so sánh với app demo](../cloudflare-rancher-docs/setup-rancher.md#nhìn-nhanh--14-cài-gì-và-để-làm-gì) | M06, Câu 4–5 | Lựa chọn Access, split DNS và TLS của lab liên quan với nhau thế nào; phân biệt kiến trúc khác với cấu hình đang dùng |
| [Cách đọc toàn bộ §14 theo ngôn ngữ đơn giản](../cloudflare-rancher-docs/setup-rancher.md#cách-đọc-toàn-bộ-14-theo-ngôn-ngữ-đơn-giản) | Câu 1–6 | Kể lại từ chuẩn bị tới browser/client, quản lý cluster, sự cố và giới hạn downstream |
| [Từ điển khái niệm nền](../cloudflare-rancher-docs/setup-rancher.md#từ-điển-khái-niệm-nền) | Tra ngay trong M01–M06, Câu 4–6 | TLS/CA/SNI và ví dụ web ngoài K8s: M03/M06; CRD/webhook: M02; etcd/snapshot: Câu 4; `local`/agent/Server URL/RBAC/service account: Câu 5; origin/tunnel/SSO/IdP/MFA/split DNS: M06/Câu 5; các từ nhanh còn lại tra khi xuất hiện |
| [Thứ tự đọc cho Câu 2/6–3/6](../cloudflare-rancher-docs/setup-rancher.md#thứ-tự-đọc-phù-hợp--chuẩn-bị-cho-câu-26-và-36) | M05–M06 | Đây là nguồn nối kiến thức với tình huống chặng nền, không phải phạm vi toàn bộ khóa học |
| [Nền tảng Helm/cert-manager/Rancher](../cloudflare-rancher-docs/setup-rancher.md#nền-tảng--helm-cert-manager-và-rancher-thực-chất-là-gì) | M01–M05, Câu 4–5 | Ai tạo tài nguyên của release, ai sinh tài nguyên runtime, ai quản cluster |
| [Chuỗi cấp certificate và các sơ đồ 1–3](../cloudflare-rancher-docs/setup-rancher.md#sơ-đồ-tuần-tự--chuỗi-cấp-certificate-chạy-ngầm-sau-helm-install) | M03–M05; liên hệ M06 | Object Certificate, certificate dữ liệu và bản tin TLS khác nhau; controller có thể chạy trong lúc Helm chờ |
| [Hai đường vào và luồng nội bộ Rancher](../cloudflare-rancher-docs/setup-rancher.md#sơ-đồ-toàn-cảnh--hai-đường-vào-cùng-một-hostname) | Câu 1, M06, Câu 5–6 | Tách đường browser, client local, API quản lý và dependency riêng của SelfUser |
| [§14.0 — gate trước thay đổi](../cloudflare-rancher-docs/setup-rancher.md#140--gate-read-only-đo-trước-khi-đụng) | Câu 4 | Mỗi kiểm tra loại trừ rủi ro nào; version, quyền, tài nguyên, Traefik/tunnel và dấu vết cài cũ |
| [§14.0.1 — backup](../cloudflare-rancher-docs/setup-rancher.md#1401--backup-cơ-chế-từng-lệnh) | Câu 4 | etcd, PKI/cấu hình, mTLS, static Pod/hostPath, copy ra ngoài VM, quyền file và checksum |
| [§14.1 — cert-manager](../cloudflare-rancher-docs/setup-rancher.md#141--cài-cert-manager) | M02/M05, Câu 4 | Phụ thuộc CRD/controller/webhook và lý do cài trước Rancher |
| [§14.2 — split DNS](../cloudflare-rancher-docs/setup-rancher.md#142--split-dns) | M06, Câu 4–5 | Vì sao chuẩn bị trước Rancher; `hosts`, `fallthrough`, TTL, test hostname Rancher từ Pod |
| [§14.3 — Rancher](../cloudflare-rancher-docs/setup-rancher.md#143--cài-rancher) | M01/M04/M05, Câu 4 | Repo/chart/version, từng key values, heredoc, render gate, mã thoát và giới hạn hỗ trợ của nền tảng host |
| [§14.4 — origin](../cloudflare-rancher-docs/setup-rancher.md#144--gate-origin-nội-bộ) | M05, Câu 3–4 và 6 | Chờ create/Ready, Secret, class/host, `--resolve`/`-k`, `/healthz` và giới hạn của phép thử |
| [§14.5 — Access rồi publish](../cloudflare-rancher-docs/setup-rancher.md#145--access-rồi-mới-publish) | M06, Câu 4 | Tách policy với route, đúng thứ tự bảo vệ và kiểm lại route detail/origin parameters |
| [§14.6 — đăng nhập](../cloudflare-rancher-docs/setup-rancher.md#146--đăng-nhập-lần-đầu) | Câu 3–5 | Hai lớp đăng nhập, bootstrap-secret, decode base64, đổi mật khẩu, Server URL và đọc log theo triệu chứng |
| [§14.7 — hoàn thành](../cloudflare-rancher-docs/setup-rancher.md#147--gate-hoàn-thành) | Câu 4–6 | Inventory/settings, `local` Active, phạm vi từng agent; PASS của `local` chưa xác nhận downstream |
| [15 cơ chế đào sâu](../cloudflare-rancher-docs/setup-rancher.md#đào-sâu-15-cơ-chế-của-14) | Bảng riêng bên dưới | Liên hệ cơ chế với thao tác và giới hạn bằng chứng |
| [Sự cố sau reboot và luồng điều tra](../cloudflare-rancher-docs/setup-rancher.md#sơ-đồ-sự-cố-sau-reboot--selfuser-404-và-cluster-local-unavailable) | Câu 6 | Toàn bộ bước 0–9: giữ bằng chứng, request lỗi, dependency, hiện tại/lịch sử, timeline, cache, gate phục hồi, hậu kiểm và kết luận đúng mức |
| [Tự kiểm tra](../cloudflare-rancher-docs/setup-rancher.md#tự-kiểm-tra) | Câu 1–6 | Vận dụng kiến thức; giữ riêng đáp án tham khảo, câu trả lời thực tế, nhận xét và điểm |

### Mười lăm cơ chế đào sâu

| Mục đào sâu | Học ở đâu | Điều cần làm rõ |
| --- | --- | --- |
| [1. Rancher là workload](../cloudflare-rancher-docs/setup-rancher.md#1-rancher-là-workload-trong-chính-cluster-nó-quản-lý) | Câu 4–5 | Cluster host, `local`, phụ thuộc trong cluster và phân biệt tương thích với nền tảng được chứng nhận |
| [2. Snapshot etcd](../cloudflare-rancher-docs/setup-rancher.md#2-vì-sao-đường-lui-duy-nhất-là-snapshot-etcd) | Câu 4 | Phạm vi backup so với uninstall và sửa lại một ConfigMap |
| [3. cert-manager trước Rancher](../cloudflare-rancher-docs/setup-rancher.md#3-vì-sao-cert-manager-phải-có-mặt-trước-rancher) | M02/M05, Câu 4 | CRD/webhook/controller cần sẵn sàng cho chuỗi cấp phát |
| [4. Hai hệ certificate](../cloudflare-rancher-docs/setup-rancher.md#4-hai-hệ-certificate-ở-tầng-ứng-dụng-không-chạm-nhau) | M03–M06 | Client nào nhận certificate nào; hai phiên TLS và trust riêng |
| [5. agentTLSMode strict](../cloudflare-rancher-docs/setup-rancher.md#5-agenttlsmode-strict--lời-hứa-và-cái-giá) | Câu 5 | `cacerts`, hostname, phạm vi ba loại agent; setting không cài agent và không chứng minh kết nối |
| [6. Split DNS](../cloudflare-rancher-docs/setup-rancher.md#6-split-dns--cùng-một-tên-hai-câu-trả-lời) | Câu 1, M06, Câu 5 | Đúng tên, đúng resolver, `fallthrough`/TTL; đường máy và giới hạn với downstream |
| [7. Hai đường vào một Ingress](../cloudflare-rancher-docs/setup-rancher.md#7-hai-đường-vào-cùng-một-ingress) | Câu 1, M06, Câu 5 | Access, danh tính TLS và tránh phụ thuộc Internet cho đường local |
| [8. template và install](../cloudflare-rancher-docs/setup-rancher.md#8-helm-template-và-helm-install-nhìn-thấy-hai-cluster-khác-nhau) | M01/M05, Câu 4 | Capabilities/`--kube-version`, giả lập với API thật, Ingress với Gateway API |
| [9. Bootstrap password](../cloudflare-rancher-docs/setup-rancher.md#9-bootstrap-password-không-nằm-trong-values) | Câu 4 | Rancher sinh Secret lúc runtime, phải chờ xuất hiện và đọc đúng thời điểm |
| [10. Traefik và annotation](../cloudflare-rancher-docs/setup-rancher.md#10-traefik-hợp-rancher-mà-không-cần-annotation-nginx) | M06, Câu 4 | IngressClass, annotation NGINX, WebSocket và `X-Forwarded-Proto`; client HTTPS và backend HTTP 80 |
| [11. Origin 443 với 80](../cloudflare-rancher-docs/setup-rancher.md#11-vì-sao-route-rancher-dùng-https443-còn-app-demo-dùng-http80) | M06 | Cấu hình TLS đã chọn và sự khác biệt với app demo; không tự đổi kiến trúc |
| [12. Origin parameters](../cloudflare-rancher-docs/setup-rancher.md#12-ba-origin-parameter-của-route-rancher) | M06 | Đích mạng, SNI, Host, verification và giới hạn suy từ lỗi; `caPool`/`matchSNItoHost` là kiến thức tham chiếu, không phải thay đổi lab |
| [13. Access với route](../cloudflare-rancher-docs/setup-rancher.md#13-access-application-và-published-application-route--hai-lớp-hai-câu-hỏi) | M06, Câu 4 | Ai được vào với request đi đâu; Access trước publish; ý nghĩa 302/401/403 |
| [14. cattle-cluster-agent](../cloudflare-rancher-docs/setup-rancher.md#14-cattle-cluster-agent--chỉ-dành-cho-downstream-cluster) | Câu 5 | Nơi chạy, tunnel outbound, khác Fleet/system agent và đường import chưa được §14 triển khai |
| [15. Giới hạn từng gate](../cloudflare-rancher-docs/setup-rancher.md#15-mỗi-gate-chứng-minh-được-gì--và-không-chứng-minh-được-gì) | M05–M06, Câu 4–6 | Không suy từ render, rollout, healthz, Access hoặc setting sang toàn bộ hệ thống |

## Khi nào hoàn tất toàn bộ tài liệu?

1. Hoàn tất M01–M06 theo tiêu chí từng bài; Câu 1/6 đã học được giữ làm nền, không bắt làm lại toàn bộ.
2. Đã trả lời Câu 4/6–6/6, nhận xét/chấm 0–3 và sửa những ý thiếu hoặc sai. Các phần của Câu 2/6–3/6
   đã được xử lý trong M05–M06, giữ riêng điểm cũ và lượt mới.
3. Đối chiếu hai bảng trên, bổ sung những mục chưa thực sự học: người học tự giải thích được chủ thể,
   quan hệ phụ thuộc, cấu hình, ý nghĩa lệnh/flag và bằng chứng của từng nhóm trong phạm vi file.
   Điểm tình huống 3/3 chưa thay thế việc đối chiếu này. Không coi việc mentor vừa giảng là bằng chứng
   người học đã hiểu; cũng không yêu cầu nhắc lại những phần đã có câu trả lời đạt.
4. Với lệnh minh họa, giải thích nơi chạy, mục đích, kết quả mong đợi và giới hạn theo nguồn hiện tại.
   Hoàn tất lý thuyết không đòi chạy lại triển khai, tạo sự cố hoặc restart để chứng minh hiểu bài.
5. Lưu đầy đủ trao đổi thực tế và checkpoint; chỉ đánh dấu đã học toàn bộ khi các khoảng trống trong
   bảng đã được giải quyết. Phân biệt mức hiểu tài liệu với trạng thái cluster thực tế.

## Tiến độ và checkpoint

[README](README.md) là nơi theo dõi tiến độ hiện tại. M01–M04 đã hoàn tất, được lưu tại
[M01.md](M01.md), [M02.md](M02.md), [M03.md](M03.md) và [M04.md](M04.md).
Bản lưu M03 còn thiếu một số lời giảng gốc; phần M04 trước phiên 17–18/10/2026 chỉ có checkpoint
người học cung cấp. Giữ các giới hạn lịch sử đã ghi, không dựng hội thoại.

| Mốc theo giáo án 6 bài | Trạng thái khi cập nhật giáo án |
| --- | --- |
| M01–M04 | Hoàn tất; giữ nguyên các bản lưu |
| M05 — Helm, cấp certificate và gate | Chưa bắt đầu; học cùng Câu 2/6 ngay từ đầu |
| Câu 2/6 | Giữ lượt/điểm cũ; chưa có lượt trả lời mới trong M05 |
| M06 — TLS, origin và khoanh vùng lỗi | Chưa bắt đầu; học cùng Câu 3/6 ngay từ đầu |
| Câu 3/6 | Chưa trả lời |
| Câu 4/6–6/6 sau M06 | Chưa học/chưa trả lời; chưa có điểm |
| Đối chiếu bao phủ toàn bộ setup-rancher.md | Chưa hoàn tất; phải bổ sung các mục chưa học sau khi giải quyết tình huống |

Khi tiến độ thay đổi, cập nhật README; không lấy bảng này để phủ nhận checkpoint mới hơn.
Đây là tiến độ học lý thuyết, không phải xác nhận trạng thái cluster thực tế.

Cuối mỗi bài hoặc khi dừng giữa tình huống, cung cấp checkpoint có thể sao chép:

```text
Chặng hiện tại: M01–M06 / áp dụng Câu 4–6 / đối chiếu toàn bộ tài liệu
Bài hoặc câu hiện tại:
Câu hỏi tình huống và ý đang giải quyết:
Phần nhỏ vừa học:
Điều người học đã tự giải thích đúng:
Điểm còn nhầm hoặc chưa có câu trả lời:
Diagram/đoạn vừa đối chiếu:
Ví dụ hoặc câu hỏi đang chờ:
Phần tiếp theo trong tình huống / bài tiếp theo:
Các câu đã trả lời và điểm lượt mới nếu có; phân biệt với điểm cũ
Các mục trong bảng đối chiếu: đã tự giải thích / còn thiếu / chưa học
```

Đọc README, checkpoint và hội thoại đã lưu để nối tiếp đúng ý đang học. Không coi một phần mới
được giảng hoặc dự định đưa ra là nội dung người học đã tự giải thích đúng.

### Lưu đầy đủ từng bài sau khi hoàn tất

Thư mục `rancher-mentor` nằm ở gốc repo, ngang cấp với `cloudflare-rancher-docs`.
Giáo án nằm tại `rancher-mentor/rancher-mentor-study-plan.md`. Mỗi bài hoàn tất có một file
riêng trong cùng thư mục: `M01.md`, `M02.md`, …, `M06.md`. Không tạo sẵn file như thể
bài đã học xong. Câu 4/6–6/6 của chặng áp dụng lưu riêng vào `C04.md`, `C05.md`, `C06.md`
trong cùng thư mục; mã C là câu tình huống, không phải mở thêm module M07–M12. Không tạo trước
các file Cxx khi chưa có trao đổi hoàn tất. Yêu cầu này là chỉ dẫn lưu bài xuyên suốt các lượt
học; không hỏi lại quyền ghi file cuối mỗi bài/câu.

Mỗi file bài Mxx hoặc câu Cxx phải bao gồm:

1. Tên bài, mục tiêu và phạm vi thực sự đã học.
2. Đầy đủ nội dung mentor đã giảng trong bài, gồm định nghĩa, ví dụ, Mermaid, code/lệnh và
   liên kết nguồn đã dùng. Nếu có lệnh, giữ nơi chạy, mục đích và kết quả mong đợi.
3. Toàn bộ câu hỏi và câu trả lời theo thứ tự trao đổi, ghi rõ **Mentor** / **Người học**.
   Giữ nguyên câu trả lời gốc của người học, kể cả điểm sai và các lượt trả lời lại; không thay
   bằng đáp án đã sửa hoặc rút gọn thành một bản tổng kết.
4. Toàn bộ nhận xét, sửa nhầm, giải thích bổ sung và câu trả lời lại thực sự đã diễn ra.
   Nếu mentor sửa một giải thích trước đó, giữ lịch sử và đánh dấu rõ nội dung đã đính chính.
5. Checkpoint cuối bài: đã hiểu gì, còn vướng gì, bài/phần tiếp theo. Chỉ ghi hoàn tất khi có
   căn cứ trong trao đổi; không tự bịa câu trả lời hoặc kết quả thực hành.

Khi lưu xong, thêm liên kết file và cập nhật trạng thái trong [README](README.md), sau đó đưa
liên kết GitHub cho người học. Nếu có trao đổi bổ sung về bài đã lưu, cập nhật vào chính file
bài đó, không bỏ lịch sử cũ. Câu 2/6 được lưu trong M05 và Câu 3/6 được lưu trong M06 khi
trao đổi thực sự diễn ra, gồm câu trả lời, điểm 0–3, nhận xét và lượt sửa. Không tạo trước
M05.md hoặc M06.md như thể bài đã hoàn tất; không sửa các bản lưu M01–M04 khi đổi giáo án.
Câu 4/6–6/6 lưu nguyên văn phần giảng, câu trả lời, nhận xét, điểm và lượt sửa trong file Cxx
tương ứng theo cùng quy tắc. Phần trao đổi bổ sung để khép khoảng trống bao phủ được ghi nối
vào file Mxx/Cxx liên quan và cập nhật checkpoint README; không suy từ đáp án mẫu thành lịch sử.
Đáp án tham khảo trong setup-rancher.md không tự biến thành lời người học; chỉ sửa tài liệu
giải thích đó hoặc ghi thêm kết quả phỏng vấn vào đó khi người học yêu cầu.

Khi tiếp tục ở chat mới, đọc README, checkpoint và các file bài cần dùng để nối tiếp đúng chỗ.
Nếu không còn đủ lịch sử để lưu nguyên nội dung, nói rõ phần thiếu; không tự dựng hội thoại.

## Prompt bắt đầu chat mới

Sao chép toàn bộ prompt sau sang chat mới. Đọc [README](README.md) và checkpoint hiện tại trước
khi tiếp tục; nếu đã có checkpoint mới hơn, thêm nó bên dưới prompt.

```text
Bạn hãy làm mentor dạy tôi theo giáo án trong repo GitHub hieupham259/ansible_playbook_k8s-installation, nhánh main.

Đọc rancher-mentor/README.md và checkpoint hiện tại; đọc các file bài đã lưu nếu cần để tiếp tục đúng chỗ.
Đọc các file hiện tại trước khi dạy:
1. Giáo án: https://github.com/hieupham259/ansible_playbook_k8s-installation/blob/main/rancher-mentor/rancher-mentor-study-plan.md
2. Tài liệu giải thích: https://github.com/hieupham259/ansible_playbook_k8s-installation/blob/main/cloudflare-rancher-docs/setup-rancher.md
3. Runbook: https://github.com/hieupham259/ansible_playbook_k8s-installation/blob/main/runbook-k8s-vmware.md
4. Tra cứu TLS khi đến bài tương ứng: https://github.com/hieupham259/ansible_playbook_k8s-installation/blob/main/cloudflare-rancher-docs/ca-tls-https-request-response.md

BỐI CẢNH VÀ MỤC TIÊU
Tôi đã trả lời câu 1/6. Tôi từng trả lời câu 2/6 nhưng còn mơ hồ về Helm, quá trình xin/cấp certificate, readiness và giới hạn của helm --wait. Câu 3/6 chưa trả lời. Tôi muốn học để tự giải thích được, không học thuộc đáp án.
M01–M06 là sáu bài của chặng nền; M01–M04 đã hoàn tất theo README hiện tại. Câu 2/6 học trong M05, Câu 3/6 trong M06 ngay từ đầu. Sau M06 học tiếp Câu 4/6–6/6 trực tiếp trong setup-rancher.md và dùng bảng đối chiếu của giáo án để bao phủ phần còn thiếu. Hoàn tất M06 chưa phải đã hiểu hết file. Các tình huống và case dùng để học, không phải trạng thái cluster hiện tại mà bạn đã kiểm tra.

CÂU 1/6–3/6 CỦA CHẶNG NỀN

Câu 1/6 — Hai đường truy cập cùng một Rancher (đã trả lời)
Giả sử §14 đã hoàn thành, có split DNS. Rancher dùng rancher.hieupn.site và Ingress do Traefik xử lý. Xét hai client độc lập:
- Browser trên laptop ngoài cluster dùng DNS public.
- Một chương trình Python trong Pod ứng dụng giả định rancher-api-client, chạy trong cụm local, dùng CoreDNS và gọi https://rancher.hieupn.site. Pod này khác cloudflared, Traefik và Rancher; không phải thành phần trung gian trên đường browser, không phải agent và không được runbook cài sẵn. Không xét downstream cluster.

Route Rancher dùng:
- Service URL: https://traefik.traefik.svc.cluster.local:443
- originServerName: rancher.hieupn.site
- httpHostHeader: rancher.hieupn.site
- noTLSVerify: true

Các ý cần giải thích:
1. Browser nhận IP của ai từ DNS public? HTTP request đi qua những thành phần nào tới Pod Rancher? Access kiểm tra người dùng ở đâu? Phân biệt DNS, chuyển tiếp request và cấu hình Ingress.
2. Python trong Pod giả định nhận IP của ai từ CoreDNS, rồi request đi đường nào? Có qua Edge, Access hoặc Tunnel không? Vì sao runbook chuẩn bị đường nội bộ này?
3. Browser, cloudflared và Python lần lượt nhận certificate do ai trình, dành cho hostname nào? TLS kết thúc ở đâu? Vì sao client dùng HTTPS nhưng Traefik → backend Rancher dùng HTTP?
4. Kiểm chứng split DNS bằng lookup cùng rancher.hieupn.site từ laptop ngoài cluster và Pod dùng CoreDNS: nêu nơi chạy, lệnh và kết quả mong đợi. Lookup trên VM master có chứng minh DNS của Pod đúng không? Chỉ lookup tên Service Traefik trong Pod đã đủ kiểm chứng split DNS của hostname Rancher chưa?

Câu 2/6 — Helm đã xong thì certificate đã sẵn sàng chưa? (học song song trong M05)
Giả sử cài Rancher ở §14.3, mode Ingress, ingress.tls.source=rancher và ingress.tls.secretName=tls-rancher-ingress.
- helm install ... --wait báo thành công.
- Deployment Rancher đã Ready.
- Issuer và Ingress đã tồn tại trong cattle-system.
- Nhưng chưa thấy Certificate và Secret tls-rancher-ingress.

Cần trả lời:
1. Thành phần nào tạo Certificate, rồi thành phần nào cấp certificate và ghi Secret?
2. Vì sao Helm --wait thành công mà các tài nguyên đó vẫn có thể chưa sẵn sàng?
3. Cần kiểm tra hoặc chờ tài nguyên/điều kiện nào trước khi publish route Rancher ra Cloudflare?

Câu 3/6 — Tunnel khỏe nhưng browser vẫn nhận 502: khoanh vùng lỗi ở đâu? (học song song trong M06; trạng thái theo README)
Giả sử dùng đúng cấu hình lab:
- Certificate tls-rancher-ingress có Ready=True; Secret TLS đã tồn tại.
- Từ master, gate §14.4 dùng curl -k --resolve gọi https://rancher.hieupn.site/healthz qua ClusterIP Traefik trả HTTP 200.
- Tunnel homelab-k8s báo Healthy; cloudflared đang Connected.
- Laptop ngoài cluster, chưa đăng nhập Access: gọi URL Rancher nhận 302 tới trang đăng nhập Access.
- Browser đăng nhập Access thành công, sau đó mở Rancher thì nhận 502 Bad Gateway.

Cần trả lời:
1. Response 302 từ Access chứng minh những chặng nào hoạt động? Nó đã chứng minh request qua tunnel tới Traefik và Rancher chưa? Giải thích.
2. Ưu tiên kiểm tra chặng nào khi nhận 502 sau đăng nhập Access? Bằng chứng trên đã đủ kết luận Pod Rancher hỏng chưa? Vì sao?
3. Với Service URL https://traefik.traefik.svc.cluster.local:443, nêu giá trị theo runbook và vai trò riêng của TLS → Origin Server Name, TLS → No TLS Verify, HTTP Settings → HTTP Host Header.

CHẶNG ÁP DỤNG SAU M06
Đọc nguyên văn Câu 4/6–6/6 trong phần Tự kiểm tra của setup-rancher.md và nối tiếp đúng checkpoint:
- Câu 4/6 — Vì sao phải cài và kiểm tra theo thứ tự này? Kiểm tra trước cài, backup, thứ tự §14, render/values, bootstrap, inventory và giới hạn gate.
- Câu 5/6 — Browser dùng được thì client và cluster khác đã dùng được chưa? Server URL, client local, quyền/trust, các loại agent, strict và downstream.
- Câu 6/6 — Trang web và node đều khỏe, vì sao Rancher vẫn lỗi? Dashboard/API, SelfUser/Steve/extension API, timeline sau reboot và condition local.
Câu 4–6 chưa có lượt trả lời tại thời điểm bổ sung; dùng README hiện tại nếu có tiến độ mới hơn.
Sau các tình huống, đối chiếu toàn bộ phần chính, các bước §14 và 15 cơ chế trong bảng của giáo án. Bổ sung phần chưa tự giải thích được trước khi xác nhận hoàn tất toàn bộ tài liệu; không bắt trả lời lại phần đã đạt.

CÁCH DẠY
- Đọc README và checkpoint để nối tiếp đúng bài/câu. M01–M04 đã hoàn tất; M05 học cùng Câu 2/6 ngay từ đầu, M06 học cùng Câu 3/6 ngay từ đầu; sau đó tiếp tục Câu 4/6–6/6 và đối chiếu bao phủ. Mỗi lượt xử lý một phần có mục tiêu của tình huống; không giảng toàn bộ bài trong một câu trả lời.
- Dùng tiếng Việt rất đơn giản, định nghĩa thuật ngữ trước, có ví dụ cụ thể gắn với lab và diagram Mermaid khi giúp hiểu. Diagram phải tách rõ chương trình chạy, object cấu hình và dữ liệu; không gom toàn bộ công việc vào một cột.
- Dùng câu hỏi có tình huống, dữ kiện và mục tiêu suy luận rõ; cho tôi trả lời từng ý hoặc hỏi chỗ chưa hiểu. Bổ sung kiến thức ngay trong tình huống rồi quay lại lập luận. Tránh hỏi vụn hoặc lặp lại tên object/trường đã hiểu. Khi tôi hỏi lại hoặc review tài liệu, tập trung giải quyết chỗ đó, không chen câu kiểm tra.
- Sau mỗi câu trả lời được đánh giá đúng, hoặc khi kết thúc phần/bài đã hiểu, chỉ rõ tôi đang ở đâu trong diagram. Mỗi khi nhắc diagram nào, phải vẽ ngay diagram đó trong câu trả lời bằng Mermaid, nêu tên phần/diagram và chỉ nhãn node hoặc mũi tên liên quan; không gửi link thay cho hình và không bắt tôi mở trang khác. Nếu vẽ trích đoạn/rút gọn hoặc đổi vai để áp dụng vào lab, ghi rõ. Ưu tiên hai phần Nền tảng và Sơ đồ tuần tự của setup-rancher.md; khi học TLS/request thì dùng đúng diagram tương ứng. Nếu chi tiết không có trong sơ đồ gốc, nói rõ và phân biệt minh họa mới. Giữ tách chương trình, object, dữ liệu, người tạo chữ ký và người gửi bản tin. Ghi cả Mermaid và các lần đối chiếu thực sự đã trao đổi vào file bài/checkpoint; không coi vị trí học là trạng thái cluster thực tế.
- Giữ đúng cấu hình lab. Phân biệt Certificate object với certificate trong Secret, ingress-shim với controller cấp phát, cấp/gia hạn nền với TLS handshake và hai phiên TLS browser–Edge / cloudflared–Traefik.
- Giải thích --wait theo đúng phiên bản và phạm vi release; không khẳng định mọi công việc runtime chỉ bắt đầu sau khi Helm kết thúc.
- Nếu minh họa lệnh, nêu nơi chạy, mục đích và kết quả mong đợi; lấy đúng lệnh/gate trong runbook. Không yêu cầu cài lại, restart hoặc đổi cấu hình chỉ để học lý thuyết. Không giả định bạn đã thấy output thực tế hoặc truy cập được cluster.
- Nếu nhắc agent, phải ghi rõ loại agent và tình huống áp dụng.
- Câu 1/6 là nền đã học; chỉ ôn phần cần dùng. Câu 2/6 là trục của M05 và Câu 3/6 là trục của M06 ngay từ đầu mỗi bài. Không yêu cầu học xong các mục lý thuyết rồi mới được trả lời. Khi đã trả lời đủ các ý, chấm mỗi câu 0–3, giải thích và cho sửa; giữ nguyên lịch sử/điểm cũ.
- Cuối mỗi bài/câu cung cấp checkpoint ngắn, gồm các mục đã hiểu/còn thiếu trong bảng đối chiếu. Khi học xong bài Mxx hoặc Câu 4–6, tự động lưu file Mxx.md hoặc C04.md/C05.md/C06.md tương ứng trong rancher-mentor trên nhánh main, lưu đầy đủ phần giảng, ví dụ, diagram, câu hỏi, nguyên văn câu trả lời của tôi, nhận xét/sửa nhầm và checkpoint; cập nhật mục lục/tiến độ trong rancher-mentor/README.md. Không rút gọn hội thoại thành đáp án mẫu, không bịa nội dung chưa diễn ra và không cần hỏi lại quyền lưu bài. Chỉ sửa các tài liệu khác khi tôi yêu cầu.

Bắt đầu bằng xác nhận checkpoint thật ngắn. Nếu đang bắt đầu M05, đưa tình huống Câu 2/6; nếu bắt đầu M06, đưa tình huống Câu 3/6; nếu xong M06, nối tiếp Câu 4/6–6/6 hoặc mục còn thiếu trong bảng đối chiếu. Nếu đang dở một ý, nối tiếp đúng ý đó. Không đưa đáp án mẫu của cả câu trong lời mở đầu.
```

## Nguồn và giới hạn phạm vi

- [Runbook §14](../runbook-k8s-vmware.md#14-cài-rancher-2143--quản-lý-cụm): nguồn lệnh,
  thứ tự thao tác, phiên bản và gate.
- [Setup Rancher](../cloudflare-rancher-docs/setup-rancher.md): giải thích cơ chế, sơ đồ, câu 1/6–6/6 và đáp án tham khảo.
- [CA, TLS và HTTP](../cloudflare-rancher-docs/ca-tls-https-request-response.md): tra cứu khi học M03 và trong tình huống của M06.
- [Tunnel và Traefik](../cloudflare-rancher-docs/tunnel-traefik.md): tra cứu đường app demo; không bắt đọc lại toàn bài.
- [Helm install](https://helm.sh/docs/helm/helm_install/) và
  [Helm 3 install](https://helm.sh/docs/v3/helm/helm_install/): đối chiếu đúng phiên bản.
- [cert-manager Ingress](https://cert-manager.io/docs/usage/ingress/) và
  [Certificate](https://cert-manager.io/docs/usage/certificate/): đối chiếu ingress-shim và cấp phát.
- [Cloudflare origin parameters](https://developers.cloudflare.com/tunnel/reference/origin-parameters/):
  đối chiếu các origin parameters trong M06.

Giáo án không tự thay đổi runbook, cấu hình cluster, policy Access hoặc gate. Khi minh họa lệnh,
nêu nơi chạy, mục đích và output mong đợi; đối chiếu nguyên văn nguồn. Không suy đoán kết quả
thực tế. Nếu tài liệu đang thiếu hoặc mâu thuẫn, chỉ rõ đoạn và dùng nguồn chính thức để giải
thích; không âm thầm dựng một quy trình cài đặt khác. Không giao việc cho custom agent khi
người học chưa yêu cầu agent đó.
