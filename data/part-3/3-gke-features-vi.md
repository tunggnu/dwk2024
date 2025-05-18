---
path: "/part-3/3-gke-features-vi"
title: "Các tính năng của GKE"
hidden: false
---

<text-box variant='learningObjectives' name='Mục tiêu học tập'>

Sau phần này bạn có thể

- So sánh DBaaS với giải pháp tự triển khai

- Thiết lập autoscale cho các pod trong cụm

- Thiết lập autoscale cho chính cụm

</text-box>

## Volume một lần nữa

Bây giờ chúng ta đến một ngã rẽ. Chúng ta có thể bắt đầu sử dụng Database as a Service (DBaaS) như Google Cloud SQL trong trường hợp này, hoặc chỉ sử dụng PersistentVolumeClaims với image Postgres của riêng mình và để Google Kubernetes Engine lo phần lưu trữ thông qua PersistentVolumes.

Cả hai giải pháp đều được sử dụng rộng rãi.

<exercise name='Bài tập 3.06: DBaaS vs DIY'>

Hãy so sánh ưu/nhược điểm của hai giải pháp về các điểm khác biệt quan trọng. Bao gồm **ít nhất** công việc và chi phí cần thiết để khởi tạo cũng như bảo trì. Phương pháp backup và mức độ dễ sử dụng cũng nên được cân nhắc.

Viết câu trả lời vào README của dự án.

</exercise>

<exercise name='Bài tập 3.07: Backup'>

