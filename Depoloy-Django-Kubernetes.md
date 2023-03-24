## Первичная настройка
* Выбираем название проекта, например `example`
* С помощью команды `kubectl get node` получаем имя ноды (уточнить у девопса), например `main-node-t5q3g`
* Namespace - `example-ns`
* Role - `node-role.kubernetes.io/Worker` (уточнить у девопса)
* Имя сервиса - `example-service`
* Доменные имена dev сервер - `example-dev.foxdev.in`, stage - `example-stage.foxdev.in`
```bash
# example - Это название namespace
KUBE_NAME=example
KUBE_NODE=main-node-t5q3g
KUBE_NS=$KUBE_NAME-ns
KUBE_IMAGE_PULL=$KUBE_NS-image-pull
KUBE_SERVICE_NAME=$KUBE_NAME-service
KUBE_SECRET=$KUBE_NAME-secret-ts
KUBE_USER_EMAIL=company.user@gmail.com
KUBE_ROLE=node-role.kubernetes.io/Worker
DEV_DOMAIN=$KUBE_NAME-dev.foxdev.in
DEV_STAGE=$KUBE_NAME-stage.foxdev.in
```
====Создаем namespace для проекта====

```bash
# example - Это название namespace
kubectl create namespace $KUBE_NS
```

====Добавляем объект Image Pull Policy GitLab====

Чтобы GitLab Runner мог деплоить приложение в кластер Kubernetes, 
необходимо дать ему соответствующие права внутри конкретного namespace. 
Для этого потребуется создать несколько RBAC-объектов:

```bash
# создать сервисную учетную запись (УЗ) командой:
kubectl create sa deploy -n $KUBE_NS

# предоставить ей права на создание объектов внутри namespace example_namespace

kubectl create rolebinding deploy \
  -n $KUBE_NS \
  --clusterrole edit \
  --serviceaccount $KUBE_NS:deploy
```
  
Токен созданной УЗ необходимо прописать в GitLab-переменных. 
С помощью этого токена GitLab Runner сможет получать доступ к кластеру Kubernetes. 
Для получения токена воспользуйтесь командой:

```bash
kubectl get secret -n $KUBE_NS \
  $(kubectl get sa -n  $KUBE_NS deploy \
    -o jsonpath='{.secrets[].name}') \
  -o jsonpath='{.data.token}'
```

Создаем переменную `K8S_CI_TOKEN` и ее заполняем ее результатов выполнеения последнего действия (без `%` в конце строки):
file:/homepage/developers-wiki/devops/razvorachivaniedjangpproektanakubernetes/2022-09-2316-20.png

Скопируйте токен в буфер обмена и вернитесь в интерфейс GitLab. В пункте меню «Settings» — «CI/CD» нужно нажать на кнопку «Expand» в секции «Variables» и затем на кнопку «Add variable».

В открывшейся форме требуется указать имя переменной, которое будет впоследствии использоваться в секции deploy манифеста gitlab-ci.yml — K8S_CI_TOKEN. В качестве значения переменной нужно использовать скопированный ранее токен сервисной УЗ. Также следует снять флажок «Protect variable», который разрешает доступ к переменным исключительно из ветки master. И нужно установить флажок «Mask variable» — в таком случае переменная будет закрытой, то есть ее значение не будет отображаться внутри логов CI/CD.

Последнее, что необходимо сделать перед запуском деплоя, — добавить в Kubernetes возможность авторизации в GitLab, чтобы получать из Docker Registry формируемый образ и запускать на его основе приложение. Для этого нужно добавить Deploy token.

В GitLab необходимо выбрать пункт меню «Repository» и нажать на кнопку «Expand» в секции «Deploy tokens».

В открывшейся форме необходимо ввести имя токена `k8s-pull-token` и установить флажок «read_registry», который будет означать доступ к образам только на чтение.

Сформированные имя пользователя и пароль нужно скопировать в следующую команду, после чего выполнить ее в консоли. Команда добавит секрет с типом `docker-registry` и именем `example-image-pull`. С помощью секрета Kubernetes сможет получать формируемые образы из GitLab.

