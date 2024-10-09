# Настраиваем бэкапы

## Задание

Настроить стенд **Vagrant** с двумя виртуальными машинами: **backup** и **client**.

Настроить удаленный бекап каталога **/etc** c сервера **client** при помощи **BorgBackup**. Резервные копии должны соответствовать следующим критериям:

- Директория для резервных копий **/var/backup**. Это должна быть отдельная точка монтирования. В данном случае для демонстрации размер не принципиален, достаточно будет и 2GB.
- Репозиторий для резервных копий должен быть зашифрован ключом или паролем.
- Имя бекапа должно содержать информацию о времени снятия бекапа.
- Глубина бекапа должна быть год, хранить можно по последней копии на конец месяца, кроме последних трех.
- Последние три месяца должны содержать копии на каждый день. Т.е. должна быть правильно настроена политика удаления старых бэкапов.
- Резервная копия снимается каждые 5 минут. Такой частый запуск в целях демонстрации.
- Написан скрипт для снятия резервных копий. Скрипт запускается из соответствующей **cron** джобы, либо **systemd timer**.
- Настроено логирование процесса бекапа. Если не в **syslog**, то обязательна ротация логов.

Проверить работу:

- Запустить стенд на 30 минут.
- Убедиться, что резервные копии снимаются.
- Удалить директорию **/etc** и восстановить ее из бекапа.

## Реализация

Задание сделано на **rockylinux/9** версии **v4.0.0**. Для автоматизации процесса написаны следующие роли **Ansible**, переменные для которых хранятся в [group_vars/all.yml](group_vars/all.yml), [host_vars/backup.yml](host_vars/backup.yml) и [host_vars/client.yml](host_vars/client.yml):

- **generate_root_ssh_key** - генерирует **ssh** ключи для пользователя **root** на **client** и сохраняет его в `passwords/client.pub`.
- **epel_release** - добавляет репозиторий **Extra Packages for Enterprise Linux (EPEL)**.
- **borgbackup** - устанавливает **borg** на **client** и **backup**.
- **borg_config** - генерирует файл `/etc/sysconfig/borg` с конфигурацией сервисов **borg** по шаблону [borg](roles/borg_config/templates/borg).
- **borg_logrotate** - настраивает ротацию логов в директории `/var/log/borg` согласно конфигурации [borg](roles/borg_logrotate/templates/borg).
- **borg_repo** - форматирует диск `/dev/sdb` на **backup**, монтирует его, настраивает на нём репозиторий **borg**, сохраняет ключ и пароль от репозитория в `passwords/backup.html` и `passwords/backup_repo.txt`, а также настраивает доступ к репозиторию для **client** через **borg serve** (ограничивая возможность клиента удалять файлы в репозитории, смотри [ssh.yml](roles/borg_repo/tasks/ssh.yml)).
- **borg_maintenance** - настраивает таймеры и сервисы [borg-prune](roles/borg_maintenance/templates/borg-prune.service), [borg-compact](roles/borg_maintenance/templates/borg-compact.service), [borg-check](roles/borg_maintenance/templates/borg-check.service).
- **borg_create** - настраивает создание резервной копии на **client** через сервис [borg-create](roles/borg_create/templates/borg-create.service).

## Запуск

Необходимо скачать **VagrantBox** для **rockylinux/9** версии **v4.0.0** и добавить его в **Vagrant** под именем **rockylinux/9/v4.0.0**. Сделать это можно командами:

```shell
curl -OL https://app.vagrantup.com/rockylinux/boxes/9/versions/4.0.0/providers/virtualbox/amd64/vagrant.box
vagrant box add vagrant.box --name "rockylinux/9/v4.0.0"
rm vagrant.box
```

Для того, чтобы **vagrant 2.3.7** работал с **VirtualBox 7.1.0** необходимо добавить эту версию в **driver_map** в файле **/usr/share/vagrant/gems/gems/vagrant-2.3.7/plugins/providers/virtualbox/driver/meta.rb**:

```ruby
          driver_map   = {
            "4.0" => Version_4_0,
            "4.1" => Version_4_1,
            "4.2" => Version_4_2,
            "4.3" => Version_4_3,
            "5.0" => Version_5_0,
            "5.1" => Version_5_1,
            "5.2" => Version_5_2,
            "6.0" => Version_6_0,
            "6.1" => Version_6_1,
            "7.0" => Version_7_0,
            "7.1" => Version_7_0,
          }
```

После этого нужно сделать **vagrant up**.

Протестировано в **OpenSUSE Tumbleweed**:

- **Vagrant 2.3.7**
- **VirtualBox 7.1.0_SUSE r164728**
- **Ansible 2.17.4**
- **Python 3.11.10**
- **Jinja2 3.1.4**

## Проверка

Проверим, что резервные копии снимаются. Выполним **sudo borg list /var/backup** на сервере **backup** и введём пароль из `passwords/backup_repo.txt`:

