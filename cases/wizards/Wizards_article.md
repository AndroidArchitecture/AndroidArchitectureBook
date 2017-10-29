## Wizards в Чистой архитектуре

Подходов в построении приложений и экранов довольно много. <br>
Один из распространенных подходов в организации экранов - это Wizards. Wizards - это, по сути, набор довольно простых экранов, которые в совокупности решают какую-то общую задачу, при этом набор и порядок данных экранов может меняться. <br>

Типичный пример - это процесс регистрации пользователя. Пользователь может выбрать способ регистрации, например, сам ввести данные о себе, либо залогиниться через FB и еще что-то. Затем пользователь может выбрать более подходящий ему способ верификации: через почту, смс и т.д. В итоге получаем большое количество экранов, разветвленную логику переходов на экрана, и все это ради одной цели - регистрация пользователя.<br>
Второй пример - это подготовка платежки в банковском приложении. Вы выбираете тип платежа и заполняете поля. Обычно дизайнеры в этом случае разбивают заполнение платежки на несколько экранов для удобства, ну и плюс какие-то данные могут подтянуться и автозаполниться для одних пользователей, а вот другим придется заполнять. То есть мы снова получаем большое количество экранов, разветвленную логику переходов, и все это тоже ради одной цели - создание платежки.<br>
Если следовать классической Чистой архитектуре, то получается, что каждый экран (а точнее, скорее всего, Презентер) будет ответственен за переход на следующий и предыдущий шаг. Но тогда мы теряем переиспользуемость экранов, так как следующий и предыдущие шаги могут меняться, и каждый раз нам придется менять соответствующие Презентера. Кроме того логика принятия решения о следующем шаге размазывается между экранами. Да и протестировать данную логику довольно не просто будет.<br>
Как же лучше организовать визарды в Чистой архитектуре, чтобы мы могли достигнуть следующих целей:<br>

 - максимальная переиспользуемость экранов без внесения изменений в соответствующие классы (Презентеры, Интеракторы и т.д.),<br>
 - независимость экранов, то есть чтобы экраны не знали друг о друге,<br>
 - сосредоточнение логики переходов в одном месте,<br>
 - тестируемость данной логики.<br>

