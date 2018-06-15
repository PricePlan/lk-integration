# Интеграция интерфейса личного кабинета PricePlan с личным кабинетом сервиса клиента.

Личный кабинет (ЛК) PricePlan может быть встроен в сервис провайдера. В
результате клиенты смогут пополнять баланс для физических и юридических лиц, управлять своими подписками (создавать новые, изменять существующие), работать
с документами, следить за зачислениями и списаниями.

**Обратите внимание**: функция по умолчанию отключена, её нужно дополнительно включить в настройках биллинга (Интеграции -> Кабинет клиента).

## Типовой сценарий

Приступая к интеграции, необходимо чётко определить варианты взаимодействия
между пользователями, порталом провайдера и ЛК PricePlan. Типовой сценарий выглядит следующим образом:

1. Пользователь авторизуется на сайте провайдера (например, посредством логина и пароля, которые хранятся на стороне провайдера).

1. Броузер осуществляет переход на страницу ЛК провайдера.

1. На данной странице скрипт pp_widgets.js создаёт дочерний компонент HTML *iframe* для указанного элемента.

1. Также этот скрипт заставляет броузер отправить запрос на ресурс, указанный в атрибуте **url_auth**.

1. Данный ресурс должен вернуть редирект вида https://<субдомен_провайдера>-lk.priceplan.pro/auth-key/<**priceplan_auth_key**>/

1. После перехода по означенному URL происходит авторизация пользователя в ЛК PricePlan, и *iframe* может быть заполнен соответствующими данными.

## Реализация

Для того, чтобы описанный сценарий заработал необходимо выполнить 3 условия.

### 1. Хранение ID пользователя PricePlan на стороне провайдера

Как уже понятно, следует обеспечить сохранение числового идентификатора клиента PricePlan в его профиле в БД провайдера.

### 2. Ресурс на сайте провайдера для авторизации пользователя в ЛК PricePlan

Для того чтобы пользователь мог быть авторизован в ЛК PricePlan (без логина и пароля PricePlan), на сервисе провайдера необходимо реализовать специальный
ресурс. Этот ресурс должен получать у PricePlan **priceplan_auth_key** и отправлять его в качестве HTTP редиректа на адрес вида:

https://<субдомен_провайдера>-lk.priceplan.pro/auth-key/<**priceplan_auth_key**>/

#### Пример реализации на Python (Django)

```python
import json
import requests

from django.views.generic.base import RedirectView

LOGIN_TPL = 'http://<субдомен_провайдера>-lk.priceplan.pro/api/login'
AUTH_KEY_TPL = \
    'https://<субдомен_провайдера>-lk.priceplan.pro/api/clients/%s/auth-key/'
REDIRECT_TPL = 'https://<субдомен_провайдера>-lk.priceplan.pro/auth-key/{key}/'

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

Результирующий HTML код на странице провайдера должен содержать примерно такой фрагмент:

```html
<script type="text/javascript"
  src="https://<субдомен_провайдера>-lk.priceplan.pro/media/js/iframeResizer.js">
</script>
<script type="text/javascript"
  src="https://<субдомен_провайдера>-lk.priceplan.pro/media/js/public_widgets/pp_widgets.js">
</script>

<script type="text/javascript">
  window.onload = function () {
    // Инициализация объекта виджета
    PPWidgets.init({
      https: true,
      customer: "<субдомен_провайдера>",
      domain: "priceplan.pro",
      // Путь ресурса на сайте провайдера, описанного в предыдущем пункте
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
