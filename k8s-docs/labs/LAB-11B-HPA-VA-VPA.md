# Lab 11b — HPA và VPA

> **Điểm bắt đầu:** snapshot `04-metrics-ready` — mốc do [Lab 11a](LAB-11A-OBSERVABILITY.md) tạo, khác
> mốc trước ở đúng một thứ: metrics-server đang chạy và API `metrics.k8s.io` đã `Available`. Repo
> `~/bookstore` ở tag `lab-11a` (hoặc bundle `~/bookstore-lab-11a.bundle` nếu bạn vừa restore
> snapshot).
> **Điểm kết thúc:** teardown trả cluster về đúng `04-metrics-ready`, không tạo snapshot mới. Repo
> `~/bookstore` có tag `lab-11b` và một bản bundle nằm ngoài VM.
> **Sprint dự án:** Sprint 18 — Tự co giãn của [dự án Bookstore](DU-AN-BOOKSTORE.md). Đây là **lab
> trả nợ**: trả phần HPA của [nợ lab #1](README.md#5-sổ-nợ-lab).
> **Lab trước:** [Lab 11a — Observability](LAB-11A-OBSERVABILITY.md).
> **Lab sau:** [Lab 12 — Vận hành vòng đời node](LAB-12-VAN-HANH-VONG-DOI-NODE.md).
>
> **Cập nhật và đối chiếu phiên bản:** 13/10/2026.

Lab này nằm ở mục [Giai đoạn 11 — Observability](../00-ALO-TRINH-ADMIN.md#giai-đoạn-11--observability)
của lộ trình, ngay sau bài thực hành [342](../342-hpa-walkthrough-vi.md). Nó không mở nhóm bài mới mà
**trả nợ**: phần thực hành HPA của bài [72](../72-horizontal-pod-autoscale-vi.md) bị hoãn từ nhóm
[4b](../00-ALO-TRINH-ADMIN.md#4b-statefulset-daemonset-job-và-autoscaling) vì khi đó cluster chưa
có metrics-server. Ở
[Lab 4b, ticket B10](LAB-4B-STATEFULSET-DAEMONSET-VA-JOB.md#b10-bs-4b-10-thiết-kế-hpa-cho-web) bạn đã
thiết kế HPA cho `web` thành tài liệu `docs/hpa-web.draft.yaml`; sprint này đưa thiết kế đó vào
chạy thật trên hệ thống.

Mọi ticket dựa vào hợp đồng kỹ thuật ở [mục 4 của file dự án](DU-AN-BOOKSTORE.md#4-hợp-đồng-kỹ-thuật)
và bố cục repo ở [mục 5.5](DU-AN-BOOKSTORE.md#55-bố-cục-repo-theo-sprint). Kế hoạch gốc của sprint
nằm ở [mục 7.2](DU-AN-BOOKSTORE.md#sprint-18--lab-11b-tự-co-giãn).

**Lab trả nợ thì đọc lại bài gốc trước khi làm.** Trước B0, đọc lại phần *Phải hiểu ở lần đọc này*
và các mục được nhắc tên của bốn bài: [71](../71-autoscaling-vi.md),
[72](../72-horizontal-pod-autoscale-vi.md), [73](../73-vertical-pod-autoscale-vi.md) và
[342](../342-hpa-walkthrough-vi.md). Bài 72 là bài lab bám chặt nhất — nếu nó đã mờ thì đọc lại toàn
bộ, không chỉ phần đầu. B0 có một gate tự khai báo cho việc này.

Cluster giữ nguyên baseline của
[bảng A1.3 trong Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa). Image của dự án và
phiên bản metrics-server nằm ở [bảng A1.4](LAB-00-MOI-TRUONG-1.35.7.md#a14-phần-stack-không-thuộc-lab-00);
lab này không chép lại con số phiên bản nào trong phần văn bản, **không cài thêm gì** và **không sửa
gì** trong `kube-system`.

Trước khi bắt đầu, chạy [quy trình mở đầu A5.5](LAB-00-MOI-TRUONG-1.35.7.md#a55-quy-trình-mở-đầu-mỗi-lab)
trên `lab-k8s-master`. B0 bên dưới chạy lại đúng gate đó.

---

## 1. Kết quả phải đạt

Hoàn thành lab khi tầng `web` của Bookstore **tự co giãn theo tải** — yêu cầu `R16` của
[dự án](DU-AN-BOOKSTORE.md#22-yêu-cầu-của-hệ-thống-mới) — **và** bạn chứng minh, giải thích được:

- Cluster sau Lab 11a đủ điều kiện cho trục ngang tự động nhưng vẫn không có trục dọc tự động; hai
  kiểu "không dùng được" — HPA thiếu số đo ở Lab 4b, VPA thiếu cả kind — khác nhau ở đâu.
- `Utilization` của HPA là một phép chia, mẫu số là tổng `requests.cpu` của **mọi** container chạy
  suốt đời Pod `web`, kể cả sidecar — chứng minh bằng chính hai con số HPA ghi trong `status`.
- Vì sao `replicas` phải ra khỏi manifest của `web` khi HPA vào: `deploy.sh` ghi đè con số HPA đã
  chọn; và vì sao gỡ thẳng dòng đó rồi triển khai là chưa đủ.
- Hạn mức của namespace phải tính cho `web` ở `maxReplicas` cộng surge của rolling update, cộng Pod
  đang tắt và Pod sinh tải — không chỉ cho trạng thái thường.
- Sinh tải có trần, có deadline, đi đúng một đường mạng được mở riêng trong mạng zero-trust của
  sprint 7; HPA mở rộng nhanh, thu hẹp chậm — đo bằng số vòng, giải thích bằng cửa sổ ổn định.
- `behavior` biến tốc độ thu hẹp thành cấu hình của chính object; API server hợp nhất phần bạn khai
  với giá trị mặc định.
- HPA và người vận hành cùng ghi một trường: `kubectl scale` bị kéo về theo hai tốc độ khác nhau;
  đọc đúng ba condition `AbleToScale`, `ScalingActive`, `ScalingLimited`.
- Vì sao VPA **không** được cài ở sprint này, vì sao HPA và VPA cùng theo CPU trên một workload là
  xung đột, và phần nào của nợ #1 vẫn treo cùng điều kiện để trả.

### 1.1. Ánh xạ tài liệu sang bài thực hành

Nhóm bài của lab là bốn bài của [nợ lab #1](README.md#5-sổ-nợ-lab): ba bài khái niệm của nhóm 4b và
bài thực hành 342 của giai đoạn 11.

| Bài | Ticket kiểm chứng |
| --- | --- |
| [71 — Tự động co giãn Workload](../71-autoscaling-vi.md) | B1 — năng lực thật của cluster theo hai trục: ngang tự động có (API lõi cộng Metrics API), dọc tự động không có; B9 — trục dọc: bản thủ công đã làm ở Lab 3c (resize CPU của `api` tại chỗ), bản tự động là add-on chưa cài. Co giãn ngang **thủ công** đã làm ở Lab 4a, 4b; ở lab này nó là đối thủ của HPA (B8) |
| [72 — Tự động co giãn Pod theo chiều ngang](../72-horizontal-pod-autoscale-vi.md) | B1 — *Hỗ trợ các API metric*, đối tượng không co giãn được (DaemonSet không có subresource `scale`); B2 — đối tượng API `autoscaling/v2`, `Utilization` tính trên `requests`, `behavior` hợp nhất với mặc định; B3 — *Chuyển Deployment sang tự động co giãn*: bỏ `spec.replicas`, thrashing; B5 — *Tự động co giãn trong khi rolling update*; B6 — công thức, dung sai, cửa sổ ổn định, metric tài nguyên cộng gộp mọi container; B7 — chính sách co giãn, `selectPolicy`, ⭐ `tolerance`; B8 — giới hạn min/max, ⭐ *Ngừng kích hoạt ngầm theo kiểu chế độ bảo trì* |
| [73 — Tự động co giãn Pod theo chiều dọc](../73-vertical-pod-autoscale-vi.md) | B1 — VPA không có trong API; B9 — server từ chối `kind: VerticalPodAutoscaler`, xung đột mẫu số với HPA, `controlledResources` và `updateMode: Off` là hai lối thoát; ba thành phần và các `updateMode` là phần đọc hiểu, kiểm ở checkpoint |
| [342 — Hướng dẫn từng bước về HorizontalPodAutoscaler](../342-hpa-walkthrough-vi.md) | B1 — *Trước khi bạn bắt đầu*: Metrics Server và APIService; B2 — *Tạo HorizontalPodAutoscaler* bằng `kubectl autoscale` (chỉ server dry-run) so với *Tạo autoscaler theo cách khai báo*, condition lúc nghỉ; B4 — Pod `load-generator` thay bằng Job có trần; B6 — *Tăng tải*, *Dừng tạo tải*, *Đại lượng*; B6, B8 — phụ lục *Các status condition* |

Các phần dưới đây **không thực hành** trong lab này, có lý do:

| Phần | Vì sao không thực hành |
| --- | --- |
| Cài add-on VPA và chạy một `VerticalPodAutoscaler` thật (bài 73) | VPA không có trong [A1.4](LAB-00-MOI-TRUONG-1.35.7.md#a14-phần-stack-không-thuộc-lab-00); cài nó là thêm CRD, controller và mutating webhook vĩnh viễn trong khi lab này trả cluster về `04-metrics-ready`; và VPA là CRD cộng controller — mô hình của [giai đoạn 14](../00-ALO-TRINH-ADMIN.md#giai-đoạn-14--khả-năng-mở-rộng). B9 ghi quyết định và ba điều kiện để trả nốt |
| Recommender, Updater, admission controller, `resourcePolicy` của VPA (bài 73) | Không có object nào trên cluster để đo nên không viết được gate `PASS:`. Phần đọc hiểu nằm ở lời giải B9 và checkpoint |
| Metric tùy chỉnh, metric bên ngoài, nhiều metric (bài 72, 342) | Cần adapter `custom.metrics.k8s.io` hoặc `external.metrics.k8s.io` của bên thứ ba; metrics-server không cung cấp hai API đó. Thuộc [giai đoạn 23](../00-ALO-TRINH-ADMIN.md#giai-đoạn-23--giám-sát-và-cảnh-báo) |
| `type: ContainerResource` (bài 72) | HPA của dự án theo cả Pod; B6 có câu hỏi thiết kế khi nào nên chuyển sang theo riêng container `nginx` |
| Bốn cờ `--horizontal-pod-autoscaler-*` của kube-controller-manager (bài 72) | Đặt chúng là sửa manifest static Pod của control plane — [giai đoạn 20](../00-ALO-TRINH-ADMIN.md#giai-đoạn-20--cấu-hình-lại-cluster-đang-chạy). B1 chỉ **đọc**; gate cuối chứng minh bằng checksum rằng file không bị sửa |
| Image `registry.k8s.io/hpa-example`, `kubectl run` vòng lặp vô hạn, `minikube addons` (bài 342) | Image ngoài bộ đã khóa; vòng lặp không trần dễ làm nghẽn hai worker nhỏ; chuỗi lab không dùng minikube. Workload đích là chính `web` của dự án, nguồn tải là Job có trần ở B4 |
| Cluster Proportional Autoscaler, KEDA, co giãn theo lịch, co giãn node (bài 71) | Dự án ngoài lõi Kubernetes hoặc thuộc tầng hạ tầng; ba VM của lab là cố định |

Tình trạng nợ sau lab — nói được ở checkpoint:

| Nợ | Phần | Sau Lab 11b |
| --- | --- | --- |
| Nợ lab #1 | Thực hành HPA (bài 72, 342) | **Đã trả** ở B2–B8 |
| Nợ lab #1 | Thực hành VPA (bài 73) | **Chưa trả** — add-on ngoài baseline; điều kiện ở B9 |

Lab không phát sinh nợ lab mới, không trả và không phát sinh mã `TD` nào — xem
[sổ nợ của file dự án](DU-AN-BOOKSTORE.md#8-sổ-nợ-kỹ-thuật-dự-án). `TD-10` vẫn ở trạng thái chấp nhận
có điều kiện từ Lab 6b.

### 1.2. Thời lượng

4–5 giờ. Phần cốt lõi (B0–B9 trừ các mục ⭐, cộng B10) khoảng 4 giờ, trong đó gần một giờ là **chờ có
kiểm soát**: HPA thu hẹp sau tải (B5, B6, B7), deploy lặp lại (B3). Mọi đoạn chờ là vòng lặp có điều
kiện thoát và đếm số vòng — chu kỳ đồng bộ của HPA và cửa sổ ổn định **phụ thuộc cấu hình**, B1 đọc
chính cấu hình đó. Các mục ⭐ ở B3, B7, B8 là tùy chọn. Có thể chia hai phiên: phiên 1 làm B0–B5,
phiên 2 làm B6–B10; đầu phiên 2 chạy lại gate ngắn A5.5 và `./scripts/deploy.sh`.

---

## 2. Quy ước và an toàn

- Mọi lệnh `kubectl` chạy trên `lab-k8s-master` trừ khi ghi rõ node khác.
- Repo dự án: `~/bookstore`. Manifest thử nghiệm: `~/lab-work/11b/`. Bằng chứng:
  `~/lab-evidence/11b/`. B0 tạo file hàm dùng chung `~/lab-work/11b/ham.sh`; mọi script nghiệm thu
  nạp nó ở dòng đầu, nên mở shell mới giữa chừng không mất gì.
- Lab chỉ dùng kiến thức các nhóm đã học tới giai đoạn 11 và bốn bài của nợ #1. **Không** cài add-on,
  **không** tạo CRD, **không** sửa `/etc/kubernetes/manifests/*`, `/var/lib/kubelet/config.yaml` hay
  Deployment `metrics-server`. B0 ghi checksum và gate cuối đối chiếu.
- **Nguồn tải là diễn tập**: Pod sinh tải ghim `lab-k8s-worker2`, có trần CPU, có deadline. Không
  tăng tải quá trần đã gate ở B4 để "cho thấy rõ hơn".
- `kubectl scale`, `kubectl patch` lên object của dự án chỉ dùng trong diễn tập (B5, B8) và luôn được
  trả về bằng `./scripts/deploy.sh`. Repo là nguồn sự thật duy nhất.
- Object thử nghiệm ở `~/lab-work/11b/` (Pod dò mạng, Deployment thử) xóa khi xong ticket. Công cụ
  kiểm thử tải ở `tests/load/` của repo **không** nằm trong `k8s/` nên `deploy.sh` không bao giờ áp
  nó; bạn áp tay khi diễn tập và xóa ngay sau đó.
- **Dừng tải khẩn cấp** — chạy trên `lab-k8s-master` khi cluster ì đi hoặc bạn bỏ dở diễn tập:

  ```bash
  # 1. Cắt nguồn tải và đường mạng riêng của nó.
  kubectl -n bookstore delete -f ~/bookstore/tests/load/ --ignore-not-found --wait=false
  kubectl -n bookstore delete pod -l app.kubernetes.io/component=loadtest --ignore-not-found --wait=false
  # 2. Trả HPA, Deployment, quota về đúng repo (gỡ mọi patch của diễn tập).
  cd ~/bookstore && ./scripts/deploy.sh
  ```

  Không xóa HPA `web` và không `kubectl delete` gì trong `kube-system` để "giải phóng tài nguyên".
- Dòng bắt đầu bằng `PASS:` là gate; không đi tiếp khi gate fail. Dòng `FAIL:` trong script nghiệm
  thu nghĩa là ticket chưa xong.

---

# Phần B — Sprint 18: Tự co giãn

## B0. Nhận bàn giao sprint

**Mục đích:** xác nhận cluster ở đúng `04-metrics-ready`, repo ở đúng tag `lab-11a`, hệ thống của
sprint 17 dựng lại được **chỉ từ repo**, và bạn đã đọc lại bài gốc của món nợ sắp trả.

Gate ngắn A5.5, chạy nguyên văn:

```bash
kubectl wait --for=condition=Ready node --all --timeout=180s
test "$(kubectl get --raw='/readyz')" = 'ok' && echo 'PASS: readyz ok'
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded
kubectl -n kube-system get deployment coredns
kubectl get pods -n default
```

Mốc `04-metrics-ready` được định nghĩa bằng nguồn metric chạy được, và chưa có HPA nào:

```bash
MS="$(kubectl -n kube-system get deployment metrics-server -o jsonpath='{.status.readyReplicas}')"
[ "${MS:-0}" -ge 1 ] && echo 'PASS: metrics-server Ready'
[ "$(kubectl top node --no-headers 2>/dev/null | wc -l)" -eq 3 ] && echo 'PASS: kubectl top node tra du ba node'
[ "$(kubectl get hpa -A --no-headers 2>/dev/null | wc -l)" -eq 0 ] && echo 'PASS: cluster chua co HPA nao'
```

Lab trả nợ — gate tự khai báo. Đổi biến thành `roi` **sau khi** đã đọc lại bốn bài 71, 72, 73, 342:

```bash
DA_DOC_LAI_71_72_73_342='chua'
case "$DA_DOC_LAI_71_72_73_342" in
  roi) echo 'PASS: da doc lai bai goc cua no #1 truoc khi mo lab tra no' ;;
  *)   echo 'FAIL: doc lai bai 71, 72, 73, 342 roi dat bien = roi va chay lai' ;;
esac
```

Repo và thư mục làm việc. Nếu vừa restore snapshot, repo có thể đã lùi về quá khứ — khôi phục từ
bundle theo [mục 5.4 của file dự án](DU-AN-BOOKSTORE.md#54-sao-lưu-repo-ra-ngoài-vm) trước:

```bash
mkdir -p ~/lab-work/11b ~/lab-evidence/11b
test -d ~/bookstore/.git || git clone ~/bookstore-lab-11a.bundle ~/bookstore
cd ~/bookstore
git describe --tags --exact-match HEAD 2>/dev/null | grep -qx 'lab-11a' && echo 'PASS: HEAD dung tag lab-11a'
test -z "$(git status --porcelain)" && echo 'PASS: repo khong co thay doi chua commit'
kubectl get namespace bookstore >/dev/null 2>&1 \
  && echo 'FAIL: namespace bookstore con sot tu lan truoc' || echo 'PASS: cluster chua co du an'
test -f docs/hpa-web.draft.yaml && echo 'PASS: co ban thiet ke HPA web tu Lab 4b'
```

Dựng lại hệ thống của sprint 17 bằng script của chính sprint đó:

```bash
cd ~/bookstore
./scripts/deploy.sh > ~/lab-evidence/11b/b0-deploy.txt 2>&1; echo "deploy exit=$?"
./tests/smoke.sh > ~/lab-evidence/11b/b0-smoke-lab-11a.txt 2>&1; echo "smoke exit=$?"
cat ~/lab-evidence/11b/b0-smoke-lab-11a.txt
```

Kéo sẵn image mà sprint này dùng nhiều nhất lên hai worker, để thời gian kéo image không lẫn vào
thời gian phản ứng của HPA. Tag lấy đúng từ [A1.3](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa)
và [A1.4](LAB-00-MOI-TRUONG-1.35.7.md#a14-phần-stack-không-thuộc-lab-00):

```bash
for N in lab-k8s-worker1 lab-k8s-worker2; do
  for I in docker.io/library/busybox:1.37 docker.io/nginxinc/nginx-unprivileged:1.30.5-alpine; do
    ssh "$N" "sudo crictl pull $I" >/dev/null && echo "PASS: $N co $I" || echo "FAIL: $N khong keo duoc $I"
  done
done
```

Ảnh chụp "trước" của những thứ lab hứa **không đụng vào**, và file hàm dùng chung cho mọi script
nghiệm thu — chép nguyên văn:

```bash
{
  echo "kcm $(sudo sha256sum /etc/kubernetes/manifests/kube-controller-manager.yaml | awk '{print $1}')"
  echo "ms-image $(kubectl -n kube-system get deployment metrics-server -o jsonpath='{.spec.template.spec.containers[0].image}')"
  kubectl -n kube-system get deployment metrics-server \
    -o jsonpath='{range .spec.template.spec.containers[0].args[*]}ms-arg {@}{"\n"}{end}'
} > ~/lab-evidence/11b/b0-khong-dung-vao.txt
grep -Eq '^kcm [0-9a-f]{64}$' ~/lab-evidence/11b/b0-khong-dung-vao.txt \
  && grep -q '^ms-image ' ~/lab-evidence/11b/b0-khong-dung-vao.txt \
  && echo 'PASS: da ghi checksum kube-controller-manager va cau hinh metrics-server'

cat > ~/lab-work/11b/ham.sh <<'EOF'
# ~/lab-work/11b/ham.sh — hàm dùng chung cho script nghiệm thu Lab 11b. Nạp: . ~/lab-work/11b/ham.sh
NS=bookstore
EV=~/lab-evidence/11b
WK=~/lab-work/11b
POLL=15
SEL='app.kubernetes.io/part-of=bookstore'
jp() { kubectl -n "$NS" get "$1" -o jsonpath="$2" 2>/dev/null; }
cpu_m() { case "$1" in ''|'<none>') echo 0 ;; *m) echo "${1%m}" ;; *) echo "$(( $1 * 1000 ))" ;; esac; }
mem_mi() { case "$1" in ''|'<none>') echo 0 ;; *Ki) echo "$(( ${1%Ki} / 1024 ))" ;; *Mi) echo "${1%Mi}" ;;
  *Gi) echo "$(( ${1%Gi} * 1024 ))" ;; *M) echo "$(( ${1%M} * 1000000 / 1048576 ))" ;;
  *G) echo "$(( ${1%G} * 1000000000 / 1048576 ))" ;; *) echo "$(( $1 / 1048576 ))" ;; esac; }
hpa_cond() { jp hpa/web "{.status.conditions[?(@.type==\"$1\")].$2}"; }
web_rep() { jp deployment/web '{.spec.replicas}'; }
web_util() { jp hpa/web '{.status.currentMetrics[0].resource.current.averageUtilization}'; }
web_pod() { kubectl -n "$NS" get pod -l "$SEL,app.kubernetes.io/component=web" \
  --field-selector=status.phase=Running -o jsonpath='{.items[0].metadata.name}' 2>/dev/null; }
web_pod_req_m() {   # tổng requests.cpu của container chính và sidecar trong một Pod web đang chạy
  local S=0 V
  for V in $(jp "pod/$(web_pod)" \
      '{.spec.containers[*].resources.requests.cpu} {.spec.initContainers[?(@.restartPolicy=="Always")].resources.requests.cpu}'); do
    S=$(( S + $(cpu_m "$V") ))
  done
  echo "$S"
}
render_web() {   # Deployment web đúng như deploy.sh sẽ apply
  kubectl kustomize --load-restrictor=LoadRestrictionsNone ~/bookstore/k8s/app \
    | awk 'BEGIN{RS="\n---\n"} /\nkind: Deployment\n/ && /\n  name: web\n/ {print}'
}
EOF
. ~/lab-work/11b/ham.sh && [ "$(cpu_m 1500m)" -eq 1500 ] && [ "$(cpu_m 2)" -eq 2000 ] \
  && [ "$(mem_mi 1Gi)" -eq 1024 ] && echo 'PASS: ham dung chung nap duoc'
```

**PASS:** gate A5.5 đạt như mô tả ở Lab 00; ba dòng `PASS:` của mốc `04-metrics-ready`; dòng
`PASS: da doc lai bai goc...`; bốn dòng `PASS:` về repo và cluster; `deploy exit=0`, `smoke exit=0` và
file smoke không có dòng `FAIL:`; đủ bốn dòng `PASS: ... co ...`; hai dòng `PASS:` cuối.

`kubectl top node` lỗi nghĩa là cluster **không** ở `04-metrics-ready` — restore cả ba VM về đúng mốc
hoặc quay lại [Lab 11a](LAB-11A-OBSERVABILITY.md). `deploy.sh` hoặc `smoke.sh` fail nghĩa là repo của
sprint 17 **chưa đủ** — sửa repo ở tag `lab-11a` trước, không sửa tay trên cluster
([mục 6.1 của file dự án](DU-AN-BOOKSTORE.md#61-mở-đầu)).

## B1. [BS-11B-01] Điều tra: cluster đã đủ điều kiện cho HPA chưa

**Loại:** Điều tra.

**Bối cảnh:** ADR về HPA của Lab 4b kết luận "chưa chạy được vì thiếu Metrics API". Sprint 17 vừa cài
metrics-server. Trước khi đưa HPA vào hệ thống thật, đội phải kiểm lại **từng điều kiện** mà ADR đó
nêu — trên chính cluster này, không dựa vào việc "Lab 11a đã PASS".

**Yêu cầu:**

1. Chứng minh Metrics API đã đăng ký và trả số đo **cho chính các Pod `web`**, kèm cửa sổ đo của mỗi
   mẫu.
2. Chứng minh HorizontalPodAutoscaler vẫn là API lõi với phiên bản ổn định được phục vụ, còn
   VerticalPodAutoscaler vẫn không có trong API.
3. Chứng minh lại ranh giới "không áp dụng cho đối tượng không co giãn được" trên `node-agent` — từ
   sprint 17 nằm ở namespace `bookstore-system` — so với `web`.
4. Chứng minh mọi container chạy suốt đời Pod `web`, gồm container chính và sidecar, có `requests.cpu`.
5. Đọc, không sửa, cấu hình HPA hiệu lực của kube-controller-manager: chu kỳ đồng bộ, cửa sổ ổn định
   thu hẹp, dung sai đang là mặc định hay đã bị đặt tường minh.
6. Tính tay: với số đo hiện tại và mục tiêu trong `docs/hpa-web.draft.yaml`, HPA sẽ đề xuất bao nhiêu
   bản sao cho `web`. B2 đối chiếu con số này với con số controller tự tính.

**Ràng buộc:** chỉ đọc. Không tạo HPA, không sửa manifest control plane hay metrics-server.

**Đọc lại:**

- Bài [71](../71-autoscaling-vi.md):
  [*Co giãn workload theo chiều ngang*](../71-autoscaling-vi.md#scaling-workloads-horizontally),
  [*Co giãn workload theo chiều dọc*](../71-autoscaling-vi.md#scaling-workloads-vertically).
- Bài [72](../72-horizontal-pod-autoscale-vi.md):
  [*HorizontalPodAutoscaler hoạt động như thế nào?*](../72-horizontal-pod-autoscale-vi.md#horizontalpodautoscaler-hoạt-động-như-thế-nào-how-does-a-horizontalpodautoscaler-work),
  [*Chi tiết thuật toán*](../72-horizontal-pod-autoscale-vi.md#algorithm-details),
  [*Hỗ trợ các API metric*](../72-horizontal-pod-autoscale-vi.md#support-for-metrics-apis),
  [*Hành vi mặc định*](../72-horizontal-pod-autoscale-vi.md#hành-vi-mặc-định-default-behavior).
- Bài [73](../73-vertical-pod-autoscale-vi.md): [*Đối tượng API*](../73-vertical-pod-autoscale-vi.md#đối-tượng-api-api-object).
- Bài [342](../342-hpa-walkthrough-vi.md):
  [*Trước khi bạn bắt đầu*](../342-hpa-walkthrough-vi.md#trước-khi-bạn-bắt-đầu-before-you-begin).

**Bàn giao:** `~/lab-evidence/11b/b1-*.txt`; câu trả lời ghi vào mục *Bối cảnh* của ADR mà B2 tạo
(`docs/adr/NNNN-hpa-web-production.md`).

**Nghiệm thu:**

- `PASS:` APIService của Metrics API `Available`; Metrics API trả số đo của một Pod `web` kèm `window`.
- `PASS:` HPA là API lõi, `autoscaling/v2` được phục vụ; VPA không có trong API.
- `PASS:` Deployment `web` có subresource `scale`, DaemonSet `node-agent` thì không.
- `PASS:` `nginx` và sidecar `status-writer` đều có `requests.cpu`.
- `PASS:` đọc được cấu hình HPA của kube-controller-manager; bằng chứng ghi rõ cờ nào được đặt.
- `PASS:` tính tay được đề xuất của HPA từ số đo thật.

**Câu hỏi thiết kế** — ghi vào mục *Bối cảnh* của ADR ở B2:

1. ADR của Lab 4b nói HPA "chưa chạy được". Thứ gì trên cluster đã đổi, thứ gì trong API **không
   hề** đổi? Nếu bây giờ bạn apply một `VerticalPodAutoscaler`, lỗi khác lỗi HPA-thiếu-số-đo ở đâu?
2. Theo số đo của bạn, `web` đang dùng bao nhiêu phần trăm `requests`? HPA với mục tiêu trong bản
   thiết kế đề xuất bao nhiêu bản sao, trước và sau khi kẹp vào khoảng 2–5? Nếu tỷ lệ
   `utilization / mục tiêu` nằm trong khoảng 0.9–1.1 thì HPA làm gì?
3. Chu kỳ đồng bộ và cửa sổ thu hẹp của HPA trên cluster của bạn là bao nhiêu? Trả lời bằng **chỗ**
   bạn đọc được, không bằng con số trong tài liệu.

<details>
<summary>Gợi ý</summary>

- `kubectl get apiservice`; `kubectl top pod --containers`; dạng thô:
  `kubectl get --raw /apis/metrics.k8s.io/v1beta1/namespaces/bookstore/pods/<tên Pod>`.
- `kubectl api-versions`; `kubectl api-resources --api-group=<nhóm> -o name`.
- Subresource `scale`: `kubectl get --raw /apis/apps/v1/namespaces/<ns>/<kind số nhiều>/<tên>/scale`.
- Cờ của controller nằm trong manifest static Pod ở `/etc/kubernetes/manifests/` (file chỉ root đọc
  được). Không có cờ nghĩa là controller dùng mặc định dựng sẵn — bảng ở mục *Hành vi mặc định* của
  bài 72.
- Công thức của bài 72: `ceil(currentReplicas × currentUtilization / targetUtilization)`. Utilization
  của một Pod là mức dùng **của cả Pod** chia tổng `requests` của nó.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong</summary>

```bash
. ~/lab-work/11b/ham.sh
cd ~/bookstore

# 1. Metrics API và số đo của chính Pod web
AV="$(kubectl get apiservice v1beta1.metrics.k8s.io -o jsonpath='{.status.conditions[?(@.type=="Available")].status}')"
P="$(web_pod)"
kubectl get --raw "/apis/metrics.k8s.io/v1beta1/namespaces/$NS/pods/$P" > "$EV/b1-raw-pod.json"
kubectl top pod -n "$NS" -l "$SEL,app.kubernetes.io/component=web" --containers | tee "$EV/b1-top-web-containers.txt"
echo "container trong PodMetrics: $(grep -o '"name":"[^"]*"' "$EV/b1-raw-pod.json" | tr '\n' ' ')"
{ [ "$AV" = 'True' ] && grep -q '"window"' "$EV/b1-raw-pod.json" && grep -q '"nginx"' "$EV/b1-raw-pod.json"; } \
  && echo "PASS: Metrics API Available va tra so do cua $P kem cua so do"

# 2. Hai trục: HPA là API lõi, VPA không có
{ kubectl api-versions | grep -qx 'autoscaling/v2' && \
  kubectl api-resources --api-group=autoscaling -o name | grep -qx 'horizontalpodautoscalers.autoscaling'; } \
  && echo 'PASS: HPA la API loi, autoscaling/v2 duoc phuc vu'
[ -z "$(kubectl api-resources --api-group=autoscaling.k8s.io -o name 2>/dev/null)" ] \
  && echo 'PASS: VerticalPodAutoscaler khong co trong API'

# 3. Subresource scale
{ kubectl get --raw "/apis/apps/v1/namespaces/$NS/deployments/web/scale" > "$EV/b1-scale-web.json" && \
  ! kubectl get --raw /apis/apps/v1/namespaces/bookstore-system/daemonsets/node-agent/scale \
      > "$EV/b1-scale-agent.txt" 2>&1; } \
  && echo 'PASS: Deployment web co subresource scale, DaemonSet node-agent thi khong'

# 4. Mẫu số của Utilization
NG="$(jp deployment/web '{.spec.template.spec.containers[?(@.name=="nginx")].resources.requests.cpu}')"
SW="$(jp deployment/web '{.spec.template.spec.initContainers[?(@.name=="status-writer")].resources.requests.cpu}')"
REQ="$(web_pod_req_m)"
echo "requests.cpu: nginx=$NG status-writer=$SW tong moi Pod=${REQ}m" | tee "$EV/b1-requests.txt"
{ [ -n "$NG" ] && [ -n "$SW" ] && [ "$REQ" -gt 0 ]; } && echo 'PASS: nginx va sidecar status-writer deu co requests.cpu'

# 5. Cấu hình HPA của controller — chỉ đọc
sudo grep -n 'horizontal-pod-autoscaler' /etc/kubernetes/manifests/kube-controller-manager.yaml \
  > "$EV/b1-kcm-hpa.txt" 2>&1 || true
echo "so co --horizontal-pod-autoscaler-* dat tuong minh: $(grep -c 'horizontal-pod-autoscaler' "$EV/b1-kcm-hpa.txt")"
cat "$EV/b1-kcm-hpa.txt"
sudo test -s /etc/kubernetes/manifests/kube-controller-manager.yaml \
  && echo 'PASS: doc duoc cau hinh HPA hieu luc cua kube-controller-manager'

# 6. Tính tay đề xuất của HPA
TGT="$(kubectl create --dry-run=server -f docs/hpa-web.draft.yaml \
  -o jsonpath='{.spec.metrics[0].resource.target.averageUtilization}')"
kubectl top pod -n "$NS" -l "$SEL,app.kubernetes.io/component=web" --no-headers > "$EV/b1-top-web.txt"
SUM=0; N=0
while read -r _ C _; do SUM=$(( SUM + $(cpu_m "$C") )); N=$(( N + 1 )); done < "$EV/b1-top-web.txt"
if [ "$N" -ge 1 ] && [ "$REQ" -gt 0 ] && [ -n "$TGT" ]; then
  UTIL=$(( SUM * 100 / (N * REQ) ))
  RAW=$(( (N * UTIL + TGT - 1) / TGT ))
  DES="$RAW"; [ "$DES" -lt 2 ] && DES=2; [ "$DES" -gt 5 ] && DES=5
  echo "web: $N Pod, tong ${SUM}m, ~${UTIL}% requests / muc tieu ${TGT}% -> cong thuc ${RAW}, sau khi kep [2,5] ${DES}" \
    | tee "$EV/b1-tinh-tay.txt"
  echo 'PASS: tinh tay duoc de xuat cua HPA tu so do that'
fi
```

</details>

<details>
<summary>Lời giải tham khảo — ý chính cho mục Bối cảnh của ADR</summary>

1. Thứ đổi: APIService `v1beta1.metrics.k8s.io` đã đăng ký và `Available`, tức nguồn số đo mà bài 72
   mục *Hỗ trợ các API metric* đòi quản trị viên phải bảo đảm. Thứ **không** đổi: HPA vẫn là API lõi
   `autoscaling/v2` — ở Lab 4b object HPA đã tạo được (server dry-run chấp nhận), chỉ là không có số
   đo để tính. VPA thì khác loại: kind không tồn tại, API server từ chối ngay ở bước ánh xạ kind (bài
   73, *Đối tượng API*: VPA là CRD phải cài riêng).
2. Ví dụ với `requests` gợi ý của [mục 3.6](DU-AN-BOOKSTORE.md#36-ngân-sách-tài-nguyên-và-lập-lịch):
   mỗi Pod `web` có `50m + 10m = 60m`; ba Pod lúc nhàn dùng tổng `6m` → `~3%`. Với mục tiêu 70%,
   `ceil(3 × 3 / 70) = 1`, kẹp lên `minReplicas` thành 2. Tỷ lệ trong khoảng 0.9–1.1 là trong dung sai
   mặc định 0.1 — HPA giữ nguyên số bản sao.
3. Nếu `b1-kcm-hpa.txt` rỗng thì cluster không đặt cờ nào: chu kỳ đồng bộ và cửa sổ thu hẹp là mặc
   định dựng sẵn trong controller, tra ở bài 72 khi cần. Nếu có dòng, giá trị thật của cluster là
   dòng đó. Câu trả lời đúng là **một chỗ đọc**, không phải một con số thuộc lòng.

Bằng chứng `b1-top-web-containers.txt` thường liệt kê cả `nginx` lẫn `status-writer`: metrics-server
đo mọi container đang chạy của Pod, kể cả sidecar. B6 dùng điều này để kiểm phép chia của HPA.

</details>

## B2. [BS-11B-02] HPA `web` lên production

**Loại:** Tính năng.

**Bối cảnh:** bản thiết kế `docs/hpa-web.draft.yaml` của Lab 4b đã qua server dry-run nhưng chưa bao
giờ chạy. B1 vừa chứng minh mọi điều kiện của nó đã đủ. Giờ nó phải thành một object do repo quản lý,
được `deploy.sh` tạo ra như mọi object khác — và phải đối chiếu lại từng quyết định của bản nháp với
số đo thật.

**Yêu cầu:**

1. HPA `web` khai trong `k8s/app/web/hpa.yaml`, được `scripts/deploy.sh` tạo ra. Bản nháp rời khỏi
   `docs/`; lịch sử của nó vẫn truy được trong git (đổi tên, không chép rồi xóa).
2. Theo [hợp đồng](DU-AN-BOOKSTORE.md#32-thành-phần-ứng-dụng): đích là Deployment `web`, 2–5 bản sao,
   co giãn theo mức dùng CPU tính trên `requests`; object mang label của dự án và của thành phần
   `web`. Mục tiêu phần trăm do bạn chọn, **mọi chỗ khác với bản nháp phải có lý do** trong ADR.
3. Thu hẹp chậm hơn mở rộng, và điều đó **nằm trong chính object** — đọc được bằng `kubectl get`, không
   phải đi tra cấu hình controller. Sprint này giữ đúng cửa sổ thu hẹp của bản nháp; B7 mới tinh
   chỉnh sau khi đã đo.
4. HPA tính được: với tới đích, tính được số bản sao từ metric, `status` có giá trị metric hiện tại.
5. Cùng thông số đó tạo bằng cách mệnh lệnh của bài 342 — **không lưu** — phải cho đúng đích, khoảng
   và mục tiêu như manifest của bạn.
6. Khi `web` ở `minReplicas` của HPA, PodDisruptionBudget `web` của sprint 3 vẫn cho phép **ít nhất
   một** gián đoạn tự nguyện — Lab 12 sẽ drain node dựa vào điều này.
7. ADR mới `docs/adr/NNNN-hpa-web-production.md`; ADR HPA của Lab 4b đổi trạng thái thành
   *bị thay bởi NNNN*.

**Ràng buộc:** không tạo HPA thật bằng `kubectl autoscale` — HPA thứ hai cùng chọn Pod `web` sẽ làm cả
hai mất khả năng co giãn. Chưa đụng tới `replicas` của Deployment `web` ở ticket này (B3). Không đổi
`requests` của `web` — đó là mẫu số đã chốt ở sprint 3.

**Đọc lại:**

- Bài [72](../72-horizontal-pod-autoscale-vi.md):
  [*Đối tượng API*](../72-horizontal-pod-autoscale-vi.md#đối-tượng-api-api-object),
  [*Hỗ trợ metric tài nguyên*](../72-horizontal-pod-autoscale-vi.md#hỗ-trợ-metric-tài-nguyên-support-for-resource-metrics),
  [*Hành vi co giãn có thể cấu hình*](../72-horizontal-pod-autoscale-vi.md#hành-vi-co-giãn-có-thể-cấu-hình-configurable-scaling-behavior),
  [*Hành vi mặc định*](../72-horizontal-pod-autoscale-vi.md#hành-vi-mặc-định-default-behavior),
  [*Hỗ trợ HorizontalPodAutoscaler trong kubectl*](../72-horizontal-pod-autoscale-vi.md#hỗ-trợ-horizontalpodautoscaler-trong-kubectl-support-for-horizontalpodautoscaler-in-kubectl).
- Bài [342](../342-hpa-walkthrough-vi.md):
  [*Tạo HorizontalPodAutoscaler*](../342-hpa-walkthrough-vi.md#create-horizontal-pod-autoscaler),
  [*Tạo autoscaler theo cách khai báo*](../342-hpa-walkthrough-vi.md#tạo-autoscaler-theo-cách-khai-báo-creating-the-autoscaler-declaratively),
  [*Phụ lục: Các status condition*](../342-hpa-walkthrough-vi.md#phụ-lục-các-status-condition-của-horizontal-pod-autoscaler-appendix-horizontal-pod-autoscaler-status-conditions).
- Bài [339](../339-configure-pdb-vi.md):
  [*Chỉ định một PodDisruptionBudget*](../339-configure-pdb-vi.md#chỉ-định-một-poddisruptionbudget-specifying-a-poddisruptionbudget)
  — đoạn cuối về `maxUnavailable` khi số replica thay đổi; [*Logic làm tròn*](../339-configure-pdb-vi.md#logic-làm-tròn-khi-chỉ-định-phần-trăm-rounding-logic-when-specifying-percentages).

**Bàn giao:** `k8s/app/web/hpa.yaml`; `k8s/app/kustomization.yaml`; manifest PDB `web` nếu phải sửa;
`docs/adr/NNNN-hpa-web-production.md`; ADR HPA của Lab 4b đã đổi trạng thái;
`~/lab-evidence/11b/b2-*.txt`.

**Nghiệm thu:**

- `PASS:` HPA nằm ở `k8s/app/web/hpa.yaml` và trong kustomization; bản nháp đã rời `docs/`.
- `PASS:` `deploy.sh` tạo HPA bằng `kubectl apply` — object mang annotation last-applied.
- `PASS:` đích `Deployment/web`, 2–5 bản sao, metric `Resource/cpu/Utilization`; label dự án.
- `PASS:` cửa sổ ổn định thu hẹp dài hơn mở rộng; API server điền mặc định cho phần `behavior` bạn
  không khai.
- `PASS:` `AbleToScale=True`; `ScalingActive=True` lý do `ValidMetricFound`; `currentMetrics` có giá
  trị.
- `PASS:` `kubectl autoscale` dạng server dry-run cho cùng đích, khoảng, mục tiêu; namespace chỉ có
  một HPA.
- `PASS:` ở `minReplicas`, PDB `web` cho phép ít nhất một gián đoạn.
- `PASS:` có ADR mới; ADR của Lab 4b ghi *bị thay bởi*.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/NNNN-hpa-web-production.md`:

1. Liệt kê mọi chỗ manifest của bạn khác bản nháp (đọc bằng `git diff -M lab-11a`). Với mục tiêu phần
   trăm: số đo ở B1 nói gì? Mục tiêu thấp hơn đổi được gì và mất gì, nếu nghĩ tới khoảng thời gian từ
   lúc tải tăng tới lúc Pod mới Ready?
2. Object `kubectl autoscale` sinh ra khác object của bạn ở những trường nào ngoài bốn trường đã so?
   Vì sao dự án không tạo HPA bằng lệnh đó dù nó ngắn hơn?
3. Bạn chỉ khai `scaleDown` mà `kubectl get hpa web -o yaml` lại hiện cả `scaleUp`. Dẫn đúng câu của
   bài 72 giải thích điều đó.
4. PDB `web` của sprint 3 viết bằng trường nào? Ở 2 bản sao nó còn cho drain một node không? Vì sao bài
   339 khuyến nghị `maxUnavailable` khi số bản sao do người khác quyết định?

<details>
<summary>Gợi ý</summary>

- `git mv docs/hpa-web.draft.yaml k8s/app/web/hpa.yaml` giữ lịch sử; sau commit,
  `git log --follow k8s/app/web/hpa.yaml` thấy cả đời bản nháp.
- Thêm `- web/hpa.yaml` vào `resources` của `k8s/app/kustomization.yaml`.
- Bảng hợp đồng [mục 4.1](DU-AN-BOOKSTORE.md#41-tên-và-label): object của thành phần `web` mang
  thêm `app.kubernetes.io/name`.
- So cách mệnh lệnh mà không tạo gì: `kubectl autoscale ... --name=web-thu --dry-run=server -o yaml`.
- Trạng thái nghỉ: `kubectl -n bookstore describe hpa web` — đọc phần `Conditions`.
- Số gián đoạn PDB cho phép đọc ở `.status.disruptionsAllowed`; làm tròn theo bài 339.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong</summary>

```bash
. ~/lab-work/11b/ham.sh
cd ~/bookstore

# 1. Repo và cách triển khai
{ test -f k8s/app/web/hpa.yaml && test ! -e docs/hpa-web.draft.yaml && grep -q 'web/hpa.yaml' k8s/app/kustomization.yaml; } \
  && echo 'PASS: HPA nam o k8s/app/web/hpa.yaml va trong kustomization; ban nhap da roi docs/'
./scripts/deploy.sh > "$EV/b2-deploy.txt" 2>&1 \
  && [ -n "$(jp hpa/web '{.metadata.annotations.kubectl\.kubernetes\.io/last-applied-configuration}')" ] \
  && echo 'PASS: deploy.sh tao HPA web bang kubectl apply'

# 2. Spec theo hợp đồng
S="$(jp hpa/web '{.spec.scaleTargetRef.kind}/{.spec.scaleTargetRef.name} {.spec.minReplicas} {.spec.maxReplicas} {.spec.metrics[0].type}/{.spec.metrics[0].resource.name}/{.spec.metrics[0].resource.target.type}')"
TGT="$(jp hpa/web '{.spec.metrics[0].resource.target.averageUtilization}')"
echo "$S muc tieu=${TGT}%" | tee "$EV/b2-spec.txt"
{ [ "$S" = 'Deployment/web 2 5 Resource/cpu/Utilization' ] && \
  [ "$(jp hpa/web '{.metadata.labels.app\.kubernetes\.io/part-of}')" = 'bookstore' ] && \
  [ "$(jp hpa/web '{.metadata.labels.app\.kubernetes\.io/component}')" = 'web' ]; } \
  && echo 'PASS: HPA web 2-5 ban sao, CPU Utilization, label du an'

# 3. behavior
SD="$(jp hpa/web '{.spec.behavior.scaleDown.stabilizationWindowSeconds}')"
SU="$(jp hpa/web '{.spec.behavior.scaleUp.stabilizationWindowSeconds}')"
SUP="$(jp hpa/web '{.spec.behavior.scaleUp.selectPolicy}')"
echo "scaleDown.window=${SD:-<khong co>} scaleUp.window=${SU:-<khong co>} scaleUp.selectPolicy=${SUP:-<khong co>}" \
  | tee -a "$EV/b2-spec.txt"
{ [ -n "$SD" ] && [ "$SD" -gt "${SU:-0}" ]; } && echo 'PASS: cua so on dinh thu hep dai hon mo rong, nam trong object'
{ [ "$SU" = '0' ] && [ "$SUP" = 'Max' ]; } && echo 'PASS: API server dien mac dinh cho scaleUp ban khong khai'

# 4. HPA tính được — chờ vài chu kỳ đồng bộ, số vòng phụ thuộc cấu hình
for i in $(seq 1 20); do
  [ "$(hpa_cond ScalingActive status)" = 'True' ] && [ -n "$(web_util)" ] && break
  sleep "$POLL"
done
kubectl -n "$NS" describe hpa web > "$EV/b2-describe-nghi.txt"
[ "$(hpa_cond AbleToScale status)" = 'True' ] && echo "PASS: AbleToScale=True ($(hpa_cond AbleToScale reason))"
{ [ "$(hpa_cond ScalingActive status)" = 'True' ] && [ "$(hpa_cond ScalingActive reason)" = 'ValidMetricFound' ]; } \
  && echo 'PASS: ScalingActive=True, ValidMetricFound'
[ -n "$(web_util)" ] && echo "PASS: currentMetrics co gia tri: $(web_util)% / muc tieu ${TGT}%"
echo "ScalingLimited=$(hpa_cond ScalingLimited status) ($(hpa_cond ScalingLimited reason))" | tee -a "$EV/b2-spec.txt"

# 5. Cách mệnh lệnh của bài 342 — không lưu
F='{.spec.scaleTargetRef.kind}/{.spec.scaleTargetRef.name} {.spec.minReplicas} {.spec.maxReplicas} {.spec.metrics[0].resource.name}/{.spec.metrics[0].resource.target.type}/{.spec.metrics[0].resource.target.averageUtilization}'
A="$(kubectl -n "$NS" autoscale deployment web --name=web-thu --min=2 --max=5 --cpu="${TGT}%" \
  --dry-run=server -o jsonpath="$F" 2>&1)"
B="$(jp hpa/web "$F")"
echo "autoscale: $A | repo: $B" | tee "$EV/b2-autoscale-vs-repo.txt"
{ [ -n "$B" ] && [ "$A" = "$B" ]; } && echo 'PASS: kubectl autoscale va manifest cho cung dich, khoang va muc tieu'
[ "$(kubectl -n "$NS" get hpa --no-headers 2>/dev/null | wc -l)" -eq 1 ] && echo 'PASS: namespace chi co mot HPA'

# 6. PDB web ở minReplicas
MIN="$(jp hpa/web '{.spec.minReplicas}')"
PMA="$(jp pdb/web '{.spec.maxUnavailable}')"; PMI="$(jp pdb/web '{.spec.minAvailable}')"
if [ -n "$PMA" ]; then
  case "$PMA" in *%) ALLOW=$(( (MIN * ${PMA%\%} + 99) / 100 )) ;; *) ALLOW="$PMA" ;; esac
else
  case "$PMI" in *%) KEEP=$(( (MIN * ${PMI%\%} + 99) / 100 )) ;; *) KEEP="${PMI:-$MIN}" ;; esac
  ALLOW=$(( MIN - KEEP ))
fi
echo "PDB web minAvailable=${PMI:--} maxUnavailable=${PMA:--}: o $MIN ban sao cho phep $ALLOW gian doan" \
  | tee "$EV/b2-pdb.txt"
[ "$ALLOW" -ge 1 ] && echo 'PASS: o minReplicas, PDB web van cho phep it nhat mot gian doan tu nguyen'

# 7. ADR
{ ls docs/adr/*-hpa-web-production.md >/dev/null 2>&1 && \
  grep -l 'bị thay bởi' docs/adr/*-hpa-web.md >/dev/null 2>&1; } \
  && echo 'PASS: co ADR hpa-web-production; ADR hpa-web cua Lab 4b da danh dau bi thay'
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

```bash
cd ~/bookstore
git mv docs/hpa-web.draft.yaml k8s/app/web/hpa.yaml
# sửa nội dung như bên dưới, rồi thêm "- web/hpa.yaml" vào resources của k8s/app/kustomization.yaml
git diff -M --cached lab-11a -- k8s/app/web/hpa.yaml   # đúng phần khác so với bản nháp
./scripts/deploy.sh
```

```yaml
# k8s/app/web/hpa.yaml — đổi tên từ docs/hpa-web.draft.yaml (Lab 4b), áp bằng scripts/deploy.sh.
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web
  namespace: bookstore
  labels:
    app.kubernetes.io/part-of: bookstore
    app.kubernetes.io/component: web
    app.kubernetes.io/name: nginx        # khác bản nháp: hợp đồng 4.1 cho object của thành phần
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web
  minReplicas: 2
  maxReplicas: 5
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50           # khác bản nháp (70) — lý do ở ADR, câu 1
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300    # giữ thiết kế 4b; B7 tinh chỉnh sau khi đo
```

Nếu PDB `web` của sprint 3 là `minAvailable: 2` (đặt khi `web` có 3 bản sao), ở 2 bản sao nó không
cho gián đoạn nào và Lab 12 sẽ không drain được node. Đổi sang dạng tự co theo số bản sao:

```yaml
# manifest PDB web (đặt ở Lab 3c) — chỉ đổi phần spec
spec:
  maxUnavailable: 1
  selector:
    matchLabels:
      app.kubernetes.io/part-of: bookstore
      app.kubernetes.io/component: web
```

ADR 4b: đổi dòng trạng thái thành `- Trạng thái: bị thay bởi NNNN`.

Ý chính cho ADR:

1. Hai chỗ khác: thêm label `name`; mục tiêu 70% → 50%. B1 cho thấy `nginx` rất rẻ CPU trên mỗi
   request. Từ lúc tải tăng tới lúc Pod mới Ready có ít nhất một cửa sổ đo của metrics-server, một
   chu kỳ đồng bộ của HPA và thời gian khởi động Pod — trong khoảng đó các Pod cũ phải gánh hết. Mục
   tiêu thấp hơn chừa nhiều đệm hơn cho khoảng ấy (bài 342 cũng chọn 50%); cái giá là chạy nhiều bản
   sao hơn ở cùng một mức tải.
2. Object của `kubectl autoscale` không mang label dự án và không có `behavior`. Quan trọng hơn: nó
   không nằm trong repo, nên teardown + deploy là mất — vi phạm quy tắc "repo là nguồn sự thật". Bài
   342 trình bày hai cách cho ra cùng một object; dự án chọn cách khai báo.
3. Bài 72, *Hành vi mặc định*: "Các giá trị tùy chỉnh này được hợp nhất (merge) với các giá trị mặc
   định." API server lưu phần `scaleUp` mặc định vào object vì bạn đã khai `behavior`.
4. Với `minAvailable: 2` và 2 bản sao, `disruptionsAllowed` là 0. Bài 339: dùng `maxUnavailable` được
   khuyến nghị "vì nó tự động phản ứng với những thay đổi về số replica của controller tương ứng" —
   đúng tình huống khi HPA quyết định số bản sao.

</details>

## B3. [BS-11B-03] Bỏ `replicas` khỏi Deployment `web`

**Loại:** Tính năng, mở đầu bằng diễn tập.

**Bối cảnh:** từ B2, `web` có hai người cùng ghi một trường: HPA ghi `replicas` qua subresource `scale`,
còn `deploy.sh` ghi `replicas` theo manifest mỗi lần triển khai. Sprint 9 cấp cho CI quyền apply
manifest — tức mỗi lần CI chạy cũng là một lần ghi đè. Bài 72 và bài 63 đều nói: HPA đang quản thì
manifest không được khai `replicas`. Nhưng gỡ sai cách cũng làm `web` tụt bản sao ngay trên hệ thống
đang chạy.

**Yêu cầu:**

1. **Trước khi sửa gì**, chứng minh xung đột trên hệ thống thật: HPA đã chọn một số bản sao khác số
   trong manifest; một lần `./scripts/deploy.sh` kéo `web` về số của manifest; HPA kéo lại sau đó.
2. Manifest Deployment `web` mà `deploy.sh` áp không còn khai `replicas`.
3. Chuyển đổi **không** làm `web` tụt dưới `minReplicas` của HPA vào bất kỳ lúc nào — đúng quy trình
   chuyển đổi của bài 72.
4. Sau chuyển đổi, chạy `./scripts/deploy.sh` không còn đụng tới số bản sao mà HPA đã chọn.
5. ⭐ Chứng minh điều bài 72 cảnh báo bằng một Deployment thử ở `~/lab-work/11b/`: gỡ thẳng `replicas`
   rồi apply thì số bản sao tụt về mặc định.
6. ADR `docs/adr/NNNN-web-bo-replicas.md`.

**Ràng buộc:** không sửa `scripts/deploy.sh`. Không `kubectl edit` hay `kubectl scale` Deployment `web`.
Deployment thử ở yêu cầu 5 không mang label `app.kubernetes.io/part-of`, ghim `lab-k8s-worker2`, đạt
PSS `restricted`, xóa khi xong.

**Đọc lại:**

- Bài [72](../72-horizontal-pod-autoscale-vi.md):
  [*Chuyển Deployment và StatefulSet sang tự động co giãn theo chiều ngang*](../72-horizontal-pod-autoscale-vi.md#chuyển-deployment-và-statefulset-sang-tự-động-co-giãn-theo-chiều-ngang-migrating-deployments-and-statefulsets-to-horizontal-autoscaling),
  [*Tính ổn định của quy mô workload*](../72-horizontal-pod-autoscale-vi.md#flapping).
- Bài [63](../63-deployment-vi.md): [*Replicas*](../63-deployment-vi.md#replicas).
- Bài [61](../61-management-vi.md): [*kubectl apply*](../61-management-vi.md#kubectl-apply) — câu "không
  ghi đè bất kỳ thay đổi tự động nào trên các thuộc tính mà bạn không chỉ định".

**Bàn giao:** manifest Deployment `web` không còn `replicas`; `docs/adr/NNNN-web-bo-replicas.md`;
`~/lab-evidence/11b/b3-*.txt`.

**Nghiệm thu:**

- `PASS:` (trước khi sửa) HPA chọn số bản sao khác manifest; trong lúc `deploy.sh` chạy, `replicas`
  nhảy về số của manifest rồi bị HPA kéo lại.
- `PASS:` bản render của Deployment `web` không còn `replicas`; annotation last-applied của nó cũng
  không.
- `PASS:` `deploy.sh` chạy lại: `replicas` không đổi suốt quá trình, không lúc nào dưới `minReplicas`.
- `PASS:` HPA không có sự kiện nào phải kéo `web` lên vì tụt dưới `minReplicas`.
- `PASS:` (⭐) Deployment thử tụt về 1 bản sao khi gỡ thẳng `replicas`.
- `PASS:` có ADR.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/NNNN-web-bo-replicas.md`:

1. Vì sao `kubectl apply` ghi đè con số HPA đã chọn khi manifest còn `replicas`, nhưng không đụng tới
   nó khi manifest không khai? Dẫn câu của bài 61.
2. Vì sao gỡ dòng `replicas` rồi chạy thẳng `deploy.sh` lại làm `web` tụt? Bài 72 và bài 63 nói giá trị
   mặc định là bao nhiêu, và "trí nhớ" nào của `kubectl apply` gây ra cú tụt đó?
3. Sau teardown + deploy sạch, Deployment `web` mới tạo có mấy bản sao trong vài giây đầu, ai đưa nó
   lên `minReplicas`, và `tests/smoke.sh` của sprint này phải tính tới điều gì?
4. CI (`ci-deployer` của sprint 15) cũng apply manifest. Quyết định này đổi gì cho CI?

<details>
<summary>Gợi ý</summary>

- Theo dõi `replicas` trong lúc một lệnh khác chạy: một vòng lặp ở nền ghi giá trị mỗi giây ra file.
- `deploy.sh` dùng `kubectl apply` phía client: nó so cấu hình mới với annotation
  `kubectl.kubernetes.io/last-applied-configuration` trên object (Lab 4a đã mổ xẻ annotation này).
- Bài 72 dùng `kubectl apply edit-last-applied` (mở trình soạn thảo). Lệnh anh em không cần trình
  soạn thảo: `kubectl apply set-last-applied -f <file>`. File phải là **đúng bản mà `deploy.sh` sẽ
  áp** — bản render của Kustomize, không phải file thô trong repo. Hàm `render_web` trong `ham.sh` in
  ra đúng Deployment `web` đó.
- Thứ tự quyết định kết quả: annotation trước, apply sau.

</details>

<details>
<summary>Script nghiệm thu — phần 1 chạy TRƯỚC khi sửa manifest; đây chính là cuộc diễn tập</summary>

```bash
. ~/lab-work/11b/ham.sh
cd ~/bookstore
MAN="$(render_web | sed -n 's/^  replicas: *\([0-9][0-9]*\).*/\1/p')"
MIN="$(jp hpa/web '{.spec.minReplicas}')"
echo "manifest replicas=${MAN:-<khong co>} | HPA minReplicas=$MIN" | tee "$EV/b3-truoc.txt"
[ -n "$MAN" ] && [ "$MAN" -gt "$MIN" ] && echo 'PASS: manifest con replicas va lon hon minReplicas — dien tap duoc'

# HPA lúc nhàn chọn minReplicas; chờ nó chọn xong
for i in $(seq 1 40); do
  R="$(web_rep)"; D="$(jp hpa/web '{.status.desiredReplicas}')"
  echo "$(date +%T) vong=$i replicas=$R hpa.desired=$D"
  [ "$R" = "$D" ] && [ "$R" != "$MAN" ] && break
  sleep "$POLL"
done
echo "HPA chon $R ban sao, manifest ghi $MAN" | tee -a "$EV/b3-truoc.txt"
[ "$R" != "$MAN" ] && echo 'PASS: HPA da tu chon so ban sao khac manifest'

# Ghi replicas mỗi giây trong lúc deploy.sh chạy
( for i in $(seq 1 120); do web_rep; echo; sleep 1; done ) > "$EV/b3-replicas-deploy-1.txt" &
W=$!
./scripts/deploy.sh > "$EV/b3-deploy-1.txt" 2>&1
wait "$W"
echo "chuoi replicas: $(grep -v '^$' "$EV/b3-replicas-deploy-1.txt" | uniq | tr '\n' ' ')" | tee -a "$EV/b3-truoc.txt"
grep -qx "$MAN" "$EV/b3-replicas-deploy-1.txt" && echo "PASS: deploy.sh keo replicas ve $MAN cua manifest"
[ "$(grep -v '^$' "$EV/b3-replicas-deploy-1.txt" | tail -1)" = "$R" ] \
  && echo "PASS: HPA keo lai ve $R — hai ben dang ghi de cung mot truong"
kubectl -n "$NS" describe hpa web | grep -A 8 '^Events:' | tee -a "$EV/b3-truoc.txt"
```

</details>

<details>
<summary>Script nghiệm thu — phần 2 chạy sau khi tự làm xong</summary>

```bash
. ~/lab-work/11b/ham.sh
cd ~/bookstore
[ -n "$(render_web)" ] && [ -z "$(render_web | grep '^  replicas:')" ] \
  && echo 'PASS: ban render cua Deployment web khong con replicas'
jp deployment/web '{.metadata.annotations.kubectl\.kubernetes\.io/last-applied-configuration}' > "$EV/b3-last-applied.json"
{ [ -s "$EV/b3-last-applied.json" ] && ! grep -q '"replicas"' "$EV/b3-last-applied.json"; } \
  && echo 'PASS: last-applied cua Deployment web khong con replicas'

R0="$(web_rep)"; MIN="$(jp hpa/web '{.spec.minReplicas}')"
( for i in $(seq 1 120); do web_rep; echo; sleep 1; done ) > "$EV/b3-replicas-deploy-2.txt" &
W=$!
./scripts/deploy.sh > "$EV/b3-deploy-2.txt" 2>&1; echo "deploy exit=$?"
wait "$W"
VALS="$(grep -v '^$' "$EV/b3-replicas-deploy-2.txt" | sort -u | tr '\n' ' ')"
echo "replicas truoc=$R0; gia tri thay trong luc deploy: $VALS" | tee "$EV/b3-sau.txt"
[ "$VALS" = "$R0 " ] && echo 'PASS: deploy.sh khong con dong toi so ban sao HPA da chon'
[ "$(grep -v '^$' "$EV/b3-replicas-deploy-2.txt" | sort -n | head -1)" -ge "$MIN" ] \
  && echo 'PASS: so ban sao khong luc nao duoi minReplicas'

kubectl -n "$NS" get events --field-selector involvedObject.kind=HorizontalPodAutoscaler,involvedObject.name=web \
  -o jsonpath='{range .items[*]}{.message}{"\n"}{end}' > "$EV/b3-events-hpa.txt"
grep -q 'below Spec.MinReplicas' "$EV/b3-events-hpa.txt" \
  && echo 'FAIL: HPA tung phai keo web len vi tut duoi minReplicas — doc b3-events-hpa.txt' \
  || echo 'PASS: khong co lan nao web tut duoi minReplicas trong luc chuyen doi'
ls docs/adr/*-web-bo-replicas.md >/dev/null 2>&1 && echo 'PASS: co ADR web-bo-replicas'
```

⭐ Deployment thử cho yêu cầu 5 — tự viết `~/lab-work/11b/bo-replicas-thu.yaml` theo ràng buộc (3 bản
sao, `busybox` ngủ), rồi:

```bash
. ~/lab-work/11b/ham.sh
kubectl apply -f "$WK/bo-replicas-thu.yaml"
kubectl -n "$NS" rollout status deployment/bo-replicas-thu --timeout=120s
grep -v '^  replicas:' "$WK/bo-replicas-thu.yaml" > "$WK/bo-replicas-thu-go-thang.yaml"
kubectl apply -f "$WK/bo-replicas-thu-go-thang.yaml"
R="$(jp deployment/bo-replicas-thu '{.spec.replicas}')"
echo "sau khi go thang dong replicas roi apply: replicas=$R" | tee "$EV/b3-thu-go-thang.txt"
[ "$R" = '1' ] && echo 'PASS: go thang replicas roi apply lam Deployment tut ve 1 ban sao'
kubectl -n "$NS" delete -f "$WK/bo-replicas-thu.yaml" --wait=true
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

Chuyển đổi không tụt bản sao — annotation trước, apply sau:

```bash
. ~/lab-work/11b/ham.sh
cd ~/bookstore
# 1. Gỡ replicas khỏi manifest trong repo; git diff phải chỉ mất đúng một dòng.
sed -i '/^  replicas: [0-9]*$/d' k8s/app/web/deployment.yaml
git diff k8s/app/web/deployment.yaml
# 2. Đặt lại riêng annotation last-applied của Deployment web cho khớp bản render mới.
render_web > "$WK/web-deployment.yaml"
grep -c '^kind: Deployment$' "$WK/web-deployment.yaml"    # phải là 1
kubectl apply set-last-applied -f "$WK/web-deployment.yaml"
# 3. Giờ mới triển khai.
./scripts/deploy.sh
```

Cách tương đương của bài 72: `kubectl -n bookstore apply edit-last-applied deployment/web`, xóa dòng
`replicas` trong trình soạn thảo, lưu — rồi mới sửa repo và chạy `deploy.sh`.

⭐ Deployment thử:

```yaml
# ~/lab-work/11b/bo-replicas-thu.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: bo-replicas-thu
  namespace: bookstore
spec:
  replicas: 3
  selector:
    matchLabels:
      thu: bo-replicas
  template:
    metadata:
      labels:
        thu: bo-replicas
    spec:
      automountServiceAccountToken: false
      terminationGracePeriodSeconds: 1
      nodeSelector:
        kubernetes.io/hostname: lab-k8s-worker2
      securityContext:
        runAsNonRoot: true
        runAsUser: 65534
        seccompProfile:
          type: RuntimeDefault
      containers:
      - name: ngu
        image: busybox:1.37
        command: ["sh", "-c", "sleep 3600"]
        resources:
          requests:
            cpu: 10m
            memory: 16Mi
          limits:
            cpu: 20m
            memory: 32Mi
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop: ["ALL"]
```

Ý chính cho ADR:

1. `kubectl apply` đặt mọi trường có trong cấu hình về đúng giá trị cấu hình, bất kể giá trị đang
   sống do ai ghi. Bài 61: apply "không ghi đè bất kỳ thay đổi tự động nào trên các thuộc tính mà bạn
   không chỉ định" — manifest không khai `replicas` thì trường đó thuộc trọn về HPA.
2. Mặc định của `.spec.replicas` là 1 (bài 63, *Replicas*). Annotation last-applied còn nhớ
   `replicas: 3`; apply thấy trường "đã bị bỏ khỏi cấu hình" nên xóa nó, API server điền mặc định 1.
   Bài 72 gọi đó là một lần suy giảm số Pod, và bảo sửa annotation trước.
3. Deployment mới tạo không có `replicas` nên bắt đầu với 1 bản sao; HPA đưa lên `minReplicas` ở
   chu kỳ đồng bộ kế tiếp — việc kẹp vào `minReplicas` không cần số đo. Trong khoảng ngắn đó `web`
   chỉ có một Pod. `smoke.sh` của sprint này phải chờ có giới hạn tới khi `web` đạt `minReplicas`
   thay vì kiểm một con số cố định.
4. CI apply manifest giờ không còn đè số bản sao; HPA là chủ duy nhất của `replicas`. Đổi số bản sao
   bằng tay trên hệ thống thật nghĩa là đổi `minReplicas`/`maxReplicas` của HPA trong repo.

</details>

## B4. [BS-11B-04] Công cụ kiểm thử tải có trần

**Loại:** Tính năng.

**Bối cảnh:** muốn tin HPA thì phải thấy nó phản ứng dưới tải. Bài 342 sinh tải bằng một `kubectl run`
chạy vòng lặp vô hạn trong một terminal riêng, dừng bằng `Ctrl+C` — quên một terminal là cluster chạy
hết đêm. Dự án cần một công cụ **lặp lại được, có phanh**, và nó phải sống được trong mạng zero-trust
của sprint 7: mặc định Pod trong `bookstore` không gọi được `web`, chỉ Traefik được vào cổng 8080.

**Yêu cầu:**

1. `tests/load/job.yaml`: một Job gửi HTTP tới Service `web` trong một khoảng thời gian cố định rồi tự
   kết thúc. Số Pod cố định; mỗi Pod có trần CPU và memory; cả Job có deadline; không thử lại khi
   lỗi; tự được dọn sau khi xong; mỗi Pod in ra số request thành công và lỗi.
2. Pod sinh tải chạy trên `lab-k8s-worker2`, dùng PriorityClass `bookstore-batch` (tác vụ nền không
   được giành chỗ của dịch vụ chính), không mount token ServiceAccount, đạt PSS `restricted`, nằm
   trong LimitRange `bookstore-limits`.
3. Tổng trần CPU của mọi Pod sinh tải không vượt **một nửa** CPU `allocatable` của `lab-k8s-worker2`.
4. `tests/load/networkpolicy.yaml`: mở **đúng một** đường — Pod sinh tải tới `web` cổng 8080 — không mở
   gì khác, không sửa ma trận NetworkPolicy của dự án trong `k8s/app/network/`.
5. Object của công cụ mang label dự án với thành phần `loadtest`. Công cụ **không** nằm trong `k8s/`:
   `deploy.sh` không bao giờ áp nó.
6. Chứng minh: trước khi áp policy, Pod mang label `loadtest` không gọi được `web`; sau khi áp thì gọi
   được; Pod không mang label đó vẫn bị chặn.
7. ADR `docs/adr/NNNN-kiem-thu-tai.md`.

**Ràng buộc:** image `busybox` đã khóa ở A1.3; không dùng `hpa-example`. Pod dò mạng ở yêu cầu 6 nằm ở
`~/lab-work/11b/`, xóa khi xong. Sau ticket, không còn object `loadtest` nào trên cluster.

**Đọc lại:**

- Bài [342](../342-hpa-walkthrough-vi.md): [*Tăng tải*](../342-hpa-walkthrough-vi.md#increase-load),
  [*Dừng tạo tải*](../342-hpa-walkthrough-vi.md#stop-load).
- Bài [67](../67-job-vi.md): [*Thực thi song song cho Job*](../67-job-vi.md#parallel-jobs),
  [*Chính sách backoff khi Pod lỗi*](../67-job-vi.md#pod-backoff-failure-policy),
  [*Kết thúc và dọn dẹp Job*](../67-job-vi.md#job-termination-and-cleanup),
  [*Tự động dọn dẹp các Job đã hoàn tất*](../67-job-vi.md#clean-up-finished-jobs-automatically).
- Bài [84](../84-network-policies-vi.md): [*Tài nguyên NetworkPolicy*](../84-network-policies-vi.md#networkpolicy-resource),
  [*Hành vi của các bộ chọn `to` và `from`*](../84-network-policies-vi.md#hành-vi-của-các-bộ-chọn-to-và-from-behavior-of-to-and-from-selectors).
- Bài [115](../115-pod-security-standards-vi.md): [*Restricted*](../115-pod-security-standards-vi.md#restricted).
- Bài [141](../141-pod-priority-preemption-vi.md): [*PriorityClass*](../141-pod-priority-preemption-vi.md#priorityclass).

**Bàn giao:** `tests/load/job.yaml`; `tests/load/networkpolicy.yaml`; `docs/adr/NNNN-kiem-thu-tai.md`;
`~/lab-evidence/11b/b4-*.txt`.

**Nghiệm thu:**

- `PASS:` công cụ nằm ở `tests/load/`, không có gì của nó trong `k8s/`.
- `PASS:` số Pod cố định, không thử lại; có deadline và TTL; PriorityClass `bookstore-batch`; không
  mount token; image đã khóa.
- `PASS:` mỗi Pod có trần CPU và memory; tổng trần CPU ≤ một nửa `allocatable` của `lab-k8s-worker2`.
- `PASS:` server dry-run không có cảnh báo PodSecurity.
- `PASS:` chưa có policy: Pod `loadtest` bị chặn. Có policy: Pod `loadtest` gọi được `web`; Pod khác
  vẫn bị chặn.
- `PASS:` dọn xong, không còn object `loadtest`; có ADR.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/NNNN-kiem-thu-tai.md`:

1. Kể ba lớp phanh của công cụ và mỗi lớp chặn kiểu sự cố nào.
2. Vì sao cần **hai** NetworkPolicy mà không phải một? Mỗi cái chọn Pod nào, mở chiều nào? Pod sinh tải
   gọi `http://web/` cổng 80 — vì sao policy lại viết cổng 8080?
3. Vì sao công cụ nằm ở `tests/` chứ không ở `k8s/`? Nếu đặt policy vào `k8s/app/network/` thì ma trận
   zero-trust của [mục 3.5](DU-AN-BOOKSTORE.md#35-thiết-kế-bảo-mật) đổi thế nào?
4. Pod sinh tải chạy trong namespace `bookstore` nên tính vào quota của hệ thống thật. Chấp nhận được
   không? Lựa chọn nào tốt hơn nếu có cluster riêng?

<details>
<summary>Gợi ý</summary>

- Job: `parallelism` bằng `completions`; `backoffLimit: 0`; `activeDeadlineSeconds`;
  `ttlSecondsAfterFinished`; Pod `restartPolicy: Never`.
- Vòng tải trong `sh` của busybox: chạy vài luồng song song bằng `( ... ) &` rồi `wait`; mỗi luồng
  lặp `wget -q -T 2 -O /dev/null "$TARGET"` và đếm thành công, lỗi; kiểm đồng hồ bằng `date +%s` sau
  mỗi vài chục request để khỏi tốn thêm tiến trình.
- PSS `restricted`: `runAsNonRoot`, `runAsUser` khác 0, `allowPrivilegeEscalation: false`, bỏ mọi
  capability, `seccompProfile: RuntimeDefault`. `busybox` chạy được với uid 65534.
- Kubernetes thay `$(TÊN)` trong `command`/`args` bằng biến môi trường cùng tên nếu có. Trong script,
  dùng `$TÊN` của shell để tránh nhầm.
- Service `web` dịch cổng 80 sang cổng `http` 8080 của Pod; policy của sprint 7 viết theo cổng của
  Pod (Traefik → `web` 8080). Mở xem policy DNS hiện có chọn Pod nào.
- Thử từ trong một Pod: `kubectl exec <pod> -- timeout 5 wget -q -T 3 -O /dev/null http://web/`, mã
  thoát khác 0 là bị chặn. `timeout` bọc ngoài vì gói tin bị policy bỏ thì kết nối treo chứ không bị
  từ chối ngay.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong</summary>

```bash
. ~/lab-work/11b/ham.sh
cd ~/bookstore
{ test -f tests/load/job.yaml && test -f tests/load/networkpolicy.yaml && \
  ! grep -rqs 'component: loadtest' k8s/; } && echo 'PASS: cong cu nam o tests/load, khong co gi trong k8s/'

# 1. Hình dạng của Job — đọc qua API server, không tạo
kubectl -n "$NS" delete -f tests/load/ --ignore-not-found --wait=true >/dev/null 2>&1
F='{.spec.parallelism}|{.spec.completions}|{.spec.backoffLimit}|{.spec.activeDeadlineSeconds}|{.spec.ttlSecondsAfterFinished}|{.spec.template.spec.priorityClassName}|{.spec.template.spec.automountServiceAccountToken}|{.spec.template.spec.containers[0].resources.limits.cpu}|{.spec.template.spec.containers[0].resources.limits.memory}|{.spec.template.spec.containers[0].image}'
OUT="$(kubectl create --dry-run=server -f tests/load/job.yaml -o jsonpath="$F" 2> "$EV/b4-dry-run-err.txt")"
echo "$OUT" | tee "$EV/b4-job.txt"
IFS='|' read -r PAR COMP BL ADS TTL PC AM LCPU LMEM IMG <<< "$OUT"
{ [ -n "$PAR" ] && [ "$PAR" = "$COMP" ] && [ "$BL" = '0' ]; } && echo "PASS: $PAR Pod co dinh, khong thu lai"
{ [ -n "$ADS" ] && [ -n "$TTL" ]; } && echo "PASS: deadline ${ADS}s, tu don sau ${TTL}s"
{ [ "$PC" = 'bookstore-batch' ] && [ "$AM" = 'false' ] && [ "$IMG" = 'busybox:1.37' ]; } \
  && echo 'PASS: PriorityClass bookstore-batch, khong mount token, image da khoa'
A2="$(cpu_m "$(kubectl get node lab-k8s-worker2 -o jsonpath='{.status.allocatable.cpu}')")"
TOT=$(( PAR * $(cpu_m "$LCPU") ))
echo "tran CPU nguon tai = ${TOT}m tren ${A2}m allocatable cua lab-k8s-worker2" | tee -a "$EV/b4-job.txt"
{ [ -n "$LMEM" ] && [ "$TOT" -gt 0 ] && [ $(( TOT * 2 )) -le "$A2" ]; } \
  && echo 'PASS: moi Pod co tran CPU va memory; tong tran CPU <= mot nua allocatable cua worker2'
! grep -qi 'PodSecurity' "$EV/b4-dry-run-err.txt" && [ -n "$OUT" ] && echo 'PASS: khong co canh bao PodSecurity'

# 2. Hai Pod dò mạng giống Pod sinh tải: một mang label loadtest, một không
cat > "$WK/do-mang.yaml" <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: do-loadtest
  namespace: bookstore
  labels:
    app.kubernetes.io/part-of: bookstore
    app.kubernetes.io/component: loadtest
spec:
  restartPolicy: Never
  automountServiceAccountToken: false
  terminationGracePeriodSeconds: 1
  nodeSelector:
    kubernetes.io/hostname: lab-k8s-worker2
  securityContext:
    runAsNonRoot: true
    runAsUser: 65534
    runAsGroup: 65534
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: do
    image: busybox:1.37
    command: ["sh", "-c", "sleep 900"]
    resources:
      requests: {cpu: 10m, memory: 16Mi}
      limits: {cpu: 20m, memory: 32Mi}
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop: ["ALL"]
---
apiVersion: v1
kind: Pod
metadata:
  name: do-khac
  namespace: bookstore
  labels:
    app.kubernetes.io/part-of: bookstore
    app.kubernetes.io/component: do-mang-thu
spec:
  restartPolicy: Never
  automountServiceAccountToken: false
  terminationGracePeriodSeconds: 1
  nodeSelector:
    kubernetes.io/hostname: lab-k8s-worker2
  securityContext:
    runAsNonRoot: true
    runAsUser: 65534
    runAsGroup: 65534
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: do
    image: busybox:1.37
    command: ["sh", "-c", "sleep 900"]
    resources:
      requests: {cpu: 10m, memory: 16Mi}
      limits: {cpu: 20m, memory: 32Mi}
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop: ["ALL"]
EOF
kubectl apply -f "$WK/do-mang.yaml"
kubectl -n "$NS" wait --for=condition=Ready pod/do-loadtest pod/do-khac --timeout=120s
goi() { kubectl -n "$NS" exec "$1" -- timeout 5 wget -q -T 3 -O /dev/null http://web/ >/dev/null 2>&1; }

goi do-loadtest && echo 'FAIL: chua co policy ma Pod loadtest da goi duoc web' \
  || echo 'PASS: chua co policy: Pod loadtest bi chan'
kubectl apply -f tests/load/networkpolicy.yaml
OK=0; for i in $(seq 1 6); do goi do-loadtest && { OK=1; break; }; sleep 5; done
[ "$OK" -eq 1 ] && echo 'PASS: co policy: Pod loadtest goi duoc web'
goi do-khac && echo 'FAIL: Pod khong mang label loadtest cung goi duoc web' \
  || echo 'PASS: Pod khong mang label loadtest van bi chan'

# 3. Dọn
kubectl -n "$NS" delete -f "$WK/do-mang.yaml" --wait=true
kubectl -n "$NS" delete -f tests/load/ --ignore-not-found --wait=true
{ [ "$(kubectl -n "$NS" get all,networkpolicy -l app.kubernetes.io/component=loadtest --no-headers 2>/dev/null | wc -l)" -eq 0 ] && \
  ls docs/adr/*-kiem-thu-tai.md >/dev/null 2>&1; } && echo 'PASS: khong con object loadtest; co ADR kiem-thu-tai'
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

```yaml
# tests/load/job.yaml — kiểm thử tải cho HPA web. KHÔNG nằm trong k8s/: chỉ áp khi diễn tập.
# Áp:  kubectl apply -f tests/load/      Dọn:  kubectl -n bookstore delete -f tests/load/
apiVersion: batch/v1
kind: Job
metadata:
  name: loadtest-web
  namespace: bookstore
  labels:
    app.kubernetes.io/part-of: bookstore
    app.kubernetes.io/component: loadtest
    app.kubernetes.io/name: busybox
spec:
  parallelism: 2
  completions: 2
  backoffLimit: 0
  activeDeadlineSeconds: 420
  ttlSecondsAfterFinished: 900
  template:
    metadata:
      labels:
        app.kubernetes.io/part-of: bookstore
        app.kubernetes.io/component: loadtest
        app.kubernetes.io/name: busybox
    spec:
      restartPolicy: Never
      priorityClassName: bookstore-batch
      automountServiceAccountToken: false
      terminationGracePeriodSeconds: 5
      nodeSelector:
        kubernetes.io/hostname: lab-k8s-worker2
      securityContext:
        runAsNonRoot: true
        runAsUser: 65534
        runAsGroup: 65534
        seccompProfile:
          type: RuntimeDefault
      containers:
      - name: gen
        image: busybox:1.37
        env:
        - name: TARGET
          value: http://web/
        - name: DURATION
          value: "240"
        - name: WORKERS
          value: "4"
        command: ["sh", "-c"]
        args:
        - |
          END=$(( $(date +%s) + DURATION ))
          echo "loadtest: bat dau $WORKERS luong toi $TARGET trong ${DURATION}s"
          w=1
          while [ "$w" -le "$WORKERS" ]; do
            ( ok=0; err=0; i=0
              while :; do
                if wget -q -T 2 -O /dev/null "$TARGET"; then ok=$((ok+1)); else err=$((err+1)); fi
                i=$((i+1))
                if [ $((i % 20)) -eq 0 ] && [ "$(date +%s)" -ge "$END" ]; then break; fi
              done
              echo "loadtest: luong $w xong ok=$ok loi=$err" ) &
            w=$((w+1))
          done
          wait
          echo "loadtest: xong"
        resources:
          requests:
            cpu: 50m
            memory: 16Mi
          limits:
            cpu: 300m
            memory: 32Mi
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop: ["ALL"]
```

```yaml
# tests/load/networkpolicy.yaml — mở đúng một đường: Pod loadtest -> web:8080. Áp cùng Job, dọn cùng Job.
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: loadtest-egress-web
  namespace: bookstore
  labels:
    app.kubernetes.io/part-of: bookstore
    app.kubernetes.io/component: loadtest
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/part-of: bookstore
      app.kubernetes.io/component: loadtest
  policyTypes: ["Egress"]
  egress:
  - to:
    - podSelector:
        matchLabels:
          app.kubernetes.io/part-of: bookstore
          app.kubernetes.io/component: web
    ports:
    - protocol: TCP
      port: 8080
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: web-ingress-loadtest
  namespace: bookstore
  labels:
    app.kubernetes.io/part-of: bookstore
    app.kubernetes.io/component: loadtest
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/part-of: bookstore
      app.kubernetes.io/component: web
  policyTypes: ["Ingress"]
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app.kubernetes.io/part-of: bookstore
          app.kubernetes.io/component: loadtest
    ports:
    - protocol: TCP
      port: 8080
```

Ý chính cho ADR:

1. Số Pod cố định (`parallelism`, không thử lại) chặn tải tự nhân lên; trần CPU mỗi Pod chặn một Pod
   ăn hết node; deadline của Job cộng đồng hồ trong script chặn tải chạy mãi khi người vận hành bỏ đi.
   TTL dọn object nhưng không phải phanh.
2. Sprint 7 cô lập **cả hai chiều**: Pod sinh tải cần quyền đi ra (egress, chọn Pod `loadtest`), Pod
   `web` cần quyền nhận vào (ingress, chọn Pod `web`). Một NetworkPolicy chỉ có một `podSelector` nên
   không chọn được cả hai phía. Policy nhìn thấy gói tin tới cổng của Pod — 8080 — sau khi Service
   đã dịch cổng 80.
3. `k8s/` là thứ hệ thống chạy mọi lúc. Đặt policy vào đó là thêm vĩnh viễn một đường vào `web` cho bất
   kỳ Pod nào tự gắn label `loadtest` — ma trận 3.5 có thêm một dòng mà không có thành phần thật nào
   đứng sau. Ở `tests/`, đường đó chỉ tồn tại trong lúc diễn tập.
4. Chấp nhận được trên cluster lab, với điều kiện quota được tính cả phần này (B5) và tải chạy ở
   `bookstore-batch`. Cluster thật thì kiểm thử tải ở môi trường riêng có cùng metrics-server.

</details>

## B5. [BS-11B-05] Hạn mức đủ cho năm bản sao cộng surge

**Loại:** Tính năng, kèm diễn tập.

**Bối cảnh:** ResourceQuota `bookstore-quota` của sprint 11 được chốt khi `web` có số bản sao cố định.
Giờ HPA có thể đưa `web` lên `maxReplicas` đúng lúc một bản phát hành mới rollout, Pod sinh tải đang
chạy và CronJob `db-backup` tới lịch. Quota hết chỗ thì Pod mới không được tạo — HPA "mở rộng" mà
không có Pod nào thêm.

**Yêu cầu:**

1. Tính và ghi lại tài nguyên namespace cần ở trường hợp xấu nhất có kiểm soát: các thành phần khác ở
   trạng thái thường, cộng `web` ở `maxReplicas` cộng surge của chiến lược rolling update, cộng mọi Pod
   sinh tải của B4, cộng một Pod `db-backup` — cho từng loại tài nguyên mà quota giới hạn.
2. `bookstore-quota` đủ cho con số đó. Nếu không đủ, sửa manifest ResourceQuota trong `k8s/app/policy/`
   (giữ tên `bookstore-quota`) và triển khai lại bằng `deploy.sh`. Không nới quá xa — ghi lý do con số.
3. Diễn tập: ghim tạm HPA ở `maxReplicas`, phát hành lại `web` theo rolling update; rollout hoàn tất
   và **không** có Pod nào bị từ chối vì quota. Trả HPA về đúng repo bằng `deploy.sh`.
4. ADR `docs/adr/NNNN-quota-cho-hpa.md`.

**Ràng buộc:** không xóa hay tạm gỡ ResourceQuota, LimitRange để "cho qua". Patch HPA chỉ trong diễn
tập và phải được `deploy.sh` trả lại. Không đổi `maxReplicas`.

**Đọc lại:**

- Bài [134](../134-resource-quotas-vi.md):
  [*Hạn ngạch cho tài nguyên hạ tầng*](../134-resource-quotas-vi.md#compute-resource-quota),
  [*Hạn ngạch theo số lượng đối tượng*](../134-resource-quotas-vi.md#quota-on-object-count),
  [*Xem và thiết lập hạn ngạch*](../134-resource-quotas-vi.md#viewing-and-setting-quotas).
- Bài [63](../63-deployment-vi.md): [*Max Surge*](../63-deployment-vi.md#max-surge); ghi chú cuối mục
  [*Cập nhật một Deployment*](../63-deployment-vi.md#updating-a-deployment) về tài nguyên vượt
  `replicas + maxSurge` cho tới khi Pod đang kết thúc hết ân hạn.
- Bài [72](../72-horizontal-pod-autoscale-vi.md):
  [*Tự động co giãn trong khi rolling update*](../72-horizontal-pod-autoscale-vi.md#tự-động-co-giãn-trong-khi-rolling-update-autoscaling-during-rolling-update).
- Bài [110](../110-manage-resources-containers-vi.md): [*Yêu cầu và giới hạn*](../110-manage-resources-containers-vi.md#requests-and-limits).

**Bàn giao:** manifest ResourceQuota (nếu phải sửa); `docs/adr/NNNN-quota-cho-hpa.md`;
`~/lab-evidence/11b/b5-*.txt`.

**Nghiệm thu:**

- `PASS:` mỗi loại tài nguyên quota giới hạn đều đủ cho trường hợp xấu nhất ở yêu cầu 1.
- `PASS:` `web` lên đủ `maxReplicas` Ready khi HPA bị ghim.
- `PASS:` rolling update ở `maxReplicas` hoàn tất; không có sự kiện `exceeded quota` mới.
- `PASS:` `deploy.sh` trả `minReplicas` của HPA về đúng repo; có ADR.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/NNNN-quota-cho-hpa.md`:

1. Surge của `web` ở `maxReplicas` là bao nhiêu Pod theo chiến lược hiện tại, và làm tròn thế nào?
   Trong diễn tập, số Pod cao nhất quota ghi nhận có vượt `maxReplicas + surge` không? Bài 63 giải
   thích phần vượt đó thế nào?
2. Bài 72 nói ai quản `replicas` của Deployment và ai chia nó cho các ReplicaSet trong lúc rollout. Hệ
   quả với quota là gì?
3. Nếu quota thiếu, triệu chứng nhìn thấy ở HPA, ở Deployment và ở sự kiện khác nhau thế nào? HPA có
   biết mình bị quota chặn không?
4. Bạn nới quota bao nhiêu và vì sao không nới thành "thật rộng cho chắc"? Nghĩ tới [mục 3.6 của file
   dự án](DU-AN-BOOKSTORE.md#36-ngân-sách-tài-nguyên-và-lập-lịch) và tenant `bookstore-staging`.

<details>
<summary>Gợi ý</summary>

- `kubectl -n bookstore describe resourcequota bookstore-quota` cho `Used` và `Hard`.
- Tài nguyên hiệu dụng của một Pod lấy từ **Pod đang chạy** (đã qua LimitRange), không từ template:
  tổng container chính và sidecar, so với init container thường lớn nhất.
- `maxSurge` dạng phần trăm làm tròn **lên** (bài 63).
- Ghim tạm: `kubectl patch hpa web --type=merge -p '{"spec":{"minReplicas":5}}'`. Phát hành lại không
  đổi nội dung: `kubectl rollout restart deployment/web`.
- Pod bị quota từ chối để lại sự kiện `FailedCreate` có chữ `exceeded quota` trên ReplicaSet.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong; script ghim HPA rồi trả lại</summary>

```bash
. ~/lab-work/11b/ham.sh
cd ~/bookstore
Q=bookstore-quota
kubectl -n "$NS" describe resourcequota "$Q" | tee "$EV/b5-quota.txt"
qh() { jp "resourcequota/$Q" "{.status.hard.$1}"; }
qu() { jp "resourcequota/$Q" "{.status.used.$1}"; }

P="$(web_pod)"
pod_res() {   # $1 requests|limits, $2 cpu|memory -> tài nguyên hiệu dụng của Pod web P, đã chuẩn hóa
  local F=cpu_m S=0 M=0 L V
  [ "$2" = memory ] && F=mem_mi
  for V in $(jp "pod/$P" "{.spec.containers[*].resources.$1.$2}"); do S=$(( S + $($F "$V") )); done
  for L in $(jp "pod/$P" "{range .spec.initContainers[*]}{.restartPolicy}={.resources.$1.$2}{\"\n\"}{end}"); do
    V="$($F "${L#*=}")"
    if [ "${L%%=*}" = 'Always' ]; then S=$(( S + V )); elif [ "$V" -gt "$M" ]; then M="$V"; fi
  done
  [ "$M" -gt "$S" ] && echo "$M" || echo "$S"
}
lr_def() { jp limitrange/bookstore-limits "{.spec.limits[?(@.type==\"Container\")].$1.$2}"; }

MAXR="$(jp hpa/web '{.spec.maxReplicas}')"; CUR="$(jp deployment/web '{.status.replicas}')"
MS="$(jp deployment/web '{.spec.strategy.rollingUpdate.maxSurge}')"; MS="${MS:-25%}"
case "$MS" in *%) SURGE=$(( (MAXR * ${MS%\%} + 99) / 100 )) ;; *) SURGE="$MS" ;; esac
LN="$(kubectl create --dry-run=client -f tests/load/job.yaml -o jsonpath='{.spec.parallelism}')"
echo "web: hien co $CUR Pod, max $MAXR, surge $SURGE (maxSurge=$MS); Pod sinh tai: $LN" | tee "$EV/b5-tinh.txt"

BAD=0
for K in requests:cpu limits:cpu requests:memory limits:memory; do
  T="${K%%:*}"; R="${K##*:}"; F=cpu_m; U=m; [ "$R" = memory ] && { F=mem_mi; U=Mi; }
  H="$(qh "$T\\.$R")"; [ -z "$H" ] && { echo "quota khong gioi han $T.$R"; continue; }
  HV="$($F "$H")"; UV="$($F "$(qu "$T\\.$R")")"; WP="$(pod_res "$T" "$R")"
  LP="$($F "$(kubectl create --dry-run=client -f tests/load/job.yaml -o jsonpath="{.spec.template.spec.containers[0].resources.$T.$R}")")"
  BV="$(jp cronjob/db-backup "{.spec.jobTemplate.spec.template.spec.containers[0].resources.$T.$R}")"
  if [ -z "$BV" ]; then if [ "$T" = requests ]; then BV="$(lr_def defaultRequest "$R")"; else BV="$(lr_def default "$R")"; fi; fi
  BP="$($F "$BV")"
  OTHER=$(( UV - CUR * WP ))
  NEED=$(( OTHER + (MAXR + SURGE) * WP + LN * LP + BP ))
  echo "$T.$R: hard=${HV}${U} | can=${NEED}${U} = khac ${OTHER} + web $((MAXR + SURGE))x${WP} + tai ${LN}x${LP} + backup ${BP}" \
    | tee -a "$EV/b5-tinh.txt"
  [ "$NEED" -le "$HV" ] || { echo "FAIL: $T.$R thieu $(( NEED - HV ))${U}"; BAD=1; }
done
HP="$(qh pods)"
if [ -n "$HP" ]; then
  NEEDP=$(( $(qu pods) - CUR + MAXR + SURGE + LN + 1 ))
  echo "pods: hard=$HP | can=$NEEDP" | tee -a "$EV/b5-tinh.txt"
  [ "$NEEDP" -le "$HP" ] || { echo "FAIL: pods thieu $(( NEEDP - HP ))"; BAD=1; }
fi
[ "$BAD" -eq 0 ] && echo 'PASS: quota du cho web o max cong surge, nguon tai va mot lan backup'

# Diễn tập: ghim HPA ở max rồi rolling update
qx() { kubectl -n "$NS" get events --field-selector reason=FailedCreate \
  -o jsonpath='{range .items[*]}{.count}|{.message}{"\n"}{end}' 2>/dev/null \
  | awk -F'|' '/exceeded quota/ {s += ($1 == "" ? 1 : $1)} END {print s + 0}'; }
X0="$(qx)"
kubectl -n "$NS" patch hpa web --type=merge -p "{\"spec\":{\"minReplicas\":$MAXR}}"
for i in $(seq 1 40); do [ "$(jp deployment/web '{.status.readyReplicas}')" = "$MAXR" ] && break; sleep 5; done
[ "$(jp deployment/web '{.status.readyReplicas}')" = "$MAXR" ] && echo "PASS: web $MAXR/$MAXR Ready khi HPA bi ghim o max"
( for i in $(seq 1 120); do jp "resourcequota/$Q" '{.status.used.pods}'; echo; sleep 2; done ) > "$EV/b5-pods-rollout.txt" &
W=$!
kubectl -n "$NS" rollout restart deployment/web
kubectl -n "$NS" rollout status deployment/web --timeout=300s && echo 'PASS: rolling update o maxReplicas hoan tat'
kill "$W" 2>/dev/null; wait "$W" 2>/dev/null
echo "so Pod cao nhat quota ghi nhan trong rollout: $(grep -v '^$' "$EV/b5-pods-rollout.txt" | sort -n | tail -1)" \
  | tee -a "$EV/b5-tinh.txt"
X1="$(qx)"; echo "su kien exceeded quota: truoc=$X0 sau=$X1"
[ "$X1" -eq "$X0" ] && echo 'PASS: khong Pod nao bi tu choi vi quota trong dien tap'

./scripts/deploy.sh > "$EV/b5-deploy-tra.txt" 2>&1
{ [ "$(jp hpa/web '{.spec.minReplicas}')" = '2' ] && ls docs/adr/*-quota-cho-hpa.md >/dev/null 2>&1; } \
  && echo 'PASS: deploy.sh tra minReplicas ve repo; co ADR quota-cho-hpa'
```

Sau script, HPA thu `web` từ `maxReplicas` về `minReplicas` theo cửa sổ thu hẹp hiện tại — vài phút,
**phụ thuộc cấu hình**. B6 tự chờ điều đó.

</details>

<details>
<summary>Lời giải tham khảo</summary>

Ví dụ tính với ngân sách gợi ý ở [mục 3.6](DU-AN-BOOKSTORE.md#36-ngân-sách-tài-nguyên-và-lập-lịch),
`maxSurge: 25%`, Pod `db-backup` 50m/64Mi xin, 200m/128Mi trần. Số của bạn đọc từ script, không chép
bảng này:

| Phần | Pod | `requests.cpu` | `limits.cpu` | `requests.memory` | `limits.memory` |
| --- | --- | --- | --- | --- | --- |
| `web` ở 5 + surge 2 | 7 | 7 × 60m = 420m | 7 × 250m = 1750m | 7 × 40Mi = 280Mi | 7 × 80Mi = 560Mi |
| `api` | 2 | 100m | 1000m | 128Mi | 512Mi |
| `db` | 1 | 500m | 500m | 512Mi | 512Mi |
| `db-backup` | 1 | 50m | 200m | 64Mi | 128Mi |
| Pod sinh tải | 2 | 100m | 600m | 32Mi | 64Mi |
| **Cộng** | **13** | **1170m** | **4050m** | **1016Mi** | **1776Mi** |

Với lời giải tham khảo của Lab 7b (`pods: "20"`, `requests.cpu: 1500m`, `requests.memory: 1536Mi`,
`limits.cpu: "6"`, `limits.memory: 3Gi`), con số trên **đã nằm trong quota** — không cần sửa, chỉ cần
ghi phép tính vào ADR. Nếu số của bạn vượt, chỉ nâng đúng key bị thiếu trong `spec.hard` của manifest
ResourceQuota ở `k8s/app/policy/`; **giữ nguyên** các key khác của Lab 7b (`services.nodeports`,
`persistentvolumeclaims`, `requests.storage`, `count/jobs.batch`...) và không hạ key nào xuống dưới
giá trị đang có.

```bash
cd ~/bookstore && ./scripts/deploy.sh
kubectl -n bookstore describe resourcequota bookstore-quota
```

Ý chính cho ADR:

1. 25% của 5 là 1.25, làm tròn lên thành 2. Số Pod cao nhất trong diễn tập thường **lớn hơn** 7: bài
   63 nói tổng tài nguyên vượt `replicas + maxSurge` cho tới khi `terminationGracePeriodSeconds` của
   các Pod đang kết thúc hết hạn — và `web` có `preStop` cộng ân hạn từ sprint 1. Quota cần biên cho
   phần đó.
2. HPA quản `replicas` của Deployment; Deployment chia con số đó cho ReplicaSet cũ và mới trong lúc
   rollout. Quota thấy **tổng Pod của mọi ReplicaSet**, không thấy con số của HPA.
3. HPA vẫn ghi `replicas` mới và `AbleToScale=True` — nó chỉ ghi được subresource `scale`. Deployment
   có `replicas` cao hơn số Pod sẵn sàng; ReplicaSet mang condition `ReplicaFailure`; sự kiện
   `FailedCreate ... exceeded quota`. HPA không biết; người vận hành phải đọc sự kiện.
4. Quota là trần chống cạn kiệt cluster (bài 134, *Hạn ngạch và dung lượng cluster*); nới quá rộng
   thì một lỗi cấu hình của `web` lại ăn được phần của `db` và của tenant staging. Biên khoảng 15–20%
   trên con số xấu nhất có kiểm soát là đủ.

</details>

## B6. [BS-11B-06] Diễn tập: tải dồn vào `web`

**Loại:** Diễn tập sự cố — nguồn tải chạy trên `lab-k8s-worker2`.

**Bối cảnh:** giờ cao điểm là vấn đề `P8` của hệ thống cũ ([hiện trạng](DU-AN-BOOKSTORE.md#21-hiện-trạng)).
Đây là lần đầu `web` gặp tải thật kể từ khi có HPA. Đội cần biết HPA phản ứng sau bao lâu, lên tới
đâu, rút về sau bao lâu, và mọi con số `status` của HPA có khớp với cơ chế bài 72 mô tả không.

**Yêu cầu:**

1. Điểm xuất phát: `web` ở `minReplicas`, không còn tải hay patch nào từ ticket trước.
2. Chạy công cụ của B4. Chứng minh HPA mở rộng `web` dưới tải, không bao giờ vượt `maxReplicas`, và
   ghi lý do vào sự kiện.
3. Ở đỉnh tải, kiểm **phép chia** bằng hai con số HPA tự báo trong `status`: mức dùng trung bình mỗi
   Pod chia tổng `requests.cpu` của container chính cộng sidecar phải ra đúng phần trăm HPA báo.
4. Ở đỉnh tải, Pod `web` trải trên cả hai worker (ràng buộc trải đều của sprint 10).
5. Nguồn tải tự kết thúc trước deadline; log cho biết số request thành công.
6. Sau khi tải dừng, HPA thu `web` về `minReplicas`. Thu hẹp tốn **nhiều vòng quan sát hơn** mở rộng.
7. Postmortem `docs/postmortem/11b-dien-tap-tai.md`.

**Ràng buộc:** không sửa HPA, Deployment hay quota trong lúc diễn tập. Không tăng tải quá trần đã gate
ở B4. Không hạ `requests` của `web` để "dễ vượt mục tiêu".

**Đọc lại:**

- Bài [342](../342-hpa-walkthrough-vi.md): [*Tăng tải*](../342-hpa-walkthrough-vi.md#increase-load),
  [*Dừng tạo tải*](../342-hpa-walkthrough-vi.md#stop-load),
  [*Đại lượng*](../342-hpa-walkthrough-vi.md#đại-lượng-quantities).
- Bài [72](../72-horizontal-pod-autoscale-vi.md):
  [*Chi tiết thuật toán*](../72-horizontal-pod-autoscale-vi.md#algorithm-details) — đặc biệt các đoạn
  về Pod chưa sẵn sàng và về giá trị gốc báo qua `status`;
  [*Hỗ trợ metric tài nguyên*](../72-horizontal-pod-autoscale-vi.md#hỗ-trợ-metric-tài-nguyên-support-for-resource-metrics)
  — ghi chú về cộng gộp mọi container;
  [*Cửa sổ ổn định*](../72-horizontal-pod-autoscale-vi.md#cửa-sổ-ổn-định-stabilization-window),
  [*Dung sai*](../72-horizontal-pod-autoscale-vi.md#tolerance).
- Bài [140](../140-topology-spread-constraints-vi.md):
  [*Các hạn chế đã biết*](../140-topology-spread-constraints-vi.md#các-hạn-chế-đã-biết-known-limitations).

**Bàn giao:** `docs/postmortem/11b-dien-tap-tai.md`; `~/lab-evidence/11b/b6-*.txt`.

**Nghiệm thu:**

- `PASS:` `web` ở `minReplicas` trước khi sinh tải; đủ số Pod sinh tải chạy trên `lab-k8s-worker2`.
- `PASS:` HPA mở rộng dưới tải; số bản sao đỉnh ≤ `maxReplicas`; sự kiện ghi lý do `above target`.
- `PASS:` `averageUtilization × requests mỗi Pod ≈ averageValue × 100`, với `requests` gồm cả sidecar.
- `PASS:` ở đỉnh, Pod `web` nằm trên cả hai worker.
- `PASS:` Job sinh tải `Complete` trước deadline; tổng request thành công > 0.
- `PASS:` HPA thu về `minReplicas`; số vòng thu hẹp > số vòng mở rộng.
- `PASS:` không còn object `loadtest`; có postmortem.

**Câu hỏi thiết kế** — trả lời trong `docs/postmortem/11b-dien-tap-tai.md`:

1. Ghi `UP_ROUND`, `DOWN_ROUND` và số bản sao đỉnh. Cơ chế nào trong bài 72 làm thu hẹp chậm? Ở object
   của bạn nó nằm ở đâu?
2. Phép chia ở yêu cầu 3 khớp với mẫu số gồm sidecar hay chỉ `nginx`? Bài 72 nói gì về cộng gộp mức
   dùng của mọi container? Khi nào dự án nên chuyển sang theo riêng container `nginx`?
3. Lần mở rộng thứ hai thường chậm hơn lần đầu một chút. Đoạn nào của *Chi tiết thuật toán* về Pod
   mới khởi động giải thích điều đó?
4. Sau khi thu hẹp, Pod `web` còn lại có trải đều hai worker không? Bài 140 nói gì về điều này?
5. Nếu tải thật vượt cả mức `maxReplicas` gánh được, condition nào báo, và đó là dấu hiệu cần làm gì?

<details>
<summary>Gợi ý</summary>

- Trong lúc tải: `kubectl -n bookstore get hpa web -w` ở terminal khác; cột `TARGETS` là trung bình
  mọi Pod (bài 342).
- `status.currentMetrics[0].resource.current` có cả `averageUtilization` (phần trăm) và
  `averageValue` (mức dùng trung bình mỗi Pod, dạng `36m` hoặc `1`) — mục *Đại lượng* của bài 342.
- Đợi Job xong: `kubectl wait --for=condition=Complete job/loadtest-web`.
- HPA không mở rộng: đi theo thứ tự ở mục 4 của lab — Service, tải có tới không, rồi mới tới HPA.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi đã đọc ticket; script này chính là cuộc diễn tập</summary>

```bash
. ~/lab-work/11b/ham.sh
cd ~/bookstore
MIN="$(jp hpa/web '{.spec.minReplicas}')"; MAX="$(jp hpa/web '{.spec.maxReplicas}')"
TGT="$(jp hpa/web '{.spec.metrics[0].resource.target.averageUtilization}')"

# 0. Điểm xuất phát
for i in $(seq 1 60); do [ "$(web_rep)" = "$MIN" ] && break; echo "$(date +%T) cho web ve min: $(web_rep)"; sleep "$POLL"; done
REP_START="$(web_rep)"
[ "$REP_START" = "$MIN" ] && echo "PASS: web o minReplicas=$MIN truoc khi sinh tai"

# 1. Bật tải
kubectl apply -f tests/load/
kubectl -n "$NS" wait --for=condition=Ready pod -l app.kubernetes.io/component=loadtest --timeout=120s
PAR="$(jp job/loadtest-web '{.spec.parallelism}')"
LN="$(kubectl -n "$NS" get pod -l app.kubernetes.io/component=loadtest \
  -o jsonpath='{range .items[*]}{.spec.nodeName}{"\n"}{end}' | sort | uniq -c)"
echo "$LN" | tee "$EV/b6-nguon-tai.txt"
{ [ "$(echo "$LN" | wc -l)" -eq 1 ] && echo "$LN" | grep -q " $PAR lab-k8s-worker2$"; } \
  && echo "PASS: $PAR Pod sinh tai chay tren lab-k8s-worker2"

# 2. Mở rộng
UP_ROUND=0
for i in $(seq 1 20); do
  echo "$(date +%T) vong=$i replicas=$(web_rep) utilization=$(web_util)%/${TGT}%" | tee -a "$EV/b6-chu-ky-1.txt"
  [ "$(web_rep)" -gt "$REP_START" ] && { UP_ROUND="$i"; break; }
  sleep "$POLL"
done
[ "$UP_ROUND" -gt 0 ] && echo "PASS: HPA mo rong duoi tai o vong $UP_ROUND"
PEAK=0
for i in $(seq 1 6); do
  R="$(web_rep)"; [ "$R" -gt "$PEAK" ] && PEAK="$R"
  echo "$(date +%T) giu tai vong=$i replicas=$R utilization=$(web_util)%" | tee -a "$EV/b6-chu-ky-1.txt"
  sleep "$POLL"
done
U="$(web_util)"; AV="$(jp hpa/web '{.status.currentMetrics[0].resource.current.averageValue}')"
kubectl -n "$NS" get hpa web | tee "$EV/b6-hpa-dinh.txt"
kubectl top pod -n "$NS" -l "$SEL,app.kubernetes.io/component=web" --containers | tee -a "$EV/b6-hpa-dinh.txt"
echo "ScalingLimited o dinh: $(hpa_cond ScalingLimited status) ($(hpa_cond ScalingLimited reason))" | tee -a "$EV/b6-hpa-dinh.txt"
[ "$PEAK" -gt "$REP_START" ] && [ "$PEAK" -le "$MAX" ] && echo "PASS: dinh $PEAK ban sao, khong vuot maxReplicas=$MAX"
kubectl -n "$NS" describe hpa web > "$EV/b6-describe-dinh.txt"
grep -q 'above target' "$EV/b6-describe-dinh.txt" && echo 'PASS: su kien SuccessfulRescale ghi ly do above target'

# 3. Phép chia
REQ="$(web_pod_req_m)"
NG="$(cpu_m "$(jp deployment/web '{.spec.template.spec.containers[?(@.name=="nginx")].resources.requests.cpu}')")"
AVM="$(cpu_m "$AV")"; LHS=$(( ${U:-0} * REQ )); RHS=$(( AVM * 100 )); D=$(( LHS - RHS )); [ "$D" -lt 0 ] && D=$(( -D ))
echo "utilization=${U}% averageValue=${AVM}m requests/Pod=${REQ}m (rieng nginx ${NG}m): ${U}x${REQ}=$LHS ~ ${AVM}x100=$RHS" \
  | tee "$EV/b6-phep-chia.txt"
echo "neu mau so chi la nginx: ${U}x${NG}=$(( ${U:-0} * NG ))" | tee -a "$EV/b6-phep-chia.txt"
{ [ -n "$U" ] && [ "$U" -gt 0 ] && [ "$D" -le $(( REQ + 100 )) ]; } \
  && echo 'PASS: averageUtilization = averageValue / tong requests.cpu cua nginx va sidecar'

# 4. Trải hai worker ở đỉnh
NW="$(kubectl -n "$NS" get pod -l "$SEL,app.kubernetes.io/component=web" --field-selector=status.phase=Running \
  -o jsonpath='{range .items[*]}{.spec.nodeName}{"\n"}{end}' | sort | uniq -c | tee "$EV/b6-trai-node.txt" | wc -l)"
[ "$NW" -ge 2 ] && echo 'PASS: o dinh, Pod web nam tren ca hai worker'

# 5. Tải tự kết thúc
kubectl -n "$NS" wait --for=condition=Complete job/loadtest-web --timeout=600s && echo 'PASS: Job sinh tai tu ket thuc truoc deadline'
kubectl -n "$NS" logs -l app.kubernetes.io/component=loadtest --tail=-1 --prefix > "$EV/b6-log-tai.txt"
OKS="$(grep -o 'ok=[0-9]*' "$EV/b6-log-tai.txt" | awk -F= '{s += $2} END {print s + 0}')"
ERS="$(grep -o 'loi=[0-9]*' "$EV/b6-log-tai.txt" | awk -F= '{s += $2} END {print s + 0}')"
echo "request thanh cong=$OKS loi=$ERS" | tee -a "$EV/b6-log-tai.txt"
[ "$OKS" -gt 0 ] && echo 'PASS: nguon tai da gui request thanh cong toi web'

# 6. Thu hẹp
DOWN_ROUND=0
for i in $(seq 1 60); do
  echo "$(date +%T) thu hep vong=$i replicas=$(web_rep) utilization=$(web_util)%" | tee -a "$EV/b6-chu-ky-1.txt"
  [ "$(web_rep)" -le "$REP_START" ] && { DOWN_ROUND="$i"; break; }
  sleep "$POLL"
done
echo "chu ky 1: UP_ROUND=$UP_ROUND DOWN_ROUND=$DOWN_ROUND PEAK=$PEAK POLL=${POLL}s" | tee "$EV/b6-ket-qua.txt"
{ [ "$DOWN_ROUND" -gt 0 ] && [ "$DOWN_ROUND" -gt "$UP_ROUND" ]; } \
  && echo 'PASS: HPA thu ve minReplicas; thu hep ton nhieu vong hon mo rong'

# 7. Dọn
kubectl -n "$NS" delete -f tests/load/ --ignore-not-found --wait=true
{ [ "$(kubectl -n "$NS" get all,networkpolicy -l app.kubernetes.io/component=loadtest --no-headers 2>/dev/null | wc -l)" -eq 0 ] && \
  test -f docs/postmortem/11b-dien-tap-tai.md; } && echo 'PASS: khong con object loadtest; co postmortem'
```

`UP_ROUND` và `DOWN_ROUND` là số vòng đo trên cluster của bạn, không phải cam kết của Kubernetes —
chúng **phụ thuộc cấu hình** của controller, của metrics-server và của chính tải. Điều lab khẳng định
là **quan hệ** giữa hai con số.

</details>

<details>
<summary>Lời giải tham khảo — ý chính của postmortem</summary>

1. Ví dụ một lần chạy: `UP_ROUND=3`, đỉnh 4–5 bản sao, `DOWN_ROUND≈22` với vòng 15 giây. Bài 72: trước
   khi co giãn, khuyến nghị được ghi lại và controller lấy **khuyến nghị cao nhất trong cửa sổ ổn
   định** — đợt tải vừa qua "giữ" số Pod thêm hết cửa sổ. Ở object của dự án, cửa sổ đó là
   `spec.behavior.scaleDown.stabilizationWindowSeconds` (B2); chiều mở rộng có cửa sổ 0 nên phản ứng
   ngay.
2. Khớp với mẫu số **gồm sidecar**: controller cộng `requests` của mọi container chạy suốt đời Pod,
   và metrics-server cũng đo cả sidecar. Bài 72 cảnh báo: cộng gộp có thể che một container đang quá
   tải trong khi trung bình cả Pod vẫn ổn. `status-writer` gần như không tốn CPU nên hiện tại chỉ làm
   loãng nhẹ; khi sidecar có tải riêng đáng kể (ví dụ một agent ghi log) thì chuyển sang
   `type: ContainerResource` cho `nginx`.
3. *Chi tiết thuật toán*: khi co giãn theo CPU, Pod chưa sẵn sàng hoặc vừa sẵn sàng trong khoảng khởi
   tạo bị để riêng và — khi đang tính mở rộng — được giả định dùng 0%. Pod vừa thêm làm giảm biên độ
   của lần mở rộng kế tiếp.
4. Thường không: bài 140, *Các hạn chế đã biết* — scale down một Deployment có thể làm phân bố mất cân
   bằng; ràng buộc chỉ được xét lúc lập lịch Pod mới.
5. `ScalingLimited=True` lý do `TooManyReplicas`: khuyến nghị bị `maxReplicas` cắt. Phụ lục của bài
   342 gọi đó là dấu hiệu cần nâng trần — kèm nâng quota (B5) và kiểm sức chứa hai worker.

</details>

## B7. [BS-11B-07] `behavior`: thu hẹp có kiểm soát

**Loại:** Tính năng, kèm đo lại.

**Bối cảnh:** B6 cho thấy với cửa sổ 300 giây của bản thiết kế, `web` giữ đỉnh lâu rồi rơi một lần về
`minReplicas`. Đội muốn rút sớm hơn nhưng **từ từ**: mỗi lần chỉ bớt một Pod, để nếu tải quay lại thì
còn đủ bản sao gánh.

**Yêu cầu:**

1. Trong `k8s/app/web/hpa.yaml`: cửa sổ ổn định thu hẹp ngắn hơn bản thiết kế, và một chính sách
   thu hẹp **không bớt quá một Pod** trong mỗi chu kỳ của chính sách. Chiều mở rộng giữ mặc định.
2. Áp bằng `deploy.sh`. Object thể hiện đúng phần bạn khai, và phần `scaleUp` vẫn là mặc định do API
   server điền.
3. Đo lại một chu kỳ tải bằng công cụ B4: thu hẹp tốn **ít vòng hơn** chu kỳ 1 của B6, và không có
   bước nào bớt quá một Pod giữa hai lần quan sát.
4. ⭐ Chứng minh trường `tolerance` của `behavior` được API server của cluster này chấp nhận và giữ lại.
5. ADR `docs/adr/NNNN-hpa-web-behavior.md`.

**Ràng buộc:** không sửa cờ của kube-controller-manager. Không đổi `minReplicas`, `maxReplicas` hay
mục tiêu ở ticket này — chỉ đổi một biến để so được.

**Đọc lại:**

- Bài [72](../72-horizontal-pod-autoscale-vi.md):
  [*Các chính sách co giãn*](../72-horizontal-pod-autoscale-vi.md#các-chính-sách-co-giãn-scaling-policies),
  [*Cửa sổ ổn định*](../72-horizontal-pod-autoscale-vi.md#cửa-sổ-ổn-định-stabilization-window),
  [*Dung sai*](../72-horizontal-pod-autoscale-vi.md#tolerance),
  [*Hành vi mặc định*](../72-horizontal-pod-autoscale-vi.md#hành-vi-mặc-định-default-behavior),
  [*Ví dụ: giới hạn tốc độ thu hẹp*](../72-horizontal-pod-autoscale-vi.md#ví-dụ-giới-hạn-tốc-độ-thu-hẹp-example-limit-scale-down-rate).

**Bàn giao:** `k8s/app/web/hpa.yaml`; `docs/adr/NNNN-hpa-web-behavior.md`; `~/lab-evidence/11b/b7-*.txt`.

**Nghiệm thu:**

- `PASS:` object: cửa sổ thu hẹp ngắn hơn bản thiết kế; một chính sách `Pods` giá trị 1; `scaleUp`
  là mặc định (cửa sổ 0, `selectPolicy: Max`).
- `PASS:` chu kỳ 2 mở rộng được; thu hẹp về `minReplicas`.
- `PASS:` số vòng thu hẹp của chu kỳ 2 < chu kỳ 1.
- `PASS:` không bước nào bớt quá một Pod giữa hai lần quan sát.
- `PASS:` (⭐) server dry-run giữ lại trường `tolerance`.
- `PASS:` có ADR; không còn object `loadtest`.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/NNNN-hpa-web-behavior.md`:

1. Cửa sổ và chính sách bạn chọn là bao nhiêu, vì sao? Từ đỉnh `maxReplicas`, theo cấu hình của bạn
   `web` về `minReplicas` sau khoảng bao lâu sau khi tải dừng?
2. Khi có nhiều chính sách, mặc định chính sách nào được chọn? `selectPolicy: Min` và `Disabled` đổi
   điều đó thế nào? Dự án có cần `Disabled` vào lúc nào không?
3. Rút ngắn cửa sổ thì rủi ro gì quay lại? Bài 72 gọi hiện tượng đó là gì?
4. ⭐ `tolerance` khác cửa sổ ổn định ở đâu — một cái lọc theo **độ lớn** dao động, cái kia theo
   **thời gian**?

<details>
<summary>Gợi ý</summary>

- Khung `behavior.scaleDown` có `stabilizationWindowSeconds`, `policies` (mỗi mục `type`, `value`,
  `periodSeconds`) và `selectPolicy`.
- Không khai gì cho `scaleUp` — API server tự điền.
- Đo chu kỳ 2 giống hệt B6: cùng công cụ, cùng điểm xuất phát `minReplicas`.
- ⭐ `kubectl apply --dry-run=server -o jsonpath=...` trả về object như API server sẽ lưu; trường bị
  tắt bởi feature gate sẽ biến mất khỏi kết quả.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong; script chạy chu kỳ tải thứ hai</summary>

```bash
. ~/lab-work/11b/ham.sh
cd ~/bookstore
./scripts/deploy.sh > "$EV/b7-deploy.txt" 2>&1
SD="$(jp hpa/web '{.spec.behavior.scaleDown.stabilizationWindowSeconds}')"
PT="$(jp hpa/web '{.spec.behavior.scaleDown.policies[*].type}')"
PV="$(jp hpa/web '{.spec.behavior.scaleDown.policies[?(@.type=="Pods")].value}')"
SDP="$(jp hpa/web '{.spec.behavior.scaleDown.selectPolicy}')"
SU="$(jp hpa/web '{.spec.behavior.scaleUp.stabilizationWindowSeconds}')"
SUP="$(jp hpa/web '{.spec.behavior.scaleUp.selectPolicy}')"
echo "scaleDown: window=$SD policies=$PT Pods.value=$PV selectPolicy=$SDP | scaleUp: window=$SU selectPolicy=$SUP" \
  | tee "$EV/b7-behavior.txt"
{ [ -n "$SD" ] && [ "$SD" -lt 300 ] && [ "$PV" = '1' ] && { [ "$PT" = 'Pods' ] || [ "$SDP" = 'Min' ]; } \
  && [ "$SU" = '0' ] && [ "$SUP" = 'Max' ]; } \
  && echo 'PASS: thu hep ngan hon ban thiet ke, moi lan toi da 1 Pod; scaleUp giu mac dinh'

MIN="$(jp hpa/web '{.spec.minReplicas}')"
for i in $(seq 1 60); do [ "$(web_rep)" = "$MIN" ] && break; sleep "$POLL"; done
REP_START="$(web_rep)"
kubectl apply -f tests/load/
kubectl -n "$NS" wait --for=condition=Ready pod -l app.kubernetes.io/component=loadtest --timeout=120s
UP2=0
for i in $(seq 1 20); do
  echo "$(date +%T) vong=$i replicas=$(web_rep) utilization=$(web_util)%" | tee -a "$EV/b7-chu-ky-2.txt"
  [ "$(web_rep)" -gt "$REP_START" ] && { UP2="$i"; break; }
  sleep "$POLL"
done
kubectl -n "$NS" wait --for=condition=Complete job/loadtest-web --timeout=600s
DOWN2=0; : > "$EV/b7-chuoi-thu-hep.txt"
for i in $(seq 1 60); do
  R="$(web_rep)"; echo "$R" >> "$EV/b7-chuoi-thu-hep.txt"
  echo "$(date +%T) thu hep vong=$i replicas=$R" | tee -a "$EV/b7-chu-ky-2.txt"
  [ "$R" -le "$REP_START" ] && { DOWN2="$i"; break; }
  sleep "$POLL"
done
DOWN1="$(sed -n 's/.*DOWN_ROUND=\([0-9]*\).*/\1/p' "$EV/b6-ket-qua.txt")"
echo "chu ky 1: DOWN_ROUND=$DOWN1 | chu ky 2: UP_ROUND=$UP2 DOWN_ROUND=$DOWN2" | tee -a "$EV/b7-behavior.txt"
{ [ "$UP2" -gt 0 ] && [ "$DOWN2" -gt 0 ]; } && echo 'PASS: chu ky 2 mo rong duoc va thu ve minReplicas'
[ "$DOWN2" -lt "${DOWN1:-0}" ] && echo 'PASS: cua so ngan hon thi thu hep ton it vong hon — do bang so'
BUOC="$(awk 'NR > 1 && p - $1 > m {m = p - $1} {p = $1} END {print m + 0}' "$EV/b7-chuoi-thu-hep.txt")"
echo "buoc bot lon nhat giua hai lan quan sat: $BUOC Pod" | tee -a "$EV/b7-behavior.txt"
[ "$BUOC" -le 1 ] && echo 'PASS: khong buoc nao bot qua mot Pod'

kubectl -n "$NS" delete -f tests/load/ --ignore-not-found --wait=true
{ [ "$(kubectl -n "$NS" get all,networkpolicy -l app.kubernetes.io/component=loadtest --no-headers 2>/dev/null | wc -l)" -eq 0 ] && \
  ls docs/adr/*-hpa-web-behavior.md >/dev/null 2>&1; } && echo 'PASS: khong con object loadtest; co ADR hpa-web-behavior'
```

⭐ `tolerance`:

```bash
. ~/lab-work/11b/ham.sh
cd ~/bookstore
TOL="$( { cat k8s/app/web/hpa.yaml; printf '    scaleUp:\n      tolerance: 0.05\n'; } \
  | kubectl apply --dry-run=server -f - -o jsonpath='{.spec.behavior.scaleUp.tolerance}' 2>&1)"
echo "tolerance sau server dry-run: $TOL" | tee "$EV/b7-tolerance.txt"
case "$TOL" in 0.05|50m) echo 'PASS: API server chap nhan va giu truong tolerance' ;; esac
```

Lệnh trên giả định `behavior:` là khối cuối của file và `scaleDown:` thụt hai mức như lời giải; nếu
file của bạn khác, tự ghép một bản thử ở `~/lab-work/11b/`.

</details>

<details>
<summary>Lời giải tham khảo</summary>

```yaml
# k8s/app/web/hpa.yaml — chỉ khối behavior đổi so với B2
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 60
      policies:
      - type: Pods
        value: 1
        periodSeconds: 30
```

```bash
cd ~/bookstore && ./scripts/deploy.sh
kubectl -n bookstore get hpa web -o jsonpath='{.spec.behavior}'; echo
```

Ý chính cho ADR:

1. 60 giây giữ đỉnh sau đợt tải cuối, rồi mỗi 30 giây bớt một Pod: từ 5 về 2 mất khoảng
   `60 + 3 × 30` giây sau khi tải dừng — con số thật **phụ thuộc cấu hình** đồng bộ của controller và
   cửa sổ đo của metrics-server.
2. Mặc định chọn chính sách cho phép **thay đổi lớn nhất**. `Min` chọn chính sách thay đổi ít nhất;
   `Disabled` tắt hẳn chiều đó — hữu ích trong một sự kiện bán hàng khi đội không muốn `web` co lại.
3. Cửa sổ ngắn làm số bản sao bám sát dao động của metric — *thrashing* hay *flapping* (bài 72, *Tính
   ổn định của quy mô workload*). Chính sách một Pod mỗi chu kỳ giảm thiệt hại nếu điều đó xảy ra.
4. ⭐ `tolerance` bỏ qua biến động **nhỏ hơn một ngưỡng** quanh mục tiêu; cửa sổ ổn định nhìn lại các
   khuyến nghị **trong một khoảng thời gian**. Bài 72 ghi trường `tolerance` ở mức beta; dry-run cho
   biết cluster này có phục vụ nó hay không. Giá trị có thể trả về dạng `50m` thay cho `0.05` — đúng
   ký pháp quantity ở mục *Đại lượng* của bài 342.

</details>

## B8. [BS-11B-08] Diễn tập: HPA gặp `kubectl scale`

**Loại:** Diễn tập sự cố — trên hệ thống thật, trả lại bằng `deploy.sh`.

**Bối cảnh:** một người trực đêm thấy trang chậm và `kubectl scale` tay `web` lên. Sáng hôm sau `web` đã
về như cũ, không ai giải thích được vì sao. Đội cần biết chính xác HPA đối xử thế nào với con số do
người ghi vào — và đọc được ba condition của nó trong từng tình huống.

**Yêu cầu:**

1. Scale tay `web` **vượt** `maxReplicas` một bản sao: chứng minh HPA kéo về `maxReplicas` sau vài chu
   kỳ đồng bộ, và sự kiện nói rõ lý do.
2. Lúc không có tải, scale tay `web` lên đúng `maxReplicas`: chứng minh HPA kéo về `minReplicas` theo
   tốc độ của `behavior` ở B7 — từng Pod một.
3. Lúc không có tải, chứng minh `ScalingLimited=True` với lý do là khuyến nghị thấp hơn `minReplicas`,
   trong khi `AbleToScale` và `ScalingActive` vẫn `True`.
4. ⭐ Scale tay `web` về 0: chứng minh HPA ngừng điều chỉnh (`ScalingActive=False`) và **không** tự đưa
   `web` lên lại; kích hoạt lại bằng tay rồi chứng minh HPA hoạt động trở lại.
5. Trả mọi thứ về repo bằng `deploy.sh`. Postmortem `docs/postmortem/11b-hpa-gap-kubectl-scale.md`.

**Ràng buộc:** chỉ `kubectl scale` Deployment `web`, không sửa HPA. Bước ⭐ làm `web` ngừng phục vụ vài
phút — chỉ làm trên cluster lab.

**Đọc lại:**

- Bài [72](../72-horizontal-pod-autoscale-vi.md):
  [*Ngừng kích hoạt ngầm theo kiểu chế độ bảo trì*](../72-horizontal-pod-autoscale-vi.md#ngừng-kích-hoạt-ngầm-theo-kiểu-chế-độ-bảo-trì-implicit-maintenance-mode-deactivation),
  [*Các chính sách co giãn*](../72-horizontal-pod-autoscale-vi.md#các-chính-sách-co-giãn-scaling-policies).
- Bài [342](../342-hpa-walkthrough-vi.md):
  [*Phụ lục: Các status condition*](../342-hpa-walkthrough-vi.md#phụ-lục-các-status-condition-của-horizontal-pod-autoscaler-appendix-horizontal-pod-autoscaler-status-conditions).
- Bài [61](../61-management-vi.md):
  [*Scale ứng dụng của bạn*](../61-management-vi.md#scale-ứng-dụng-của-bạn-scaling-your-application).

**Bàn giao:** `docs/postmortem/11b-hpa-gap-kubectl-scale.md`; `~/lab-evidence/11b/b8-*.txt`.

**Nghiệm thu:**

- `PASS:` scale vượt trần: HPA kéo về `maxReplicas`; sự kiện có lý do `above Spec.MaxReplicas`.
- `PASS:` scale lên trần lúc nhàn: HPA kéo về `minReplicas`, không bước nào bớt quá một Pod.
- `PASS:` lúc nhàn: `ScalingLimited=True` lý do `TooFewReplicas`; `AbleToScale`, `ScalingActive` `True`.
- `PASS:` (⭐) về 0: `ScalingActive=False` lý do `ScalingDisabled`, `web` vẫn ở 0; kích hoạt lại thì
  `ScalingActive=True`.
- `PASS:` `deploy.sh` trả lại; `web` ở `minReplicas`; có postmortem.

**Câu hỏi thiết kế** — trả lời trong postmortem:

1. Vì sao lần kéo từ trên trần về trần nhanh, còn lần kéo từ trần về sàn đi từng bước? Một lần là
   **giới hạn** của HPA, lần kia là **khuyến nghị** từ metric — cái nào đi qua `behavior`?
2. Lab 4a dạy `kubectl scale` là lệnh dứt khoát. Có HPA rồi thì nó là gì? Người trực đêm nên làm gì
   thay vì `kubectl scale`?
3. `ScalingLimited=True` lúc nhàn có phải sự cố không? Ba condition trả lời ba câu hỏi nào?
4. ⭐ Vì sao HPA không tự đưa `web` từ 0 lên `minReplicas`, trong khi lúc tạo mới (B3, câu 3) nó lại đưa
   từ 1 lên? Lab 12 có thể dùng hành vi này ở đâu, và nguy hiểm ở đâu?

<details>
<summary>Gợi ý</summary>

- Lý do mỗi lần đổi số bản sao nằm trong sự kiện `SuccessfulRescale`:
  `kubectl -n bookstore describe hpa web | grep -A 10 '^Events:'`.
- Đọc một condition: `kubectl get hpa web -o jsonpath='{.status.conditions[?(@.type=="ScalingLimited")].reason}'`.
- Bài 72 mô tả điều kiện kích hoạt lại: chỉnh tay số bản sao của đích **hoặc** `minReplicas` của HPA.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi đã đọc ticket; script này chính là cuộc diễn tập</summary>

```bash
. ~/lab-work/11b/ham.sh
cd ~/bookstore
MIN="$(jp hpa/web '{.spec.minReplicas}')"; MAX="$(jp hpa/web '{.spec.maxReplicas}')"
for i in $(seq 1 60); do [ "$(web_rep)" = "$MIN" ] && break; sleep "$POLL"; done

# 1. Vượt trần
kubectl -n "$NS" scale deployment web --replicas=$(( MAX + 1 ))
for i in $(seq 1 12); do
  echo "$(date +%T) vong=$i replicas=$(web_rep)" | tee -a "$EV/b8-vuot-tran.txt"
  [ "$(web_rep)" = "$MAX" ] && break
  sleep 5
done
kubectl -n "$NS" describe hpa web > "$EV/b8-describe-vuot-tran.txt"
{ [ "$(web_rep)" = "$MAX" ] && grep -q 'above Spec.MaxReplicas' "$EV/b8-describe-vuot-tran.txt"; } \
  && echo "PASS: HPA keo $((MAX + 1)) ve maxReplicas=$MAX; su kien ghi ly do above Spec.MaxReplicas"

# 2. Lên trần lúc nhàn, HPA kéo về sàn theo behavior
kubectl -n "$NS" scale deployment web --replicas="$MAX"
: > "$EV/b8-chuoi-ve-san.txt"; BACK=0
for i in $(seq 1 40); do
  R="$(web_rep)"; echo "$R" >> "$EV/b8-chuoi-ve-san.txt"
  echo "$(date +%T) vong=$i replicas=$R"
  [ "$R" = "$MIN" ] && { BACK="$i"; break; }
  sleep "$POLL"
done
BUOC="$(awk 'NR > 1 && p - $1 > m {m = p - $1} {p = $1} END {print m + 0}' "$EV/b8-chuoi-ve-san.txt")"
{ [ "$BACK" -gt 0 ] && [ "$BUOC" -le 1 ]; } \
  && echo "PASS: HPA keo $MAX ve minReplicas=$MIN o vong $BACK, moi buoc toi da 1 Pod"

# 3. Ba condition lúc nhàn — đợi hết chu kỳ của chính sách thu hẹp, nếu không lý do còn là ScaleDownLimit
for i in $(seq 1 8); do [ "$(hpa_cond ScalingLimited reason)" = 'TooFewReplicas' ] && break; sleep "$POLL"; done
{ echo "AbleToScale    $(hpa_cond AbleToScale status) $(hpa_cond AbleToScale reason)"
  echo "ScalingActive  $(hpa_cond ScalingActive status) $(hpa_cond ScalingActive reason)"
  echo "ScalingLimited $(hpa_cond ScalingLimited status) $(hpa_cond ScalingLimited reason)"; } | tee "$EV/b8-condition-nhan.txt"
{ [ "$(hpa_cond ScalingLimited status)" = 'True' ] && [ "$(hpa_cond ScalingLimited reason)" = 'TooFewReplicas' ] && \
  [ "$(hpa_cond AbleToScale status)" = 'True' ] && [ "$(hpa_cond ScalingActive status)" = 'True' ]; } \
  && echo 'PASS: luc nhan ScalingLimited=True TooFewReplicas; AbleToScale, ScalingActive van True'
```

⭐ Chế độ bảo trì ngầm — `web` ngừng phục vụ trong lúc chạy:

```bash
. ~/lab-work/11b/ham.sh
MIN="$(jp hpa/web '{.spec.minReplicas}')"
kubectl -n "$NS" scale deployment web --replicas=0
for i in $(seq 1 12); do [ "$(hpa_cond ScalingActive reason)" = 'ScalingDisabled' ] && break; sleep 5; done
sleep $(( POLL * 3 ))
echo "sau 3 chu ky: replicas=$(web_rep) ScalingActive=$(hpa_cond ScalingActive status) $(hpa_cond ScalingActive reason)" \
  | tee "$EV/b8-bao-tri.txt"
{ [ "$(web_rep)" = '0' ] && [ "$(hpa_cond ScalingActive status)" = 'False' ] && \
  [ "$(hpa_cond ScalingActive reason)" = 'ScalingDisabled' ]; } \
  && echo 'PASS: ve 0 thi HPA ngung dieu chinh va khong tu dua web len'
kubectl -n "$NS" scale deployment web --replicas="$MIN"
for i in $(seq 1 20); do [ "$(hpa_cond ScalingActive status)" = 'True' ] && break; sleep "$POLL"; done
[ "$(hpa_cond ScalingActive status)" = 'True' ] && echo 'PASS: kich hoat lai bang tay thi HPA hoat dong tro lai'
```

Trả lại và đóng ticket:

```bash
. ~/lab-work/11b/ham.sh
cd ~/bookstore
./scripts/deploy.sh > "$EV/b8-deploy-tra.txt" 2>&1
MIN="$(jp hpa/web '{.spec.minReplicas}')"
for i in $(seq 1 40); do [ "$(web_rep)" = "$MIN" ] && [ "$(jp deployment/web '{.status.readyReplicas}')" = "$MIN" ] && break; sleep "$POLL"; done
{ [ "$(web_rep)" = "$MIN" ] && test -f docs/postmortem/11b-hpa-gap-kubectl-scale.md; } \
  && echo 'PASS: web ve minReplicas sau deploy.sh; co postmortem'
```

</details>

<details>
<summary>Lời giải tham khảo — ý chính của postmortem</summary>

1. Số bản sao nằm **ngoài** khoảng `[minReplicas, maxReplicas]` là vi phạm giới hạn: HPA đặt thẳng về
   biên, sự kiện ghi `Current number of replicas above Spec.MaxReplicas` — không có khuyến nghị nào
   để làm mượt. Số bản sao **trong** khoảng thì HPA tính khuyến nghị từ metric, khuyến nghị đó đi qua
   cửa sổ ổn định và chính sách `Pods: 1` của B7 — nên từng bước, sự kiện ghi `All metrics below
   target`.
2. Có HPA, `kubectl scale` chỉ là một con số sống tới chu kỳ đồng bộ kế tiếp. Người trực muốn giữ nhiều
   bản sao hơn thì nâng `minReplicas` của HPA — trong repo, qua `deploy.sh` hoặc CI — và ghi lý do.
3. Không: phụ lục bài 342 — `AbleToScale` hỏi HPA có lấy và cập nhật được `scale` không; `ScalingActive`
   hỏi có tính được số bản sao mong muốn không (`False` thường là lỗi lấy metric); `ScalingLimited`
   hỏi khuyến nghị có bị min hoặc max cắt không. Lúc nhàn, khuyến nghị là 1, bị sàn 2 cắt — đúng thiết
   kế.
4. ⭐ Bài 72, *Ngừng kích hoạt ngầm*: đích có 0 bản sao và `minReplicas` lớn hơn 0 thì HPA ngừng điều
   chỉnh, `ScalingActive=False`, tới khi có người chỉnh tay. Lúc tạo mới, đích có 1 bản sao nên HPA
   vẫn hoạt động và kẹp lên sàn. Lab 12 có thể dùng 0 bản sao để tạm dừng `web` có chủ đích; nguy hiểm
   là quên kích hoạt lại — HPA sẽ không bao giờ tự cứu.

</details>

## B9. [BS-11B-09] Trục dọc: quyết định về VPA

**Loại:** Điều tra — sản phẩm là ADR.

**Bối cảnh:** sprint 3 đã chốt `requests` bằng tay từ số đo trong cgroup, và Lab 3c đã đổi CPU của
`api` tại chỗ — đó là co giãn **dọc thủ công**. Một thành viên đề xuất cài VPA để `requests` tự điều
chỉnh. Nợ lab #1 cũng gồm phần VPA. Đội cần một quyết định có căn cứ, ghi lại để không ai "cài thử cho
xong".

**Yêu cầu:**

1. Chứng minh trên cluster: một `VerticalPodAutoscaler` cho `web` theo đúng mẫu của bài 73 bị API server
   từ chối, và lý do là kind không tồn tại — không phải lỗi quyền hay lỗi cú pháp.
2. ADR `docs/adr/NNNN-vpa.md` gồm: vì sao **không** cài VPA ở sprint này; vì sao bật VPA quản CPU cạnh
   HPA theo `Utilization` của CPU trên cùng `web` là xung đột — lập luận bằng phép chia đã đo ở B6;
   hai lối thoát mà tài liệu để lại; ba điều kiện phải có cùng lúc để trả nốt phần VPA của nợ #1.
3. Ghi rõ trong ADR: phần HPA của nợ #1 đã trả ở ticket nào; phần VPA còn treo.

**Ràng buộc:** không cài CRD, không cài add-on, không kéo manifest VPA từ mạng.

**Đọc lại:**

- Bài [73](../73-vertical-pod-autoscale-vi.md):
  [*Đối tượng API*](../73-vertical-pod-autoscale-vi.md#đối-tượng-api-api-object),
  [*VerticalPodAutoscaler hoạt động như thế nào?*](../73-vertical-pod-autoscale-vi.md#verticalpodautoscaler-hoạt-động-như-thế-nào-how-does-a-verticalpodautoscaler-work),
  [*Các chế độ cập nhật*](../73-vertical-pod-autoscale-vi.md#các-chế-độ-cập-nhật-update-modes),
  [*Off*](../73-vertical-pod-autoscale-vi.md#updateMode-Off),
  [*Recreate*](../73-vertical-pod-autoscale-vi.md#updateMode-Recreate),
  [*Chính sách tài nguyên*](../73-vertical-pod-autoscale-vi.md#chính-sách-tài-nguyên-resource-policies).
- Bài [71](../71-autoscaling-vi.md): [*Co giãn workload thủ công*](../71-autoscaling-vi.md#co-giãn-workload-thủ-công-scaling-workloads-manually),
  [*Co giãn workload theo chiều dọc*](../71-autoscaling-vi.md#scaling-workloads-vertically).
- [Sổ nợ lab](README.md#5-sổ-nợ-lab), nợ #1.

**Bàn giao:** `docs/adr/NNNN-vpa.md`; `~/lab-evidence/11b/b9-*.txt`.

**Nghiệm thu:**

- `PASS:` server dry-run `VerticalPodAutoscaler` cho `web` bị từ chối với lý do không nhận ra kind.
- `PASS:` nhóm API `autoscaling.k8s.io` không có tài nguyên nào.
- `PASS:` có ADR VPA.

Phần lập luận của ADR **không có gate**: trên cluster không có object VPA nào để đo. Nó được kiểm ở
[checkpoint](#3-checkpoint-11b) bằng vấn đáp.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/NNNN-vpa.md`:

1. Kể ba thành phần của VPA và việc của từng cái. Ở `updateMode: Recreate`, thành phần nào **thật sự**
   ghi `requests` mới lên Pod?
2. Bật VPA quản CPU cho `web` trong khi HPA theo `Utilization` của CPU: vòng lặp tự kích sinh ra thế
   nào? Dùng chính phép chia của B6.
3. Hai lối thoát — một bằng `resourcePolicy`, một bằng `updateMode` — là gì, và lối nào an toàn hơn
   cho dự án ở giai đoạn này?
4. Ba điều kiện để trả nốt phần VPA của nợ #1 là gì, và vì sao thiếu một là phải để nợ?

<details>
<summary>Gợi ý</summary>

- Ví dụ YAML đầu tiên của bài 73, mục *Các chế độ cập nhật*, là đủ — đổi tên, namespace, đích.
- `kubectl create --dry-run=server -f <file>` gửi object tới API server mà không lưu.
- Lối thoát thứ nhất: cho hai autoscaler theo **hai tài nguyên khác nhau**.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong</summary>

```bash
. ~/lab-work/11b/ham.sh
cd ~/bookstore
cat > "$WK/vpa-web.yaml" <<'EOF'
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: web
  namespace: bookstore
spec:
  targetRef:
    apiVersion: "apps/v1"
    kind: Deployment
    name: web
  updatePolicy:
    updateMode: "Off"
EOF
if kubectl create --dry-run=server -f "$WK/vpa-web.yaml" > "$EV/b9-vpa.txt" 2>&1; then
  echo 'FAIL: cluster nay dang co VPA — no khong con o 04-metrics-ready'
else
  cat "$EV/b9-vpa.txt"
  grep -q 'no matches for kind' "$EV/b9-vpa.txt" \
    && echo 'PASS: API server tu choi VerticalPodAutoscaler vi khong nhan ra kind'
fi
[ -z "$(kubectl api-resources --api-group=autoscaling.k8s.io -o name 2>/dev/null)" ] \
  && echo 'PASS: nhom API autoscaling.k8s.io khong co tai nguyen nao'
ls docs/adr/*-vpa.md >/dev/null 2>&1 && echo 'PASS: co ADR vpa'
rm -f "$WK/vpa-web.yaml"
```

</details>

<details>
<summary>Lời giải tham khảo — ý chính của ADR</summary>

1. Bài 73: **Recommender** phân tích mức dùng hiện tại và quá khứ, ghi ba mức khuyến nghị (Target, cận
   dưới, cận trên) vào `.status.recommendation`; **Updater** so `requests` đang chạy với khuyến nghị
   rồi evict Pod hoặc sửa tại chỗ; **admission controller** là mutating webhook chặn request tạo Pod
   và áp khuyến nghị Target. Ở `Recreate`, Updater chỉ evict; Deployment tạo Pod thay thế; **admission
   controller** mới là thành phần ghi `requests` mới lên Pod đó.
2. B6 đo: `averageUtilization = averageValue / requests.cpu`. HPA hành động khi phần trăm vượt mục
   tiêu; VPA đổi chính `requests.cpu` — **mẫu số**. Tải không đổi mà mẫu số đổi thì phần trăm nhảy, HPA
   đổi số bản sao; số bản sao đổi làm mức dùng trên mỗi Pod đổi, VPA lại đổi `requests`. Đó là
   *thrashing* mà bài 72 gọi tên ở mục *Tính ổn định của quy mô workload*.
3. (a) `controlledResources: ["memory"]` trong `resourcePolicy` — VPA chỉ quản memory, HPA giữ CPU,
   không còn mẫu số chung. (b) `updateMode: "Off"` — VPA vẫn phân tích, khuyến nghị chỉ nằm trong
   `.status`, người vận hành đọc rồi tự quyết. Với dự án, (b) an toàn hơn: không có gì evict `web`
   hay `db-0`, mà vẫn có số liệu để chốt lại `requests` như sprint 3 đã làm bằng tay.
4. Một: bảng [A1.4](LAB-00-MOI-TRUONG-1.35.7.md#a14-phần-stack-không-thuộc-lab-00) có dòng VPA với
   phiên bản đã chốt, đối chiếu tương thích với baseline A1.3 — như metrics-server ở Lab 11a. Hai: người
   học đã qua [giai đoạn 14](../00-ALO-TRINH-ADMIN.md#giai-đoạn-14--khả-năng-mở-rộng) — VPA là CRD cộng
   controller cộng webhook, cài trước là nhảy cóc. Ba: có một mốc snapshot chứa nó, hoặc lab trả nợ tự
   gỡ sạch trước khi kết thúc — Lab 11b trả cluster về `04-metrics-ready` nên không chứa được. Thiếu
   một điều kiện là vi phạm một nguyên tắc của thư mục lab, nên phần VPA **vẫn là nợ**. Phần HPA đã
   trả ở B2–B8.

</details>

## B10. Đóng sprint: cleanup và gate cuối

**Mục đích:** bàn giao một repo dựng lại được toàn bộ hệ thống của sprint 18 — `web` tự co giãn —
rồi trả cluster về đúng `04-metrics-ready`, chứng minh lab không đụng vào control plane và
metrics-server.

1. Thay `tests/smoke.sh` bằng phiên bản của Lab 11b — kiểm thử khói **tích lũy** mọi hợp đồng còn hiệu
   lực ở [mục 4 của file dự án](DU-AN-BOOKSTORE.md#4-hợp-đồng-kỹ-thuật), chép nguyên văn. Vì `web` không
   còn số bản sao cố định và Deployment mới tạo bắt đầu ở 1 bản sao (B3), script **chờ có giới hạn**
   tới khi HPA đưa `web` lên `minReplicas` và có số đo đầu tiên:

```bash
cat > ~/bookstore/tests/smoke.sh <<'EOF'
#!/usr/bin/env bash
# Kiểm thử khói — trạng thái sau Lab 11b: hệ thống sprint 1–17 cộng HPA web (sprint 18).
# Tích lũy: giữ mọi kiểm tra của smoke Lab 11a; số bản sao web do HPA quyết định nên kiểm theo khoảng
# của HPA thay cho số cố định. Tên ConfigMap đọc từ spec của workload (configMapGenerator gắn hậu tố
# hash từ 5a). Không in mật khẩu.
set -u
cd "$(dirname "$0")/.."
NS=bookstore
SYS=bookstore-system
TNS=bookstore-staging
SEL='app.kubernetes.io/part-of=bookstore'
FAIL=0
pass() { echo "PASS: $*"; }
fail() { echo "FAIL: $*"; FAIL=1; }
k() { kubectl -n "$NS" "$@"; }
jp() { kubectl -n "$NS" get "$1" -o jsonpath="$2" 2>/dev/null; }
lbl() { kubectl get "$1" -o jsonpath="{.metadata.labels.$2}" 2>/dev/null; }
can() { kubectl auth can-i "$@" -n "$NS" 2>/dev/null; }
psa() { kubectl get namespace "$1" -o jsonpath="{.metadata.labels.pod-security\.kubernetes\.io/$2}" 2>/dev/null; }
pst() { kubectl -n "$1" exec "$2" -c "$3" -- sh -c "awk '/^$4:/{print \$2}' /proc/1/status" 2>/dev/null; }
dry_priv() {
  printf '%s\n' 'apiVersion: v1' 'kind: Pod' 'metadata: {name: smoke-dac-quyen}' 'spec:' \
    '  containers: [{name: c, image: busybox:1.37, securityContext: {privileged: true}}]' \
    | kubectl -n "$1" create --dry-run=server -f - 2>&1
}
V="$(kubectl get --raw /version 2>/dev/null)"
PSA_VER="v$(echo "$V" | sed -n 's/.*"major": *"\([0-9]*\).*/\1/p').$(echo "$V" | sed -n 's/.*"minor": *"\([0-9]*\).*/\1/p')"
cond() { jp hpa/web "{.status.conditions[?(@.type==\"$1\")].status}"; }

# 0. HPA cần một chu kỳ để đưa web lên minReplicas và một cửa sổ đo để có số liệu — chờ có giới hạn.
MIN="$(jp hpa/web '{.spec.minReplicas}')"; MAX="$(jp hpa/web '{.spec.maxReplicas}')"
for i in $(seq 1 48); do
  R="$(jp deployment/web '{.spec.replicas}')"; RR="$(jp deployment/web '{.status.readyReplicas}')"
  U="$(jp hpa/web '{.status.currentMetrics[0].resource.current.averageUtilization}')"
  [ -n "$MIN" ] && [ "${R:-0}" -ge "$MIN" ] && [ "${RR:-0}" = "${R:-x}" ] && [ -n "$U" ] && break
  sleep 5
done

# 1. Namespace, label dự án, Pod Security (9b, 11a)
for X in "$NS:restricted" "$TNS:restricted" "$SYS:privileged"; do
  N="${X%%:*}"; L="${X##*:}"
  { [ "$(lbl "namespace/$N" 'app\.kubernetes\.io/part-of')" = 'bookstore' ] && \
    [ "$(lbl "namespace/$N" 'pod-security\.kubernetes\.io/enforce')" = "$L" ]; } \
    && pass "namespace $N, PSS enforce=$L" || fail "namespace $N thieu, thieu label du an hoac PSS khac $L"
done
for N in "$NS" "$TNS"; do
  OK=1
  for M in enforce warn audit; do
    { [ "$(psa "$N" "$M")" = restricted ] && [ "$(psa "$N" "$M-version")" = "$PSA_VER" ]; } || OK=0
  done
  [ "$OK" -eq 1 ] && pass "$N restricted o ca ba che do, version $PSA_VER" || fail "$N sai nhan Pod Security"
  dry_priv "$N" | grep -q "violates PodSecurity \"restricted:$PSA_VER\"" \
    && pass "$N tu choi Pod dac quyen" || fail "$N khong tu choi Pod dac quyen"
done
kubectl label --dry-run=server --overwrite ns "$NS" pod-security.kubernetes.io/enforce-version=latest 2>&1 \
  | grep -q 'violate' && fail 'co Pod dang chay vi pham restricted' || pass 'moi Pod dang chay dat restricted'

# 2. Object cấp cluster, label node (3a, 6a, 7a, 9a)
PRIO="$(kubectl get priorityclass bookstore-critical -o jsonpath='{.value}' 2>/dev/null)"
for PC in bookstore-critical bookstore-batch; do
  [ "$(lbl "priorityclass/$PC" 'app\.kubernetes\.io/part-of')" = bookstore ] \
    && pass "co PriorityClass $PC" || fail "thieu PriorityClass $PC"
done
[ "$(kubectl get storageclass bookstore-retain -o jsonpath='{.reclaimPolicy}' 2>/dev/null)" = 'Retain' ] \
  && pass 'StorageClass bookstore-retain, Retain' || fail 'StorageClass bookstore-retain thieu hoac khong Retain'
[ "$(kubectl get nodes -l bookstore.example.com/data=true -o jsonpath='{.items[*].metadata.name}')" = lab-k8s-worker2 ] \
  && pass 'chi lab-k8s-worker2 mang label du lieu' || fail 'label node bookstore.example.com/data sai'
N_CL="$(kubectl get clusterrole,clusterrolebinding -l "$SEL" --no-headers 2>/dev/null | wc -l)"
[ "$N_CL" -eq 0 ] && pass 'du an khong co ClusterRole, ClusterRoleBinding' || fail "co $N_CL object RBAC cap cluster"

# 2b. HPA web (11b)
[ "$(jp hpa/web '{.metadata.labels.app\.kubernetes\.io/part-of}')" = 'bookstore' ] && pass 'HPA web ton tai, mang label du an' || fail 'thieu HPA web'
[ "$(jp hpa/web '{.spec.scaleTargetRef.kind}/{.spec.scaleTargetRef.name} {.spec.minReplicas}-{.spec.maxReplicas} {.spec.metrics[0].resource.name}/{.spec.metrics[0].resource.target.type}')" = 'Deployment/web 2-5 cpu/Utilization' ] \
  && pass 'HPA web: Deployment/web, 2-5 ban sao, CPU Utilization' || fail 'HPA web sai dich, khoang hoac metric'
{ [ "$(cond AbleToScale)" = 'True' ] && [ "$(cond ScalingActive)" = 'True' ]; } \
  && pass 'HPA AbleToScale=True, ScalingActive=True' || fail "HPA AbleToScale=$(cond AbleToScale) ScalingActive=$(cond ScalingActive)"
[ -n "$(jp hpa/web '{.status.currentMetrics[0].resource.current.averageUtilization}')" ] \
  && pass 'HPA co so do CPU hien tai' || fail 'HPA chua co so do — kiem metrics-server va requests cua web'
LA="$(jp deployment/web '{.metadata.annotations.kubectl\.kubernetes\.io/last-applied-configuration}')"
{ [ -n "$LA" ] && ! echo "$LA" | grep -q '"replicas"'; } \
  && pass 'manifest web khong khai replicas — HPA so huu truong nay' || fail 'manifest web con replicas'
[ "$(k get job,networkpolicy,pod -l app.kubernetes.io/component=loadtest --no-headers 2>/dev/null | wc -l)" -eq 0 ] \
  && pass 'khong con object kiem thu tai' || fail 'con object loadtest tren cluster'

# 3. Cấu hình, bí mật, PDB, hạn mức, trạng thái backup (3b, 3c, 5b, 7b, 9a)
for P in web-config api-config db-init; do
  k get configmap -o name 2>/dev/null | grep -q "^configmap/$P" && pass "co ConfigMap $P" || fail "thieu ConfigMap $P"
done
WL="$(k get deployment,statefulset -o json 2>/dev/null)"
for B in web-config api-config db-init; do
  CM="$(echo "$WL" | grep -o "\"$B\(-[a-z0-9]*\)\?\"" | head -1 | tr -d '"')"
  [ -n "$CM" ] && k get configmap "$CM" >/dev/null 2>&1 && pass "ConfigMap $CM ton tai, duoc workload tham chieu" || fail "thieu ConfigMap $B duoc workload tham chieu"
done
D="$(jp secret/db-credentials '{.data}')"
{ echo "$D" | grep -q '"postgres-password"' && echo "$D" | grep -q '"authenticator-password"'; } \
  && pass 'Secret db-credentials du hai key' || fail 'Secret db-credentials thieu key'
[ "$(jp secret/bookstore-tls '{.type}')" = 'kubernetes.io/tls' ] && pass 'Secret bookstore-tls kieu TLS' || fail 'thieu Secret TLS bookstore-tls'
for R in pdb/web pdb/api resourcequota/bookstore-quota limitrange/bookstore-limits configmap/backup-status; do
  k get "$R" >/dev/null 2>&1 && pass "co $R" || fail "thieu $R"
done

# 4. Controller, vị trí, bảo mật (4a, 4b, 7a, 9b)
[ "$(jp statefulset/db '{.status.readyReplicas}')" = '1' ] && pass 'StatefulSet db 1/1 Ready' || fail 'StatefulSet db chua Ready'
{ [ "$(jp pod/db-0 '{.spec.nodeName}')" = 'lab-k8s-worker2' ] && [ "$(jp pod/db-0 '{.status.qosClass}')" = 'Guaranteed' ] && \
  [ -n "$PRIO" ] && [ "$(jp pod/db-0 '{.spec.priority}')" = "$PRIO" ]; } \
  && pass 'db-0 tren lab-k8s-worker2, Guaranteed, bookstore-critical' || fail 'db-0 sai node, QoS hoac priority'
[ "$(jp pod/db-0 '{.metadata.ownerReferences[0].kind}')" = 'StatefulSet' ] && pass 'db-0 thuoc StatefulSet' || fail 'db-0 khong thuoc StatefulSet'
W="$(jp deployment/api '{.spec.replicas}')"; R="$(jp deployment/api '{.status.readyReplicas}')"
{ [ -n "$W" ] && [ "$W" -ge 2 ] && [ "${R:-0}" = "$W" ]; } && pass "Deployment api $R/$W Ready" || fail "Deployment api ${R:-0}/${W:-?} Ready"
# web: số bản sao do HPA quyết định (11b) — kiểm theo khoảng của HPA.
R="$(jp deployment/web '{.spec.replicas}')"; RR="$(jp deployment/web '{.status.readyReplicas}')"
{ [ -n "$MIN" ] && [ "${R:-0}" -ge "$MIN" ] && [ "${R:-0}" -le "$MAX" ] && [ "${RR:-0}" = "$R" ]; } \
  && pass "Deployment web $RR/$R Ready trong khoang HPA $MIN-$MAX" || fail "Deployment web ${RR:-0}/${R:-0} ngoai khoang HPA hoac chua Ready"
NW="$(k get pod -l "$SEL,app.kubernetes.io/component=web" --field-selector=status.phase=Running \
  -o jsonpath='{range .items[*]}{.spec.nodeName}{"\n"}{end}' 2>/dev/null | sort -u | wc -l)"
[ "$NW" -ge 2 ] && pass 'Pod web trai tren hai worker' || fail 'Pod web chi nam tren mot node'
[ "$(jp pdb/web '{.status.disruptionsAllowed}')" -ge 1 ] 2>/dev/null \
  && pass 'PDB web cho phep it nhat mot gian doan o so ban sao hien tai' || fail 'PDB web khong cho gian doan nao — drain se ket'
ORPHAN="$(kubectl get pods -A -l "$SEL" -o jsonpath='{range .items[*]}{.metadata.name}:{.metadata.ownerReferences[0].kind}{"\n"}{end}' 2>/dev/null | grep -c ':$')"
[ "$ORPHAN" -eq 0 ] && pass 'moi Pod cua du an deu co controller' || fail "$ORPHAN Pod tran"
MISS="$(kubectl get pods -A -l "$SEL" -o jsonpath='{range .items[*]}{range .spec.containers[*]}{.name}={.resources.requests.cpu},{.resources.limits.memory}{"\n"}{end}{end}' 2>/dev/null | grep -E '=,|,$' | sort -u)"
[ -z "$MISS" ] && pass 'moi container co requests va limits' || fail "container thieu tai nguyen: $(echo $MISS)"

# 5. Hardening (9b): db non-root, không capability, rootfs chỉ đọc
[ "$(pst "$NS" db-0 postgres Uid)" = 999 ] && pass 'db chay uid 999' || fail 'db khong chay uid 999'
C="$(pst "$NS" db-0 postgres CapEff)"
[ "$((0x${C:-1}))" -eq 0 ] && pass 'db khong giu capability nao' || fail 'db con capability'
NODROP="$(k get pods -l "$SEL" -o jsonpath='{range .items[*]}{range .spec.containers[*]}{.name}={.securityContext.capabilities.drop}{"\n"}{end}{end}' 2>/dev/null \
          | grep -v 'ALL' | sort -u)"
[ -z "$NODROP" ] && pass 'moi container cua du an drop ALL capability' || fail "container chua drop ALL: $(echo $NODROP)"
RO() { jp "deployment/$1" "{.spec.template.spec.containers[?(@.name==\"$2\")].securityContext.readOnlyRootFilesystem}"; }
{ [ "$(RO web nginx)" = true ] && [ "$(RO api postgrest)" = true ]; } \
  && pass 'nginx, postgrest khai rootfs chi doc' || fail 'nginx hoac postgrest chua rootfs chi doc'
SC='{.spec.jobTemplate.spec.template.spec.securityContext'
{ [ "$(jp cronjob/db-backup "$SC.runAsUser}")" = 999 ] && [ "$(jp cronjob/db-backup "$SC.runAsNonRoot}")" = true ]; } \
  && pass 'db-backup chay uid 999, kubelet kiem non-root' || fail 'db-backup chua non-root'
jp deployment/api '{range .spec.template.spec.securityContext.sysctls[*]}{.name}={.value}{" "}{end}' \
  | grep -q 'net.ipv4.tcp_keepalive_time=60' && pass 'api dat sysctl keepalive' || fail 'api thieu sysctl keepalive'

# 6. Dữ liệu bền (4b, 6a)
# backup-store (WaitForFirstConsumer) chỉ Bound sau lần chạy thử db-backup bên dưới (6a).
[ "$(jp pvc/data-db-0 '{.status.phase}')" = 'Bound' ] && pass 'PVC data-db-0 Bound' || fail 'PVC data-db-0 chua Bound'
[ "$(jp pvc/data-db-0 '{.spec.storageClassName}')" = 'bookstore-retain' ] && pass 'data-db-0 dung bookstore-retain' || fail 'data-db-0 sai StorageClass'
N="$(k exec db-0 -c postgres -- psql -U postgres -d bookstore -tAc 'SELECT count(*) FROM api.books' 2>/dev/null)"
[ "${N:-0}" -ge 8 ] && pass "db co $N cuon sach" || fail "db co ${N:-0} cuon sach, can >= 8"
if k get job db-migrate >/dev/null 2>&1; then
  [ "$(jp job/db-migrate '{.status.conditions[?(@.type=="Complete")].status}')" = 'True' ] \
    && pass 'Job db-migrate Complete' || fail 'Job db-migrate chua Complete'
else
  pass 'Job db-migrate da duoc TTL don, ket qua nam trong db'
fi

# 7. Service, DNS, đường đi của request (5a, 9b)
[ "$(jp service/db '{.spec.clusterIP}')" = 'None' ] && pass 'Service db headless' || fail 'Service db khong headless'
[ "$(jp service/web '{.spec.type}/{.spec.ports[0].port}')" = 'ClusterIP/80' ] && pass 'Service web ClusterIP:80' || fail 'Service web sai'
[ "$(jp service/api '{.spec.ports[0].port}')" = '3000' ] && pass 'Service api cong 3000' || fail 'Service api sai cong'
WEB="$(k get pod -l "$SEL,app.kubernetes.io/component=web" --field-selector=status.phase=Running -o jsonpath='{.items[0].metadata.name}' 2>/dev/null)"
WNODE="$(jp "pod/$WEB" '{.spec.nodeName}')"
get() { k exec "$WEB" -c status-writer -- timeout 10 wget -q -T 5 -O- "$1" 2>/dev/null; }
T="$(get http://api:3000/books | grep -o '"title"' | wc -l)"
[ "$T" -ge 8 ] && pass "web goi api bang ten: $T cuon sach" || fail "web goi api bang ten: $T cuon sach"
get http://127.0.0.1:8080/healthz >/dev/null && pass 'web /healthz' || fail 'web /healthz loi'
get http://127.0.0.1:8080/ | grep -q 'Bookstore' && pass 'web phuc vu trang chu' || fail 'web khong phuc vu trang chu'
get http://127.0.0.1:8080/status.json | grep -Eq "\"pod\" *: *\"$WEB\"" && pass 'status.json dung ten Pod' || fail 'status.json sai hoac thieu'
T="$(get http://127.0.0.1:8080/api/books | grep -o '"title"' | wc -l)"
[ "$T" -ge 8 ] && pass "web /api/books tra $T cuon sach" || fail "web /api/books tra $T cuon sach"
DB_IP="$(jp pod/db-0 '{.status.podIP}')"
k exec "$WEB" -c status-writer -- nslookup "db-0.db.$NS.svc.cluster.local" 2>/dev/null | grep -qwF "$DB_IP" \
  && pass 'ten db-0.db phan giai ve IP cua db-0' || fail 'ten db-0.db khong phan giai dung'
k exec "$WEB" -c nginx -- sh -c 'touch /thu-ghi' >/dev/null 2>&1 \
  && fail 'nginx ghi duoc root filesystem' || pass 'root filesystem cua nginx chi doc'

# 8. Ingress HTTPS qua Traefik (5b)
{ [ "$(jp ingress/bookstore '{.spec.rules[0].host}')" = 'bookstore.lab.local' ] && \
  [ "$(jp ingress/bookstore '{.spec.tls[0].secretName}')" = 'bookstore-tls' ]; } \
  && pass 'Ingress bookstore dung host va TLS' || fail 'Ingress bookstore sai host hoac TLS'
NP="$(kubectl -n traefik get svc traefik -o jsonpath='{.spec.ports[?(@.name=="websecure")].nodePort}' 2>/dev/null)"
IP="$(kubectl get node lab-k8s-worker1 -o jsonpath='{.status.addresses[?(@.type=="InternalIP")].address}')"
if [ -n "$NP" ] && curl -ks --max-time 5 --resolve "bookstore.lab.local:$NP:$IP" "https://bookstore.lab.local:$NP/" | grep -q 'Bookstore'; then
  pass "HTTPS qua Traefik (NodePort $NP) tra trang chu"
else
  fail 'khong vao duoc trang chu qua Ingress HTTPS'
fi

# 9. Chính sách mạng (5b, 9a, 11a)
for N in "$NS" "$SYS"; do
  P="$(kubectl -n "$N" get networkpolicy -o jsonpath='{range .items[*]}{.spec.podSelector}|{.spec.policyTypes}{"\n"}{end}' 2>/dev/null | grep '^{}|')"
  { echo "$P" | grep -q 'Ingress' && echo "$P" | grep -q 'Egress'; } \
    && pass "moi Pod trong $N bi co lap hai chieu" || fail "namespace $N thieu policy co lap hai chieu"
done
# web không tới được API server. Đối chứng dương cùng đường đi: Pod busybox ở namespace thử
# bookstore-thu (không NetworkPolicy; chỉ xóa nếu smoke tự tạo) gọi được đúng URL đó bằng đúng lệnh đó.
# Phủ định chỉ tính khi wget hết giờ hoặc bị từ chối kết nối; lỗi khác là "không kết luận được".
WGET_API='timeout 10 wget -q -T 5 -O /dev/null --no-check-certificate https://kubernetes.default.svc/version'
PNS=bookstore-thu; MADE=0
if ! kubectl get namespace "$PNS" >/dev/null 2>&1; then
  kubectl create namespace "$PNS" >/dev/null 2>&1 \
    && kubectl label namespace "$PNS" app.kubernetes.io/part-of=bookstore >/dev/null 2>&1 && MADE=1
fi
CTRL="$(kubectl -n "$PNS" run "smoke-api-$(date +%s)" --rm -i --restart=Never --image=busybox:1.37 --command -- \
        sh -c "$WGET_API && echo TOI-DUOC || echo KHONG-TOI" 2>/dev/null)"
[ "$MADE" -eq 1 ] && kubectl delete namespace "$PNS" --ignore-not-found --wait=true --timeout=120s >/dev/null 2>&1
OUT="$(k exec "$WEB" -c status-writer -- sh -c "$WGET_API" 2>&1)"; RC=$?
if ! echo "$CTRL" | grep -q 'TOI-DUOC'; then fail 'doi chung: Pod ngoai NetworkPolicy khong goi duoc API server — khong ket luan duoc'
elif [ "$RC" -eq 0 ]; then fail 'web toi duoc API server'
elif echo "$OUT" | grep -Eq 'timed out|Connection refused|exit code 143'; then pass 'web bi NetworkPolicy chan toi API server'
else fail "web toi API server loi khac chan mang — khong ket luan duoc: $(echo $OUT)"; fi

# 10. CronJob db-backup và backup-status (4b, 6a, 7a, 9a)
[ "$(jp cronjob/db-backup '{.spec.concurrencyPolicy}')" = Forbid ] && pass 'db-backup Forbid' || fail 'db-backup khong Forbid'
SDS="$(jp cronjob/db-backup '{.spec.startingDeadlineSeconds}')"
[ "${SDS:-0}" -ge 10 ] && pass "db-backup startingDeadlineSeconds=$SDS" || fail 'db-backup thieu startingDeadlineSeconds'
[ "$(jp cronjob/db-backup '{.spec.suspend}')" != true ] && pass 'db-backup khong bi tam ngung' || fail 'db-backup dang suspend'
[ "$(jp cronjob/db-backup '{.spec.jobTemplate.spec.template.spec.priorityClassName}')" = bookstore-batch ] \
  && pass 'db-backup dung bookstore-batch' || fail 'db-backup khong dung bookstore-batch'
J="smoke-backup-$(date +%s)"
if k create job "$J" --from=cronjob/db-backup >/dev/null 2>&1 \
   && k wait --for=condition=Complete "job/$J" --timeout=300s >/dev/null 2>&1; then
  LINE="$(k logs "job/$J" 2>/dev/null | grep '^db-backup: OK' | tail -1)"
  F="$(echo "$LINE" | awk '{print $3}')"; B="$(echo "$LINE" | awk '{print $4}')"
  [ -n "$LINE" ] && pass "chay thu db-backup: $F ($B byte)" || fail 'chay thu db-backup khong in db-backup: OK'
  { [ -n "$F" ] && [ "$(jp configmap/backup-status '{.data.file}')" = "$F" ] && \
    [ "$(jp configmap/backup-status '{.data.bytes}')" = "$B" ]; } \
    && pass 'backup-status ghi dung lan chay vua roi' || fail 'backup-status khong khop lan chay'
else
  fail 'chay thu db-backup that bai'
fi
k delete job "$J" --ignore-not-found --wait=true >/dev/null 2>&1
[ "$(jp pvc/backup-store '{.status.phase}')" = 'Bound' ] && pass 'PVC backup-store Bound (sau lan chay thu)' || fail 'PVC backup-store chua Bound'

# 11. Danh tính và quyền (9a, 9b); ServiceAccount node-agent chuyển sang bookstore-system (11a)
BAD=0
for X in deployment/web:web deployment/api:api statefulset/db:db; do
  [ "$(jp "${X%%:*}" '{.spec.template.spec.serviceAccountName}')" = "${X##*:}" ] || { echo "FAIL: ${X%%:*} sai ServiceAccount"; BAD=1; }
done
[ "$(jp cronjob/db-backup '{.spec.jobTemplate.spec.template.spec.serviceAccountName}')" = db-backup ] \
  || { echo 'FAIL: db-backup sai ServiceAccount'; BAD=1; }
for SA in default web api db db-migrate db-backup ci-deployer; do
  [ "$(jp "serviceaccount/$SA" '{.automountServiceAccountToken}')" = false ] \
    || { echo "FAIL: ServiceAccount $SA thieu hoac tu mount token"; BAD=1; }
done
{ [ "$(kubectl -n "$SYS" get daemonset node-agent -o jsonpath='{.spec.template.spec.serviceAccountName}' 2>/dev/null)" = node-agent ] && \
  [ "$(kubectl -n "$SYS" get serviceaccount node-agent -o jsonpath='{.automountServiceAccountToken}' 2>/dev/null)" = false ]; } \
  || { echo 'FAIL: node-agent khong dung ServiceAccount node-agent khong token o bookstore-system'; BAD=1; }
[ "$BAD" -eq 0 ] && pass 'moi thanh phan mot ServiceAccount, khong ai tu mount token' || FAIL=1
TOKP="$(kubectl get pods -A -l "$SEL" -o jsonpath='{range .items[*]}{range .spec.volumes[*]}{.name}{"\n"}{end}{end}' 2>/dev/null | grep -c '^kube-api-access-')"
[ "$TOKP" -eq 0 ] && pass 'khong Pod nao cua du an mang token tu dong' || fail "$TOKP volume token tu dong"
B=system:serviceaccount:bookstore:db-backup
{ [ "$(can patch configmaps/backup-status --as="$B")" = yes ] && [ "$(can list configmaps --as="$B")" = no ] && \
  [ "$(can get secrets --as="$B")" = no ]; } && pass 'db-backup chi sua duoc backup-status' || fail 'quyen db-backup sai'
CI=system:serviceaccount:bookstore:ci-deployer
{ [ "$(can patch deployments --as="$CI")" = yes ] && [ "$(can get secrets --as="$CI")" = no ] && \
  [ "$(can create pods --subresource=exec --as="$CI")" = no ] && [ "$(can patch namespaces/bookstore --as="$CI")" = no ]; } \
  && pass 'ci-deployer sua Deployment, khong Secret, exec hay Namespace' || fail 'quyen ci-deployer sai'
{ [ "$(can list pods --as=dev --as-group=bookstore-devs)" = yes ] && [ "$(can get secrets --as=dev --as-group=bookstore-devs)" = no ] && \
  [ "$(can patch pods --subresource=ephemeralcontainers --as=dev --as-group=bookstore-devs)" = no ]; } \
  && pass 'bookstore-devs chi doc' || fail 'quyen bookstore-devs sai'
{ [ "$(can patch pods --subresource=ephemeralcontainers --as=truc --as-group=bookstore-oncall)" = yes ] && \
  [ "$(can get secrets --as=truc --as-group=bookstore-oncall)" = no ]; } \
  && pass 'bookstore-oncall go loi duoc, khong doc Secret' || fail 'quyen bookstore-oncall sai'

# 12. Tenant bookstore-staging (9b)
{ kubectl -n "$TNS" get resourcequota staging-quota >/dev/null 2>&1 && \
  kubectl -n "$TNS" get limitrange staging-limits >/dev/null 2>&1; } \
  && pass 'staging co quota va LimitRange' || fail 'staging thieu quota hoac LimitRange'
kubectl -n "$TNS" get networkpolicy -o jsonpath='{range .items[*]}{.spec.podSelector}|{.spec.policyTypes}{"\n"}{end}' 2>/dev/null \
  | grep -q '^{}|.*Ingress.*Egress' && pass 'staging co default deny hai chieu' || fail 'staging thieu default deny'
SW="$(kubectl -n "$TNS" get pod -l "$SEL,app.kubernetes.io/component=web" --field-selector=status.phase=Running \
        -o jsonpath='{.items[0].metadata.name}' 2>/dev/null)"
kubectl -n "$TNS" exec "$SW" -- wget -qO- -T 5 http://web/ 2>/dev/null | grep -q 'Bookstore staging' \
  && pass 'staging phuc vu trang qua Service cua chinh no' || fail 'staging khong phuc vu duoc trang'
kubectl -n "$TNS" exec "$SW" -- wget -qO- -T 3 http://api.bookstore.svc.cluster.local:3000/books >/dev/null 2>&1 \
  && fail 'staging goi duoc api cua bookstore' || pass 'staging bi chan khi goi bookstore'
DEV='--as=dev --as-group=bookstore-devs'
{ [ "$(kubectl auth can-i create deployments -n "$TNS" $DEV 2>/dev/null)" = yes ] && \
  [ "$(kubectl auth can-i get secrets -n "$TNS" $DEV 2>/dev/null)" = no ] && \
  [ "$(kubectl auth can-i patch "namespace/$TNS" $DEV 2>/dev/null)" = no ] && \
  [ "$(can create deployments $DEV)" = no ]; } \
  && pass 'bookstore-devs trien khai duoc o staging; khong Secret, Namespace, bookstore' || fail 'quyen staging sai'

# 13. node-agent ở bookstore-system (4b, 5a, 11a)
NODES="$(kubectl get nodes --no-headers | wc -l)"
R="$(kubectl -n "$SYS" get daemonset node-agent -o jsonpath='{.status.numberReady}' 2>/dev/null)"
[ "${R:-0}" -eq "$NODES" ] && pass "node-agent Ready tren $R/$NODES node" || fail "node-agent ${R:-0}/$NODES node"
[ "$(k get daemonset,service,serviceaccount --field-selector metadata.name=node-agent --no-headers 2>/dev/null | wc -l)" -eq 0 ] \
  && pass 'khong con node-agent trong bookstore, ke ca ServiceAccount' || fail 'con node-agent trong bookstore'
[ "$(kubectl -n "$SYS" get svc node-agent -o jsonpath='{.spec.internalTrafficPolicy}/{.spec.ports[0].port}' 2>/dev/null)" = 'Local/9100' ] \
  && pass 'Service node-agent Local, cong 9100' || fail 'Service node-agent sai'
get "http://node-agent.$SYS.svc.cluster.local:9100/node.json" | grep -Eq "\"node\" *: *\"$WNODE\"" \
  && pass "web doc /node.json cua agent cung node ($WNODE)" || fail 'web khong doc duoc agent cung node'
AS="$(kubectl get service -A -l "$SEL,app.kubernetes.io/component=agent" -o jsonpath='{.items[0].metadata.namespace}/{.items[0].metadata.name}' 2>/dev/null)"
{ [ "$(kubectl -n "${AS%/*}" get service "${AS#*/}" -o jsonpath='{.spec.internalTrafficPolicy}' 2>/dev/null)" = 'Local' ] && \
  get "http://${AS#*/}.${AS%/*}.svc.cluster.local:9100/node.json" | grep -Eq "\"node\" *: *\"$WNODE\""; } \
  && pass 'node-agent cung node tra /node.json qua Service Local tim theo label' || fail 'node-agent qua Service Local loi'
BAD=0
for L in $(kubectl -n "$SYS" get pod -l "$SEL,app.kubernetes.io/component=agent" -o jsonpath='{range .items[*]}{.spec.nodeName}={.status.podIP}{"\n"}{end}' 2>/dev/null); do
  get "http://${L##*=}:9100/node.json" | grep -Eq "\"node\" *: *\"${L%%=*}\"" || { echo "FAIL: node-agent tren ${L%%=*} sai /node.json"; BAD=1; }
done
[ "$BAD" -eq 0 ] && pass 'moi node-agent tra /node.json dung ten node' || FAIL=1
LOGS=''
for i in $(seq 1 12); do
  LOGS="$(get "http://node-agent.$SYS.svc.cluster.local:9100/logs.txt")"
  echo "$LOGS" | grep -q "^last /var/log/pods/${NS}_${WEB}_" && break
  sleep 5
done
{ echo "$LOGS" | tail -n 1 | grep -q "^node=$WNODE files=[1-9]" && \
  echo "$LOGS" | grep -q "^file [0-9]* /var/log/pods/${NS}_${WEB}_" && \
  echo "$LOGS" | grep -q "^last /var/log/pods/${NS}_${WEB}_"; } \
  && pass "/logs.txt cua agent tren $WNODE liet ke log cua $WEB" || fail '/logs.txt thieu hoac sai'

# 14. Metric (11a)
[ "$(kubectl get apiservice v1beta1.metrics.k8s.io -o jsonpath='{.status.conditions[?(@.type=="Available")].status}' 2>/dev/null)" = 'True' ] \
  && pass 'APIService v1beta1.metrics.k8s.io Available' || fail 'Metrics API chua Available'
[ "$(kubectl top node --no-headers 2>/dev/null | wc -l)" -eq "$NODES" ] && pass 'kubectl top node du moi node' || fail 'kubectl top node thieu node'
TOP=''
for i in $(seq 1 18); do
  TOP="$(kubectl top pod -n "$NS" --no-headers 2>/dev/null)"
  echo "$TOP" | grep -q '^db-0 ' && break
  sleep 5
done
echo "$TOP" | grep -q '^db-0 ' && pass 'kubectl top pod co db-0' || fail 'kubectl top pod chua co db-0'
DBN="$(jp pod/db-0 '{.spec.nodeName}')"
kubectl get --raw "/api/v1/nodes/$DBN/proxy/metrics/resource" 2>/dev/null | grep '^container_memory_working_set_bytes{' \
  | grep 'container="postgres"' | grep -q 'pod="db-0"' \
  && pass 'kubelet /metrics/resource co bo nho cua postgres' || fail 'kubelet /metrics/resource thieu db-0'
API="$(k get pod -l "$SEL,app.kubernetes.io/component=api" --field-selector=status.phase=Running -o jsonpath='{.items[0].metadata.name}' 2>/dev/null)"
kubectl -n "$NS" port-forward "pod/$API" 13001:3001 >/dev/null 2>&1 & PF=$!
M=''
for i in $(seq 1 20); do M="$(curl -fsS http://127.0.0.1:13001/metrics 2>/dev/null)" && [ -n "$M" ] && break; sleep 1; done
kill "$PF" 2>/dev/null; wait "$PF" 2>/dev/null
echo "$M" | grep -q '^pgrst_db_pool_max' && pass 'PostgREST /metrics doc duoc qua port-forward' || fail 'khong doc duoc /metrics cua PostgREST'

# 15. Repo và bí mật (3b, 9a)
git check-ignore -q secrets/ci-deployer.kubeconfig \
  && pass 'kubeconfig CI nam ngoai git' || fail 'secrets/ khong bi .gitignore chan'
PGPW="$(k get secret db-credentials -o go-template='{{index .data "postgres-password"}}' 2>/dev/null | base64 -d 2>/dev/null)"
if [ -n "$PGPW" ]; then
  git grep --untracked -qF -e "$PGPW" && fail 'mat khau nam trong file se vao git' || pass 'repo khong chua mat khau'
  kubectl get pods -A -l "$SEL" -o yaml 2>/dev/null | grep -qF -e "$PGPW" \
    && fail 'mat khau xuat hien trong spec Pod' || pass 'spec Pod khong chua mat khau'
else
  fail 'khong doc duoc Secret db-credentials'
fi

exit "$FAIL"
EOF
chmod +x ~/bookstore/tests/smoke.sh
```

2. `docs/tech-debt.md`: sprint này không trả và không phát sinh mã `TD` nào — không sửa bảng. Tình trạng
   nợ lab #1 nằm trong ADR VPA của B9.

3. Dựng lại sạch từ repo và chạy kiểm thử khói:

```bash
cd ~/bookstore
./scripts/teardown.sh && ./scripts/deploy.sh > ~/lab-evidence/11b/b10-deploy.txt 2>&1
./tests/smoke.sh > ~/lab-evidence/11b/b10-smoke.txt 2>&1; echo "smoke exit=$?"
cat ~/lab-evidence/11b/b10-smoke.txt
```

4. Commit, tag, đóng gói, chép ra ngoài VM theo
   [mục 5.4 của file dự án](DU-AN-BOOKSTORE.md#54-sao-lưu-repo-ra-ngoài-vm):

```bash
cd ~/bookstore
git status --short
git add -A && git commit -m 'lab-11b: sprint 18 — HPA web, bo replicas, kiem thu tai, quota'
git tag lab-11b
git bundle create ~/bookstore-lab-11b.bundle --all && git bundle verify ~/bookstore-lab-11b.bundle
```

Chép `~/bookstore-lab-11b.bundle` về máy host bằng `scp` trước khi đi tiếp.

5. Dọn object thử nghiệm và trả cluster:

```bash
kubectl -n bookstore delete -f ~/bookstore/tests/load/ --ignore-not-found --wait=true
kubectl -n bookstore delete deployment,pod -l '!app.kubernetes.io/part-of' --ignore-not-found --wait=true
rm -rf ~/lab-work/11b
cd ~/bookstore && ./scripts/teardown.sh
```

6. Gate cuối:

```bash
cd ~/bookstore
EV=~/lab-evidence/11b
test -z "$(git status --porcelain)" && echo 'PASS: repo khong con thay doi chua commit'
git describe --tags --exact-match HEAD | grep -qx 'lab-11b' && echo 'PASS: HEAD mang tag lab-11b'
test -s ~/bookstore-lab-11b.bundle && echo 'PASS: co bundle'
RENDER_WEB="$(kubectl kustomize --load-restrictor=LoadRestrictionsNone k8s/app \
  | awk 'BEGIN{RS="\n---\n"} /\nkind: Deployment\n/ && /\n  name: web\n/ {print}')"
{ test -f k8s/app/web/hpa.yaml && grep -q 'web/hpa.yaml' k8s/app/kustomization.yaml && test ! -e docs/hpa-web.draft.yaml \
  && test -f tests/load/job.yaml && test -f tests/load/networkpolicy.yaml \
  && [ -n "$RENDER_WEB" ] && ! echo "$RENDER_WEB" | grep -q '^  replicas:'; } && echo 'PASS: bo cuc repo dung muc 5.5 cong tests/load'
BAD=0
for F in 'adr/*-hpa-web-production.md' 'adr/*-web-bo-replicas.md' 'adr/*-kiem-thu-tai.md' 'adr/*-quota-cho-hpa.md' \
         'adr/*-hpa-web-behavior.md' 'adr/*-vpa.md' 'postmortem/11b-dien-tap-tai.md' 'postmortem/11b-hpa-gap-kubectl-scale.md'; do
  ls docs/$F >/dev/null 2>&1 || { echo "FAIL: thieu docs/$F"; BAD=1; }
done
[ "$BAD" -eq 0 ] && grep -l 'bị thay bởi' docs/adr/*-hpa-web.md >/dev/null 2>&1 && echo 'PASS: du ADR va postmortem cua sprint 18'
{ [ "$(kubectl get namespace,priorityclass,storageclass -l app.kubernetes.io/part-of=bookstore --no-headers 2>/dev/null | wc -l)" -eq 0 ] && \
  [ "$(kubectl get all,hpa -A -l app.kubernetes.io/part-of=bookstore --no-headers 2>/dev/null | wc -l)" -eq 0 ] && \
  [ "$(kubectl get hpa -A --no-headers 2>/dev/null | wc -l)" -eq 0 ]; } && echo 'PASS: khong con object nao cua du an, khong con HPA nao'
test ! -e ~/lab-work/11b && echo 'PASS: da don lab-work'

{
  echo "kcm $(sudo sha256sum /etc/kubernetes/manifests/kube-controller-manager.yaml | awk '{print $1}')"
  echo "ms-image $(kubectl -n kube-system get deployment metrics-server -o jsonpath='{.spec.template.spec.containers[0].image}')"
  kubectl -n kube-system get deployment metrics-server \
    -o jsonpath='{range .spec.template.spec.containers[0].args[*]}ms-arg {@}{"\n"}{end}'
} > "$EV/b10-khong-dung-vao.txt"
diff -u "$EV/b0-khong-dung-vao.txt" "$EV/b10-khong-dung-vao.txt" \
  && echo 'PASS: kube-controller-manager va metrics-server khong doi mot ky tu nao'
[ "$(kubectl top node --no-headers 2>/dev/null | wc -l)" -eq 3 ] && echo 'PASS: kubectl top node van tra du ba node'

kubectl wait --for=condition=Ready node --all --timeout=180s
test "$(kubectl get --raw='/readyz')" = 'ok' && echo 'PASS: readyz ok'
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded
kubectl -n kube-system get deployment coredns
kubectl get pods -n default
```

**PASS:** `smoke exit=0` và file smoke ở bước 3 in toàn `PASS:`; chín dòng `PASS:` của gate cuối xuất
hiện, không có dòng `FAIL:`; dòng `PASS: readyz ok`; ba node `Ready`; lệnh field selector trả
`No resources found`; CoreDNS đủ replica; namespace `default` không có Pod. Thư mục
`~/lab-evidence/11b/` **giữ lại**. Cluster trở về `04-metrics-ready`; **không tạo snapshot mới**.

---

## 3. Checkpoint 11b

Không đánh dấu lab hoàn tất chỉ vì script nghiệm thu PASS. Tự trả lời, không nhìn tài liệu:

- [ ] Ở Lab 4b, HPA `web` "không chạy được"; ở sprint này nó chạy. Thứ gì trên cluster đổi, thứ gì
      trong API vẫn y nguyên? Apply một `VerticalPodAutoscaler` hỏng theo kiểu khác thế nào?
- [ ] Pod `web` có `nginx` và sidecar `status-writer`. `averageUtilization` của HPA chia cho cái gì?
      Bạn chứng minh bằng hai con số nào trong `status`? Khi nào nên chuyển sang theo riêng `nginx`?
- [ ] HPA mục tiêu X%, 3 bản sao dùng trung bình 3X%. Công thức cho bao nhiêu, bị cắt còn bao nhiêu,
      condition nào báo? Nếu chỉ vượt mục tiêu 5% thì sao?
- [ ] Kể vòng giành giật giữa `deploy.sh` và HPA bạn đo được ở B3. Vì sao gỡ thẳng dòng `replicas`
      rồi triển khai là chưa đủ, và hai bước bạn đã làm theo thứ tự nào?
- [ ] Sau teardown và deploy sạch, `web` có mấy bản sao trong vài giây đầu, ai đưa nó lên 2, và
      `smoke.sh` của sprint này xử lý điều đó thế nào?
- [ ] Quota của namespace phải đủ cho những gì khi HPA vào? Vì sao số Pod cao nhất trong diễn tập B5
      có thể vượt `maxReplicas + surge`? Khi quota thiếu, HPA có biết không?
- [ ] PDB `web` phải viết thế nào để Lab 12 drain được node khi HPA giữ `web` ở 2 bản sao? Vì sao bài
      339 khuyến nghị `maxUnavailable`?
- [ ] Công cụ kiểm thử tải: ba lớp phanh; vì sao cần hai NetworkPolicy; vì sao chạy trên
      `lab-k8s-worker2` với `bookstore-batch`; vì sao nằm ở `tests/` chứ không ở `k8s/`?
- [ ] Vì sao HPA mở rộng nhanh, thu hẹp chậm? Cơ chế tên gì, ở object của bạn nó nằm ở trường nào,
      và chu kỳ đồng bộ của controller trên cluster bạn đọc ở đâu?
- [ ] Bạn chỉ khai `behavior.scaleDown` mà object có cả `scaleUp` — vì sao? Chính sách `Pods: 1` bạn
      quan sát được bằng chuỗi số nào?
- [ ] `kubectl scale` lên `maxReplicas + 1` và lên `maxReplicas` lúc nhàn: vì sao một lần về nhanh, một
      lần về từng bước? `ScalingLimited=True` lý do `TooFewReplicas` lúc nhàn có phải sự cố không?
- [ ] VPA: ba thành phần; ở `Recreate` ai ghi `requests` mới; vì sao HPA và VPA cùng theo CPU là xung
      đột; hai lối thoát; ba điều kiện để trả nốt nợ #1.

Nợ còn mở sau lab này — nói được **vì sao chưa trả**:

- Nợ lab #1, **phần VPA**: add-on không có trong A1.4, là CRD cộng controller của giai đoạn 14, và lab
  này không tạo mốc snapshot để chứa nó. **Phần HPA đã trả** ở B2–B8.
- Nợ lab #5, #6, #7, #8 vẫn ở đúng chỗ của [sổ nợ lab](README.md#5-sổ-nợ-lab); sprint này không chạm.
- Nợ kỹ thuật: sprint này không trả, không phát sinh mã `TD` nào; `TD-10` vẫn chấp nhận có điều kiện
  (Lab 6b).

### Bài giải thích cuối cùng

Trong vài phút, kể lại một giờ cao điểm của Bookstore sau sprint 18: tải từ Traefik dồn vào Service
`web`; ai đo mức dùng CPU của từng container và số đo đó đi qua API nào; controller nào đọc nó, theo
chu kỳ hay liên tục; phép chia nó làm và mẫu số gồm những container nào; nó ghi kết quả vào trường nào,
qua cửa nào; ai biến con số đó thành Pod thật, và vì sao quota, PDB và ràng buộc trải đều phải được
tính trước cho đúng lúc ấy. Rồi kể đường về khi tải tan: vì sao chậm, chậm bao nhiêu do cấu hình nào
của bạn quyết định, và vì sao `deploy.sh` hay CI chạy giữa chừng không còn làm `web` giật số bản sao.
Kết thúc bằng việc chỉ ra trục dọc vẫn đứng ở đâu, và Lab 12 sẽ dựa vào những gì của sprint này khi
drain node.

---

## 4. Troubleshooting của lab này

Sự cố dựng môi trường xem [mục 4 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#4-troubleshooting-môi-trường).
Sự cố của chính metrics-server xem mục troubleshooting của [Lab 11a](LAB-11A-OBSERVABILITY.md). Dưới
đây chỉ là sự cố phát sinh trong nội dung bài học.

| Triệu chứng | Nguyên nhân thường gặp | Xử lý |
| --- | --- | --- |
| B0: `kubectl top node` báo lỗi | Cluster không ở `04-metrics-ready` | Restore cả ba VM về đúng mốc, hoặc quay lại Lab 11a. Không cài metrics-server trong lab này |
| B0: `smoke.sh` của tag `lab-11a` fail | Repo sprint 17 chưa đủ — có thứ chỉ tồn tại trên cluster cũ | Sửa repo ở tag `lab-11a`, tag lại, rồi mới mở lab |
| B1, B2: `kubectl get hpa` hiện `<unknown>`, `ScalingActive=False` | Hai nguyên nhân cùng triệu chứng: container thiếu `requests.cpu` (mẫu số), hoặc Metrics API không trả số (tử số) | `kubectl top pod -n bookstore` có số thì thiếu mẫu số — kiểm `requests` của **cả sidecar**; không có số thì quay về B1 phần 1. Không sửa mục tiêu của HPA để "cho chạy" |
| B2: `ScalingActive=False` lý do `AmbiguousSelector` | Có HPA thứ hai chọn cùng Pod `web` — thường do chạy `kubectl autoscale` không kèm `--dry-run` | `kubectl -n bookstore get hpa`; xóa HPA thừa, giữ HPA `web` của repo |
| B2: dry-run `kubectl autoscale` ra chuỗi rỗng phần metric | `kubectl` tạo object bằng `autoscaling/v1` | Đối chiếu bằng `-o yaml`; object `v1` mang `targetCPUUtilizationPercentage` thay cho khối `metrics` |
| B3: `apply set-last-applied` báo thiếu annotation | Deployment `web` từng được tạo bằng `create` | Thêm `--create-annotation=true` vào chính lệnh đó |
| B3: `web` tụt về 1 bản sao sau khi sửa | Chạy `deploy.sh` trước khi cập nhật annotation last-applied | HPA kéo lại lên `minReplicas` ở chu kỳ kế tiếp, sự kiện ghi `below Spec.MinReplicas`. Gate đó còn FAIL tới khi sự kiện cũ hết hạn — ghi sự cố vào ADR, đó chính là cú tụt bài 72 cảnh báo |
| B3: phần 1 báo không diễn tập được | `replicas` trong manifest bằng `minReplicas` | Hai bên ghi cùng một số nên không thấy xung đột; ghi nhận vào ADR và chuyển sang phần 2 |
| B4, B6: Pod sinh tải không được tạo, Job có sự kiện `FailedCreate` | `violates PodSecurity "restricted"`; `exceeded quota`; vượt `max` của LimitRange | Sửa `securityContext`; làm B5 trước; hạ trần của Pod sinh tải vào trong LimitRange |
| B4: có policy mà Pod `loadtest` vẫn bị chặn | Policy DNS của sprint 7 không chọn Pod `loadtest`; policy viết cổng 80; thiếu một trong hai chiều | `kubectl -n bookstore describe networkpolicy`; viết theo cổng 8080 của Pod, đủ egress và ingress |
| B4: chưa có policy mà Pod `loadtest` đã gọi được `web` | Mặc định chặn hai chiều của sprint 7 không còn hiệu lực | Sự cố của hệ thống, không phải của ticket: kiểm NetworkPolicy mặc định và CNI trước khi đi tiếp |
| B5: rollout ở `maxReplicas` kẹt, ReplicaSet có `ReplicaFailure` | Quota thiếu cho surge cộng Pod đang tắt | Nâng `spec.hard` trong `k8s/app/policy/`, `deploy.sh`, chạy lại diễn tập |
| B6: tải chạy mà HPA không mở rộng | Đi theo thứ tự: Pod sinh tải không tới được `web`; tải tới nhưng `nginx` rẻ quá so với mục tiêu | Đọc `b6-log-tai.txt` (số `ok`, `loi`); `kubectl top pod -l app.kubernetes.io/component=web`. Nếu tải tới mà thiếu, tăng `parallelism` thêm một và chạy lại gate trần của B4. **Không** hạ `requests` của `web` |
| B6: HPA nhảy thẳng lên `maxReplicas` | Tải vượt mục tiêu nhiều lần | Không phải sự cố; `ScalingLimited=True` lý do `TooManyReplicas` — ghi vào postmortem |
| B6: phép chia lệch đúng tỷ lệ `nginx / (nginx + sidecar)` | Controller trên cluster tính mẫu số khác cách lab mô tả | Ghi cả hai con số vào postmortem kèm `b6-phep-chia.txt`; đối chiếu ghi chú cộng gộp của bài 72 |
| B6, B7: chờ hết vòng mà `web` không về `minReplicas` | Còn Pod sinh tải; `minReplicas` còn bị patch từ B5 | `kubectl -n bookstore get pod -l app.kubernetes.io/component=loadtest`; `./scripts/deploy.sh` để trả HPA |
| B7: `deploy.sh` báo lỗi validation ở `behavior` | `periodSeconds` vượt 1800, `type` sai chính tả | Sửa theo mục *Các chính sách co giãn* của bài 72 |
| B8: lúc nhàn `ScalingLimited=True` nhưng lý do là `ScaleDownLimit` | Vừa có một bước thu hẹp trong chu kỳ của chính sách `Pods: 1`; chính sách đang giới hạn tốc độ, chưa tới lượt sàn | Chờ hết `periodSeconds` của chính sách rồi đọc lại; lý do chuyển sang `TooFewReplicas` |
| B8: sau khi về 0, HPA không đưa `web` lên | Đúng thiết kế — chế độ bảo trì ngầm | `kubectl -n bookstore scale deployment web --replicas=2` hoặc `./scripts/deploy.sh` |
| B10: `smoke.sh` báo `web` ngoài khoảng HPA hoặc chưa có số đo | Chạy ngay sau deploy, chưa hết vòng chờ; hoặc `ScalingActive=False` | Chạy lại sau một phút; nếu vẫn fail, đọc `kubectl describe hpa web` theo dòng thứ ba của bảng |
| B10: `diff` checksum báo khác | Có thứ trong `kube-system` hoặc manifest control plane đã bị sửa | Lab không tạo snapshot nên sai lệch sẽ đi theo cluster: tắt và restore **cả ba VM** về `04-metrics-ready`, làm lại từ B0 |

---

## 5. Nguồn chính thức

- [Kubernetes v1.35 — Autoscaling Workloads](https://v1-35.docs.kubernetes.io/docs/concepts/workloads/autoscaling/)
  — bài [71](../71-autoscaling-vi.md)
- [Kubernetes v1.35 — Horizontal Pod Autoscaling](https://v1-35.docs.kubernetes.io/docs/concepts/workloads/autoscaling/horizontal-pod-autoscale/)
  — bài [72](../72-horizontal-pod-autoscale-vi.md)
- [Kubernetes v1.35 — Vertical Pod Autoscaling](https://v1-35.docs.kubernetes.io/docs/concepts/workloads/autoscaling/vertical-pod-autoscale/)
  — bài [73](../73-vertical-pod-autoscale-vi.md)
- [Kubernetes v1.35 — HorizontalPodAutoscaler Walkthrough](https://v1-35.docs.kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/)
  — bài [342](../342-hpa-walkthrough-vi.md)
- [Kubernetes v1.35 — HorizontalPodAutoscaler API `autoscaling/v2`](https://v1-35.docs.kubernetes.io/docs/reference/kubernetes-api/workload-resources/horizontal-pod-autoscaler-v2/)
- [Kubernetes v1.35 — `kubectl autoscale`](https://v1-35.docs.kubernetes.io/docs/reference/kubectl/generated/kubectl_autoscale/)
- [Kubernetes v1.35 — `kubectl apply set-last-applied`](https://v1-35.docs.kubernetes.io/docs/reference/kubectl/generated/kubectl_apply/kubectl_apply_set-last-applied/)
- [Kubernetes v1.35 — Deployments](https://v1-35.docs.kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Kubernetes v1.35 — Resource Quotas](https://v1-35.docs.kubernetes.io/docs/concepts/policy/resource-quotas/)
- [Kubernetes v1.35 — Specifying a Disruption Budget for your Application](https://v1-35.docs.kubernetes.io/docs/tasks/run-application/configure-pdb/)
- [Kubernetes v1.35 — Jobs](https://v1-35.docs.kubernetes.io/docs/concepts/workloads/controllers/job/)
- [Kubernetes v1.35 — Network Policies](https://v1-35.docs.kubernetes.io/docs/concepts/services-networking/network-policies/)
- [Kubernetes v1.35 — Pod Security Standards](https://v1-35.docs.kubernetes.io/docs/concepts/security/pod-security-standards/)
- [Kubernetes v1.35 — Pod Topology Spread Constraints](https://v1-35.docs.kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints/)
- [Kubernetes v1.35 — Feature Gates](https://v1-35.docs.kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/)
  — trạng thái `HPAConfigurableTolerance` cho ⭐ của B7
- [Kubernetes — mã nguồn controller HPA nhánh release-1.35](https://github.com/kubernetes/kubernetes/tree/release-1.35/pkg/controller/podautoscaler)
  — lý do của condition và cách tính `requests` gồm sidecar, dùng để đối chiếu B6, B8
- [Vertical Pod Autoscaler — kho GitHub](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler)
  — nơi lấy bảng tương thích nếu về sau trả nốt phần VPA của nợ #1 theo ba điều kiện ở B9
