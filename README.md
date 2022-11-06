# Лабораторна робота №1.

### Тема: Розгортання додатку в середовищі *Kubernetes.*

### Завдання:

1. Створити кластер *Kubernetes*.
2. Реалізувати кілька сервісів (1 сервіс на студента + клієнт). Описати для них *Dockerfile*.
3. Розгорнути сервіси в середовищі *Kubernetes*.
4. Реалізувати доступ до сервісів за допомогою *Ingress*.

### 1. Створення кластеру *Kubernetes* (або скорочено *k8s*)

Для створення кластеру *Kubernetes* потрібно встановити:

1. [Minikube](https://kubernetes.io/uk/docs/tasks/tools/install-minikube/) - інструмент який дозволяє запустити *Kubernetes* кластер з одного вузла локально на віртуальній машині.

> При роботі з *minikube* для коректної роботи *ingress* перевірені наступні драйвери: для *MacOS* - *virtualbox*, для *Windows* - *hyper-v*, для *Ubuntu* - *kvm*
> При роботі з драйвером *docker* після запуску *minikube* додатково необхідно виконати команду `minikube tunnel`, тоді доступ до кластера буде виконано через `localhost` (перевірено на *MacOS*).

> Альтернативно можна налаштувати кластер *kubernetes* за допомогою хмарних провайдерів (*Amazon AWS*, *Google Cloud*, *Microsoft Azure*).  
> В методичці будуть використовуватись приклади з *minikube*

2. [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/) - інтерфейс командного рядка, для роботи з *Kubernetes*

### 2 Реалізація сервісу в *Docker*

[Рекомендації по створенню Dockerfile](../0_docker/Readme.md)

### 3.1 Розгортання додатку. Створення компонент *Pod* та *Deployment*

***Pod*** - це група з одного або декількох контейнерів із загальним сховищем та мережевими ресурсами та специфікацією запуску контейнерів.
Для розгортання додатку в кластері *k8s* потрібно створити *Pod* з контейнером, що міститиме ваш сервіс.

> Для даної демонстрації буде використовуватись образ з сервером `nginx`.

Для створення *Pod* потрібно задати 2 обов'язкові параметри ім'я `nginx-pod` та образ `nginx:alpine`:

```
kubectl run nginx-pod --image=nginx:alpine
```

Перевірити наявні *Pod* можна виконавши наступну команду:

```shell
kubectl get pods
```

Вивід має виглядати наступним чином:

```log
NAME           READY   STATUS    RESTARTS   AGE
nginx-pod      1/1     Running   0          2s
```

Хоча *Kubernetes* дозволяє працювати з *Pod* напряму, в такому випадку *Pod* не буде контролюватись оркестратором *Kubernetes*.
*Pod* не буде перестворюватись у випадку відмови, та її не можна буде масштабувати. Для цього потрібно створити *Deployment*.

Перед тим як продовжити, видалимо створену *Pod*

```
kubectl delete pod nginx-pod
```

***Deployment*** - це конфігурація *Kubernetes*, що дозволяє описувати бажаний стан системи і являє собою набір з декількох однакових *Pod* без унікальних ідентифікаційних даних.
На основі конфігурації *Deployment*, *Deployment Controller* запускає задану кількість реплік вашої програми та автоматично замінює будь-які екземпляри, які не працюють або не реагують.
Таким чином, *Deployment* допомагає забезпечити доступність одного або декількох екземплярів вашої програми.

Створити *Deployment* можна наступною командою:

```shell
kubectl create deployment nginx --image=nginx:alpine                         
```

Результат `kubectl get deployments` має виглядати наступним чином:

```log
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
nginx   1/1     1            1           5s
```

Результат  `kubectl get pods`:

```log
NAME                      READY   STATUS    RESTARTS   AGE
nginx-565785f75c-k5tmz    1/1     Running   0          2m19s
```

Після того як створено *Deployment*, його *Pod* будуть доступні в середині кластера *k8s*, але ззовні до них немає доступу.
Для того, щоб можна було отримати доступ до створених *Pod*, потрібно створити проксі,
який перенаправлятиме запити ззовні в приватну мережу *kubernetes*.

```shell
kubectl proxy
```

> Проксі запускається на `8001` порту, переконайтесь, що він не зайнятий іншим процесом

При запуску проксі *kubernetes* автоматично створює точки входу для кожного *Pod* на основі його імені.
Для того, щоб отримати ім'я *Pod* можемо подивитись список *Pod*, як було показано вище,
або за допомогою наступної команди, одразу збережемо ім'я поди в змінну середовища:

```shell
export POD_NAME=$(kubectl get pods -o go-template --template '{{range .items}}{{.metadata.name}}{{"\n"}}{{end}}')
```

Звернутись до запущеної поди з командного рядка можна за допомогою `curl`:
```shell
curl http://localhost:8001/api/v1/namespaces/default/pods/$POD_NAME/proxy/
```

Або відкрити у браузері
```shell
http://localhost:8001/api/v1/namespaces/default/pods/nginx-565785f75c-k5tmz/proxy/
```

При роботі з *Pod* можна використовувати наступні команди, для перевірки роботи

- Для виводу логів: `kubectl logs $POD_NAME`
- Для виконання команди в контейнері (наприклад переглянути змінні середовища): `kubectl exec $POD_NAME -- env`

> Дані команди працюватимуть, якщо в *Pod* запущений один контейнер

### 3.2 Розгортання додатку. Створення *Service*

*Pod* не надійний елемент в *Kubernetes* вони можуть вмирати, і після цього вони ніколи не відновлюються,
на їх місце будуть створені нові *Pod*, щоб забезпечити стабільну роботу додатку, але доступ до них буде змінено, *IP* адреси будуть інші.

**Service** у *Kubernetes* - це абстракція, що об'єднує логічний набір *Pod* і забезпечує доступ до них
Набір *Pod*, призначених для *Service*, зазвичай визначається через [`selector`](k8s/service1/service1-service.yaml#L10).

Для того, щоб створити *Service* потрібно виконати наступну команду:

```shell
kubectl expose deployment/nginx --type="NodePort" --port 8080
```

```shell
kubectl get services

NAME         TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
kubernetes   ClusterIP   10.96.0.1      <none>        443/TCP          8d
nginx        NodePort    10.96.118.59   <none>        8080:32689/TCP   150m
```

Тепер, до запущеного додатка можна звернутись через *NodePort*, який, в даному випадку - `32689`

```shell
export NODE_PORT=$(kubectl get services/nginx -o go-template='{{(index .spec.ports 0).nodePort}}')
```

Виконавши цю команду, отримаємо початкову сторінку *nginx*

```shell
curl $(minikube ip):$NODE_PORT
```

Також варто звернути увагу, що всі сутності, які ми створили об'єднані однаковими мітками.
Перевірити це можна виконавши наступні команди:

```
kubectl get deployments --show-labels
kubectl get pods --show-labels
kubectl get services --show-labels
```

Перед тим як продовжити, потрібно очистити створені сутності.
Якщо видалити запущену *Pod*, *Deployment Controller* запустить на її місце іншу.
Тому для очищення ресурсів потрібно видалити *Deployment*

```
kubectl delete deployment nginx
```

Видалення сервісу відбувається аналогічно до інших сутностей

```
kubectl delete service nginx
```

### 3.3 Розгортання додатку. Створення сутностей за допомогою файлів конфігурацій

До цього для створення компонент в *k8s* ми використовували командний рядок.
Такий підхід добре підходить для демонстрацій, але в рельний проектах практично не використовується.

Кожна конфігурація має містити наступні поля:

- `apiVersion` - версія *Kubernetes API* для створення об'єкту (для кожного об'єкту ця версія може бути різною)
- `kind` - Тип сутності (*Deployment*, *Service*, *Ingress*, ...)
- `metadata` - Допомагає ідентифікувати екземпляр об'єкту, включає такі поля, як `name`, `UID`, та `namespace`
- `spec` - Описує стан об'єкту, унікальний для кожного типу сутності

В директорії `k8s` міститься мінімальний набір конфігурацій для розгортання застосунку.
Демонстраційний додаток складається з 2-х примітивних сервісів, що мають одну точку входу по якій повертають повідомлення.
Кожен сервіс має містити 2 файли конфігурацій
1) `deployment.yaml` - для створення *Deployment* та *Pod*
2) `service.yanl`  - для створення *Service*

