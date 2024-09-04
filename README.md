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
Возможен запуск ПО FiscalDriveService в режиме работы в качестве эмулятора ФМ версии 0400. В данном режиме эмулируется физический ФМ который будет доступен по указанному в параметре запуска IP-адресу и TCP-порту. В парамете запуска также можно указать память ОЗУ и ПЗУ а также можно эмулировать (с заданной вероятностью) сбой подключения (при выполнении заданной APDU-команды) в ФМ. 

Память эмулятора храниться в ОЗУ компьютера сбрасывается при каждом завершении эмулятора. При каждом запуске, эмулятору ФМ назначается новый заводской номер.

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

Для установки состояния ФМ работающего в тестовом режиме или эмулятора (при выполнении синхронизации состояния с сервером ОФД), тестовый сервер предоставляет разработчику API по адресу http://127.0.0.1:13448/state. API дает возможность блокировать/разблокировать и отвязать от ККМ ФМ работающего в тестовом режиме или эмулятора. После установки состояния в API, следует выполнить синхронизации состояния ФМ с сервером ОФД.

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

## JSON-чек

| Поле | Тип | Описание |
|------|-----|----------|
| Time | string | Текущая дата, время регистрации чека (формат ГГГГ-ММ-ДД ЧС-МН-СК) |
| ReceivedCash | uint64 | Наличная сумма полученная от продажи в тийин |
| ReceivedCard | uint64 | Безналичная сумма полученная от продажи в тийин |
| Type | uint8 | Тип чека: 0 - Обычный (покупка), 1 - Аванс, 2 - Кредит |
| Operation | uint8 | Тип операции: 0 - Продажа, 2 - Возврат (отзыв) |
| Location | Object | Геолокация торговой точки |
| Items | []Object | Массив информации о товарах/услугах и их цен |
| ExtraInfo | Object | Доп. Информация |
| RefundInfo | Object | Информацию об отозванном чеке (заполняется в чеке возврата) |

#### Location

| Поле | Тип | Описание |
|------|-----|----------|
| Latitude | float64 | Широта |
| Longitude | float64 | Долгота |

#### Item

| Поле | Тип | Описание |
|------|-----|----------|
| Name | string | Наименование товара или услуги |
| Barcode | string | Товарный код |
| Label | string | Код маркировки |
| SPIC | string | ИКПУ (идентификационный код продукта и услуги по Единому электронному каталогу) |
| Units | uint64 | Единица измерения |
| PackageCode | string | Код упаковки |
| OwnerType | uint8 | Тип владельца продукта/услуги (см. справочник) |
| Amount | uint64 | Количество товара |
| Price | uint64 | Общая сумма позиции без учета скидок |
| Discount | uint64 | Cкидка |
| Other | uint64 | Прочие (Оплата по страховки и др.) |
| VATPercent | uint8 | % НДС |
| VAT | uint64 | НДС сумма |
| CommissionInfo | Object | Признак комиссионного товара/услуги |

#### CommissionInfo

| Поле | Тип | Описание |
|------|-----|----------|
| TIN | string | ИНН |
| PINFL | string | ПИНФЛ |

#### ExtraInfo

| Поле | Тип | Описание |
|------|-----|----------|
| TIN | string | ИНН |
| PINFL | string | ПИНФЛ |
| PhoneNumber | string | №. Телефона (формат 998XXYYYAABB) |
| CarNumber | string | Гос. №. знак авто |
| QRPaymentID | string | Идентификатор платежа по QR-коду |
| QRPaymentProvider | uint16 | Код провайдера услуг по оплате по QR-коду |
| CashedOutFromCard | uint64 | Сумма обналиченных из карты денег |
| CardType | uint8 | Тип пластиковой карты (2-личная, 1-корпоративная) |
| PPTID | string | Идентификатор транзакции платежа процессингово центра |

#### RefundInfo

| Поле | Тип | Описание |
|------|-----|----------|
| TerminalID | string | Сер.№ ФМ где был зарегистрирован отзываемый чек |
| ReceiptSeq | uint64 | Номер отзываемого чека |
| DateTime | string | Дата, время отзываемого чека (формат ГГГГММДДЧСМНСК) |
| FiscalSign | string | ФП отзываемого чека |

