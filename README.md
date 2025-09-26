# Домашнее задание к занятию ««Репликация и масштабирование. Часть 1.»

## Шкутов Иван Владимирович

---

## Задание 1

### На лекции рассматривались режимы репликации master-slave, master-master, опишите их различия.

Ответить в свободной форме.

### РЕШЕНИЕ:

---

### Репликация Master-Slave.

Это способ организации базы данных, где есть один главный сервер (Master) и один или несколько подчинённых (Slave). 

### Как это работает: 

клиенты записывают данные только в Master, а Master автоматически передаёт изменения Slaves. Клиенты могут читать данные как с Master, так и с Slaves (чаще читают со Slaves, чтобы снизить нагрузку на Master). 

### Преимущества репликации Master-Slave:

Отказоустойчивость. Slave-системы действуют как горячие резервные копии, готовые стать новым Master в случае отказа основного сервера. Такая установка сводит к минимуму время простоя и потерю данных.

Масштабируемость. Запросы на чтение могут быть распределены между slave-серверами, что разгружает master-сервер и облегчает работу с повышенной нагрузкой.

Аварийное восстановление. Slave-серверы защищают данные от аппаратных сбоев или катастрофических событий, обеспечивая быстрое восстановление.

### Недостатки репликации Master-Slave:

Задержка синхронизации. Данные на slave-серверах могут немного отставать от master.

Не подходит для интенсивной записи. Так как записи идут только в один узел, он может стать узким местом.

---

### Репликация Master-Master (multi-master репликация).

Это конфигурация базы данных, в которой два или больше главных сервера одновременно принимают и обрабатывают запросы на запись и чтение. 

### Как это работает: 

изменения, внесённые на одном главном сервере, автоматически распространяются на другие, чтобы на всех узлах были одинаковые актуальные данные. 

### Преимущества репликации Master-Master:

Высокая отказоустойчивость. Если один сервер выходит из строя, другой продолжает работать.

Балансировка нагрузки. Можно распределять запросы между серверами, снижая нагрузку на каждый из них.

Быстрота для географически распределённых систем. Пользователи могут работать с ближайшим сервером, уменьшая задержки.

Гибкость. Приложения могут писать на любой узел, что упрощает архитектуру.

### Недостатки репликации Master-Master:

Конфликты данных. Если два сервера изменяют одни и те же данные одновременно, могут возникнуть проблемы с их синхронизацией.

Сложность настройки и поддержки. Требуется механизм разрешения конфликтов и контроль синхронизации.

Задержки репликации. Хотя данные синхронизируются, это не всегда происходит мгновенно, что может привести к временным рассинхронизациям.

---

## Задание 2

### Выполните конфигурацию master-slave репликации, примером можно пользоваться из лекции.

Приложите скриншоты конфигурации, выполнения работы: состояния и режимы работы серверов.

Дополнительные задания (со звёздочкой*)

Эти задания дополнительные, то есть не обязательные к выполнению, и никак не повлияют на получение вами зачёта по этому домашнему заданию. Вы можете их выполнить, если хотите глубже шире разобраться в материале.

### РЕШЕНИЕ:

---

### 1. Создадим каталог проекта

mkdir -p ~/mysql-replication/{master,slave}

cd ~/mysql-replication

### 2. Создадим файл docker-compose.yml в дирректории mysql-replication

### 3. Создадим файлы конфигурации для MASTER и SLAVE (master/my.cnf, slave/my.cnf)


[mysqld]

server-id=1

log_bin=mysql-bin

binlog_format=ROW

gtid_mode=ON

enforce_gtid_consistency=ON

log_slave_updates=ON


[mysqld]

server-id=2

log_bin=mysql-bin

binlog_format=ROW

gtid_mode=ON

enforce_gtid_consistency=ON

relay_log=mysqld-relay-bin

read_only=ON

log_slave_updates=ON


### 4. Запускаем контейнеры

docker compose up -d

### 5. Создаем пользователя репликации на master

docker compose exec master mysql -uroot -prootpass -e "

> CREATE USER 'repl'@'%' IDENTIFIED BY 'replpass';

> GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';

> FLUSH PRIVILEGES;"

### 6. Настраиваем SLAVE и подключаемся к MASTER

docker compose exec slave mysql -uroot -prootpass -e "

> STOP SLAVE;

> CHANGE MASTER TO

  MASTER_HOST='master',
  
  MASTER_USER='repl',
  
  MASTER_PASSWORD='replpass',
  
  MASTER_AUTO_POSITION=1;
  
> START SLAVE;"

### 7. Проверяем статус репликации

docker compose exec slave mysql -uroot -prootpass -e "SHOW SLAVE STATUS\G"

docker compose exec master mysql -uroot -prootpass -e "SHOW MASTER STATUS\G"
 
### 8. Добавляем данные на MASTER и реплицируем на SLAVE

docker compose exec master mysql -uroot -prootpass -e "

> CREATE DATABASE IF NOT EXISTS test_rep;

> USE test_rep;

> CREATE TABLE IF NOT EXISTS t1 (id INT AUTO_INCREMENT PRIMARY KEY, v VARCHAR(50));

> INSERT INTO t1 (v) VALUES ('hello from master');"


### 9. Проверка добавленные данных на SLAVE

docker compose exec slave mysql -uroot -prootpass -e "SELECT * FROM test_rep.t1;"

### 10. Проверка логов на MASTER и SLAVE

docker compose logs -f master

docker compose logs -f slave

 
![1](https://github.com/Ivan-Shkutov/sdb-homeworks-12-06/blob/main/1.png)

![2](https://github.com/Ivan-Shkutov/sdb-homeworks-12-06/blob/main/2.png)

![3](https://github.com/Ivan-Shkutov/sdb-homeworks-12-06/blob/main/3.png)

![4](https://github.com/Ivan-Shkutov/sdb-homeworks-12-06/blob/main/4.png)

![5](https://github.com/Ivan-Shkutov/sdb-homeworks-12-06/blob/main/5.png)


---

## Задание 3*

Выполните конфигурацию master-master репликации. Произведите проверку.

Приложите скриншоты конфигурации, выполнения работы: состояния и режимы работы серверов.
