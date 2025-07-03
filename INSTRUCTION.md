# Інструкція для валідації змін (RBAC, ServiceAccount, secrets)

## Передумови

- Кластер Kubernetes розгорнуто за допомогою kind та працює (`kubectl get nodes` — всі ноди Ready).
- Усі зміни додані у відповідні файли репозиторію (зокрема, security/rbac.yml та Deployment manifest).

---

## Крок 1. Створіть потрібний namespace

```sh
kubectl create namespace todoapp
```
> Якщо namespace вже існує — з’явиться відповідне попередження, це нормально.

---

## Крок 2. Застосуйте RBAC-маніфест

## security/rbac.yml

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: todo-sa
  namespace: app

---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: secret-reader
  namespace: app
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "list"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: secret-reader-binding
  namespace: app
subjects:
- kind: ServiceAccount
  name: todo-sa
  namespace: app
roleRef:
  kind: Role
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
```

```sh
kubectl apply -f security/rbac.yml
```

---

Переконайтеся, що не виникає помилок.  
Очікуваний вивід: створені ServiceAccount, Role та RoleBinding.

---

## Крок 3. Перевірте наявність ресурсів

```sh
kubectl get serviceaccount,role,rolebinding -n todoapp
```
Вивід має містити:
- ServiceAccount: `todo-sa`
- Role: `secret-reader`
- RoleBinding: `secret-reader-binding`

---

## Крок 4. Застосуйте Deployment

```sh
$ kubectl apply -f .infrastructure/app/pvc.yml
```
**УВАГА:**  
У Deployment у полі `spec.template.spec.serviceAccountName` має бути вказано саме `todo-sa`, а `metadata.namespace` — `todoapp`.

---

## Крок 5. Дочекайтеся готовності pod

```sh
kubectl get pods -n todoapp
```
Pod має перейти у статус **Running**.

---

## Крок 6. Перевірте, що pod використовує правильний ServiceAccount

```sh
kubectl get pod -n todoapp <pod-name> -o jsonpath="{.spec.serviceAccountName}"
```
Очікувано:  
```
todo-sa
```

---

## Крок 7. Виконайте curl-запит до Kubernetes API на secrets

```sh
kubectl exec -n todoapp <pod-name> -- \
  curl -s --cacert /var/run/secrets/kubernetes.io/serviceaccount/ca.crt \
  -H "Authorization: Bearer $(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
  https://kubernetes.default.svc/api/v1/namespaces/todoapp/secrets
```

---

## Крок 8. Перевірте результат

- Вивід має містити список secrets у форматі JSON (типу `"items": [...]`).
- Якщо отримуєте помилку на кшталт `Forbidden`, `RBAC: access denied`, або подібні — перевірте Role/RoleBinding.

---


## Крок 9. Додаткові команди для перевірок

### Чи має ServiceAccount доступ до list secrets:
```sh
kubectl auth can-i list secrets --as=system:serviceaccount:todoapp:todo-sa -n todoapp
```
Очікувано:
```
yes
```

### Перевірити всі secrets у неймспейсі:
```sh
kubectl get secrets -n todoapp
```

### Спробувати виконати заборонену дію (наприклад, create secrets):
```sh
kubectl auth can-i create secrets --as=system:serviceaccount:todoapp:todo-sa -n todoapp
```
Очікувано:
```
no
```

### Отримати токен ServiceAccount (для ручного тесту)
```sh
kubectl get secret -n todoapp $(kubectl get serviceaccount todo-sa -n todoapp -o jsonpath="{.secrets[0].name}") -o yaml
```

---

## Крок 10. Додайте скріншот

- Зробіть скріншот виконання curl-команди із виводом списку secrets у pod.
- Додайте скріншот у PR.

---

## Крок 11. Оформіть Pull Request

- Додайте файл INSTRUCTION.md, security/rbac.yml, Deployment-маніфест і скріншот.
- Створіть PR для рев’ю.

---

## Поширені помилки та дебаг

- **namespaces "todoapp" not found**: створіть namespace перед застосуванням RBAC.
- **serviceaccount не підхопився**: перевірте поле `serviceAccountName` в Deployment.
- **curl: (6) Could not resolve host**: pod не має DNS/мережі — перевірте CNI, kube-dns.
- **curl: (60) SSL certificate problem**: шлях до ca.crt неправильний або pod не має доступу до serviceaccount токену.
- **"Forbidden" або "RBAC: access denied"**: перевірте Role/RoleBinding та чи збігаються імена serviceaccount.

---

## Очікуваний результат

- ServiceAccount має дозвіл на list secrets у своєму namespace.
- Виконання curl-команди всередині pod повертає список secrets (у JSON).
- Всі дії підтверджені скріншотом у PR.

---