> Если было передано некоректное значение в полях RefundInfo, то регистрация чека не произойдет, выдаст ошибку.

## Порядок тестирования

- Установите FiscalDriveService от имени Администратора на компьютер разработчика, если уже установлено, то остановите Windows-службу “FiscalDriveService”.
- Откройте командную строку cmd.exe и перейдите в директорию установки ПО FiscalDriveService.
- Запустите эмулятор ФМ командой: 
    ```
    fiscal-drive-service.exe devtool fiscal-drive-emulator
    ```
- Откройте новую командную строку cmd.exe и перейдите в директорию установки ПО FiscalDriveService.
- Запустите тестовый сервер командой:
    ```
    fiscal-drive-service.exe devtool test-server -e 7
    ```
- Откройте новую командную строку cmd.exe и перейдите в директорию установки ПО FiscalDriveService.
- Запустите REST-API командой:
    ```
    fiscal-drive-service.exe api config.ini -o - -e 7 --server-address 127.0.0.1:13447 --use-fiscal-drive-emulator-address 127.0.0.1:4387
    ```
- Откройте адрес в браузере http://127.0.0.1:3449/swagger/index.html.
- Откройте вкадку `/FiscalDrive/List`, нажмите `Try it out`, нажмите `Execute`. В разделе `Server response` появится JSON-ответ, например:
    ```
    [
        {
            "ReaderName": "127.0.0.1:4387",
            "ATR": "3b8f800180318065b08503010101030201040105",
            "Description": "",
            "FactoryID": "002024083000152503000000000000000000",
            "AppletVersion": "0400"
        }
    ]
    ```
    где:
    * `FactoryID` - заводской номер ФМ (в данном случае эмулятора). Скопируйте значение этого поля в блокнот для последующей вставки.
- Откройте вкадку `/FiscalDrive/Info/{FactoryID}`, нажмите `Try it out`, вставьте в поле `FactoryID` заводской номер ФМ, нажмите `Execute`. В разделе `Server response` появится JSON-ответ, например:
    ```
    {
        "AppletVersion": "0400",
        "TerminalID": "ZZ000000000000",
        "SyncChallenge": "6ae708515ad6ee84",
        "Locked": true,
        "JCREVersion": "0300",
        "POSLocked": false,
        "POSAuth": false,
        "MemoryInfo": {
            "AvailablePersistentMemory": 694,
            "AvailableResetMemory": 3230,
            "AvailableDeselectMemory": 3230
        }
    }
    ```
    где:
    * `Locked: true` означает что ФМ блокирован (и нет возможности открыть/закрыть ZReport и зарегистрировать чек в ФМ). Для разблокировки нужно выполнить операцию `/FiscalDrive/State/Sync/{FactoryID}`, но для этого оператор ОФД должен будет разрешить разблокировку для этого ФМ. Тестовый сервер имеет свой API для разблокировки тестового или эмулятора ФМ.
