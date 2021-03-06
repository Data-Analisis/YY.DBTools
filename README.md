# Набор библиотек для работы с СУБД

Набор небольших библиотек и приложений на .NET Core для упрощения работы с некоторыми СУБД в части задач разработки и администрирования.

| Проект | Актуальная версия | Описание |
| ----------- | ----------------- | -------- |
| YY.DBTools.Core | [![NuGet version](https://badge.fury.io/nu/YY.DBTools.Core.svg)](https://badge.fury.io/nu/YY.DBTools.Core) | Базовый пакет |
| YY.DBTools.SQLServer.XEvents | [![NuGet version](https://badge.fury.io/nu/YY.DBTools.SQLServer.XEvents.svg)](https://badge.fury.io/nu/YY.DBTools.SQLServer.XEvents) | Пакет для чтения файлов расширенных событий SQL Server |
| YY.DBTools.SQLServer.XEvents.ToClickHouse | [![NuGet version](https://badge.fury.io/nu/YY.DBTools.SQLServer.XEvents.ToClickHouse.svg)](https://badge.fury.io/nu/YY.DBTools.SQLServer.XEvents.ToClickHouse) | Пакет для экспорта расширенных событий SQL Server в базу ClickHouse |
| YY.DBTools.SQLServer.ExtendedEventsToClickHouse | [последний релиз](https://github.com/YPermitin/YY.DBTools/releases) | Консольное приложение для экспорта расширенных событий SQL Server в ClickHouse |

Последние новости об этой и других разработках, а также выходе других материалов, **[смотрите в Telegram-канале](https://t.me/DevQuietPlace)**.

### Состояние сборки
| Windows |  Linux |
|:-------:|:------:|
| - | ![.NET](https://github.com/YPermitin/YY.DBTools/workflows/.NET/badge.svg) |

## Приложение экспорта расширенных событий SQL Server в ClickHouse

**YY.DBTools.SQLServer.ExtendedEventsToClickHouse** - простое приложение для экспорта данных расширенных событий SQL Server в базу ClickHouse. 

Для использования нужно сформировать файл конфигурации:
```json
{
  "ConnectionStrings": {
    "XEventsDatabase": "Host=<АдресСервера>;Port=8123;Username=<Пользователь>;password=<Пароль>;Database=<ИмяБазы>;"
  },
  "XEvents": {
    "SourcePath": "C:\\Logs",
    "Portion": 10000,
    "DelayMs": 60000,
    "UseWatchMode": true
  }
}
```

Как видно, в файле нужно указать строку соединения с ClickHouse, а в секции настроек "XEvents" задать каталог с файлами расширенных событий (*.xel*), размер порции отправки данных в базу, задержку проверки новых данных в файле и режим "отслеживания".
Последние две настройки позволяют настроить приложение так, чтобы оно было запущено на постоянной основе и периодически проверяло новые данные в файлах логов.

Кроме этого, само приложение имеет ряз параметров запуска:

* **config** - путь к файлу конфигурации. Если не указан, то в каталоге приложения ищется файл с именем "appsettings.json".
* **logDirectoryPath** - путь к каталогу, куда приложение будет сохранять логи. По умолчанию создается в том же месте, где и приложение.
* **AllowInteractiveCommands** - включает возможность интерактивной работы с приложением. Например, остановить экспорт по нажатию CTRL + C. По умолчанию включен. В некоторых сценариях может понадобиться отключение.

## Библиотека чтения расширенных событий SQL Server

**YY.DBTools.SQLServer.XEvents** - библиотека для чтения расширенных событий SQL Server из файлов *.xel. Внутри себя использует официальное решение от Microsoft - **[Microsoft.SqlServer.XEvent.XELite](https://www.nuget.org/packages/Microsoft.SqlServer.XEvent.XELite/)**, расширяя ее в части обработки событий и некоторых других моментах.

Пример использования можно посмотреть в приложении в этом же репозитории. Вот краткий пример:

```csharp
using (var reader = new ExtendedEventsReader(eventPath,
    (sender, EventArgs) =>
    {
        // Событие при чтении каждого события
        // EventArgs.EventNumber - номер события
        // EventArgs.EventData - данные события
    },
    (sender, EventArgs) =>
    {
        // Событие при чтении метаданных файла
    },
    (sender, EventArgs) =>
    {
        // Событие перед чтением файла
        // EventArgs.Cancel - признак отмены чтения. Если установить в True, то файл будет пропущен
        // EventArgs.FileName - полный путь к файлу
    },
    (sender, EventArgs) =>
    {
        // Событие после чтением файла
        // EventArgs.FileName - полный путь к файлу
    },
    (sender, EventArgs) =>
    {
        // Событие при возникновении ошибок
        // EventArgs.Exception - информация об исключении
    }))
{
    await reader.StartReadEvents(CancellationToken.None);
}
```

Не обязательно передавать обработчики всех событий.

## Библиотека экспорта расширенный событий SQL Server

**YY.DBTools.SQLServer.ExtendedEventsToClickHouse** - библиотека экспорта расширенных событий SQL Server в ClickHouse.

Позволяет инициализировать базу данных ClickHouse универсальной структуры для хранения расширенных событий и выполнить их экспорт порциями.

Простой пример использования:
```csharp
using (IXEventExportMaster exporter = new XEventExportMaster(
    (e) =>
    {
        // Событие перед отправкой данных
        // e.Cancel - признак отмены отправки. Если установить в True, то данные не будут отправлены в базу.
        // e.Rows - список событий для отправки
    },
    (e) =>
    {
        // Событие после отправки данных
        // e.CurrentPosition - информация о позиции чтения данных в файле с логами
        // e.CurrentPosition.EventUUID - идентификатор события
        // e.CurrentPosition.CurrentFileData - путь к файлу данных
        // e.CurrentPosition.EventPeriod - период события
        // e.CurrentPosition.EventNumber - номер события
    },
    (e) =>
    {
        // Событие при возникновении ошибок
        // e.Exception - информация об исключении
    }))
{
    exporter.SetXEventsPath(_settings.XEventsPath);
    IXEventsOnTarget target = new ExtendedEventsOnClickHouse(
            ConnectionString, // Строка подключения к БД
            Portion // Размер порции выгрузки данных
        );
    exporter.SetTarget(target);
    await exporter.StartSendEventsToStorage(CancellationToken.None);
}
```

При первом подключении будет создана база данных ClickHouse для хранения событий XEvent любой структуры и с любым составом полей.

## Сценарии использования

При анализе больших объемов расширенных событий лучше их сохранять в какое-либо хранилище. В классическом варианте их сохраняют в базу SQL Server, но это не всегда удобно и возможно. 
Плюс ко всему, эта библиотека выполняет некоторые операции предобработки SQL-запросов, чтобы их было удобнее группировать (обработка имет временных таблицы, параметров и др.).

В итоге, обрабатывать расширенные события в базе ClickHouse - это быстро и эффективно.

## TODO

Планы в части разработки:

* Сделать Wiki с примерами использования библиотек и приложений
* Добавить возможность экспорта логов PostgreSQL
* Добавить возможность экспорта XEvents онлайн с помощью запросов к SQL Server
* Добавить онлайн получение данных логов для PostgreSQL
* В базы ClickHouse добавить специализированные представления для упрощения анализа данных
* Расширение unit-тестов библиотек и приложений

## Лицензия

MIT - делайте все, что посчитаете нужным. Никакой гарантии и никаких ограничений по использованию.
