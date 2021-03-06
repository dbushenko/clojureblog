---
layout: post
title: "Почему после двух лет Haskell я снова вернулся к Clojure"
excerpt: "Программы на Haskell получаются такие красивые именно потому, что основная задача Haskell-программиста -- создать полную, корректную, стройную, математичную модель данных. Сколько на это тратится времени и решает ли эта программа какую-нибудь задачу -- не имеет значения."
tags: [haskell]
categories: [разработка]
author: Дмитрий Бушенко
comments: true
image:
  feature: 2016-07-09-head.jpg
---

### Как я попал в Haskell

В процессе изучения Clojure еще в конце 2009-го года я стал всё чаще встречать упоминания о Haskell. В Clojure пришли хаскелисты и притащили туда монады, монадические комбинаторы парсеров, идею о борьбе с NPE через тип Maybe и многое другое. Упоминали также, что и сам Clojure подвергся очень сильному влиянию Haskell: все эти персистентные структуры данных, транзакционная память и, наверняка, многое другое позаимствовали в Haskell. Мне стало интересно, и я решил докопаться до истоков.

Оказалось, что, действительно, Haskell -- очень мощный язык, программы на котором выглядят потрясающе красиво. Более того -- Haskell затягивает. Одного лишь изучения языка становится мало, хочется понять, на каких принципах он основан. Даже беглое ознакомление с теорией категорий приводит к мысли, что Haskell -- это просто способ записать на компьютере утверждения из математики, а программа на Haskell -- это *одна большая формула*.