- Для разблокировки ФМ со стороны оператора ОФД на тестовом сервере откройте адрес в браузере http://127.0.0.1:13448/state. Установите значения `ChangeLockState = Yes`,  `LockState = Unlock` и нажмите кнопку `Set`. После этого следует выполнить синхронизацию состояния на стороне ФМ.
- Откройте вкадку `/FiscalDrive/State/Sync/{FactoryID}`, нажмите `Try it out`, вставьте в поле `FactoryID` заводской номер ФМ, нажмите `Execute`. В разделе `Server response` появится ответ `OK`. Для проверки состояния выполните операцию `/FiscalDrive/Info/{FactoryID}` и убедитесь что поле изменилось на `Locked: false`.
- Откройте вкадку `/FiscalDrive/ZReport/Open/{FactoryID}`, нажмите `Try it out`, вставьте в поле `FactoryID` заводской номер ФМ, нажмите `Execute`. В разделе `Server response` появится ответ `OK`. Таким образом будет открыт ZReport в ФМ и можно будет регистрировать чеки.
- Откройте вкадку `/FiscalDrive/Receipt/GetTXID/{FactoryID}`, нажмите `Try it out`, вставьте в поле `FactoryID` заводской номер ФМ, в поле `JsonReceipt` введите чек в формате JSON (или сгенерируйте командой `fiscal-drive-service.exe devtool receipt generate`, скопируйте и вставьте) например:
    ```
    {
        "ReceivedCash": 50000,
        "ReceivedCard": 50000,
        "Time": "2024-09-04 09:45:38",
        "Type": 0,
        "Operation": 0,
        "Location": {
            "Latitude": 69.21882407810848,
            "Longitude": 41.29480079557751
        },
        "Items": [
            {
                "Name": "AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA",
                "Barcode": "1247059838170",
                "Label": "8919525180645",
                "SPIC": "90363039122553629",
                "Units": 244272402,
                "PackageCode": "11580107597508352307",
                "OwnerType": 1,
                "Price": 100000,
                "VATPercent": 12,
                "VAT": 5357,
                "Amount": 2000,
                "Discount": 50000,
                "Other": 0,
                "CommissionInfo": {
                        "TIN": "346496427"
                }
            },
            {
                "Name": "BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB",
                "Barcode": "5868469653263",
                "Label": "6573308424703",
                "SPIC": "07219001697739637",
                "Units": 142235374,
                "PackageCode": "87290838460667151841",
                "OwnerType": 0,
                "Price": 50000,
                "VATPercent": 12,
                "VAT": 5357,
                "Amount": 4000,
                "Discount": 0,
                "Other": 0
            }
        ],
        "ExtraInfo": {
            "CarNumber": "17D613JU",
            "PhoneNumber": "998726286833",
            "QRPaymentID": "17adc290-3672-4675-b86d-43256432d0f6",
            "QRPaymentProvider": 18,
            "CashedOutFromCard": 18134,
            "PPTID": "35",
            "CardType": 1
        }
    }
    ```
    нажмите `Execute`. В разделе `Server response` появится номер чека в БД - `TXID`.
- Откройте вкадку `/FiscalDrive/Receipt/RegisterTXID/{FactoryID}`, нажмите `Try it out`, вставьте в поле `FactoryID` заводской номер ФМ, в поле `TXID` номер чека в БД (от предыдущей операции), нажмите `Execute`. В разделе `Server response` появится JSON-ответ, например:
    ```
    {
        "TerminalID": "ZZ000000000000",
        "ReceiptSeq": 1,
        "DateTime": "2024-09-04 09:45:38",
        "FiscalSign": "320262508539",
        "QRCodeURL": "https://ofd.soliq.uz/check?t=ZZ000000000000&r=1&c=20240904094538&s=320262508539"
    }
    ```
    где:
    * `QRCodeURL` - URL адрес ссылки для проверки чека. Должен быть напечатан в виде QR-кода на чеке. Данный JSON-ответ должен быть сохранен в БД ПО Кассы и будет нужен в случае выполнения операции возврат по чеку. Если при выполнении возник сбой соединения с ФМ, то данную операцию можно выполнить повторно (после восстановления соединения), если во время сбоя ФМ незарегистрировал чек то он зарегистрирует и вернет ответ, а если этот чек уже был заристрирован в ФМ то ФМ вернет ответ от прежней регистрации (**не будет создан дубликат или пустой чек**).
- Для того чтобы получить информацию об уже зарегистрированных чеков в ФМ (который еще не стерлись из памяти ФМ после отправки их на сервер ОФД), то откройте вкадку `/FiscalDrive/Receipt/Info/{FactoryID}`, нажмите `Try it out`, вставьте в поле `FactoryID` заводской номер ФМ, в поле `Index` индекс чека в памяти ФМ, нажмите `Execute`. В разделе `Server response` появится JSON-ответ, например:
    ```
    {
        "TerminalID": "ZZ000000000000",
        "ReceiptSeq": 1,
        "Time": "2024-09-04 09:45:38",
        "FiscalSign": "320262508539",
        "ReceiptType": "Purchase",
        "OperationType": "Sale",
        "ReceivedCash": 50000,
        "ReceivedCard": 50000,
        "TotalVAT": 10714,
        "ItemsCount": 2
    }
    ```
