# VK Group Bot
Думаю, что многие из нас сталкивались со следующей проблемой: много и даже очень много мусора в новостной ленте VK, от которого не может спасти т.н. *"Умная лента"*. А вы всё пролистываете и пролистываете эту нескончаемую ленту ради немногих действительно полезных / интересных / смешных постов, не так ли? Нельзя ли подписаться лишь на одну группу, а не 50, и получать только лучшие посты в новостях, да и в принципе тратить своё время на что-то полезное, а не бесконечное листание? Да, можно, и именно для этого и предназначен данный бот.
Основная идея - Бот ежедневно сам определяет какие посты были лучшими за день во всех интересующих вас группах и выкладывает нужное вам количество в группу.
# Оглавление

 1. [Принцип работы](#Принцип-работы)
	 1. [Устройство класса Post](#Устройство-класса-post)
	 2. [Как всё работает](#Как-всё-работает)
 2. [Установка и настройка](#Установка-и-настройка)
	 1. [Скачивание](#Скачивание)
	 2. [Создание Standalone приложения](#Создание-standalone-приложения)
	 3. [Получение access token](#Получение-access-token)
	 4. [Создание группы](#Создание-группы)
	 5. [Выбор нужных групп](#Выбор-нужных-групп)
	 6. [Автоматический запуск Бота](#Автоматический-запуск-Бота)
 3. [Дополнительно](#Дополнительно)
	 1. [Несколько групп](#Несколько-групп)
	 2. [Параметры в конфигурационном файле](#Параметры-в-конфигурационном-файле)
	 3. [Логи](#Логи)
	 4. [Используемые технологии](#Используемые-технологии)
# [Принцип работы](#Оглавление)
Тут ничего разжёвываться не будет, в отличии от следующего пункта.
Всё начинается с того, что Бот читает конфиг, проверяет на наличие ошибок, если находит их, то ~~посылает~~ пишет в чём проблема и завершает работу. Если всё хорошо, то далее он производи следующие шаги для всех групп.
## [Устройство класса Post](#Оглавление)
У каждого медиа файла в VK есть ID владельца и ID самого файла.
Post содержит информация о том какой группе он принадлежит, какое у группы название, сколько участников, информация о метриках и о медиа данных (фото, видео, музыка, опросы, документы, заметки), которые в нём содержатся.
## [Как всё работает](#Оглавление)
Для начала, получаем все так называемые т.н. короткие имена групп, которые лежат в ссылках, получаем их ID, названия (они нам пригодятся для логирования) и количество участников. Потом из каждой группы берём 100 последних постов (100 постов можно получить максимально одним запросом и там 100% будут содержаться все вчерашние посты, т.к. в день максимально можно выкладывать 50 постов), выбираем из них те, в которых нет рекламы и прочей шелухи, потом выбираем вчерашние.  

Для каждого поста у нас есть 3 основные метрики: лайки, репосты и просмотры. Мы дополняем их отношением лайков к просмотрам и репостов к просмотрам (то есть сколько людей из тех, кто увидели пост, лайкнули или репостнули его), то бишь объединяем 3 метрики в 2 новые и более релевантные. Далее мы выбираем лучшим постом вчерашнего дня тупо тот пост, который собрал больше всего лайков - это наилучший подход, доказано эмпирическим путём.

Теперь у нас собраны по одному лучшему вчерашнему посту почти со всех групп. Почти, потому как где-то могли и не выкладывать посты вчера. Теперь для выбора `number_of_posts` постов, которые мы хотим выкладывать в день, нам необходимо их отсортировать по годности. Это - камень преткновения, и именно об него бились мои алгоритмы выбора самых "годных" постов. В итоге был вычислен следующий способ:

 1. Берём наш массив постов, копируем и сортируем по метрике "лайк за просмотр" в порядке возрастания;
 2. Берём наш массив постов, копируем и сортируем по метрике "репост за просмотр" в порядке возрастания;

Вот теперь-то мы и готовы к тому, чтобы выявить меру "годности" постов.

 3. Для каждого поста мы смотрим сумму индексов, на которых находится пост в массивах из п. 1 и 2. Так как те массивы отсортированы по убыванию, то пост с наибольшим суммарным индексом наиболее интересен публике;
 4. Сортируем наш изначальный массив по суммарному индексу из прошлого пункта в порядке убывания.

Спрашивается, а почему бы тоже не брать тупо по лайкам, или тупо по репостам? А вот почему: 

 - "лайкни или котик умрёт" 
 - "репост, если тоже любишь маму"

Примеры, конечно, образные... Или нет??? Нет, не образные, а используя комбинирование наших двух релевантных метрик мы получаем наилучшую метрику, которая будет защищать нас и нашу группу от постов подобного уровня. Ну и в любом случае, глупо полагаться лишь на что-то одно. А так мы получаем лучшие посты. 

Ну получили мы отсортированный массив по годности, и дальше просто обрезаем этот массив и выкладыв... Нет, не просто. Нужно создать эффект случайности, именно поэтому мы не берём просто `number_of_posts` постов, нет, мы берём `number_of_posts * scatter` постов, потом их перемешиваем и уже потом обрезаем до размеров `number_of_posts`.

Как вообще выкладывать что-то на стену? Ну что ж, если это медиа файл, то для начала его нужно загрузить на сервера VK, для дальнейшего получения его ID и прочего для размещения записи на стене 

> А вообще, процесс этот на редкость мерзкий и долгий, содержит очень много шагов, лишних запросов, которые можно было бы убрать. Единственный раздел VK API, который меня не устраивает - это именно загрузка медиа файлов.

, такой способ был реализован в прототипе этого Бота, написан [он](https://github.com/NikitiusD/VK_Group_Bot) был на C#, у него была иная логика и принцип действия, но так или иначе, зёрна идей, которыми я вдохновлялся при написании этого Бота, были заложены ещё во времена, когда я хотел реализовать его на C#. Реализовал, конечно, но не запустил из-за загруженности во время сессии. А дальше уже и идеи для улучшения подоспели, было решено переписать полностью - получилось то, что вы сейчас и видите.

Благо, идеи имеют свойство развиваться и было принято решение ничего не загружать на сервера, а работать с тем, что имеем. А имеем мы, как вы помните, ID всех наших медиа файлов и ID владельцев. С их помощью можно разместить на стене, как бы, не свой файл. То есть на стене ты видишь просто, например, картинку, а при нажатии открывается окошко с ней, а там уже видно и группу, и лайки, и репосты, и так далее.

Далее мы делаем `number_of_posts` отложенных записей на завтрашний день в нашу группу с равными промежутками. Но только если не вмешается ошибка, связанная с тем, что длина запроса слишком большая. Такое бывает, если вы пытаетесь запостить, например, очень длинный анекдот (такое уже было), а длина запроса не резиновая. Думаю, это один из многих способов защиты от дудоса серверов VK. Но Бот из-за этой ошибки, естественно, не летит, а просто пишет, что произошла ошибка.

Потом мы логируем всю информацию о выложенных постах, о группах, в которых вчера не было постов (о плохих группах), а также интересные графики. Логи, кстати, пишутся без единого вмешательства человека. Сама и папочка сделается, и логи по разным группам не потеряются. Графики стали фичей исключительно из-за того, что во время опытов по подбору алгоритма определения годности постов мне нужно было видеть не просто цифры с метрик, а красиво представленные значения метрик по всем постам, чтобы "зрить в корень". 

Всё.
# [Установка и настройка](#Оглавление)
## [Скачивание](#Оглавление)
Скачайте и распакуйте в любом удобном вам месте.
Подразумевается, что у вас уже установлен Python 3.
## [Создание Standalone приложения](#Оглавление)
Для начала вам нужно создать своё приложение VK:
[Ваши приложения](https://vk.com/apps?act=manage) -> "Создать приложение" -> в списке "Платформа" выбираете "Standalone-приложение", называете как вам угодно -> ничего не настраиваете, снова переходите в [ваши приложения](https://vk.com/apps?act=manage) -> "Редактировать" -> выбираете вкладку "Настройки" -> вам нужно только "ID приложения".

***Вы создали приложение, которое необходимо для создания ключа доступа.***
## [Получение access token](#Оглавление)
В папке с ботом создайте пустой текстовый файл `access_token.txt`.
Там же лежит конфигурационный файл `config.json`, откройте его и скопируйте значение поля `"token"` и вставьте его в адресную строку браузера, должно получиться что-то такое:

    https://oauth.vk.com/authorize?client_id=6289595&display=page&redirect_uri=https://oauth.vk.com/blank.html&scope=friends,photos,audio,video,status,wall,offline,groups,messages,manage&response_type=token&v=5.74

В ссылке есть следующий элемент: `client_id=6289595`. Замените цифры на ID вашего приложения, который вы получили из предыдущего пункта. Пока не запускайте страницу. В конце ссылки есть `v=5.74`. Пройдите [сюда](https://vk.com/dev/versions) и посмотрите какая версия VK API последняя. Опять же, эти цифры нужно заменить на актуальные. После замены ID и версии VK API в адресе запустите страницу. Вы будете перенаправлены на страницу, в которой ваше приложение запросит доступ к некоторым возможностям вашего аккаунта -> нажимаете "Разрешить" -> вас перенаправляют на страницу, в которой будет написано 

> Пожалуйста, не копируйте данные из адресной строки для сторонних
> сайтов. Таким образом Вы можете потерять доступ к Вашему аккаунту.

Вы должны посмотреть на адрес, на который вас перенаправило, там вы увидите такой элемент: `access_token=...&`. На месте "..." должно быть много латинских букв вперемешку с цифрами. Скопируйте все символы, которые стоят между `=` и `&` и вставьте их в недавно созданный `access_token.txt`.

***Вы получили ключ доступа, с помощью которого Бот будет работать с VK API.***
## [Создание группы](#Оглавление)
Можете использовать группу, которую вы создали ранее, либо создайте новую.
[Мои группы](https://vk.com/groups) -> "Создать сообщество" -> Можете выбрать тип "Тематическое сообщество", придумываете название, тематику и создаёте -> вы оказываетесь на странице настройки группы -> в "Адрес страницы" видите что-то вроде такого: `https://vk.com/public123456789`, копируете цифры после `public` и вставляете их в файл `config.json` в поле `"group_id"` вместо имеющегося там значения. Не забудьте обернуть в двойные кавычки. 

***Вы дали Боту ID вашей группы, в неё он будет выкладывать лучшие посты.***
## [Выбор нужных групп](#Оглавление)
Перейдите в раздел ссылки и добавьте все группы, которые вам интересны. Например, в той группе, в которой Бот был использован впервые, в ссылках лежали около 100 групп с мемами и смешными картинками. 

***Вы выбрали группы, из которых Бот будет выбирать лучшие посты.***
## [Автоматический запуск Бота](#Оглавление)
Если у вас нет своего сервера, то вот как нужно поступить.
Подразумевается, что у вас Windows.
 1. В поиске Windows найдите "Планировщик заданий"
 2. Вкладка "Действие" -> "Создать задачу"
 3. Во вкладе "Общие" указываете Имя
 4. Во вкладке "Триггер" нажимаете "Создать", далее выбираете пункт "Ежедневно", дату в "Начать" можете оставить прежнюю, а время выбирайте во второй половине дня, лучше вечером, примерно в то время, когда у вас гарантированно включен компьютер (потому как скрипт сможет запуститься только, если компьютер выключен). 
 5. Во вкладке "Действия" нажимаете "Создать", далее в поле "Программа или сценарий" вставьте путь до файла `launch.bat`, который лежит в папке `\src`. Например, `C:\Projects\VK-Group-Bot-v.2\src\launch.bat`. В "Рабочая папка" вставьте тоже самое, но без `\launch.bat`. Например, `C:\Projects\VK-Group-Bot-v.2\src`.
 6. Нажмите "Ок"

Если же у вас есть свой сервер, то вы сможете его настроить с помощью docker и [этого](https://hub.docker.com/r/kaylaweb/python-hello-world/~/dockerfile/).
Если же вы линуксоид, то настроить автозапуск скрипта для вас не будет проблемой, как и поменять батник на bash'евский скрипт. Удачи!

***Вы сделали так, что Бот будет запускаться автоматически.***
# [Дополнительно](#Оглавление)
## [Несколько групп](#Оглавление)

Если вы хотите, чтобы у вас было, несколько групп на разные тематики, вы легко можете расширить работу Бота для нескольких групп. Что для этого необходимо? 

 - Повторение шагов [Создание группы](#Создание-группы) и [Выбор нужных групп](#Выбор-нужных-групп) для каждой из групп, которую вы хотите создать;
 - В файле `config.json` вам необходимо из поля `groups` скопировать всё, что содержится внутри фигурных скобок, включая сами скобки, поставить запятую после `}`, сделать перенос строки, вставить скопированное и изменить информацию, как минимум, о `group_id`. 
Например,
```
"groups":
[
	{
		"group_id": "158155713",
		"number_of_posts": 25,
		"repost_border": 5,
		"scatter": 1.4
	},
	{
		"group_id": "123456789",
		"number_of_posts": 25,
		"repost_border": 5,
		"scatter": 1.4
	}
],
```
 - Сохранить конфигурационный файл;
 - Радоваться жизни.
## [Параметры в конфигурационном файле](#Оглавление)
 - `"token"` - ссылка, необходимая для получения `access token`;
 - `"groups"` - массив групп, которые Бот должен вести;
	 - `"group_id"` - ID группы;
	 - `"number_of_posts"` - количество постов, которые Бот будет отбирать и выкладывать каждый день;
	 - `"repost_border"` - уровень репостов, ниже которых Бот будет пропускать посты и не рассматривать их как лучшие;
	 - `"scatter"` - коэффициент, с которым добавляется небольшая случайность в финальном отборе лучших постов. Не может быть ниже 1. Не рекомендуется присваивать значение больше 2;
 - `"vk_limit"` - внутренне ограничение VK по максимальному числу постов, которые можно откладывать в сутки;
 - `"version"` - версия VK API, рекомендуется поставить актуальную.
## [Логи](#Оглавление)
Структура логов:
 - Логи постов: \logs\log_{ID группы}_{дата запуска Бота}.txt
 - Логи плохих групп: \logs\log_{ID группы}_{дата запуска Бота}_bad_groups.txt
 - Логи графиков (1920*1080): \logs\log_{ID группы}_{дата запуска Бота}.png
## [Используемые технологии](#Оглавление)
- VK API - рекомендую познакомиться поближе для более полного понимания принципов работы Бота, начать можно [отсюда](https://vk.com/dev/first_guide).
- Python библиотеки `json`, `requests`