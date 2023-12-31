# [Django, HTMX and Alpine](https://danjacob.net/posts/djangohtmxalpine/)

24 января 2022 [DJANGO](https://danjacob.net/tags/django/) / [JAVASCRIPT](https://danjacob.net/tags/javascript/) / [HTMX](https://danjacob.net/tags/htmx/) / [ALPINEJS](https://danjacob.net/tags/alpinejs/) / [PYTHON](https://danjacob.net/tags/python/) / [WEB DEVELOPMENT](https://danjacob.net/tags/web-development/)

В течение последнего года я внимательно следил за интересной тенденцией в веб-разработке, когда волна новых библиотек и фреймворков позволяет разработчикам создавать современные веб-приложения более просто, дешево и быстро.

В настоящее время доминирующей парадигмой в веб-разработке является архитектура одностраничных приложений (Single Page Application или SPA). Как правило, она состоит из бэкэнд-интерфейса или API, подключенных к фронтенду, построенному на Javascript-фреймворке, таком как React, Vue или Svelte. API взаимодействует с фронтендом через полезную нагрузку в формате JSON, а фронтенд отвечает только за отображение данных в DOM.

Такая модель имеет ряд преимуществ. Она позволяет разделить проблемы: если мы хотим обслуживать не только веб-страницы, но и, скажем, приложение для Android, то оно может обращаться к тому же API, что и наше веб-приложение. Логика бэкенда может быть полностью изменена - в один прекрасный день вы можете решить перейти с Django на Rails, например, и до тех пор, пока ваш API остается неизменным, фронтенд не должен знать об этом или заботиться (то же самое, конечно, относится и к фронтенду). Это позволяет разработчикам из разных команд и дисциплин - бэкенд- и фронтенд-специалистам - работать независимо друг от друга. Кроме того, фронтенд-фреймворк, подобный React, может - по крайней мере, теоретически - обеспечить более плавное взаимодействие с конечным пользователем, без полной загрузки страницы и медленной обработки форм. Легко понять, почему модель SPA стала де-факто паттерном веб-разработки в последнее десятилетие.

Однако модель SPA имеет и ряд недостатков. Логика, например, проверка форм, должна дублироваться на клиенте и сервере. Возможно, придется размещать два отдельных приложения в разных доменах, что усложняет такие "решенные" проблемы, как аутентификация. Ошибка в Javascript-коде может привести не просто к полуфункциональному сайту, а к пустой странице без подсказки разработчику или пользователю, как ее исправить. SEO становится более проблематичным, поскольку сайты с Javascript-рендерингом непрозрачны (или, по крайней мере, неоптимальны) для поисковых систем и сервисов обмена информацией в социальных сетях. Изначально веб-страница может загружаться быстрее, но пользователю остается смотреть на многочисленные крутящиеся gif-изображения и мигающие каркасы, пока полдюжины вызовов API загружают отдельные части страницы. Существуют решения этих проблем, от SSR до GraphQL и [CORS](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS), но они влекут за собой еще большее усложнение и увеличение количества зависимостей.

Все эти дополнительные сложности приводят к увеличению затрат времени и денег на вывод продукта на рынок или добавление новых функций, а также к увеличению затрат на заработную плату или оплату услуг консультантов, поскольку приходится нанимать большие команды специалистов (а управление большими, многопрофильными командами влечет за собой дополнительные расходы на управление и коммуникации). Это также создает большую площадь для ошибок и потенциальных проблем с безопасностью (огромное количество модулей NPM, необходимых для работы среднего SPA-фронтенда, означает, что рано или поздно вы случайно [импортируете небезопасный или даже вредоносный код](https://arstechnica.com/information-technology/2021/09/npm-package-with-3-million-weekly-downloads-had-a-severe-vulnerability/), независимо от того, сколько аудита вы проводите). Более того, эти зависимости приводят к огромным начальным нагрузкам, исчисляемым мегабайтами даже после сжатия активов, что, возможно, не так важно для разработчиков, работающих на дорогих стационарных компьютерах в городах с отличным высокоскоростным Интернетом, но не так интересно для тех, кто использует старые мобильные устройства в сельской местности с плохим соединением. Конечно, достаточно квалифицированная команда может решить все эти проблемы, но [большинство команд не обладают достаточными навыками](https://www.baldurbjarnason.com/2021/single-page-app-morality-play) (или временем и ресурсами) для создания SPA такого высокого качества.

Однако следует признать, что SPA, независимо от их стоимости и недостатков, стали популярными потому, что они решили реальные проблемы старой модели рендеринга на стороне сервера, например, полную перезагрузку страницы при каждом нажатии на кнопку или ссылку или утомительный цикл проверки формы после валидации и репоста, и предоставили более эффективные средства для создания более сложных и интерактивных приложений.

Однако в то время как популярность SPA-фреймворков росла в течение последнего десятилетия, другие разработчики начали работать над альтернативным подходом к созданию сложных веб-приложений.

Для смягчения проблем, присущих традиционной серверно-рендерной архитектуре, распространенным решением до наступления эры фронтендов было использование AJAX для получения отдельных фрагментов HTML с сервера и вставки их в DOM, вместо того чтобы выполнять полную загрузку страницы при каждом запросе. Очень часто для этого использовался [jQuery.load()](https://api.jquery.com/load/), учитывая популярность jQuery в этот период. Это хорошо подходило для отдельных случаев, но становилось сложным для разработки и поддержки в больших и более сложных приложениях. Более продвинутые фреймворки, такие как Angular и React, предоставляли больше структуры и оптимизаций для обработки сложных манипуляций с DOM в больших проектах с более "тяжёлым" фронтендом.

Этот паттерн, который иногда пренебрежительно называют FAJAX, т.е. "фальшивый AJAX", не исчез полностью во время бума Javascript-фреймворков 2010-х годов. Появилось множество библиотек, предоставляющих более простые и структурированные средства для работы с AJAX и взаимодействием с DOM, в том числе:

- [PJAX](https://github.com/defunkt/jquery-pjax)
- [Turbolinks](https://github.com/turbolinks/turbolinks)
- [Intercooler](https://github.com/bigskysoftware/intercooler-js)
- [Unpoly](https://unpoly.com/)
- [Laravel Livewire](https://laravel-livewire.com/)
- [Phoenix Channels](https://www.phoenixframework.org/)

Turbolinks был включен в проект [Hotwire](https://hotwire.dev/), разрабатываемый командой Rails. В 2021 году был выпущен преемник Intercooler, также разработанный Карсоном Гроссом: [HTMX](https://htmx.org/).

# HTMX

HTMX позволяет постепенно добавлять функциональность AJAX в традиционное веб-приложение с серверным рендерингом с помощью набора пользовательских HTML-атрибутов. Например, предположим, что на сайте имеется кнопка "Подписаться", позволяющая следить за обновлениями другого пользователя или канала контента. Когда эта кнопка нажимается, пользователь подписывается на канал, и текст кнопки должен быть обновлен до "Отписаться".

Традиционное серверно-рендерное приложение может выглядеть следующим образом:

```html
<form method="POST" action="/subscribe/12345/">
    <button type="submit">Subscribe</button>
</form>
```

При нажатии на кнопку наша форма отправляется на сервер в виде действия POST. Серверный код, скорее всего, должен будет ответить HTTP-переадресацией на ту же страницу, что приведет к полной перезагрузке страницы. Кнопка сама по себе не может выполнить действие POST, поэтому ее нужно обернуть в тег `<form>`.

Полная перезагрузка страницы не только мешает пользователю, но и расточительно расходует ресурсы сервера: например, если вам нужно отобразить панель навигации, нижний колонтитул, информацию о странице и т.д., что потребует дополнительных обращений к базе данных? Хорошо для начальной загрузки страницы, но делать все это только для того, чтобы обновить текст кнопки, - значит тратить много ресурсов впустую.

HTMX позволяет нам справиться с этой проблемой более изящным способом. Используя пользовательские атрибуты с префиксом `hx-*` (приверженцы HTML могут также использовать `data-hx-`), кнопка сама может обрабатывать POST:

```html
<button hx-post="/subscribe/12345/"
        hx-target="this"
        hx-swap="outerHTML">Subscribe</button>
```

Атрибут `hx-post` означает "отправить HTTP POST на этот URL". `hx-target` указывает DOM, который будет заменен (это может быть любой допустимый DOM-селектор, например, ID или класс; это специальное обозначение, означающее "этот элемент"), а `hx-swap` - фактические манипуляции с DOM, которые должны быть выполнены в результате - в данном случае замена всей `<button>` на любой HTML, возвращенный с URL. Только эти три атрибута могут многое сделать; в инструментарии HTMX имеется [несколько десятков подобных директив](https://htmx.org/reference/), обеспечивающих всевозможную функциональность, связанную с AJAX, без написания ни одной строки Javascript.

Конечная точка URL - неважно, на каком языке, HTMX работает с любым серверным языком, от Python или PHP до Go или Rust - должна возвращать фрагмент HTML, примерно такой:

```html
<button hx-post="/unsubscribe/12345/"
        hx-target="this"
        hx-swap="outerHTML">Unsubscribe</button>
```

Как видно, в ответе текст "Подписка" меняется на "Отписка", а HTTP POST-урл - на соответствующее действие "Отписка". Кроме выполнения аутентификации и обновления базы данных, необходимых для создания и удаления подписок, нам нужно вернуть только небольшой фрагмент HTML, а не целую веб-страницу. Затем HTMX вставит этот новый `<button>` в DOM в соответствии с указаниями.

Сам HTMX представляет собой небольшую зависимость от Javascript. Самый простой способ начать работу - просто использовать CDN:

```html
<script src="https://unpkg.com/htmx.org@1.6.1"
        integrity="sha384-tvG/2mnCFmGQzYC1Oh3qxQ7CkQ9kMzYjWZSNtrRZygHPDDqottzEJsqS4oUVodhW"
        crossorigin="anonymous"></script>
```

В дополнение к небольшим AJAX-взаимодействиям можно использовать HTMX для полностраничной навигации, используя функцию `hx-boost`. Это работает аналогично Turbolinks/Hotwire: вы возвращаете с сервера полные страницы, но различные элементы DOM в `<body>` меняются местами, а не выполняют полную перезагрузку. Это обеспечивает более плавную навигацию пользователя по сайту, аналогичную использованию SPA. В HTMX также реализована поддержка технологий "push", таких как [Server-Sent Events](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events) (SSE) и WebSockets.

# HTMX и Django

Интеграция HTMX с Django достаточно проста и не требует ничего, кроме включения CDN в базовый шаблон. Следует иметь в виду, что HTMX предоставляет ряд [заголовков запросов](https://htmx.org/reference/#request_headers), что позволяет обеспечить более эффективные и целевые ответы.

Например, в представлении вы можете вернуть фрагмент HTML, если знаете, что запрос исходит от действия HTMX, или ответ на всю страницу в других случаях:

```py
def my_view(request):
    if request.headers.get("HX-Request"):
        return TemplateResponse(request, "_some_snippet.html")
    else:
        return TemplateResponse(request, "full_page.html")
```

Заголовок `HX-Request` автоматически добавляется ко всем HTMX-запросам, поэтому можно проверить, что запрос исходит от HTMX-действия.

Вы также можете добавить специфические для HTMX заголовки в свой исходящий ответ. Например, вы можете вернуть заголовок `HX-Redirect`, чтобы проинструктировать HTMX выполнить переадресацию на стороне клиента на другое место.

Хотя для начала работы с HTMX и Django это не обязательно, я настоятельно рекомендую использовать [пакет Django-HTMX](https://github.com/adamchainz/django-htmx), разработанный Адамом Джонсоном. Он предоставляет промежуточное программное обеспечение Django и другие полезные помощники, которые делают интеграцию Django и HTMX более удобной. Например, если установить промежуточное ПО django-htmx, то приведенный выше код можно переписать:

```py
def my_view(request):
    if request.htmx:
        return TemplateResponse(request, "_some_snippet.html")
    else:
        return TemplateResponse(request, "full_page.html")
```

Промежуточное ПО добавляет атрибут htmx к экземпляру HTTPRequest, передаваемому всем представлениям.

# Alpine

Конечно, Javascript может делать гораздо больше, чем просто обрабатывать AJAX-взаимодействия. Например, можно использовать модалы, выпадающие меню, "флэш"-сообщения и другие взаимодействия и эффекты, которые сложнее реализовать с помощью CSS.

Появилось множество библиотек для выполнения подобной внутристраничной работы, обеспечивающих более простой способ организации кода фронтенда, не требующий большого фреймворка фронтенда или "спагетти" из ванильного JS или jQuery. Одним из таких решений, опять же входящим в семейство Hotwire, является [Stimulus](https://stimulus.hotwired.dev/). Другой фреймворк, разработанный Карсоном Гроссом и другими, - [Hyperscript](https://hyperscript.org/). Однако моим личным фаворитом является [Alpine.js](https://alpinejs.dev/).

Alpine, как и HTMX, использует атрибуты для расширения функциональности. Обычно они начинаются с `x-`. Например:

```html
<button x-on:click="alert('hello')">Click me!</button>
```

Директива `x-on` может быть сокращена до просто `@`:

```html
<button @click="alert('hello')">Click me!</button>
```

События могут быть дополнительно модифицированы с помощью специальных ключевых слов: например, `@click.prevent` предотвратит распространение события, а `@click.window` вызовет событие при нажатии на любую часть страницы.

Alpine позволяет манипулировать DOM с помощью этого декларативного синтаксиса - если у вас есть опыт работы с Vue, вы увидите некоторые сходства. Например, предположим, что вы хотите скрыть кнопку, когда она нажата:

```html
<button x-data="{show: true}"
        x-show="show"
        @click="show=false">Click me and I'll hide!</button>
```

Атрибут `x-data` инициализирует данные (в виде объекта Javascript), относящиеся к данному элементу (и всем дочерним элементам). Здесь мы устанавливаем значение по умолчанию для `show` равным `true`. Директива `x-show` определяет условие, при котором элемент будет отображаться, т.е. до тех пор, пока `show` равно `true`. Наконец, наше событие `@click` устанавливает значение `show` в `false`, и в этом случае кнопка исчезает.

Можно также выполнять манипуляции с классами, опять же аналогично Vue:

```html
<button x-data="{clicked: false}"
        :class="{'text-red': clicked}"
        @click="clicked=true">Click me!</button>
```

Таким образом, когда срабатывает событие `@click`, к элементу `<button>` применяется класс `text-red`.

Как и в случае с HTMX, Alpine.js можно установить с помощью CDN:

```html
<script src="https://unpkg.com/alpinejs@3.7.1/dist/cdn.min.js"
        defer
        integrity="sha384-KLv/Yaw8nAj6OXX6AvVFEt1FNRHrfBHziZ2JzPhgO9OilYrn6JLfCR4dZzaaQCA5"
        crossorigin="anonymous"></script>
```

В остальном директивы Alpine могут быть вставлены в стандартные шаблоны Django или Jinja2 без дополнительной настройки.

# Резюме
Сочетание Alpine и HTMX является очень мощным и расширяет границы возможностей "традиционной" архитектуры Django. Вы можете получить большую часть функциональности и удобства модели SPA без сопутствующей хрупкости, сложности и стоимости. Это представляет особый интерес для стартапов на ранних стадиях, разработчиков-любителей и небольших SAAS-компаний, которые не могут позволить себе большую команду специалистов по бэкенду и фронтенду. Кроме того, эти библиотеки имеют очень короткую кривую обучения: каждая из них состоит всего из пары десятков пользовательских директив, которые может быстро освоить backend-разработчик с базовыми знаниями HTML и Javascript. Вы даже можете добавить HTMX и Alpine в существующий "старый" проект Django, добавив в него несколько действий для улучшения пользовательского опыта.

Стоит ли после этого выбрасывать свой код на React или Vue и наслаждаться более простой жизнью? Как всегда, ответ "зависит от ситуации". Если вы создаете очень сложный фронтенд с большим количеством взаимодействий с пользователем - например, игру или что-то вроде Notion или Google Docs, - то тяжелый фронтенд-фреймворк может оказаться лучшим выбором. Большая начальная нагрузка не имеет большого значения, поскольку пользователи будут держать вкладку приложения открытой в браузере в течение длительного времени. SEO также не будет проблемой, если пользователям придется входить в приложение, чтобы получить к нему доступ. Кроме того, эти фреймворки имеют развитые экосистемы, обеспечивающие наличие библиотек сторонних разработчиков, учебников и документации, а также большой кадровый резерв. Также можно использовать, например, React для одной небольшой части сайта, где это имеет смысл, в то время как остальная часть сайта использует многостраничную архитектуру - например, сложная интерактивная панель управления.

Однако проблема заключается не в самой архитектуре SPA, а в том, что в настоящее время доминирует представление о SPA как о парадигме по умолчанию для всех веб-проектов, а не как об одном из возможных подходов среди множества других, которые следует тщательно рассмотреть, исходя из требований проекта и навыков команды разработчиков. Именно поэтому такие библиотеки, как HTMX и Alpine, являются отличным дополнением к набору инструментов.