Ở [phần 2](/part-2/4-statefulsets-and-jobs#jobs-and-cronjobs) chúng ta đã tạo một Job để backup Database bằng lệnh _pg_dump_. Tuy nhiên, bản backup chưa được lưu ở đâu cả. Bây giờ hãy tạo một CronJob để backup database (mỗi 24 giờ một lần) và lưu lên [Google Object Storage](https://cloud.google.com/storage).

Trong bài tập này, bạn có thể tạo secret cho quyền truy cập cloud từ dòng lệnh, do đó không cần tạo trong GitHub action.

Khi cron job hoạt động, bạn có thể tải backup về bằng Google Cloud Console:

<img src="../img/bucket.png">

</exercise>

## Scaling

Scaling có thể là scaling ngang (horizontal) hoặc scaling dọc (vertical). Scaling dọc là tăng tài nguyên cho một pod hoặc node. Scaling ngang là tăng số lượng pod hoặc node, đây là kiểu scaling phổ biến nhất. Chúng ta sẽ tập trung vào scaling ngang.

### Scaling pod

Có nhiều lý do để scale một ứng dụng. Lý do phổ biến nhất là số lượng request nhận được vượt quá khả năng xử lý. Giới hạn thường là số request framework có thể xử lý hoặc giới hạn thực tế về CPU hoặc RAM.

Tôi đã chuẩn bị một [ứng dụng](https://github.com/kubernetes-hy/material-example/tree/master/app7) khá tốn CPU. Có sẵn image Docker `jakousa/dwk-app7:e11a700350aede132b62d3b5fd63c05d6b976394`. Ứng dụng nhận tham số truy vấn _?fibos=25_ để điều chỉnh độ dài tính toán. Bạn nên dùng giá trị từ 15 đến 30.

Đây là cấu hình để chạy ứng dụng:

**deployment.yaml**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cpushredder-dep
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cpushredder
  template:
    metadata:
      labels:
        app: cpushredder
    spec:
      containers:
        - name: cpushredder
          image: jakousa/dwk-app7:e11a700350aede132b62d3b5fd63c05d6b976394
          resources:
            limits:
              cpu: "150m"
              memory: "100Mi"
```

Lưu ý cuối cùng chúng ta đã đặt [giới hạn tài nguyên](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/) cho Deployment. Hậu tố "m" của giới hạn CPU nghĩa là "một phần nghìn của một core". Vậy `150m` tương đương 15% một core CPU (`150/1000=0,15`).

Service thì đã rất quen thuộc:

**service.yaml**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: cpushredder-svc
spec:
  type: LoadBalancer
  selector:
    app: cpushredder
  ports:
    - port: 80
      protocol: TCP
      targetPort: 3001
```

Tiếp theo là [HorizontalPodAutoscaler](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/). Đây là một Resource mới rất thú vị.

**horizontalpodautoscaler.yaml**

```yaml
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: cpushredder-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: cpushredder-dep
  minReplicas: 1
  maxReplicas: 6
  targetCPUUtilizationPercentage: 50
```

HorizontalPodAutoscaler sẽ tự động scale pod theo chiều ngang. File yaml này xác định Deployment mục tiêu, số replica tối thiểu và tối đa. Mục tiêu sử dụng CPU cũng được đặt. Nếu sử dụng CPU vượt quá mục tiêu thì sẽ tạo thêm replica cho đến khi đạt tối đa.

Hãy thử xem điều gì xảy ra:

```console
$ kubectl top pod -l app=cpushredder
  NAME                               CPU(cores)   MEMORY(bytes)
  cpushredder-dep-85f5b578d7-nb5rs   1m           20Mi

$ kubectl get hpa
  NAME              REFERENCE                    TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
  cpushredder-hpa   Deployment/cpushredder-dep   0%/50%    1         6         1          62s

$ kubectl get svc
  NAME              TYPE           CLUSTER-IP      EXTERNAL-IP      PORT(S)        AGE
  cpushredder-svc   LoadBalancer   10.31.254.209   35.228.149.206   80:32577/TCP   94s
```

Mở external-ip ở trên, ví dụ http://35.228.149.206, trên trình duyệt sẽ bắt đầu một tiến trình tốn CPU. Làm mới trang vài lần, nếu bạn gửi request vượt quá giới hạn thì pod sẽ bị dừng.

```console
$ kubectl logs -f cpushredder-dep-85f5b578d7-nb5rs
  Started in port 3001
  Received a request
  started fibo with 20
  Received a request
  started fibo with 20
  Fibonacci 20: 10946
  Closed
  Fibonacci 20: 10946
  Closed
  Received a request
  started fibo with 20
```

Sau vài request, _HorizontalPodAutoscaler_ sẽ tạo thêm replica khi sử dụng CPU tăng lên. Do tài nguyên dao động, đôi khi rất lớn khi khởi động hoặc tắt, _HPA_ mặc định sẽ chờ 5 phút giữa các lần giảm số replica. Nếu ứng dụng của bạn có nhiều replica dù ở 0%/50% thì hãy chờ. Nếu thời gian chờ quá ngắn, số replica có thể "nhảy loạn".

Tôi khuyên bạn mở cụm bằng Lens và làm mới trang để quan sát. Nếu không có Lens, `kubectl get deployments --watch` và `kubectl get pods --watch` cũng cho thấy hành vi thời gian thực.

Mặc định sẽ mất 300 giây để scale down. Bạn có thể thay đổi thời gian này bằng cách thêm vào HorizontalPodAutoscaler:

```yaml
behavior:
  scaleDown:
    stabilizationWindowSeconds: 30
```

Tìm hiểu autoscale với HorizontalPodAutoscaler có thể là một trong những nhiệm vụ khó nhất. Việc chọn tài nguyên nào để theo dõi và khi nào scale không hề dễ. Ở đây chúng ta chỉ stress CPU, nhưng ứng dụng thực tế có thể cần scale dựa trên nhiều tài nguyên khác như network, disk hoặc memory.

<exercise name='Bài tập 3.08: Dự án v1.5'>

Đặt giới hạn tài nguyên hợp lý cho dự án. Giá trị chính xác không quan trọng. Hãy thử nghiệm xem giá trị nào phù hợp.

</exercise>

<exercise name='Bài tập 3.09: Giới hạn tài nguyên'>

Đặt giới hạn tài nguyên hợp lý cho ứng dụng Ping-pong và Log output. Giá trị chính xác không quan trọng. Hãy thử nghiệm xem giá trị nào phù hợp.

</exercise>

### Scaling node

Scaling node là một tính năng được hỗ trợ trong GKE. Với cluster autoscaling, chúng ta có thể sử dụng đúng số node cần thiết.

```console
$ gcloud container clusters update dwk-cluster --zone=europe-north1-b --enable-autoscaling --min-nodes=1 --max-nodes=5
  Updated [https://container.googleapis.com/v1/projects/dwk-gke/zones/europe-north1-b/clusters/dwk-cluster].
  To inspect the contents of your cluster, go to: https://console.cloud.google.com/kubernetes/workload_/gcloud/europe-north1-b/dwk-cluster?project=dwk-gke
```

Để có cụm mạnh mẽ hơn, xem ví dụ tạo cụm tại: <https://cloud.google.com/kubernetes-engine/docs/concepts/cluster-autoscaler>

<img src="../img/gke_scaling.png">

Cluster autoscaling có thể làm gián đoạn pod bằng cách di chuyển chúng khi số node tăng hoặc giảm. Để giải quyết vấn đề này, có thể dùng resource [PodDisruptionBudget](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/#how-disruption-budgets-work). Với resource này, bạn có thể đặt yêu cầu cho pod qua hai trường: _minAvailable_ và _maxUnavailable_.

**poddisruptionbudget.yaml**

```yaml
apiVersion: policy/v1beta1
kind: PodDisruptionBudget
metadata:
  name: example-app-pdb
spec:
  maxUnavailable: 50%
  selector:
    matchLabels:
      app: example-app
```

Cấu hình này đảm bảo không quá một nửa số pod bị gián đoạn. Tài liệu Kubernetes ghi rõ "Budget chỉ bảo vệ khỏi các lần loại bỏ tự nguyện, không phải mọi nguyên nhân gây gián đoạn."

Ngoài scaling ngang (nhiều node), bạn cũng có thể scale từng node với [VerticalPodAutoscaler](https://cloud.google.com/kubernetes-engine/docs/concepts/verticalpodautoscaler). Điều này giúp đảm bảo bạn luôn sử dụng tối đa tài nguyên đã trả tiền.

_Lưu ý thêm:_ Kubernetes cũng cho phép giới hạn tài nguyên theo namespace. Điều này giúp ngăn ứng dụng ở namespace phát triển tiêu tốn quá nhiều tài nguyên. Google có một video tuyệt vời giải thích về đối tượng `ResourceQuota`.

<iframe width="560" height="315" src="https://www.youtube-nocookie.com/embed/xjpHggHKm78" frameborder="0" allow="accelerometer; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

<exercise name='Bài tập 3.10: Dự án v1.6'>

GKE đã tích hợp sẵn hệ thống giám sát nên chỉ cần bật monitoring.

Đọc tài liệu về Kubernetes Engine Monitoring [tại đây](https://cloud.google.com/monitoring/kubernetes-engine) và thiết lập logging cho dự án trên GKE.

Bạn có thể tích hợp thêm Prometheus nếu muốn.

Nộp ảnh chụp log khi tạo một todo mới.

</exercise>
