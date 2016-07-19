---
layout: post
title: "Clojure vs NULL"
excerpt: "Возвращаясь к Clojure после нескольких лет Haskell и Java снова переосмысливаю старые проблемы. Через призму нового опыта хочется поразмышлять о том, как справиться с набившей оскомину NullPointerException. В Haskell их не бывает, т.к. там нет такого понятия, как NULL. В Java NPE настолько распространенное явление, что его или отлавливают через try/catch или вообще не обращают на него внимания, позволяя пролезать ему на самый верх и в логи. Еще одной частой проблемой в Java являются лесенки из вложенных if-ов, проверяющих входные параметры на NULL. А как дела с NPE в Clojure?"
tags: [haskell]
categories: [tips&tricks]
author: Дмитрий Бушенко
comments: true
image:
  feature: 2016-07-19.jpg
---

Возвращаясь к Clojure после нескольких лет Haskell и Java снова переосмысливаю старые проблемы. Через призму нового опыта хочется поразмышлять о том, как справиться с набившей оскомину NullPointerException. В Haskell их не бывает, т.к. там нет такого понятия, как NULL. В Java NPE настолько распространенное явление, что его или отлавливают через try/catch или вообще не обращают на него внимания, позволяя пролезать ему на самый верх и в логи. Еще одной частой проблемой в Java являются лесенки из вложенных if-ов, проверяющих входные параметры на NULL. Давайте посмотрим, как дела с NPE в Clojure.

### Почему возникает NullPointerException ?

Ответ на этот вопрос важен в контексте этой статьи. Дело в том, что *null* в Java и *nil* в Clojure, как ни странно, кардинально разные вещи. В Java null означает отсутствие объекта. Поэтому любой доступ к любому члену отсутствующего объекта не возможен, что и приводит к NPE. А в Clojure nil -- это не отсутствие объекта, это [NullObject](https://en.wikipedia.org/wiki/Null_Object_pattern), т.е. объект, на котором возможно вызывать любые функции, просто они будут возвращать пустые результаты.


{% highlight clojure %}

(cons nil nil)
;=> (nil)

(assoc nil nil nil)
;=> {nil nil}

(first nil)
;=> nil

(next nil)
;=> nil

(rest nil)
;=> ()

{% endhighlight %}


*Nil* в Clojure не является проблемой. Проблемы возникают при передаче nil в Java-код.
{: .notice}

В Clojure мы так часто взаимодействуетвуем с Java-кодом, что *nil* приносит те же проблемы, что и *null* в Java.

### Ну и в чем проблема?

Теоретически, если выполнить все проверки во всех местах, где может возникнуть NPE, всё должно работать. Но читабельным [такой код](https://github.com/spring-projects/spring-framework/blob/master/spring-core/src/main/java/org/springframework/util/CollectionUtils.java) никак не назовешь.

{% highlight clojure %}
public static <K, V> void mergePropertiesIntoMap(Properties props, Map<K, V> map) {
		if (map == null) {
			throw new IllegalArgumentException("Map must not be null");
		}
		if (props != null) {
			for (Enumeration<?> en = props.propertyNames(); en.hasMoreElements();) {
				String key = (String) en.nextElement();
				Object value = props.getProperty(key);
				if (value == null) {
					// Potentially a non-String value...
					value = props.get(key);
				}
				map.put((K) key, (V) value);
			}
		}
	}
{% endhighlight %}

Проблема здесь -- потеря фокуса. Думаю, все знают, что идеальная функция должна делать ровно одну вещь. Посмотрите, как сильно отвлекается внимание на анализ вот этих вот *что-то == null*. Эта функция помимо своего основного "пути исполнения" программы содержит еще несколько неосновных веток. Идеальная функция не должна содержать веток вообще, путь исполнения должен быть один, и он должен быть прямой.

Весь вопрос здесь -- в *поддерживаемости* кода. Чем он проще, тем легче код поддерживать, тем меньше тратится времени на то, чтобы его понять или изменить. Если функция содержит только один "путь исполнения", без условных операторов с проверками на null, то такую функцию намного легче понять. Если функция загажена целой лестницей if-ов, проверяющих результаты вычислений на null, то такой код намного труднее понять и, соответственно, дороже поддерживать.

Nil может образоваться в трех различных местах функции, и в каждом из них может быть свой способ упростить код:

- в аргументах функции
- в теле функции при вычислении промежуточных значений
- в возвращаемом значении функции

### Как бороться с *nil* в аргументах функции

Если коротко -- *никак*. Здесь я полностью согласен с Егором Бугаенко, который в [[1]](#book1) утверждает, что ни принимать null в аргументах, ни проверять на null аргументы функции не следует. Егор очень убедительно показывает, что это просто не ООП-стиль (по крайней мере в понимании Егора). Я же со своей стороны напомню, что такие проверки создают лишнюю ветку, а любой if значительно усложняет понимание кода. Вот две моих рекомендации по поводу nil в аргументах функции.

1) Впринципе, позволить коду "упасть" с NPE из-за неверных аргументов -- не самая плохая идея. Однако есть некоторая разница: функция "упадет" сразу, как только будет вызвана на nil, или же где-нибудь в середине, успев создать пару side effect-ов. Наверное, функция, не закончившая своё выполнение и записавшая промежуточные результаты куда-нибудь на диск, например, или в БД, -- гораздо хуже, чем функция, "упавшая" сразу. Поэтому мой совет здесь -- использовать *:pre* для предварительных проверок на nil. Например, так:

{% highlight clojure %}
(defn maynil [v]
  {:pre [(not (nil? v))]}

  (when v
    v))
{% endhighlight %}

2) Использовать [декораторы](http://www.yegor256.com/2016/01/26/defensive-programming.html), если нужно какое-то другое поведение, кроме NPE. В Clojure это значительно проще, чем в Java.

{% highlight clojure %}
(defn maynil [v] v)

 (defn default [v d]
   (if-not (nil? v)
     v
     d))

 (def a (default (maynil nil) 10)) ;;=> 10
{% endhighlight %}

### Как бороться с *nil* в теле функции

Еще раз напомню, что проблема здесь -- в возникновении "лесенок" из if-ов, проверяющих результаты промежуточных вычислений на nil.

{% highlight clojure %}
(defn may-return-nil [ num ] (if (odd? num) nil num))
 (defn always-num [ num ] num)

 (defn ladder-1 [ num ]
   (let [ n1 (may-return-nil num) ]
       (if-not (nil? n1)
         (let [ n2 (may-return-nil (* n1 n1)) ]
              (if-not (nil? n2)
                (let [ n3 (always-num (* n2 n2)) ]
                     [ n1 n2 n3 ])
                nil))
         nil)))
{% endhighlight %}

Обратите внимание, насколько сложна для понимания функция *ladder-1*! Часто ли вы встречаете что-то подобное в своем коде?

Вот мои рекомендации, что можно сделать в таком случае.

1) Функции никогда не должны возвращать nil. Просто позвольте коду "упасть" с NPE. Обнаружив NPE из-за *may-return-nil*, мы понимаем, что эту функцию нужно переписать так, чтобы она никогда не возвращала nil.

2) Если nil возвращает чужая функция, код которой невозможно исправить, -- добавьте к ней декоратор со значением по умолчанию, как показано выше.

