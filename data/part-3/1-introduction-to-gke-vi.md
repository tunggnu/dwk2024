---
path: "/part-3/1-introduction-to-gke-vi"
title: "Giới thiệu về Google Kubernetes Engine"
hidden: false
---

<text-box variant='learningObjectives' name='Mục tiêu học tập'>

Sau phần này bạn có thể

- Tạo cụm (cluster) của riêng bạn trên GKE

- Triển khai ứng dụng lên Google Kubernetes Engine

</text-box>

Chúng ta đã sử dụng bản phân phối Kubernetes là K3s bằng các container Docker thông qua k3d. Trong môi trường sản xuất, gánh nặng duy trì một cụm Kubernetes thường được giao cho bên thứ ba. Một dịch vụ _Kubernetes được quản lý_ thường là lựa chọn tốt nhất vì công việc bổ sung trong bảo trì vượt quá lợi ích của việc tự vận hành cụm. Trong một số trường hợp khá hiếm, việc tự thiết lập và duy trì cụm Kubernetes là hợp lý. Một ví dụ là công ty/tổ chức của bạn đã có sẵn phần cứng và/hoặc muốn độc lập với các nhà cung cấp, ví dụ như một trường Đại học. Một lý do chính khác để tự vận hành cụm Kubernetes là các quy định pháp lý không cho phép các lựa chọn khác.

