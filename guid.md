من برای دانلود ایمیج ها یک فایل yaml در همین ریپو دارم که با گیت هاب اکشن اجراش میکنم و بهم پنج تا ایمیج میده برای دانلود : 

import images
```
root@worker:/home/milad/kyverno# ctr -n k8s.io images import kyverno-background-controller-v1.11.1-amd64.tar
unpacking ghcr.io/kyverno/background-controller:v1.11.1 (sha256:5415f25b6f2c67de3ca3ede2dc0c2d8610867a4731e4baead3ab56a86de9e513)...done

root@worker:/home/milad/kyverno# ctr -n k8s.io images import kyverno-cleanup-controller-v1.11.1-amd64.tar
unpacking ghcr.io/kyverno/cleanup-controller:v1.11.1 (sha256:633307713ebda5e7f953c14b5ac6e637c0f2b43d6c97c720b18e3690d6917e38)...done

root@worker:/home/milad/kyverno# ctr -n k8s.io images import kyverno-cleanup-controller-v1.11.1-amd64.tarkyverno-kyvernopre-v1.11.1-amd64.tar
ctr: open kyverno-cleanup-controller-v1.11.1-amd64.tarkyverno-kyvernopre-v1.11.1-amd64.tar: no such file or directory

root@worker:/home/milad/kyverno# ctr -n k8s.io images import kyverno-kyvernopre-v1.11.1-amd64.tar
unpacking ghcr.io/kyverno/kyvernopre:v1.11.1 (sha256:f009b0a010aed87ecf15fc5691e43293913bcf7b2e0e967eb5816097678f557d)...done

root@worker:/home/milad/kyverno# ctr -n k8s.io images import kyverno-kyverno-v1.11.1-amd64.tar
unpacking ghcr.io/kyverno/kyverno:v1.11.1 (sha256:ee50efd5b09c5d888d67d86a4cfaa491ce07ee41a2c1ca93495ea1395ea8f41d)...done

root@worker:/home/milad/kyverno# ctr -n k8s.io images import kyverno-reports-controller-v1.11.1-amd64.tar
unpacking ghcr.io/kyverno/reports-controller:v1.11.1 (sha256:903c9a976e7ddbc075e77bcf72c29cdb246ae9be58c01fc3e71612b8ae244a48)...done
```
```
crictl images | grep kyverno
ghcr.io/kyverno/background-controller   v1.11.1             c4d81fcc7e855       97.8MB
ghcr.io/kyverno/cleanup-controller      v1.11.1             65ef9292b51cf       92.6MB
ghcr.io/kyverno/kyverno                 v1.11.1             e77b0c7575938       99.2MB
ghcr.io/kyverno/kyvernopre              v1.11.1             2038cd85ead4c       90.9MB
ghcr.io/kyverno/reports-controller      v1.11.1             070fb86421a8c       98MB
```
bash script forr load,tag and push to myrepo
```bash
# 1. Load images
docker load -i kyverno-kyvernopre-v1.11.1-amd64.tar
docker load -i kyverno-kyverno-v1.11.1-amd64.tar
docker load -i kyverno-background-controller-v1.11.1-amd64.tar
docker load -i kyverno-cleanup-controller-v1.11.1-amd64.tar
docker load -i kyverno-reports-controller-v1.11.1-amd64.tar

# 2. Tag images for Nexus
REGISTRY=192.168.72.60:32082

for IMAGE in \
  kyvernopre \
  kyverno \
  background-controller \
  cleanup-controller \
  reports-controller
do
  docker tag \
    ghcr.io/kyverno/${IMAGE}:v1.11.1 \
    ${REGISTRY}/kyverno/${IMAGE}:v1.11.1
done

# 3. Push images to Nexus
for IMAGE in \
  kyvernopre \
  kyverno \
  background-controller \
  cleanup-controller \
  reports-controller
do
  docker push ${REGISTRY}/kyverno/${IMAGE}:v1.11.1
done
```
hala deployment ro apply mikonim
bad az apply kardan error mikhore ke image haro nemitone begire
baraye ine ke nexus man http ast baraye in moshkel bayad be containerd begam be url nexus man chie 

