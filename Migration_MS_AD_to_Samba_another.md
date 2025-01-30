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


