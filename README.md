# Интеграция интерфейса личного кабинета PricePlan с личным кабинетом сервиса клиента.

Личный кабинет \(ЛК\) PricePlan может быть встроен в сервис клеиетна. В результате пользователи сервиса смогут пополнять баланс для физических и юридических лиц, управлять своими подписками \(создавать новые, изменять существующие\), работать  
с документами, следить за зачислениями и списаниями.

**Обратите внимание**: функция по умолчанию отключена, её нужно дополнительно включить в настройках биллинга \(Интеграции -&gt; Кабинет клиента\).

## <a name="scenario">Типовой сценарий</a>

Приступая к интеграции, необходимо чётко определить варианты взаимодействия  
между пользователями, порталом клиента и ЛК PricePlan. Типовой сценарий выглядит следующим образом:

1. Пользователь авторизуется на сайте клиента \(например, посредством логина и пароля, которые хранятся на стороне клиента\).

2. Броузер осуществляет переход на страницу ЛК клиента.

3. На данной странице скрипт pp\_widgets.js создаёт дочерний компонент HTML _iframe_ для указанного элемента.

4. Также этот скрипт заставляет броузер отправить запрос на ресурс, указанный в атрибуте **url\_auth**.

5. Данный ресурс должен вернуть редирект вида [https://&lt;субдомен\_клиента&gt;-lk.priceplan.pro/auth-key/&lt;\*\*priceplan\_auth\_key\*\*&gt;/](https://<субдомен_клиента>-lk.priceplan.pro/auth-key/<**priceplan_auth_key**>/)

6. После перехода по означенному URL происходит авторизация пользователя в ЛК PricePlan, и _iframe_ может быть заполнен соответствующими данными.

## Реализация

Для того, чтобы описанный сценарий заработал необходимо выполнить 3 условия.

### 1. Хранение ID пользователя PricePlan на стороне клиента

Как уже понятно, следует обеспечить сохранение числового идентификатора клиента PricePlan в его профиле в БД клиента.

### 2. Ресурс на сайте клиента для авторизации пользователя в ЛК PricePlan

Для того чтобы пользователь мог быть авторизован в ЛК PricePlan \(без логина и пароля PricePlan\), на сервисе клиента необходимо реализовать специальный  
ресурс. Этот ресурс должен получать у PricePlan **priceplan\_auth\_key** и отправлять его в качестве HTTP редиректа на адрес вида:

[https://&lt;субдомен\_клиента&gt;-lk.priceplan.pro/auth-key/&lt;\*\*priceplan\_auth\_key\*\*&gt;/](https://<субдомен_клиента>-lk.priceplan.pro/auth-key/<**priceplan_auth_key**>/)

#### Пример реализации на Python \(Django\) {#example}

```python
import json
import requests

from django.views.generic.base import RedirectView

LOGIN_TPL = 'http://<субдомен_клиента>-lk.priceplan.pro/api/login'
AUTH_KEY_TPL = \
    'https://<субдомен_клиента>-lk.priceplan.pro/api/clients/%s/auth-key/'
REDIRECT_TPL = 'https://<субдомен_клиента>-lk.priceplan.pro/auth-key/{key}/'

class PPAuthView(RedirectView):
    """
    PricePlan personal area authentication view.

    """
    payload_manager = {
        'user': '<manager_username>',
        'password': '<manager_password>'
    }

    def get_key(self, user_id):
        """Return PricePlan authentication key."""
        # Авторизация в качестве менеджера
        rsp_login = requests.post(LOGIN_TPL, data=json.dumps(payload_manager))
        # Получение авторизационных данных для пользователя
        rsp_auth_key = requests.post(AUTH_KEY_TPL % user_id,
                                     cookies=rsp_login.cookies)
        auth_data = json.loads(rsp_auth_key.content)
        priceplan_auth_key = auth_data['data']['key']
        return priceplan_auth_key

    def get_redirect_url(self, *args, **kwargs):
        """Make redirect URL."""
        user = ... # Тут нужно получить объект пользователя
        priceplan_auth_key = self.get_key(user_id=user.id)
        return REDIRECT_TPL.format(key=priceplan_auth_key)
```

### 3. Добавление скриптов в код страницы

Результирующий HTML код на странице клиента должен содержать примерно такой фрагмент:

```html
<script type="text/javascript"
  src="https://<субдомен_клиента>-lk.priceplan.pro/media/js/iframeResizer.js">
</script>
<script type="text/javascript"
  src="https://<субдомен_клиента>-lk.priceplan.pro/media/js/public_widgets/pp_widgets.js">
</script>

<script type="text/javascript">
  window.onload = function () {
    // Инициализация объекта виджета
    PPWidgets.init({
      https: true,
      customer: "<субдомен_клиента>",
      domain: "priceplan.pro",
      // Путь ресурса на сайте клиента, описанного в предыдущем пункте
      url_auth: "/pp-auth/",

      isReady: function () {
        // Событие вызывается когда ЛК полностью загрузится
        // console.log("isReady!");
        $('#preloaded').css('display', 'none');
        $('#pp_cabinet_iframe').attr('style', '');
        $('#loaded').attr('style', '');
      },

      afterLogout: function () {
        // Событие вызывается, когда пользователь совершит выход
        // из ЛК Priceplan
        // console.log("afterLogout!");
      }
    });

    // Вызов виджета и отображение его в элементе с id element_id
    PPWidgets.Cabinet("element_id", { width: '900px' });
    // Запуск resize iframe, который будет срабатывать при
    // изменение высоты виджета
    iFrameResize({});
  }
</script>
```



