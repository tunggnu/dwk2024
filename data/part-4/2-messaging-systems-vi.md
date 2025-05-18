---
path: "/part-4/2-messaging-systems-vi"
title: "Hệ thống nhắn tin"
hidden: false
---

<text-box variant='learningObjectives' name='Mục tiêu học tập'>

Sau phần này, bạn có thể

- Tạo kiến trúc microservice phức tạp với NATS làm hệ thống nhắn tin

</text-box>

[Hàng đợi tin nhắn](https://en.wikipedia.org/wiki/Message_queue) là một phương pháp giao tiếp giữa các dịch vụ. Chúng có nhiều trường hợp sử dụng và rất hữu ích khi bạn muốn mở rộng ứng dụng. Nhiều dịch vụ HTTP REST API muốn giao tiếp với nhau sẽ cần biết địa chỉ của nhau. Trong khi đó, khi sử dụng hàng đợi tin nhắn, các tin nhắn được gửi đến và nhận từ hàng đợi, không cần biết địa chỉ dịch vụ khác.

Trong phần này, chúng ta sẽ sử dụng một hệ thống nhắn tin có tên là [NATS](https://docs.nats.io/) để khám phá lợi ích của nhắn tin. Trước khi bắt đầu, hãy tìm hiểu những điều cơ bản về NATS.

Trong NATS, các ứng dụng giao tiếp bằng cách gửi và nhận tin nhắn. Những tin nhắn này được định danh và phân loại bởi [subjects](https://docs.nats.io/nats-concepts/subjects). Bên gửi _publish_ tin nhắn với một subject. Bên nhận _subscribe_ vào subject để nhận các tin nhắn đã publish. Trong mô hình [publish-subscribe](https://docs.nats.io/nats-concepts/core-nats/pubsub) mặc định, tất cả các subscriber của subject sẽ nhận được tin nhắn. Ngoài ra còn có mô hình [queue](https://docs.nats.io/nats-concepts/core-nats/queue), trong đó mỗi tin nhắn chỉ được gửi cho **một** subscriber.

NATS cung cấp nhiều chế độ gửi tin nhắn khác nhau. Chức năng cơ bản của [Core NATS](https://docs.nats.io/nats-concepts/core-nats) là gửi tin nhắn _at most once_: nếu không có subscriber nào lắng nghe subject (không khớp subject), hoặc không hoạt động khi tin nhắn được gửi, tin nhắn sẽ không được nhận. Bằng cách sử dụng chức năng [Jetstream](https://docs.nats.io/nats-concepts/jetstream), cũng có thể đạt được gửi tin nhắn _at least once_ hoặc _exactly once_ với tính năng lưu trữ.

Với những điều này, chúng ta có thể thiết kế ứng dụng đầu tiên sử dụng nhắn tin để giao tiếp.

Chúng ta có một tập dữ liệu gồm 100000 đối tượng JSON cần xử lý nặng và sau đó lưu lại dữ liệu đã xử lý. Thật không may, xử lý một đối tượng JSON mất rất nhiều thời gian, nên xử lý toàn bộ dữ liệu sẽ mất hàng giờ. Để giải quyết, tôi đã chia ứng dụng thành các dịch vụ nhỏ hơn để có thể scale riêng biệt.

[Ứng dụng](https://github.com/kubernetes-hy/material-example/tree/master/app9) được chia thành 3 phần:

- Fetcher: lấy dữ liệu chưa xử lý và gửi vào NATS.
- Mapper: xử lý dữ liệu từ NATS và sau khi xử lý gửi lại vào NATS.
- Saver: nhận dữ liệu đã xử lý từ NATS và cuối cùng (có thể) lưu lại.

<img src="../img/app9-plan.png">

Như đã đề cập, nhắn tin trong NATS xoay quanh _subjects_. Thông thường, mỗi mục đích sẽ có một subject riêng. Ứng dụng sử dụng bốn subject:

<img src="../img/nats2.png">

Fetcher chia dữ liệu thành các khối 100 đối tượng và ghi lại những khối nào chưa được xử lý. Ứng dụng được thiết kế để Fetcher không thể scale.

Fetcher subscribe vào subject _mapper_status_ và sẽ chờ Mapper publish một tin nhắn xác nhận đã sẵn sàng xử lý dữ liệu. Khi Fetcher nhận được thông tin này, nó publish một khối dữ liệu vào subject _mapper_data_ và bắt đầu lại từ đầu.

Khi một Mapper sẵn sàng xử lý dữ liệu, nó publish thông tin sẵn sàng vào subject _mapper_status_. Nó cũng subscribe vào subject _mapper_data_. Khi Mapper nhận được tin nhắn, nó xử lý và publish dữ liệu đã xử lý vào subject _saver_data_ rồi lặp lại. Subject _mapper_data_ hoạt động ở chế độ [queue](https://docs.nats.io/nats-concepts/core-nats/queue) nên mỗi tin nhắn chỉ được nhận bởi một Mapper.

Saver subscribe vào subject _saver_data_. Khi nhận được tin nhắn, nó lưu lại và publish một tin xác nhận vào subject _processed_data_. Fetcher subscribe vào subject này để theo dõi các khối dữ liệu đã được lưu. Như vậy, kể cả khi một phần ứng dụng bị crash, toàn bộ dữ liệu cuối cùng vẫn sẽ được xử lý và lưu lại. Ngoài ra, subject _saver_data_ cũng dùng chế độ queue nên mỗi khối dữ liệu chỉ do một Saver xử lý.

Để đơn giản, việc lưu vào database và lấy dữ liệu từ API ngoài bị lược bỏ khỏi ứng dụng.

Trước khi triển khai ứng dụng, chúng ta sẽ dùng Helm để cài đặt NATS vào cụm. Thay vì dùng chart Helm của nhóm NATS, chúng ta sẽ dùng chart của [Bitnami](https://bitnami.com/).

Chart này có [tài liệu](https://artifacthub.io/packages/helm/bitnami/nats) mô tả một loạt [tham số](https://artifacthub.io/packages/helm/bitnami/nats#parameters) có thể dùng để cấu hình NATS. Hãy tắt xác thực bằng cách đặt _auth.enabled_ thành _false_.

Tham số có thể được đặt khi cài đặt như sau:

```shell
$ helm install --set auth.enabled=false my-nats oci://registry-1.docker.io/bitnamicharts/nats

  NAME: my-nats
  LAST DEPLOYED: Wed May  8 22:57:17 2024
  NAMESPACE: default
  STATUS: deployed
  REVISION: 1
  ...
  NATS có thể truy cập qua port 4222 với DNS sau trong cụm:

   my-nats.default.svc.cluster.local

  Dịch vụ monitoring của NATS có thể truy cập qua port 8222 với DNS sau trong cụm:

    my-nats.default.svc.cluster.local

  Để truy cập dịch vụ Monitoring từ ngoài cụm, làm theo các bước sau:

  1. Lấy URL Monitoring bằng lệnh:

      echo "Monitoring URL: http://127.0.0.1:8222"
      kubectl port-forward --namespace default svc/my-nats 8222:8222

  2. Mở trình duyệt và truy cập URL Monitoring
  ...
```

Quá trình cài đặt sẽ in ra nhiều thông tin hữu ích.

Bây giờ chúng ta đã sẵn sàng triển khai [ứng dụng](https://github.com/kubernetes-hy/material-example/tree/master/app9) sử dụng [nats.js](https://github.com/nats-io/nats.js) làm thư viện client. Lưu ý ứng dụng ví dụ dùng nats.js phiên bản 1.5. Phiên bản hiện tại của thư viện có nhiều thay đổi không tương thích API.

**deployment.yaml** truyền URL kết nối _nats://my-nats:4222_ cho các pod qua biến môi trường _NATS_URL_ như sau:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mapper-dep
spec:
  replicas: 10
  selector:
    matchLabels:
      app: mapper
  template:
    metadata:
      labels:
        app: mapper
    spec:
      containers:
        - name: mapper
          image: jakousa/dwk-app9-mapper:0bcd6794804c367684a9a79bb142bb4455096974
          env:
            - name: NATS_URL
              value: nats://my-nats:4222
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fetcher-dep
spec:
  replicas: 1
  selector:
    matchLabels:
      app: fetcher
  template:
    metadata:
      labels:
        app: fetcher
    spec:
      containers:
        - name: fetcher
          image: jakousa/dwk-app9-fetcher:0bcd6794804c367684a9a79bb142bb4455096974
          env:
            - name: NATS_URL
              value: nats://my-nats:4222
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: saver-dep
spec:
  replicas: 1
  selector:
    matchLabels:
      app: saver
  template:
    metadata:
      labels:
        app: saver
    spec:
      containers:
        - name: saver
          image: jakousa/dwk-app9-saver:0bcd6794804c367684a9a79bb142bb4455096974
          env:
            - name: NATS_URL
              value: nats://my-nats:4222
```

Sau khi apply các deployment, chúng ta có thể kiểm tra mọi thứ hoạt động bằng cách đọc log của fetcher - `kubectl logs fetcher-dep-7d799bb6bf-zz8hr -f`:

```bash
Ready to send #827
Sent data #827, 831 remaining
Ready to send #776
Sent data #776, 830 remaining
Ready to send #516
Sent data #516, 829 remaining
Ready to send #382
Sent data #382, 828 remaining
Ready to send #709
...
```

Chúng ta cũng muốn giám sát trạng thái của NATS. NATS có một [web service](https://docs.nats.io/running-a-nats-service/nats_admin/monitoring) cung cấp nhiều dữ liệu monitoring. Có thể truy cập dịch vụ này từ trình duyệt với `kubectl port-forward my-nats-0 8222:8222` tại <http://localhost:8222>:

<img src="../img/nats-stats.png">

Chúng ta đã dùng Prometheus để giám sát nên cần cấu hình thêm. Đầu tiên phải bật [Prometheus exporter](https://github.com/nats-io/prometheus-nats-exporter), mở metrics ở port 7777. Theo tài liệu chart Helm [tại đây](https://artifacthub.io/packages/helm/bitnami/nats#metrics-parameters), đặt _metrics.enabled_ thành _true_ để bật metrics.

Lệnh [helm upgrade](https://helm.sh/docs/helm/helm_upgrade/) sẽ thực hiện điều này:

```bash
$ helm upgrade --set metrics.enabled=true,auth.enabled=false my-nats oci://registry-1.docker.io/bitnamicharts/nats

  ...
  3. Lấy URL Prometheus Metrics của NATS bằng lệnh:

    echo "Prometheus Metrics URL: http://127.0.0.1:7777/metrics"
    kubectl port-forward --namespace default svc/my-nats-metrics 7777:7777

  4. Truy cập metrics bằng URL trên trình duyệt.
```

Kết quả đã hướng dẫn cách truy cập exporter bằng trình duyệt. Hãy kiểm tra:

```bash
kubectl port-forward --namespace default svc/my-nats-metrics 7777:7777
```

<img src="../img/nats-metrics.png">

Kết nối Prometheus với exporter cần một resource mới [ServiceMonitor](https://github.com/prometheus-operator/prometheus-operator/blob/main/Documentation/user-guides/getting-started.md), là một [Custom Resource Definition](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/) (CRD).

Chúng ta có thể tự tạo resource này, nhưng với một chút cấu hình, chart Helm sẽ làm giúp. Các thiết lập sau sẽ tạo ServiceMonitor:

```
metrics.serviceMonitor.enabled=true
metrics.serviceMonitor.namespace=prometheus
```

Vì đã đặt khá nhiều tham số, hãy khai báo trong một file:

**myvalues.yaml**

```yaml
auth:
  enabled: false

metrics:
  enabled: true
  serviceMonitor:
    enabled: true
    namespace: prometheus
```

Bây giờ hãy nâng cấp chart:

```bash
helm upgrade -f myvalyes.yaml my-nats oci://registry-1.docker.io/bitnamicharts/nats
```

Có thể xác nhận ServiceMonitor _my-nats-metrics_ đã được tạo:

```bash
$ kubectl get servicemonitors.monitoring.coreos.com -n prometheus
NAME                                                        AGE
kube-prometheus-stack-1714-alertmanager                     6d10h
...
my-nats-metrics                                             8m8s
```

Chúng ta cần thêm label phù hợp để Prometheus biết lắng nghe NATS.

Hãy dùng label mà các ServiceMonitor hiện có đang dùng. Kiểm tra bằng lệnh:

```shell
$ kubectl -n prometheus get prometheus
  NAME                                    VERSION   REPLICAS   AGE
  kube-prometheus-stack-1714-prometheus   v2.51.2   1          39h

$ kubectl describe prometheus -n prometheus kube-prometheus-stack-1714-prometheus
...
  Service Monitor Selector:
    Match Labels:
      Release:  kube-prometheus-stack-1714644114
...
```

Vậy label cần là _release: kube-prometheus-stack-1714644114_ trừ khi bạn muốn định nghĩa một resource Prometheus mới. Label này được đặt vào _metrics.service.labels_.

Có thể gán label cho ServiceMonitor bằng lệnh [kubectl label](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_label/):

```bash
kubectl label servicemonitors.monitoring.coreos.com -n prometheus my-nats-metrics release=kube-prometheus-stack-1714644114
```

Bây giờ Prometheus sẽ truy cập được dữ liệu mới. Kiểm tra Prometheus:

```shell
$ kubectl -n prometheus port-forward prometheus-kube-prometheus-stack-1714-prometheus-0 9090
Forwarding from 127.0.0.1:9090 -> 9090
Forwarding from [::1]:9090 -> 9090
```

Nếu mọi thứ ổn, ServiceMonitor NATS sẽ xuất hiện trong danh sách _targets_:

<img src="../img/nats-prome.png">

Bây giờ có thể truy vấn dữ liệu:

<img src="../img/nats-prome2.png">

API của Prometheus cũng sẽ trả về kết quả:

```shell
$ curl http://localhost:9090/api/v1/query\?query\=gnatsd_connz_in_msgs
  {
    "status":"success",
    "data":{
        "resultType":"vector",
        "result":[
          {
              "metric":{
                "__name__":"gnatsd_connz_in_msgs",
                "container":"metrics",
                "endpoint":"tcp-metrics",
                "instance":"10.40.0.138:7777",
                "job":"my-nats-metrics",
                "namespace":"default",
                "pod":"my-nats-0",
                "server_id":"http://localhost:8222",
                "service":"my-nats-metrics"
              },
              "value":[
                1715203297.971,
                "1997"
              ]
          }
        ]
    }
  }
```

Nếu kết quả ở đây rỗng, thì có gì đó sai. Kết quả có thể là success dù truy vấn không hợp lệ.

Bây giờ chỉ cần thêm dashboard Grafana cho dữ liệu này. Hãy import dashboard từ [đây](https://raw.githubusercontent.com/nats-io/prometheus-nats-exporter/5084a32850823b59069f21f3a7dde7e488fef1c6/walkthrough/grafana-nats-dash.json) thay vì tự cấu hình.

```shell
$ kubectl -n prometheus port-forward kube-prometheus-stack-1602180058-grafana-59cd48d794-4459m 3000
```

Tại đây, bạn có thể dán JSON vào "Import dashboard" và chọn Prometheus làm nguồn dữ liệu.

<img src="../img/grafana-import2.png">

Và bây giờ chúng ta đã có dashboard đơn giản với dữ liệu:

<img src="../img/grafana-nats.png">

Đây là cấu hình cuối cùng:

<img src="../img/app9-nats-prometheus-grafana.png">

<exercise name='Bài tập 4.06: Dự án v2.0'>

_Nếu bạn dùng JavaScript, lưu ý [ứng dụng ví dụ](https://github.com/kubernetes-hy/material-example/tree/master/app9) dùng thư viện nats.js phiên bản 1.5. [Phiên bản hiện tại](https://www.npmjs.com/package/nats) của thư viện có nhiều thay đổi lớn về API, nên copy-paste code từ ví dụ sẽ không hoạt động._

Tạo một service riêng để gửi thông báo trạng thái của todos tới một ứng dụng chat phổ biến. Gọi service mới là "broadcaster".

Yêu cầu:

1. Backend lưu hoặc cập nhật todo phải gửi một tin nhắn tới NATS
2. Broadcaster phải subscribe các tin nhắn từ NATS
3. Broadcaster phải gửi tiếp tin nhắn tới **một dịch vụ ngoài** theo định dạng mà dịch vụ đó hỗ trợ

Dịch vụ ngoài có thể là:

- Discord (bạn có thể dùng Discord của khóa học Full stack, xem [tại đây](https://fullstackopen.com/en/part11/expanding_further#exercise-11-18) để biết chi tiết)
- Telegram
- Slack

hoặc nếu không muốn dùng các dịch vụ trên, hãy dùng "Generic" với một URL đặt qua biến môi trường và payload ví dụ như sau:

```json
{
  "user": "bot",
  "message": "A todo was created"
}
```

Broadcaster phải có khả năng scale mà không gửi trùng tin nhắn. Kiểm tra rằng nó có thể chạy với 6 replica mà không gặp vấn đề. Tin nhắn chỉ cần gửi tới dịch vụ ngoài nếu tất cả các service đều hoạt động bình thường. Vì vậy, nếu thỉnh thoảng thiếu tin nhắn cũng không sao, nhưng không được gửi trùng.

Ví dụ một broadcaster hoạt động:
<img src="../img/ex406-solution.gif">

Không được ghi API key dưới dạng văn bản thuần.

</exercise>
