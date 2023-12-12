# Deployment 

```shell
kubectl create deployment guestbook --image=ibmcom/guestbook:v1

kubectl get pod
kubectl get pod -o custom-columns=POD_NAME:.metadata.name,CONTAINER_NAME:.spec.containers[*].name,IMAGE_NAME:.spec.containers[*].image


kubectl scale deployment guestbook --replicas=10
kubectl set image deployment/guestbook guestbook=ibmcom/guestbook:v2
#edit maxSurge maxAvailable
# Sử dụng command sau để edit giá trị của deployment guestbook
kubectl edit deployment guestbook
# Khi edit xong nhấn tuần tự các phím sau để lưu lại giá trị edit
# Esc -> ":wq"
# Hoặc sử dụng command sau để edit giá trị maxSurge và maxAvailable
kubectl patch deployment guestbook -p '{"spec":{"strategy":{"rollingUpdate":{"maxSurge":1,"maxUnavailable":1}}}}'

# Command để quan sát sự thay đổi khi tăng replica, thay đổi image
watch -n 1 kubectl get pods -o custom-columns=POD_NAME:.metadata.name,CONTAINER_NAME:.spec.containers[*].name,IMAGE_NAME:.spec.containers[*].image
```

## Prompt request trên GPT

Kubectl sẽ làm việc của yếu bằng file yaml, được gọi là manifest. Thông qua 3 command phổ biến chính

```shell
# Tạo resource
kubectl create -f <file_manifest>
# Ví dụ 
kubectl create -f cluster.yaml

# Command về apply
kubectl apply -f <file_manifest>
# Đối với apply, nó sẽ thực hiện tác vụ createOrUpdate. Nghĩa là nếu resource đó chưa có trên cluster k8s thì nó sẽ khởi tạo. Còn không thì nó sẽ update các giá trị mới.

# Command về delete
kubectl delete -f <file_manifest>
# Command này sẽ xóa toàn bộ các resource được defined trong file manifest
```

Để hỏi chatGPT mình sẽ chat với "prompt như sau"

```
Tạo một file manifest với kind là <dạng kind>  replicas=<số lượng replica> cho image=<ten image container>

# Ví dụ
Tạo một file manifest với kind là Deployment replicas=2 cho image=nginx
Tạo một file manifest với kind là StatefulSet replicas=1 cho image=nginx
```

Những từ khóa quan trọng để chatGPT hiểu đó là `manifest`, `kind`, `image`

### Thay đổi giá trị của của Manifest

Cú pháp
thêm `healthcheck` `livenessProbe` cho `deployment` sử dụng `httpGet `đến url `/liveProb`

Các từ khóa trên là bắt bược. Trong đó livenessProbe sẽ được tùy chỉnh như trong slide. Như các giá trị: livenessProbe, startupProbe, readinessProbe


## DesignPartents trong manifest

Do kubernetes được based hoàn toàn bằng GoLang nên các giá trị key-values cũng tuân thủ theo design pattern của Golang.

- Chữ cái đầu tiên sẽ viết thường, sau đó sẽ viết Hoa
Ví dụ: livenessProbe, startupProbe, imagePullSecrets.