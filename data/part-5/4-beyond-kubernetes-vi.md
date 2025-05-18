---
path: "/part-5/4-beyond-kubernetes-vi"
title: "Vượt ra ngoài Kubernetes"
hidden: false
---

<text-box variant='learningObjectives' name='Mục tiêu học tập'>

Sau phần này bạn có thể

- Liệt kê và so sánh các lựa chọn nền tảng.

- Thiết lập một nền tảng serverless (Knative) và triển khai một workload đơn giản

</text-box>

Cuối cùng, vì Kubernetes là một nền tảng, chúng ta sẽ điểm qua một số thành phần xây dựng phổ biến sử dụng Kubernetes.

<blockquote class="twitter-tweet"><p lang="en" dir="ltr">Kubernetes là một nền tảng để xây dựng các nền tảng khác. Nó là điểm khởi đầu tốt hơn; không phải là đích đến cuối cùng.</p>&mdash; Kelsey Hightower (@kelseyhightower) <a href="https://twitter.com/kelseyhightower/status/935252923721793536?ref_src=twsrc%5Etfw">November 27, 2017</a></blockquote>

[OpenShift](https://www.openshift.com/) là một Kubernetes "doanh nghiệp" ([Red Hat OpenShift Overview](https://developers.redhat.com/products/openshift/overview)). Nói rằng bạn không có Kubernetes vì bạn dùng OpenShift cũng giống như nói ["Tôi không có động cơ. Tôi có một chiếc xe hơi!"](https://www.openshift.com/blog/enterprise-kubernetes-with-openshift-part-one). Để xem các lựa chọn Kubernetes sẵn sàng cho sản xuất khác, hãy xem [Rancher](https://rancher.com/), có thể bạn đã thấy ở trang này [https://github.com/rancher/k3d](https://github.com/rancher/k3d), và [Anthos GKE](https://cloud.google.com/anthos/gke), nghe cũng quen thuộc. Tất cả đều là các lựa chọn khi bạn phải quyết định chọn bản phân phối Kubernetes nào hoặc muốn dùng dịch vụ được quản lý.

<exercise name='Bài tập 5.05: So sánh nền tảng'>

Chọn một nhà cung cấp dịch vụ như Rancher và so sánh với một nhà cung cấp khác như OpenShift.

Tùy ý quyết định nhà cung cấp nào "tốt hơn" và lập luận vì sao nó tốt hơn nhà cung cấp còn lại.

Chỉ cần nộp danh sách gạch đầu dòng là đủ.

</exercise>

### Serverless

[Serverless](https://en.wikipedia.org/wiki/Serverless_computing) đã trở nên rất phổ biến và dễ hiểu lý do tại sao. Dù là Google Cloud Run, Knative, OpenFaaS, OpenWhisk, Fission hay Kubeless thì chúng đều chạy trên Kubernetes, hoặc ít nhất là có thể chạy trên đó. Nền tảng serverless càng cũ thì càng có khả năng không chạy trên Kubernetes. Vì vậy, nói "Kubernetes cạnh tranh với serverless" là không hợp lý.

Vì đây không phải là khóa học về serverless nên chúng ta sẽ không đi sâu, nhưng serverless nghe thật hấp dẫn. Đó là lý do tiếp theo chúng ta sẽ thiết lập một nền tảng serverless trên k3d. Ở đây, chúng ta chọn [Knative](https://knative.dev/) vì đây là giải pháp mà [Google Cloud Run](https://cloud.google.com/blog/products/serverless/knative-based-cloud-run-services-are-ga) dựa vào và có vẻ là lựa chọn mã nguồn mở tốt so với các lựa chọn khác. Nó cũng phù hợp với chủ đề "nền tảng cho nền tảng" vì bạn có thể dùng nó để tạo nền tảng serverless của riêng mình.

Knative có một [runtime contract](https://github.com/knative/specs/blob/main/specs/serving/runtime-contract.md) do cộng đồng hỗ trợ. Nó mô tả các tính năng mà một ứng dụng cần có để chạy đúng với vai trò FaaS. Yêu cầu quan trọng là ứng dụng phải stateless và cấu hình được qua biến môi trường. Đặc tả mã nguồn mở kiểu này giúp dự án được chấp nhận rộng rãi hơn. Ví dụ, [Google Cloud Run đã triển khai](https://ahmet.im/blog/cloud-run-is-a-knative/) cùng hợp đồng này.

<exercise name='Bài tập 5.06: Thử Serverless'>

Cài đặt thành phần Knative Serving vào cụm k3d của bạn.

Để Knative hoạt động trên k3d, bạn cần tạo cụm không có Traefik:

```console
$ k3d cluster create --port 8082:30080@agent:0 -p 8081:80@loadbalancer --agents 2 --k3s-arg "--disable=traefik@server:0"
```

Sau đó làm theo [hướng dẫn này](https://knative.dev/docs/install/yaml-install/serving/install-serving-with-yaml/).

Bạn có thể gặp tình huống như sau ở bước _verify the installation_:

```bash
$ kubectl get pods -n knative-serving
NAME                                      READY   STATUS             RESTARTS      AGE
activator-67855958d-w2ws8                 0/1     Running            0             64s
autoscaler-5ff4c5d679-54l28               0/1     Running            0             64s
webhook-5446675b97-2ngh6                  0/1     CrashLoopBackOff   3 (12s ago)   64s
net-kourier-controller-58b6bf4fbc-g7dlp   0/1     CrashLoopBackOff   3 (10s ago)   55s
controller-6d8b579f9-p42dx                0/1     CrashLoopBackOff   3 (6s ago)    64s
```

Xem log của pod bị crash để biết cách khắc phục.

Tiếp theo, hãy thử các ví dụ trong [Deploying a Knative Service](https://knative.dev/docs/getting-started/first-service/), [Autoscaling](https://knative.dev/docs/getting-started/first-autoscale/) và [Traffic splitting](https://knative.dev/docs/getting-started/first-traffic-split/).

Lưu ý bạn có thể truy cập service từ máy host như sau:

```bash
curl -H "Host: hello.default.192.168.240.3.sslip.io" http://localhost:8081
```

Trong đó _Host_ là URL bạn lấy bằng lệnh sau:

```bash
kubectl get ksvc
```

</exercise>

<exercise name='Bài tập 5.07: Triển khai lên Serverless'>

Biến ứng dụng Ping-pong thành serverless.

Đọc [tài liệu này](https://knative.dev/docs/serving/convert-deployment-to-knative-service/) có thể hữu ích.

GỢI Ý: Ứng dụng của bạn nên lắng nghe trên port 8080 hoặc tốt nhất là có biến môi trường `PORT` để cấu hình port này.

</exercise>

<exercise name='Bài tập 5.08: Landscape'>

Xem sơ đồ CNCF Cloud Native Landscape [png](https://landscape.cncf.io/images/landscape.png) (cũng có bản [tương tác](https://landscape.cncf.io/))

Khoanh tròn logo của mọi sản phẩm / dự án bạn đã từng sử dụng. Không nhất thiết phải trong khóa học này. "Đã sử dụng" ở đây nghĩa là bạn biết chắc mình đã dùng nó. Sau đó dùng màu khác để khoanh tròn những cái mà thứ bạn dùng phụ thuộc vào, trừ những cái đã khoanh trước đó. Sau đó lập danh sách thông tin về nơi chúng được sử dụng. Bất cứ thứ gì ngoài phạm vi khóa học này có thể ghi là "ngoài khóa học".

Ví dụ:

1. Tôi đã dùng **HELM** để cài Prometheus ở phần 2.
2. Tôi đã gián tiếp dùng **Flannel** vì k3d (thông qua k3s) sử dụng nó. Nhưng tôi không biết nó hoạt động thế nào.
3. Tôi đã dùng **Istio** ngoài khóa học.

Bạn có thể truy vết gián tiếp sâu bao nhiêu tùy thích, như ví dụ k3d -> k3s -> flannel, nhưng hãy dùng lý trí để hình ảnh cuối cùng có ý nghĩa.

</exercise>
