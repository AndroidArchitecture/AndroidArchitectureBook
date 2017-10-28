## Аутентификация <br>
Думаю, что во многих приложениях, особенно в банковских, вы встречались со следующим поведением приложения: работаете, работаете, а потом вдруг появляется экран ввода пин-кода. Такое поведение обосновано мерами безопасности. Опишем данный процесс чуть более подробно с некоторыми техническими подробностями. <br>
При старте приложения вам показывается экран с вводом пинкода. Вы посылаете запрос на сервер, и в случае успеха (верного пинкода) вам возвращается токен, и стартует сессия. Далее часть запросов вы шлете с использованием данного токена. Токен прикрепляется либо к заголовку запроса, либо к телу. Подобные запросы еще также называются запросами авторизованной зоны. То есть сессия - это время, в течении которого данный токен действителен на стороне сервера.<br>
Через какое-то время сессия завершается, обычно по истечению какого-то конкретного промежутка времени, и тогда токен "протухает". Запросы с протухшим токеном возвращают ошибку. Тогда либо нужно просто обновить токен специальным запросом, либо нужно пользователя снова перебросить на экран ввода пин-кода, обновить токен, и продолжить слать запросы авторизованной зоны с обновившимся токеном. <br>
А теперь представим, что подобное поведение вам необходимо реализовать в своем приложении. И архитектура вашего приложения - Чистая архитектура. <br>
Поэтому возникает ряд интересных вопросов: <br>
1. На каком слое должна производиться подстановка токена в запросы?<br>
2. Кто должен отвечать за обновление токена? На каком уровне?<br>
3. Как обновлять токен, если для этого еще нужно попросить пользователя ввести пин-код?<br>
4. Кто должен отлавливать авторизованные ошибки и как?<br>
5. Ну и как это в целом выглядит в рамках Чистой архитектуры?<br>

Давайте для начала определим стек технологий:<br>
 1. Java<br>
 2. RxJava 2<br>
 3. Retrofit 2<br>

Думаю, подобным стеком пользуются многие, поэтому примеры будут на основании него. Но примеры нужны для понимания концепции. Как только станет понятна концепция, набор инструментов (RxJava и Retrofit) и язык - это все-таки дело вторичное и больше о деталях.<br>
Сразу скажу, примеры - это просто примеры, поэтому там нет обработок ошибок, отсутствия сети и вот этого всего.<br>

