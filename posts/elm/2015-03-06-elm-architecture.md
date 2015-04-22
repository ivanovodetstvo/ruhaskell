---
author:      Александр Бондаренко
title:       Архитектура приложения в Elm
tags:        frp,elm
hrefToOriginal: https://github.com/evancz/elm-architecture-tutorial
description: Мы изучим очень простой способ компоновки прилоложения, представляющий собой бесконечно вкладываемые блоки. Он сильно улучшает модульность, упрощает повторное использование кода и тестирование.
---

Это руководство описывает общую архитектуру, которую вы встретите во всех приложениях
на [Elm](http://elm-lang.org/), от [TodoMVC](https://github.com/evancz/elm-todomvc)
до [dreamwriter](https://github.com/rtfeldman/dreamwriter#dreamwriter).

Мы изучим очень простой способ компоновки прилоложения, представляющий собой
бесконечно вкладываемые блоки. Он сильно улучшает модульность, упрощает
повторное использование кода и тестирование. С его помощью можно создавать
сложные приложения, разбивая их на составные части. Мы начнём с маленького
примера и будем постепенно расширять его, используя эти базовые принципы.

Интересно, что подобная архитектура *возникает* в Elm естественным образом.
Дизайн языка сам подводит вас к ней, читали ли вы этот текст или нет. Я и сам
обнаружил этот паттерн просто используя Elm и был шокирован его простотой и силой.

**Замечание**: Чтобы опробовать приведённый здесь код вам потребуется
[установить Elm](http://elm-lang.org/Install.elm) и форкнуть
[репозиторий](https://github.com/evancz/elm-architecture-tutorial).
Каждый пример имеет инструкцию как запустить код.

## Основной приём

Логика каждой программы в Elm разбивается на три чётко разделённые части:
модель, обновление и отображение. Вы можете каждый раз начинать с этого
скелета проекта, а затем постепенно заполнять его подробностями под вашу
конкретную задачу.

```haskell
-- Модель

type alias Model = { ... }

-- Обновление

type Action = Reset | ...

update : Action -> Model -> Model
update action model =
  case action of
    Reset -> ...
    ...

-- Отображение

view : Model -> Html
view =
  ...
```

Весь туториал мы будем использовать этот паттерн с небольшими изменениями и дополнениями.

## Пример №1: Счётчик

Наш первый пример это простой счётчик, который можно увеличивать или уменьшать.
Чтобы посмотреть его в действии, зайдите в каталог `1/`, запустите там `elm-reactor` и откройте в браузере
[http://localhost:8000/Counter.elm?debug](http://localhost:8000/Counter.elm?debug).

Код начинается с очень простой модели. Нам просто надо отслеживать одно число:

```haskell
type alias Model = Int
```

При обновлении модели тоже всё очень просто. Мы определяем набор действий, которые
могут выполняться и добавляем их обработку в функцию `update`:

```haskell
type Action = Increment | Decrement

update : Action -> Model -> Model
update action model =
  case action of
    Increment -> model + 1
    Decrement -> model - 1
```

Обратите внимание, что наш [тип-объединение][] `Action` ничего не *делает*.
Он просто описывает какие действия возможны. Если кому-то потребуется сделать
чтобы счётчик удваивался, когда нажимается кнопка, то надо просто добавить
новый конструктор в `Action`. Это означает, что наш код будет очень чётко
указывать на то, как может изменяться наша модель. Любой, кто будет читать код
моментально узнает что допустимо, а что нет. Кроме того, сразу становится
понятно куда добавлять новые возможности.

[тип-объединение]: http://elm-lang.org/learn/Union-Types.elm

Теперь осталось только сделать отображение (view) для нашей модели. Мы будем
использовать [elm-html][] для создания HTML, который отобразится в браузере.
Мы создаём `div`, содержащий в себе: кнопку уменьшения, `div` с текущим
значением счётчика и кнопку увеличения.

[elm-html]: http://elm-lang.org/blog/Blazing-Fast-Html.elm

```haskell
view : Signal.Address Action -> Model -> Html
view address model =
  div []
    [ button [ onClick address Decrement ] [ text "-" ]
    , div [ countStyle ] [ text (toString model) ]
    , button [ onClick address Increment ] [ text "+" ]
    ]

countStyle : Attribute
countStyle =
  ...
```

Самая сложная часть функции `view` это `Signal.Address`. Мы займёмся
этим в следующем части, а пока я хочу, чтобы вы отметили, что этот код
является **полностью декларативным**. Мы берём `Model` и выдаём некий `Html`.
И всё. Ни в каком месте мы не занимаемся ручным изменением DOM, что открывает
библиотеке [большой простор для оптимизации][elm-html] и даже значительно всё
ускоряет. Это просто здорово. Более того, функция `view` это самая обычная
функция и при её создании мы можем пользоваться всей мощью системы модулей Elm,
фреймворками для тестирования и библиотеками.

Это и есть суть компоновки любых приложений в Elm. Все примеры которые мы далее
увидим будут лишь небольшими вариациями этого базвого паттерна: модель (`Model`),
обновление (`update`), отображение (`view`).

## Отступление: оживление вашего приложения с помощью сигналов

Теперь разберём часть кода с `Signal.Address`.

До этого мы говорили только о чистых функциях и неизменяемых данных. Это
здорово, но нам надо также и реагировать на события из внешнего мира. В Elm этим
занимаются [сигналы][]. Сигнал это значение, которое изменяется со временем,
что позволяет нам говорить о том, как будет изменяться наша модель.

В принципе все программы будут иметь этот небольшой кусочек кода, который
обслуживает всё приложение. В примере №1 он выглядит вот так:

```haskell
main : Signal Html
main =
  Signal.map (view actions.address) model

model : Signal Model
model =
  Signal.foldp update 0 actions.address

actions : Signal.Mailbox Action
actions =
  Signal.mailbox Increment
```

Хочу обратить ваше внимание на несколько деталей:

  1. Мы начинаем с 0 в качестве стартового значения модели.
  2. Мы используем функцию `update` для продвижения состояния модели.
  3. Мы реагируем на поступающие в канал `actions` действия (`Action`).
  4. Мы выводим всё это на экран через функцию `view`.

Вместо того, чтобы сразу пытаться понять что же тут происходит на каждой строке,
я предлагаю сначала взглянуть на схему происходящего на высоком уровне.

[сигналы]: http://elm-lang.org/learn/Using-Signals.elm

![](https://raw.githubusercontent.com/evancz/elm-architecture-tutorial/master/diagrams/signal-graph-summary.png)

Голубая часть это наша программа, т.е. ровно те модель/обновление/отображание,
о которых мы говорили. Большую часть времени вы можете работать, не выходя
за границы этого поля.

Новым тут являются "каналы" и то, как они позволяют новым действиям (`Action`)
возникать в ответ на пользовательский ввод. На картинке они изображены
пунктирными линиями от монитора к вашей программе. Когда мы назначаем
определённые каналы в функции `view`, мы определяем, каким образом действия
пользователя будут попадать в наш код. Обратите внимание, что мы не
*выполняем* эти действия, а просто регистрируем их для нашей основной программы.
Это разделение является ключевой особенностью!

Я хочу отметить, что этот код с `Signal` по большому счёту одинаковый для всех
програм в Elm. Вы можете захотеть [узнать больше о сигналах][сигналы], но для
продолжения чтения вам будет достаточно этого общего понимания. Мы хотим
описать архитектуру, а не увязнуть в том, как всё устроено. Поэтому давайте
перейдём к расширению нашего примера!

## Пример №2: Пара счётчиков

В первом примере мы создали простой счётчик, но как это будет масштабироваться,
когда нам понадобится *два* счётчика? Сможем ли мы сохранить модульность?

Чтобы посмотреть пример №2 в действии, зайдите в каталог `2/` и запустите там
`elm-reactor`, после чего откройте
[http://localhost:8000/CounterPair.elm?debug](http://localhost:8000/CounterPair.elm?debug).

Основной нашей задачей сейчас является повторное использование *всего* кода
предыдущего примера. Чтобы этого добиться, мы создадим самостоятельный модуль
`Counter`, в который положим все детали реализации счётчика. Единственное
изменение будет в функции `view`, поэтому я не буду раскрывать все старые
определения.

```haskell
module Counter (Model, init, Action, update, view) where

type Model = ...

init : Int -> Model
init = ...

type Action = ...

update : Action -> Model -> Model
update = ...

view : LocalChannel Action -> Model -> Html
view channel model = ...
```

Создание модульного кода требует создания сильных абстракций. Нам нужны
границы, которые обеспечат функционал и скроют реализацию.
Снаружи модуля `Counter` мы видим просто набор значений:
`Model`, `init`, `Action`, `update` и `view`. Нам не важно как это всё
реализовано. В действительности, *невозможно* узнать как они реализованы,
что не даст никому завязываться на детали, которые не были опубликованы.

Теперь, когда у нас есть наш базовый модуль `Counter`, займёмся созданием
приложения `CounterPair`. Как всегда, начинаем с модели:

```haskell
type alias Model =
    { topCounter    : Counter.Model
    , bottomCounter : Counter.Model
    }

init : Int -> Int -> Model
init top bottom =
    { topCounter    = Counter.init top
    , bottomCounter = Counter.init bottom
    }
```

Наша модель является записью с двумя полями, по одному для каждого счётчика,
который мы хотим показать на экране. Это полностью отражает состояние приложения.
Ещё у нас есть функция `init`, создающая новую модель когда нам это потребуется.

Далее мы определяем набор действий, которые мы хотим поддерживать. В этот раз
у нас будет: сброс всех счётчиков, обновление верхнего счётчика или обновление
нижнего счётчика.

```haskell
type Action
    = Reset
    | Top    Counter.Action
    | Bottom Counter.Action
```

Обратите внимание, что наш [тип-объединение][] указывает на тип `Counter.Action`,
но мы не знанием подброностей этих действий. Когда мы создаём функцию `update`
мы просто отправляем эти действия в правильное место:

```haskell
update : Action -> Model -> Model
update action model =
  case action of
    Reset -> init 0 0

    Top act ->
      { model |
          topCounter <- Counter.update act model.topCounter
      }

    Bottom act ->
      { model |
          bottomCounter <- Counter.update act model.bottomCounter
      }
```

И напоследок остаётся только сделать функцию отображения, которая покажет
на экране оба наших счётчика и кнопку сброса.

```haskell
view : Signal.Address -> Model -> Html
view address model =
  div []
    [ Counter.view (Signal.forwardTo address Top) model.topCounter
    , Counter.view (Signal.forwardTo address Bottom) model.bottomCounter
    , button [ onClick address Reset ] [ text "RESET" ]
    ]
```

Здорово, что мы смогли использовать функцию `Counter.view` для обоих счётчиков.
Для каждого счётчика мы создаём адрес пересылки. Сообщения на этот адрес будут
отмечены как `Top` или `Bottom`, чтобы мы могли их различать.

Вот и всё. С помощью локальных каналов мы можем сколько
угодно вкладывать наш паттерн модель/обновлние/отображение. Например можно
взять модуль `CounterPair`, опубликовать ключевые значения и функции, а затем
создать пару пар счётчиков или что нам будет ещё угодно.

## Пример №3: Динамический список счётчиков

Пара счётчиков это круто, но как насчёт списка счётчиков, в который можно
добавлять новые и удалять по требованию? Будет ли наш приём работать и тут?

Чтобы посмотреть пример №2 в действии, зайдите в каталог `3/` и запустите там
`elm-reactor`, после чего откройте
[http://localhost:8000/CounterList.elm?debug](http://localhost:8000/CounterList.elm?debug).

В этом примере мы будем использовать тот же самый модуль `Counter`, что и
в примере №2.

```haskell
module Counter (Model, init, Action, update, view)
```

Это значит, что мы приступим сразу к модулю `CounterList`. Как обычно,
начинаем с модели:

```haskell
type alias Model =
    { counters : List ( ID, Counter.Model )
    , nextID : ID
    }

type alias ID = Int
```

Теперь наша модель это список счётчиков, каждый со своим уникальным ID.
Эти идентификакторы позволяют нам различать счётчики и когда мы захотим
обновить четвёртый, у нас будет простой способ это сделать. (Кроме того
эти ID дают удобный способ определять уникальный [`ключ`][key] когда мы
задумаемся об оптимизации отрисовки компонентов, но это мы не будем сейчас
особо останавливаться на этой теме.) Кроме того, у модели есть поле
`nextId`, помогающее нам назначать уникальные идентификаторы при добавлении
новых счётчиков.

[key]: http://package.elm-lang.org/packages/evancz/elm-html/latest/Html-Attributes#key

Опишем набор действий, которые мы можем применять к нашей модели. Мы
хотим иметь возможность добавлять счётчики, удалять их, а так же изменять
значения отдельных счётчиков.

```haskell
type Action
    = Insert
    | Remove
    | Modify ID Counter.Action
```

Наш [тип-объединение][] `Action` весьма похож на описанное выше поведение.

Давайте опишем функцию `update`.

```haskell
update : Action -> Model -> Model
update action model =
  case action of
    Insert ->
      let newCounter = ( model.nextID, Counter.init 0 )
          newCounters = model.counters ++ [ newCounter ]
      in
          { model |
              counters <- newCounters,
              nextID <- model.nextID + 1
          }

    Remove ->
      { model | counters <- List.drop 1 model.counters }

    Modify id counterAction ->
      let updateCounter (counterID, counterModel) =
            if counterID == id
                then (counterID, Counter.update counterAction counterModel)
                else (counterID, counterModel)
      in
          { model | counters <- List.map updateCounter model.counters }
```

Вот общее описение каждого случая:

  * `Insert` &mdash; Сначала мы создаём новый счётчик и кладём его в
    конец списка. Затем мы увеличиваем `nextID` чтобы в следующий раз
    у нас уже был готовый ID.

  * `Remove` &mdash; Удаляем первый элемент в нашем списке счётчиков.

  * `Modify` &mdash; Проходимся по списку счётчиков и, если попался нужный
    ID, вызываем `Action` для этого счётчика.

Всё что осталось это функция отображения.

```haskell
view : Signal.Address Action -> Model -> Html
view address model =
  let counters = List.map (viewCounter address) model.counters
      remove = button [ onClick address Remove ] [ text "Remove" ]
      insert = button [ onClick address Insert ] [ text "Add" ]
  in
      div [] ([remove, insert] ++ counters)

viewCounter : Signal.Address Action -> (ID, Counter.Model) -> Html
viewCounter address (id, model) =
  Counter.view (Signal.forwardTo address (Modify id)) model
```

Забавно, что функция `viewCounter` использует всё ту же функцию
`Counter.view`, но в этот раз мы используем адрес пересылки, помечающий
все сообщения выводящегося в данный момент счётчика его идентификатором.

Когда мы создаём функцию `view` приложения, мы применяем функцию `viewCounter`
на каждый элемент списка. А когда мы создаём кнопки добавления и удаления,
которые шлют сообщения в канал приложения `address` напрямую.

Подобный трюк с ID может быть использован в любом месте, где вам нужно
динамическое количество вложенных компонентов. Счётчики это довольно просто,
но такой паттерн будет работать в точности одинаково если у вас будет
список профилей пользователей, твитов, элементов ленты новостей или продуктов.

## Пример №4: Продвинутый список счётчиков

Окей, держать вещи простыми и модульными для списка счётчиков это здорово,
но что если вместо общей кнопки сброса у каждого счётчика будет своя кнопка
удаления? Уж это наверняка всё сломает!

Не, всё работает.

Чтобы посмотреть пример №4 в действии, зайдите в каталог `4/` и запустите там
`elm-reactor`, после чего откройте
[http://localhost:8000/CounterList.elm?debug](http://localhost:8000/CounterList.elm?debug).

В данном случае нам потребуется новый способ создавать счётчики вместе с их
кнопками. Мы можем оставить старую функцию `view` и просто добавить новую
`viewWithRemoveButton`, которая будет немного по-другому рисовать модель.
Нам не потребуется дублировать код или делать сумашедшие трюки с наследованием
или перегрузкой. Мы просто добавим в публичный интерфейс модуля новую функцию,
реализующую новую функциональность!

```haskell
module Counter (Model, init, Action, update, view, viewWithRemoveButton, Context) where

...

type alias Context =
    { actions : Signal.Address Action
    , remove : Signal.Address ()
    }

viewWithRemoveButton : Context -> Model -> Html
viewWithRemoveButton context model =
  div []
    [ button [ onClick context.actions Decrement ] [ text "-" ]
    , div [ countStyle ] [ text (toString model) ]
    , button [ onClick context.actions Increment ] [ text "+" ]
    , div [ countStyle ] []
    , button [ onClick context.remove () ] [ text "X" ]
    ]
```

Функция `viewWithRemoveButton` добавляет одну дополнительную кнопку.
Обратите внимание, что функции увеличения и уменьшения отправляют сообщения
в канал actionChan, а кнопка удаления шлёт их на адрес `actions`. Сообщения,
попадающие в канал `remove` как бы говорят: "Эй, кто там меня создал,
удаляй меня!". Что конкретно надо сделать для удаления решает уже
тот, кто создал этот конкретный счётчик.

Теперь, когда у нас есть `viewWithRemoveButton`, мы можем создать модуль
`CounterList`, в котором соберём все счётчики вместе. Тип `Model` будет
использоваться такой же, как в примере №3: список счётчиков и уникальный номер.

```haskell
type alias Model =
    { counters : List ( ID, Counter.Model )
    , nextID : ID
    }

type alias ID = Int
```

Возможные действия будут немного отличаться. Вместо удаления всех старых
счётчтиков мы хотим удалить только тот, чей ID совпадает с указанным.

```haskell
type Action
    = Insert
    | Remove ID
    | Modify ID Counter.Action
```

Функция обновления тоже не сильно отличается от предыдущего примера.

```haskell
update : Action -> Model -> Model
update action model =
  case action of
    Insert ->
      { model |
          counters <- ( model.nextID, Counter.init 0 ) :: model.counters,
          nextID <- model.nextID + 1
      }

    Remove id ->
      { model |
          counters <- List.filter (\(counterID, _) -> counterID /= id) model.counters
      }

    Modify id counterAction ->
      let updateCounter (counterID, counterModel) =
            if counterID == id
                then (counterID, Counter.update counterAction counterModel)
                else (counterID, counterModel)
      in
          { model | counters <- List.map updateCounter model.counters }
```

Когда приходит `Remove` мы убираем счётчик имеющий ID того, который мы
должны убрать. В остальном, всё примерно тоже самое.

И наконец, мы собираем всё это вместе в функции `view`:

```haskell
view : Signal.Address Action -> Model -> Html
view address model =
  let insert = button [ onClick address Insert ] [ text "Add" ]
  in
      div [] (insert :: List.map (viewCounter address) model.counters)

viewCounter : Signal.Address Action -> (ID, Counter.Model) -> Html
viewCounter address (id, model) =
  let context =
        Counter.Context
          (Signal.forwardTo address (Modify id) actionChannel)
          (Signal.forwardTo address (always (Remove id)) actionChannel)
  in
      Counter.viewWithRemoveButton context model
```

В функции `viewCounter` мы создаём `Counter.Context` и передаём туда обратный
адрес. В обоих случаях мы помечаем `Counter.Action` чтобы знать кого обновлять
или удалять.

## Основые уроки

**Базовый приём** &mdash; Всё строится вокруг типа `Model`, функции `update`
для её обновления и функции `view` для отображения. Дальше идут только
вариации этого приёма.

**Вложенные модули** &mdash; Адрес пересылки позволяет с лёгкостью углублять
основной приём, полностью скрывая детали реализации. Мы можем строить
какие угодно глубокие компоненты и каждый уровень должен знать только
о том, что находится непосредственно внутри него.

**Добавление контекста** &mdash; Иногда требуется дополнительная информация
для обновления или отображения модели. Мы всегда можем добавить в эти
функции контекст, не загружая основной тип `Model`.

```haskell
update : Context -> Action -> Model -> Model

view : Context' -> Model -> Html
```

На каждом уровне вложенности мы можем определить тот `Context` который нужен
для внутренних компонентов.

**Простота тестирования** &mdash; Все функции являются [чистыми][pure].
Это позволяет их очень просто тестировать - не требуется никакой особой
инициалиазции или искуственного окружения, просто передайте им аргументы
которые вы хотите опробовать.

[pure]: http://en.wikipedia.org/wiki/Pure_function

## Ещё один шаблон

Существует ещё один важный способ расширить основной приём. К примеру,
вам может потребоваться обновить компонент и, в зависимости от результата,
надо поменять что-то ещё в другой части программы. Вы можете расширить функцию
`update` чтобы она возвращала больше информации.

```haskell
type Request = RefreshPage | Print

update : Action -> Model -> (Model, Maybe Request)
```

В зависимости от логики обработки `update` мы можем сказать кому-то выше
обновить содержимое или вывести что-нибудь. Похожим образом компонент
может удалить себя:

```haskell
update : Action -> Model -> Maybe Model
```

Если не очень понятно как это работает, я может быть напишу пример 5,
использующий этот приём. А пока вы можете посмотреть похожие примеры в
[забавной версии приложения TodoMVC на Elm][fancy].

[fancy]: https://github.com/evancz/elm-todomvc/blob/trim/Todo.elm