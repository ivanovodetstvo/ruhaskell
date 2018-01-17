---
author:         Юрий Сыровецкий
title:          Эффекты в Haskell
tags:           эффекты
description:    Реализация эффектов в Haskell.
---

В [предыдущей статье](../10/effects.html) мы познакомились с основными видами
эффектов в математике и программировании.
Сегодня мы докажем, что для процедур,
то есть «функций с эффектами» не нужна особая поддержка со стороны языка
программирования,
достаточно реализации обычных «чистых» функций.

## 0. Отсутствие эффектов

Представим чистую функцию `f :: a -> b` в виде чёрного ящика:

<center>![](../../../../../files/posts/2018-01-18/pure.svg)</center>

## 1. Эффект частичности

<center>![](../../../../../files/posts/2018-01-18/partial.svg)</center>

Частичная функция либо возвращает результат, либо не возвращает.

Такая вариативность легко моделируется с помощью типа-суммы.

```haskell
data Maybe a = Nothing | Just a
```

Значение типа `Maybe a` либо содержит значение типа `a`, либо нет.

Мы можем описать частичную процедуру, иногда возвращающую `b`, как функцию,
всегда возвращающую `Maybe b`.

<center>![](../../../../../files/posts/2018-01-18/partial-pure.svg)</center>

```haskell
p :: a -> Maybe b
```

Обратите внимание, что тип `Maybe` принадлежит `Functor`, `Applicative`,
`Monad` и многим другим интересным и полезным классам.

(На практике также применяется тип `Except`, реализующий тот же эффект,
но позволяющий добавить информацию о том,
почему вычисление не может быть завершено.)

## 2. Эффекты недетерменированности (неопределённости)

### 2.1. Эффект нестабильности

<center>![](../../../../../files/posts/2018-01-18/unstable.svg)</center>

Если процедура для одного и того же значения аргумента может вернуть от раза
к разу разные результаты,
это значит, что на самом деле результат зависит от чего-то ещё.

Даже генератор случайных чисел (настоящий, аппаратный) — это «чистая» функция,
зависящая от состояния источника энтропии.

Чтобы представить этот эффект чистой функцией,
надо всего лишь неявную зависимость сделать явной.

<center>![](../../../../../files/posts/2018-01-18/unstable-pure.svg)</center>

```haskell
p :: a -> r -> b
```

Для удобства рассуждений об эффектах удобно ввести синоним

```haskell
type Reader r b = r -> b
p :: a -> Reader r b
```

Обратите внимание, что тип `Reader r`
(конструктор типа `Reader`, частично применённый к одному аргументу)
принадлежит `Functor`, `Applicative`,
`Monad` и многим другим интересным и полезным классам.

### 2.2. Эффект множественности

<center>![](../../../../../files/posts/2018-01-18/many.svg)</center>

Здесь всё просто и очевидно.
Функция, дающая много ответов сразу — это функция,
имеющая единственный ответ-множество.

В Хаскеле есть тип `Set` для множеств,
но для моделирования эффекта множественности оказывается более удобным список —
`[]`.

<center>![](../../../../../files/posts/2018-01-18/many-pure.svg)</center>

```haskell
p :: a -> [b]
```

Обратите внимание, что тип `[]` (конструктор типа списка)
принадлежит `Functor`, `Applicative`,
`Monad` и многим другим интересным и полезным классам.

Посколько множество результатов может быть и пустое,
то множественность можно рассматривать как частный случай случай частичности, —
функция, возвращающая множество ответов,
может для некоторых значений аргумента вернуть пустое множество,
то есть не вернуть ни одного ответа.

Таким образом, тип `[]` реализует и эффект частичности.

## 3. Побочный эффект

<center>![](../../../../../files/posts/2018-01-18/side.svg)</center>

Побочный эффект — это просто неявный результат. Сделаем же неявное явным!

<center>![](../../../../../files/posts/2018-01-18/side-pure.svg)</center>

```haskell
p :: a -> (b, s)
```

