>{.breadcrumbs} [Содержание](../readme.md) / [Основы управления памятью](./01-00-MemoryManagement-Intro.md)

# Время жизни сущностей

Один из вопросов, которые могут очень сильно влиять на производительность наших приложений - это политика выбора типа сущности: класс или структура, и контроль за их временем жизни. Ведь архитекторы платформы не просто так выделили для нас эти два типа данных. Это деление в первую очередь обусловлено возможностями оптимизации приложений, архитектура которых учитывает особенности обоих групп типов.

Как уже было сказано в главе [Ссылочные и значимые типы данных](../ReferenceTypesVsValueTypes.md), огромным преимуществом значимых типов данных является то, что их не надо аллоцировать. Т.е. другими словами если кто-то располагает экземпляр значимого типа в локальных переменных или параметрах метода, то расположение идёт на стеке (не выделяя дополнительной памяти в куче). Эта операция - та, о которой вам надо мечтать, т.к. именно она максимально быстра и эффективна. Если же структура располагается в полях ссылочного типа (класса), то под нее операция выделения памяти также не вызывается: ведь она является структурной частью этого ссылочного типа. Однако всё гораздо сложнее с ссылочными типами. Ведь, если речь идет о них, то мы имеем целый набор сложностей при выделении памяти под их экземпляры. Причём, что самое печальное, а может быть даже обидное - от нас почти никак не зависит, на какой из алгоритмов выделения памяти мы напоремся: на самый быстрый вариант из четырёх или же на самый тяжеловесный.

## Ссылочные типы

### Общий обзор

Для целостности картины при дальнейшем чтении рассмотрим особенности времени жизни экземпляров ссылочных типов данных. Ссылочные типы обладают следующими свойствами в вопросе времени собственной жизни:

  - У ссылочных типов в отличии от значимых детерменированное начало жизни. Другими словами они порождаются тогда и только тогда, когда кто-либо запросил их создание;
  - Однако, они имеют недетерменированный освобождение: мы не знаем, когда произойдет освобождение памяти из под них. Мы не можем вызвать GC для конкретного экземпляра даже для случая с Large Objects Heap, где эта операция могла бы быть вполне уместна;

Эти два свойства дают нам немного пищи для размышлений:

  - экземпляры классов уничтожаются в случайное время в неопределенно отдаленном будущем;
  - их уничтожение обуславливается утерей ссылок на них;
  - поэтому с одной стороны это значит, что операция освобождения последней ссылки на объект превращается в детерменированную операцию "удаления" объекта из зоны видимости приложения. Он ещё есть, существует, но недосягаем для всего остального приложения;
  - однако, с другой стороны мы далеко не всегда в курсе, какое именно обнуление ссылки будет последним, что лишает нас свойства детерменированности в обнулении последней ссылки.

Еще одним очень важным свойством является наличие виртуального метода финализации объекта. Этот метод вызывается во время срабатывания сборщика мусора: т.е. в неопределенном будущем. И необходим данный метод для одного: корректного закрытия неуправляемых ресурсов, которыми владеет объект в тех и только тех случаях, когда что-то пошло не так (например, было выброшено исключение) и программа более не сможет самостоятено это сделать (код, который отвечает за освобождение данных ресурсов никогда более не вызовется вследствие срабатывания исключительной ситуации). И, поскольку время вызова данного метода ровно как и освобождение памяти из под объекта от нас не зависят, его вызов также не является детерменированным. Мало того, он является асинхронным, т.к. осуществяется в отдельном потоке во время исполнения приложения. Это важно помнить, т.к. если, например, ваше приложение имеет логику повторной попытки работы с ресурсом и если произошла какая-то ошибка (например, `ThreadAbortException`), в результате которой ресурсы "повисли" в очереди на финализацию, то это значит, что вы не сможете открыть этот ресурс (например, файл), пока не отработает очередь на финализацию, в которой этот ресурс будет освобождён.

> Однако, там где есть неопределенность, программисту всегда хочется внести определенность и как результат, возник интерфейс `IDisposable`, речь о котором пойдет чуть позже, в следующей главе. Я сейчас могу сказать лишь одно: он реализуется если необходимо, чтобы внешний код мог самостоятельно отдать команду на освобождение ресурсов объекта. Т.е. детерменированно сообщить объекту, что он более не нужен.


### В защиту текущего подхода

