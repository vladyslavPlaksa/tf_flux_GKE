# Створення коду Terraform для Flux на kind_cluster

1. Виконаємо ініціалізацію terraform:
```sh
✗ terraform init
Terraform has been successfully initialized!
```

2. Перевіримо код.
```sh
✗ tf validate
Success! The configuration is valid.
```

3. Виконаємо початкові команду `terraform apply`.
```sh
✗ tf apply
Apply complete! Resources: 5 added, 0 changed, 0 destroyed.
```

- Створені ресурси:
```sh
$ tf state list
module.flux_bootstrap.flux_bootstrap_git.this
module.github_repository.github_repository.this
module.github_repository.github_repository_deploy_key.this
module.kind_cluster.kind_cluster.this
module.tls_private_key.tls_private_key.this
```

4. Розміщення файлу в [bucket](https://console.cloud.google.com/storage/browser)  
Щоб розмістити файл state в бакеті, ви можете використовувати команду terraform init з опцією --backend-config. Наприклад, щоб розмістити файл state в бакеті Google Cloud Storage, ви можете виконати наступну команду:
```sh
# Створимо bucket:
$ gsutil mb gs://<YOUR BUCKET NAME>
Creating gs://<YOUR BUCKET NAME>/...

# Перевірити вміст диску:
$ gsutil ls gs://<YOUR BUCKET NAME>
gs://<YOUR BUCKET NAME>/terraform/
```
5. Як створити bucket [читаємо документацію](https://developer.hashicorp.com/terraform/language/settings/backends/gcs#example-configuration) та додаємо до основного файлу конфігурації наступний код:

```hcl
terraform {
  backend "gcs" {
    bucket = "<YOUR BUCKET NAME>"
    prefix = "terraform/state"
  }
}
```
```sh
$ tf init
$ tf show | more
```

6. Перевіримо список ns по стан поду системи flux:
```sh
✗ k get ns
NAME                 STATUS   AGE
default              Active   16m
flux-system          Active   15m
kube-node-lease      Active   16m
kube-public          Active   16m
kube-system          Active   16m
local-path-storage   Active   16m
```
```sh
✗ k get po -n flux-system
NAME                                       READY   STATUS    RESTARTS   AGE
helm-controller-69dbf9f968-qsgq9           1/1     Running   0          16m
kustomize-controller-796b4fbf5d-jxqdx      1/1     Running   0          16m
notification-controller-78f97c759b-c8vpr   1/1     Running   0          16m
source-controller-7bc7c48d8d-c8kxk         1/1     Running   0          16m
``` 

7. Для зручності встановимо [CLI клієнт Flux](https://fluxcd.io/flux/installation/)
```sh
✗ curl -s https://fluxcd.io/install.sh | bash
✗ flux get all
```

8. Скопіюємо репозиторій та додамо в каталог `cluster` файл `demo/ns.yaml` що містить маніфест довільного `namespace`  
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: demo
```
- Після зміни стану репозиторію контролер Flux їх виявить:
    - зробить git clone  
    - підготує артефакт   
    - виконає узгодження поточного стану IC   

У даному випадку буде створено `ns demo`:
```sh
✗ flux logs -f
2023-12-19T08:36:29.686Z info GitRepository/flux-system.flux-system - stored artifact for commit 'Create ns.yaml' 
2023-12-19T08:37:31.484Z info GitRepository/flux-system.flux-system - garbage collected 1 artifacts 
```
```sh
✗ k get ns 
NAME                 STATUS   AGE
default              Active   23m
demo                 Active   4s
```
Це був приклад як Flux може керувати конфігурацією ІС Kubernetes

9. Застосуємо CLI Flux для генерації маніфестів необхідних ресурсів:
```sh
$ git clone https://github.com/<REPO OWNER>/flux-gitops.git
$ cd ../flux-gitops 

$ flux create source git kbot-telegram \
    --url=https://github.com/<REPO OWNER>/<PROJECT> \
    --branch=main \
    --namespace=demo \
    --export > clusters/demo/kbot-telegram-gr.yaml

$ flux create helmrelease kbot-telegram \
    --namespace=demo \
    --source=GitRepository/kbot-telegram \
    --chart="./helm" \
    --interval=1m \
    --export > clusters/demo/kbot-telegram-hr.yaml

$ git add .
$ git commit -m"add manifest"
$ git push
```
```sh
$ flux logs -f
2023-12-19T08:58:45.061Z info GitRepository/flux-system.flux-system - stored artifact for commit 'add manifest' 
2023-12-19T08:58:45.466Z info Kustomization/flux-system.flux-system - server-side apply for cluster definitions completed 
2023-12-19T08:58:45.559Z info Kustomization/flux-system.flux-system - server-side apply completed 
2023-12-19T08:58:45.596Z info Kustomization/flux-system.flux-system - Reconciliation finished in 498.659581ms, next run in 10m0s 
2023-12-19T08:59:46.501Z info GitRepository/flux-system.flux-system - garbage collected 1 artifacts 
```

10. Перевіримо наявність пода з нашим PET-проектом та розберемо кластер:
```sh
$ k get po -n demo
NAME                         READY   STATUS             RESTARTS       AGE
kbot-helm-6796599d7c-sqwx7   0/1     CrashLoopBackOff   7 (100s ago)   12m
k describe po -n demo | grep Warning
  Warning  BackOff    4m35s (x47 over 14m)  kubelet            Back-off restarting failed container kbot in pod kbot-helm-6796599d7c-sqwx7_demo(401ca7a7-2b0c-4a27-b81c-e053936cd9ed)
```
```sh
$ tf destroy
$ tf state list 
```
