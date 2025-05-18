---
path: "/part-2/4-statefulsets-and-jobs-vi"
title: "StatefulSet và Job"
hidden: false
---

<text-box variant='learningObjectives' name='Mục tiêu học tập'>

Sau phần này bạn

- có thể triển khai một ứng dụng có trạng thái (stateful), ví dụ như cơ sở dữ liệu, lên Kubernetes

- biết cách sử dụng job và cronjob để thực thi các tác vụ định kỳ hoặc một lần

</text-box>

## StatefulSet

Ở [phần 1](/part-1/4-introduction-to-storage) chúng ta đã học cách sử dụng volume với PersistentVolume và PersistentVolumeClaim. Chúng ta đã dùng _Deployment_ với chúng và mọi thứ hoạt động tốt cho mục đích kiểm thử. Nếu chỉ có một pod trong deployment thì không vấn đề gì. Khi mở rộng (scaling), mọi thứ có thể khác. Vấn đề là _Deployment_ tạo và mở rộng các pod là _bản sao_ (replica) - chúng là các bản sao mới của cùng một container chạy song song. Vì vậy, volume sẽ được chia sẻ bởi tất cả các pod trong deployment đó. Với volume chỉ đọc thì không sao, nhưng với volume có quyền đọc-ghi, điều này có thể gây ra vấn đề và tệ nhất là gây hỏng dữ liệu.

[StatefulSet](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/) tương tự như _Deployment_ nhưng đảm bảo rằng nếu một pod chết thì pod thay thế sẽ giống hệt, với cùng tên và nhận diện mạng. Ngoài ra, nếu pod được mở rộng, mỗi bản sao sẽ có vùng lưu trữ riêng. Vì vậy StatefulSet dành cho _ứng dụng có trạng thái_, nơi trạng thái được lưu bên trong ứng dụng, không phải bên ngoài, ví dụ như cơ sở dữ liệu. Bạn có thể dùng StatefulSet để mở rộng server game cần lưu trạng thái, như server Minecraft. Hoặc chạy một cơ sở dữ liệu. Để đảm bảo an toàn dữ liệu khi xóa, StatefulSet sẽ không xóa volume mà nó liên kết.

<text-box name="ReplicaSets" variant="hint">
Deployment tạo pod bằng một tài nguyên gọi là "ReplicaSet". Chúng ta đang sử dụng ReplicaSet thông qua Deployment nên chưa cần nói nhiều về nó.
</text-box>