Что же, давайте начнем с [простого примера, ветка sample_1](https://github.com/AndroidArchitecture/WizardCase/tree/sample_1). Нам необходимо написать Визард регистрации пользователя. Состоять он будет из трех экранов:<br>

![image](https://habrastorage.org/web/e36/327/0b9/e363270b99114584b74d12d743341110.png)

На первом экране мы ознакамливаемся со стартовой информацией. Затем попадаем на экран принятия лицензии. И в конце вводим свои логин/пароль или же выбираем free-режим. Стрелками обозначены данные переходы. <br>
Отмечу, что с каждого экрана пользователь может вернуться на предыдущий, либо уже выйти с приложения. Вот и вся логика по нажатию на кнопку Back для упрощения. Поэтому данные действия я не буду отображать на схеме, но они будут действенны для абсолютно всех схем.<br>

Итак, начнем с простого. Каждый экран должен быть независим. И каждый экран представляет собой классический архитектурный слоеный пирог. Например, экран принятия лицензии внешне будет выглядеть вот так:<br>
![image](https://habrastorage.org/web/c87/48f/d70/c8748fd702d0438e96ec6b248de11f73.png)

А архитектурно вот так:

![image](https://habrastorage.org/web/891/5b2/9fa/8915b29fa7324352a277c83752d4d063.png)

И так для каждого экрана. <br>
Ок, теперь нам нужно как-то связать экраны с Визардами. Давайте рассуждать логически. Возьмем все тот же экран принятия лицензии. Что данный экран может сообщить чему-то внешнему, что отвечает за Визард. А по сути данный экран может сообщить только это:<br>

 - пользователь принял лицензию,<br>
 - пользователь вышел с данного экрана.<br>

Таким образом напрашивается интерфейс, через который наш экран будет общаться с Визардом:<br>
```java
    public interface LicenseWizardPart {
          void licenseWizardAccept();
          void licenseWizardBack();
    }
```
Данный интерфейс будет передаваться внешней зависимостью в Презентер, и из Презентера мы будем Визарду сообщать о соответствующих действиях:<br>
```java
@InjectViewState
public class LicensePresenter extends MvpPresenter<LicenseView> {

    private LicenseWizardPart licenseWizardPart;
    private LicenseInteractor licenseInteractor;

    private Disposable disposable;

    public LicensePresenter(LicenseWizardPart licenseWizardPart, LicenseInteractor licenseInteractor) {
        this.licenseWizardPart = licenseWizardPart;
        this.licenseInteractor = licenseInteractor;
    }

    public void acceptLicense() {
        if (disposable != null && !disposable.isDisposed()) {
            return;
        }
        disposable = licenseInteractor.acceptLicense()
                .doOnSubscribe(disposable -> getViewState().showProgress())
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe(aBoolean -> {
                    getViewState().hideProgress();
                    licenseWizardPart.licenseWizardAccept();
                }, throwable -> {});
    }

    public void clickBack() {
        licenseWizardPart.licenseWizardBack();
    }

    @Override
    public void onDestroy() {
        if (disposable != null) {
            disposable.dispose();
        }
        super.onDestroy();
    }

}
```

Таким образом мы выделили интерфейс, по которому данный экран будет общаться с Визардом. Причем он может общаться с абсолютно любым визардом, который поддерживает данный интерфейс. Получаем переиспользуемость экрана плюс гибкость в подключении к любому Визарду.<br>
Так мы для каждого экрана готовим подобный интерфейс:<br>
![image](https://habrastorage.org/web/2ca/fea/c88/2cafeac88eac4847b23ee654e7235a5d.png)

И теперь мы подходим к главному вопросу. Какая сущность должна связывать все эти экраны и каким-то образом управлять ими? <br>
В нашей Чистой архитектуре такой сущность, к сожалению, нет. <br>
Презентер? Но Презентер в нашем представлении отвечает за конкретную вьюшку, а не за несколько. Плюс Презентеру необходимо будет еще динамически как-то подключать новые вьюшки. Вообщем это не его работа.<br>
Интерактор? Интерактор отвечает за бизнес-логику, но ничего не знает о вьюшках, способах их расположения и т.д. Поэтому тоже нет.<br>
Роутер? Роутер отвечает на навигацию и только. Принятие решения о том, куда нам идти дальше, не его прерогатива.<br>

Поэтому введем новую сущность под названием **SmartRouter**:<br>
![image](https://habrastorage.org/webt/59/ce/84/59ce848923d9d439944582.png)
**SmartRouter** ответственен за навигацию между экранами, содержит некоторую логику, на основании которой принимает решение о следующем шаге, и содержит некоторые состояния (обычно одно состояние - текущий шаг).<br>
Давайте посмотрим, как выглядит `MainWizardSmartRouter` вживую:<br>
```java
public class MainWizardSmartRouter {

    private final Router router;
    private WizardStep currentWizardStep = NONE;

    private final InfoWizardPart infoWizardPart = new InfoWizardPart() {

        @Override
        public void infoWizardNext() {
            currentWizardStep = LICENSE;
            router.navigateTo(LICENSE_SCREEN);
        }

        @Override
        public void infoWizardBack() {
            router.finishChain();
        }

    };

    private final LicenseWizardPart licenseWizardPart = new LicenseWizardPart() {

        @Override
        public void licenseWizardAccept() {
            currentWizardStep = ACTIVATION;
            router.navigateTo(ACTIVATION_SCREEN);
        }

        @Override
        public void licenseWizardBack() {
            currentWizardStep = WizardStep.START_INFO;
            router.backTo(INFO_SCREEN);
        }

    };

    private final ActivationWizardPart activationWizardPart = new ActivationWizardPart() {

        @Override
        public void activationWizardFreeNext() {
            currentWizardStep = FINISH_INFO;
            router.finishChain();
        }

        @Override
        public void activationLoginWizardSuccess() {
            currentWizardStep = FINISH_INFO;
            router.finishChain();
        }

        @Override
        public void activationWizardBack() {
            currentWizardStep = WizardStep.LICENSE;
            router.backTo(LICENSE_SCREEN);
        }

    };

    public WizardSmartRouter(Router router) {
        this.router = router;
    }

    public void startWizard() {
        if (currentWizardStep != NONE) {
            return;
        }
        currentWizardStep = START_INFO;
        router.navigateTo(INFO_SCREEN);
    }

    public InfoWizardPart getInfoWizardPart() {
        return infoWizardPart;
    }

    public LicenseWizardPart getLicenseWizardPart() {
        return licenseWizardPart;
    }

    public ActivationWizardPart getActivationWizardPart() {
        return activationWizardPart;
    }

}
```

Вы можете заметить имплементации известных нам уже `InfoWizardPart`, `LicenseWizardPart` и `ActivationWizardPart`. `currentWizardStep`  - это состояние текущего шага Визарда. Ну а вопросы непосредственной навигации делегируются через также известный нам `Router`. Если количество используемых в Визарде экранов большое, то имплементации интерфейсов можно вынести в отдельные классы.<br>
Таким образом мы получаем единую сущность, ответственную за организацию Визарда, сущность, которая объединяет все, по сути, независимые экраны.<br>

Детали реализации вы можете посмотреть в [примере](https://github.com/AndroidArchitecture/WizardCase/tree/sample_1) и как там это все выглядит вместе с Dagger2 и Moxy. Обязательно просмотрите код, прежде чем продолжать читать дальше.<br>
Что еще хочу добавить к деталям реализации: визард реализован по канонам Андроида, то есть одна Активити для всего Визарда (`MainActivity`) с тремя в данном случае фрагментами (`InfoFragment`, `LicenseFragment`, `ActivationFragment`). Поэтому в `MainActivity` реализован `NavigatorHolder` для роутинга, и также стартуется Визард (вызов `wizardSmartRouter.startWizard()` в `onResume()`).<br>
С точки зрения Даггера мы имеем два модуля (`WizardModule` и `WizardNavigationModule`), соединенных вместе в `WizardComponent`. Вообщем-то и все особенности.<br>

Начнем усложнять нашу жизнь. Представим, что нам необходимо реализовать такой Визард:<br>
![image](https://habrastorage.org/web/c51/1a8/22c/c511a822c4e6459eb21cef2c1eee599c.png)

То есть в Визарде нам необходимо показать целых два информационных экрана, которые будут отличаться только выводимой информацией. Можно, конечно же, сделать просто копи-паст, но это не "true way". Плюс хочется переиспользуемости. Тогда в этом случае мы можем выделить абстрактную вьюшку, а конкретные реализации будут просто пробрасывать нужные Репозитории и интерфейсы взаимодействия с Визардом. <br>
Как это будет выглядеть с учетом Dagger2 и Moxy. <br>
Вот наша абстрактная вьюшка:<br>
```java
public abstract class InfoFragment extends MvpAppCompatFragment implements InfoView, BackButtonListener {

    private TextView infoText;

    @Nullable
    @Override
    public View onCreateView(LayoutInflater inflater, @Nullable ViewGroup container, @Nullable Bundle savedInstanceState) {
        View view = inflater.inflate(R.layout.fmt_info, container, false);
        //
        infoText = (TextView)view.findViewById(R.id.info_text);
        Button nextButton = (Button) view.findViewById(R.id.btn_next);
        nextButton.setOnClickListener(v -> getPresenter().clickNext());
        //
        return view;
    }

    @Override
    public void showText(TextType textType) {
        if (textType == TextType.START) {
            infoText.setText(getString(R.string.fmt_info_text_start));
        } else if (textType == TextType.FINISH) {
            infoText.setText(getString(R.string.fmt_info_text_finish));
        } else if (textType == TextType.LOGIN) {
            infoText.setText(getString(R.string.fmt_info_text_login));
        }

    }

    @Override
    public boolean onBackPressed() {
        getPresenter().clickBack();
        return true;
    }

    protected abstract InfoPresenter getPresenter();

}
```

А вот ее наследники:<br>
```java
public class InfoStartFragment extends InfoFragment {

    @Inject
    @Named(INFO_START_ANNOTATION)
    InfoWizardPart infoWizardPart;

    @ProvidePresenter
    InfoPresenter provideInfoPresenter() {
        return new InfoPresenter(infoWizardPart, TextType.START);
    }

    @InjectPresenter
    InfoPresenter infoPresenter;

    @Override
    public void onCreate(Bundle savedInstanceState) {
        ComponentManager.getInstance().getMainComponent().inject(this);
        super.onCreate(savedInstanceState);
    }

    @Override
    protected InfoPresenter getPresenter() {
        return infoPresenter;
    }

}

public class InfoFinishFragment extends InfoFragment {

    @Inject
    @Named(INFO_FINISH_ANNOTATION)
    InfoWizardPart infoWizardPart;

    @ProvidePresenter
    InfoPresenter provideInfoPresenter() {
        return new InfoPresenter(infoWizardPart, TextType.FINISH);
    }

    @InjectPresenter
    InfoPresenter infoPresenter;

    @Override
    public void onCreate(Bundle savedInstanceState) {
        ComponentManager.getInstance().getMainComponent().inject(this);
        super.onCreate(savedInstanceState);
    }

    @Override
    protected InfoPresenter getPresenter() {
        return infoPresenter;
    }

}
```

В итоге получаем максимально простую переиспользуемость одного и того же экрана в рамках одного Визарда. Более подробно код [здесь (ветка sample_2)](https://github.com/AndroidArchitecture/WizardCase/tree/sample_2).<br>

А теперь представим такой пример:<br>

![image](https://habrastorage.org/web/10e/68d/808/10e68d808c994bcdb3f8000724f36ee9.png)

Активация немного усложнилась =) Рассмотрим поподробнее, что происходит. <br>
Допустим, что сейчас мы на экране Активации (**Activation screen**):<br>

![image](https://habrastorage.org/web/fe6/abf/2d2/fe6abf2d29194ff492010495874ffa4f.png)

По нажатию на кнопку "To personal account" пользователь попадает на экраны логина/регистрации.<br>
Сначала нас ждет снова информационный экран (нижний в схеме **Information screen**):<br>

![image](https://habrastorage.org/web/091/fcc/6e4/091fcc6e49334173ba13dc218f5eb157.png)

Далее экран Логина (**Login screen**):<br>

![image](https://habrastorage.org/web/723/2ac/3c1/7232ac3c16f04f0991acbf2b122d56db.png)

На этом экране мы можем ввести свои креды и попасть на финальный экран Визарда, либо же зарегистрировать новый аккаунт (**Registration screen**):<br>

![image](https://habrastorage.org/web/962/6ea/7c8/9626ea7c860c4dc4a1aeddcd21dc2861.png)

И если регистрация проходит успешно, то мы также попадем на финальный экран Визарда.<br>
А теперь еще такая вводная: у нас есть несколько визардов, и в любом из них может понадобиться логин/регистрация. Получается, что во всех визардах нам придется вводить эти три экрана, логика взаимодействия которых между друг другом везде одинаковая. <br>
Разве никак нельзя как-то избежать этого безжалостного дублирования кода? На самом деле можно. Давайте еще раз взглянем на схему:<br>
![image](https://habrastorage.org/web/10e/68d/808/10e68d808c994bcdb3f8000724f36ee9.png)

На самом деле ее можно трансформитровать до такой схемы:<br>
![image](https://habrastorage.org/web/b72/6ee/ab7/b726eeab7080457bb6a0258cfc2dd8db.png)

То есть последовательность экранов InformationScreen, LoginScreen и RegistrationScreen и логику их взаимодействия мы выделяем в новый **AccountWizard**. Этот **AccountWizard** может сообщить внешнему Визарду, допустим, только две вещи:<br>
- пользователь вошел в нашу систему (залогинился или зарегистрировался, неважно),<br>
- пользователь не вошел в нашу систему (будем полагать, что просто вышел с данных экранов, ошибки логина/регистрации вовне не выносим).<br>

Таким образом получаем интерфейс:<br>
```java
public interface AccountWizardPart {
    void onSuccess();
    void onBack();
}
``` 
Теперь взглянем на получившийся ```AccountWizardSmartRouter``` :<br>
```java
public class AccountWizardSmartRouter {

    private final Router router;
    private final AccountWizardPart accountWizardPart;
    private AccountWizardStep accountWizardStep = NONE;

    private final InfoWizardPart infoWizardPart = new InfoWizardPart() {

        @Override
        public void infoWizardNext() {
            accountWizardStep = LOGIN;
            router.navigateTo(LOGIN_SCREEN);
        }

        @Override
        public void infoWizardBack() {
            router.finishChain();
        }

    };

    private final LoginWizardPart loginWizardPart = new LoginWizardPart() {

        @Override
        public void loginWizardSuccess() {
            router.finishChain();
            accountWizardPart.onSuccess();
        }

        @Override
        public void loginWizardBack() {
            router.finishChain();
            accountWizardPart.onBack();
        }

        @Override
        public void loginWizardNewAccount() {
            accountWizardStep = REGISTRATION;
            router.navigateTo(REGISTRATION_SCREEN);
        }

    };

    private final RegistrationWizardPart registrationWizardPart = new RegistrationWizardPart() {

        @Override
        public void registrationWizardSuccess() {
            router.finishChain();
            accountWizardPart.onSuccess();
        }

        @Override
        public void registrationWizardBack() {
            accountWizardStep = LOGIN;
            router.backTo(LOGIN_SCREEN);
        }

    };

    public AccountWizardSmartRouter(Router router,
                                    AccountWizardPart accountWizardPart) {
        this.router = router;
        this.accountWizardPart = accountWizardPart;
    }

    public void startWizard() {
        if (accountWizardStep != NONE) {
            return;
        }
        accountWizardStep = INFO;
        router.navigateTo(INFO_ACCOUNT_SCREEN);
    }

    public InfoWizardPart getInfoWizardPart() {
        return infoWizardPart;
    }

    public LoginWizardPart getLoginWizardPart() {
        return loginWizardPart;
    }

    public RegistrationWizardPart getRegistrationWizardPart() {
        return registrationWizardPart;
    }

}
```

Особо ничего нового за исключением того, что в необходимых местах дергается ```AccountWizardPart```.<br>
Данный визард с технической точки зрения реализован точно также, как мы до этого реализовывали ```MainWizardSmartRouter```. То есть это:<br>
- android: отдельная активити (```AccountActivity```) с тремя фрагментами (```InfoAccountFragment```, ```LicenseFragment```, ```RegistrationFragment```),<br>
- dagger2: два модуля (```AccountWizardModule```, ```AccountNavigationModule```), замыкающихся в одном компоненте (```AccountWizardComponent```) со своим скоупом (```AccountWizardScope```). ```AccountWizardComponent``` является сабкомпонентом от ```WizardComponent```.<br>

Все подробности вы можете самостоятельно просмотреть в [примере, ветка sample_3](https://github.com/AndroidArchitecture/WizardCase/tree/sample_3).<br>
А что же поменялось для ```MainWizardSmartRouter```? Давайте посмотрим на код:<br>
```java
public class MainWizardSmartRouter {

    private final Router router;
    private MainWizardStep currentMainWizardStep = MainWizardStep.NONE;

    private final InfoWizardPart infoStartWizardPart = new InfoWizardPart() {

        @Override
        public void infoWizardNext() {
            currentMainWizardStep = MainWizardStep.LICENSE;
            router.navigateTo(LICENSE_SCREEN);
        }

        @Override
        public void infoWizardBack() {
            router.finishChain();
        }

    };

    private final LicenseWizardPart licenseWizardPart = new LicenseWizardPart() {

        @Override
        public void licenseWizardAccept() {
            currentMainWizardStep = MainWizardStep.ACTIVATION;
            router.navigateTo(ACTIVATION_SCREEN);
        }

        @Override
        public void licenseWizardBack() {
            currentMainWizardStep = MainWizardStep.START_INFO;
            router.backTo(INFO_START_SCREEN);
        }

    };

    private final ActivationWizardPart activationWizardPart = new ActivationWizardPart() {

        @Override
        public void activationWizardPersonalAccountNext() {
            router.navigateTo(ACCOUNT_SCREEN);
        }

        @Override
        public void activationWizardFreeNext() {
            currentMainWizardStep = MainWizardStep.FINISH_INFO;
            router.navigateTo(INFO_FINISH_SCREEN);
        }

        @Override
        public void activationWizardBack() {
            currentMainWizardStep = MainWizardStep.LICENSE;
            router.backTo(LICENSE_SCREEN);
        }

    };

    private final AccountWizardPart accountWizardPart = new AccountWizardPart() {

        @Override
        public void onSuccess() {
            currentMainWizardStep = MainWizardStep.FINISH_INFO;
            router.navigateTo(INFO_FINISH_SCREEN);
        }

        @Override
        public void onBack() {
            currentMainWizardStep = MainWizardStep.ACTIVATION;
            router.backTo(ACTIVATION_SCREEN);
        }

    };

    private final InfoWizardPart infoFinishWizardPart = new InfoWizardPart() {

        @Override
        public void infoWizardNext() {
            router.finishChain();
        }

        @Override
        public void infoWizardBack() {
            router.finishChain();
        }

    };

    public MainWizardSmartRouter(Router router) {
        this.router = router;
    }

    public void startWizard() {
        if (currentMainWizardStep != MainWizardStep.NONE) {
            return;
        }
        currentMainWizardStep = MainWizardStep.START_INFO;
        router.navigateTo(INFO_START_SCREEN);
    }

    public InfoWizardPart getInfoStartWizardPart() {
        return infoStartWizardPart;
    }

    public LicenseWizardPart getLicenseWizardPart() {
        return licenseWizardPart;
    }

    public ActivationWizardPart getActivationWizardPart() {
        return activationWizardPart;
    }

    public InfoWizardPart getInfoFinishWizardPart() {
        return infoFinishWizardPart;
    }

    public AccountWizardPart getAccountWizardPart() {
        return accountWizardPart;
    }

}
```

По сути добавился еще один ```...wizardPart``` интерфейс и все. А то, что под этим ```AccountWizardPart``` скрывается отдельный Визард, ```MainWizardSmartRouter``` не знает. И в этом основная красота построения Визардов через **SmartRouter** и интерфейсы **...wizardPart**.<br>
В итоге подобное построение Визардов позволяет достигнуть обозначенных выше целей.