Мы никогда не задумывались (а может только я?) над тем, что было бы, будь всё по-другому: если бы память освобождалась детерменированно. Текущий подход с автоматической памятью, когда мы не задумываемся, где выделять объекты и когда их освобождать нам не всегда нравится: ведь бывают случаи, когда готовых к освобождению объектов накапливается слишком много и их освобождение тормозит всё приложение. Однако, в защиту текущего подхода давайте немного отвлечемся на сценарии, о которых иногда начинаешь задумываться, мечтая сменить текущий набор алгоритмов:

  - если вместо того чтобы освобождать объекты по срабатыванию GC мы будем освобождать их с потерей последней ссылки, что произойдёт? Вот наш код присваивает некоторой переменной `null`. Тогда получается, что на каждом присвоении необходимо проверять, идёт ли присвоение `null` или какой-либо другой рабочей ссылки. Если да, то надо понять, последняя ли это была ссылка. Каждый раз считать входящие с кучи ссылки, перебирая все объекты SOH/LOH - дорого. Значит, надо чтобы каждый объект считал все входящие ссылки сам: инкрементируя и декрементируя счётчик на каждой операции. Это - дополнительное место + дополнительные действия. Плюс ко всему получается, что мы уже не можем сжимать кучу: после каждого присваивания это делать слишком дорого: подходит только метод `Sweep`. Как мы видим, уже на поверхности всплывает очень много проблем, не говоря уже о подробностях;
  - если ввести оператор `delete`, чтобы как в C++ освобождать объекты по требованию, дополнительно воскрешает деструктор, как средство детерменированного освобождения памяти: ведь если мы освобождаем объект оператором `delete`, необходимо таким же образом освобождать те объекты, которыми этот объект владеет. Значит, необходим метод, который будет вызываться при разрушении объекта: деструктор экземпляра типа. Это приведет к увеличению сложности разработки и удорожанию сопровождения программ: утечки будут постоянно. Плюс ко всему прочему возникнет путаница при освобождении памяти: мы лишаемся возможности освобождать её в последней точке использования. Т.е. теперь мы должны это делать в строго отведенном месте.
  - если вводить смешанный алгоритм: в целом чтобы работало как сейчас, но чтобы был оператор `delete`. Например, вы мне скажете, вам захочется освобождать массивы данных, которые были использованы под скачивание изображений ровно в определенный момент. Потому что если наше приложение качает изображения друг за другом и при этом они достаточно быстро становятся не нужны, то мы вхолостую выделяем кучу памяти, которая быстро копится и приводит к вызову GC. Это особенно актуально для мобильных приложений на Xamarin и элемента управления "виртуальный список", где при быстром скролле изображений они в больших количествах грузятся, а потом становятся ненужными. Если удалять их сразу, то не будет ситуации с большим GC, который испортит анимацию прокрутки. Однако, тут возникнут сложности для GC. При ручном освобождении памяти, последняя в свою очередь станет фрагментирована и может перестать вмещать в себя те массивы данных, которые вы запросите под следующие изображения. Как следствие - всё равно произойдет GC. Если блоки памяти с ручным управлением располагать в LOH, то ручное освобождение хорошо "ляжет" на его алгоритмы. Однако, всё также будет приводить к фрагментации и дальнейшему срабатыванию полного GC. Единственно верное решение - использовать пул массивов и Span/Memory для доступа к поддиапазону индексов. Но тогда зачем вводить `delete`?

Тогда получается, что текущее решение - прекрасно и надо просто научиться им правильно пользоваться. Этим мы чуть позже и займемся.

### Предварительные выводы

Из всего сказанного можно увидеть, что у любого объекта есть некоторое время его существования. Это может показаться тривиальной мыслью, которая лежит на поверхности, но не все так однозначно:

  - важно понимать, как, когда и при каких иных условиях *объект создается*. Ведь его создание занимает некоторое не всегда короткое время. И вы не можете заранее угадать, по какому алгоритму он будет создан: простым переносом указателя в случае наличия места в allocation context, вследтвие необходимости расширения или переноса allocation context, необходимости сжатия эфимерного сегмента или же необходимости создания нового эфимерного сегмента с его полным структурированием;
  - также стоит понимать, насколько "популярным" будет объект *во время его жизни* и как долго он будет существовать: какое количество иных объектов будет на него ссылаться и как долго. Этот фактор влияет как на сборку мусора, фрагментацию кучи, время создания других объектов и что самое интересное - на время обхода графа объектов в фазе маркировки достижимых объектов сборщиком мусора;
  - а также, что логично и очень важно: как объект будет *достигать* состояния освобождения (состояние выброшенности звучит грустно). Это значит, будет ли осуществляться детерменированное его разрушение или нет. Например, при помощи `IDisposable.Dispose`
  - и освобождаться - быть подхваченным Garbage Collector'ом с дальнейшей возможностью вызова финализатора.

Каждый из этих этапов определяет производительность приложения и логичность его архитектуры. Рассмотрим каждый этап в отдельности.

## Создание объекта

Все начинается с операции запроса памяти к подсистеме управления памятью:

```csharp
var x = new A();
```

Эта казалось бы самая простая операция платформы .NET кроет в себе огромный пласт работы, который будет описан в отдельной главе. 