Рассмотрим [первый пример (ветка - sample_1)](https://github.com/AndroidArchitecture/AuthCase/tree/sample_1). Суть его заключается в том, что для обновления токена нам не нужно просить пользователя ввести пин-код, и при протухании токена возвращается 401 ошибка.<br>
Собственно, первый вопрос. А где же должна производиться подстановка токена. <br>
Вообще подстановка токена, добавление какой-то служебной информации в запросы и прочее не относятся ни к UI, ни к бизнес-логике. Это детали реализации вашей серверной части и не более. Собственно и находиться они должны в слое **Data**. <br>
Для хранения токена лучше использовать отдельную сущность -  **AuthHolder**. <br>
Поэтому примерная схема взаимодействия слоев будет такой:<br>
![enter image description here](https://i.imgur.com/vuOjbJA.png)

**AuthRepository** хочет осуществить запрос. Для этого он использует классы из **AuthNetwork**. **AuthNetwork** включает в себя все необходимые для осуществления запроса классы, т.е. в нашем случае это Retrofit со всеми вспомогательными классами (OkHttp, Interceptor, Authenticator). В [примере](https://github.com/AndroidArchitecture/AuthCase/tree/sample_1) все они объединены в **AuthNetworkModule**.<br>
Для осуществления запроса в авторизованную зону **AuthNetwork** берет с **AuthHolder** токен и прикрепляет его, скажем, к заголовку запроса через **Interceptor**.<br>
Давайте посмотрим на **Interceptor**:<br>
```java
public class MainInterceptor implements Interceptor {

    private AuthHolder authHolder;
    private static final String ACCESS_TOKEN_HEADER = "Access-Token";

    public MainInterceptor(AuthHolder authHolder) {
        this.authHolder = authHolder;
    }

    @Override
    public Response intercept(Chain chain) throws IOException {

        Request.Builder requestBuilder = chain.request().newBuilder();
        if (chain.request().header(ACCESS_TOKEN_HEADER) == null) {
            requestBuilder.addHeader(ACCESS_TOKEN_HEADER, authHolder.getToken());
        }

        return chain.proceed(requestBuilder.build());
    }

}
```
Допустим, что если токен протухает, то сервер возвращает нам 401 ошибку. В этом случае мы можем воспользоваться **Authenticator**, специальным механизмом Retrofit для обработки подобных ситуаций. И в **Authenticator** мы должны у **AuthHolder** запустить процесс обновления токена:<br>
```java
public class MainAuthenticator implements Authenticator {

    private static final String ACCESS_TOKEN_HEADER = "Access-Token";

    private AuthHolder authHolder;

    public MainAuthenticator(AuthHolder authHolder) {
        this.authHolder = authHolder;
    }

    @Nullable
    @Override
    public synchronized Request authenticate(Route route, Response response) throws IOException {
        String storedToken = authHolder.getToken();
        String requestToken = response.request().header(ACCESS_TOKEN_HEADER);

        Request.Builder requestBuilder = response.request().newBuilder();

        if (storedToken.equals(requestToken)) {
            authHolder.refresh();
        }

        return buildRequest(requestBuilder);
    }

    private Request buildRequest(Request.Builder requestBuilder) {
        return requestBuilder.header(ACCESS_TOKEN_HEADER, authHolder.getToken()).build();
    }

}
```
Благодаря добавлению ```synchronized``` к методу ```authenticate``` и блокирующему методу ```authHolder.refresh()``` мы добиваемся того, что при возникновении 401 ошибки мы первым делом  идем обновлять токен. Все остальные запросы, которые возвращают 401 ошибку, тоже ждут обновления токена, и только после этого идут повторные запросы с обновленным токеном. Очень удобно!<br>
Логичный вопрос. А как должен обновляться  **AuthHolder**, через что он организизует свой запрос обновления токена. Так как опять-таки это внутренняя кухня организации работы с сервером, то Repositories и другие слои про это не знают. Это внутреннее дело **Data**. <br>
Хорошо, а через что тогда лучше осуществлять запрос? Через **CommonNetwork**, который будет включать классы для осуществления запросов неавторизованной зоны (запросов, не требующих токена). В [примере](https://github.com/AndroidArchitecture/AuthCase/tree/sample_1) эти классы объединены в **CommonNetworkModule**.<br>
Обновленная схема приобретет вид:<br>
![enter image description here](https://i.imgur.com/xvFKOZK.png)

А теперь еще посмотрим на **AuthHolder**:<br>
```java
public class AuthHolder {

    private CommonApi commonApi;

    @NonNull
    private volatile String token = "StartToken";

    public AuthHolder(CommonApi commonApi) {
        this.commonApi = commonApi;
    }

    @NonNull
    public String getToken() {
        return token;
    }

    public void refresh() {
        updateToken().blockingGet();
    }

    private Single<String> updateToken() {
        return commonApi.updateToken()
                .singleOrError()
                .doOnSuccess(newToken -> token = newToken);
    }

}
```
На всякий случай напомню, что у **CommonNetwork** и **AuthNetwork** должны быть **разные** пулы потоков.<br>
А что делать, если сервер при подобной ошибке возвращает не 401 ошибку, а, допустим, успех, но в теле ответа содержится инфа, что нет авторизации. Это уже по сути особенности реализации, но вообще подобную ситуацию вы можете обработать с помощью **Interceptor**. Посмотрите реализацию **okhttp3.internal.http.RetryAndFollowUpInterceptor**, в котором, кстати говоря, и дергается наш **Authenticator**.<br>

Теперь рассмотрим более сложный случай. Когда токен протухает, то нам необходимо попросить пользователя ввести пин-код еще раз, и в случае правильного пин-кода обновить токен. <br>
Как можно увидеть, здесь уже задействуется UI. То есть будут задействоваться все слои. Таким образом нам нужно как-то отследить, что необходимо запросить пин-код и вызвать соответствующий экран. Отследить в принципе несложно. Добавляем листенер в **AuthHolder** и слушаем его в  **AuthRepositoryImpl**:<br>
```java
public class AuthRepositoryImpl implements AuthRepository {

    private AuthApi authApi;
    private AuthHolder authHolder;
    //...
    
    public AuthRepositoryImpl(AuthApi authApi, AuthHolder authHolder) {
        this.authApi = authApi;
        this.authHolder = authHolder;
        subscribeToAuthHolder();
    }

    private void subscribeToAuthHolder() {
        authHolder.subscribeToSessionExpired(() -> {
            // listen AuthHolder
        });
    }

    //...

}
```
Но данное событие нам нужно пробрасывать далее, так как Репозиторий не может принимать решения по бизнес-логики. Поэтому создаем **PinInteractor**, который будет теперь слушать  **AuthRepository**:<br>
```java
public class PinInteractorImpl implements PinInteractor {

    // Context or Router
    private Context context;
    private AuthRepository authRepository;

    public PinInteractorImpl(Context context, AuthRepository authRepository) {
        this.context = context;
        this.authRepository = authRepository;
        subscribeToAuthRepos();
    }

    private void subscribeToAuthRepos() {
        authRepository.subscribePinCodeNeedUpdate(() -> {
            // some logic
        });
    }

    //...

}
```
А вот уже Интерактор у нас уполномочен принимать решения. В данном случае решение заключается в том, чтобы показать экран ввода Пин-кода. Перед нами один из тех немногих случаев, когда Роутер находится в Интеракторе, а не в Презентере. Давайте взглянем на получающуюся у нас схему:<br>
![enter image description here](https://i.imgur.com/16uA8m7.png)

Отмечу, что время жизни **PinInteractor** также скорее всего равняется времени жизни приложения, как и у **AuthRepository** и **AuthHolder**.
Ну и метод ```subscribeToAuthRepos()``` у ```PinInteractorImpl``` чуть изменится:
```java
private void subscribeToAuthRepos() {
        authRepository.subscribePinCodeNeedUpdate(() -> {
            router.navigateTo(GlobalNavigator.PIN_SCREEN);
        });
    }
```
Весь код вы кстати можете найти в [примере, ветка - sample_2](https://github.com/AndroidArchitecture/AuthCase/tree/sample_2). Так как это пример, то некоторые вещи там упрощена, типа Presentation слоя, либо вечно живущего **PinPresenter**.<br>
Хорошо, как доставить событие о необходимости ввода пин-кода, и как это все организовать со слоями, мы разобрались. Собственно когда пользователь вводит пин-код, то мы его спускаем по слоям до **AuthHolder**. Схема немного преобразится:<br>
![enter image description here](https://i.imgur.com/4KkywiV.png)

Ну и чтобы допрояснить картину, вспомним, как выглядит наш **MainAuthenticator**:<br>
```java
public class MainAuthenticator implements Authenticator {

    private static final String ACCESS_TOKEN_HEADER = "Access-Token";

    private AuthHolder authHolder;

    public MainAuthenticator(AuthHolder authHolder) {
        this.authHolder = authHolder;
    }

    @Nullable
    @Override
    public synchronized Request authenticate(Route route, Response response) throws IOException {
        String storedToken = authHolder.getToken();
        String requestToken = response.request().header(ACCESS_TOKEN_HEADER);

        Request.Builder requestBuilder = response.request().newBuilder();

        if (storedToken.equals(requestToken)) {
            authHolder.refresh();
        }

        return buildRequest(requestBuilder);
    }

    private Request buildRequest(Request.Builder requestBuilder) {
        return requestBuilder.header(ACCESS_TOKEN_HEADER, authHolder.getToken()).build();
    }

}
```
Как только у одного запроса приходит 401 ошибка, то мы идем обновлять токен. Строчка:<br>
```java 
authHolder.refresh();
```
В это время остальные запросы, вернувшие 401 ошибку, ждут, так как мы вошли в блок ```synchronized``` у метода ```authenticate```.<br>
А вот **AuthHolder** выглядит интереснее:<br>
```java
public class AuthHolder {

    private CommonApi commonApi;
    private SessionListener sessionListener;
    private volatile CountDownLatch countDownLatch;

    @NonNull
    private volatile String token = "StartToken";
    @Nullable
    private volatile String pinCode;

    public AuthHolder(CommonApi commonApi) {
        this.commonApi = commonApi;
    }

    @NonNull
    public String getToken() {
        return token;
    }

    public void refresh() {
        pinCode = null;
        if (sessionListener != null) {
            sessionListener.sessionExpired();
        }
        countDownLatch = new CountDownLatch(1);
        try {
            countDownLatch.await();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        updateToken().blockingGet();
    }

    private Single<String> updateToken() {
        return commonApi.updateToken(new RequestTokenModel(pinCode))
                .singleOrError()
                .doOnSuccess(newToken -> token = newToken);
    }

    public void updatePinCode(@NonNull String pinCode) {
        this.pinCode = pinCode;
        countDownLatch.countDown();
    }

    public void subscribeToSessionExpired(SessionListener sessionListener) {
        this.sessionListener = sessionListener;
    }

}
```
Обратим внимание на метод ```public void refresh()```. В нем мы обнуляем пинкод, сообщаем подписчику об истечении сессии, в данном случае сообщаем **AuthRepository**. Затем через обычный ```CountDownLatch``` мы блокируем поток до тех пор пока не будет вызван метод ```public void updatePinCode(@NonNull String pinCode)```. В данном методе приходит введенный пользователем пин-код (**синие стрелочки** на рисунке выше). <br>
На всякий случай отмечу следующий момент. В данном случае обновление пин-кода - синхронная операция и выполняется она в главном потоке, поэтому с главного потока мы без проблем освобождаем поток, задержанный  ```CountDownLatch```. Но если обновление пин-кода была бы асинхронной операцией, и выполнялась бы в другом потоке, то поток этот должен был браться не из ```Executors``` в **AuthNetwork**, иначе теоретически могло получиться, что все потоки были бы в режиме ожидания, и обновить пин-код было бы некому.<br>
После того как пришел пин-код, поток, вызвавший ```refresh```, разблокируется и запускает запрос по обновлению токена (метод ```private Single<String> updateToken()```). В случае успешного обновления, остальные запросы с 401 ошибкой также перестартуются.<br>
Конечно же, механизм по обновлению токена с ожиданиями через ```CountDownLatch``` вы можете переделать по своему усмотрению. В этом решении также могут быть не учтены некоторые крайние случаи. Тут важна опять-таки - концепция.<br>
Вот собственно и все манипуляции =) 