- Для того чтобы закрыть ZReport, откройте вкадку `/FiscalDrive/ZReport/Close/{FactoryID}`, нажмите `Try it out`, вставьте в поле `FactoryID` заводской номер ФМ, нажмите `Execute`. В разделе `Server response` появится ответ `OK`. Таким образом будет закрыть ZReport в ФМ.
- Для того чтобы получить информацию о ZReport в ФМ, то откройте вкадку `/FiscalDrive/ZReport/Info/{FactoryID}`, нажмите `Try it out`, вставьте в поле `FactoryID` заводской номер ФМ, в поле `Index` индекс ZReport в памяти ФМ, нажмите `Execute`. В разделе `Server response` появится JSON-ответ, например:
    ```
    {
        "TerminalID": "ZZ000000000000",
        "OpenTime": "2024-09-04 09:39:14",
        "CloseTime": "2024-09-04 10:15:31",
        "TotalSaleCount": 1,
        "TotalRefundCount": 0,
        "TotalCash": {
            "Sale": 50000,
            "Refund": 0
        },
        "TotalCard": {
            "Sale": 50000,
            "Refund": 0
        },
        "TotalVAT": {
            "Sale": 10714,
            "Refund": 0
        },
        "FirstReceiptSeq": 1,
        "LastReceiptSeq": 1
    }
    ```
- Для того чтобы получить информацию о фискальной памяти в ФМ, то откройте вкадку `/FiscalDrive/FiscalMemory/Info/{FactoryID}`, нажмите `Try it out`, вставьте в поле `FactoryID` заводской номер ФМ, в поле `Index` индекс ZReport в памяти ФМ, нажмите `Execute`. В разделе `Server response` появится JSON-ответ, например:
    ```
    {
        "TerminalID": "ZZ000000000000",
        "ReceiptSeq": 1,
        "LastOperationTime": "2024-09-04 10:15:31",
        "FirstUnacknowledgedReceiptTime": "2024-09-04 09:45:38",
        "ZReportsCount": 1,
        "ReceiptsCount": 1,
        "CashAccomulator": {
            "Sale": 50000,
            "Refund": 0
        },
        "CardAccomulator": {
            "Sale": 50000,
            "Refund": 0
        },
        "VATAccomulator": {
            "Sale": 10714,
            "Refund": 0
        }
    }
    ```
    где: 
    * `ReceiptSeq` - Последний номер зарегистрированного чека
    * `LastOperationTime` - дата-время последней операции или дата-время сервера ОФД после выполнения операции синхронизации состояния ФМ
    * `FirstUnacknowledgedReceiptTime` - дата-время самого первого зарегистрированного чека в ФМ который еще не был отравлен на сервер ОФД (если эта дата-время отстает на 2 или более дней от `LastOperationTime` то ФМ не даст выполнить операцию открыть/закрыть ZReport и зарегистрировать чек в ФМ пока не будут отправлены все чеки на сервер ОФД)
    * `ZReportsCount` - кол-во ZReport в ФМ
    * `ReceiptsCount` - кол-во чеков в ФМ которые еще не были отправлены на сервер ОФД.
    * `***Accomulator` - общая накопленная сумма.
- Для того чтобы отправить файлы чеков из БД на сервер ОФД, то откройте вкадку `/DataBase/Files/Sync/FullReceipts/{FactoryID}`, нажмите `Try it out`, вставьте в поле `FactoryID` заводской номер ФМ, в поле `ItemsCount` максимальное кол-во файлов (не более 32), нажмите `Execute`. В разделе `Server response` появится JSON-ответ, например:
    ```
    {
        "Items": [
            {
                "TerminalID": "ZZ000000000000",
                "ReceiptSeq": 1,
                "StatusText": "Acknowledged"
            }
        ],
        "SuccessfulsCount": 1
    }
    ```
    где:
    * `StatusText: Acknowledged` - файл чека был отправлен на сервер ОФД и ответ от него успешно принят ФМ и ФМ стер чек из памяти. Выполните операцию `/FiscalDrive/FiscalMemory/Info/{FactoryID}` и убедитесь что поле стало `ReceiptsCount: 0`. 
- Для того чтобы узнать индексы ZReport которые еще не были отправлены на сервер ОФД, то то откройте вкадку `/FiscalDrive/ZReport/UnackowledgedIndexes/{FactoryID}`, нажмите `Try it out`, вставьте в поле `FactoryID` заводской номер ФМ, нажмите `Execute`. В разделе `Server response` появится JSON-ответ, например:
    ```
    [
        0
    ]
    ```
    в данном ответе ZReport с индексом 0 еще не были отправлены на сервер ОФД.
