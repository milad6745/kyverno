```YAML
cat kyverno-sensitive-workload.yaml
# Kyverno 1.11.1 / Kubernetes 1.25.6
# Production-oriented baseline, scoped initially to namespace sensitive-apps.
# Start in Audit; after reviewing violations change to Enforce.
# This is NOT a replacement for NetworkPolicies, RBAC, or signature verification.
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: sensitive-workloads-hardening
  annotations:
    policies.kyverno.io/title: Sensitive Workloads Hardening Baseline
    policies.kyverno.io/description: >-
      Pod security, trusted image registry, resource sizing and ownership labels
      for sensitive-apps namespace. Staged rollout is required.
spec:
  validationFailureAction: Audit
  background: true
  rules:
    - name: restricted-pod-security
      match:
        any:
          - resources:
              kinds:
                - Pod
              namespaces:
                - sensitive-apps
      validate:
        message: >-
          Pods must comply with Kubernetes Pod Security Standards Restricted v1.25.
        podSecurity:
          level: restricted
          version: v1.25

    - name: images-from-internal-registry-only
      match:
        any:
          - resources:
              kinds:
                - Pod
              namespaces:
                - sensitive-apps
      validate:
        message: >-
          Every regular, init, and ephemeral container image must come from
          192.168.72.60:32082. Replace this with a TLS-protected production registry.
        pattern:
          spec:
            containers:
              - image: "192.168.72.60:32082/*"
            "=(initContainers)":
              - image: "192.168.72.60:32082/*"
            "=(ephemeralContainers)":
              - image: "192.168.72.60:32082/*"

    - name: require-container-resources
      match:
        any:
          - resources:
              kinds:
                - Pod
              namespaces:
                - sensitive-apps
      validate:
        message: >-
          Every app container needs CPU/memory requests and memory limits.
          Size actual values according to workload profiles and LimitRanges.
        pattern:
          spec:
            containers:
              - name: "*"
                resources:
                  requests:
                    cpu: "?*"
                    memory: "?*"
                  limits:
                    memory: "?*"

    - name: require-app-and-owner-labels
      match:
        any:
          - resources:
              kinds:
                - Pod
              namespaces:
                - sensitive-apps
      validate:
        message: "Pods need nonempty app and owner labels for inventory and accountability."
        pattern:
          metadata:
            labels:
              app: "?*"
              owner: "?*"

    - name: disable-default-service-account-token
      match:
        any:
          - resources:
              kinds:
                - Pod
              namespaces:
                - sensitive-apps
      validate:
        message: >-
          Workloads must explicitly set automountServiceAccountToken: false.
          API clients require an approved service-specific exception.
        pattern:
          spec:
            automountServiceAccountToken: false
```



برای کلاستر عملیاتی حساس، پیشنهاد من طراحی یک Security Baseline چندلایه (Defense in Depth) است؛ یعنی حتی اگر یک Container آسیب‌پذیر شد، نتواند به‌سادگی به Host، سایر Namespaceها، Kubernetes API یا اطلاعات حساس دسترسی پیدا کند.

نکته مهم این است که Kyverno جایگزین RBAC، NetworkPolicy، Runtime Security یا Image Scanning نمی‌شود. نقش آن جلوگیری از ورود Configurationهای ناامن به کلاستر است.

من Policy را با توجه به شرایط فعلی تو آماده کرده‌ام:

- Kubernetes: `v1.25.6`
- Kyverno: `v1.11.1`
- Registry آزمایشگاهی: `192.168.72.60:32082`
- محیط هدف: Workloadهای حساس در یک کلاستر مشترک
- شروع با `Audit`، سپس تبدیل به `Enforce`

## ۱. مهم‌ترین کنترل‌های امنیتی Kyverno

۱. جلوگیری از Privileged Container — Critical

کانتینر نباید با `privileged: true` اجرا شود؛ چون این وضعیت بخش بزرگی از جداسازی امنیتی میان Container و Host را از بین می‌برد.

ریسک: Container Escape، دسترسی به Deviceهای میزبان و افزایش Blast Radius.

۲. جلوگیری از اجرای Container با Root — High

باید `runAsNonRoot: true` اعمال شود تا برنامه با کاربر غیر Root اجرا شود.

ریسک: سوءاستفاده از دسترسی Root داخل Container و افزایش اثر آسیب‌پذیری‌ها. اجرای Non-root به‌تنهایی مانع Container Escape نیست.

۳. محدودکردن HostPath و Host Namespace — Critical

باید دسترسی‌های خطرناک مانند `hostPID`، `hostIPC`، `hostNetwork` و `hostPath` برای برنامه‌های عادی محدود شوند.