Tự duy trì cụm Kubernetes là một cách để tăng chi phí. Cách khác là vận hành các máy chủ không sử dụng đến. Để tránh chi phí này, các giải pháp [Serverless](https://en.wikipedia.org/wiki/Serverless_computing) có thể tiết kiệm hơn. Một cụm Kubernetes có kích thước không phù hợp có thể trở nên rất tốn kém rất nhanh. Một lựa chọn tuyệt vời để bắt đầu là môi trường cao cấp cho các workload serverless, như Google Cloud Run hoặc AWS Fargate. Ở đó bạn có thể chạy bất kỳ container nào, so với các giải pháp cũ như Cloud Functions hoặc AWS Lambda, vốn có hỗ trợ rất khác nhau cho các ngôn ngữ, môi trường và công cụ.

<!---
<text-box variant='hint' name='Đăng ký cho sinh viên Đại học Mở'>
  Nếu bạn là sinh viên tại Phần Lan, có số an sinh xã hội Phần Lan hoặc là sinh viên Đại học Helsinki, hãy đăng ký khóa học ngay. Làm theo <a href="/registration-and-completion">hướng dẫn</a> để nhận Google Cloud Platform Education Grant.
</text-box>
-->

Hãy tập trung vào chi phí của Google Kubernetes Engine (GKE) trước. Lưu ý rằng GKE đắt hơn một chút so với các đối thủ.

Công cụ tính giá tại đây [https://cloud.google.com/products/calculator](https://cloud.google.com/products/calculator) cho chúng ta cái nhìn về giá cả. Tôi đã thử một lựa chọn rẻ: 5 node trong 1 cụm zonal, mỗi node 1 vCPU. Địa điểm trung tâm dữ liệu ở Phần Lan và tôi không cần ổ đĩa lưu trữ lâu dài. Nếu chúng ta muốn ít hơn 5 node thì tại sao lại dùng Kubernetes? Tổng chi phí cho ví dụ này khoảng 250 USD mỗi tháng. Thêm các dịch vụ bổ sung như Load balancer sẽ tăng chi phí. Nếu bạn thấy việc tính phí của Google Cloud Platform khó hiểu, bạn không đơn độc: Coursera có khóa học ~5 giờ "[Hiểu về chi phí Google Cloud Platform của bạn](https://www.coursera.org/learn/gcp-cost-management)".

Trong phần 3 này, chúng ta sẽ sử dụng GKE bằng các khoản tín dụng miễn phí do Google cung cấp. Bạn có trách nhiệm đảm bảo rằng tín dụng đủ dùng cho cả phần này và nếu dùng hết, tôi không thể giúp bạn.

Sau khi nhận tín dụng, chúng ta có thể tạo một dự án với tài khoản thanh toán. Giao diện Google Cloud có thể gây nhầm lẫn. Trên [trang tài nguyên](https://console.cloud.google.com/cloud-resource-manager) chúng ta có thể tạo một dự án mới và đặt tên là "dwk-gke" cho mục đích của khóa học này.

Cài đặt Google Cloud SDK. Hướng dẫn [tại đây](https://cloud.google.com/sdk/install). Sau đó đăng nhập và đặt dự án vừa tạo làm dự án sử dụng.

```console
$ gcloud -v
  Google Cloud SDK 471.0.0
  bq 2.1.3
  core 2024.03.29
  gcloud-crc32c 1.0.0
  gsutil 5.27

$ gcloud auth login
  ...
  Bạn đã đăng nhập thành công

$ gcloud config set project dwk-gke
  Đã cập nhật thuộc tính [core/project].
```

Bây giờ chúng ta có thể tạo một cụm, với lệnh [gcloud container clusters create](https://cloud.google.com/sdk/gcloud/reference/container/clusters/create). Bạn có thể chọn bất kỳ vùng nào từ danh sách [tại đây](https://cloud.google.com/about/locations/). Tôi chọn Phần Lan. Lưu ý rằng một khu vực (ví dụ europe-north1) có thể có nhiều vùng (ví dụ -a). Hãy thêm một tham số nữa: `--cluster-version=1.29`. Tham số này sẽ yêu cầu GKE sử dụng phiên bản sẽ là mặc định vào tháng 4/2024. Lịch phát hành GKE xem [tại đây](https://cloud.google.com/kubernetes-engine/docs/release-schedule).

```console
$ gcloud container clusters create dwk-cluster --zone=europe-north1-b --cluster-version=1.29 --disk-size=32 --num-nodes=3 --machine-type=e2-micro

ERROR: (gcloud.container.clusters.create) ResponseError: code=403, message=Kubernetes Engine API chưa được sử dụng trong dự án dwk-gke-xxxxxx hoặc đã bị vô hiệu hóa. Kích hoạt nó bằng cách truy cập https://console.developers.google.com/apis/api/container.googleapis.com/overview?project=dwk-gke-xxxxxx rồi thử lại.
```

Bạn có thể truy cập liên kết được cung cấp và bật Kubernetes Engine API, lưu ý rằng url được xuất ra là dành riêng cho tên dự án của bạn. Hoặc, bạn chỉ cần thực hiện lệnh sau trên dòng lệnh:

```console
$ gcloud services enable container.googleapis.com
  Hoạt động "operations/acf.p2-385245615727-2f855eed-e785-49ac-91da-896925a691ab" đã hoàn thành thành công.

$ gcloud container clusters create dwk-cluster --zone=europe-north1-b --cluster-version=1.29
  ...
  Đang tạo cụm dwk-cluster tại europe-north1-b...
  ...
  Đã tạo kubeconfig cho dwk-cluster.
  NAME         LOCATION         MASTER_VERSION   MASTER_IP       MACHINE_TYPE  NODE_VERSION     NUM_NODES  STATUS
  dwk-cluster  europe-north1-b  1.29.8-gke.2200  35.228.176.118  e2-medium     1.29.8-gke.2200  3          RUNNING
```

Nếu lệnh không hoạt động, bạn cần cài đặt _gke-gcloud-auth-plugin_ bằng cách làm theo [hướng dẫn này](https://cloud.google.com/kubernetes-engine/docs/how-to/cluster-access-for-kubectl#install_plugin).

Nó đã đặt kubeconfig trỏ đúng hướng. Nhưng nếu cần làm lại, chúng ta có thể đặt kubeconfig như sau:

```console
$ gcloud container clusters get-credentials dwk-cluster --zone=europe-north1-b
  Đang lấy endpoint và dữ liệu xác thực của cụm.
  Đã tạo kubeconfig cho dwk-cluster.
```

Bạn có thể kiểm tra thông tin cụm với `kubectl cluster-info` để xác nhận nó đang trỏ đúng.

### Triển khai lên GKE

Cụm mà chúng ta có bây giờ _gần như_ giống với cụm chạy cục bộ. Hãy áp dụng [ứng dụng này](https://github.com/kubernetes-hy/material-example/tree/master/app6) tạo một chuỗi ngẫu nhiên và sau đó phục vụ một hình ảnh dựa trên chuỗi ngẫu nhiên đó. Điều này sẽ tạo 6 bản sao của tiến trình "seedimage".

```console
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes-hy/material-example/e11a700350aede132b62d3b5fd63c05d6b976394/app6/manifests/deployment.yaml
```

Việc expose service là nơi bắt đầu có sự khác biệt. Thay vì Ingress, chúng ta sẽ dùng service kiểu [LoadBalancer](https://kubernetes.io/docs/concepts/services-networking/service/#loadbalancer). Lưu ý bước tiếp theo sẽ làm tăng [chi phí của cụm](https://cloud.google.com/compute/all-pricing#lb).

Áp dụng file sau:

**service.yaml**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: seedimage-svc
spec:
  type: LoadBalancer # Đây là phần duy nhất có thể lạ
  selector:
    app: seedimage
  ports:
    - port: 80
      protocol: TCP
      targetPort: 3000
```

Một service kiểu load balancer sẽ yêu cầu Google cung cấp cho chúng ta một load balancer. Chúng ta có thể chờ đến khi service nhận được IP bên ngoài:

```console
$ kubectl get svc --watch
  NAME            TYPE           CLUSTER-IP      EXTERNAL-IP    PORT(S)        AGE
  kubernetes      ClusterIP      10.31.240.1     <none>         443/TCP        144m
  seedimage-svc   LoadBalancer   10.31.241.224   35.228.41.16   80:30215/TCP   94s
```

Nếu bây giờ truy cập http://35.228.41.16 bằng trình duyệt, chúng ta sẽ thấy ứng dụng đang chạy. Làm mới trang sẽ thấy load balancer đôi khi trả về hình ảnh khác nhau. Lưu ý rằng chỉ có cổng 80, nên https sẽ không hoạt động.

<div class="highlight-box" markdown="1">
Để tránh tiêu tốn tín dụng, hãy xóa cụm khi không cần dùng

```console
$ gcloud container clusters delete dwk-cluster --zone=europe-north1-b
```

Và khi tiếp tục, hãy tạo lại cụm.

```console
$ gcloud container clusters create dwk-cluster --zone=europe-north1-b --cluster-version=1.29
```

Đóng cụm cũng sẽ xóa mọi thứ bạn đã triển khai trên cụm. Vì vậy nếu bạn nghỉ vài ngày trong lúc làm bài tập, có thể phải làm lại. May mắn là chúng ta dùng cách tiếp cận khai báo nên chỉ cần áp dụng lại các file yaml là tiếp tục được.

### Lưu trữ dữ liệu trên GKE

Google Kubernetes Engine sẽ tự động cung cấp một ổ đĩa lâu dài cho [PersistentVolumeClaim](https://kubernetes.io/docs/concepts/storage/persistent-volumes/) - chỉ cần không đặt storage class. Nếu muốn, bạn có thể đọc thêm [tại đây](https://cloud.google.com/kubernetes-engine/docs/concepts/persistent-volumes).

<exercise name='Bài tập 3.01: Pingpong GKE'>

Triển khai ứng dụng Ping-pong lên GKE.

Trong bài tập này, hãy sử dụng service kiểu LoadBalancer để expose service.

Nếu log của Postgres báo

```console
initdb: error: directory "/var/lib/postgresql/data" exists but is not empty
It contains a lost+found directory, perhaps due to it being a mount point.
Using a mount point directly as the data directory is not recommended.
Create a subdirectory under the mount point.
```

bạn có thể thêm cấu hình _subPath_:

**statefulset.yaml**

```yaml
# ...
volumeMounts:
  - name: data
    mountPath: /var/lib/postgresql/data
    subPath: postgres
# ...
```

Điều này sẽ tạo thư mục Postgres nơi dữ liệu sẽ được lưu. subPath cũng giúp dùng một volume cho nhiều mục đích.

</exercise>

### Từ Service sang Ingress

Service khá đơn giản. Hãy thử dùng Ingress vì nó cung cấp thêm công cụ đổi lại sự phức tạp.

Service kiểu _NodePort_ là bắt buộc khi dùng Ingress trên GKE. Dù là NodePort, GKE không expose nó ra ngoài cụm. Hãy kiểm tra điều này bằng cách tiếp tục ví dụ trước.

**service.yaml**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: seedimage-svc
spec:
  type: NodePort
  selector:
    app: seedimage
  ports:
    - port: 80
      protocol: TCP
      targetPort: 3000
```

**ingress.yaml**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: seedimage-ing
spec:
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: seedimage-svc
                port:
                  number: 80
```

Khi bạn đã áp dụng các file này, bạn có thể xem port từ danh sách ingress:

```
$ kubectl get ing
  NAME            CLASS    HOSTS   ADDRESS         PORTS   AGE
  seedimage-ing   <none>   *       34.120.61.234   80      2m13s
```

Bây giờ địa chỉ ở đây sẽ là cách truy cập ứng dụng. Sẽ mất một lúc để triển khai, phản hồi có thể là 404 hoặc 502 khi nó đang khởi tạo. Ingress thực hiện kiểm tra sức khỏe bằng cách gửi GET tới / và mong đợi phản hồi HTTP 200.

<exercise name='Bài tập 3.02: Quay lại với Ingress'>

Triển khai các ứng dụng "Log output" và "Ping-pong" lên GKE và expose chúng bằng Ingress.

"Ping-pong" sẽ phải phản hồi tại đường dẫn /pingpong. Điều này có thể yêu cầu bạn chỉnh sửa lại một số phần của mã nguồn.

Lưu ý rằng Ingress mong đợi một service trả về phản hồi thành công tại đường dẫn / ngay cả khi service đó được ánh xạ tới đường dẫn khác!

</exercise>