Для удобства рассуждений об эффектах удобно ввести обёртку

```haskell
newtype Side s b = (b, s)
p :: a -> Side s b
```

Обратите внимание, что тип `Side s`
(конструктор типа `Side`, частично применённый к одному аргументу)
принадлежит `Functor`, `Applicative`,
`Monad` и многим другим интересным и полезным классам.

(На практике чаще применяется тип `Writer`, структурно идентичный,
но с более полезными свойствами.)

## 2 + 3. Эффект состояния

<center>![](../../../../../files/posts/2018-01-18/state.svg)</center>

Если соединить результат побочного эффекта и источник нестабильности,
из их комбинации (композиции) получается эффект состояния — процедура,
которая может и зависеть от текущего состояния «переменной»,
и задавать ей новое состояние.

Проведя рассуждения, аналогичные случаям `Reader` и `Side`, получим

<center>![](../../../../../files/posts/2018-01-18/state-pure.svg)</center>

```haskell
p :: a -> s -> (b, s)
```

Для удобства рассуждений об эффектах удобно ввести обёртку

```haskell
newtype State s b = s -> (b, s)
p :: a -> State s b
```

Не будет сюрпризом, что тип `State s`
(конструктор типа `State`, частично применённый к одному аргументу)
принадлежит `Functor`, `Applicative`,
`Monad` и многим другим интересным и полезным классам.

## 0. Отсутствие эффектов (продолжение)

Рассмотрим тип

```haskell
newtype Identity a = Identity a
```

Тип `Identity a` полностью аналогичен типу `a`.
То есть это своего рода функция `id`, только на уровне типов.

Тип `Identity` не может выражать никаких эффектов.
С другой стороны, можно сказать, что он выражает отсутствие эффектов.

Конечно же, конструктор типа `Identity` принадлежит `Functor`, `Applicative`,
`Monad` и многим другим интересным и полезным классам.

## Эффекты! Эффекты повсюду!

В Хаскеле есть специальный тип `IO`, реализующий сразу все возможные эффекты.
В нём можно прерывать программу, обмениваться данными с ресурсами,
не указанными явно в аргументах и возвращаемом значении.
В `IO` нет ограничений, доступен на чтение и запись весь мир,
в том числе ядерные ракеты.
Как если бы у нас был в программе объект типа `RealWorld` и мы могли бы изменять
его, как переменную под `State`.

Реализация `IO` не определена в спецификации языка.
Если вы заглянете в исходники стандартной библиотеки, скорее всего,
действительно увидите что-то подобное `State RealWorld`,
но эта реализация нужна только для внутренних нужд компилятора и пропадает при
компиляции,
так что верить ей не стоит.

Вы уже догадались, что конструктор типа `IO` принадлежит `Functor`,
`Applicative`, `Monad` и многим другим интересным и полезным классам.

## Заключение

| Чистота | Эффект | Тип |
|---------|--------|-----|
| Полная | Нет | `Identity` |
| Тотальность | Частичность | `Maybe`, `Except e`, `[]` |
| Детерминированность | Нестабильность | `Reader r` |
| Детерминированность | Множественность | `[]` |
| Их отсутствие | Побочные | `Side s`, `Writer w` |
| Детерминированность и отсутствие побочных | Состояние | `State s`
| Есть | Все | `IO` |

Все упомянутые типы встроены в язык или легко находятся в стандартной библиотеке
(пакеты `base`, `mtl`), кроме `Side`.
`Side` я придумал только для иллюстрации, но его легко представить как `State`,
ограниченный до операции `put`.

Обратите внимание, что все упомянутые типы принадлежат `Functor`, `Applicative`,
`Monad` и многим другим интересным и полезным классам
(разве что `Writer` с некоторыми ограничениями).

Существуют и другие типы, реализующие эти эффекты иными,
более сложными и полезными способами.
Они всегда являются аппликативными функторами и почти всегда монадами.

Если будет интерес читателей,
можно будет раскрыть подробности реализации эффектов данными типами в будущих
статьях.