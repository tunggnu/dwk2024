---
path: "/part-5/1-kubernetes-internals-vi"
title: "Bên trong Kubernetes"
hidden: false
---

<text-box variant='learningObjectives' name='Mục tiêu học tập'>

Sau phần này bạn

- có thể giải thích tổng quan cách Kubernetes vận hành

</text-box>

Thay vì nghĩ về Kubernetes như một thứ hoàn toàn mới, tôi nhận thấy việc so sánh nó với hệ điều hành sẽ giúp dễ hiểu hơn. Tôi không phải chuyên gia về hệ điều hành nhưng chúng ta đều đã từng sử dụng chúng.

Kubernetes là một lớp phía trên để chạy ứng dụng của chúng ta. Nó lấy các tài nguyên từ các lớp bên dưới và quản lý ứng dụng cũng như tài nguyên của chúng ta. Nó cũng cung cấp các dịch vụ như DNS cho ứng dụng. Với tư duy hệ điều hành này, chúng ta cũng có thể nhìn ngược lại: Có thể bạn đã từng dùng [cron](https://en.wikipedia.org/wiki/Cron) (hoặc [task scheduler](https://en.wikipedia.org/wiki/Windows_Task_Scheduler) của Windows) để lên lịch các tác vụ như sao lưu ứng dụng. Điều tương tự trong Kubernetes có thể thực hiện với [CronJobs](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/).

Bây giờ khi bắt đầu nói về bên trong, chúng ta sẽ học thêm nhiều điều mới về Kubernetes và có thể phòng tránh, giải quyết các vấn đề phát sinh từ bản chất của nó.

Vì phần này chủ yếu lặp lại tài liệu chính thức của Kubernetes, tôi sẽ đính kèm nhiều liên kết tới tài liệu gốc. Dù nói về nội tại, chúng ta sẽ không bàn về cách tự dựng cụm Kubernetes thủ công. Nếu bạn muốn tự tay thiết lập, hãy đọc và hoàn thành [Kubernetes the Hard Way](https://github.com/kelseyhightower/kubernetes-the-hard-way) của Kelsey Hightower.

### Controllers

[Controllers](https://kubernetes.io/docs/concepts/architecture/controller/) theo dõi trạng thái của cụm và cố gắng đưa trạng thái hiện tại về gần trạng thái mong muốn. Khi bạn khai báo X bản sao Pod trong deployment.yaml, một controller gọi là [Replication Controller](https://kubernetes.io/docs/concepts/workloads/controllers/replicationcontroller/) sẽ đảm bảo điều đó thành hiện thực. Có nhiều controller cho các nhiệm vụ khác nhau.

### Kubernetes Control Plane

[Kubernetes Control Plane](https://kubernetes.io/docs/concepts/overview/components/#control-plane-components)
chịu trách nhiệm quản lý cụm Kubernetes. Đây là thành phần điều phối chính đảm bảo trạng thái mong muốn của cụm khớp với trạng thái thực tế. Control plane đưa ra các quyết định toàn cục cho cụm (như lên lịch), cũng như phát hiện và phản hồi các sự kiện (như khởi động pod mới khi số replica của deployment chưa đủ).

Control plane bao gồm:

- etcd

  - Một kho lưu trữ key-value mà Kubernetes dùng để lưu toàn bộ dữ liệu cụm.

- kube-scheduler

  - Quyết định pod sẽ chạy trên node nào.

- kube-controller-manager

  - Chịu trách nhiệm và chạy tất cả các controller.

- kube-apiserver
  - Cung cấp API để truy cập Control Plane của Kubernetes.

Ngoài ra còn có cloud-controller-manager cho phép bạn kết nối cụm với API của nhà cung cấp đám mây. Nếu bạn muốn dựng cụm trên [Hetzner](https://www.hetzner.com/cloud), có thể dùng [hcloud-cloud-controller-manager](https://github.com/hetznercloud/hcloud-cloud-controller-manager) trên các VM của họ.

### Node Components

Mỗi node có một số [thành phần](https://kubernetes.io/docs/concepts/overview/components/#node-components) để duy trì các pod đang chạy.

- kubelet

  - Đảm bảo các container đang chạy trong một Pod

- kube-proxy
  - Proxy mạng và duy trì các quy tắc mạng. Cho phép kết nối trong và ngoài cụm cũng như các Service hoạt động như chúng ta đã dùng.

Mỗi node cũng có [Container Runtime](https://kubernetes.io/docs/setup/production-environment/container-runtimes/). Trong khóa học này, chúng ta dùng Docker làm runtime.

<img src="../img/kube-diag.svg">

### Addons

Ngoài các thành phần trên, Kubernetes còn có [Addons](https://kubernetes.io/docs/concepts/cluster-administration/addons/) sử dụng các resource Kubernetes quen thuộc để mở rộng chức năng. Bạn có thể xem các resource mà addon tạo ra trong namespace `kube-system`.

```console
$ kubectl -n kube-system get all
  NAME                                                            READY   STATUS    RESTARTS   AGE
  pod/event-exporter-v0.2.5-599d65f456-vh4st                      2/2     Running   0          5h42m
  pod/fluentd-gcp-scaler-bfd6cf8dd-kmk2x                          1/1     Running   0          5h42m
  pod/fluentd-gcp-v3.1.1-9sl8g                                    2/2     Running   0          5h41m
  pod/fluentd-gcp-v3.1.1-9wpqh                                    2/2     Running   0          5h41m
  pod/fluentd-gcp-v3.1.1-fr48m                                    2/2     Running   0          5h41m
  pod/heapster-gke-9588c9855-pc4wr                                3/3     Running   0          5h41m
  pod/kube-dns-5995c95f64-m7k4j                                   4/4     Running   0          5h41m
  pod/kube-dns-5995c95f64-rrjpx                                   4/4     Running   0          5h42m
  pod/kube-dns-autoscaler-8687c64fc-xv6p6                         1/1     Running   0          5h41m
  pod/kube-proxy-gke-dwk-cluster-default-pool-700eba89-j735       1/1     Running   0          5h41m
  pod/kube-proxy-gke-dwk-cluster-default-pool-700eba89-mlht       1/1     Running   0          5h41m
  pod/kube-proxy-gke-dwk-cluster-default-pool-700eba89-xss7       1/1     Running   0          5h41m
  pod/l7-default-backend-8f479dd9-jbv9l                           1/1     Running   0          5h42m
  pod/metrics-server-v0.3.1-5c6fbf777-lz2zh                       2/2     Running   0          5h41m
  pod/prometheus-to-sd-jw9rs                                      2/2     Running   0          5h41m
  pod/prometheus-to-sd-qkxvd                                      2/2     Running   0          5h41m
  pod/prometheus-to-sd-z4ssv                                      2/2     Running   0          5h41m
  pod/stackdriver-metadata-agent-cluster-level-5d8cd7b6bf-rfd8d   2/2     Running   0          5h41m

  NAME                           TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)         AGE
  service/default-http-backend   NodePort    10.31.251.116   <none>        80:31581/TCP    5h42m
  service/heapster               ClusterIP   10.31.247.145   <none>        80/TCP          5h42m
  service/kube-dns               ClusterIP   10.31.240.10    <none>        53/UDP,53/TCP   5h42m
  service/metrics-server         ClusterIP   10.31.249.74    <none>        443/TCP         5h42m

  NAME                                      DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR                                                              AGE
  daemonset.apps/fluentd-gcp-v3.1.1         3         3         3       3            3           beta.kubernetes.io/fluentd-ds-ready=true,beta.kubernetes.io/os=linux       5h42m
  daemonset.apps/metadata-proxy-v0.1        0         0         0       0            0           beta.kubernetes.io/metadata-proxy-ready=true,beta.kubernetes.io/os=linux   5h42m
  daemonset.apps/nvidia-gpu-device-plugin   0         0         0       0            0           <none>                                                                     5h42m
  daemonset.apps/prometheus-to-sd           3         3         3       3            3           beta.kubernetes.io/os=linux                                                5h42m

  NAME                                                       READY   UP-TO-DATE   AVAILABLE   AGE
  deployment.apps/event-exporter-v0.2.5                      1/1     1            1           5h42m
  deployment.apps/fluentd-gcp-scaler                         1/1     1            1           5h42m
  deployment.apps/heapster-gke                               1/1     1            1           5h42m
  deployment.apps/kube-dns                                   2/2     2            2           5h42m
  deployment.apps/kube-dns-autoscaler                        1/1     1            1           5h42m
  deployment.apps/l7-default-backend                         1/1     1            1           5h42m
  deployment.apps/metrics-server-v0.3.1                      1/1     1            1           5h42m
  deployment.apps/stackdriver-metadata-agent-cluster-level   1/1     1            1           5h42m

  NAME                                                                  DESIRED   CURRENT   READY   AGE
  replicaset.apps/event-exporter-v0.2.5-599d65f456                      1         1         1       5h42m
  replicaset.apps/fluentd-gcp-scaler-bfd6cf8dd                          1         1         1       5h42m
  replicaset.apps/heapster-gke-58bf4cb5f5                               0         0         0       5h42m
  replicaset.apps/heapster-gke-9588c9855                                1         1         1       5h41m
  replicaset.apps/kube-dns-5995c95f64                                   2         2         2       5h42m
  replicaset.apps/kube-dns-autoscaler-8687c64fc                         1         1         1       5h42m
  replicaset.apps/l7-default-backend-8f479dd9                           1         1         1       5h42m
  replicaset.apps/metrics-server-v0.3.1-5c6fbf777                       1         1         1       5h41m
  replicaset.apps/metrics-server-v0.3.1-8559697b9c                      0         0         0       5h42m
  replicaset.apps/stackdriver-metadata-agent-cluster-level-5d8cd7b6bf   1         1         1       5h41m
  replicaset.apps/stackdriver-metadata-agent-cluster-level-7bd5ddd849   0         0         0       5h42m
```

Để có cái nhìn tổng thể về cách các thành phần giao tiếp với nhau, hãy đọc bài viết [what happens when k8s](https://github.com/jamiehannaford/what-happens-when-k8s) giải thích điều gì xảy ra khi bạn chạy `kubectl run nginx --image=nginx --replicas=3`, giúp làm sáng tỏ thêm những điều kỳ diệu phía sau hậu trường.

## Tự phục hồi (Self-healing)

Ở [phần 1](/part-1), chúng ta đã nói một chút về tính "tự phục hồi" của Kubernetes và cách pod có thể bị xóa và tự động được tạo lại.

Hãy xem điều gì xảy ra nếu chúng ta xóa một node đang chạy pod. Đầu tiên hãy triển khai pod, một ứng dụng web với ingress từ phần 1, xác nhận nó đang chạy và xem pod đó đang chạy trên node nào.

```console
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes-hy/material-example/master/app2/manifests/deployment.yaml \
                -f https://raw.githubusercontent.com/kubernetes-hy/material-example/master/app2/manifests/ingress.yaml \
                -f https://raw.githubusercontent.com/kubernetes-hy/material-example/master/app2/manifests/service.yaml
  deployment.apps/hashresponse-dep created
  ingress.extensions/dwk-material-ingress created
  service/hashresponse-svc created

$ curl localhost:8081
9eaxf3: 8k2deb

$ kubectl describe po hashresponse-dep-57bcc888d7-5gkc9 | grep 'Node:'
  Node:         k3d-k3s-default-agent-1/172.30.0.2
```

Trong trường hợp này pod chạy trên agent-1. Hãy làm node này "offline" bằng lệnh pause:

```console
$ docker ps
  CONTAINER ID        IMAGE                      COMMAND                  CREATED             STATUS              PORTS                                           NAMES
  5c43fe0a936e        rancher/k3d-proxy:v3.0.0   "/bin/sh -c nginx-pr…"   10 days ago         Up 2 hours          0.0.0.0:8081->80/tcp, 0.0.0.0:50207->6443/tcp   k3d-k3s-default-serverlb
  fea775395132        rancher/k3s:latest         "/bin/k3s agent"         10 days ago         Up 2 hours                                                          k3d-k3s-default-agent-1
  70b68b040360        rancher/k3s:latest         "/bin/k3s agent"         10 days ago         Up 2 hours          0.0.0.0:8082->30080/tcp                         k3d-k3s-default-agent-0
  28cc6cef76ee        rancher/k3s:latest         "/bin/k3s server --t…"   10 days ago         Up 2 hours                                                          k3d-k3s-default-server-0

$ docker pause k3d-k3s-default-agent-1
k3d-k3s-default-agent-1
```

Bây giờ chờ một lúc và trạng thái mới sẽ như sau:

```console
$ kubectl get po
NAME                                READY   STATUS        RESTARTS   AGE
hashresponse-dep-57bcc888d7-5gkc9   1/1     Terminating   0          15m
hashresponse-dep-57bcc888d7-4klvg   1/1     Running       0          30s

$ curl localhost:8081
6xluh2: ta0ztp
```

Chuyện gì vừa xảy ra vậy? Đọc [giải thích này về cách kubernetes xử lý node offline](https://dev.to/duske/how-kubernetes-handles-offline-nodes-53b5)

Vậy nếu bạn xóa node control-plane duy nhất thì sao? Không có gì tốt cả. Trong cụm cục bộ của chúng ta, đó là điểm lỗi duy nhất. Xem tài liệu Kubernetes về "[Các lựa chọn cho kiến trúc Highly Available](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/ha-topology/)" để tránh cả cụm bị tê liệt chỉ vì một phần cứng hỏng.