ریسک: مشاهده Processهای Host، دسترسی به فایل‌های Node و کاهش جداسازی Workloadها.

۴. جلوگیری از Privilege Escalation — Critical

الزام `allowPrivilegeEscalation: false` و حذف Linux Capabilityهای غیرضروری با `capabilities.drop: ["ALL"]`.

ریسک: کسب دسترسی بالاتر توسط Processهای داخل Container.

۵. Seccomp Profile — High

اجرای Container با `RuntimeDefault` یا Profile مجاز و مشخص‌شده به جای `Unconfined`.

ریسک: افزایش سطح دسترسی به System Callهای حساس Kernel.

۶. Trusted Image Registry — High

فقط Imageهایی که از Registry مورد تأیید سازمان می‌آیند مجاز باشند؛ از جمله `initContainers` و `ephemeralContainers`.

ریسک: اجرای Image ناشناس یا تأییدنشده. البته مجازبودن Registry تضمین‌کننده سالم‌بودن Image نیست.

۷. محدودکردن ServiceAccount Token — High

برای برنامه‌هایی که نیازی به Kubernetes API ندارند، مقدار `automountServiceAccountToken: false` الزامی باشد.

ریسک: سرقت Token داخل Pod و استفاده از مجوزهای ServiceAccount.

۸. Resource Requests & Limits — High

برای Containerها مقدار CPU و Memory Requests و Memory Limit مشخص شود.

ریسک: مصرف کنترل‌نشده منابع و اختلال در سایر Workloadها؛ از جمله سناریوهای Resource Exhaustion.

۹. Network Isolation — Critical

تمام Namespaceهای حساس باید NetworkPolicy مناسب داشته باشند؛ ترجیحاً با رویکرد Default Deny و مجازکردن ارتباطات ضروری.

ریسک: Lateral Movement میان سرویس‌ها. Kyverno می‌تواند وجود NetworkPolicy را الزام کند، اما اجرای واقعی محدودیت شبکه بر عهده CNI است.

۱۰. Image Signature Verification — Critical

قبل از پذیرش Image، امضاهای Cosign توسط Kyverno تأیید شوند؛ در معماری تو، امضای Developer و تأیید Security به‌صورت دو مرجع مستقل.

ریسک: اجرای Artifact دست‌کاری‌شده یا فاقد تأیید امنیتی. این کنترل نیازمند Policy جداگانه و تنظیم Trust Material است.

پنج کنترل اول تا حد زیادی توسط استاندارد رسمی Pod Security Standards — Restricted پوشش داده می‌شوند. به‌جای نوشتن چندین Rule تکراری، از قابلیت `validate.podSecurity` در Kyverno 1.11 استفاده کرده‌ام و نسخه PSS را روی `v1.25` ثابت گذاشته‌ام.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://release-1-11-0.kyverno.io\&sz=32)

Kyverno+1



## ۲. فایل ClusterPolicy پیشنهادی

دانلود ClusterPolicy برای Workloadهای حساس

YAML — سازگار از نظر ساختار با Kyverno 1.11.1، اعتبارسنجی YAML انجام شده

این فایل یک `ClusterPolicy` با پنج Rule دارد:

| Rule                                    | آنچه بررسی می‌کند                                   |
| --------------------------------------- | --------------------------------------------------- |
| `restricted-pod-security`               | استاندارد امنیتی Restricted در Kubernetes 1.25      |
| `images-from-internal-registry-only`    | مبدأ Image برای Containerهای عادی، Init و Ephemeral |
| `require-container-resources`           | CPU/Memory Requests و Memory Limits                 |
| `require-app-and-owner-labels`          | وجود Labelهای مالکیت و شناسایی                      |
| `disable-default-service-account-token` | جلوگیری از Mount خودکار ServiceAccount Token        |

این یک Baseline اولیه قابل آزمایش است، نه مجموعه نهایی تمام کنترل‌های یک محیط Production. پنج Rule بالا منابع `Pod` را ارزیابی می‌کنند و فعلاً فقط در Namespace به نام `sensitive-apps` اعمال می‌شوند. Resourceهای Cluster-scoped مثل `ClusterRoleBinding`، تنظیمات Ingress و تغییر خود Webhookها نیازمند Policyهای مستقل هستند.

### بخش اصلی فایل

```
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: sensitive-workloads-hardening
spec:
  validationFailureAction: Audit
  background: true
  rules:
    - name: restricted-pod-security
      match:
        any:
          - resources:
              kinds:
                - Pod
              namespaces:
                - sensitive-apps
      validate:
        message: >-
          Pods must comply with Kubernetes
          Pod Security Standards Restricted v1.25.
        podSecurity:
          level: restricted
          version: v1.25
```