Для того, щоб розгорнути `service1` потрібно виконати наступне:

1) Створити образ *Docker* для додатку `service1`

> При роботі з локальними образами, перед створенням образу потрібно виконати
> `eval $(minikube docker-env)`, цю команду потрібно запускати в кожному вікні терміналу
> В налаштуванні вказати `imagePullPolicy: Never`, щоб *k8s* не намагався завантажити образ

```
docker build -t service1:0.1 -f services/service1/Dockerfile .
```

2) Розгорнути *Deployment*

```
kubectl apply -f k8s/service1/deployment.yaml
```

> При першому запуску можна використовувати
> `kubectl create -f k8s/service1/deployment.yaml`, яка так само створить *Deployment*
> Але ця команда видасть помилку, якщо *Deployment* вже створено,
> тоді як `apply` створить Deployment, якщо його немає, або оновить, якщо він існує.

3) Додати *Service*

```
kubectl apply -f k8s/service1/service1-service.yaml
```

Щоб перевірити, що все працює можна створити проксі

```shell
kubectl proxy
```

Та зробити запит на наступний *URL*

```shell
curl http://localhost:8001/api/v1/namespaces/default/services/service1-service/proxy/api/service1
```

Аналогічно можна запустити `service2` та клієнт

##### 4. Налаштування доступу до додатку за допомогою *Ingress*

