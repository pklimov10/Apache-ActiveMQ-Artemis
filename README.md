# Руководство по настройке высокодоступного кластера Apache ActiveMQ Artemis

## Содержание
- [Обзор](#обзор)
- [Изменения в конфигурации](#изменения-в-конфигурации)
- [Настройка HAProxy](#настройка-haproxy)
- [Настройка Keepalived](#настройка-keepalived)
- [Тестовые сценарии](#тестовые-сценарии)
- [Мониторинг и обслуживание](#мониторинг-и-обслуживание)
- [Устранение неисправностей](#устранение-неисправностей)

---

## Обзор

Это руководство описывает процесс настройки высокодоступного кластера Apache ActiveMQ Artemis. Кластер состоит из следующих компонентов:

### Компоненты
- **2 узла Apache ActiveMQ Artemis** в режиме active-active
- **2 узла HAProxy** для балансировки нагрузки
- **Keepalived** для обеспечения отказоустойчивости HAProxy

### Топология сети
- **Artemis Node 1**: `10.7.39.9`
- **Artemis Node 2**: `10.7.39.12`
- **Virtual IP (VIP)**: `192.168.1.100`

---

## Изменения в конфигурации

### Изменения в конфигурации Apache ActiveMQ Artemis

#### Настройки кластеризации
```<cluster-connections>
    <cluster-connection name="my-cluster">
        <address>jms,#</address>
        <connector-ref>broker2-connector</connector-ref>
        <message-load-balancing>STRICT</message-load-balancing>
        <max-hops>1</max-hops>
    </cluster-connection>
</cluster-connections>
```
```
      <!-- Коннекторы для связи между брокерами -->
      <connectors>
         <connector name="broker1-connector">tcp://10.7.39.9:61616?tcpSendBufferSize=1048576;tcpReceiveBufferSize=1048576</connector>
         <connector name="broker2-connector">tcp://10.7.39.12:61616?tcpSendBufferSize=1048576;tcpReceiveBufferSize=1048576</connector>
      </connectors>

```
```
     <!-- Акцепторы -->
      <acceptors>
         <acceptor name="artemis">tcp://10.7.39.12:61616?tcpSendBufferSize=1048576;tcpReceiveBufferSize=1048576;protocols=CORE,AMQP,STOMP,HORNETQ,MQTT,OPENWIRE;useEpoll=true;tcpKeepAlive=true</acceptor>
         <acceptor name="amqp">tcp://10.7.39.12:5672?protocols=AMQP;useEpoll=true</acceptor>
         <acceptor name="stomp">tcp://10.7.39.12:61613?protocols=STOMP;useEpoll=true</acceptor>
         <acceptor name="mqtt">tcp://10.7.39.12:1883?protocols=MQTT;useEpoll=true</acceptor>
      </acceptors>

```

```
      <!-- Настройка кластера для режима active-active -->
      <cluster-connections>
         <cluster-connection name="my-cluster">
            <address>jms,#</address>
            <connector-ref>broker2-connector</connector-ref>
            <retry-interval>500</retry-interval>
            <use-duplicate-detection>true</use-duplicate-detection>
            <message-load-balancing>STRICT</message-load-balancing>
            <max-hops>1</max-hops>
            <confirmation-window-size>1048576</confirmation-window-size>
            <static-connectors>
               <connector-ref>broker2-connector</connector-ref>
            </static-connectors>
         </cluster-connection>
      </cluster-connections>

```
### a. Настройка соединителей (<connectors>):

- На каждом брокере определите соединители для связи с собой и другими брокерами.

- На брокере artemis1 (192.168.1.101):

```<connectors>
   <connector name="artemis1-connector">tcp://192.168.1.101:61616</connector>
   <connector name="artemis2-connector">tcp://192.168.1.102:61616</connector>
</connectors>
```


### На брокере artemis2 (192.168.1.102):

```<connectors>
   <connector name="artemis1-connector">tcp://192.168.1.101:61616</connector>
   <connector name="artemis2-connector">tcp://192.168.1.102:61616</connector>
</connectors>
```

### b. Настройка акцепторов (<acceptors>):

- Оставьте стандартный акцептор для приема клиентских подключений.

```<acceptors>
   <acceptor name="netty">tcp://0.0.0.0:61616</acceptor>
</acceptors>
```


### c. Настройка кластерных соединений (<cluster-connections>):

- Добавьте кластерное соединение, которое объединяет брокеры в кластер.

- На обоих брокерах (artemis1 и artemis2):
```
<cluster-connections>
   <cluster-connection name="my-cluster">
      <address>jms</address>
      <connector-ref>artemis1-connector</connector-ref> <!-- Указывается локальный соединитель арбитража -->
      <retry-interval>1000</retry-interval>
      <use-duplicate-detection>true</use-duplicate-detection>
      <message-load-balancing>ON_DEMAND</message-load-balancing>
      <max-hops>1</max-hops>
      <static-connectors>
         <connector-ref>artemis1-connector</connector-ref>
         <connector-ref>artemis2-connector</connector-ref>
      </static-connectors>
   </cluster-connection>
</cluster-connections>
```


###  Оптимизация производительности
- **Размер TCP-буфера**: tcpSendBufferSize=1048576
- **Пулинг журналов**: <journal-pool-files>10</journal-pool-files>
- **Режим журнала**: ASYNCIO

###  Повышение надежности
- **Включен TCP KeepAlive**: tcpKeepAlive=true
-  Настроена политика повторной доставки сообщений
-  Включена дедупликация сообщений
-  Изменения в параметрах очередей

## Настройка HAProxy
Для балансировки нагрузки между узлами Apache ActiveMQ Artemis и обеспечения отказоустойчивости необходимо настроить HAProxy.

###Установка HAProxy:
```
bash
apt update && apt install haproxy
```
- Конфигурация HAProxy (/etc/haproxy/haproxy.cfg):
```text
global
    log /dev/log    local0
    maxconn 4096
    daemon
    # Добавляем пользователя для админки
    stats socket /var/lib/haproxy/stats mode 660 level admin
    stats timeout 30s

defaults
    log     global
    mode    tcp
    option  tcplog
    option  dontlognull
    option  tcpka
    timeout connect 30s
    timeout client  3600s
    timeout server  3600s

# Добавляем frontend для админки
frontend stats
    bind *:8404
    mode http
    stats enable
    stats uri /stats
    stats refresh 10s
    stats auth admin:your_password_here    # Замените на свой пароль
    stats admin if TRUE

frontend artemis_frontend
    bind *:61616
    mode tcp
    default_backend artemis_backend
    
backend artemis_backend
    mode tcp
    balance source                    
    stick-table type ip size 200k    
    stick on src                     
    
    server artemis1 10.7.39.9:61616 check inter 5s rise 2 fall 3 weight 100
    server artemis2 10.7.39.12:61616 check inter 5s rise 2 fall 3 weight 100
```

## Настройка Keepalived
- Установка Keepalived:
```
apt update && apt install keepalived
```
### Конфигурация Keepalived (/etc/keepalived/keepalived.conf)
```
vrrp_script check_haproxy {
    script "killall -0 haproxy"
    interval 2
    weight 2
}

vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 51
    priority 101
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass ваш_секретный_пароль
    }
    virtual_ipaddress {
        192.168.1.100/24
    }
    track_script {
        check_haproxy
    }
}
```
### Конфигурация Keepalived для Резервного узла (/etc/keepalived/keepalived.conf на резервном HAProxy)
```
vrrp_script check_haproxy {
    script "killall -0 haproxy"
    interval 2
    weight 2
}

vrrp_instance VI_1 {
    state BACKUP
    interface eth0
    virtual_router_id 51
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass ваш_секретный_пароль
    }
    virtual_ipaddress {
        192.168.1.100/24
    }
    track_script {
        check_haproxy
    }
}
```

```systemctl restart keepalived```

## Тестовые сценарии

### 1. Тест использования диска

Скрипт, который отправляет сообщения в очередь и отслеживает использование диска на сервере. Если использование диска превышает 90%, скрипт завершится с предупреждением.

```bash
#!/bin/bash
QUEUE_NAME="test.queue"
MESSAGE_SIZE="1M"
MESSAGE_COUNT=1000

for i in $(seq 1 $MESSAGE_COUNT); do
    echo "Отправка сообщения $i"
    artemis producer --destination $QUEUE_NAME --message-size $MESSAGE_SIZE --message-count 1
    
    DISK_USAGE=$(df -h | grep '/data' | awk '{print $5}' | cut -d'%' -f1)
    if [ $DISK_USAGE -gt 90 ]; then
        echo "Предупреждение: использование диска достигло ${DISK_USAGE}%"
        break
    fi
done
```

### Ожидаемые результаты:
- **Хороший результат:** Скрипт отправляет сообщения без ошибок, использование диска не превышает 90%.
- **Плохой результат:** Если использование диска превышает 90%, скрипт завершится с предупреждением, и процесс отправки сообщений остановится. Это может указывать на проблемы с недостаточной емкостью диска или неправильной настройкой хранения данных.

### 2. Тест отказоустойчивости кластера
   Скрипт запускает продюсера, который отправляет сообщения в очередь. Затем имитируется сбой одного из узлов, и проверяется, продолжает ли сервис работать через виртуальный IP (VIP).
   
```shell
#!/bin/bash
VIP="192.168.1.100"
QUEUE="test.failover.queue"

artemis producer --url tcp://$VIP:61616 --destination $QUEUE --message-count 1000 --threads 10 &

sleep 30
ssh artemis1 "systemctl stop artemis"

for i in {1..10}; do
    if nc -z $VIP 61616; then
        echo "Сервис доступен после failover"
        break
    fi
    sleep 1
done
```

### Ожидаемые результаты:
- **Хороший результат:** Сервис продолжает работать после сбоя одного из узлов, клиент может продолжать отправку сообщений.
- **Плохой результат:**  Если в течение теста сервис не доступен после сбоя одного из узлов (через VIP), это может свидетельствовать о проблемах с отказоустойчивостью или настройки HAProxy/Keepalived.

### 3. Тест производительности
Скрипт запускает продюсера для отправки большого количества сообщений в очередь, имитируя нагрузку на систему. В конце анализируются статистики очереди.
```shell
#!/bin/bash
VIP="192.168.1.100"
QUEUE="test.failover.queue"

artemis producer --url tcp://$VIP:61616 --destination $QUEUE --message-count 1000 --threads 10 &

sleep 30
ssh artemis1 "systemctl stop artemis"

for i in {1..10}; do
    if nc -z $VIP 61616; then
        echo "Сервис доступен после failover"
        break
    fi
    sleep 1
done
```

### Ожидаемые результаты:
- **Хороший результат:** Скрипт успешно завершает отправку всех сообщений и выводит статистику очереди. Ожидается, что нагрузка распределится равномерно между потоками, и система обработает все сообщения без сбоев.
- **Плохой результат:**  Если система не может обработать указанное количество сообщений (например, ошибки в процессе отправки или перегрузка системы), это указывает на проблемы с производительностью или настройкой брокера Artemis. Также, если в статистике очереди видны проблемы, такие как высокая задержка или потеря сообщений, это может указывать на узкие места в конфигурации.

### 3. Тест с остановкой HAProxy
Скрипт проверяет работу кластера и доступность виртуального IP (VIP) при остановке сервера HAProxy. Этот тест помогает проверить отказоустойчивость балансировщика нагрузки.
```shell
#!/bin/bash
VIP="192.168.1.100"
QUEUE="test.queue"

# Запуск продюсера
artemis producer --url tcp://$VIP:61616 --destination $QUEUE --message-count 1000 --threads 10 &

# Остановка HAProxy (для имитации сбоя балансировщика)
ssh haproxy-server "systemctl stop haproxy"

# Проверка, что VIP всё ещё доступен (Keepalived должно переключить VIP на другой сервер)
for i in {1..10}; do
    if nc -z $VIP 61616; then
        echo "VIP доступен после остановки HAProxy"
        break
    fi
    sleep 1
done
```

### Ожидаемые результаты:
- **Хороший результат:** После остановки HAProxy, VIP продолжает быть доступен, благодаря работе Keepalived, и клиент продолжает работу с другим балансировщиком нагрузки.
- **Плохой результат:** Если VIP становится недоступен после остановки HAProxy, это указывает на проблемы с настройкой Keepalived или его интеграцией с HAProxy.
