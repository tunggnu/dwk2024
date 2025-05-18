---
path: "/part-1/2-introduction-to-debugging-vi"
title: "Giới thiệu về Debugging"
hidden: false
---

<text-box variant='learningObjectives' name='Mục tiêu học tập'>

Sau phần này bạn

- có thể bắt đầu debug khi có sự cố xảy ra

- biết cách sử dụng Lens để khám phá các tài nguyên Kubernetes

</text-box>

Kubernetes là một hệ thống "tự phục hồi", và chúng ta sẽ quay lại tìm hiểu Kubernetes gồm những gì và hoạt động như thế nào ở phần 5. Nhưng ở giai đoạn này, "tự phục hồi" là một khái niệm tuyệt vời: thông thường, bạn (người bảo trì hoặc lập trình viên) không cần phải làm gì khi có sự cố xảy ra với một pod hoặc container.

Đôi khi bạn cần can thiệp, hoặc bạn có thể gặp vấn đề với cấu hình của chính mình. Khi bạn cố gắng tìm lỗi trong cấu hình, hãy loại trừ từng khả năng một cách tuần tự. Chìa khóa là phải có hệ thống và **luôn đặt câu hỏi về mọi thứ**. Dưới đây là các công cụ sơ bộ để giải quyết vấn đề.

Công cụ đầu tiên là `kubectl describe`, nó có thể cho bạn biết hầu hết mọi thứ bạn cần về bất kỳ tài nguyên nào.

Công cụ thứ hai là `kubectl logs`, cho phép bạn theo dõi log của phần mềm có thể đang bị lỗi.

Công cụ thứ ba là `kubectl delete`, sẽ đơn giản xóa tài nguyên và trong một số trường hợp, như với pod trong deployment, một pod mới sẽ tự động được tạo ra.