```bash
cat  /etc/containerd/config.toml
version = 2
root = "/var/lib/containerd"
state = "/run/containerd"
oom_score = 0

[grpc]
  max_recv_message_size = 16777216
  max_send_message_size = 16777216

[debug]
  level = "info"

[metrics]
  address = ""
  grpc_histogram = false

[plugins]

  [plugins."io.containerd.grpc.v1.cri"]
    sandbox_image = "registry.k8s.io/pause:3.8"
    max_container_log_line_size = -1
    enable_unprivileged_ports = false
    enable_unprivileged_icmp = false

    [plugins."io.containerd.grpc.v1.cri".containerd]
      default_runtime_name = "runc"
      snapshotter = "overlayfs"

      [plugins."io.containerd.grpc.v1.cri".containerd.runtimes]

        [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
          runtime_type = "io.containerd.runc.v2"
          base_runtime_spec = "/etc/containerd/cri-base.json"

          [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
            systemdCgroup = true

    # ================= REGISTRY CONFIG =================
    [plugins."io.containerd.grpc.v1.cri".registry]

      [plugins."io.containerd.grpc.v1.cri".registry.mirrors]

        # Docker Hub
        [plugins."io.containerd.grpc.v1.cri".registry.mirrors."docker.io"]
          endpoint = ["https://registry-1.docker.io"]

        # Existing private HTTP registry (preserved)
        [plugins."io.containerd.grpc.v1.cri".registry.mirrors."10.233.52.98:8082"]
          endpoint = ["http://10.233.52.98:8082"]

        # Nexus Kyverno Docker HTTP registry (NodePort 32082)
        [plugins."io.containerd.grpc.v1.cri".registry.mirrors."192.168.72.60:32082"]
          endpoint = ["http://192.168.72.60:32082"]

      # Existing private registry TLS setting (preserved)
      [plugins."io.containerd.grpc.v1.cri".registry.configs."10.233.52.98:8082".tls]
        insecure_skip_verify = true
```
apply deployment
```
kubectl apply -f deployment-new.yaml
```
check pod
```
kubectl get pods -n kyverno -o wide
NAME                                            READY   STATUS    RESTARTS   AGE     IP               NODE     NOMINATED NODE   READINESS GATES
kyverno-admission-controller-6dbcb964c5-x8mwx   1/1     Running   0          3h25m   10.233.103.139   worker   <none>           <none>
kyverno-background-controller-989f9f6dc-jznbb   1/1     Running   0          3h25m   10.233.103.144   worker   <none>           <none>
kyverno-cleanup-controller-b4646864d-cq5hr      1/1     Running   0          3h25m   10.233.103.189   worker   <none>           <none>
kyverno-reports-controller-5c5fc76b55-vzk8v     1/1     Running   0          3h25m   10.233.103.173   worker   <none>           <none>
```
بعد از اجرای kyverno میایم و برایش یک require policy مینویسیم :

```
cat require-label.yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-app-label
spec:
  validationFailureAction: Enforce
  background: false
  rules:
    - name: check-app-label
      match:
        any:
          - resources:
              kinds:
                - Pod
              namespaces:
                - kyverno-test
      validate:
        message: "Pod must have an app label."
        pattern:
          metadata:
            labels:
              app: "?*"
```
```
kubectl apply -f require-label.yaml
```
create a new pod and admission webhook denie the request
```
 kubectl get clusterpolicies.kyverno.io
NAME                ADMISSION   BACKGROUND   VALIDATE ACTION   READY   AGE   MESSAGE
require-app-label   true        false        Enforce           True    33s   Ready
```
```
kubectl run test-no-label   --image=192.168.72.60:32082/kyverno/kyverno:v1.11.1   --restart=Never   -n kyverno-test
Error from server: admission webhook "validate.kyverno.svc-fail" denied the request:

resource Pod/kyverno-test/test-no-label was blocked due to the following policies

require-app-label:
  check-app-label: 'validation error: Pod must have an app label. rule check-app-label
    failed at path /metadata/labels/app/'
```