Hãy chạy cơ sở dữ liệu key-value [Redis](https://redis.io) và lưu một số dữ liệu vào đó. Chúng ta sẽ cần một PersistentVolume cũng như một ứng dụng sử dụng Redis.

StatefulSet yêu cầu một "Headless Service" để đảm nhiệm nhận diện mạng. Hãy bắt đầu bằng cách định nghĩa một "headless service" với `clusterIP: None`, điều này sẽ báo cho Kubernetes không thực hiện proxy hay cân bằng tải, mà cho phép truy cập trực tiếp vào các Pod:

**service.yaml**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: redis-svc
  labels:
    app: redis
spec:
  ports:
    - port: 6379
      name: web
  clusterIP: None
  selector:
    app: redisapp
```

StatefulSet với hai container, Redis và [redisfiller](https://github.com/kubernetes-hy/material-example/tree/master/app5), là một app đơn giản sử dụng Redis:

**statefulset.yaml**

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis-stset
spec:
  serviceName: redis-svc
  replicas: 2
  selector:
    matchLabels:
      app: redisapp
  template:
    metadata:
      labels:
        app: redisapp
    spec:
      containers:
        - name: redisfiller
          image: jakousa/dwk-app5:54203329200143875187753026f4e93a1305ae26
        - name: redis
          image: redis:5.0
          ports:
            - name: web
              containerPort: 6379
          volumeMounts:
            - name: redis-data-storage
              mountPath: /data
  volumeClaimTemplates:
    - metadata:
        name: redis-data-storage
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: local-path
        resources:
          requests:
            storage: 100Mi
```

Lưu ý rằng vì các container giờ nằm trong cùng một pod nên chúng chia sẻ mạng và app _redisfiller_ sẽ thấy Redis ở địa chỉ _localhost:6379_.

StatefulSet trông rất giống _Deployment_ nhưng sử dụng [volumeClaimTemplate](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/#volume-claim-templates) để tạo volume riêng cho từng pod.

Ở phần 1 chúng ta phải làm khá nhiều để có storage, nhưng giờ chúng ta dùng storage _dynamically provisioned_ do K3s cung cấp bằng cách chỉ định `storageClassName: local-path`.

Vì [local-path storage](https://docs.k3s.io/storage#setting-up-the-local-storage-provider) được cấp phát động, chúng ta không cần tạo PersistentVolume cho volume, K3s sẽ tự lo việc đó.

Để tìm hiểu thêm, xem [tài liệu Rancher](https://rancher.com/docs/k3s/latest/en/storage/) và đọc thêm về [dynamic provisioning](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#dynamic). Nếu muốn, bạn có thể xem lại các ví dụ và bài tập ở phần 1 và dùng dynamic provisioning thay cho manual provisioning trong ứng dụng của mình!

Giờ bạn có thể mở hai terminal và chạy `$ kubectl logs -f redis-stset-X redisfiller` với X là 0 hoặc 1. Để xác nhận mọi thứ hoạt động, bạn có thể xóa một pod và nó sẽ khởi động lại và tiếp tục đúng nơi đã dừng. Ngoài ra, bạn có thể xóa StatefulSet và volume vẫn còn, sẽ được gắn lại khi bạn apply StatefulSet lần nữa.

Hãy nhấn mạnh lại: StatefulSet tạo một volume riêng cho mỗi replica. Ta có thể thấy thực sự có hai PersistentVolumeClaim cho app:

```bash
$ kubectl get pvc
NAME              STATUS   VOLUME                   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
data-redis-ss-0   Bound    pvc-f318ca82-d584-4e10   100Mi      RWO            local-path     53m
data-redis-ss-1   Bound    pvc-d8e5b81a-05ec-420b   100Mi      RWO            local-path     53m
```

Như vậy, `volumeClaimTemplates` trong định nghĩa StatefulSet được dùng để tạo PersistentVolumeClaim riêng cho từng replica trong set.

Hãy quan sát kỹ hơn cách Headless Service hoạt động. Như đã thấy ở trên, nó được định nghĩa với `clusterIP: None`, nên service không có cluster IP và truy cập phải thực hiện trực tiếp tới các pod.

Hãy thử _ping_ tới service từ pod busybox:

```bash
$ ping redis-svc
PING redis-svc (10.42.2.25): 56 data bytes
64 bytes from 10.42.2.25: seq=0 ttl=64 time=0.165 ms
```

Vậy có thể truy cập service chỉ bằng tên _redis-svc_, nó resolve ra IP _10.42.2.25_.

Dùng lệnh [nslookup](https://en.wikipedia.org/wiki/Nslookup) ta thấy tên miền _redis-svc_ resolve ra hai địa chỉ IP khác nhau:

```bash
$ nslookup redis-svc
Name:	redis-svc.default.svc.cluster.local
Address: 10.42.2.25
Name:	redis-svc.default.svc.cluster.local
Address: 10.42.1.32
```

Lệnh ping chỉ chọn địa chỉ IP đầu tiên. Tất cả các replica đều có tên miền riêng:

```bash
$ ping redis-stset-0.redis-svc
PING redis-ss-0.redis-svc (10.42.2.25): 56 data bytes
64 bytes from 10.42.2.25: seq=0 ttl=64 time=0.214 ms

$ ping redis-stsets-1.redis-svc
PING redis-ss-1.redis-svc (10.42.1.32): 56 data bytes
64 bytes from 10.42.1.32: seq=0 ttl=62 time=0.140 ms
```

Nhận diện của các pod là cố định, nên nếu ví dụ pod _redis-stset-0_ chết, nó sẽ luôn có cùng tên khi được lên lịch lại, và vẫn gắn với cùng volume.

Lưu ý rằng bạn có thể định nghĩa StatefulSet và Headless Service trong cùng một file bằng cách ngăn cách chúng bằng ba dấu gạch ngang:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis-stset
spec:
  serviceName: redis-svc
  replicas: 2
  selector:
    matchLabels:
      app: redisapp
  # more rows
---
apiVersion: v1
kind: Service
metadata:
  name: redis-svc
  labels:
    app: redis
spec:
  ports:
    - port: 6379
      name: web
  clusterIP: None
  selector:
    app: redisapp
```

<exercise name='Exercise 2.07: Ứng dụng có trạng thái'>

Chạy một cơ sở dữ liệu Postgres và lưu bộ đếm của ứng dụng Ping-pong vào cơ sở dữ liệu.

Cơ sở dữ liệu Postgres và ứng dụng Ping-pong **không được** nằm trong cùng một pod.
Một cơ sở dữ liệu Postgres là đủ và nó có thể biến mất cùng với cụm, nhưng nó phải tồn tại ngay cả khi tất cả các pod bị tắt.

**Gợi ý:** nên đảm bảo cơ sở dữ liệu đã sẵn sàng và có thể kết nối trước khi kết nối từ ứng dụng Ping-pong. Để làm vậy, bạn có thể khởi động một pod độc lập chạy image Postgres:

```bash
  kubectl run -it --rm --restart=Never --image postgres psql-for-debugging sh
  $ psql postgres://yourpostgresurlhere
  psql (16.2 (Debian 16.2-1.pgdg120+2))
  Type "help" for help.
  postgres=# \d
  Did not find any relations.
```

</exercise>

<exercise name='Exercise 2.08: Project v1.2'>

Tạo một cơ sở dữ liệu và lưu các todo vào đó. Cơ sở dữ liệu cũng nên có pod riêng.

Sử dụng Secret và/hoặc ConfigMap để backend truy cập vào cơ sở dữ liệu.

</exercise>

## Job và CronJob

Tài nguyên [Job](https://kubernetes.io/docs/concepts/workloads/controllers/job/) được dùng để chạy các workload không phải dịch vụ liên tục, mà chỉ chạy từ đầu đến cuối. Trạng thái của job được lưu lại để có thể theo dõi sau khi thực thi xong. Job có thể cấu hình để chạy nhiều instance cùng lúc, tuần tự hoặc cho đến khi đạt số lần thành công nhất định.

Một ví dụ sử dụng job là tạo backup từ cơ sở dữ liệu. _Job_ của chúng ta sẽ dùng biến môi trường URL làm địa chỉ lấy dữ liệu dump và gửi nó đến một server lưu trữ. Cơ sở dữ liệu sẽ là Postgres và công cụ tạo backup là [pg_dump](https://www.postgresql.org/docs/current/app-pgdump.html). Chỉ cần một script bash đơn giản là đủ.

```bash
#!/usr/bin/env bash
set -e

if [ $URL ]
then
  pg_dump -v $URL > /usr/src/app/backup.sql

  echo "Not sending the dump actually anywhere"
  # curl -F ‘data=@/usr/src/app/backup.sql’ https://somewhere
fi
```

Script trên đã được đóng gói thành image [jakousa/simple-backup-example](https://hub.docker.com/r/jakousa/simple-backup-example).

Vì chúng ta chưa có Postgres, hãy triển khai một cái trước:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres-svc
  labels:
    app: postgres
spec:
  ports:
    - port: 5432
      name: web
  clusterIP: None
  selector:
    app: postgres
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres-ss
spec:
  serviceName: postgres
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:13.0
          ports:
            - name: postgres
              containerPort: 5432
          env:
            - name: POSTGRES_PASSWORD
              value: "example"
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: local-path
        resources:
          requests:
            storage: 100Mi
```

Apply file trên và kiểm tra pod đã chạy:

```shell
$ kubectl get po
  NAME                                READY   STATUS    RESTARTS   AGE
  postgres-ss-0                       1/1     Running   0          65s
```

Giờ chúng ta có thể apply job sau sử dụng image trên:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: backup
spec:
  template:
    spec:
      containers:
        - name: backup
          image: jakousa/simple-backup-example
          env:
            - name: URL
              value: "postgres://postgres:example@postgres-svc:5432/postgres"
      restartPolicy: Never # Lần này chỉ chạy một lần
```

Pod có một số cấu hình sẵn có. Ví dụ, có thể ép nó thử lại nhiều lần bằng cách định nghĩa `backoffLimit`.

```shell
$ kubectl get jobs
  NAME     COMPLETIONS   DURATION   AGE
  backup   1/1           7s         35s

$ kubectl logs backup-wj9r5
  ...
  pg_dump: saving encoding = UTF8
  pg_dump: saving standard_conforming_strings = on
  pg_dump: saving search_path =
  pg_dump: implied data-only restore
  Not sending the dump actually anywhere
```

[CronJob](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/) tương tự Job nhưng chạy theo lịch. Có thể bạn đã từng dùng [cron](https://vi.wikipedia.org/wiki/Cron) để lên lịch tác vụ trên server, CronJob về cơ bản cũng giống vậy nhưng cho container.

<exercise name='Exercise 2.09: Todo hàng ngày'>

Tạo một CronJob tạo một todo mới mỗi giờ để nhắc bạn "Read < URL >".

Trong đó < URL > là một bài viết Wikipedia được job chọn ngẫu nhiên. Không cần phải là hyperlink, người dùng có thể copy-paste URL từ todo.

https://en.wikipedia.org/wiki/Special:Random sẽ trả về redirect tới một trang Wikipedia ngẫu nhiên nên bạn có thể dùng nó để lấy một bài viết ngẫu nhiên để đọc. GỢI Ý: Kiểm tra header location

</exercise>