Однако уже на этапе просто ознакомления с языком стали появляться проблемы, которым я, впрочем, не придал тогда большого значения. Изучение Haskell шло медленно. Тогда еще не было отличного [курса](https://stepic.org/course/%D0%A4%D1%83%D0%BD%D0%BA%D1%86%D0%B8%D0%BE%D0%BD%D0%B0%D0%BB%D1%8C%D0%BD%D0%BE%D0%B5-%D0%BF%D1%80%D0%BE%D0%B3%D1%80%D0%B0%D0%BC%D0%BC%D0%B8%D1%80%D0%BE%D0%B2%D0%B0%D0%BD%D0%B8%D0%B5-%D0%BD%D0%B0-%D1%8F%D0%B7%D1%8B%D0%BA%D0%B5-Haskell-75/syllabus) Дениса Москвина, в деталях объясняющего много тонкостей. Не было потрясающе понятной книжки [HaskellBook](http://haskellbook.com/), подробно разжевывающей сложные определения. Не было даже ориентированной на практику [Изучаем Haskell](http://www.ozon.ru/context/detail/id/35161141/) Алехандро Серано Мена. А то, что [было](http://www.ozon.ru/context/detail/id/24811492/), [добавляло](http://book.realworldhaskell.org/) больше вопросов, чем ответов.

Честно говоря, сейчас, по прошествии нескольких лет, я всё так же считаю, что *изучать* Haskell -- довольно сложно, даже несмотря на появление более качественных и доступных материалов.

В результате я решил, что моё поверхностное знакомство с Haskell длится слишком долго. Пора разобраться с Haskell основательно: нужно или сделать его одним из своих инструментов, или полностью отказаться от него.

В 2014-м я выделил действительно много времени на изучение языка, а весь 2015-й и часть 2016-го -- практиковался. Помимо множества хобби-проектов, вроде [кодогенератора trurl](https://github.com/dbushenko/trurl), я написал и запустил во "внутренний" продакшен два single-page веб-приложения, где backend был полностью на Haskell, а frontend -- на PureScript. Я это к тому, что все проблемы, которые я выявил при использовании Haskell, прошли через вот этот, пусть небольшой, но всё же опыт использования Haskell в реальных проектах.

### C чем я столкнулся на практике

Во время разработки двух внутренних проектов на Haskell я столкнулся с несколькими странностями, которые, на мой взгляд, здорово мешают использовать Haskell на реальных задачах. Здесь я хотел бы подчеркнуть, что мои задачи -- это вовсе не разработка компиляторов или каких-то наукоёмких технологий. Всё проще: я делаю веб-сервисы и веб-приложения.

##### Haskell сложно отлаживать

К сожалению -- это всё-таки имеет значение. Самый обычный дебаггер с самыми обычными брейкпоинтами в самом деле могут помочь *быстро* решить проблему. Если у вас Java или C#, но никак не Haskell. Теоретически, брейкпоинты можно использовать и в Haskell, но это настолько сложно, что на практике вы будете использовать просто вывод в консоль при помощи *trace* или *traceShow*. Дело в том, что в Haskell программа -- это длинная формула, да еще и ленивая. Иногда вам просто некуда будет этот брейкпоинт поставить!

А вторая проблема отладки -- сложность контекстов. Это всё в теории выглядит хорошо -- чистые функции, красивые типы. А на практике у вас будет гораздо чаще монада, завёрнутая в несколько слоёв монадических преобразователей, и понять где там что -- та ещё задача. Там даже *trace* влепить некуда, так что приходится полагаться в основном на юнит-тесты, которые, кстати, в таких сложных контекстах писать тоже сложно.

##### В Haskell нет нормального стека для веб-разработки

Тут я, конечно, рискую словить пару тухлых помидоров от любителей [Yesod](http://www.yesodweb.com/) или [Servant](haskell-servant.github.io/), но всё же дайте мне пояснить мою позицию. Я, как инженер, терпеть не могу всякую "черную магию". Если я не понимаю технологию или инструмент, то сам, по своей воле, ни за что его не выберу. Если я вынужден использовать фреймворк, в котором я не разбираюсь или не способен разобраться, то чем я отличаюсь от поклонников [карго-культа](http://lurkmore.to/%D0%9A%D1%83%D0%BB%D1%8C%D1%82_%D0%BA%D0%B0%D1%80%D0%B3%D0%BE)?

![Cargo cult](/img/2016-07-09-cargo-cult-lead.jpg)

Я не понимаю Yesod и врядли когда-нибудь у меня будет пара свободных лет, чтобы разобраться, как он внутри устроен. Даже в разных Lisp-ах я никогда не видел столько метапрограммирования, столько макросов, как в Yesod. Но, в отличие от Lisp-овых макросов, те же квазицитаты Template Haskell устроены очень сложно, их трудно писать и еще труднее понимать. Если в Yesod что-нибудь пойдет не так, как ожидается, -- смогу я сам всё починить, или буду годами ждать чужих багфиксов?

Помимо метапрограммирования, Yesod использует множество хитрых расширений языка, с которыми тоже нужно разбираться. Или не разбираться -- и тогда не понимать, как работает даже ваша собственная программа.

Yesod, как и любой фреймворк, накрепко привязан к целому ряду решений, от которых сложно отказаться. Этот и Persistent для доступа к БД, и Shakespearean Templates для фронтенда, и многое другое. То, что Yesod "толстоват", -- это, конечно, проблема почти всех фреймворков. Но разобрать его на составляющие и использовать только то, что нужно, -- задача не из простых.

И при том, что в Yesod много готовых решений для типовых задач, там иногда нету самых простых вещей. Например, в Yesod нечем организовать сессию в памяти. Вы можете хранить сессию на клиенте, и, если этого достаточно, то всё ОК. Но вы не сможете положить в сессию большой граф объектов, который дорого вычислять на каждый запрос. Можно, конечно, организовать кэширование, привязанное к id пользователя, но это уже -- велосипедостроение, а причина -- в отсутсвии сессии в памяти.

Проблемы Servant примерное такие же: сложное внутреннее устройство, да еще и общая бедность фреймворка. Нечем сделать сессию, нечем сделать нормальную аутентификацию, сложность стектрейсов и т.д.

Я лично обходился связкой из рутера [Scotty](https://github.com/scotty-web/scotty), Persistent-а для БД и aeson-а для общения с внешним миром. Не густо, как видите.

Надо сказать, что в Haskell вообще очень медленно проникают технологии из мейнстрима.

##### В Haskell нет нормального доступа к БД

Есть, конечно, Persistent или mysql-simple / postgresql-simple и многое другое, в том числе -- даже для NoSQL баз. И всё это совершенно неюзабельно. Ну вот, например, есть у вас сущность Cat, у неё есть свойства -- id, head, tail. Представим себе веб-сервис, который создаёт котиков, передавая нам JSON примерно такого вида: {"head": "white", "tail": "black"}. Пока что всё ОК. А теперь нам нужно его проапдейтить, и мы присылаем еще и его ID: {"id": 17, "head": "white", "tail": "black"}. Вот здесь уже начнутся проблемы. Даже если у нас и есть сущность вида

{% highlight haskell %}

data Cat = Cat { id   :: Maybe Int
                , head :: String
                , tail :: String
                }

{% endhighlight %}

всё равно мы не можем передать её в Persistent для изменения существующей записи. Проблема в том, что Persistent использует типизированные id сущностей, и поэтому их нельзя объединять в один тип данных. CatID -- отдельно, Cat -- отдельно. На UI -- наоборот, это вполне себе цельная сущность, и её неудобно посылать по частям. Единственный правильный выход здесь -- создавать тип данных Cat отдельно для UI и отдельно для Persistent, что приводит к значительному дублированию кода.

Про *-simple даже говорить не буду, для любого типа данных, у которых больше двух-трёх полей эти библиотеки использовать неудобно.

##### Haskell -- не для мейнстримных задач

Самая моя большая претензия к Haskell -- он решает не те задачи. Когда пишешь на Haskell основное внимание сосредотачиваешь на построении правильной модели данных, на выявление всех нужных типов данных и поиск различных закономерностей на этих типах.

В противовес этому мои ежедневные задачи -- это что-то прочитать из HTTP-запроса, что-то записать в БД, что-то где-то вычислить и записать в файл. Мои типовые задачи не имеют ничего общего с построением полной, корректной модели предметной области и отображением её на теорию категорий. Ей-богу, мне всё равно, если вдруг в моих данных случайно обнаружится возможность свёртки, и я смогу реализовать Monoid. Мне почти всегда без разницы, что я смогу применить функцию ко всем составным частям моих данных и реализовать Functor или даже Applicative Functor. Моя задача -- взять из HttpServletRequest какую-нибудь фигню, провалидировать и положить её в БД. Я хочу сделать это максимально простым и наглядным способом.

В Haskell программист большую часть времени тратит на разработку красивой, корректной, стройной, "математичной" модели предметной области. И именно поэтому, если всё получилось хорошо, исходник на Haskell выглядит так красиво, стройно и "математично". Сколько на это тратится времени и решает ли эта программа какую-нибудь задачу -- не имеет значения.
{: .notice}


Конечно, мне нравится Haskell, ведь программы на нём получаются и корректные, красивые. Но за такую роскошь приходится платить непомерную цену времени.

### Назад к Clojure

Когда ко мне пришел новый проект, в котором нужно было связать сразу несколько технологий, я и сам не заметил, как сделал всё на Clojure. Я совершенно не собирался использовать его как основной язык проекта. Просто пока экспериментируешь в REPL программа пишется сама собой, в конце остаётся всего лишь более-менее нормально всё оформить, завернуть в Component и Compojure, добавить БД -- и готово!

В отличие от Haskell, Clojure очень сильно направлен на практику. Здесь есть все нужные фреймворки и библиотеки, есть средства организации кода, такие как [Component](https://github.com/stuartsierra/component) и [Mount](https://github.com/tolitius/mount), есть сервер приложений [Immutant](http://immutant.org/), очень удобный в использовании рутер [Compojure](https://github.com/weavejester/compojure), целый ворох самых разных библиотек для доступа к данным.

Но самое главное: при работе с Clojure сосредотачиваешься именно на выполнении задачи.

Самое кардинальное отличие Clojure от Haskell в том, что Haskell заставляет создавать отдельные типы данных для всего подряд. Clojure, наоборот, предлагает использовать существующие типы данных (список, вектор, отображение) и не заморачиваться.
{: .notice}

А как же рефакторинг? Ведь правильная модель данных здорово упрощает рефакторинг, в Clojure рефакторинг гораздо сложнее!

Горькая правда в том, что большинство моего кода просто не доживает до рефакторинга. Мне нужно решение задачи здесь и сейчас, потому что завтра этой задачи может и не быть вовсе. И далеко не факт, что написанный даже на Haskell код мне когда-нибудь придется рефакторить.

### Напоследок

После двух лет изучения Haskell я сильно разочарован. С другой стороны, конечно, я понимаю, что применил Haskell не в той области, для которой он предназначен.

Вот мой основной вывод, к которму я пришел за эти два года: *не используйте Haskell в веб-разработке*.

