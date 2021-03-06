Ресурсы
=========

RESTful API интерфейсы предназначены для доступа и управления *ресурсами*. Вы можете представлять ресурсы как
[модели](structure-models.md) в архитектуре [MVC](http://ru.wikipedia.org/wiki/Model-View-Controller).

Хотя в Yii нет никаких ограничений в том как представить ресурс, здесь вы можете представлять ресурсы
как объект наследующий свойства и методы [[yii\base\Model]] или дочерних классов (например [[yii\db\ActiveRecord]]), потому как:

* [[yii\base\Model]] реализует [[yii\base\Arrayable]] интерфейс, настраиваемый как вам удобно, для представления
данных через RESTful API интерфейс.
* [[yii\base\Model]] поддерживает [валидацию](input-validation.md), что полезно для RESTful API
  реализующего ввод данных.
* [[yii\db\ActiveRecord]] реализует мощный функционал для работы с БД, будет полезным если данные ресурса хранятся в поддерживаемых БД.

В этом разделе, мы опишем как использовать методы наследуемые вашим класом ресурсов от [[yii\base\Model]] (или дочерних классов) необходимые RESTful API.
> Если класс ресурса не наследуется от [[yii\base\Model]], то будут возвращены все public поля.


## Поля <a name="fields"></a>

Когда ресурс включается в ответ RESTful API, необходимо представить(сеарилизовать) ресурс как строку.
Yii разбивает процесс сеарилизации на два шага. На первом шаге, ресурс конвертируется в массив [[yii\rest\Serializer]].
На втором шаге, массив сеарилизуется в строку в требуемом формате (JSON, XML) при помощи
[[yii\web\ResponseFormatterInterface|интерфейса для форматирования ответа]]. Это сделано для того чтобы при разработке вы могли сосредоточится на разработке класса ресурсов.

При переопределении методов [[yii\base\Model::fields()|fields()]] и/или [[yii\base\Model::extraFields()|extraFields()]],
вы можете указать какие данные будут отображаться при представлении в виде массива.
Разница между этими двумя методами в том, что первый определяет стандартный набор полей которые будут включены в представлении массивом, а второй
определяет дополнительные поля, которые могут быть включены в массив если запрос пользователя к ресурсу использует дополнительные параметры.
Например:

```
// вернёт все поля объявленные в fields()
http://localhost/users

// вернёт только поля id и email, если они объявлены в методе fields()
http://localhost/users?fields=id,email

// вернёт все поля обявленные в fields() и поле profile если оно указано в extraFields()
http://localhost/users?expand=profile

// вернёт только id, email и profile, если они объявлены в fields() и extraFields()
http://localhost/users?fields=id,email&expand=profile
```


### Переопределение `fields()` <a name="overriding-fields"></a>

По умолчанию, [[yii\base\Model::fields()]] вернёт все атрибуты модели как поля, пока что
[[yii\db\ActiveRecord::fields()]] возвращает только те атрибуты которые были объявлены в схеме БД.

Вы можете переопределить `fields()` при этом добавить, удалить, переименовать или переобъявить поля. Возвращаемое значение `fields()`
должно быть массивом. Ключи массива это имена полей, и значения массива могут быть именами свойств/атрибутов или анонимных функций, возвращающих соответсвующее значение полей.
Если имя атрибута такое же, как ключ массива вы можете не заполнять значение. Например:
```php
// явное перечисление всех атрибутов, лучше всего использовать когда вы хотите убедиться что изменение
// таблицы БД или атрибутов модели не повлияет на изменение полей в представлении для API (для поддержки обратной совместимости с API).
public function fields()
{
    return [
        // название поля совпадает с названием атрибута
        'id',
        // ключ массива "email", соответсвует значению атрибута "email_address"
        'email' => 'email_address',
        // ключ массива "name", это PHP callback функция возвращающая значение
        'name' => function () {
            return $this->first_name . ' ' . $this->last_name;
        },
    ];
}

// Для фильтрации полей лучше всего использовать, поля наследуемые от родительского класса
// и blacklist для не безопасных полей.
public function fields()
{
    $fields = parent::fields();

    // удаляем не безопасные поля
    unset($fields['auth_key'], $fields['password_hash'], $fields['password_reset_token']);

    return $fields;
}
```

> Внимание: По умолчанию все атрибуты модели будут включены в представление для API, вы должны
> убедиться что не безопасные данные, не попадут в представление. Если в модели есть не безопасные поля,
> вы должны переопределить метод `fields()` для их фильтрации. В приведённом примере
> мы удаляем из представления `auth_key`, `password_hash` и `password_reset_token`.


### Переопределение `extraFields()` <a name="overriding-extra-fields"></a>

По умолчанию, [[yii\base\Model::extraFields()]] ничего не возвращает, а [[yii\db\ActiveRecord::extraFields()]]
возвращает названия отношений объявленных в ДБ.

Формат вовзращаемызх данных `extraFields()` такой же как `fields()`. Как правило, `extraFields()`
используется для указания полей, значения которых являются объектами. Например учитывая следующее объявление полей

```php
public function fields()
{
    return ['id', 'email'];
}

public function extraFields()
{
    return ['profile'];
}
```

запрос `http://localhost/users?fields=id,email&expand=profile` может возвращать следующие JSON данные:

```php
[
    {
        "id": 100,
        "email": "100@example.com",
        "profile": {
            "id": 100,
            "age": 30,
        }
    },
    ...
]
```


## Связи <a name="links"></a>

[HATEOAS](http://en.wikipedia.org/wiki/HATEOAS), аббревиатура для Hypermedia as the Engine of Application State,
необходим для того чтобы RESTful API, мог отобразить информацию которая позволяет клиентам просматривать возможности, поддерживаемые ресурсом. Ключ HATEOAS возвращает список ссылок с
информацией о параметрах доступных в методах API.
Ваши классы ресурсов могу поддерживать HATEOAS реализуя [[yii\web\Linkable]] интерфейс. Интерфейс
реализует один метод [[yii\web\Linkable::getLinks()|getLinks()]] который возвращает список [[yii\web\Link|links]].
Вы должны вернуть существующий URL на метод ресурса. Например:

```php
use yii\db\ActiveRecord;
use yii\web\Link;
use yii\web\Linkable;
use yii\helpers\Url;

class User extends ActiveRecord implements Linkable
{
    public function getLinks()
    {
        return [
            Link::REL_SELF => Url::to(['user/view', 'id' => $this->id], true),
        ];
    }
}
```

При отправке ответа объект `User` будет содержать поле `_links` содержащий ссылки связанные с объектом `User`.
Например:
```
{
    "id": 100,
    "email": "user@example.com",
    // ...
    "_links" => [
        "self": "https://example.com/users/100"
    ]
}
```


## Коллекции <a name="collections"></a>

Объекты ресурсов могут групироваться в *коллекции*. Каждая коллекция включает список объектов ресурсов одного типа.

Так как коллекции представляются в виде массива, их удобнее использовать как [проводник данных](output-data-providers.md).
Так как проводник данных поддерживает операции сортировки, разбиения на страницы это удобно использовать для RESTful API.
Например следующей метод возвращает проводник данных о почтовом ресурсе:

```php
namespace app\controllers;

use yii\rest\Controller;
use yii\data\ActiveDataProvider;
use app\models\Post;

class PostController extends Controller
{
    public function actionIndex()
    {
        return new ActiveDataProvider([
            'query' => Post::find(),
        ]);
    }
}
```

При отправке ответа RESTful API, [[yii\rest\Serializer]] добавит текущую страницу ресурсов и сеарилизует все объекты ресурсов.
Кроме того, [[yii\rest\Serializer]] добавит HTTP заголовки содержащие информацию о нумерации страниц:

* `X-Pagination-Total-Count`: Количество ресурсов;
* `X-Pagination-Page-Count`: Количество страниц ресурсов;
* `X-Pagination-Current-Page`: Текущая страница (начинается с 1);
* `X-Pagination-Per-Page`: Количество ресурсов отображаемых на 1 странице;
* `Link`: Набор ссылок позволяющий клиенту пройти все ресурсы, страница за страницей.

Примеры вы можете найти в разделе [Быстрый старт](rest-quick-start.md#trying-it-out).
