С точки зрения теории в интернетах гуляет очень много материала. Но есть источники, которые заслуживают отдельного упоминания, и которые мы рекомендуем прежде всего.

Первое, это серия статей от Five: <br>
[Android Architecture: Part 1 -  every new beginning is hard](http://five.agency/android-architecture-part-1-every-new-beginning-is-hard/) <br>
[Android Architecture: Part 2 - the clean architecture](http://five.agency/android-architecture-part-2-clean-architecture/)<br>
[Android Architecture: Part 3 - Applying clean architecture on Android](http://five.agency/android-architecture-part-3-applying-clean-architecture-android/)<br>
[Android Architecture: Part 4 - Applying Clean Architecture on Android, Hands on (source code included)](http://five.agency/android-architecture-part-4-applying-clean-architecture-on-android-hands-on/)<br>
[Android Architecture: Part 5 - How to Test Clean Architecture](https://five.agency/android-architecture-part-5-test-clean-architecture/)<br>

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
```
project
├─ di
│  ├─ app
│  ├─ payments
│  └─ operation
├─ presentation
│  ├─ view
│  │  ├─ payments
│  │  │  ├─ PaymentsView
│  │  │  └─ PaymentsFragment
│  │  └─ operations
│  │     ├─ OperationsView
│  │     └─ OperationsFragment
│  └─ presenter
│     ├─ payments
│     │  └─ PaymantsPresenter
│     └─ operations
│        └─ OperationsPresenter
├─ domain (он же business)
│  ├─ payments
│  │  ├─ PaymentsInteractor
│  │  ├─ PaymentsInteractorImpl
│  │  └─ CurrencyHandler (вспомогательный класс для PaymentsInteractor)
│  └─ operations
│     ├─ OperationsInteractor
│     ├─ rubs
│     │  ├─ OperationsInteractorRubs
│     │  └─ RubsManager (вспомогательный класс для OperationsInteractorRubs)
│     └─ currency
│        ├─ OperationsInteractorCurr
│        └─ CurrencyManager (вспомогательный класс для OperationsInteractorCurr)
├─ repositories
│  ├─ payments
│  │  ├─ PaymentsRepository
│  │  └─ PaymentsRepositoryImpl
│  └─ operations
│     ├─ OperationsRepository
│     └─ OperationsRepositoryImpl
├─ data
│  ├─ network
│  └─ db
└─ models (по сути хранилище всех dto)
   ├─ payments
   │  ├─ PaymentsModel
   └─ operations
      ├─ presentation
      │  └─ OperationUIModel
      ├─ domain
      │  ├─ OperationsRubModel
      │  └─ OperationCurrModel
      └─ data
         ├─ OperationsRubNetworkModel
         └─ OperationCurrNetworkModel
```

3. Есть несколько вариантов трактования понятия "Репозиторий". Подробно можно почитать, например, [здесь](http://hannesdorfmann.com/android/evolution-of-the-repository-pattern). В Андроид-мире "Репозиторий" - это абстракция для получения данных, то есть она скрывает, с какого именно источника получены те или иные данные. <br>
Кроме того Репозиторий может внутри себя реализовывать логику кэширования данных и соответственно выдачи либо закэшированных данных, либо данных с сети. 

4. Дядюшка Боб говорит, что *Interactor*- это объект, реализующий *Use Case*. Более того, предлагается создавать их с помощью паттерна [Команда](https://refactoring.guru/ru/design-patterns/command). У нас же сложилась тенденция объединять различные пользовательские сценарии, связанные с одним функционалом, в отдельные классы - Use case feature facade. Вдобавок многие использует в своих проектах RxJava, и мы получаем довольно лаконичный способ описания основного функционала. Были некоторые споры о том, должны ли такие классы называться в стиле "FeatureInteractor", или "FeatureInteractors" (во множественном числе), но больше прижился первый способ наименования.

5. Domain ничего не знает о других слоях. По классике к domain относятся Интеракторы и интерфейсы Репозиториев. Поэтому Интерактор не может являться маппером моделей. Мапперами моделей будут выступать Репозитории (из моделей data в модели domain и наоборот) и Презентеры (из моделей domain в модели View и наоборот).<br>
Когда мы проектируем фичу и разрабатываем ее по слоям, то в отношении моделей нужно исходить все-таки из принципа минимализма. Если вы знаете, что для вашей фичи вам достаточно будет просто сходить в сеть и полученный результат отобразить на экране и ничего более, и такое поведение вряд ли когда-нибудь изменится, то на все слои вашей фичи лучше держать одну модель.<br>
Чаще бывает, когда полученный результат с сервера нам нужно немного подкорректировать, и уже можно отображать на экране. Тогда на фичу будет две модели (data и domain). <br>
Ну и бывает, что по сути на каждый слой необходима своя модель (presentation, domain, data).

6. Activity, Service, BroadcastReceivers относятся ко внешнему кругу Чистой архитектуры, так как они являются частью платформы. Роли же у них могут быть разные: они могут быть и точками входа в приложение, и являться частью view, и работать в качестве источника данных. Соответственно, так как они относятся ко внешнему кругу, вы при необходимости можете внедрить туда и Интерактор, и Репозиторий, и другие классы из внутренних кругов.

7. В [видео 2016 года](https://www.youtube.com/watch?v=AlxMGxs2QnM&t=2509s&list=PLb1A91j1236pH1yoUvq5YDZUWAJz1T4DF&index=4) для Интерфейса Вьюшки много методов (setName, setAccountNumber, setCardNumber, setNearestDepartments). На самом деле все эти методы можно заменить на один типа setData, и в аргументы передавать какую-то специальную модельку.

9. Взаимодействие между вьюшкой и презентером и решение вопроса с ЖЦ лучше делегировать какой-то специальной библиотеке. Например, Moxy. Тогда вы избавитесь от большой части довольно нудной работы - обработка ЖЦ.

Кроме того про распространенные заблуждения с отсылом в первоисточники хорошо описано в [статье Василия](https://habrahabr.ru/company/mobileup/blog/335382/).

Вот кратко и вся теория, которую мы рекомендуем изучить в первую очередь. Остальные материалы на базе приобретенных вами знаний станут более понятными и логичными. У вас составится определенная картина, определенная база, от которой уже можно отталкиваться.


