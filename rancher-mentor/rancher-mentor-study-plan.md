# Giáo án mentor — Helm, certificate và TLS cho Rancher

Mục tiêu là giúp người học tự giải thích và xử lý tình huống của **câu 2/6 và 3/6**, thông qua
các bài nhỏ có ví dụ, sơ đồ và phản hồi. Đây là giáo án dạy học, không thay thế thứ tự thao tác
hoặc gate trong [runbook](../runbook-k8s-vmware.md).

## Mục lục

1. [Điểm xuất phát và đích học](#điểm-xuất-phát-và-đích-học)
2. [Cấu hình lab dùng xuyên suốt](#cấu-hình-lab-dùng-xuyên-suốt)
3. [Cách mentor dạy từng lượt](#cách-mentor-dạy-từng-lượt) — [đối chiếu diagram](#đối-chiếu-diagram-sau-mỗi-phần-đã-hiểu)
4. [Bản đồ 12 bài học](#bản-đồ-12-bài-học)
5. [Nội dung và mức độ cần đạt từng bài](#nội-dung-và-mức-độ-cần-đạt-từng-bài)
6. [Hai câu hỏi đích](#hai-câu-hỏi-đích)
7. [Tiến độ và checkpoint](#tiến-độ-và-checkpoint)
8. [Prompt bắt đầu chat mới](#prompt-bắt-đầu-chat-mới)
9. [Nguồn và giới hạn phạm vi](#nguồn-và-giới-hạn-phạm-vi)

## Điểm xuất phát và đích học

- Người học đã học hai đường truy cập Rancher và đã làm câu 1/6.
- Đã từng trả lời câu 2/6, nhưng tự nhận chưa hiểu vững; cần học lại phần chuẩn bị certificate
  trước khi trả lời lại. Không coi đáp án đã chấm là bằng chứng mọi kiến thức nền đều vững.
- Câu 3/6 chưa trả lời. Tình huống và câu hỏi được lưu nguyên ý trong giáo án này để chat mới
  không phải đoán nội dung từ lịch sử.
- Chặng A, M01–M07: Helm → tài nguyên Kubernetes → cấp certificate → readiness → gate.
- Chặng B, M08–M12: một phiên TLS → hai phiên TLS của lab → origin parameters → khoanh vùng lỗi.
- Không đòi người học đọc hết tài liệu hoặc thực hành cài lại trước khi được giải thích.

Mở [bảng thứ tự đọc trong setup-rancher.md](../cloudflare-rancher-docs/setup-rancher.md#thứ-tự-đọc-phù-hợp--chuẩn-bị-cho-câu-26-và-36)
để đối chiếu tài liệu. Mentor đọc phần cần dùng và hướng dẫn cụ thể, không chỉ giao danh sách
thuật ngữ để người học tự tìm.

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

1. Nêu bài hiện tại, một mục tiêu nhỏ và vì sao nó cần cho lab.
2. Giải thích bằng câu ngắn và từ quen thuộc; định nghĩa thuật ngữ mới ngay khi dùng.
   Có thể dùng ví dụ “bộ mẫu + lựa chọn → bản cấu hình”, “phiếu yêu cầu → kết quả”, nhưng
   luôn chỉ rõ phần nào là ví dụ và tên thật trong Kubernetes.
3. Đưa một ví dụ cụ thể. Khi sơ đồ giúp hiểu, dùng Mermaid gọn, nhãn ngắn, tối đa năm
   thành phần nằm ngang; phân biệt chương trình đang chạy với object hoặc dữ liệu.
   Không gom Helm, Kubernetes, cert-manager, CA và Secret vào một cột rồi bỏ qua vai trò.
4. Nối một hoặc hai chi tiết với đúng tiêu đề/đoạn trong tài liệu, không đổ cả chương cho người học.
5. Hỏi 1–2 câu ngắn, ghi rõ client nào, namespace nào, trạng thái ban đầu và điều cần giải thích.
   Dừng để người học trả lời; đây là kiểm tra hiểu nhỏ, chưa phải bài phỏng vấn đích.
6. Nhận xét: hiểu đúng gì → nhầm gì → giải thích lại bằng ví dụ khác → cho diễn đạt lại.
   Sau mỗi câu trả lời đúng, chỉ vị trí tương ứng trong diagram theo hướng dẫn bên dưới.
   Không chuyển sang phụ thuộc khó hơn khi điểm nền còn nhầm; đi tiếp khi đã hiểu hoặc người học
   chủ động yêu cầu. Không lặp câu xin phép “có muốn học tiếp không” sau mỗi đoạn.
7. Kết thúc bài bằng checkpoint. Chưa có checkpoint thì bắt đầu M01, không suy đoán đã học xong.
   Theo yêu cầu lưu bài của người học, khi hoàn tất bài nào thì tự động tạo file `Mxx.md` của bài đó
   trong thư mục `rancher-mentor`, lưu đầy đủ phần giảng và toàn bộ hỏi–đáp, rồi cập nhật mục lục/
   tiến độ trong [README](README.md). Không cần xin lại xác nhận cho việc lưu bài đã hoàn tất;
   các chỉnh sửa tài liệu khác vẫn chỉ thực hiện khi được yêu cầu.

Mỗi lượt chỉ dạy một phần nhỏ của một bài. Không bắt người học trả lời câu 2/6 hoặc 3/6 ngay
lúc mở chat; không đưa đáp án mẫu của cả hai câu trong lời mở đầu. Vẫn giải thích đầy đủ
kiến thức cần học, không giấu kiến thức chỉ để giữ bài kiểm tra.

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
| Helm --wait và công việc bất đồng bộ — M06 | [Sơ đồ 2 — Helm và cert-manager chạy song song](../cloudflare-rancher-docs/setup-rancher.md#sơ-đồ-2--helm-và-cert-manager-chạy-song-song) | Chỉ nhánh `Nhánh chờ của Helm` hoặc `Nhánh cấp certificate`, và ghi chú `Hai nhánh không được bảo đảm hoàn tất cùng lúc` theo ý vừa trả lời. |
| Ready, Secret và gate — M07 | [Sơ đồ 1](../cloudflare-rancher-docs/setup-rancher.md#sơ-đồ-1--ai-tạo-gì-trong-chuỗi-cấp-certificate); [Sơ đồ 3](../cloudflare-rancher-docs/setup-rancher.md#sơ-đồ-3--theo-dõi-từng-bước-qua-kubernetes-api); [Đọc gate §14.4](../cloudflare-rancher-docs/setup-rancher.md#đọc-gate-144-theo-từng-câu-xác-nhận) | Phân biệt `Certificate đạt Ready=True` với `Traefik nạp certificate và private key` và việc kiểm endpoint bằng gate. Đây không phải bằng chứng các trạng thái thực luôn cập nhật theo thứ tự tuyệt đối của sơ đồ. |

Cấu trúc phản hồi: nêu tên phần/diagram → vẽ khối Mermaid ngay trong câu trả lời → chỉ nhãn
node/mũi tên liên quan → nhận xét phần đã hiểu và phần còn nhầm. Không dùng mẫu chỉ có link.

Nếu diagram hiện có không vẽ chi tiết đang học, nêu rõ giới hạn và giải thích ngay nội dung
cần thiết. Có thể vẽ minh họa bổ sung nhưng phải ghi rõ đó là minh họa mới, không tự đặt tên
một node rồi khẳng định node đó có trong sơ đồ gốc. Với M08–M12, các sơ đồ cấp certificate này
chỉ dùng để liên hệ phần chuẩn bị/đọc Secret; khi học handshake, request hoặc khoanh vùng lỗi,
dùng đúng diagram TLS/đường truy cập ở phần tương ứng, nêu tên nguồn và vẽ lại ngay trong câu trả lời.

“Đã hiểu đoạn nào” là tiến độ học, không phải xác nhận cluster thực tế đã chạy tới bước ấy.
Phân biệt mũi tên quan hệ tạo/sử dụng tài nguyên ở Sơ đồ 1 với lời gọi qua API ở Sơ đồ 3;
không diễn giải chúng thành đường HTTP request của browser. Không coi webhook/cainjector
đã được vẽ thành node, hoặc private key đã được gửi cho client, chỉ từ nhãn tổng quát của diagram.

Khi lưu bài `Mxx.md`, giữ cả các lần đối chiếu diagram thực sự đã đưa trong hội thoại.
Checkpoint ghi thêm diagram/đoạn vừa đối chiếu và đoạn tiếp theo. Chỉ ghi phần được người học
trả lời đúng; nội dung chưa trả lời hoặc mới giảng chưa được đánh dấu đã hiểu.

## Bản đồ 12 bài học

| Bài | Kiến thức trọng tâm | Đọc / sơ đồ chính | Đích |
| --- | --- | --- | --- |
| [M01](#m01-helm-làm-gì) | Helm: chart, values, manifest, release | Setup → Nền tảng → Helm | Hiểu đầu vào và kết quả của lệnh cài |
| [M02](#m02-object-controller-và-crd) | Object, controller, CRD; các thành phần cert-manager | Setup → CRD và webhook; Nền tảng → cert-manager | Phân biệt bản khai với chương trình thực hiện |
| [M03](#m03-certificate-ca-và-key-ở-mức-cần-thiết) | Certificate, CA, public/private key, hostname | Setup → Certificate, CA và “ký” | Hiểu đang xin cấp thứ gì và dùng để làm gì |
| [M04](#m04-ingress-issuer-certificate-và-hai-secret) | Ingress, Issuer, Certificate, CertificateRequest, hai Secret | Setup → Nền tảng → cert-manager; Sơ đồ 1 | Biết ai khai nhu cầu, ai giữ CA, ai giữ kết quả |
| [M05](#m05-chuỗi-xin-và-cấp-certificate) | Chuỗi cấp phát đầu tiên, vai trò từng controller | Setup → Sơ đồ 1 và Sơ đồ 3 | Tự kể quá trình từ Ingress đến certificate được nạp |
| [M06](#m06-helm---wait-và-công-việc-bất-đồng-bộ) | Phạm vi `--wait`, hai nhánh chạy song song | Setup → Helm; Sơ đồ 2 | Giải thích vì sao Helm thành công chưa đủ |
| [M07](#m07-khi-nào-sẵn-sàng-và-đọc-gate-144) | Create, Ready, Secret, Traefik nạp, curl gate | Runbook §14.4; Setup → Đọc gate §14.4 | Sau bài này trả lời lại câu 2/6 |
| [M08](#m08-một-phiên-tls-ai-là-client-ai-là-server) | TLS client/server, trình cert, verify, terminate | Tài liệu CA/TLS; Setup → Terminate TLS và SNI | Biết ai kiểm tra certificate của ai |
| [M09](#m09-handshake-tls-13-vừa-đủ-để-đọc-diagram) | Hello, key_share, certificate, Finished | Tài liệu CA/TLS → TLS 1.3 | Đọc diagram theo đúng thời điểm; không học toán mật mã |
| [M10](#m10-hai-phiên-tls-và-hai-hệ-dns-trong-lab) | Browser–Edge và cloudflared–Traefik; tunnel, DNS | Setup → Đường browser; hai hệ certificate | Xác định endpoint và certificate của từng phiên |
| [M11](#m11-origin-url-origin-server-name-và-http-host) | Ba origin parameter, hostname và địa chỉ kết nối | Setup → mục đào sâu 11, 12 | Phân biệt đích mạng, SNI/tên TLS và HTTP Host |
| [M12](#m12-access-302-502-và-khoanh-vùng-theo-bằng-chứng) | Access, Healthy tunnel, `302`, `502`, giới hạn gate | Setup → mục đào sâu 13, 15; runbook §14.5 | Sau bài này trả lời câu 3/6 |

Các liên kết “Setup” trỏ tới [setup-rancher.md](../cloudflare-rancher-docs/setup-rancher.md);
“Tài liệu CA/TLS” là [ca-tls-https-request-response.md](../cloudflare-rancher-docs/ca-tls-https-request-response.md).
Không biến các bài thành hướng dẫn sửa cluster khi người học chỉ đang học lý thuyết.

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

### M05. Chuỗi xin và cấp certificate

- **Dạy:** Helm tạo Ingress/Issuer; Rancher chuẩn bị CA; ingress-shim tạo Certificate; controller
  cấp phát tạo key/yêu cầu ký, dùng CA theo Issuer, ghi Secret và cập nhật status; Traefik nạp dữ liệu.
- **Ví dụ:** Certificate đã xuất hiện nhưng CA/Issuer chưa dùng được thì bước cấp vẫn chưa xong.
- **Sơ đồ:** học từng nhánh của Sơ đồ 1, sau đó Sơ đồ 3 để thấy đọc/ghi object qua Kubernetes API.
  Không ép các controller vào một thứ tự lịch chạy tuyệt đối.
- **Đạt khi:** tự kể ai tạo gì đến lúc Traefik có cert; hiểu cấp/gia hạn không được kích hoạt cho mỗi request.

### M06. Helm --wait và công việc bất đồng bộ

- **Dạy:** hai nhánh có thể chạy cùng lúc; Helm thành công chỉ xác nhận phạm vi chờ của release,
  không chứng minh mọi tài nguyên runtime hoặc chức năng Rancher đã sẵn sàng.
- **Ví dụ:** Deployment Rancher Ready, nhưng Certificate do ingress-shim tạo vẫn đang xử lý.
- **Sơ đồ:** “Sơ đồ 2 — Helm và cert-manager chạy song song”; cho người học thử đổi thứ tự hoàn tất.
- **Chính xác theo phiên bản:** chi tiết readiness/`--wait` khác giữa Helm 3 và Helm 4/strategy.
  Không khẳng định mọi custom resource đều bị bỏ qua trong mọi phiên bản. Trong lab, Certificate
  do ingress-shim tạo ngoài manifest chart Ingress nên phải có gate riêng.
- **Đạt khi:** giải thích được “Helm xong” khác “certificate xong” mà không nói cert-manager
  chỉ bắt đầu sau lúc lệnh Helm kết thúc.

### M07. Khi nào sẵn sàng và đọc gate §14.4

- **Dạy:** object xuất hiện → Certificate Ready → Secret kết quả → Traefik nạp → endpoint trả lời.
  Đây là các mốc cần phân biệt, không phải một chuỗi đồng bộ mà mọi status luôn đổi đúng thứ tự.
- **Ví dụ:** `wait --for=create` khác `wait --for=condition=Ready`; Secret tồn tại chưa tự chứng
  minh Traefik đã trình đúng cert; `curl -k` bỏ verification; `--resolve` giữ hostname nhưng ép IP.
- **Thực hành đọc:** lấy nguyên văn gate §14.4 hiện tại, giải thích một lệnh mỗi lần, nơi chạy và
  kết quả mong đợi. Không mặc định yêu cầu thực thi.
- **Đạt khi:** nói được mỗi bằng chứng xác nhận điều gì và chưa xác nhận điều gì.
- **Sau bài:** đưa lại câu 2/6, chờ trả lời rồi chấm 0–3 và cho sửa.

### M08. Một phiên TLS: ai là client, ai là server?

- **Dạy:** client chủ động kết nối; server trình certificate; client quyết định verification
  theo cấu hình/trust; terminate là nơi xử lý và giải mã TLS.
- **Ví dụ:** chỉ xét `cloudflared` gọi Traefik trước khi đưa thêm browser vào.
- **Liên hệ:** cert-manager cấp cert trước; nó không ngồi giữa request TLS. Không kéo thêm Rancher agent.
- **Đạt khi:** chỉ được bên trình cert, bên kiểm cert và điểm kết thúc phiên.

### M09. Handshake TLS 1.3 vừa đủ để đọc diagram

- **Dạy:** ClientHello/ServerHello có key_share; tính khóa handshake rồi mới gửi phần certificate
  được mã hóa; CertificateVerify/Finished phục vụ xác thực handshake.
- **Ví dụ:** đọc một handshake mới đầy đủ; chỉ sau đó mới nói việc tái sử dụng kết nối/session.
- **Giới hạn:** đây là ví dụ TLS 1.3; không suy ra mọi kết nối thực tế đều thương lượng TLS 1.3.
  Không biến bài thành giải toán mật mã hoặc so sánh sâu các cipher suite.
- **Đạt khi:** không nhầm trình tự “verify certificate xong mới trao đổi key_share” và phân biệt
  ký certificate lúc cấp với xác thực handshake lúc dùng.

### M10. Hai phiên TLS và hai hệ DNS trong lab

- **Dạy:** browser–Edge dùng cert public; connector–Traefik dùng cert origin; tunnel có transport
  mã hóa riêng. Public DNS và CoreDNS trả lời cho các tên/đường tương ứng.
- **Ví dụ:** browser resolve `rancher.hieupn.site`; connector resolve
  `traefik.traefik.svc.cluster.local`. Client API local giả định là một luồng khác có split DNS.
- **Sơ đồ:** tách hai handshake trước, rồi ghép; kết thúc ở Traefik nếu chưa cần backend.
- **Đạt khi:** không cho browser nhìn thấy certificate trong Secret origin và không kéo TLS xuyên
  từ browser qua Edge tới Traefik như một phiên duy nhất.

### M11. Origin URL, Origin Server Name và HTTP Host

- **Dạy:** URL origin chỉ đích/giao thức kết nối; Origin Server Name chọn tên TLS;
  HTTP Host Header chọn danh tính HTTP cho routing; No TLS Verify điều khiển verification origin.
- **Ví dụ:** kết nối Service DNS Traefik nhưng giữ danh tính website Rancher. So sánh app demo
  `http://…:80`: không có origin TLS, nên default `noTLSVerify: false` không làm phát sinh TLS.
- **Sơ đồ/bảng:** ba cột “đích mạng / tên TLS / HTTP Host”, gắn từng tham số vào đúng bước.
- **Đạt khi:** điền đúng ba setting của route Rancher và giải thích từng vai trò, không gộp SNI với Host.

### M12. Access, 302, 502 và khoanh vùng theo bằng chứng

- **Dạy:** Access cho người truy cập là lớp kiểm soát ở Edge; response từ Access có thể xuất hiện
  trước khi request đi vào tunnel. Healthy tunnel chưa kiểm chứng chặng connector–origin.
- **Ví dụ:** dùng đúng tình huống câu 3/6 phía dưới; kiểm tra từng chặng theo bằng chứng, không đoán
  Pod Rancher hỏng chỉ vì nhìn thấy `502`. Lỗi `502` có nhiều nguyên nhân.
- **Sơ đồ:** đánh dấu chặng đã có bằng chứng và chặng chưa được kiểm; lấy log/test phù hợp với nơi phát lỗi.
- **Đạt khi:** giải thích được `302`, Healthy tunnel và `curl -k` nội bộ giới hạn kết luận ra sao.
- **Sau bài:** đưa câu 3/6, chờ trả lời rồi chấm 0–3; nếu thiếu kiến thức thì quay đúng bài liên quan.

## Hai câu hỏi đích

Chỉ đưa các câu này cho người học ở mốc tương ứng. Câu 2 lấy từ
[phần Tự kiểm tra](../cloudflare-rancher-docs/setup-rancher.md#câu-26--helm-đã-xong-thì-certificate-đã-sẵn-sàng-chưa);
đáp án/nhận xét đã có ở đó để mentor đối chiếu sau khi người học trả lời.

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

## Tiến độ và checkpoint

Tiến độ hiện tại và danh sách file bài đã hoàn tất được theo dõi tại [README](README.md).
M01 và M02 đã hoàn tất, với đầy đủ phần giảng và hỏi–đáp được lưu tại
[M01.md](M01.md) và [M02.md](M02.md). Đang bắt đầu M03 từ certificate và cặp khóa;
checkpoint, câu kiểm tra đang chờ và các điểm cần củng cố nằm trong README.
Đây là tiến độ học lý thuyết, không phải xác nhận trạng thái cluster thực tế.

| Mốc | Trạng thái hiện tại |
| --- | --- |
| M01 | Hoàn tất; đã lưu [M01.md](M01.md) |
| M02 | Hoàn tất; đã lưu [M02.md](M02.md) |
| M03 | Đang học phần certificate và cặp khóa |
| M04–M07 | Chưa học theo giáo án này |
| Câu 2/6 | Đã có lượt trả lời cũ; chờ học nền và trả lời lại sau M07 |
| M08–M12 | Chưa học theo giáo án này |
| Câu 3/6 | Chưa trả lời; đưa sau M12 |

Cuối mỗi bài, mentor cung cấp checkpoint có thể sao chép:

```text
Bài hiện tại:
Phần nhỏ vừa học:
Điều người học đã tự giải thích đúng:
Điểm còn nhầm:
Diagram/đoạn vừa đối chiếu:
Đoạn diagram hoặc phần tiếp theo:
Ví dụ hoặc câu hỏi nhỏ đang chờ trả lời:
Bài/phần tiếp theo:
Câu 2/6 và 3/6: trạng thái, điểm lượt mới nếu có
```

Khi tiếp tục từ checkpoint, kiểm tra nhẹ điểm nền đang cần dùng rồi nối tiếp; không tự coi một
checkpoint do mentor dự định viết là kết quả học đã xảy ra.

### Lưu đầy đủ từng bài sau khi hoàn tất

Thư mục `rancher-mentor` nằm ở gốc repo, ngang cấp với `cloudflare-rancher-docs`.
Giáo án nằm tại `rancher-mentor/rancher-mentor-study-plan.md`. Mỗi bài hoàn tất có một file
riêng trong cùng thư mục: `M01.md`, `M02.md`, …, `M12.md`. Không tạo sẵn file như thể
bài đã học xong. Yêu cầu này là chỉ dẫn lưu bài xuyên suốt các lượt học; không hỏi lại quyền
ghi file cuối mỗi bài.

Mỗi file bài phải bao gồm:

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
bài đó, không bỏ lịch sử cũ. Câu 2/6 sau M07 và câu 3/6 sau M12 được bổ sung vào file bài tương
ứng khi trao đổi thực sự diễn ra, gồm câu trả lời, điểm 0–3, nhận xét và lượt sửa.

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
Câu 1/6–3/6 là các câu phỏng vấn ban đầu; M01–M12 là các bài học của giáo án. Ba tình huống dưới đây độc lập, không phải mô tả trạng thái cluster thực tế mà bạn đã kiểm tra.

NỘI DUNG CỤ THỂ CỦA BA CÂU

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

Câu 2/6 — Helm đã xong thì certificate đã sẵn sàng chưa? (cần học nền rồi trả lời lại)
Giả sử cài Rancher ở §14.3, mode Ingress, ingress.tls.source=rancher và ingress.tls.secretName=tls-rancher-ingress.
- helm install ... --wait báo thành công.
- Deployment Rancher đã Ready.
- Issuer và Ingress đã tồn tại trong cattle-system.
- Nhưng chưa thấy Certificate và Secret tls-rancher-ingress.

Cần trả lời:
1. Thành phần nào tạo Certificate, rồi thành phần nào cấp certificate và ghi Secret?
2. Vì sao Helm --wait thành công mà các tài nguyên đó vẫn có thể chưa sẵn sàng?
3. Cần kiểm tra hoặc chờ tài nguyên/điều kiện nào trước khi publish route Rancher ra Cloudflare?

Câu 3/6 — Tunnel khỏe nhưng browser vẫn nhận 502: khoanh vùng lỗi ở đâu? (chưa trả lời)
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

CÁCH DẠY
- Bắt đầu từ M01, trừ khi tôi cung cấp checkpoint mới hơn. Mỗi lượt chỉ dạy một kiến thức nhỏ của bài hiện tại; không giảng toàn bộ giáo án trong một câu trả lời.
- Dùng tiếng Việt rất đơn giản, định nghĩa thuật ngữ trước, có ví dụ cụ thể gắn với lab và diagram Mermaid khi giúp hiểu. Diagram phải tách rõ chương trình chạy, object cấu hình và dữ liệu; không gom toàn bộ công việc vào một cột.
- Cuối mỗi lượt hỏi 1–2 câu kiểm tra ngắn, ghi rõ chủ thể và ngữ cảnh, rồi chờ tôi trả lời. Nhận xét hiểu đúng gì, sửa điểm nhầm và đổi ví dụ nếu cần. Khi tôi đã hiểu hoặc yêu cầu chuyển bài thì tiếp tục.
- Sau mỗi câu trả lời được đánh giá đúng, hoặc khi kết thúc phần/bài đã hiểu, chỉ rõ tôi đang ở đâu trong diagram. Mỗi khi nhắc diagram nào, phải vẽ ngay diagram đó trong câu trả lời bằng Mermaid, nêu tên phần/diagram và chỉ nhãn node hoặc mũi tên liên quan; không gửi link thay cho hình và không bắt tôi mở trang khác. Nếu vẽ trích đoạn/rút gọn hoặc đổi vai để áp dụng vào lab, ghi rõ. Ưu tiên hai phần Nền tảng và Sơ đồ tuần tự của setup-rancher.md; khi học TLS/request thì dùng đúng diagram tương ứng. Nếu chi tiết không có trong sơ đồ gốc, nói rõ và phân biệt minh họa mới. Giữ tách chương trình, object, dữ liệu, người tạo chữ ký và người gửi bản tin. Ghi cả Mermaid và các lần đối chiếu thực sự đã trao đổi vào file bài/checkpoint; không coi vị trí học là trạng thái cluster thực tế.
- Giữ đúng cấu hình lab. Phân biệt Certificate object với certificate trong Secret, ingress-shim với controller cấp phát, cấp/gia hạn nền với TLS handshake và hai phiên TLS browser–Edge / cloudflared–Traefik.
- Giải thích --wait theo đúng phiên bản và phạm vi release; không khẳng định mọi công việc runtime chỉ bắt đầu sau khi Helm kết thúc.
- Nếu minh họa lệnh, nêu nơi chạy, mục đích và kết quả mong đợi; lấy đúng lệnh/gate trong runbook. Không yêu cầu cài lại, restart hoặc đổi cấu hình chỉ để học lý thuyết. Không giả định bạn đã thấy output thực tế hoặc truy cập được cluster.
- Nếu nhắc agent, phải ghi rõ loại agent và tình huống áp dụng.
- Câu 1/6 là nền đã học; chỉ ôn phần cần dùng, không bắt làm lại toàn bộ. Không bắt trả lời ngay câu 2/6 hoặc 3/6. Sau M07 cho tôi trả lời lại câu 2/6; sau M12 đưa câu 3/6. Chấm mỗi câu 0–3, giải thích và cho tôi sửa câu trả lời.
- Cuối mỗi bài cung cấp checkpoint ngắn. Khi học xong bài nào, tự động tạo file Mxx.md tương ứng trong rancher-mentor trên nhánh main, lưu đầy đủ phần giảng, ví dụ, diagram, câu hỏi, nguyên văn câu trả lời của tôi, nhận xét/sửa nhầm và checkpoint; cập nhật mục lục/tiến độ trong rancher-mentor/README.md. Không rút gọn hội thoại thành đáp án mẫu, không bịa nội dung chưa diễn ra và không cần hỏi lại quyền lưu bài. Chỉ sửa các tài liệu khác khi tôi yêu cầu.

Bắt đầu ngay bằng M01: giải thích Helm làm gì qua ví dụ chart Rancher + rancher-values.yaml, rồi hỏi một câu nhỏ để kiểm tra tôi hiểu.
```

## Nguồn và giới hạn phạm vi

- [Runbook §14](../runbook-k8s-vmware.md#14-cài-rancher-2143--quản-lý-cụm): nguồn lệnh,
  thứ tự thao tác, phiên bản và gate.
- [Setup Rancher](../cloudflare-rancher-docs/setup-rancher.md): giải thích cơ chế, sơ đồ, câu 1/6 và câu 2/6.
- [CA, TLS và HTTP](../cloudflare-rancher-docs/ca-tls-https-request-response.md): tra cứu khi học M03, M08–M11.
- [Tunnel và Traefik](../cloudflare-rancher-docs/tunnel-traefik.md): tra cứu đường app demo; không bắt đọc lại toàn bài.
- [Helm install](https://helm.sh/docs/helm/helm_install/) và
  [Helm 3 install](https://helm.sh/docs/v3/helm/helm_install/): đối chiếu đúng phiên bản.
- [cert-manager Ingress](https://cert-manager.io/docs/usage/ingress/) và
  [Certificate](https://cert-manager.io/docs/usage/certificate/): đối chiếu ingress-shim và cấp phát.
- [Cloudflare origin parameters](https://developers.cloudflare.com/tunnel/reference/origin-parameters/):
  đối chiếu M11.

Giáo án không tự thay đổi runbook, cấu hình cluster, policy Access hoặc gate. Khi minh họa lệnh,
nêu nơi chạy, mục đích và output mong đợi; đối chiếu nguyên văn nguồn. Không suy đoán kết quả
thực tế. Nếu tài liệu đang thiếu hoặc mâu thuẫn, chỉ rõ đoạn và dùng nguồn chính thức để giải
thích; không âm thầm dựng một quy trình cài đặt khác. Không giao việc cho custom agent khi
người học chưa yêu cầu agent đó.