На даний момент, має бути запущено 2 сервіси, що доступні по адресах (з увімкненим проксі):

```shell
http://localhost:8001/api/v1/namespaces/default/services/service1-service/proxy/api/service1
http://localhost:8001/api/v1/namespaces/default/services/service2-service/proxy/api/service2
```

Для реальних додатків використовувати `kubectl proxy` не можна, оскільки це відкриває доступ до внутрішньої мережі, і створить проблеми з безпекою.

Є 3 варіанти, як можна відкрити доступ до додатку без використання проксі

1) Змінити тип *Service* з *ClusterIp* на *NodePort*. Та застосувати зміни за допомогою команди `kubectl apply -f k8s/service1/`
   Тепер сервіси будуть доступні ззовні кластеру *k8s* за адресами:

```shell
curl $(minikube ip):$NODE_PORT
```

Цей підхід має ряд проблем, через які також не використовується в реальному середовищі:
- Можна запустити лише один сервіс на одному порту
- Порт можна вибрати лише в рамках 30000–32767
- Якщо *IP* адреса вузла чи віртуальної машини змінилась, це потрібно якось обробити

> Детальніше можна почитати [тут](https://medium.com/google-cloud/kubernetes-nodeport-vs-loadbalancer-vs-ingress-when-should-i-use-what-922f010849e0)

2) Використати тип сервісу *LoadBalancer* цей тип доступний лише для хмарних провайдерів,
   і деталі реалізації також залежать від провайдера. Цей тип не буде розглядатись в даному курсі

3) Використати *Ingress*

**Ingress** - це об'єкт *API*, що контролює зовнішній доступ до сервісів в кластері *k8s*, як правило через *HTTP*.

> Перед використанням, потрібно увімкнути розширення для *minikube* `minikube addons enable ingress`

Для кожного сервісу визначаємо вланий *Ingress* знаходиться у файлі `k8s/{service}/ingress.yaml`

Застосувати її можна аналогічно до інших конфігурацій

```shell
kubectl apply -f k8s/service1/ingress.yaml
```

> Команди в *Kubernetes* можна запускати не лише на рівні файлів, а й на рівні директорій.
> `kubectl apply -f k8s/service1` застосує конфігурації з трьох файлів `ingress.yaml` `deployment.yaml` та `service.yaml`
 
Тепер сервіси знаходяться на одному фіксованому порту, і для зовнішніх клієнтів виглядають як один додаток, а не різні його частини

```
curl $(minikube ip)/api/service1
curl $(minikube ip)/api/service2
```
Клієнт можна відкрити в браузері за адресою `$(minikube ip)`

##### Додатково

На сайті kubernetes є хороший [інтерактивний курс](https://kubernetes.io/uk/docs/tutorials/kubernetes-basics/deploy-app/deploy-interactive/)