```bash
kubectl create secret docker-registry $KUBE_IMAGE_PULL \ 
  --docker-server registry.gitlab.com \
  --docker-email 'admin@mycompany.com' \
  --docker-username '<gitlab_token_name>' \
  --docker-password '<gitlab_token_password>' \
  --namespace $KUBE_NS
```

====Добавляем манифест для Django проекта====

* Создаем файл манифеста kubernetes/web_deployment.yaml, 
* Добавляем в манифест переменные окружения

```bash
mkdir kubernetes
touch kubernetes/web_deployment.yaml
code -r kubernetes/web_deployment.yaml

echo '
apiVersion: apps/v1
# Тип размещения Deployment
kind: Deployment
metadata:
  # Представление пода 
  name: <KUBE_NAME>
  # Пространство имен проекта
  namespace: <K8S_NAMESPACE>
  labels:
    # Метка деплоймента
    app: <KUBE_NAME>
spec:
  # Колиkчество реплик - 1
  replicas: 1
  selector:
    matchLabels:
      # Метка пода
      app: <KUBE_NAME>
  template:
    metadata:
      labels:
        # Метка контейнера
        app: <KUBE_NAME>
    spec:
      imagePullSecrets:
        # Секрет для получения docker изображения из приватного хранилища (GitLab)
        # Ниже будет инструкция по его созданию
        - name: <KUBE_IMAGE_PULL>
      containers:
        # Образ для контейнера
        - image: <IMAGE> 
        # Название контейнера
          name: web
        # Команда запуска проекта
          command: ["/bin/sh"]
          args: ["-c", "python manage.py migrate --no-input 
            && python manage.py collectstatic --no-input 
            && python manage.py createcachetable
            && gunicorn conf.asgi:application -k uvicorn.workers.UvicornWorker --workers=1 --timeout 30 --max-requests=10000 --bind 0.0.0.0:8000"]
          envFrom:
            - configMapRef:
                # Название configmap от куда будут подтягиваться переменные окружения
                name: <K8S_ENV>
          resources:
           # Количество запрашиваемых ресурсов для контейнера при старте
            requests:
              cpu: 100m
              memory: 500Mi
           # Лимит ресурсов для контейнера
            limits:
              cpu: 300m
              memory: 1000Mi 
          # Указываем палитику для загрузки образа контейнера
          # Данная политика указывает что если образ уже загружен на ноду то не загружать его снова                          
          imagePullPolicy: IfNotPresent
          # Подставляется при CI/CD Gitlab
          ports:
            # Порт на котором будет запущено приложение
            - containerPort: 8000
          volumeMounts:
            # Название контейнера шарится внутри деплоймента в любой контейнер
            # Если нужно шарить глобально в ноде нужно создавать Persictem Volume
            - name: static
              mountPath: /app/static/
              
        # Если нужен Celery
        # Контейнер celery-beat
        - image: <IMAGE> 
          name: celery-beat
          command: [celery, -A, conf, beat, -l, DEBUG]
          envFrom:
            - configMapRef:
                name: <K8S_ENV>
          resources:
            requests:
              cpu: 2m
              memory: 116Mi
            limits:
              cpu: 100m
              memory: 200Mi 
          imagePullPolicy: IfNotPresent
          
        # Контейнер celery-worker
        - image: <IMAGE>
          name: celery-worker
          command: [celery, -A, conf, worker, -l, INFO]
          envFrom:
            - configMapRef:
                name: <K8S_ENV>
          resources:
            requests:
              cpu: 4m
              memory: 209Mi
            limits:
              cpu: 10m
              memory: 400Mi 
          imagePullPolicy: IfNotPresent
          
        # Контейнер для nginx
        - image: nginx:1.19.0-alpine
          name: nginx
          ports:
          - containerPort: 80
          resources:
            requests:
              cpu: 2m
              memory: 30Mi
            limits:
              cpu: 100m
              memory: 200Mi 
          imagePullPolicy: IfNotPresent

          volumeMounts:
            # Прокидываем конфиг в nginx
            # Название configmap для nginx
            - name: nginx-config
              mountPath: /etc/nginx/conf.d/
            # Название configmap для web
            - name: static
              mountPath: /app/static/
        
      volumes:
        - name: static
        - name: nginx-config
          configMap:
            name: nginx-config
      # Нод селектор при планировки деплоймента будут искаться ноды с этой меткой 
      nodeSelector:
        nodePool: cluster 
      restartPolicy: Always

---
apiVersion: v1
# Разворачиваем сервис, который будет пробрасывать порты контейнера в ноду
kind: Service
metadata:
  labels:
    # Метка сервиса, через нее будут идти обращения в поды Django из других контейнеров
    app: <KUBE_SERVICE_NAME>
  # Представление сервиса
  name: <KUBE_SERVICE_NAME>
  # Пространство имен тоже что и у проекта
  namespace: <K8S_NAMESPACE>
spec:
  selector:
    # Метка пода из matchLabels проекта
    app: <KUBE_NAME>
  ports:
    - protocol: TCP
    # Название порта
      name: web-port
      port: 80
      targetPort: 80
' > kubernetes/web_deployment.yaml

sed -i -e "s,<KUBE_NAME>,$KUBE_NAME,g" kubernetes/web_deployment.yaml
sed -i -e "s,<KUBE_SERVICE_NAME>,$KUBE_SERVICE_NAME,g" kubernetes/web_deployment.yaml
sed -i -e "s,<KUBE_IMAGE_PULL>,$KUBE_IMAGE_PULL,g" kubernetes/web_deployment.yaml

```

