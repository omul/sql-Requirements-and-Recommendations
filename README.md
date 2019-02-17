# Требования и Рекомендации по установке Microsoft SQL Server в виртуальной среде (vmWare)

## Виртуальная машина

### Процессор

#### Требования

-   Количество vCPU не не должно быть больше физических ядер (physical core), доступных на хосте

#### Рекомендации

-   В случае использование версии Microsoft SQL Server Standard, не рекомендуетмя предоставлять виртуальной машине процессорных ядер больше, чем входит в pNUMA. См. [ограничения версий SQL](https://docs.microsoft.com/en-us/sql/sql-server/editions-and-components-of-sql-server-2016?view=sql-server-2017#RDBMSSP)

### Память

#### Требования

-   Для определения размера ОЗУ необходимо воспользоваться формулой:


    Объём ОЗУ = SQL Max Server Memory + ThreadStack + OS Mem + VM Overhead + Память для приложений (антивирус, агенты мониторинга и т.п.)

    ThreadStack = SQL Max Worker Threads * ThreadStackSize
    
    ThreadStackSize = 2MB для x64 систем
    
    SQL Max Worker Threads (При [max worker threads = 0](https://docs.microsoft.com/ru-ru/sql/database-engine/configure-windows/configure-the-max-worker-threads-server-configuration-option?view=sql-server-2017#Recommendations)):<br>
        = 512 (если число логических ЦП <= 4)<br>
        = 512 + ((число логических ЦП – 4) * 16). 
    
    OS Mem: 1GB на каждые 4 логических ЦП
    
    VM Overhead =     
    
    | Memory | 1 vCPU | 2 vCPU | 3 vCPU | 4 vCPU |
    |--------|--------|--------|--------|--------|
    | 256    | 20,29  | 24,28  | 32,23  | 48,16  |
    | 1024   | 25,90  | 29,91  | 37,86  | 53,82  |
    | 4096   | 48,64  | 52,72  | 60,67  | 76,78  |
    | 16384  | 139,62 | 143,98 | 151,93 | 168,60 |

#### Рекомендации

-   Без необходимости, не превышать объём памяти pNUMA физического сервера, на котором размещена виртуальная машина

-   В случае использования SQL Server в качестве критичной системы для бизнеса можно использовать резервирование всей гостевой памяти виртуальной машины для предотварщения случаев балунинга или свопинга. Это гаранитрует, что виртуальная память вм будет размещена в памяти хоста. Но это чревато проблемами при миграции и в случае старта виртуальной машины на другом хосте после сбоя.

### Дисковая подсистема

#### Требования

-   Необходимо использовать VMware Paravirtualized SCSI (PVSCSI) Controller в качестве контроллера дисков

-   Системный диск должен быть не менее 50 Гигбайт.

-   Для сервера баз данных необходимо создать четыре диска: системный, данные, журнал транзакций, tempdb

-   Тома, на которых расположены VMDK виртуальных дисков должны быть VMFS5 или выше.

#### Рекомендации

-   Каждый диск на отдельном контроллере.

-   Размер системного диска 100 Гигбайт.

-   Дополнительно можно использовать отдельные диски для восстановления резервных копий, filestream и т.п.

-   VMFS version 6.

-   "Толстые диски" с предварительным обнулением - Eagerzeroed.

### Сеть

#### Требования

-   Использовать VMXNET3 paravirtualized NIC

#### Рекмендации

-   Активировать Receive Side Scaling (RSS) в следующих местах:
    1. Включить в ядре Windows:`netsh interface tcp set global rss=enabled`. Проверить `Netsh int tcp show global`.
    1. Включить в настройках драйвера VMXNET3: В Windows в свойствах Сетевого адаптерах VMXNET3, на вкладке Advanced установить значение `Receive-side scaling` в `Enabled`.

## Сервер

### Операционная система

#### Требования

-   Версия ОС не ниже Microsoft Windows 2012R2.

-   Установка последних патчей.

-   Установка VmWare Tools

-   Выставить план управления питанием в High performance.

#### Рекмендации

-   Версия ОС Microsoft Windows 2016 или выше.

### Дисковая подсистема

#### Требования

-   На одном диске необходимо размещать не больше одного раздела.

-   Диски, используемые для файлов БД, журналов транзакций, tempdb, filestream должны быть отформатированы с размером кластера в 64 Кб.

-   Отключение генерации имён 8.3 (`fsutil 8dot3name query буква диска:` - проверка, `fsutil 8dot3name set буква диска: 1` - отключение). [Статья](https://docs.microsoft.com/en-us/windows-server/administration/windows-commands/fsutil-8dot3name)

-   Отключение отслеживания последнего времени обращения (`fsutil behavior query DisableLastAccess` - проверка, `fsutil behavior set DisableLastAccess 1` - отключение) [статья](https://docs.microsoft.com/en-us/windows-server/administration/windows-commands/fsutil-behavior)

-   Отключение индексирования диска. (Остановка службы Windows Search и перевод её в состояние запуска - Disabled, и снять галку в свойствах диска БД "allow files on this drive to have contents indexed")

-   Отключение сжатия диска (Снять галку в свойствах диска "compress this drive to save disk space")

-   Проверка отключения автоматической дефрагментации (На диска ПКМ -\> Tools -\> Defragment Now -\> Turn on schedule - и снять галку)

#### Рекомендации

-   Буквы дисков сервера БД:

    -   Database - D:

    -   Transactoin logs - L:

    -   TempDB - T:

-   Проверка выравнивания созданных дисковых разделов (diskpart.exe -\> select disk *N* -\> list partition). Если смещение не равно 1024 Кб, то удалить разделы и создать заново (create partition primary align=1024). 

> [Актуально для Windows Server 2003 и ниже](https://support.microsoft.com/ru-ru/help/929491/disk-performance-may-be-slower-than-expected-when-you-use-multiple-dis)

### Windows page file

#### Рекомендации

-  Установить размер в 2 Gb

### Антивирус

#### Требования

-   Добавить в исключения согласно [статье](http://support.microsoft.com/kb/309422)

### Учётные записи

#### Требования

-   Отдельная учётная запись для каждого сервиса

-   Использование Kerberos для аутентификации на сервере (Для регистрации аттрибутов SPN `"MSSQL\"` у учётной записи, под которой SQL Server представлен в домене)

>   !!! Без настройки Max Server Memory, выполнение следующего пункта может привести к краху системы !!!

-   Блокировка страниц в памяти (secpol.msc -\> Local Policies -\> User Rights Assigment -\> Lock pages in memory - добавить УЗ  SQL Server)

-   Database instant file initialization (secpol.msc -\> Local Policies -\> User Rights Assigment -\> Perform volume maintenance tasks - добавить УЗ SQL Server)

#### Рекомендации

-   Для повышения безопасности и удобства [рекомендовано](https://docs.microsoft.com/en-us/sql/database-engine/configure-windows/configure-windows-service-accounts-and-permissions?view=sql-server-2016) использование MSA/gMSA. Для этого направить заявку на группу администраторов домена:

> !!! В тексте надо заменть servicename на реальное имя, а пути и OU на реальные !!!

======== 8\< =======

>   Создать OU

`corp/Security Principals/Applications/MSSQL/ServiceAccounts/servicename`

>   В этом OU создать группу *mssql_servername_sysadmins*

>   С правами учётной записи Domain Admins, необходимо выполнить следующие команды

```powershell
Import-module ActiveDirectory
 
New-ADServiceAccount mssql_servername_svc –RestrictToSingleComputer -Description -Path "OU=servicename,OU=ServiceAccounts,OU=MSSQL,OU=Applications,OU=Security Principals,DC=corp"

Add-ADComputerServiceAccount -Identity servername -ServiceAccount mssql_servername_svc
```

>   Затем подключиться на servername по RDP и с правами администратора сервера выполнить команду:

`Install-ADServiceAccount -Identity mssql_servername_svc`

>   Для проверки выполнить команду

`Test-ADServiceAccount mssql_servername_svc`

>   которая в случае успеха должна вернуть **true**

======== 8\< =======

-   Потом в свойствах настройки УЗ для старта SQL Server необходимо прописать *mssql_servername_svc\$* в имени УЗ, от которой будет запускаться SQL Server. Поля «пароль» и «подтверждение пароля» оставить пустым.

## Microsoft SQL Server

### Установка и конфигурация

#### Требования

-   Установка только необходимых компонентов SQL Server

-   Необходимо выбрать правильный collation

-   Необходимо правильно задать пути по умолчанию для файлов базы данных, журналов транзакций и tempdb 

-   Установка последних SP и CU

-   Проверить IFI (Создать БД размером в 5Гб. БД должна создасться мгновенно)

-   Установка Max Server Memory. После достаточно длительных (30-60 дней) наблюдений за свободной памятью, настройку можно изменить. [SQL Server может испольлзовать память выше установленного предела](https://support.microsoft.com/ru-kz/help/2663912/memory-configuration-and-sizing-considerations-in-sql-server-2012-and)

-   Для БД с несколькими файлами в одной файловой группе настроить одновременное увеличение файлов файловой группы [ALTER DATABASE \<dbname\> MODIFY FILEGROUP \<filegroup\> AUTOGROW_ALL_FILES](https://blogs.msdn.microsoft.com/sql_server_team/sql-server-2016-changes-in-default-behavior-for-autogrow-and-allocations-for-tempdb-and-user-databases/).

-   Установить Max Degree of Parallelism не более, чем количество ядер процессора (в случае, если приложение не требует иного. Например, Microsoft Sharepoint требует 1). Если MaxDop > 1, то значение должно быть чётным. 

-   Установить Cost Threshold for Parallelism:

    -   50 для OLTP систем

    -   25 для DWH систем

-   Необходимо задать для БД model фиксированный (не процентный!) прирост:

    -   256 Mb для файла БД

    -   128 Mb для файла журнала транзакций

-   Активировать [Dedicated Admin Connection](https://www.brentozar.com/archive/2011/08/dedicated-admin-connection-why-want-when-need-how-tell-whos-using/)

-   Проверить отключение AutoShrink и AutoClose

-   Установить [настройку](https://docs.microsoft.com/ru-ru/sql/database-engine/configure-windows/optimize-for-ad-hoc-workloads-server-configuration-option?view=sql-server-2017) optimize for ad hoc workloads в 1

-   Установить [настройку](https://docs.microsoft.com/ru-ru/sql/t-sql/statements/set-arithabort-transact-sql?view=sql-server-2017) arithabort в 1

#### Рекомендации

-   Версия Microsofr SQL Server не ниже 2016

-   В случае наличия на сервере дополнительного ПО выставить настройку Min Server Memory

-   В случае использования SQL Server в качестве критичного для бизнеса сервиса, выставить Min Server Memory = Max Server Memory

-   Использовать Windows аутентификацию в SQL Server

-   Добавить группу `CORP\MSSQL_Enterprise_admins` в группу `sysadmins` SQL server 

-   В случае использования версии Microsofr SQL Server ниже 2016 проверить выставления флагов 1117, 1118. А также [других](https://support.microsoft.com/en-us/help/2964518/recommended-updates-and-configuration-options-for-sql-server-2012-and).

### TempDB

#### Требования

-   Количество файлов TempDB должно равняться числу ядер процессора, но не больше 8. [Если](https://support.microsoft.com/ru-kz/help/2154845/recommendations-to-reduce-allocation-contention-in-sql-server-tempdb-d) при работе наблюдается большое количества ожиданий блокировки страниц SGAM или PFS TempDB (PAGELATCH_UP), необходимо увеличивать количество файлов TempDB, порциями по 4 файла.

#### Рекомендации

-   Проверить наличие опций TempDB по одновременному увеличению файлов в файловой группе и по хранению данных в однородных экстентах, согласно этому [документу](https://blogs.msdn.microsoft.com/sql_server_team/sql-server-2016-changes-in-default-behavior-for-autogrow-and-allocations-for-tempdb-and-user-databases/)

-   Размер файлов можно сразу выбрать таким, что бы максимально заполнить выделенное дисковое пространство.

### Обслуживание

#### Требования

-   Должны быть настроены задачи по обслуживанию (План РК должен быть согласован с группой по резервному копированию) :
   
    -   Полное резеврное копирование пользовательских и системных баз данных

    -   Резервное копирование журналов транзакций (в случае полной модели восстановления)

    -   Проверка целостности баз данных (CheckDB)

    -   Обслуживание индексов и статистски

-   Не использовать Планы обслуживания

-   Не использовать сжатие (Shrink) в автоматическом режиме

#### Рекомендации

-   Рекомендуется использовать скрипты [Ola Hallengren](http://ola.hallengren.com/), либо [Trivadis Toolbox](http://www.trivadis.com/)

### Мониторинг

#### Требования

-   Для постановки сервера БД на мониторинг, необходимо подать заявку на Группу мониторинга

#### Рекомендации

-   Для отправки почтовых сообщений о результатах выполнения задач и критичных событий необходимо (Например, по этой [статье](https://habr.com/ru/post/132902/)):

    -   Настроить Database Mail 

    -   Создать операторов для SQL Server Agent

    -   Настроить Database Mail в свойствах SQL Server Agent

    -   Настроить алерты для событий высокой важности: [Blitz Result: No SQL Server Agent Alerts Configured](https://www.brentozar.com/blitz/configure-sql-server-alerts/)

    -   Перезагрузить сервис SQL Server Agent и протестировать срабатывание алерта: RAISERROR('Alert Test', 18,1) WITH LOG;

-   Установить [sp_WhoIsActive](https://www.brentozar.com/archive/2010/09/sql-server-dba-scripts-how-to-find-slow-sql-server-queries/)

## Источники

[Brent Ozar - SQL Server Setup Checklist](https://www.brentozar.com/archive/2008/03/sql-server-2005-setup-checklist-part-1-before-the-install/)

[Planning a SQL Server Installation](https://docs.microsoft.com/en-us/sql/sql-server/install/planning-a-sql-server-installation?view=sql-server-2016)

[Andre Essing - SQL Server Best Practices](https://ru.scribd.com/document/366769757/DEUTSCH-20160401-SQL-Saturday-494-Vienna-SQL-Server-Best-Practices)

[ARCHITECTING MICROSOFT SQL SERVER ON VMWARE VSPHERE](https://www.vmware.com/content/dam/digitalmarketing/vmware/en/pdf/solutions/sql-server-on-vmware-best-practices-guide.pdf)

[DBCC TRACEON — флаги трассировки](https://docs.microsoft.com/ru-ru/sql/t-sql/database-console-commands/dbcc-traceon-trace-flags-transact-sql?view=sql-server-2016)

[Recommended updates and configuration options for SQL Server 2012 and SQL Server 2014 with high-performance workloads](https://support.microsoft.com/en-us/help/2964518/recommended-updates-and-configuration-options-for-sql-server-2012-and)

