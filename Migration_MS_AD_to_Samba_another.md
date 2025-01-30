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