* Создаем файл манифеста kubernetes/redis_deployment.yaml
```bash
echo '
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
  namespace: <KUBE_NS>
  labels:
    deployment: redis
spec:
  selector:
    matchLabels:
      pod: redis
  replicas: 1
  template:
    metadata:
      labels:
        pod: redis
    spec:
      containers:
        - name: redis
          image: redis
          resources:
            limits:
              cpu: 20m
              memory: 200Mi
            requests:
              cpu: 5m
              memory: 10Mi
          ports:
          - containerPort: 6379
      nodeSelector:
        nodePool: cluster 

---
apiVersion: v1
kind: Service
metadata:
  name: redis-service
  namespace: <KUBE_NS>
spec:
  selector:
    pod: redis
  ports:
  - protocol: TCP
    port: 6379
    targetPort: 6379
' > kubernetes/redis_deployment.yaml

sed -i -e "s,<KUBE_NS>,$KUBE_NS,g" kubernetes/redis_deployment.yaml
```

```yaml
apiVersion: apps/v1
# Тип размещения Deployment
kind: Deployment
metadata:
  # Представление пода 
  name: <KUBE_NAME>
  # Пространство имен проекта
  namespace: <KUBE_NS>
  labels:
    # Метка деплоймента
    app: <KUBE_NAME>
spec:
  # Колиkчество реплик - 1
  replicas: 1
  selector:
    matchLabels:
      # Метка пода
      app: <KUBE_NAME>
  template:
    metadata:
      labels:
        # Метка контейнера
        app: <KUBE_NAME>
    spec:
      imagePullSecrets:
        # Секрет для получения docker изображения из приватного хранилища (GitLab)
        # Ниже будет инструкция по его созданию
        - name: example-image-pull
      containers:
        # Образ для контейнера
        - image: <IMAGE> 
        # Название контейнера
          name: web
        # Команда запуска проекта
          command: ["/bin/sh"]
          args: ["-c", "python manage.py migrate --no-input && python manage.py collectstatic --no-input && gunicorn conf.asgi:application -k uvicorn.workers.UvicornWorker --workers=1 --timeout 30 --max-requests=10000 --bind 0.0.0.0:8000"]
          envFrom:
            - configMapRef:
                # Название configmap от куда будут подтягиваться переменные окружения
                name: env
          resources:
           # Количество запрашиваемых ресурсов для контейнера при старте
            requests:
              cpu: 100m
              memory: 500Mi
           # Лимит ресурсов для контейнера
            limits:
              cpu: 300m
              memory: 1000Mi 
          # Указываем палитику для загрузки образа контейнера
          # Данная политика указывает что если образ уже загружен на ноду то не загружать его снова                          
          imagePullPolicy: IfNotPresent
          # Подставляется при CI/CD Gitlab
          ports:
            # Порт на котором будет запущено приложение
            - containerPort: 8000
          volumeMounts:
            # Название контейнера шарится внутри деплоймента в любой контейнер
            # Если нужно шарить глобально в ноде нужно создавать Persictem Volume
            - name: static
              mountPath: /app/static/
              
        # Если нужен Celery
        # Контейнер celery-beat
        - image: <IMAGE> 
          name: celery-beat
          command: [celery, -A, conf, beat, -l, DEBUG]
          envFrom:
            - configMapRef:
                name: env
          resources:
            requests:
              cpu: 2m
              memory: 116Mi
            limits:
              cpu: 100m
              memory: 200Mi 
          imagePullPolicy: IfNotPresent
          
        # Контейнер celery-worker
        - image: <IMAGE>
          name: celery-worker
          command: [celery, -A, conf, worker, -l, INFO]
          envFrom:
            - configMapRef:
                name: env
          resources:
            requests:
              cpu: 4m
              memory: 209Mi
            limits:
              cpu: 10m
              memory: 400Mi 
          imagePullPolicy: IfNotPresent
          
        # Контейнер для nginx
        - image: nginx:1.19.0-alpine
          name: nginx
          ports:
          - containerPort: 80
          resources:
            requests:
              cpu: 2m
              memory: 30Mi
            limits:
              cpu: 100m
              memory: 200Mi 
          imagePullPolicy: IfNotPresent

          volumeMounts:
            # Прокидываем конфиг в nginx
            # Название configmap для nginx
            - name: nginx-config
              mountPath: /etc/nginx/conf.d/
            # Название configmap для web
            - name: static
              mountPath: /app/static/
        
      volumes:
        - name: static
        - name: nginx-config
          configMap:
            name: nginx-config
      # Нод селектор при планировки деплоймента будут искаться ноды с этой меткой 
      nodeSelector:
        nodePool: cluster 
      restartPolicy: Always

---
apiVersion: v1
# Разворачиваем сервис, который будет пробрасывать порты контейнера в ноду
kind: Service
metadata:
  labels:
    # Метка сервиса, через нее будут идти обращения в поды Django из других контейнеров
    app: <KUBE_SERVICE_NAME>
  # Представление сервиса
  name: <KUBE_SERVICE_NAME>
  # Пространство имен тоже что и у проекта
  namespace: <KUBE_NS>
spec:
  selector:
    # Метка пода из matchLabels проекта
    app: <KUBE_NAME>
  ports:
    - protocol: TCP
    # Название порта
      name: web-port
      port: 80
      targetPort: 80
```

