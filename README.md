# Резервное копирование
### Настраиваем бэкапы  

**\- Настроить стенд Vagrant с двумя виртуальными машинами: 'backup_server' и 'client'**  
**\- Настроить удаленный бекап каталога /etc c сервера 'client' при помощи borgbackup. Резервные копии должны соответствовать следующим критериям:**  
- директория для резервных копий /var/backup. Это должна быть отдельная точка монтирования. В данном случае для демонстрации размер не принципиален, достаточно будет и 2GB;  
- репозиторий дле резервных копий должен быть зашифрован ключом или паролем - на ваше усмотрение;
- имя бекапа должно содержать информацию о времени снятия бекапа;
- глубина бекапа должна быть год, хранить можно по последней копии на конец месяца, кроме последних трех. Последние три месяца должны содержать копии на каждый день. Т.е. должна быть правильно настроена политика удаления старых бэкапов;
- резервная копия снимается каждые 5 минут. Такой частый запуск в целях демонстрации;
- написан скрипт для снятия резервных копий. Скрипт запускается из соответствующей Cron джобы, либо systemd timer-а - на ваше усмотрение;
- настроено логирование процесса бекапа. Для упрощения можно весь вывод перенаправлять в logger с соответствующим тегом. Если настроите не в syslog, то обязательна ротация логов.
- Запустите стенд на 30 минут.
- Убедитесь что резервные копии снимаются.
- Остановите бекап, удалите (или переместите) директорию /etc и восстановите ее из бекапа.
- Для сдачи домашнего задания ожидаем настроенные стенд, логи процесса бэкапа и описание процесса восстановления  

---
1. Поднимем две виртуальные машины **backup** и **client**. В данном задании использовался VirtualBox и две виртуальные машины с Centos 7.  
2. На машине **backup** был добавлен диск размером 2Gb.  
3. Теперь создадим раздел на добавленном диске с помощью утилиты fdisk:  
`fdisk /dev/sdb` и далее выбираем нужные параметры.    
4. Создадим на новом диске файловую систему Ext4:  
`mkfs.ext4 /dev/sdb1`  
5. Создадим папку для примонтирования:  
`mkdir /var/backup`  
6. Теперь примонтируем диск к корневой системе:  
`mount /dev/sdb1 /var/backup`  
7. Теперь на обеих машинах (backup и client) подключим epel-репозиторий:  
`yum install epel-release`  
8. Устанавливаем на client и backup пакет **borgbackup:**  
`yum install borgbackup`  
9. На сервере backup создаем пользователя и назначаем на него права пользователя **borg**:  
`useradd -m borg`  
`chown borg:borg /var/backup/`  
10. На сервер **backup** создаем каталог ~/.ssh/authorized_keys в каталоге /home/borg:  
`su - borg`  
`mkdir .ssh`  
`touch .ssh/authorized_keys`  
`chmod 700 .ssh`  
`chmod 600 .ssh/authorized_keys`  
11. На **client** генерируем ssh-ключ и добавляем публичный ключ на сервер **backup** в файл authorized_keys:  
`ssh-keygen`  

**Все дальнейшие действия будут проходить на client сервере.**  

