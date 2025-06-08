# Functional-Light JavaScript
# Глава 9: Операции над списками

> *Если ты можешь делать что-то крутое — делай это много раз подряд.*

Ранее в книге мы уже мельком встречали утилиты, которые теперь рассмотрим подробно: `map(..)`, `filter(..)` и `reduce(..)`. В JavaScript они реализованы как методы массива, но гораздо интереснее понять концепции, лежащие в их основе.

Перед тем как говорить про конкретные методы массива, важно понять, для чего вообще нужны такие операции. В этой главе не менее важно осознать, *почему* операции над списками так полезны, а не только *как* их реализовать.

В большинстве примеров (и не только в этой книге) такие операции показывают на тривиальных задачах — например, удваивание всех чисел в массиве. Не спешите пропускать эти примеры — на их основе вы поймёте, как моделировать цепочки задач, которые иначе пришлось бы выражать множеством переменных и промежуточных присваиваний.

Дело не только в том, чтобы писать более лаконичный код. Мы стремимся перейти от императивного стиля к декларативному, сделать паттерны кода более узнаваемыми — а значит, и более читаемыми.

Есть и ещё более важный момент. В императивном коде промежуточные результаты сохраняются в переменных через присваивание. Чем больше таких присваиваний, тем больше риск ошибок и сложнее следить за состоянием.

При объединении или композиции операций над списками промежуточные результаты отслеживаются неявно и гораздо лучше защищены от ошибок.