====Создаем configmap для web====
* Создаем файл web-configmap.yaml
* Добавляем в файл

```bash
echo '
apiVersion: v1
kind: ConfigMap
metadata:
  # Название configmap которое указавали в диплоймент выше
  name: env
  namespace: <KUBE_NS>
data:
# Переменнве указываем в формате yaml
# Каждая переменная с новой строки
  DOMAIN_NAME: <DEV_DOMAIN>' > kubernetes/web-configmap.yaml
 
sed -i -e "s,<DEV_DOMAIN>,$DEV_DOMAIN,g" kubernetes/web-configmap.yaml
sed -i -e "s,<KUBE_NS>,$KUBE_NS,g" kubernetes/web-configmap.yaml
```

====Создаем configmap для nginx-cong====
* Создаем файл nginx-configmap.yaml
* Добавляем в файл

```bash
echo '
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
  namespace: <KUBE_NS>
data:
  default.conf: |
    upstream web {
        server localhost:8000;
    }

    server {
        listen 80;
        listen [::]:80;

        client_max_body_size 100M;
        
        server_name <DEV_DOMAIN>;

        location / {
            proxy_pass http://web ;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header Host $host;
            proxy_redirect off;
        }

        location /static/ {
            alias /app/static/;
        }
        ' > kubernetes/nginx-configmap.yaml 

sed -i -e "s,<DEV_DOMAIN>,$DEV_DOMAIN,g" kubernetes/nginx-configmap.yaml
sed -i -e "s,<KUBE_NS>,$KUBE_NS,g" kubernetes/nginx-configmap.yaml
```