12. Создадим папку /temp и сгенерируем в ней файл. С этой папки будем делать резервное копирование:  
`mkdir /temp`
`dd if=/dev/zero of=daygeek2.txt  bs=10M  count=1`  
13. Инициализируем репозиторий на backup сервере (находясь на client):  
`borg init --encryption=repokey borg@192.168.11.160:/var/backup/`  
13. Теперь создадим бэкап с папки /temp  
`borg create --stats --list borg@192.168.11.160:/var/backup/::"temp-{now:%Y-%m-%d_%H:%M:%S}" /temp`  
14. Смотрим, что получилось  
`borg list borg@192.168.11.160:/var/backup/`  
![](https://github.com/remizovk/backup/blob/23ccf5f4be826bf060d3b16dd4c4a446ad2591f2/screenshots/02_%D0%BF%D1%80%D0%BE%D0%B2%D0%B5%D1%80%D0%BA%D0%B0%20%D1%81%D0%BE%D0%B7%D0%B4%D0%B0%D0%BD%D0%B8%D1%8F%20%D0%B1%D1%8D%D0%BA%D0%B0%D0%BF%D0%B0.PNG)  
15. Посмотрим содержимое бэкапа  
`borg list borg@192.168.11.160:/var/backup/::etc-2021-10-15_23:00:15`  
![](https://github.com/remizovk/backup/blob/23ccf5f4be826bf060d3b16dd4c4a446ad2591f2/screenshots/03_%D1%81%D0%BC%D0%BE%D1%82%D1%80%D0%B8%D0%BC%20%D1%81%D0%BE%D0%B4%D0%B5%D1%80%D0%B6%D0%B8%D0%BC%D0%BE%D0%B5%20%D0%B0%D1%80%D1%85%D0%B8%D0%B2%D0%B0.PNG)  
16. Достаем файл из бекапа (бэкап восстановится в текущей папке):  
`borg extract borg@192.168.50.190:/var/backup/::temp-2021-10-15_23:00:15 temp/hostname`  
![](https://github.com/remizovk/backup/blob/23ccf5f4be826bf060d3b16dd4c4a446ad2591f2/screenshots/04_%D0%B2%D0%BE%D1%81%D1%81%D1%82%D0%B0%D0%BD%D0%B0%D0%B2%D0%BB%D0%B8%D0%B2%D0%B0%D0%B5%D0%BC%20%D0%B0%D1%80%D1%85%D0%B8%D0%B2.PNG)  

### Автоматизируем создание бэкапов с помощью systemd  
1. Создаем **сервис** и **таймер** в каталоге /etc/systemd/system/  
`vi /etc/systemd/system/borg-backup.service`  
>[Unit]  
>Description=Borg Backup  
>
>[Service]  
>Type=oneshot  
>
>\# Парольная фраза  
>Environment="BORG_PASSPHRASE=Otus1234"  
>
>\# Репозиторий  
>Environment=REPO=borg@192.168.50.190:/var/backup/  
>
>\# Что бэкапим  
>Environment=BACKUP_TARGET=/temp  
>
>\# Создание бэкапа  
>ExecStart=/bin/borg create \  
>--stats \  
>${REPO}::temp-{now:%%Y-%%m-%%d_%%H:%%M:%%S}
>${BACKUP_TARGET}  
>
>\# Проверка бэкапа  
>ExecStart=/bin/borg check ${REPO}  
>
>\# Очистка старых бэкапов  
>ExecStart=/bin/borg prune \  
>--keep-daily 90 \  
>--keep-monthly 12 \  
>--keep-yearly 1 \  
>${REPO}  

`vi /etc/systemd/system/borg-backup.timer`  
>[Unit]  
>Description=Borg Backup  
>
>[Timer]  
>OnUnitActiveSec=5min  
>
>[Install]  
>WantedBy=timers.target  

3. Включаем и запускаем службу таймера:  
`systemctl enable borg-backup.timer`  
`systemctl start borg-backup.timer`  
4. Проверяем работу таймера:  
`systemctl list-timers --all`  
![](https://github.com/remizovk/backup/blob/23ccf5f4be826bf060d3b16dd4c4a446ad2591f2/screenshots/05_%D0%BF%D1%80%D0%BE%D0%B2%D0%B5%D1%80%D0%BA%D0%B0%20%D1%80%D0%B0%D0%B1%D0%BE%D1%82%D1%8B%20%D1%82%D0%B0%D0%B9%D0%BC%D0%B5%D1%80%D0%B0.PNG)  
5. проверяем список бэкапов:  
`borg list borg@192.168.50.190:/var/backup/`  
