# Домашнее задание к занятию "`Хранение в K8s. Часть 2`" - `Дедюрин Денис`

---
## Задание 1

Создать **Deployment** приложения, использующего локальный **PV**, созданный вручную.

1. Создать **Deployment** приложения, состоящего из контейнеров **busybox** и **multitool**.
2. Создать **PV** и **PVC** для подключения папки на локальной ноде, которая будет использована в поде.
3. Продемонстрировать, что **multitool** может читать файл, в который **busybox** пишет каждые пять секунд в общей директории. 
4. Удалить **Deployment** и **PVC**. Продемонстрировать, что после этого произошло с **PV**. Пояснить, почему.
5. Продемонстрировать, что файл сохранился на локальном диске ноды. Удалить **PV**.  Продемонстрировать что произошло с файлом после удаления **PV**. Пояснить, почему.
5. Предоставить манифесты, а также скриншоты или вывод необходимых команд.

### Ответ:

1. Создаем манифест **Deployment** приложения, состоящего из контейнеров **busybox** и **multitool**:

**deployment.yaml**

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mt-bb-pv-pvc
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mt-bb
  template:
    metadata:
      labels:
        app: mt-bb
    spec:
      containers:
        - name: busybox
          image: busybox
          command: ["/bin/sh", "-c", "while true; do echo $(date) > /data/hello.txt; sleep 5; done"]
          volumeMounts:
            - name: data
              mountPath: /data
        - name: multitool
          image: wbitt/network-multitool
          command: ["/bin/sh", "-c", "while true; do if [ -f /data/hello.txt ]; then cat /data/hello.txt; fi; sleep 2; done"]
          volumeMounts:
            - name: data
              mountPath: /data
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: local-pvc
```

2. Создаем манифесты **PV** и **PVC** для подключения папки на локальной ноде:

**local-pv.yaml**

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: local-storage
  local:
    path: /home/dnd/hw-k8s-storage-2/data
  nodeAffinity:
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: kubernetes.io/hostname
              operator: In
              values:
                - dnd-virtualbox
```

**local-pvc.yaml**

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: local-pvc
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: local-storage
  resources:
    requests:
      storage: 1Gi
```

3. Применяем все манифесты:

```
sudo kubectl apply -f local-pv.yaml
```

```
sudo kubectl apply -f local-pvc.yaml
```

```
sudo kubectl apply -f deployment.yaml
```

```
sudo kubectl get pv
```

```
sudo kubectl get pvc
```

```
sudo kubectl get deployment
```

```
sudo kubectl get pods
```

<img src = "img/01.png" width = 100%>

Проверяем, что **multitool** может читать файл, в который пишет **busybox**:

```
sudo kubectl logs -f mt-bb-pv-pvc-987d58d5b-cvwxb -c multitool
```

<img src = "img/02.png" width = 100%>

4. Удаляем **Deployment** и **PVC** и проверяем что после этого произошло с **PV**:

```
sudo kubectl delete deployment mt-bb-pv-pvc
```

```
sudo kubectl delete pvc local-pvc
```

```
sudo kubectl get pv
```

<img src = "img/03.png" width = 100%>

Видим, что **PV** перешёл в статус **Released**. Это значит, что **PVC** была удалена, но сам том остался. Так как в настройках указано `persistentVolumeReclaimPolicy: Retain`, **Kubernetes** не удаляет том и не трогает файлы. Данные на диске остались.

5. Проверяем, что файл сохранился на локальном диске ноды:

```
cat /home/dnd/hw-k8s-storage-2/data/hello.txt
```

<img src = "img/04.png" width = 100%>

Удаляем **PV** и ещё раз проверяем что произошло с файлом после удаления **PV**:

```
sudo kubectl delete pv local-pv
```

```
cat /home/dnd/hw-k8s-storage-2/data/hello.txt
```

<img src = "img/05.png" width = 100%>

Видим, что файл не удалился, даже после удаления **PV**.

---
## Задание 2

Создать **Deployment** приложения, которое может хранить файлы на **NFS** с динамическим созданием **PV**.

1. Включить и настроить **NFS-сервер** на **MicroK8S**.
2. Создать **Deployment** приложения состоящего из **multitool**, и подключить к нему **PV**, созданный автоматически на сервере **NFS**.
3. Продемонстрировать возможность чтения и записи файла изнутри пода. 
4. Предоставить манифесты, а также скриншоты или вывод необходимых команд.

### Ответ:

1. Включаем **NFS-сервер** на **MicroK8S**:

```
microk8s enable hostpath-storage
```

<img src = "img/06.png" width = 100%>

Проверяем что он появился:

```
sudo kubectl get storageclass
```

<img src = "img/07.png" width = 100%>

2. Создаем манифест **Deployment** приложения:

**deployment-nfs.yaml**

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: multitool-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: microk8s-hostpath
  resources:
    requests:
      storage: 1Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: multitool
spec:
  replicas: 1
  selector:
    matchLabels:
      app: multitool
  template:
    metadata:
      labels:
        app: multitool
    spec:
      containers:
        - name: multitool
          image: wbitt/network-multitool
          volumeMounts:
            - name: multitool-storage
              mountPath: /mnt/data
      volumes:
        - name: multitool-storage
          persistentVolumeClaim:
            claimName: multitool-pvc
```

Применяем манифест:

```
sudo kubectl apply -f deployment-nfs.yaml
```

<img src = "img/08.png" width = 100%>

```
sudo kubectl get pods
```

<img src = "img/09.png" width = 100%>

3. Пробуем проверить возможность записи и чтения файла изнутри пода:

```
sudo kubectl exec -it multitool-f86c7d557-kzp22 -- /bin/sh
```

```
echo "hostpath works!" > /mnt/data/test.txt
```

```
cat /mnt/data/test.txt
```

<img src = "img/10.png" width = 100%>

Проверяем на хостовой машине хранилище и созданный файл:

```
ls /var/snap/microk8s/common/default-storage/
```

```
cat /var/snap/microk8s/common/default-storage/default-multitool-pvc-pvc-45a2a4d4-7f28-42ed-bd7f-be3c4003566a/test.txt
```

<img src = "img/11.png" width = 100%>