====Разворачиваем configmap в kubernetes====
```bash
kubectl apply -f web-configmap.yaml
kubectl apply -f nginx-configmap.yaml
```

===Добавляем объект ClusterIssure===
* Этот объект говорит cert-manager выписывать сертификаты для ingress в текущем namespace
* Создаем файл cluster-issure.yaml
* Добавляем в файл

```bash
echo '
apiVersion: cert-manager.io/v1
# Содание эмитента для получения сертификата Letsencrypt
kind: ClusterIssuer
metadata:
  # Название эмитента обязательно использовать это название letsencrypt-prod
  name: letsencrypt-prod
  namespace: <KUBE_NS>
spec:
  # Automated Certificate Management Environment (ACME)
  # При определении ACME cert-manager будет автоматически генерировать privat key
  # для идентификации пользователя на ACME сервере
  acme:
    # Ссылка для получения сертификата
    server: https://acme-v02.api.letsencrypt.org/directory
    email: <KUBE_USER_EMAIL>
    # Подключение приватного ключа для эмитента
    privateKeySecretRef:
      # Название эмитента
      name: prod-issuer-account-key
    # Добавьте единственное средство решения проблем, HTTP01, используя nginx
    solvers:
    - http01:
        ingress:
          class: nginx
    ' > kubernetes/cluster-issure.yaml

sed -i -e "s,<KUBE_USER_EMAIL>,$KUBE_USER_EMAIL,g" kubernetes/cluster-issure.yaml
sed -i -e "s,<KUBE_NS>,$KUBE_NS,g" kubernetes/cluster-issure.yaml
kubectl apply -f kubernetes/cluster-issure.yaml
```

===Добавляем объект Ingress===
* Объект для получения ssl сертификата и маршрутизации трафика
* Создаем файл ingress-rulles.yaml
* Добавляем в файл

```bash
echo '
apiVersion: networking.k8s.io/v1
#Правила маршрутизации ingress
kind: Ingress
metadata:
  # Название ingresss
  name: ingress-rule
  # Пространство имен где будет развернут ingress
  namespace: <KUBE_NS>
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: 30m
    # Анатации необходимые для подключени cert-manager и ingress-nginx
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/rewrite-target: / 
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    ingress.kubernetes.io/ssl-redirect: "true"

spec:
  tls:
  # Указываем cert-manager какому хосту выписать сертифика 
    - hosts:
      - <DEV_DOMAIN>
      # Название которое будет присвоино сертификату
      secretName: <KUBE_SECRET>
  rules:
  # Перенаправляем трафик в наше приложение
  - host: <DEV_DOMAIN>
    http:
      paths:
        - pathType: Prefix
          path: "/"
          backend:
            # Сервис нашего прилочения
            service:
              name: <KUBE_SERVICE_NAME>
              port: 
                number: 80
          ' > kubernetes/ingress-rulles.yaml
                
sed -i -e "s,<KUBE_SERVICE_NAME>,$KUBE_SERVICE_NAME,g" kubernetes/ingress-rulles.yaml
sed -i -e "s,<KUBE_SECRET>,$KUBE_SECRET,g" kubernetes/ingress-rulles.yaml
sed -i -e "s,<KUBE_NS>,$KUBE_NS,g" kubernetes/ingress-rulles.yaml
sed -i -e "s,<DEV_DOMAIN>,$DEV_DOMAIN,g" kubernetes/ingress-rulles.yaml
kubectl apply -f kubernetes/ingress-rulles.yaml
```
===Добавляем файл .gitlab-ci.yml для настройки CI/CD===
* Все переменные которые указанны с префиксом CI_ не нужно не где создовать они подтянуться автоматически

* В gitlab задаем переменные окружения 

