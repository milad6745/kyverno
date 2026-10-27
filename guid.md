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


