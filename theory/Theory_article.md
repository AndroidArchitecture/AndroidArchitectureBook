С точки зрения теории в интернетах гуляет очень много материала. Но есть источники, которые заслуживают отдельного упоминания, и которые мы рекомендуем прежде всего.

Первое, это серия статей от Five: <br>
[Android Architecture: Part 1 -  every new beginning is hard](http://five.agency/android-architecture-part-1-every-new-beginning-is-hard/) <br>
[Android Architecture: Part 2 - the clean architecture](http://five.agency/android-architecture-part-2-clean-architecture/)<br>
[Android Architecture: Part 3 - Applying clean architecture on Android](http://five.agency/android-architecture-part-3-applying-clean-architecture-android/)<br>
[Android Architecture: Part 4 - Applying Clean Architecture on Android, Hands on (source code included)](http://five.agency/android-architecture-part-4-applying-clean-architecture-on-android-hands-on/)<br>
[Android Architecture: Part 5 - How to Test Clean Architecture](http://five.agency/android-architecture-part-5-how-to-test-clean-architecture/)<br>

Очень грамотное теоретическое описание трансформации классической Clean Architecture от дядюшки Боба в Clean Architecture для Android. 
Единственное, вторая статья как-то немного запутала с этими input/output портами для UseCase и решением проблемы флоу данных через наследование и композицию. Немного оторвано от практики, как нам показалось. Лучше сразу к третьей части приступить, если вы также не совсем поняли.<br>
Еще есть моменты, над которыми можно похоливарить.

Второе, это [видео 2016 года](https://www.youtube.com/watch?v=AlxMGxs2QnM&t=2509s&list=PLb1A91j1236pH1yoUvq5YDZUWAJz1T4DF&index=4) и [2017 года](https://www.youtube.com/watch?v=pmlGgIOlz9w&t=15114s) о Чистой архитектуре от Евгения Мацюка и Александра Блинова.
Однако, с течением времени некоторые вещи, упомянутые в видео, уточняются и переосмысливаются. И в этом нужно сказать спасибо всем участникам архитектурного чатика. Поэтому мы настоятельно советуем пересмотреть доклады и прочитать дополнения.<br>

Итак дополнения:

 1. Проектировать фичу лучше начинать сверху вниз, а не наоборот, так как вы прежде всего отталкиваетесь от того, что видит конечный пользователь.
 
2. Общая структура. По канонам интерфейсы Репозитория относятся к бизнес-логике, а конкретные имплементации - к data. Мы для упрощения решили интерфейсы и имплементации Репозиториев вынести в отдельный слой (про это в 2017 рассказываем). <br>
Кроме того и структура пакетов меняется соответствующим образом. Плюс еще немного поменялось представление о том, где же должны быть модели данных.<br>
Модели данных обычно представляют собой простые POJO, то есть только данные без какой-либо логики. Для упрощения все подобные модели лучше размещать в специальном пакете под названием "models" (можно подобрать более удачное название, чтобы не так созвучно было с понятием "model").<br>

Структура пакетов:

-di<br>
---app<br>
---payments<br>
---operation<br>
-presentation<br>
---view<br>
-----payments<br>
-------PaymentsView<br>
-------PaymentsFragment<br>
-----operations<br>
-------OperationsView<br>
-------OperationsFragment<br>
---presenter<br>
-----payments<br>
-------PaymantsPresenter<br>
-----operations<br>
-------OperationsPresenter<br>
-domain (он же business)<br>
---payments<br>
-----PaymentsInteractor<br>
-----PaymentsInteractorImpl<br>
-----CurrencyHandler (вспомогательный класс для PaymentsInteractor)<br>
---operations<br>
-----OperationsInteractor<br>
-------rubs<br>
---------OperationsInteractorRubs<br>
---------RubsManager (вспомогательный класс для OperationsInteractorRubs)<br>
-------currency<br>
---------OperationsInteractorCurr<br>
---------CurrencyManager (вспомогательный класс для OperationsInteractorCurr)<br>
-repositories<br>
---payments<br>
-----PaymentsRepository<br>
-----PaymentsRepositoryImpl<br>
---operations<br>
-----OperationsRepository<br>
-----OperationsRepositoryImpl<br>
-data<br>
---network<br>
---db<br>
-models (по сути хранилище всех dto)<br>
---payments<br>
-----PaymentsModel<br>
---operations<br>
-----presentation<br>
-------OperationUIModel<br>
-----domain<br>
-------OperationsRubModel<br>
-------OperationCurrModel<br>
-----data<br>
-------OperationsRubNetworkModel<br>
-------OperationCurrNetworkModel<br>
 
3. Есть несколько вариантов трактования понятия "Репозиторий". Подробно можно почитать, например, [здесь](http://hannesdorfmann.com/android/evolution-of-the-repository-pattern). В Андроид-мире "Репозиторий" - это абстракция для получения данных, то есть она скрывает, с какого именно источника получены те или иные данные. <br>
Кроме того Репозиторий может внутри себя реализовывать логику кеширования данных и соответственно выдачи либо закешированных данных, либо данных с сети. 

4. Дядюшка Боб говорит, что *Interactor*- это объект, реализующий *Use Case*. Более того, предлагается создавать их с помощью паттерна [Команда](https://refactoring.guru/ru/design-patterns/command). У нас же сложилась тенденция объединять различные пользовательские сценарии, связанные с одним функционалом, в отдельные классы - Use case feature facade. Вдобавок многие использует в своих проектах RxJava, и мы получаем довольно лаконичный способ описания основного функционала. Были некоторые споры о том, должны ли такие классы называться в стиле "FeatureInteractor", или "FeatureInteractors" (во множественном числе), но больше прижился первый способ наименования.

5. Domain ничего не знает о других слоях. По классике к domain относятся Интеракторы и интерфейсы Репозиториев. Поэтому Интерактор не может являться маппером моделей. Мапперами моделей будут выступать Репозитории (из моделей data в модели domain и наоборот) и Презентеры (из моделей domain в модели View и наоборот).<br>
Когда мы проектируем фичу и разрабатываем ее по слоям, то в отношении моделей нужно исходить все-таки из принципа минимализма. Если вы знаете, что для вашей фичи вам достаточно будет просто сходить в сеть и полученный результат отобразить на экране и ничего более, и такое поведение вряд ли когда-нибудь изменится, то на все слои вашей фичи лучше держать одну модель.<br>
Чаще бывает, когда полученный результат с сервера нам нужно немного подкорректировать, и уже можно отображать на экране. Тогда на фичу будет две модели (data и domain). <br>
Ну и бывает, что по сути на каждый слой необходима своя модель (presentation, domain, data).

6. Activity, Service, BroadcastReceivers - это точки входа в приложение, это служебные классы. Поэтому относить их к какому-либо конкретному слою в Архитектуре неверно. В тот же Service вы при необходимости можете и Интерактор инжектить, и Репозиторий.

7. В [видео 2016 года](https://www.youtube.com/watch?v=AlxMGxs2QnM&t=2509s&list=PLb1A91j1236pH1yoUvq5YDZUWAJz1T4DF&index=4) для Интерфейса Вьюшки много методов (setName, setAccountNumber, setCardNumber, setNearestDepartments). На самом деле все эти методы можно заменить на один типа setData, и в аргументы передавать какую-то специальную модельку.

9. Взаимодействие между вьюшкой и презентером и решение вопроса с ЖЦ лучше делегировать какой-то специальной библиотеке. Например, Moxy. Тогда вы избавитесь от большой части довольно нудной работы - обработка ЖЦ.

Кроме того про распространенные заблуждения с отсылом в первоисточники хорошо описано в [статье Василия](https://habrahabr.ru/company/mobileup/blog/335382/).

Вот кратко и вся теория, которую мы рекомендуем изучить в первую очередь. Остальные материалы на базе приобретенных вами знаний станут более понятными и логичными. У вас составится определенная картина, определенная база, от которой уже можно отталкиваться.


