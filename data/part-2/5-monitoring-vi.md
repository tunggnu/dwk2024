---
path: "/part-2/5-monitoring-vi"
title: "Giám sát"
hidden: false
---

<text-box variant='learningObjectives' name='Mục tiêu học tập'>

Sau phần này bạn

- biết về nhiều công cụ giám sát và có thể triển khai logging đơn giản

- biết về CRD hoặc Custom Resource Definition

- biết về Helm, trình quản lý gói cho Kubernetes

</text-box>

Cụm của chúng ta và các ứng dụng trong đó cho đến giờ vẫn khá giống một "hộp đen". Chúng ta triển khai mọi thứ và chỉ hy vọng mọi thứ hoạt động ổn. Bây giờ, chúng ta sẽ sử dụng [Prometheus](https://prometheus.io/) để giám sát cụm và [Grafana](https://grafana.com/) để xem dữ liệu.

Trước khi bắt đầu, hãy tìm hiểu cách quản lý ứng dụng Kubernetes dễ dàng hơn. [Helm](https://helm.sh/) sử dụng một định dạng đóng gói gọi là _chart_ để định nghĩa các phụ thuộc của ứng dụng. Ngoài ra, Helm Chart còn chứa thông tin về phiên bản chart, yêu cầu của ứng dụng như phiên bản Kubernetes cũng như các chart khác mà nó phụ thuộc.

Hướng dẫn cài đặt Helm có tại [đây](https://helm.sh/docs/intro/install/).

Sau khi cài đặt, chúng ta có thể thêm repository chart chính thức:

```shell
$ helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
$ helm repo add stable https://charts.helm.sh/stable
```

Tiếp theo, chúng ta có thể cài đặt [kube-prometheus-stack](https://artifacthub.io/packages/helm/prometheus-community/kube-prometheus-stack). Mặc định, mọi thứ sẽ được cài vào namespace mặc định. Hãy tạo một namespace mới và cài vào đó.

```shell
$ kubectl create namespace prometheus
$ helm install prometheus-community/kube-prometheus-stack --generate-name --namespace prometheus
...
NAME: kube-prometheus-stack-1635945330
LAST DEPLOYED: Wed Nov  3 15:15:37 2021
NAMESPACE: prometheus
STATUS: deployed
REVISION: 1
NOTES:
kube-prometheus-stack đã được cài đặt. Kiểm tra trạng thái bằng lệnh:
  kubectl --namespace prometheus get pods -l "release=kube-prometheus-stack-1635945330"
```

Điều này đã thêm rất nhiều thứ vào cụm của chúng ta. Trong đó có một số [custom resource](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/). Đây là cách mở rộng API của Kubernetes để cung cấp các tài nguyên mới mà Kubernetes không hỗ trợ sẵn. Chúng ta sẽ tự thiết kế custom resource ở [phần 5](https://devopswithkubernetes.com/part5/).

Bạn có thể xóa gần như mọi thứ với `helm delete [name]` với tên lấy từ lệnh `helm list -n prometheus`. Custom Resource Definition sẽ còn lại và phải xóa thủ công nếu cần. Các định nghĩa này không làm gì cả nên để lại cũng không sao.

Hãy mở đường truy cập vào Grafana để xem dữ liệu.

```shell
$ kubectl get po -n prometheus
 NAME                                                              READY   STATUS    RESTARTS   AGE
 kube-prometheus-stack-1602180058-prometheus-node-exporter-nt8cp   1/1     Running   0          53s
 kube-prometheus-stack-1602180058-prometheus-node-exporter-ft7dg   1/1     Running   0          53s
 kube-prometheus-stack-1602-operator-557c9c4f5-wbsqc               2/2     Running   0          53s
 kube-prometheus-stack-1602180058-prometheus-node-exporter-tr7ns   1/1     Running   0          53s
 kube-prometheus-stack-1602180058-kube-state-metrics-55dccdkkz6w   1/1     Running   0          53s
 alertmanager-kube-prometheus-stack-1602-alertmanager-0            2/2     Running   0          35s
 kube-prometheus-stack-1602180058-grafana-59cd48d794-4459m         2/2     Running   0          53s
 prometheus-kube-prometheus-stack-1602-prometheus-0                3/3     Running   1          23s

$ kubectl -n prometheus port-forward kube-prometheus-stack-1602180058-grafana-59cd48d794-4459m 3000
  Forwarding from 127.0.0.1:3000 -> 3000
  Forwarding from [::1]:3000 -> 3000
```

Truy cập [http://localhost:3000](http://localhost:3000) bằng trình duyệt và dùng tài khoản admin / prom-operator. Ở góc trên bên trái bạn có thể duyệt các dashboard khác nhau:

<img src="../img/grafana1.png">

Các dashboard đã hiển thị rất nhiều thông tin thú vị về cụm nhưng chúng ta muốn biết thêm về các ứng dụng đang chạy. Hãy thêm [Loki](https://grafana.com/oss/loki/) để xem log.

Để xác nhận mọi thứ hoạt động, chúng ta nên có một ứng dụng sẽ xuất log ra stdout. Hãy chạy ứng dụng Redis từ trước bằng cách apply [file này](https://raw.githubusercontent.com/kubernetes-hy/material-example/master/app5/manifests/statefulset.yaml). Giữ nó chạy để tạo ra nhiều log cho chúng ta.

[Loki-stack Chart](https://github.com/grafana/helm-charts/tree/main/charts/loki-stack) bao gồm mọi thứ cần thiết:

```shell
$ helm repo add grafana https://grafana.github.io/helm-charts
$ helm repo update
$ kubectl create namespace loki-stack
  namespace/loki-stack created

$ helm upgrade --install loki --namespace=loki-stack grafana/loki-stack

$ kubectl get all -n loki-stack
  NAME                      READY   STATUS    RESTARTS   AGE
  pod/loki-promtail-n2fgs   1/1     Running   0          18m
  pod/loki-promtail-h6xq2   1/1     Running   0          18m
  pod/loki-promtail-8l84g   1/1     Running   0          18m
  pod/loki-0                1/1     Running   0          18m

  NAME                    TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
  service/loki            ClusterIP   10.43.170.68   <none>        3100/TCP   18m
  service/loki-headless   ClusterIP   None           <none>        3100/TCP   18m

  NAME                           DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
  daemonset.apps/loki-promtail   3         3         3       3            3           <none>          18m

  NAME                    READY   AGE
  statefulset.apps/loki   1/1     18m
```

Ở đây ta thấy Loki đang chạy ở cổng 3100. Ngoài ra, vì cài đặt loki-stack, chúng ta đã có [Promtail](https://grafana.com/docs/loki/latest/clients/promtail/), giúp gửi log từ ứng dụng đến Loki rất dễ dàng. Thực tế, chúng ta không cần làm gì ngoài việc cấu hình Grafana để hiển thị Loki.

Mở Grafana, vào phần cài đặt, chọn _Connections_, _Data Sources_, rồi _Add data source_. Chọn Loki và nhập đúng URL. Từ thông tin trên, ta đoán cổng là 3100, namespace là loki-stack và tên service là loki. Vậy URL sẽ là http://loki.loki-stack:3100. Không cần thay đổi trường nào khác.

Giờ bạn có thể dùng tab Explore (biểu tượng la bàn) để khám phá dữ liệu.

<img src="../img/loki_app_redisapp.png">

<exercise name='Exercise 2.10: Project v1.3'>

Project thực sự cần có logging.

Thêm logging cho các request để bạn có thể giám sát mọi _todo_ gửi đến backend.

Đặt giới hạn 140 ký tự cho todo ở backend. Dùng Postman hoặc curl để kiểm tra todo quá dài bị chặn và bạn có thể thấy thông báo không hợp lệ trong Grafana.

</exercise>

### Cách dễ nhất

Có một cách dễ hơn để cài Prometheus chỉ với vài cú nhấp chuột. Nếu bạn cần cài lại, hãy thử cách này:

1. Mở [Lens](https://k8slens.dev/)
2. Nhấp chuột phải vào biểu tượng cụm ở góc trên bên trái và chọn "Settings"
3. Kéo xuống phần "Features" dưới "Metrics" và nhấn "Install"

Đây là lựa chọn tuyệt vời cho cụm cục bộ hoặc cụm cá nhân của bạn.
