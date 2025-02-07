## Миграция подразделений, групп и пользователей из домена MS AD в Samba с использованием промежуточного контроллера домена на Windows

### Исходные данные

**1)** DC1 -  Windows Server 2016 контроллер домена wind.old на MS AD

**2)** DC2 - Windows Server 2019  конртоллер домена samba.new на MS AD

**3)** DC3  - ALT Server контроллер домена samba.new на MS AD

### Цель

Произвести миграцию подразделений, групп и пользователей из домена wind.old, размещённого на контроллере домена Windows Server 2016 с MS AD, в домен samba.new, размещённый на контроллере ALT Server с Samba AD, используя промежуточный контроллер на Windows Server 2019.

Мы имеем старый домен wind.old, размещённый на контроллере домена Windows Server 2016 с MS AD, и необходимо заменить его новым доменом samba.new на контроллере ALT Server с Samba AD. Миграцию будем выполнять в два этапа:

  **1)** На Windows Server 2019 развернём домен samba.new и с помощью ADMT (Active Directory Migration Tool) проведём миграцию подразделений, групп и пользователей из домена wind.old в домен samba.new.

 **2)** На втором этапе сделаем ALT Server вторичным контроллером домена samba.new и выведем Windows Server 2019 из состава домена.

 ---

### Миграция с использованием ADMT из домена wind.old в samba.new
#### Подготовка к миграции:

