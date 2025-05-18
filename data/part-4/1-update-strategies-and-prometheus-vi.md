---
path: "/part-4/1-update-strategies-and-prometheus"
title: "Chiến lược cập nhật và Prometheus"
hidden: false
---

<text-box variant='learningObjectives' name='Mục tiêu học tập'>

Sau phần này, bạn có thể

- So sánh các chiến lược cập nhật

- Triển khai phần mềm một cách **tự tin** với canary releases

- Sử dụng Prometheus cho các truy vấn tùy chỉnh

</text-box>

Ở phần trước, chúng ta đã thực hiện cập nhật tự động với pipeline triển khai. Việc cập nhật _đơn giản là hoạt động_, nhưng chúng ta không biết quá trình cập nhật thực sự diễn ra như thế nào, ngoài việc một pod đã được thay đổi. Bây giờ hãy xem cách làm cho quá trình cập nhật an toàn hơn để giúp chúng ta đạt được mức [sẵn sàng cao](https://en.wikipedia.org/wiki/High_availability).

Có nhiều chiến lược cập nhật/triển khai/phát hành khác nhau. Chúng ta sẽ tập trung vào hai chiến lược:

- Rolling update (Cập nhật cuốn chiếu)
- Canary release (Phát hành canary)

Cả hai chiến lược này đều nhằm đảm bảo ứng dụng hoạt động trong và sau khi cập nhật. Thay vì cập nhật tất cả pod cùng lúc, ý tưởng là cập nhật từng pod một và xác nhận ứng dụng vẫn hoạt động.

### Rolling update

Mặc định, Kubernetes thực hiện [rolling update](https://kubernetes.io/docs/concepts/workloads/management/#updating-your-application-without-an-outage) khi chúng ta thay đổi image. Điều này có nghĩa là mỗi pod sẽ được cập nhật tuần tự. Rolling update là mặc định tuyệt vời vì nó giúp ứng dụng luôn sẵn sàng trong quá trình cập nhật. Nếu chúng ta đẩy một image bị lỗi, quá trình cập nhật sẽ tự động dừng lại.

Tôi đã chuẩn bị một ứng dụng với 5 phiên bản. Tag v1 luôn hoạt động, v2 không bao giờ hoạt động, v3 hoạt động 90% thời gian, v4 sẽ chết sau 20 giây và v5 luôn hoạt động.

**deployment.yaml**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flaky-update-dep
spec:
  replicas: 4
  selector:
    matchLabels:
      app: flaky-update
  template:
    metadata:
      labels:
        app: flaky-update
    spec:
      containers:
        - name: flaky-update
          image: jakousa/dwk-app8:v1
```

```shell
$ kubectl apply -f deployment.yaml
  deployment.apps/flaky-update-dep created

$ kubectl get po
  NAME                                READY   STATUS    RESTARTS   AGE
  flaky-update-dep-7b5fd9ffc7-27cxt   1/1     Running   0          87s
  flaky-update-dep-7b5fd9ffc7-mp8vd   1/1     Running   0          88s
  flaky-update-dep-7b5fd9ffc7-m4smm   1/1     Running   0          87s
  flaky-update-dep-7b5fd9ffc7-nzl98   1/1     Running   0          88s
```

Bây giờ hãy đổi tag thành v2 và áp dụng lại.

```shell
$ kubectl apply -f deployment.yaml
$ kubectl get po --watch
...
```

Bạn sẽ thấy rolling update được thực hiện nhưng tiếc là ứng dụng không còn hoạt động. Ứng dụng vẫn chạy, chỉ là có bug khiến nó không hoạt động đúng. Làm sao để thông báo lỗi này ra ngoài ứng dụng? Đây là lúc [_ReadinessProbes_](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/#define-readiness-probes) phát huy tác dụng.

**Kubernetes Best Practices - Kubernetes Health Checks with Readiness and Liveness Probes**

<iframe width="560" height="315" src="https://www.youtube-nocookie.com/embed/mxEvAPQRwhw" frameborder="0" allow="accelerometer; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

Với _ReadinessProbe_, Kubernetes có thể kiểm tra pod đã sẵn sàng xử lý request chưa. Ứng dụng có endpoint [/healthz](https://stackoverflow.com/questions/43380939/where-does-the-convention-of-using-healthz-for-application-health-checks-come-f) trên port 3541, chúng ta có thể dùng để kiểm tra sức khỏe. Nó sẽ trả về status code 500 nếu không hoạt động và 200 nếu hoạt động.

Hãy rollback về v1 để thử lại cập nhật lên v2.

**deployment.yaml**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flaky-update-dep
spec:
  replicas: 4
  selector:
    matchLabels:
      app: flaky-update
  template:
    metadata:
      labels:
        app: flaky-update
    spec:
      containers:
        - name: flaky-update
          image: jakousa/dwk-app8:v1
          readinessProbe:
            initialDelaySeconds: 10 # Đợi ban đầu trước khi kiểm tra readiness
            periodSeconds: 5 # Tần suất kiểm tra
            httpGet:
              path: /healthz
              port: 3541
```

Ở đây _initialDelaySeconds_ và _periodSeconds_ nghĩa là probe sẽ gửi sau 10 giây khi container khởi động và sau đó mỗi 5 giây. Nếu đổi tag thành v2 và áp dụng, kết quả sẽ như sau:

```shell
$ kubectl apply -f deployment.yaml
  deployment.apps/flaky-update-dep configured

$ kubectl get po
  NAME                                READY   STATUS    RESTARTS   AGE
  flaky-update-dep-f5c79dbc-8lnqm     1/1     Running   0          115s
  flaky-update-dep-f5c79dbc-86fmd     1/1     Running   0          116s
  flaky-update-dep-f5c79dbc-qzs9p     1/1     Running   0          98s
  flaky-update-dep-54888b877b-dkctl   0/1     Running   0          25s
  flaky-update-dep-54888b877b-dbw29   0/1     Running   0          24s
```

Ba pod hoạt động bình thường, một pod v1 bị loại bỏ để nhường chỗ cho pod v2, nhưng vì pod v2 không hoạt động nên không bao giờ READY và quá trình cập nhật không thể tiếp tục.

### GKE ingress và pod readiness

Nếu bạn dùng Ingress trong ứng dụng, hãy lưu ý ReadinessProbe không hoạt động như mong đợi, request vẫn có thể được chuyển đến pod chưa _ready_. Tài liệu [khuyến nghị](https://cloud.google.com/kubernetes-engine/docs/concepts/ingress#interpreted_hc):

_Nếu các pod phục vụ cho Service của bạn có nhiều container, hoặc bạn dùng GKE Enterprise Ingress controller, hãy dùng BackendConfig CRD để định nghĩa tham số health check._ Nếu muốn dùng ingress mặc định của GKE, nên định nghĩa [BackendConfig](https://cloud.google.com/kubernetes-engine/docs/concepts/ingress#direct_hc_instead). Thay vì dùng Ingress của nhà cung cấp, bạn nên dùng [ingress-nginx](https://github.com/kubernetes/ingress-nginx/tree/main) để có hành vi nhất quán ở mọi nơi.

Cài đặt _ingress-nginx_ rất dễ. Chạy các lệnh sau:

```bash
$ helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
$ helm repo update
$ helm install nginx-ingress ingress-nginx/ingress-nginx
```

Trong [ingress spec](https://kubernetes.io/docs/reference/kubernetes-api/service-resources/ingress-v1/#IngressSpec), trường _ingressClassName_ chỉ định controller sẽ dùng:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ping-pong-ingress
spec:
  ingressClassName: nginx # thêm dòng này
  rules:
    - http:
        # rules ở đây
```

Các ingress controller khả dụng (ngoài mặc định) có thể kiểm tra bằng _kubectl_:

```bash
$ kubectl get IngressClass
NAME    CONTROLLER             PARAMETERS   AGE
nginx   k8s.io/ingress-nginx   <none>       3h38m
```

Có một bài học ở đây:

<img src="../img/jami-ingress.png">

<exercise name='Bài tập 4.01: Readiness Probe'>

Tạo ReadinessProbe cho ứng dụng Ping-pong. Nó chỉ sẵn sàng khi kết nối được với database.

Và một ReadinessProbe khác cho ứng dụng Log output. Nó chỉ sẵn sàng khi nhận được dữ liệu từ ứng dụng Ping-pong.

Kiểm tra bằng cách áp dụng mọi thứ trừ statefulset của database. Kết quả `kubectl get po` sẽ như sau trước khi database sẵn sàng:

```shell
NAME                             READY   STATUS    RESTARTS   AGE
logoutput-dep-7f49547cf4-ttj4f   1/2     Running   0          21s
pingpong-dep-9b698d6fb-jdgq9     0/1     Running   0          21s
```

Thêm database sẽ tự động chuyển trạng thái READY thành 2/2 và 1/1 cho Log output và Ping-pong.

</exercise>

Dù v2 không hoạt động, ít nhất ứng dụng vẫn chạy. Chúng ta chỉ cần đẩy bản cập nhật mới lên v2. Hãy thử v4, bản này sẽ hỏng sau một thời gian ngắn:

```shell
$ kubectl apply -f deployment.yaml
  deployment.apps/flaky-update-dep configured
```

ReadinessProbe có thể pass trong 20 giây đầu, nhưng sau đó mọi pod sẽ hỏng. Đáng tiếc _ReadinessProbe_ không thể làm gì, deployment thành công nhưng ứng dụng bị lỗi.

```shell
$ kubectl get po
  NAME                               READY   STATUS    RESTARTS   AGE
  flaky-update-dep-dd78944f4-vv27w   0/1     Running   0          111s
  flaky-update-dep-dd78944f4-dnmcg   0/1     Running   0          110s
  flaky-update-dep-dd78944f4-zlh4v   0/1     Running   0          92s
  flaky-update-dep-dd78944f4-zczmw   0/1     Running   0          90s
```

Hãy rollback về phiên bản trước. Điều này rất hữu ích nếu bạn cần rollback gấp:

```shell
$ kubectl rollout undo deployment flaky-update-dep
  deployment.apps/flaky-update-dep rolled back
```

Lệnh này sẽ rollback về phiên bản trước. Nếu đó là v2 (không hoạt động), cần dùng thêm flag:

```shell
$ kubectl describe deployment flaky-update-dep | grep Image
    Image:        jakousa/dwk-app8:v2

$ kubectl rollout undo deployment flaky-update-dep --to-revision=1
  deployment.apps/flaky-update-dep rolled back

$ kubectl describe deployment flaky-update-dep | grep Image
    Image:        jakousa/dwk-app8:v1
```

Đọc `kubectl rollout undo --help` để biết thêm!

Có một probe khác có thể giúp trong tình huống như v4. [_LivenessProbes_](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/#define-readiness-probes) cấu hình tương tự _ReadinessProbes_, nhưng nếu check fail thì container sẽ được restart.

**deployment.yaml**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flaky-update-dep
spec:
  replicas: 4
  selector:
    matchLabels:
      app: flaky-update
  template:
    metadata:
      labels:
        app: flaky-update
    spec:
      containers:
        - name: flaky-update
          image: jakousa/dwk-app8:v1
          readinessProbe:
            initialDelaySeconds: 10 # Đợi ban đầu trước khi kiểm tra readiness
            periodSeconds: 5 # Tần suất kiểm tra
            httpGet:
              path: /healthz
              port: 3541
          livenessProbe:
            initialDelaySeconds: 20 # Đợi ban đầu trước khi kiểm tra liveness
            periodSeconds: 5 # Tần suất kiểm tra
            httpGet:
              path: /healthz
              port: 3541
```

Bây giờ hãy triển khai phiên bản tệ nhất, v3.

```shell
$ kubectl apply -f deployment.yaml
  deployment.apps/flaky-update-dep configured
```

Sau một lúc, có thể sẽ như sau (nếu bạn may mắn):

```shell
$ kubectl get po
  NAME                                READY   STATUS    RESTARTS   AGE
  flaky-update-dep-fd65cd468-4vgwx   1/1     Running   3          2m30s
  flaky-update-dep-fd65cd468-9h877   0/1     Running   4          2m49s
  flaky-update-dep-fd65cd468-jpz2m   0/1     Running   3          2m13s
  flaky-update-dep-fd65cd468-529nr   1/1     Running   4          2m50s
```

Ít nhất vẫn còn pod hoạt động!

Nếu ứng dụng của bạn khởi động chậm, có thể dùng [StartupProbe](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/#define-startup-probes) để trì hoãn liveness probe, tránh bị restart sớm. Bạn có thể cần trong thực tế nhưng sẽ không bàn sâu ở khóa học này.

<exercise name='Bài tập 4.02: Dự án v1.7'>

Tạo các probe và endpoint cần thiết cho Dự án để đảm bảo ứng dụng hoạt động và kết nối được với database.

Kiểm tra probe bằng cách dùng phiên bản không truy cập được database, ví dụ nhập sai URL hoặc thông tin đăng nhập.

</exercise>

### Canary release

Với rolling update và các Probe, chúng ta có thể phát hành không downtime cho người dùng. Đôi khi như vậy là chưa đủ, bạn cần phát hành _một phần_ cho một số người dùng và thu thập dữ liệu trước khi phát hành rộng rãi. [Canary release](https://martinfowler.com/bliki/CanaryRelease.html) là chiến lược phát hành cho một phần nhỏ người dùng dùng thử phiên bản mới. Khi tự tin hơn, tăng dần số người dùng cho đến khi không còn ai dùng phiên bản cũ.

Hiện tại, Canary không phải là chiến lược được Kubernetes hỗ trợ sẵn. Có thể do cách triển khai canary rất đa dạng. Chúng ta sẽ dùng [Argo Rollouts](https://argoproj.github.io/argo-rollouts/) để thử một kiểu canary release:

```shell
$ kubectl create namespace argo-rollouts
$ kubectl apply -n argo-rollouts -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml
```

Bây giờ chúng ta có resource mới [Rollout](https://argoproj.github.io/argo-rollouts/migrating/). Rollout sẽ thay thế Deployment trước đó và cho phép dùng trường mới:

**rollout.yaml**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: flaky-update-dep
spec:
  replicas: 4
  selector:
    matchLabels:
      app: flaky-update
  strategy:
    canary:
      steps:
        - setWeight: 25
        - pause:
            duration: 30s
        - setWeight: 50
        - pause:
            duration: 30s
  template:
    metadata:
      labels:
        app: flaky-update
    spec:
      containers:
        - name: flaky-update
          image: jakousa/dwk-app8:v1
          readinessProbe:
            initialDelaySeconds: 10 # Đợi ban đầu trước khi kiểm tra readiness
            periodSeconds: 5 # Tần suất kiểm tra
            httpGet:
              path: /healthz
              port: 3541
          livenessProbe:
            initialDelaySeconds: 20 # Đợi ban đầu trước khi kiểm tra liveness
            periodSeconds: 5 # Tần suất kiểm tra
            httpGet:
              path: /healthz
              port: 3541
```

Chiến lược trên sẽ chuyển 25% ([_setWeight_](https://argoproj.github.io/argo-rollouts/features/canary/#overview)) pod sang phiên bản mới (ở đây là 1 pod), sau đó chờ 30 giây, chuyển 50% pod và lại chờ 30 giây cho đến khi tất cả pod được cập nhật. [kubectl plugin](https://argoproj.github.io/argo-rollouts/features/kubectl-plugin/) của Argo cũng cung cấp lệnh [promote](https://argoproj.github.io/argo-rollouts/generated/kubectl-argo-rollouts/kubectl-argo-rollouts_promote/) để tạm dừng rollout và chủ động chuyển tiếp.

Có các tùy chọn khác như [_maxUnavailable_](https://argoproj.github.io/argo-rollouts/features/bluegreen/#maxunavailable) nhưng mặc định là đủ. Tuy nhiên, chỉ rollout chậm lên production là chưa đủ cho canary. Cũng như rolling update, _chúng ta cần biết trạng thái ứng dụng_.

Với resource AnalysisTemplate đã cài cùng Argo Rollouts, chúng ta có thể định nghĩa một bài test để ngăn phiên bản lỗi được phát hành.

Hãy mở rộng rollout với một analysis:

```yaml
  ...
  strategy:
    canary:
      steps:
      - setWeight: 50
      - analysis:
          templates:
          - templateName: restart-rate
    # ...
  ...
```

Analysis sẽ dùng template _restart-rate_. Tiếp theo, cần định nghĩa cách thực hiện analysis. Chúng ta sẽ cần Prometheus mà đã dùng ở [phần 2](/part-2/5-monitoring).

<exercise name='Bài tập 4.03: Prometheus'>

Ở phần 2, chúng ta đã khởi động Prometheus nhưng mới chỉ làm quen. Hãy thực hiện một truy vấn thực tế để hiểu hơn.

Khởi động Prometheus bằng Helm và dùng port-forward để truy cập giao diện web. Port mặc định là 9090:

```shell
$ kubectl -n prometheus get pods
 NAME                                                              READY   STATUS    RESTARTS   AGE
  alertmanager-kube-prometheus-stack-1714-alertmanager-0            2/2     Running   0          3h19m
  kube-prometheus-stack-1714-operator-94c596dbd-n5pcl               1/1     Running   0          3h19m
  kube-prometheus-stack-1714644114-grafana-54cbbc4c46-m26f6         3/3     Running   0          3h19m
  kube-prometheus-stack-1714644114-kube-state-metrics-7cb796tpjbd   1/1     Running   0          3h19m
  kube-prometheus-stack-1714644114-prometheus-node-exporter-kdpln   1/1     Running   0          3h19m
  kube-prometheus-stack-1714644114-prometheus-node-exporter-sp9pg   1/1     Running   0          3h19m
  kube-prometheus-stack-1714644114-prometheus-node-exporter-vbbjk   1/1     Running   0          3h19m
  prometheus-kube-prometheus-stack-1714-prometheus-0                2/2     Running   0          3h19m

$ kubectl -n prometheus port-forward prometheus-kube-prometheus-stack-1714-prometheus-0 9090:9090
  Forwarding from 127.0.0.1:9090 -> 9090
  Forwarding from [::1]:9090 -> 9090
```

Truy cập [http://localhost:9090](http://localhost:9090) để viết truy vấn.

**Viết truy vấn** hiển thị số pod được tạo bởi StatefulSets trong namespace _prometheus_. Với setup trên, _Value_ sẽ là 3 pod khác nhau:

<img src="../img/prometheus-p4.png">

Truy vấn "kube_pod_info" sẽ có đủ trường để lọc. Xem [tài liệu](https://prometheus.io/docs/prometheus/latest/querying/basics/) để biết thêm về truy vấn.

</exercise>

AnalysisTemplate sẽ dùng Prometheus để truy vấn trạng thái deployment. Kết quả truy vấn sẽ được so sánh với giá trị đặt trước. Trong ví dụ đơn giản này, nếu tổng số lần restart trong 2 phút qua lớn hơn 2, analysis sẽ fail. _initialDelay_ đảm bảo test chỉ chạy khi đã có đủ dữ liệu. Template như sau:

**analysistemplate.yaml**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: restart-rate
spec:
  metrics:
    - name: restart-rate
      initialDelay: 2m
      successCondition: result < 2
      provider:
        prometheus:
          address: http://kube-prometheus-stack-1602-prometheus.prometheus.svc.cluster.local:9090 # DNS của Prometheus, tìm bằng kubectl describe svc ...
          query: |
            scalar(
              sum(kube_pod_container_status_restarts_total{namespace="default", container="flaky-update"}) -
              sum(kube_pod_container_status_restarts_total{namespace="default", container="flaky-update"} offset 2m)
            )
```

Với Rollout và AnalysisTemplate mới, chúng ta có thể thử triển khai bất kỳ phiên bản nào một cách an toàn. Deploy v2 sẽ bị ngăn bởi các Probe đã thiết lập. Deploy v3 sẽ tự động rollback khi phát hiện crash ngẫu nhiên. v4 cũng sẽ fail, nhưng thời gian test ngắn có thể vẫn để lọt một phiên bản lỗi.

Plugin [kubectl của Argo Rollouts](https://argoproj.github.io/argo-rollouts/features/kubectl-plugin/) cho phép bạn xem trực quan Rollout và các resource liên quan. Để theo dõi Rollout khi deploy, chạy ở chế độ watch:

```bash
kubectl argo rollouts get rollout flaky-update-dep --watch
```

<img src="../img/rollout.png">

Ngoài pod, còn có [AnalysisRun](https://argoproj.github.io/argo-rollouts/architecture/#analysistemplate-and-analysisrun) là instance của bài test. Bạn cũng có thể thử [dashboard của Argo Rollouts](https://argoproj.github.io/argo-rollouts/dashboard/) để xem trạng thái rollout trực quan hơn.

Nói chung, _AnalysisTemplate_ không phụ thuộc vào Prometheus mà có thể dùng nguồn khác, ví dụ endpoint JSON.

<exercise name='Bài tập 4.04: Dự án v1.8'>

Tạo AnalysisTemplate cho Dự án để theo dõi mức sử dụng CPU của tất cả container trong namespace.

Nếu tổng **tốc độ** sử dụng CPU của namespace vượt quá giá trị đặt trước (bạn tự chọn giá trị phù hợp cho dự án) trong vòng 10 phút, hãy revert cập nhật.

Đảm bảo ứng dụng không được cập nhật nếu giá trị đặt quá thấp.

</exercise>

### Các chiến lược triển khai khác

Kubernetes hỗ trợ [Recreate](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#recreate-deployment), chiến lược này dừng toàn bộ pod cũ và thay thế bằng pod mới. Điều này tạo ra một khoảng downtime nhưng đảm bảo không có nhiều phiên bản chạy cùng lúc. Argo Rollouts hỗ trợ [BlueGreen](https://argoproj.github.io/argo-rollouts/features/bluegreen/), trong đó phiên bản mới chạy song song với phiên bản cũ, nhưng traffic sẽ được chuyển sang phiên bản mới tại một thời điểm nhất định, ví dụ sau khi chạy script cập nhật hoặc QA đã duyệt phiên bản mới.

<exercise name='Bài tập 4.05: Dự án v1.9'>

Nói về cập nhật.
Ứng dụng todo của chúng ta nên có trường "Done" cho các todo đã hoàn thành.
Nó nên là một request PUT tới `/todos/<id>`.

</exercise>