3) Если нет возможности добавить декоратор со значением по умолчанию, а падение с NPE -- нежелательное поведение, то нужно воспользоваться монадой [Maybe](http://funcool.github.io/cats/latest/#maybe). Предыдущая функция *ladder-1* превращается в что-то, похожее на *ladder-2*:

{% highlight clojure %}
(defn may-return-nothing [ num ]
   (if (odd? num)
     (maybe/nothing)
     (maybe/just num)))

 (defn ladder-2 [ num ]
   (mlet [ n1 (may-return-nothing num)
           n2 (may-return-nothing (* n1 n1))
           n3 (may-return-nothing (* n2 n2))
         ]
         (return [n1 n2 n3])))
{% endhighlight %}

Фишка этого кода в том, что nil здесь не возникает никогда и ни при каких обстоятельствах. Если функция may-return-nothing не может вернуть какое-либо значение, то она вернеть не nil, а объект типа Nothing. При этом mlet обнаружит такую ситуацию и прервет вычисления, также вернув Nothing. В случае успеха функция ladder-2 вернет результат в обертке из объекта Just.

Функция *ladder-2* значительно проще, чем *ladder-1*, т.к. в ней всего один "путь исполнения", нет никаких ответвлений, ни одного if-а.

### Как бороться с *nil* в возвращаемом значении

Главная мысль здесь -- никогда не возвращайте nil.

1) Если результатом вашей функции должна быть коллекция -- верните пустую коллекцию.

2) Если результат вашей функции -- единичный элемент, и функция определена не на всех значениях, то оберните её результат в Maybe. Тогда, если на каких-то аргументах функция не сможет вернуть значение, она вернет Nothing.

### Выводы

Проблема NPE в коде -- это не проблема неотловленных исключений или ошибок в коде. Это проблема читабельности кода. Если корректность кода достигается за счет множества запутанных проверок, то такой код становится слишком сложно поддерживать. Пусть этот код и корректный, но что-то поменять в нем будет чрезвычайно сложно. Поэтому основные усилия при борьбе с NPE нужно прилагать к тому, чтобы корректный код стал читабельным. Добиться этого можно за счет 1) assert-ов в секции *:pre*; 2) проверяющих декораторов; 3) использования монады Maybe.

### Литература

<div id="book1">1. Bugayenko Ye., 3.3 Never accept NULL arguments. In: Elegant Objects. Location: CreateSpace, 2016, pp.146-152.</div>

