# РУКОВОДСТВО ПРОГРАММИСТА ВИРТУАЛЬНОЙ КАССЫ ДЛЯ ИНТЕГРАЦИИ С FiscalDriveService-v10

## Введение
В данной инструкции описаны действия по установки и настройки ПО FiscalDriveService, описание всех функций ПО FiscalDriveService а также вызов функций ПО FiscalDriveService посредством swagger-интерфейса. 

ПО FiscalDriveService приходит на замену ПО FiscalDriveAPI. ПО FiscalDriveService поддерживает ФМ версии 0400. В связи с фундаментальными изменениями в ФМ версии 0400, ПО FiscalDriveAPI не будет поддерживать ФМ версии 0400. 

Информация по ФМ версии 0400 по адресу https://github.com/qo0p/acrsim-android.

## Определения
* **ПО** - программное обеспечение.
* **ОС** - операционная система.
* **TCP** - протокол передачи данных по сети интернет.
* **REST** - архитектурный стиль взаимодействия компонентов распределённого приложения в сети.  (https://ru.wikipedia.org/wiki/REST).
* **Swagger** - представляет собой формализованную спецификацию и полноценный фреймворк для описания, создания, использования и визуализации веб-сервисов REST. https://ru.wikipedia.org/wiki/OpenAPI_(%D1%81%D0%BF%D0%B5%D1%86%D0%B8%D1%84%D0%B8%D0%BA%D0%B0%D1%86%D0%B8%D1%8F)
* **ОФД** - оператора фискальных данных.
* **ФМ** - фискальный модуль выданный ОФД в виде USB-ключа или Смарт-карты. 
* **ЭЦП** - электронно-цифровая подпись.
* **БД** - база данных.
* **PCSCD** - Linux сервис предоставляющий программный доступ к подключенным к компьютеру смарт-картам.
* **TLS** - протокол защиты транспортного уровня.
* **ИБ** - информационная безопасность.
* **PFX** - формат файла для хранения ключей и сертификатов ЭЦП в зашифрованном виде.

## Принцип работы ПО FiscalDriveService
После установки FiscalDriveService в ОС Windows регистрируется как сервис «FiscalDriveService». FiscalDriveService слушает TCP порт (по умолчанию 3449) на котором запускается REST-API сервис по адресу http://127.0.0.1:3449/ а также swagger-интерфейс по адресу http://127.0.0.1:3449/swagger/index.html. 

FiscalDriveService получает REST-API команды и транслирует их в команды ФМ. Ответы ФМ транслирует обратно в ответы JSON. Чеки шифруются и подписываются ЭЦП ФМ и временно сохраняются в файле БД FiscalDriveService (определяемым параметром `--db-dir`). По команде виртуальной кассы FiscalDriveService соединяется с серверами ОФД и отправляет файлы чеков. В ответ получает файлы подтверждения и передаются в ФМ. ФМ принимает файлы подтверждения и стирает чек из памяти ФМ. 

В файле конфигурации FiscalDriveService (`config.ini`) прописываются IP-адреса или доменные имена серверов ОФД. Если FiscalDriveService не смог соединится с одним сервером, он случайным образом выбирает другой сервер и пытается заново отправить файлы.

## Технические требования к компьютеру и ОС
ПО FiscalDriveService может работать в ОС Windows 11 (x64), Linux (x64, arm). Объем ОЗУ не менее 2 Гбайт. Свободное пространство на диске не менее 512 Мбайт. Наличие Интернет соединения. Наличие блока бесперебойного питания для компьютера, во избежании потери данных в файле БД при аварийном отключении компьютера. 

## Требования к ОС Windows
Для работы в ОС Windows, требуется установленный драйвер Смарт-карт а также запущенная Windows-служба “Смарт-карта”.

## Требования к ОС Linux
Для работы в ОС Linux, требуются установка пакетов `pcscd` `libccid` `libpcsclite1` `libpcsclite-dev` `pcsc-tools` и запущена служба PCSCD.
В некоторых случаях, если ФМ не определился, требуется внести запись VendorID/ProductID ФМ в файл `/usr/lib/pcsc/drivers/ifd-ccid.bundle/Contents/Info.plist` и перезапустить службу PCSCD. Пример приведен в документе https://www.rutoken.ru/manual/RutokenOSXUnix.pdf.

## Требования к точности системных часов компьютера
Системные часы компьютера должны быть синхронизированы с сервером точного времени. В случае если системные часы компьютера будут опережать реальное время на более чем 2 дня и в этот момент произойдет операция с ФМ, то ФМ не выполнит операцию. После установки точного времени, ФМ будет выполнять операцию. 

## Установка ПО FiscalDriveService
ПО FiscalDriveService и ФМ устанавливаются непосредственно на компьютер где установлено ПО Кассы. Требуется интернет соединение для этого компьютера. 
Если на компьютере уже была установлена раняя версия ПО FiscalDriveService, то перед установкой новой версии убедитесь, что в памяти ФМ нет неотправленных чеков. После отправки всех неотправленных чеков из памяти ФМ, можно будет удалить старую версию ПО FiscalDriveService и перейти к установки FiscalDriveService.

### Режим работы с применением TLS
Если требуется обеспечить информационную безопасность при обмене данных между ПО кассы и ПО FiscalDriveService, то применяется данный режим.
По умолчанию ПО FiscalDriveService запускается с параметром без-TLS.

Для запуска сервиса FiscalDriveService в режиме (с параметром `--use-tls`) TLS требуется предварительно создать корневой сертификат и ключ (`root.crt`, `root.key`), сертификат и ключ сервера (`server.crt`, `server.key`) и PFX ключ клиента для ПО кассы. В режиме TLS ПО кассы соединения с ПО FiscalDriveService происходит по протоколу HTTPS, с применением PFX ключа.

### Режим работы без применения TLS
При запуске ПО FiscalDriveService без режима TLS. ПО кассы соединяется с ПО FiscalDriveService по протоколу HTTP. 

### Режим сервера для приёма тестовых чеков
Возможен запуск ПО FiscalDriveService в режиме работы в качестве сервера для приёма тестовых чеков от ФМ который работает в тестовом режиме или эмулятора ФМ. Данный режим позволяет разработчику выполнять разработку и тестирование кассового ПО без подключения к интернету и отправки чеков на сервереа ОФД.  В данном режиме ПО FiscalDriveService принимает только чеки сформированные от ФМ который работает в тестовом режиме или эмулятора ФМ. Сервера ОФД не принимают тестовые чеки, также и ПО FiscalDriveService работающий в данном режиме не принимает чеки от зарегистрированого в ОФД ФМ. 

Серийный номер ФМ работающего в тестовом режиме или эмулятора - **ZZ000000000000**.

### Режим эмулятора ФМ
Возможен запуск ПО FiscalDriveService в режиме работы в качестве эмулятора ФМ версии 0400. В данном режиме эмулируется физический ФМ который будет доступен по указанному в параметре запуска IP-адресу и TCP-порту. В парамете запуска также можно указать память ОЗУ и ПЗУ а также можно эмулировать (с заданной вероятностью) сбой подключения (при выполнении заданной APDU-команды) в ФМ. Память эмулятора храниться в ОЗУ компьютера сбрасывается при каждом завершении эмулятора.

### Подключение ФМ в компьютер
В первую очередь необходимо подключить ФМ в USB-порт компьютера. Установка драйвера ФМ осуществляется автоматически в ОС Windows. Для ускорения установки стандартного драйвера возможно потребуется временно отключить Интернет соединение перед подключением ФМ. 

ОС Linux автоматически определяет подключенный ФМ если установлены соответствующие программные пакеты (см. Раздел **Требования к ОС Linux**) 

### Установка ПО FiscalDriveService в ОС Windows
Для установки ПО FiscalDriveService в ОС Windows нужно запустить программу установщик Setup-FiscalDriveService-v10.XX.exe от имени Администратора и следовать инструкциям установщика. 

Установщик предложит выбрать режим установки:
- **Обычная установка** - установка для работы ПО FiscalDriveService в режиме без TLS. 
- **Установка с применением TLS** - установка для работы ПО FiscalDriveService в режиме TLS.

Установщик распакует программу `fiscal-drive-service.exe` и файл конфигурации `config.ini` в заданную установщиком директорию (обычно в `c:\Program Files\FiscalDriveService`)
Также возможна “тихая” установка. Для этого откройте командную строку и наберите команду:
```
Setup-FiscalDriveService-v10.XX.exe /SILENT
```

По умолчанию будет выбран режим “**Обычная установка**”.

### Установка ПО FiscalDriveService в ОС Linux
Для установки ПО FiscalDriveService в ОС Linux нужно распаковать файл `fiscal-drive-service-v10.XX.tar.gz` в директорию. После распаковке в заданной директории появятся программа `fiscal-drive-service` и файл конфигурации `config.ini`.

## Конфигурация ПО FiscalDriveService
ПО FiscalDriveService обычно не нуждается в конфигурации и может запускаться с конфигурацией по умолчанию.

Для того чтобы удостоверится что ПО FiscalDriveService может подключиться к ФМ подключенному в компьютер, откройте командную строку (в ОС Windows - cmd.exe, в ОС Linux - Gnome Terminal) и перейдите в директорию установки ПО FiscalDriveService.
Для получения списка подключенных ФМ наберите команду:

* Для ОС Windows
    ```
    fiscal-drive-service.exe admin fd list
    ```

* Для ОС Linux
    ```
    ./fiscal-drive-service admin fd list
    ```

> _Для просмотра дополнительных опций по этой команде введите команду с параметром `-h`_   

В результате на экране должен отобразится список подключенных ФМ с полями:
- `FactoryID` - заводской номер ФМ состоящий из 36 символов. (В эмуляторе ФМ заводской номер меняется при каждом перезапуске). 
- `ReaderName` - название считывателя смарт-карт.
- `ATR` - ATR смарт-карты.
- `AppletVersion` - Версия апплетя ФМ (ПО FiscalDriveService работает только в версией 0400).

Если на экране ничего не появилось или появился текст ошибки, то возможно команда была введена не правильно или с другой директории, или ОС не определила ФМ (см. Раздел **Требования к ОС Windows** и Раздел **Требования к ОС Linux**).  

Вводить заводской номер ФМ в файл конфигурации не нужно (, как это было в ПО FiscalDriveAPI). ПО кассы должна будет сохранить заводской номер ФМ в своем файле конфигурации и при вызове REST-API метода передавать его.

### Конфигурация ПО FiscalDriveService в режиме TLS
Если был выбран режим установки **Установка с применением TLS**, то перед запуском требуется создать корневой сертификат и ключ (`root.crt`, `root.key`), сертификат и ключ сервера (`server.crt`, `server.key`) и PFX ключ клиента для ПО кассы.

Создание TLS ключей и сертификатов должно производится на компьютере администратора/специалиста по ИБ. 

После создания всех нужных ключей и сертификатов файлы:
* `root.crt` - копируются в директорию tls в директории установки на компьютере Кассы
* `server.crt`, `server.key` - копируются в директорию tls в директории установки на компьютере кассы (для каждой кассы или ее определенной группы может быть созданы разные ключи и сертификаты)
* `keystore.pfx` - копируются в директорию кассового ПО с соответствии с ее настройками или устанавливаются в ОС кассы.

#### Создание корневого сертификата и ключа
Для создание корневого сертификата и ключа введите команду (в одну строку):

* Для ОС Windows
    ```
    fiscal-drive-service.exe admin tls root “<название торговой точки>” “<название организации>” --valid-years “<срок сертификата в годах>”
    ```

* Для ОС Linux
    ```
    ./fiscal-drive-service admin tls root “<название торговой точки>” “<название организации>” --valid-years “<срок сертификата в годах>”
    ```

> _Для просмотра дополнительных опций по этой команде введите команду с параметром `-h`_   

В директории установки появится новая директория tls с файлами `root.crt` и `root.key`. 

#### Создание сертификата и ключ сервера
Для создание сертификата и ключа кассы введите команду (в одну строку):

* Для ОС Windows
    ```
    fiscal-drive-service.exe admin tls server “<название и номер кассы>” “<название организации>” --valid-years “<срок сертификата в годах>” --san-domain localhost --san-ip 127.0.0.1
    ```

* Для ОС Linux
    ```
    ./fiscal-drive-service admin tls server “<название и номер кассы>” “<название организации>” --valid-years “<срок сертификата в годах>” --san-domain localhost --san-ip 127.0.0.1
    ```

> _Для просмотра дополнительных опций по этой команде введите команду с параметром `-h`_   

В директории tls появится файлы `server.crt` и `server.key`. 

#### Создание PFX ключа для ПО кассы
Для создание PFX ключа для ПО кассы введите команду (в одну строку):

* Для ОС Windows
    ```
    fiscal-drive-service.exe admin tls client “<название и номер кассы>” “<название организации>” “<путь к файлу pfx>“ --valid-years “<срок сертификата в годах>”
    ```

* Для ОС Linux
    ```
    ./fiscal-drive-service admin tls client  “<название и номер кассы>” “<название организации>” “<путь к файлу pfx>“ --valid-years “<срок сертификата в годах>”
    ```

> _Для просмотра дополнительных опций по этой команде введите команду с параметром `-h`_   

На экране появится запрос о вводе пароля нового PFX ключа. После ввода пароля будет создан PFX файл по указанному пути в команде.

_Для тестирования REST-API сервиса посредством браузера Internet Explorer или Chrome (через swagger-интерфейс), нужно импортировать (установить) сертификат `root.crt` в раздел «доверительные корневые центры сертификации» и PFX файл в раздел по умолчанию._

## Запуск сервиса FiscalDriveService

### Запуск сервиса FiscalDriveService на ОС Windows

#### Запуск через “Службы и приложения”
Для запуска сервиса необходимо через управления компьютером во вкладки “Службы и приложения” осуществить запуск (перезапуск) службы “FiscalDriveService”
Если служба не запустилась, то возможно имеются ошибки в файле конфигурации FiscalDriveService. Для определения причины откройте лог-файл `service.log` в директории установки. Также для определения ошибки запустите в командной строке и посмотрите лог на экране.

#### Запуск в режиме командной строки
Для этого необходимо через управления компьютером во вкладки “Службы и приложения” осуществить остановку службы “FiscalDriveService” и в командном окне выполнить команду:

* В обыном режиме
    ```
    fiscal-drive-service.exe api config.ini --log-level 5 --log-output -
    ```

* В режиме с TLS
    ```
    fiscal-drive-service.exe api config.ini --log-level 5 --log-output - --use-tls
    ```

> _Для просмотра дополнительных опций по этой команде введите команду с параметром `-h`_   

Параметр `--log-level` определяет уровень логирования.

### Запуск сервиса FiscalDriveService на ОС Linux
* В обыном режиме
    ```
    ./fiscal-drive-service api config.ini --log-level 5 --log-output -
    ```
* В режиме с TLS
    ```
    ./fiscal-drive-service api config.ini --log-level 5 --log-output - --use-tls
    ```

> _Для просмотра дополнительных опций по этой команде введите команду с параметром `-h`_    

Параметр `--log-level` определяет уровень логирования.

Для автоматического запуска сервиса FiscalDriveService администратор должен будет оформить команду запуска в виде файла сервиса upstart или systemd согласно документации дистрибутива Linux

### Запуск ПО FiscalDriveService в режиме приёма тестовых чеков
В данном режиме ПО FiscalDriveService принимает чеки только от тестового ФМ с серийным номером **ZZ000000000000**.

Выполнить команду:

* Для ОС Windows
    ```
    fiscal-drive-service.exe devtool test-server
    ```

* Для ОС Linux
    ```
    ./fiscal-drive-service devtool test-server
    ```
> _Для просмотра дополнительных опций по этой команде введите команду с параметром `-h`_   

Тестовый сервер откроет TCP-порт 13447.

Для того чтобы FiscalDriveService отправляло файлы тестовых чеков с тестового ФМ на тестовый сервер, то следует запустить FiscalDriveService со следующими параметрами командной строки:

* Для ОС Windows
    ```
    fiscal-drive-service.exe api config.ini -o - --log-level 7 --server-address 127.0.0.1:13447
    ```

* Для ОС Linux
    ```
    ./fiscal-drive-service api config.ini -o - --log-level 7 --server-address 127.0.0.1:13447
    ```

> _Для просмотра дополнительных опций по этой команде введите команду с параметром `-h`_   

> _При указании адреса сервера в параметре `--server-address`, адреса серверов прописанных в файле конфигурации будут игнорированы. Можно указать несколько параметров `--server-address`, например добавить несуществующий адрес для тестирования случая когда определенный сервер не доступен._


### Запуск ПО FiscalDriveService в режиме эмулятора ФМ

Выполнить команду:

* Для ОС Windows
    ```
    fiscal-drive-service.exe devtool fiscal-drive-emulator
    ```

* Для ОС Linux
    ```
    ./fiscal-drive-service devtool fiscal-drive-emulator
    ```
> _Для просмотра дополнительных опций по этой команде введите команду с параметром `-h`_   

Для эмулировани сбоя соединения с ФМ нужно добавить параметры:
- `--fail-apdu 001700` - эмулировать сбой соединения при выполнении APDU-команды 001700 (регистрация чека в ФМ)
- `--fail-chance 25` - сбой с вероятностью 25% при выполнении APDU-команды.

Эмулятор откроет TCP-порт 4387.

Для того чтобы FiscalDriveService подключалась к эмулатору ФМ, то следует запустить FiscalDriveService со следующими параметрами командной строки:

* Для ОС Windows
    ```
    fiscal-drive-service.exe api config.ini -o - --log-level 7 --server-address 127.0.0.1:13447 --use-fiscal-drive-emulator-address 127.0.0.1:4387
    ```

* Для ОС Linux
    ```
    ./fiscal-drive-service api config.ini -o - --log-level 7 --server-address 127.0.0.1:13447 --use-fiscal-drive-emulator-address 127.0.0.1:4387
    ```

> _Для просмотра дополнительных опций по этой команде введите команду с параметром `-h`_   

## Описание методов REST-API
После запуска FiscalDriveService, REST-API доступен по адресу http://127.0.0.1:3449/, а swagger-интерфейс по адресу http://127.0.0.1:3449/swagger/index.html. 

| Метод | Описание |
|-------|----------|
| `/FiscalDrive/List` | Получить список подключенных ФМ |
| `/FiscalDrive/Info/{FactoryID}` | Получить информацию о ФМ |
| `/FiscalDrive/State/Sync/{FactoryID}` | Синхронизировать состояние ФМ с сервером для установки серверного времени, разблокировки и др. |
| `/FiscalDrive/FiscalMemory/Info/{FactoryID}` | Получить информацию о фискальной памяти ФМ |
| `/FiscalDrive/ZReport/Open/{FactoryID}` | Открыть ZReport |
| `/FiscalDrive/ZReport/Close/{FactoryID}` | Закрыть ZReport |
| `/FiscalDrive/ZReport/Info/{FactoryID}` | Получить информацию о ZReport |
| `/FiscalDrive/ZReport/UnackowledgedIndexes/{FactoryID}` | Получить индексы неотправленных на сервер ZReport |
| `/FiscalDrive/Receipt/GetTXID/{FactoryID}` | Записать в БД JSON-чек и получить TXID (идентификатор чека в БД) |
| `/FiscalDrive/Receipt/RegisterTXID/{FactoryID}` | Зарегистрировать JSON-чек по TXID в ФМ. Данный метод можно вызывать повторно, в случае сбоя соединения с ФМ. |
| `/FiscalDrive/Receipt/Info/{FactoryID}` | Получить информацию о Receipt |
| `/DataBase/Files/Count` | Получить кол-во файлов в БД |
| `/DataBase/Files/Status/Reset` | Сбросить состояние файла в БД, для последующей повториной её отправки на сервер ОФД |
| `/DataBase/Files/List/{FactoryID}/{Limit}/{Offset}` | Получить список файлов в БД и их состояния |
| `/DataBase/Files/Sync/FullReceipts/{FactoryID}` | Отправка файлов типа FullReceipt из БД на сервер ОФД и получение ответный файлов и передача их в ФМ |
| `/DataBase/Files/Sync/ZReports/{FactoryID}` | Отправка файлов типа ZReportFile из ФМ на сервер ОФД и получение ответный файлов и передача их в ФМ |
| `/DataBase/Files/Sync/Receipts/{FactoryID}` | Отправка файлов типа ReceiptFile из ФМ на сервер ОФД и получение ответный файлов и передача их в ФМ |
| `/Auth/SignChallenge/{FactoryID}` | Применяется для аутентификации по ФМ на сайте (или API) который поддерживает данную функцию, API сайта возвращает Challenge который передается в ФМ, ФМ возвращает подписанный ответ. Подписанный ответ следует отправить на API сайта для верификации и получения Access-Token. Далее по Access-Token вызывает закрытые методы API сайта. |
| `/POS/Lock/{FactoryID}` | Устанавливает секретный ключ ЦТО в ФМ. При вызове метода передается сам секретный ключ и его хеш SHA-256. После установки ключа для каждой операций открытии/закрытии ZReport или регистрации чека нужно передавать ключ аутентификации в HTTP-заголовке `X-POS-Auth` который вычисляется как `POSAuth = SHA-256(секретный ключ + POSChallenge)`. Разблокировка возможно только по команде сервера ОФД. |
| `/POS/Challenge/{FactoryID}` | Для получения `POSChallenge`. Меняется после каждой операции открытии/закрытии ZReport или регистрации чека. |
| `/POS/Auth/{FactoryID}` | Аутентификация по секретному ключу в ФМ для тестирования |

> _Тестовый JSON-чек можно сгенерировать командой `fiscal-drive-service devtool receipt generate`. Для просмотра дополнительных опций по этой команде введите команду с параметром `-h`_ 

## Коды и описания ошибок

Коды и описания ошибок по ФМ версии 0400 смотрите по адресу https://github.com/qo0p/acrsim-android.