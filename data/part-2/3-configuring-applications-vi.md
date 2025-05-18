---
path: "/part-2/3-configuring-applications-vi"
title: "Cấu hình ứng dụng"
hidden: false
---

<text-box variant='learningObjectives' name='Mục tiêu học tập'>

Sau phần này bạn

- biết cách truyền biến vào pod của mình

- có các lựa chọn để lưu trữ secret vào hệ thống quản lý phiên bản

- có kinh nghiệm tra cứu thông tin trong tài liệu Kubernetes

</text-box>

Kubernetes có hai loại tài nguyên để quản lý cấu hình. [Secret](https://kubernetes.io/docs/concepts/configuration/secret/) dùng cho thông tin nhạy cảm được truyền vào container khi chạy. [ConfigMap](https://kubernetes.io/docs/concepts/configuration/configmap/) khá giống secret nhưng có thể chứa bất kỳ loại cấu hình nào. Trường hợp sử dụng ConfigMap rất đa dạng: bạn có thể ánh xạ một ConfigMap thành một file với các giá trị mà server đọc khi chạy. Thay đổi ConfigMap sẽ lập tức thay đổi hành vi của ứng dụng. Cả hai đều có thể dùng để truyền biến môi trường.

### Secret

Hãy sử dụng [pixabay](https://pixabay.com/) để hiển thị ảnh trên một web app đơn giản. Xác thực với API được thực hiện bằng API key. Theo [tài liệu API](https://pixabay.com/api/docs/) bạn chỉ cần đăng nhập để lấy key.

Các manifest của ứng dụng ở [đây](https://github.com/kubernetes-hy/material-example/tree/master/app4/manifests). Hãy khởi động app với service và ingress:

```shell
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes-hy/material-example/master/app4/manifests/deployment.yaml \
                -f https://raw.githubusercontent.com/kubernetes-hy/material-example/master/app4/manifests/ingress.yaml \
                -f https://raw.githubusercontent.com/kubernetes-hy/material-example/master/app4/manifests/service.yaml
```

Ứng dụng yêu cầu biến môi trường API_KEY với giá trị là một API key hợp lệ.

Biến môi trường này được định nghĩa là một [secret](https://kubernetes.io/docs/concepts/configuration/secret/) trong deployment như sau:

**deployment.yaml**

```yaml
# ...
containers:
  - name: imageagain
    envFrom:
      - secretRef:
          name: pixabay-apikey
```

Điều này giả định rằng secret _pixabay-apikey_ định nghĩa key dưới tên biến _API_KEY_. Nếu tên biến môi trường trong secret khác, bạn có thể dùng dạng khai báo dài hơn trong deployment:

**deployment.yaml**

```yaml
# ...
containers:
  - name: imageagain
    env:
      - name: API_KEY # Tên biến môi trường truyền vào container
        valueFrom:
          secretKeyRef:
            name: pixabay-apikey
            key: API_KEY # Tên biến trong secret
```

Ứng dụng sẽ không chạy ngay và bạn sẽ thấy:

```bash
$ kubectl get pod
NAME                            READY   STATUS                       RESTARTS       AGE
imageapi-dep-6cdd4879f7-zwlbr   0/1     CreateContainerConfigError   0              13m
```

Lý do chính xác có thể xem bằng lệnh `describe`:

```bash
$ kubectl describe pod imageapi-dep-6cdd4879f7-zwlbr
Name:             imageapi-dep-6cdd4879f7-zwlbr
Status:           Pending
IP:               10.42.0.89

...

Events:
  Type     Reason     Age     From       Message
  ----     ------     ----    ----       -------
  Warning  Failed     21m     kubelet    Error: secret "pixabay-apikey" not found
  Normal   Pulled     3m15s   kubelet    Container image "jakousa/dwk-app4:b7fc18de2376da80ff0cfc72cf581a9f94d10e64" already present on machine
```

Hãy sử dụng secret để truyền biến môi trường API key vào ứng dụng.

Secret sử dụng mã hóa [base64](https://en.wikipedia.org/wiki/Base64) để tránh gặp vấn đề với ký tự đặc biệt. Chúng ta cũng nên dùng mã hóa để tránh lộ API_KEY ra ngoài. Đầu tiên, để kiểm thử, hãy tạo và apply một file secret.yaml như sau:

**secret.yaml**

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: pixabay-apikey
data:
  API_KEY: aHR0cDovL3d3dy55b3V0dWJlLmNvbS93YXRjaD92PWRRdzR3OVdnWGNR
  # Giá trị đã mã hóa base64, lưu ý giá trị này không dùng được
```

Bạn có thể tạo key mã hóa base64 bằng [công cụ online này](https://www.base64encode.org/) hoặc trên terminal với lệnh `base64`:

```bash
$ echo -n 'my-string' | base64
bXktc3RyaW5n
```

Vì container đã được cấu hình lấy biến môi trường từ secret nên nó sẽ tự động nhận giá trị. Giờ bạn có thể xác nhận app hoạt động tại http://localhost:8081.

Vì ai cũng có thể giải mã base64 nên ta không nên lưu file này vào hệ thống quản lý phiên bản. Nếu muốn lưu cấu hình lâu dài, ta cần mã hóa giá trị.

Có nhiều giải pháp quản lý secret tùy nền tảng. Các nhà cung cấp đám mây có thể có giải pháp riêng, như Google Cloud [Secret Manager](https://cloud.google.com/secret-manager) hoặc [AWS Secrets Manager](https://aws.amazon.com/secrets-manager/). Với Kubernetes, bạn có thể dùng [SealedSecrets](https://github.com/bitnami-labs/sealed-secrets). Thực tế, SealedSecrets từng được dùng ở phiên bản trước của khóa học này.

Chúng ta sẽ dùng [SOPS](https://github.com/mozilla/sops) để mã hóa file secret.yaml. Công cụ này khá linh hoạt, bạn có thể dùng cho nhiều môi trường khác nhau, kể cả với file Docker compose. Hãy đọc qua Readme, hoặc ít nhất phần [Motivation](https://github.com/mozilla/sops#motivation). Chúng ta sẽ dùng [age](https://github.com/FiloSottile/age) để mã hóa vì nó được khuyến nghị hơn PGP. Hãy cài cả SOPS và age.

Tạo một cặp khóa trước:

```bash
$ age-keygen -o key.txt
  Public key: age17mgq9ygh23q0cr00mjn0dfn8msak0apdy0ymjv5k50qzy75zmfkqzjdam4
```

File key.txt này chứa cả public và secret key. Secret key không được đưa vào hệ thống quản lý phiên bản, đó là khóa cá nhân của bạn. Hãy mã hóa giá trị trong file secret.yaml. Bạn cũng có thể bỏ qua --encrypted-regex nếu muốn.

```bash
$ sops --encrypt \
       --age age17mgq9ygh23q0cr00mjn0dfn8msak0apdy0ymjv5k50qzy75zmfkqzjdam4 \
       --encrypted-regex '^(data)$' \
       secret.yaml > secret.enc.yaml
```

File secret.enc.yaml sẽ trông như sau:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: pixabay-apikey
data:
  API_KEY: ENC[AES256_GCM,data:geKXBLn4kZ9A2KHnFk4RCeRRnUZn0DjtyxPSAVCtHzoh8r6YsCUX3KiYmeuaYixHv3DRKHXTyjg=,iv:Lk290gWZnUGr8ygLGoKLaEJ3pzGBySyFJFG/AjwfkJI=,tag:BOSX7xJ/E07mXz9ZFLCT2Q==,type:str]
sops:
  kms: []
  gcp_kms: []
  azure_kv: []
  hc_vault: []
  age:
    - recipient: age17mgq9ygh23q0cr00mjn0dfn8msak0apdy0ymjv5k50qzy75zmfkqzjdam4
      enc: |
        -----BEGIN AGE ENCRYPTED FILE-----
        YWdlLWVuY3J5cHRpb24ub3JnL3YxCi0+IFgyNTUxOSBDczBhbGNxUkc4R0U0SWZI
        OEVYTEdzNUlVMEY3WnR6aVJ6OEpGeCtJQ1hVCjVSbDBRUnhLQjZYblQ0UHlneDIv
        UmswM2xKUWxRMHZZQjVJU21UbDNEb3MKLS0tIGhOMy9lQWx4Q0FUdVhoVlZQMjZz
        dDEreFAvV3Nqc3lIRWh3RGRUczBzdXcKh7S4q8qp5SrDXLQHZTpYlG43vLfBlqcZ
        BypI8yEuu18rCjl3HJ+9jbB0mrzp60ld6yojUnaggzEaVaCPSH/BMA==
        -----END AGE ENCRYPTED FILE-----
  lastmodified: "2021-10-29T12:20:40Z"
  mac: ENC[AES256_GCM,data:qhOMGFCDBXWhuildW81qTni1bnaBBsYo7UHlv2PfQf8yVrdXDtg7GylX9KslGvK22/9xxa2dtlDG7cIrYFpYQPAh/WpOzzn9R26nuTwvZ6RscgFzHCR7yIqJexZJJszC5yd3w5RArKR4XpciTeG53ygb+ng6qKdsQsvb9nQeBxk=,iv:PZLF3Y+OhtLo+/M0C0hqINM/p5K94tb5ZGc/OG8loJI=,tag:ziFOjWuAW/7kSA5tyAbgNg==,type:str]
  pgp: []
  encrypted_regex: ^(data)$
  version: 3.7.1
```

Chúng ta có thể lưu file này vào hệ thống quản lý phiên bản một cách an toàn, vì chỉ những ai có secret key của `age17mgq9ygh23q0cr00mjn0dfn8msak0apdy0ymjv5k50qzy75zmfkqzjdam4` mới giải mã được. Hãy nhớ dùng khóa của riêng bạn!

Nếu muốn mã hóa file cho cả nhóm, bạn cần thêm danh sách public key khi mã hóa. Bất kỳ ai có private key đều có thể giải mã file. Thực tế, cách tốt nhất là (gần như) không ai giữ private key! Public key dùng để mã hóa từng file, còn private key lưu riêng và chỉ dùng khi cần giải mã.

Bạn có thể giải mã file đã mã hóa bằng cách export biến môi trường _SOPS_AGE_KEY_FILE_ trỏ tới file key.txt và chạy sops với cờ --decrypt.

```shell
$ export SOPS_AGE_KEY_FILE=$(pwd)/key.txt

$ sops --decrypt secret.enc.yaml > secret.yaml
```

Bạn cũng có thể apply file secret yaml qua pipe trực tiếp, giúp tránh tạo file secret.yaml dạng plain:

```shell
$ sops --decrypt secret.enc.yaml | kubectl apply -f -
```

<exercise name='Exercise 2.05: Secrets'>

Trong tất cả các bài tập sau, nếu bạn sử dụng API key hoặc mật khẩu (ví dụ mật khẩu database), hãy dùng Secret. Bạn có thể dùng `SOPS` để lưu vào git repository. _Không bao giờ_ lưu file chưa mã hóa vào git repository.

Không có gì cụ thể để nộp, tất cả các bài nộp sau phải tuân thủ quy tắc trên.

</exercise>

### ConfigMap

[ConfigMap](https://kubernetes.io/docs/concepts/configuration/configmap/) cũng tương tự nhưng dữ liệu không cần mã hóa và không được mã hóa. Giả sử bạn có một server game nhận file cấu hình _serverconfig.txt_ như sau:

```ini
maxplayers=12
difficulty=2
```

Ta có thể dùng ConfigMap để chứa file này:

**configmap.yaml**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: example-configmap
data:
  serverconfig.txt: |
    maxplayers=12
    difficulty=2
```

Giờ ConfigMap có thể được mount vào container như một volume. Khi thay đổi giá trị, ví dụ "maxplayers", rồi apply lại ConfigMap thì thay đổi sẽ được phản ánh vào volume đó.

<exercise name='Exercise 2.06: Documentation and ConfigMaps'>

Sử dụng tài liệu chính thức của Kubernetes cho bài tập này.

- [https://kubernetes.io/docs/concepts/configuration/configmap/](https://kubernetes.io/docs/concepts/configuration/configmap/) và
- [https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/)

sẽ chứa tất cả những gì bạn cần.

Tạo một ConfigMap cho ứng dụng "Log output". ConfigMap này nên định nghĩa một file _information.txt_ và một biến môi trường _MESSAGE_.

Ứng dụng nên mount file này làm volume, đặt biến môi trường và in nội dung của chúng cùng với output thông thường:

```text
file content: this text is from file
env variable: MESSAGE=hello world
2024-03-30T12:15:17.705Z: 8523ecb1-c716-4cb6-a044-b9e83bb98e43.
Ping / Pongs: 3
```

</exercise>