**Примечание:** В этой главе ради краткости кода будем часто использовать стрелочные функции ES6. Мои [замечания о стрелочных функциях из главы 2](ch2.md#functions-without-function) по-прежнему в силе.

## Не-FP-операции над массивами

Перед тем как начать, выделю несколько операций с массивами, которые похожи по виду, но не относятся к функциональным операциям над списками:

* `forEach(..)`
* `some(..)`
* `every(..)`

`forEach(..)` — просто инструмент для итерации, обычно используется для побочных эффектов; это противоречит духу FP.

`some(..)` и `every(..)` требуют чистых функций-предикатов (как и `filter(..)`), но в итоге возвращают единое булево значение — по сути, редуцируют список к true/false.

## map

Начнем с одной из самых базовых: `map(..)`.

Mapping — это преобразование одного значения в другое. Например, если вы умножаете 2 на 3 и получаете 6 — вы сделали mapping.

```js
var x = 2, y;

// преобразование (projection)
y = x * 3;

// мутация (reassignment)
x = x * 3;
```

Если вынести умножение в функцию, она становится mapping-функцией:

```js
var multiplyBy3 = v => v * 3;
var x = 2, y;
y = multiplyBy3(x);
```

`map(..)` расширяет эту идею на коллекции: преобразует каждый элемент списка в новый элемент, результат — новый список:

<p align="center">
    <img src="images/fig9.png" width="50%">
</p>

Реализация:

```js
function map(mapperFn, arr) {
    var newList = [];
    for (let [idx, v] of arr.entries()) {
        newList.push(
            mapperFn(v, idx, arr)
        );
    }
    return newList;
}
```

**Примечание:** Порядок аргументов `mapperFn, arr` выглядит не по-JS-овски, но так принято в FP, чтобы было удобнее делать композицию и каррирование.

`mapperFn(..)` получает не только значение, но и индекс, и исходный массив (как и в стандартном `Array.prototype.map`). Иногда это мешает — тогда используйте `unary(..)` (см. главу 3).

Пример с `parseInt(..)` и `unary(..)` (см. [главу 3](ch3.md#user-content-mapunary)):

```js
map(unary(parseInt), ["1","2","3"]);
// [1,2,3]
```

**Примечание:** В стандартных методах массива можно передать и thisArg для биндинга контекста — см. [главу 2](ch2.md#whats-this).

Примеры нетривиального использования map:

```js
var one = () => 1;
var two = () => 2;
var three = () => 3;

[one, two, three].map(fn => fn());
// [1,2,3]

var increment = v => ++v;
var decrement = v => --v;
var square = v => v * v;

var double = v => v * 2;

[increment, decrement, square]
.map(fn => compose(fn, double))
.map(fn => fn(3));
// [7,5,36]
```

В теории map может быть реализован параллельно (в языках, где нет побочных эффектов), но в JS нельзя, потому что нельзя гарантировать отсутствие эффектов.

### Синхронный vs асинхронный map

В этой главе рассматривается синхронный map, который работает на уже готовых массивах. Но есть и концепция — "ленивые списки" (streams), где map реагирует на появление новых значений.

Пример (фиктивный):

```js
var newArr = arr.map();
arr.addEventListener("value", multiplyBy3);
```

В FP такие структуры называются streams (или observable), их мы рассматривать не будем.

### map vs forEach

Иногда используют map вместо forEach, чтобы получить возвращаемый массив:

```js
[1,2,3,4,5]
.map(v => {
    console.log(v); // побочный эффект!
    return v;
});
```

Не делайте так: map предназначен для преобразования значений, а не для побочных эффектов. Используйте инструмент по назначению.

### Слово: функтор

В FP функтор — структура, у которой есть операция map, сохраняющая композицию. В JS массив — это функтор: у массива есть map, который применяет функцию к каждому элементу и возвращает новый массив.

Пример функтора-строки:

```js
function uppercaseLetter(c) {
    var code = c.charCodeAt(0);
    if (code >= 97 && code <= 122) {
        code = code - 32;
    }
    return String.fromCharCode(code);
}

function stringMap(mapperFn, str) {
    return [...str].map(mapperFn).join("");
}

stringMap(uppercaseLetter, "Hello World!");
// HELLO WORLD!
```

## filter

filter — отбирает из списка значения по предикату.

В быту мы говорим «фильтруем что-то ненужное», а в программировании filter возвращает те элементы, для которых функция-предикат возвращает true (т.е. «пропускает» их дальше).

```js
function filter(predicateFn, arr) {
    var newList = [];
    for (let [idx, v] of arr.entries()) {
        if (predicateFn(v, idx, arr)) {
            newList.push(v);
        }
    }
    return newList;
}
```

Пример с предикатом:

```js
var isOdd = v => v % 2 == 1;
[1,2,3,4,5].filter(isOdd); // [1,3,5]
```

Обратите внимание: привычнее сказать «отфильтровали чётные», но по сути мы фильтруем *внутрь* значения, для которых функция возвращает true.

Если хочется фильтровать *наружу* (отсеять), можно сделать утилиту filterOut:

```js
var filterIn = filter;

function filterOut(predicateFn, arr) {
    return filterIn(not(predicateFn), arr);
}
```

Теперь можно писать читаемое:

```js
filterIn(isOdd, [1,2,3,4,5]);   // [1,3,5]
filterOut(isEven, [1,2,3,4,5]); // [1,3,5]
```

## reduce

reduce («свёртка», «fold») сворачивает список к одному значению (числу, строке, объекту и др.).

```js
function reduce(reducerFn, initialValue, arr) {
    var acc, startIdx;
    if (arguments.length == 3) {
        acc = initialValue;
        startIdx = 0;
    }
    else if (arr.length > 0) {
        acc = arr[0];
        startIdx = 1;
    }
    else {
        throw new Error("Must provide at least one value.");
    }
    for (let idx = startIdx; idx < arr.length; idx++) {
        acc = reducerFn(acc, arr[idx], idx, arr);
    }
    return acc;
}
```

Пример: перемножение чисел с начальными значением:

```js
[5,10,15].reduce((product, v) => product * v, 3); // 2250
```

Пример: compose через reduce (см. [главу 4](ch4.md#user-content-composereduce)):

```js
function compose(...fns) {
    return function composed(result){
        return [...fns].reverse().reduce((result, fn) => fn(result), result);
    };
}
```

Существуют и правосторонние reduce (reduceRight), которые позволяют реализовать compose без reverse:

```js
function compose(...fns) {
    return function composed(result){
        return fns.reduceRight((result, fn) => fn(result), result);
    };
}
```

### map и filter через reduce

map:

```js
var double = v => v * 2;
[1,2,3,4,5].reduce((list, v) => (list.push(double(v)), list), []);
// [2,4,6,8,10]
```

filter:

```js
var isOdd = v => v % 2 == 1;
[1,2,3,4,5].reduce((list, v) => (isOdd(v) ? list.push(v) : undefined, list), []);
// [1,3,5]
```

## Продвинутые операции

### unique

Оставить только уникальные значения:

```js
var unique = arr =>
    arr.filter((v, idx) => arr.indexOf(v) == idx);
```

или через reduce:

```js
var unique = arr =>
    arr.reduce((list, v) => list.indexOf(v) == -1 ? (list.push(v), list) : list, []);
```

### flatten

Сделать одномерный массив из вложенного:

```js
var flatten = arr =>
    arr.reduce((list, v) =>
        list.concat(Array.isArray(v) ? flatten(v) : v), []);
```

Можно ограничить глубину:

```js
var flatten = (arr, depth = Infinity) =>
    arr.reduce((list, v) =>
        list.concat(
            depth > 0 ?
                (depth > 1 && Array.isArray(v) ? flatten(v, depth - 1) : v) :
                [v]
        ), []);
```

### flatMap

flatMap совмещает map и flatten (на 1 уровень):

```js
var flatMap = (mapperFn, arr) =>
    flatten(arr.map(mapperFn), 1);
```

или через reduce:

```js
var flatMap = (mapperFn, arr) =>
    arr.reduce((list, v) => list.concat(mapperFn(v)), []);
```

### zip

zip объединяет два списка по парам:

```js
function zip(arr1, arr2) {
    var zipped = [];
    arr1 = [...arr1];
    arr2 = [...arr2];
    while (arr1.length > 0 && arr2.length > 0) {
        zipped.push([arr1.shift(), arr2.shift()]);
    }
    return zipped;
}
```

### merge

mergeLists объединяет два списка чередованием:

```js
function mergeLists(arr1, arr2) {
    var merged = [];
    arr1 = [...arr1];
    arr2 = [...arr2];
    while (arr1.length > 0 || arr2.length > 0) {
        if (arr1.length > 0) merged.push(arr1.shift());
        if (arr2.length > 0) merged.push(arr2.shift());
    }
    return merged;
}
```

## Методы vs отдельные функции

В JS есть и цепочки методов (`arr.map().filter().reduce()`) и отдельные функции (`reduce(map(filter(arr, fn1), fn2), fn3, init)`). FP обычно предпочитает последний стиль.

Для композиции методов нужен `this`-aware compose:

```js
var partialThis = (fn, ...presetArgs) =>
    function partiallyApplied(...laterArgs) {
        return fn.apply(this, [...presetArgs, ...laterArgs]);
    };

var composeChainedMethods = (...fns) =>
    result =>
        fns.reduceRight(
            (result, fn) => fn.call(result),
            result
        );
```

Для stand-alone функций удобнее, когда массив передаётся последним аргументом и функции каррируются.

```js
var filter = curry((predicateFn, arr) => arr.filter(predicateFn));
var map = curry((mapperFn, arr) => arr.map(mapperFn));
var reduce = curry((reducerFn, initialValue, arr) => arr.reduce(reducerFn, initialValue));
```

## Моделирование цепочек

Переход от императивных цепочек к декларативным операциями над списками требует практики. Рассмотрите такую задачу:

```js
var getSessionId = partial(prop, "sessId");
var getUserId = partial(prop, "uId");

var session = getCurrentSession();
if (session != null) sessionId = getSessionId(session);
if (sessionId != null) user = lookupUser(sessionId);
if (user != null) userId = getUserId(user);
if (userId != null) orders = lookupOrders(userId);
if (orders != null) processOrders(orders);
```

Это можно переписать с помощью цепочки функций и guard:

```js
var guard = fn => arg => arg != null ? fn(arg) : arg;
[ getSessionId, lookupUser, getUserId, lookupOrders, processOrders ]
.map(guard)
.reduce((result, nextFn) => nextFn(result), getCurrentSession());
```

## Fusion

Часто встречаются длинные цепочки map, filter и reduce. Чтобы не проходить по массиву много раз, используют композицию функций (fusion):

```js
words
.map(compose(elide, upper, removeInvalidChars));
```

или

```js
words
.map(pipe(removeInvalidChars, upper, elide));
```

## За пределами массивов

Операции map/filter/reduce применимы не только к массивам, но и к любым структурам данных (например, деревьям).

Пример для бинарного дерева:

```js
var BinaryTree = (value, parent, left, right) => ({ value, parent, left, right });

BinaryTree.map = function map(mapperFn, node) { ... };
BinaryTree.reduce = function reduce(reducerFn, initialValue, node) { ... };
BinaryTree.filter = function filter(predicateFn, node) { ... };
```

## Итог

Три основных операции:
* `map(..)`: преобразует значения в новый список;
* `filter(..)`: выбирает/отсекает значения в новый список;
* `reduce(..)`: сводит список к одному значению.

Есть и более продвинутые операции: `unique(..)`, `flatten(..)`, `merge(..)`.

Fusion объединяет цепочки map для оптимизации и улучшения декларативности.

Операции над списками — это общий приём FP, применимый к любым структурам, а не только к массивам.