**1.** Для выполнения миграции необходимо настроить двустороннее доверительное отношение между доменами *wind.old* и *samba.new*. Подробное описание процесса можно найти в статье по следующей ссылке: [Настройка доверительных отношений между доменами](https://www.altlinux.org/ActiveDirectory/Trusts).

**2.** На контроллере домена Windows Server 2016 необходимо установить SQL Server 2016 Express. Скачать можно по следующей ссылке: [SQL Server 2016 Express](https://www.microsoft.com/ru-ru/download/details.aspx?id=56840).

**3.** После установки SQL Server, нужно установить ADMT 3.2 (Active Directory Migration Tool). Скачайте его по следующему адресу: [ADMT 3.2](https://www.microsoft.com/en-us/download/details.aspx?id=56570).

**4.** Установите и запустите *Password Export Server* в сервисах Windows. Инструкции и скачивание доступны по ссылке: [Password Export Server](https://www.microsoft.com/en-us/download/details.aspx?id=1838).

**5.** Администратор домена *wind.old* должен быть добавлен в группу «Администраторы» в домене *samba.new* в разделе «Builtin».

---

#### Миграция подразделений

В ADMT не предусмотрена возможность миграции подразделений (OU). Однако подразделения можно экспортировать. Для этого необходимо выполнить следующие шаги:

**1.** На контроллере домена Windows Server 2016 с помощью утилиты *ldifde* создадим файл для экспорта, содержащий информацию о подразделениях в домене *wind.old*.

   Для этого выполните следующую команду:

   ```bash
   C:\Users\Администратор> ldifde -f C:\OUExport.ldif -d "DC=wind,DC=old" -p subtree -r "(objectClass=organizationalUnit)" -l "dn,objectClass"
   Подключение к "DC1.wind.old"
   Вход от имени текущего пользователя с помощью SSPI
   Экспорт каталога в файл C:\OUExport.ldif
   Поиск элементов...
   Записываются элементы...
   3 элементов экспортировано
   Команда успешно выполнена
   ```

**2.** Полученный текстовый файл *OUExport.ldif* необходимо отправить на контроллер домена *ALT Server*, воспользовавшись общим диском или любым другим доступным способом передачи файлов.

**3.** Для работы с файлом *OUExport.ldif* в новом домене, необходимо внести изменения в файл.
 В файле *OUExport.ldif* необходимо удалить информацию о подразделении *Domain Controllers* во избежание конфликтов при экспорте, а также заменить имя домена. Для замены имени домена выполните следующую команду в PowerShell:

   ```powershell
   (Get-Content -Path "C:\OUExport.ldif") -replace "wind", "samba" -replace "old", "new" | Set-Content -Path "C:\OUExport.ldif"
   ```

   _**Пример содержания файла до изменений:**_

   ```text
   dn: OU=Domain Controllers,DC=wind,DC=old
   changetype: add
   objectClass: top
   objectClass: organizationalUnit

   dn: OU=OU1,DC=wind,DC=old
   changetype: add
   objectClass: top
   objectClass: organizationalUnit

   dn: OU=OU2,DC=wind,DC=old
   changetype: add
   objectClass: top
   objectClass: organizationalUnit
   ```

   _**Пример содержания файла после изменений:**_

   ```text
   dn: OU=OU1,DC=samba,DC=new
   changetype: add
   objectClass: top
   objectClass: organizationalUnit

   dn: OU=OU2,DC=samba,DC=new
   changetype: add
   objectClass: top
   objectClass: organizationalUnit
   ```


**4.** После выполнения всех необходимых изменений в файле можно произвести импорт данных в новый AD. Для этого откройте командную строку от имени администратора и выполните следующую команду:

   ```bash
   C:\Users\Администратор> ldifde -i -f C:\OUExport.ldif
   Connecting to "DC2.samba.new"
   Logging in as current user using SSPI
   Importing directory from file "C:\OUExport.ldif"
   Loading entries...
   2 entries modified successfully.

   The command has completed successfully
   ```

   В выводе видно, что 2 записи успешно обновлены, и команда завершена без ошибок. Также в RSAT (Remote Server Administration Tools) можно увидеть подразделения из старого домена.

   Миграция подразделений *OU1* и *OU2* успешно выполнена.

---

#### Миграция групп

Для выполнения миграции групп в данном примере используется Active Directory Migration Tool (ADMT). Один из важных моментов при работе с ADMT заключается в том, что группу можно мигрировать вместе с её пользователями, без необходимости выполнения отдельных действий для переноса пользователей. Однако если миграция группы выполняется одновременно с её участниками, все пользователи будут перемещены в то же подразделение (OU), что и группа.

Например, в исходном домене:

  +  OU1 содержит трёх пользователей: test1, test2, test3.
  +  OU2 содержит группу Group1.

Если при миграции группы Group1 в новый домен её участники будут переноситься одновременно, то все пользователи (test1, test2, test3) окажутся в том же OU, что и группа, а именно в OU2 на целевом сервере Samba. Это может нарушить исходную структуру домена. Поэтому целесообразно мигрировать группы и пользователей раздельно, если они не принадлежат одному и тому же подразделению.

**Пошаговая инструкция по миграции групп с использованием ADMT**

**1. Запуск ADMT и выбор параметров миграции**

 - Откройте ADMT на сервере с Windows Server 2016.
 - На панели программы выберите «Action» → Group Account Migration Wizard.
 - В появившемся окне выберите:

 Исходный домен:  wind.old (контроллер: \\DC1.wind.old).

 Целевой домен: samba.new (контроллер: \\DC2.samba.new).

 - Нажмите Далее.

**2. Выбор группы для миграции**

  - В разделе Select Groups from Domain выберите группу, которую нужно перенести, например, Group1.

  - Нажмите Далее.

**3. Указание целевого OU**

   - Укажите целевое подразделение (OU) для миграции группы. В данном примере это OU2.
   -  Нажмите Далее.

**4. Настройка параметров группы**

  -  В разделе Group Options:
        Если необходимо перенести группу вместе с её участниками, установите галочку Copy Group Members.
        Если перенос пользователей вместе с группой не требуется, оставьте параметры по умолчанию.
 - При миграции SID будет предложено включить аудит и создание специальной группы с чем необходимо согласиться.
-  Нажмите Далее.


**5. Исключение атрибутов**

  -  В разделе Object Property Exclusion можно исключить определённые атрибуты группы из процесса миграции.
   -  В данном примере оставьте настройки по умолчанию и нажмите Далее.

**6. Управление конфликтами**

 - В окне Conflict Management задаются параметры для разрешения конфликтов при миграции. Например, при совпадении имён объектов.
  -   Оставьте настройки по умолчанию и нажмите Далее.

**7. Завершение миграции**

   - После проверки настроек нажмите Готово.

   - В появившемся окне будет отображён процесс миграции, а также статус её завершения.  Убедитесь, что миграция завершилась без ошибок.

**8. Проверка на целевом домене**

- Выполните проверку наличия в RSAT на целевом домене samba.new в подразделении OU2  группы Group1.

---

#### Миграция пользователей

Миграция пользователей осуществляется с помощью Active Directory Migration Tool (ADMT). Этот процесс включает перенос учетных записей пользователей из исходного домена в целевой, с возможностью настройки паролей, сохранения членства в группах и других параметров.

**Пошаговая инструкция по миграции пользователей**

**1. Запуск ADMT и настройка параметров миграции**

  - Запустите ADMT на сервере с Windows Server 2016.

  - На панели программы выберите «Action» → User Account Migration Wizard.
  - В появившемся окне укажите:

     Исходный домен: wind.old (контроллер: \\DC1.wind.old ).

       Целевой домен: samba.new (контроллер: \\DC2.samba.new).

   - Нажмите Далее.

**2. Выбор пользователей для миграции**

  - В разделе Select Users from Domain выберите учетные записи, которые необходимо перенести. Например, test1, test2, test3.
   - Нажмите Далее.

**3. Указание целевого OU**

  - Укажите целевое подразделение (OU) для миграции пользователей. В данном примере это OU1.
 -  Нажмите Далее.

**4. Настройка параметров паролей**

- В разделе Password Options выберите Migrate Passwords.


 - Нажмите Далее.



**5. Настройка перехода учетных записей**

  -  В окне Account Transition Options выберите параметр Target same as source, если нет необходимости применять другие настройки.
   - Нажмите Далее.

**6. Настройка членства в группах**

   - В разделе User Options выберите Fix Users' Membership.
    Эта настройка автоматически добавляет пользователей в группы целевого домена, соответствующие их участию в исходных группах.
  -  Нажмите Далее.

**7. Исключение атрибутов**

  - В разделе Object Property Exclusion можно исключить определённые атрибуты учетных записей из процесса миграции.
  -  В данном примере оставьте настройки по умолчанию.
    Нажмите Далее.

**8. Управление конфликтами**

  -  В разделе Conflict Management задайте параметры для разрешения конфликтов, например, при совпадении имён учетных записей.
  - Оставьте настройки по умолчанию.
  - Нажмите Далее.

**9. Завершение миграции**

  - Проверьте параметры, нажмите Готово.
  - В появившемся окне отобразится процесс миграции и статус её завершения.
    Убедитесь, что миграция завершилась без ошибок.


**10. Проверка наличия пользователей в целевом домене**

- В RSAT в домене samba.new в OU1 проверьте наличие пользователей и принадлежность их к группам.

- По умолчанию у пользователей после миграции будет выставлен атрибут «требовать смены пароля при следующем входе в систему», для того что бы продолжить использовать такой же пароль для учётной записи пользователя как в старом домене, необходимо сменить на атрибут «срок действия пароля не ограничен».

---

### Создание второго контроллера домена на Samba AD и вывод первого контроллера из домена

В данном разделе рассматривается процесс добавления второго контроллера домена (DC3) в существующий домен Samba AD (samba.new) и замены первого контроллера домена (DC2) на новый. Этот процесс включает установку необходимых пакетов, настройку Kerberos, ввод нового сервера в домен, проверку репликации, передачу ролей FSMO и удаление старого контроллера.


**1. Установка и настройка второго контроллера домена (DC3)**

**1.1 Установка пакетов**
Для корректной работы второго контроллера домена на сервере DC3 необходимо установить пакет **task-samba-dc**, который включает поддержку Heimdal Kerberos.

```bash
apt-get install task-samba-dc 
```

Samba на базе Heimdal Kerberos использует KDC, несовместимый с MIT Kerberos. Поэтому на контроллере домена на базе Heimdal Kerberos из пакета samba-dc, для совместимости с клиентской библиотекой libkrb5, в файле *krb5.conf* (в блоке `libdefaults`) необходимо отключить использование ядерного кеша ключей — `KEYRING:persistent:%{uid}`:

```bash
control krb5-conf-ccache default
```

**1.2 Отключение конфликтующих сервисов**
Перед настройкой нового контроллера отключаем ненужные службы:

```bash
for service in bind krb5kdc nmb smb slapd; do chkconfig $service off; service $service stop; done
```

**1.3 Очистка предыдущих данных**
Удаляем конфигурационные файлы и создаём каталог :

```bash
rm -f /etc/samba/smb.conf
rm -rf /var/lib/samba
rm -rf /var/cache/samba
mkdir -p /var/lib/samba/sysvol
```

**1.4 Настройка Kerberos**
Редактируем файл */etc/krb5.conf*, чтобы задать параметры домена SAMBA.NEW:

```ini
[libdefaults]
dns_lookup_kdc = true
dns_lookup_realm = false
default_realm = SAMBA.NEW
```

**1.5 Получение билета администратора**

Аутентифицируемся в домене с помощью Kerberos:

```bash
kinit Администратор
```

Проверяем билет:

```bash
klist
```

Вывод должен содержать информацию о выданном билете Kerberos.

**1.6 Ввод DC3 в домен**

Теперь вводим сервер DC3 в домен Samba:

```bash
samba-tool domain join --option="ad dc functional level = 2016" samba.new DC -U SAMBA\\Администратор --realm=samba.new
```

После успешного ввода в домен в файле *resolv.conf* необходимо сменить адрес PDC на адрес вторичного DC.

**1.7 Настройка Kerberos**

В момент создания домена Samba автоматически конфигурирует шаблон файла */var/lib/samba/private/krb5.conf* для вашего домена. Его можно просто скопировать с заменой:

```bash
cp /var/lib/samba/private/krb5.conf /etc/krb5.conf
```

 **1.8 Запуск и проверка сервиса Samba**
Включаем автоматический запуск Samba и проверяем её работу:

```bash
systemctl enable --now samba
```

Проверяем информацию о домене:

```bash
samba-tool domain info 127.0.0.1
Forest           : samba.new
Domain           : samba.new
Netbios domain   : SAMBA
DC name          : dc3.samba.new
DC netbios name  : DC3
Server site      : Default-First-Site-Name
Client site      : Default-First-Site-Name
```

Просмотр предоставляемых служб:

```bash
[root@DC3 ~]# smbclient -L localhost -U Администратор
Password for [SAMBA\Администратор]:

	Sharename       Type      Comment
	---------       ----      -------
	sysvol          Disk      
	netlogon        Disk      
	IPC$            IPC       IPC Service (Samba 4.20.6-alt3)
SMB1 disabled -- no workgroup available
```


Проверяем имена хостов:
- Адрес `_kerberos._udp.*адрес домена с точкой`:
  ```bash
  [root@DC3 ~]# host -t SRV _kerberos._udp.samba.new
  _kerberos._udp.samba.new has SRV record 0 100 88 dc2.samba.new.
  _kerberos._udp.samba.new has SRV record 0 100 88 dc3.samba.new.
  ```

- Адрес `_ldap._tcp.*адрес домена с точкой`:
  ```bash
  [root@DC3 ~]# host -t SRV _ldap._tcp.samba.new.
  _ldap._tcp.samba.new has SRV record 0 100 389 dc2.samba.new.
  _ldap._tcp.samba.new has SRV record 0 100 389 dc3.samba.new.
  ```

- Адрес хоста `*адрес домена с точкой`:
  ```bash
  [root@DC3 ~]# host -t A samba.new.
  samba.new has address 10.64.238.200
  samba.new has address 10.64.238.173
  ```

**1.8 Проверка репликации**

Проверяем статус репликации между контроллерами:

```bash
samba-tool drs showrepl
```

Если репликация успешна, переходим к следующему шагу.

**1.9 Проверка наличия целевых объектов для миграции**
- Проверяем наличие пользователей:
  ```bash
  [root@DC3 ~]# samba-tool user list
  Гость
  test2
  Администратор
  test3
  test1
  krbtgt
  ```

- Проверяем группы:
  ```bash
  [root@DC3 ~]# samba-tool group list
  Группа с запрещением репликации паролей RODC
  Пользователи журналов производительности
  Гости
  Пользователи
  Пользователи системного монитора
  Администраторы основного уровня предприятия
  Администраторы предприятия
  Group1
  …..
  ```

- Проверяем членов группы Group1:
  ```bash
  [root@DC3 ~]# samba-tool group listmembers Group1
  test3
  test1
  test2
  ```

---

**1.10 Передача ролей FSMO и удаление DC2**

Перед тем как удалить DC2, необходимо убедиться, что он не является владельцем ролей FSMO. Проверяем текущее состояние:

```bash
[root@DC3 ~]# samba-tool fsmo show
SchemaMasterRole owner: CN=NTDS Settings,CN=DC2,CN=Servers,CN=Default-First-Site-Name,CN=Sites,CN=Configuration,DC=samba,DC=new
InfrastructureMasterRole owner: CN=NTDS Settings,CN=DC2,CN=Servers,CN=Default-First-Site-Name,CN=Sites,CN=Configuration,DC=samba,DC=new
RidAllocationMasterRole owner: CN=NTDS Settings,CN=DC2,CN=Servers,CN=Default-First-Site-Name,CN=Sites,CN=Configuration,DC=samba,DC=new
PdcEmulationMasterRole owner: CN=NTDS Settings,CN=DC2,CN=Servers,CN=Default-First-Site-Name,CN=Sites,CN=Configuration,DC=samba,DC=new
DomainNamingMasterRole owner: CN=NTDS Settings,CN=DC2,CN=Servers,CN=Default-First-Site-Name,CN=Sites,CN=Configuration,DC=samba,DC=new
DomainDnsZonesMasterRole owner: CN=NTDS Settings,CN=DC2,CN=Servers,CN=Default-First-Site-Name,CN=Sites,CN=Configuration,DC=samba,DC=new
ForestDnsZonesMasterRole owner: CN=NTDS Settings,CN=DC2,CN=Servers,CN=Default-First-Site-Name,CN=Sites,CN=Configuration,DC=samba,DC=new
```

Все роли принадлежат DC2.

Передаем все роли FSMO на DC3:

```bash
[root@DC3 ~]# samba-tool fsmo seize --force --role=all
```

После выполнения команды проверяем ещё раз:

```bash
samba-tool fsmo show
SchemaMasterRole owner: CN=NTDS Settings,CN=DC3,CN=Servers,CN=Default-First-Site-Name,CN=Sites,CN=Configuration,DC=samba,DC=new
InfrastructureMasterRole owner: CN=NTDS Settings,CN=DC3,CN=Servers,CN=Default-First-Site-Name,CN=Sites,CN=Configuration,DC=samba,DC=new
RidAllocationMasterRole owner: CN=NTDS Settings,CN=DC3,CN=Servers,CN=Default-First-Site-Name,CN=Sites,CN=Configuration,DC=samba,DC=new
PdcEmulationMasterRole owner: CN=NTDS Settings,CN=DC3,CN=Servers,CN=Default-First-Site-Name,CN=Sites,CN=Configuration,DC=samba,DC=new
DomainNamingMasterRole owner: CN=NTDS Settings,CN=DC3,CN=Servers,CN=Default-First-Site-Name,CN=Sites,CN=Configuration,DC=samba,DC=new
DomainDnsZonesMasterRole owner: CN=NTDS Settings,CN=DC3,CN=Servers,CN=Default-First-Site-Name,CN=Sites,CN=Configuration,DC=samba,DC=new
ForestDnsZonesMasterRole owner: CN=NTDS Settings,CN=DC3,CN=Servers,CN=Default-First-Site-Name,CN=Sites,CN=Configuration,DC=samba,DC=new
```
Теперь все роли FSMO принадлежат контроллеру домена ALT Server c именем DC3, что необходимо для последующих действий по выводу из домена контроллере DC2.

**2. Вывод контроллера на базе Windows Server 2019 из домена**

**2.1 Удаление DC2 из домена**

Выводим контроллер домена из эксплуатации, удаляя всю информацию о нём. Для этого на контроллере домена ALT Server выполните команду:

```bash
[root@DC3 ~]# samba-tool domain demote --remove-other-dead-server=dc2 -U Администратор
```

Если процесс прошёл успешно, DC2 больше не будет числиться в домене.

Проверяем отсутствие записей о DC2:
```bash
[root@DC3 ~]# host -t SRV _ldap._tcp.samba.new.
_ldap._tcp.samba.new has SRV record 0 100 389 dc3.samba.new.

[root@DC3 ~]# host -t SRV _kerberos._udp.samba.new.
_kerberos._udp.samba.new has SRV record 0 100 88 dc3.samba.new.

[root@DC3 ~]# host -t A samba.new.
samba.new has address 10.64.238.200
```

**2.2 Очистка метаданных старого контроллера**

После удаления DC2 необходимо очистить его записи в AD. Проверяем список компьютеров:

```bash
[root@DC3 ~]# samba-tool computer list
DC3$
DC2$
```

Удаляем DC2 из базы LDAP:
```bash
[root@DC3 ~]# ldbdel -H /var/lib/samba/private/sam.ldb "CN=DC2,OU=Domain Controllers,DC=samba,DC=new"
Deleted 1 record

[root@DC3 ~]# samba-tool computer list
DC3$
```

---
### Заключение

Процесс миграции включает несколько ключевых этапов, таких как подготовка доверительных отношений, настройка инструментов для миграции и перенос данных и ролей на новую платформу. Применение промежуточного контроллера домена на Windows Server 2019 позволяет обеспечить плавную миграцию от Windows AD к Samba AD, минимизируя риски потери данных и нарушений в структуре домена. Поскольку использование инструмента ADMT не поддерживает  миграцию паролей учётных записей в Samba AD, для решения этой задачи был использован дополнительный сервер на базе Windows, что позволило корректно перенести пароли пользователей и сохранить их работоспособность в новом домене.

---



