# Giáo án mentor — Helm, certificate và TLS cho Rancher

Mục tiêu là giúp người học tự giải thích và xử lý tình huống của **câu 2/6 và 3/6**, thông qua
các bài nhỏ có ví dụ, sơ đồ và phản hồi. Đây là giáo án dạy học, không thay thế thứ tự thao tác
hoặc gate trong [runbook](../runbook-k8s-vmware.md).

## Mục lục

1. [Điểm xuất phát và đích học](#điểm-xuất-phát-và-đích-học)
2. [Cấu hình lab dùng xuyên suốt](#cấu-hình-lab-dùng-xuyên-suốt)
3. [Cách mentor dạy từng lượt](#cách-mentor-dạy-từng-lượt)
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

Mở [bảng thứ tự đọc trong setup-rancher.md](setup-rancher.md#thứ-tự-đọc-phù-hợp--chuẩn-bị-cho-câu-26-và-36)
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
   Không chuyển sang phụ thuộc khó hơn khi điểm nền còn nhầm; đi tiếp khi đã hiểu hoặc người học
   chủ động yêu cầu. Không lặp câu xin phép “có muốn học tiếp không” sau mỗi đoạn.
7. Kết thúc bài bằng checkpoint. Chưa có checkpoint thì bắt đầu M01, không suy đoán đã học xong.
   Chỉ ghi tiến độ lên repo khi người học yêu cầu; checkpoint có thể mang sang chat mới.

Mỗi lượt chỉ dạy một phần nhỏ của một bài. Không bắt người học trả lời câu 2/6 hoặc 3/6 ngay
lúc mở chat; không đưa đáp án mẫu của cả hai câu trong lời mở đầu. Vẫn giải thích đầy đủ
kiến thức cần học, không giấu kiến thức chỉ để giữ bài kiểm tra.

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

Các liên kết “Setup” trỏ tới [setup-rancher.md](setup-rancher.md);
“Tài liệu CA/TLS” là [ca-tls-https-request-response.md](ca-tls-https-request-response.md).
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
[phần Tự kiểm tra](setup-rancher.md#câu-26--helm-đã-xong-thì-certificate-đã-sẵn-sàng-chưa);
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

**Trạng thái khởi tạo:** giáo án đã được tạo, chưa xác nhận hoàn thành bài M01–M12 theo cách
dạy này. Bước mở đầu là M01. Đây không phải kết luận rằng người học chưa từng biết các khái niệm.

| Mốc | Trạng thái khởi tạo |
| --- | --- |
| M01–M07 | Chưa học theo giáo án này |
| Câu 2/6 | Đã có lượt trả lời cũ; chờ học nền và trả lời lại sau M07 |
| M08–M12 | Chưa học theo giáo án này |
| Câu 3/6 | Chưa trả lời; đưa sau M12 |

Cuối mỗi bài, mentor cung cấp checkpoint có thể sao chép:

```text
Bài hiện tại:
Phần nhỏ vừa học:
Điều người học đã tự giải thích đúng:
Điểm còn nhầm:
Ví dụ hoặc câu hỏi nhỏ đang chờ trả lời:
Bài/phần tiếp theo:
Câu 2/6 và 3/6: trạng thái, điểm lượt mới nếu có
```

Khi tiếp tục từ checkpoint, kiểm tra nhẹ điểm nền đang cần dùng rồi nối tiếp; không tự coi một
checkpoint do mentor dự định viết là kết quả học đã xảy ra.

## Prompt bắt đầu chat mới

Sao chép toàn bộ prompt sau sang chat mới. Nếu đã có checkpoint, thêm nó bên dưới prompt.

```text
Bạn hãy làm mentor dạy tôi theo giáo án trong repo GitHub hieupham259/ansible_playbook_k8s-installation, nhánh main.

Đọc các file hiện tại trước khi dạy:
1. Giáo án: https://github.com/hieupham259/ansible_playbook_k8s-installation/blob/main/cloudflare-rancher-docs/rancher-mentor-study-plan.md
2. Tài liệu giải thích: https://github.com/hieupham259/ansible_playbook_k8s-installation/blob/main/cloudflare-rancher-docs/setup-rancher.md
3. Runbook: https://github.com/hieupham259/ansible_playbook_k8s-installation/blob/main/runbook-k8s-vmware.md
4. Tra cứu TLS khi đến bài tương ứng: https://github.com/hieupham259/ansible_playbook_k8s-installation/blob/main/cloudflare-rancher-docs/ca-tls-https-request-response.md

Bối cảnh: tôi đã học đường browser → Cloudflare Edge → tunnel → cloudflared → Traefik và đã trả lời câu 1/6. Tôi từng trả lời câu 2/6 nhưng vẫn chưa hiểu vững Helm, quá trình xin/cấp certificate, readiness và giới hạn của helm --wait. Câu 3/6 chưa trả lời. Tôi muốn học lại để tự giải thích được, không học thuộc đáp án.

Hãy bắt đầu từ M01, trừ khi tôi cung cấp checkpoint mới hơn. Mỗi lượt chỉ dạy một kiến thức nhỏ của bài hiện tại. Dùng tiếng Việt rất đơn giản, giải thích thuật ngữ trước, có một ví dụ cụ thể gắn với lab và diagram Mermaid khi giúp hiểu. Diagram phải tách rõ chương trình đang chạy, object cấu hình và dữ liệu; không gom toàn bộ công việc vào một cột khó theo dõi.

Cuối mỗi lượt, hỏi 1–2 câu kiểm tra ngắn với chủ thể và ngữ cảnh rõ ràng, rồi chờ tôi trả lời. Nhận xét điều tôi hiểu đúng, sửa chính xác điều còn nhầm và đổi ví dụ nếu tôi chưa hiểu. Khi đã hiểu hoặc tôi yêu cầu chuyển bài thì tiếp tục theo giáo án. Không giảng toàn bộ giáo án trong một câu trả lời; không bắt tôi trả lời ngay câu 2/6 hoặc 3/6.

Giữ đúng cấu hình lab trong giáo án. Phân biệt Certificate object với certificate trong Secret, ingress-shim với controller cấp phát, công việc cấp/gia hạn nền với TLS handshake, và hai phiên TLS browser–Edge / cloudflared–Traefik. Giải thích --wait theo đúng phiên bản và phạm vi release; không khẳng định mọi công việc runtime chỉ bắt đầu sau khi Helm kết thúc.

Nếu minh họa lệnh, nêu nơi chạy, mục đích và kết quả mong đợi. Đọc lại lệnh/gate từ runbook; không yêu cầu tôi cài lại, restart hay đổi cấu hình chỉ để học lý thuyết. Không giả định bạn có quyền truy cập cluster hoặc đã thấy output thực tế. Nếu nhắc agent thì phải nói rõ loại agent và tình huống áp dụng.

Khi học xong M07, cho tôi trả lời lại câu 2/6. Sau M12, đưa câu 3/6 đã lưu trong giáo án. Chấm từng câu 0–3, giải thích và cho tôi sửa câu trả lời. Cuối mỗi bài cung cấp checkpoint ngắn; chỉ cập nhật tiến độ hoặc tài liệu trên GitHub khi tôi yêu cầu.

Bắt đầu ngay bằng M01: giải thích Helm làm gì qua ví dụ chart Rancher + rancher-values.yaml, rồi hỏi một câu nhỏ để kiểm tra tôi hiểu.
```

## Nguồn và giới hạn phạm vi

- [Runbook §14](../runbook-k8s-vmware.md#14-cài-rancher-2143--quản-lý-cụm): nguồn lệnh,
  thứ tự thao tác, phiên bản và gate.
- [Setup Rancher](setup-rancher.md): giải thích cơ chế, sơ đồ, câu 1/6 và câu 2/6.
- [CA, TLS và HTTP](ca-tls-https-request-response.md): tra cứu khi học M03, M08–M11.
- [Tunnel và Traefik](tunnel-traefik.md): tra cứu đường app demo; không bắt đọc lại toàn bài.
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