Cuối cùng, chúng ta có công cụ tổng quát [Lens "The Kubernetes IDE"](https://k8slens.dev/), bạn nên bắt đầu sử dụng ngay để làm quen với cách dùng.

Trong quá trình làm bài tập, bạn cũng có kênh Discord của chúng tôi để hỗ trợ (bạn đã tham gia ở [phần 0](/part-0)).

Hãy thử các công cụ này và thực hành với Lens. Bạn có thể sẽ gặp thử thách debug thực sự trong các bài tập và sẽ có một thử thách debug khác ở phần 5 khi chúng ta có nhiều thành phần chuyển động hơn.

Hãy triển khai ứng dụng và xem điều gì đang diễn ra.

```console
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes-hy/material-example/master/app1/manifests/deployment.yaml
  deployment.apps/hashgenerator-dep created

$ kubectl describe deployment hashgenerator-dep
  Name:                   hashgenerator-dep
  Namespace:              default
  CreationTimestamp:      Fri, 05 Apr 2024 10:42:30 +0300
  Labels:                 <none>
  Annotations:            deployment.kubernetes.io/revision: 1
  Selector:               app=hashgenerator
  Replicas:               1 desired | 1 updated | 1 total | 1 available | 0 unavailable
  StrategyType:           RollingUpdate
  MinReadySeconds:        0
  RollingUpdateStrategy:  25% max unavailable, 25% max surge
  Pod Template:
    Labels:  app=hashgenerator
    Containers:
     hashgenerator:
      Image:        jakousa/dwk-app1:b7fc18de2376da80ff0cfc72cf581a9f94d10e64
      Port:         <none>
      Host Port:    <none>
      Environment:  <none>
      Mounts:       <none>
    Volumes:        <none>
  Conditions:
    Type           Status  Reason
    ----           ------  ------
    Available      True    MinimumReplicasAvailable
    Progressing    True    NewReplicaSetAvailable
  OldReplicaSets:  <none>
  NewReplicaSet:   hashgenerator-dep-75bdcc94c (1/1 replicas created)
  Events:
    Type    Reason             Age    From                   Message
    ----    ------             ----   ----                   -------
    Normal  ScalingReplicaSet  8m39s  deployment-controller  Scaled up replica set hashgenerator-dep-75bdcc94c to 1
```

Có rất nhiều thông tin mà chúng ta chưa sẵn sàng để đánh giá. Hãy dành chút thời gian đọc qua tất cả. Có ít nhất một vài thông tin quan trọng mà chúng ta biết, chủ yếu vì chúng ta đã định nghĩa chúng trước đó trong yaml. _Events_ thường là nơi bạn nên kiểm tra lỗi.

Lệnh `describe` cũng có thể dùng cho các tài nguyên khác. Hãy xem tiếp pod:

```console
$ kubectl describe pod hashgenerator-dep-75bdcc94c-whwsm
  ...
  Events:
    Type    Reason     Age   From                              Message
    ----    ------     ----  ----                              -------
    Normal  Scheduled  26s   default-scheduler  Successfully assigned default/hashgenerator-dep-7877df98df-qmck9 to k3d-k3s-default-server-0
    Normal  Pulling    15m   kubelet            Pulling image "jakousa/dwk-app1:b7fc18de2376da80ff0cfc72cf581a9f94d10e64"
    Normal  Pulled     26s   kubelet            Container image "jakousa/dwk-app1:b7fc18de2376da80ff0cfc72cf581a9f94d10e64"
    Normal  Created    26s   kubelet            Created container hashgenerator
    Normal  Started    26s   kubelet            Started container hashgenerator
```

Lại có rất nhiều thông tin, nhưng lần này hãy tập trung vào phần events. Ở đây bạn có thể thấy mọi thứ đã xảy ra. Scheduler đã pull image thành công và khởi động container trên node có tên "k3d-k3s-default-server-0". Mọi thứ đang hoạt động như mong đợi, rất tuyệt. Ứng dụng đã chạy.

Tiếp theo, hãy kiểm tra xem ứng dụng thực sự có hoạt động đúng không bằng cách đọc log.

```console
$ kubectl logs hashgenerator-dep-75bdcc94c-whwsm
  jst944
  3c2xas
  s6ufaj
  cq7ka6
```

Mọi thứ có vẻ ổn. Tuy nhiên, sẽ tuyệt hơn nếu có một dashboard để xem mọi thứ đang diễn ra? Hãy xem Lens có thể làm gì.

Đầu tiên, bạn cần thêm cluster vào Lens. Nếu config không có sẵn trong dropdown, bạn có thể lấy kubeconfig cho custom bằng lệnh `kubectl config view --minify --raw`. Sau khi đã thêm cluster, mở tab Workloads/Overview. Một giao diện tương tự như sau sẽ hiện ra

<img src="../img/lens_during_deploy.png">

Ở dưới cùng, chúng ta có thể thấy mọi event, và ở trên cùng, chúng ta thấy trạng thái của các tài nguyên khác nhau trong cluster. Hãy thử xóa và apply lại deployment, bạn sẽ thấy các event xuất hiện trên dashboard. Đây chính là output mà bạn sẽ thấy từ `kubectl get events`

Tiếp theo, hãy chuyển sang tab Workloads/Pods và click vào pod có tên "hashgenerator-dep-...".

<img src="../img/lens_pod.png">

Giao diện này hiển thị cho chúng ta cùng thông tin như trong phần mô tả. Nhưng GUI còn cung cấp các hành động. Ba biểu tượng được đánh số ở góc trên bên phải là:

1. Mở terminal vào container trong pod
2. Xem log
3. Xóa tài nguyên

"Tại sao tôi dùng Lens? Nếu bạn dùng kubectl từ terminal, ví dụ để liệt kê các pod, không có gì đảm bảo rằng các pod đó vẫn còn khi bạn chạy lệnh kubectl tiếp theo. Ngược lại, với Lens, bạn có một giao diện đồ họa cho mọi thứ, và mọi thứ đều cập nhật trực tiếp, nên bạn có thể thấy khi pod hoặc bất kỳ object nào biến mất hoặc có thay đổi xảy ra." - [Matti Paksula](http://github.com/matti)