```text
[vagrant@backup ~]$ sudo borg list /var/backup
Enter passphrase for key /var/backup:
client-2024-10-09T20:15:02           Wed, 2024-10-09 20:15:02 [d04aabd43f28c1f7bc8be2f8eef6c7f9f54724fd3a0784fc71d94926b25a2cf2]
client-2024-10-09T20:20:02           Wed, 2024-10-09 20:20:03 [b623ff3681985de3310301183b3e032070c38c7d2f4ceb2a819e83f57432606a]
client-2024-10-09T20:25:02           Wed, 2024-10-09 20:25:03 [cad22a14b1eb1139d5964b4a4784fb8de5d22d6d97e337244294b0e3fbdfed36]
client-2024-10-09T20:55:08           Wed, 2024-10-09 20:55:09 [a6a7b938484e014b6d6e6d4498ef4367755e167e49735ed6d0caf94a11b01b4a]
client-2024-10-09T21:00:08           Wed, 2024-10-09 21:00:09 [67ec399480d35de410c147818e994b2e991467f877bd2a080fbade1376acd8e5]
client-2024-10-09T21:05:08           Wed, 2024-10-09 21:05:09 [9bcd7cdc487897487f01cd2242261a0c2ca29aaaa9a06649f32ccc46fbf29928]
client-2024-10-09T21:10:08           Wed, 2024-10-09 21:10:09 [534bc80c15e60175fa39aacd7249681edcb020d0c382ec33774a07149a0201bd]
client-2024-10-09T21:15:08           Wed, 2024-10-09 21:15:09 [d3e7577f21e08e1dcc365ad7a883546c81fe2159091bc49bf44e94d20f8ad7bd]
client-2024-10-09T21:20:08           Wed, 2024-10-09 21:20:09 [01dd76f67e57dc2f0fa43ab10b6cbddeb5931222f24e49a9e9aec27ce5995c58]
```

Как видно архивы успешно создаются.

Удалим директорию **etc** на **client** и восстановим её из резервной копии выше (необходимо пересоздать файл **/etc/passwd**, чтобы работал **ssh**):

```text
[root@client ~]# rm /etc -rf
[root@client ~]# mkdir -p /etc
[root@client ~]# echo root:x:0:0:root:/root:/bin/bash > /etc/passwd
[root@client ~]# cd /
[root@client /]# borg extract 'ssh://borg@192.168.11.160/var/backup::client-2024-10-09T20:15:02'
Enter passphrase for key ssh://borg@192.168.11.160/var/backup:

```

Проверим работу **prune**, **compact**, **check**:

```text
[vagrant@backup ~]$ sudo systemctl start borg-prune.service
[vagrant@backup ~]$ sudo systemctl start borg-compact.service
[vagrant@backup ~]$ sudo systemctl start borg-check.service
[vagrant@backup ~]$ sudo cat /var/log/borg/{prune,compact,check}.log
Synchronizing chunks cache...
Archives: 1, w/ cached Idx: 0, w/ outdated Idx: 0, w/o cached Idx: 1.
Fetching and building archive index for client-2024-10-09T20:15:02 ...
Merging into master chunks index ...
Done.
Keeping archive (rule: daily #1):        client-2024-10-09T20:15:02           Wed, 2024-10-09 20:15:02 [d04aabd43f28c1f7bc8be2f8eef6c7f9f54724fd3a0784fc71d94926b25a2cf2]
------------------------------------------------------------------------------
                       Original size      Compressed size    Deduplicated size
Deleted data:                    0 B                  0 B                  0 B
All archives:               20.10 MB              4.29 MB              4.31 MB

                       Unique chunks         Total chunks
Chunk index:                     411                  421
------------------------------------------------------------------------------
terminating with success status, rc 0
compaction freed about 1.11 kB repository space.
terminating with success status, rc 0
Starting repository check
finished segment check at segment 9
Starting repository index check
Index object count match.
Finished full repository check, no problems found.
Starting archive consistency check...
Starting cryptographic data integrity verification...
Finished cryptographic data integrity verification, verified 412 chunks with 0 integrity errors.
Analyzing archive client-2024-10-09T20:15:02 (1/1)
Archive consistency check complete, no problems found.
terminating with success status, rc 0
```

Логи хранятся в `/var/log/borg` на **backup** и **client**:

```text
[root@backup ~]# ls -la /var/log/borg/
total 16
drwxr-x---.  2 borg borg   59 Oct  9 20:19 .
drwxr-xr-x. 10 root root 4096 Oct  9 20:12 ..
-rw-r--r--.  1 root root  504 Oct  9 20:19 check.log
-rw-r--r--.  1 root root   87 Oct  9 20:19 compact.log
-rw-r--r--.  1 root root  934 Oct  9 20:18 prune.log

[root@client ~]# ls -la /var/log/borg/
total 40
drwxr-x---.  2 root root    24 Oct  9 20:15 .
drwxr-xr-x. 10 root root  4096 Oct  9 20:12 ..
-rw-r--r--.  1 root root 36622 Oct  9 21:25 create.log

```