- Для того чтобы отправить ZReport из ФМ на сервер ОФД, то откройте вкадку `/DataBase/Files/Sync/ZReports/{FactoryID}`, нажмите `Try it out`, вставьте в поле `FactoryID` заводской номер ФМ, в поле `ItemsCount` максимальное кол-во файлов (не более 32), нажмите `Execute`. В разделе `Server response` появится JSON-ответ, например:
    ```
    {
        "Items": [
            {
                "TerminalID": "ZZ000000000000",
                "CloseTime": "2024-09-04 10:15:31",
                "StatusText": "Acknowledged"
            }
        ],
        "SuccessfulsCount": 1
    }
    ```
    где:
    * `StatusText: Acknowledged` - ZReport был отправлен на сервер ОФД и ответ от него успешно принят ФМ и ФМ пометил ZReport как отправленный. Выполните операцию `/FiscalDrive/ZReport/Info/{FactoryID}` и убедитесь что появилось поле `AcknowledgedTime: "2024-09-04 10:51:40"`. 
- Операция `/DataBase/Files/Sync/Receipts/{FactoryID}` нужна только в случае потери или повреждении БД и в ФМ остались неотправленные чеки.
- Для того чтобы привязать ПО Кассы с ФМ каждое ЦТО генерирует свой секретный ключ 32-байт (SecretKey) и хранит этот ключ в защищенном от постороннего доступа месте (например в коде программы кассы или в файле конфигурации ПО Кассы предварительно зашифровав). Для привязки нужно выполнить операцию `/POS/Lock/{FactoryID}`, где в поле `SecretKey` нужно передать `SecretKey+SHA-256(SecretKey)`. Например `SecretKey = 66d11d7deee1f4eb5a49aa9c1865801e248bde364053eef7127ce07cc690dc5c` , то `SHA-256(SecretKey) = dff63f03c751d633aac3363f80cc2e48818e2b4886f7b909abab7414495f4adf`. `SecretKey+SHA-256(SecretKey) = 66d11d7deee1f4eb5a49aa9c1865801e248bde364053eef7127ce07cc690dc5cdff63f03c751d633aac3363f80cc2e48818e2b4886f7b909abab7414495f4adf`. Вводим в поле `SecretKey` `66d11d7deee1f4eb5a49aa9c1865801e248bde364053eef7127ce07cc690dc5cdff63f03c751d633aac3363f80cc2e48818e2b4886f7b909abab7414495f4adf`, нажмите `Execute`. В разделе `Server response` появится ответ `OK`. Выполните операцию `/FiscalDrive/Info/{FactoryID}` и убедитесь что поле `POSLocked: true`. Теперь **каждый раз** перед выполнением операций `/FiscalDrive/ZReport/Open/{FactoryID}`, `/FiscalDrive/ZReport/Close/{FactoryID}` и `/FiscalDrive/Receipt/RegisterTXID/{FactoryID}` нужно выполнить операцию /`POS/Challenge/{FactoryID}`, получить ответ (Challenge), например `bef8820355972fe5386012d836cb052d92ac4f9c93439f07d5ac89e94e0bcb74`, вычислить `SHA-256(SecretKey+Challenge) = 8e3c346fe5588ea5f28dfb60db9e496d78b88b0e1871773476c7d7906036b9bc` и вставить в поле `X-POS-Auth` операций где имеется это поле. Таким образом данный ФМ будет защищен от использования ПО Касс других ЦТО. Для развязки тестового или эмулятора ФМ от ККМ, откройте адрес http://127.0.0.1:13448/state и установите `POS Unlock = POS Unlock` и нажмите кнопку `Set`, далее выполните синхронизацию состояния ФМ с сервером ОФД.
    > Для выполнения вычислений по алгоритму `SHA-256` при тестировании данной операции можно воспользоваться онлайн инструментом по адресу https://emn178.github.io/online-tools/sha256.html (установите _Auto Update_, _Input Encoding = HEX_, _Output Encoding = HEX_).