1. KUBERNETES_CLUSTER
2. KUBERNETES_CLUSTER_USER
3. K8S_CI_TOKEN
4. KUB_SERVER

* Значение переменных нужно взять из файла ./kube/config

- NAMESPACE ветки
1. K8S_NAMESPACE
- Название configmap
2. K8S_ENV

```bash
echo '
# Приложение собирается внутри gitlab образ тоже храниться в gitlab
build_kube:
  # образ докера
  image: docker:19.03.12
  # Указываем что будем билдить
  stage: build
  # Переменое окружение из корого будут браться переменные обычно соответствует названию ветки
  environment:
    name: dev
  services:
    - docker:19.03.12-dind
  variables:
    DOCKER_HOST: tcp://docker:2375
    DOCKER_TLS_CERTDIR: ""
    # Эту переменную указываем если во время сборки нужно подтянуть проект из другого репозитория
    GIT_SUBMODULE_STRATEGY: recursive
  before_script:
    - docker login -u "$CI_REGISTRY_USER" -p "$CI_REGISTRY_PASSWORD" $CI_REGISTRY
  script:
    docker build -t "$CI_REGISTRY_IMAGE:$CI_COMMIT_REF_SLUG.$CI_PIPELINE_ID" .;
    docker push "$CI_REGISTRY_IMAGE:$CI_COMMIT_REF_SLUG.$CI_PIPELINE_ID";
  only:
    - dev
    
deploy_kube:
  stage: deploy
  environment:
    name: dev
  image: 
    name: bitnami/kubectl:1.20
    entrypoint: [""]
  before_script:
  # Set kubernetes credentials
    - export KUBECONFIG=/tmp/kubeconfig
    - kubectl config set-cluster $KUBERNETES_CLUSTER --insecure-skip-tls-verify=true --server="$(echo $KUB_SERVER)"
    - kubectl config set-credentials $KUBERNETES_CLUSTER_USER --token="$(echo $K8S_CI_TOKEN | base64 -d)"
    - kubectl config set-context $KUBERNETES_CLUSTER_USER --cluster=$KUBERNETES_CLUSTER --user=$KUBERNETES_CLUSTER_USER  
    - kubectl config use-context $KUBERNETES_CLUSTER_USER
  script:
    sed -i -e "s,<IMAGE>,$CI_REGISTRY_IMAGE:$CI_COMMIT_REF_SLUG.$CI_PIPELINE_ID,g" kubernetes/web-deployment.yaml;
    sed -i -e "s,<K8S_NAMESPACE>,$K8S_NAMESPACE,g" kubernetes/web-deployment.yaml;
    sed -i -e "s,<K8S_ENV>,$K8S_ENV,g" kubernetes/web-deployment.yaml;
    kubectl apply -f kubernetes/web-deployment.yaml;
  only:
    - dev
  ' > .gitlab-ci.yml
```
* Комитем все измения и выполняем **git push**
* Проверяем что все собралось и залилось

====Полезные alias для работы с kubctl====
```bash
#Kubernetes

#Pods
alias kgp='kubectl get pods'
alias kdp='kubectl describe pod'
alias ktp='kubectl top pod --use-protocol-buffers --containers '

#Nods
alias kgn='kubectl get node'
alias ktn='kubectl top node --use-protocol-buffers'

#Service
alias kgs='kubectl get svc'

#Order
alias kgo='kubectl get order'
alias kdo='kubectl describe order'

#ClusterIssure
alias kgci='kubectl get clusterissuer'
alias kdci='kubectl describe clusterissuer'

#Issure
alias kgis='kubectl get issuer'
alias kdis='kubectl describe issuer'

#Challange
alias kgch='kubectl get challenge'
alias kdch='kubectl describe challenge'

#CertificateRequest
alias kgcrr='kubectl get certificaterequest'
alias kdcrr='kubectl describe certificaterequest'

#Certificate
alias kgcr='kubectl get certificate'
alias kdcr='kubectl describe certificate'

alias kaf='kubectl apply -f'
alias kdf='kubectl delete -f'
```