بقیه Ruleهای مربوط به Registry، Resources، Labels و ServiceAccount در فایل دانلودی قرار دارند.

### چرا `Audit` و نه `Enforce`؟

برای محیط حساس، در نهایت باید کنترل‌های حیاتی Enforce شوند؛ ولی اگر از روز اول Restricted را روی همه سرویس‌های عملیاتی فعال کنی، احتمال Block شدن برنامه‌های Legacy، Jobها و بعضی Agentها وجود دارد.

پس ترتیب درست چنین است:

۱

Audit

شناسایی تخلفات بدون Block

۲

Remediation

اصلاح Manifestها و ثبت استثناهای ضروری

۳

Enforce

جلوگیری از پذیرش Workloadهای نامنطبق

برای قابلیت‌هایی مثل HostPath و Privileged، سیاست Enforce باید اصل باشد و استثناها صرفاً محدود، مستند و دارای زمان انقضا باشند.

## ۳. نصب و تست Policy روی کلاستر فعلی

چون Namespace به نام `kyverno` از پردازش بعضی Webhookها مستثناست، تست را در Namespace مستقل انجام می‌دهیم.

ابتدا:

```
kubectl create namespace sensitive-apps
```

سپس Policy را نصب کن:

```
kubectl apply --server-side \
  -f kyverno-sensitive-workloads-v1.11.1.yaml
```

و وضعیت آن را بررسی کن:

```
kubectl get cpol sensitive-workloads-hardening
kubectl describe cpol sensitive-workloads-hardening
```

مهم است که Policy در وضعیت `Ready` باشد.

حالا یک Pod نامنطبق ایجاد کن:

```
apiVersion: v1
kind: Pod
metadata:
  name: insecure-test
  namespace: sensitive-apps
spec:
  containers:
    - name: test
      image: nginx:latest
      securityContext:
        privileged: true
```

در حالت `Audit`، انتظار داریم درخواست به دلیل Policy رد نشود، اما تخلفات آن گزارش شوند. ممکن است Pod به دلیل نبود Image در محیط آفلاین اجرا نشود؛ این موضوع مستقل از ارزیابی Kyverno است.

گزارش‌ها را بررسی کن:

```
kubectl get policyreport -n sensitive-apps

kubectl describe policyreport -n sensitive-apps

kubectl logs -n kyverno \
  deployment/kyverno-admission-controller \
  --tail=100
```

اگر Ruleها به‌درستی کار کردند و برنامه‌های مجاز را اصلاح کردی، می‌توانی در فایل مقدار زیر را:

```
validationFailureAction: Audit
```

به این مقدار تبدیل کنی:

```
validationFailureAction: Enforce
```

و Policy را مجدداً Apply کنی.

## ۴. چه کنترل‌های دیگری برای محیط عملیاتی حساس لازم داریم؟

پیشنهاد می‌کنم این کنترل‌ها در مرحله بعد، به شکل `ClusterPolicy`های مستقل طراحی شوند تا تغییر یا غیرفعال‌کردن یک کنترل روی بقیه اثر نگذارد.

| کنترل تکمیلی                   | اولویت   | روش پیشنهادی                                                         |
| ------------------------------ | -------- | -------------------------------------------------------------------- |
| Cosign Signature Verification  | Critical | بررسی امضای Developer و Security                                     |
| Image Digest Pinning           | High     | الزام `@sha256` به جای Tag متغیر                                     |
| NetworkPolicy Enforcement      | Critical | الزام Default Deny و مسیرهای مجاز                                    |
| RBAC Hardening                 | Critical | جلوگیری از ایجاد مجوزهای خطرناک یا دسترسی گسترده                     |
| Ingress Security               | High     | کنترل TLS، Host، مسیرها و Annotationهای حساس                         |
| Read-Only Root Filesystem      | High     | الزام `readOnlyRootFilesystem: true` در صورت سازگاری برنامه          |
| Privileged Workload Exceptions | Critical | استثنای محدود برای Agentها با تأیید امنیتی                           |
| Secret Handling                | High     | جلوگیری از Secretهای حساس داخل Env/Manifest و کنترل روش دریافت آن‌ها |
| Policy Governance              | Critical | محدودکردن افرادی که مجاز به ویرایش Policy و Webhook هستند            |
RBAC و Network Isolation است. این‌ها مکمل Baseline فعلی خواهند بود، نه جایگزین آن.
