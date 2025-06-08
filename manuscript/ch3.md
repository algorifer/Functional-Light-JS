# Functional-Light JavaScript
# Глава 3: Управление входами функции

[Глава 2](ch2.md) исследовала суть функций в JS и заложила основы того, что делает функцию настоящей FP *функцией*. Но для раскрытия истинной мощи FP нам нужны паттерны и практики, которые помогут удобно и гибко управлять входами функций.

В этой главе мы сосредоточимся на параметрах функций. Когда вы будете комбинировать функции с разными сигнатурами в своих программах, вы быстро столкнётесь с несовместимостью по количеству/порядку входов.

Иногда, ради читаемости, вам даже захочется определить функцию так, чтобы полностью скрыть её входы!

Такие техники абсолютно необходимы, чтобы функции были по-настоящему *функциональными*.

## Всё ради одного

Представьте, что вы передаёте функцию в утилиту, которая будет вызывать её с несколькими аргументами, но вам нужно, чтобы функция получила только один из них.

Можно написать простой хелпер-обёртку, который гарантирует, что функция получит только один аргумент (превращая её в унарную):

<a name="unary"></a>

```js
function unary(fn) {
    return function onlyOneArg(arg){
        return fn(arg);
    };
}
```

FP-разработчики часто предпочитают короткую стрелочную запись:

```js
var unary =
    fn =>
        arg =>
            fn(arg);
```

**Примечание:** Да, это короче и больше похоже на математику, но, на мой вкус, теряется читаемость из-за лишней вложенности.

Типичный пример применения `unary(..)` — с методом `map(..)` и функцией `parseInt(..)`. `map(..)` вызывает функцию для каждого элемента массива, передавая второй аргумент — индекс. Но `parseInt(str, radix)` воспринимает второй аргумент как систему счисления, что нам не нужно:

```js
["1","2","3"].map(parseInt);
// [1,NaN,NaN]
```

Используем `unary(..)`:

<a name="mapunary"></a>

```js
["1","2","3"].map(unary(parseInt));
// [1,2,3]
```

### Один на один

Ещё один часто встречающийся хелпер — функция, которая просто возвращает переданное значение:

```js
function identity(v) {
    return v;
}

// или стрелочная форма
var identity = v => v;
```

Кажется, что польза минимальна, но в FP даже такие функции нужны. Например, чтобы отфильтровать не пустые строки:

```js
var words = "   Now is the time for all...  ".split(/\s|\b/);
// ["","Now","is","the","time","for","all","...",""]

words.filter(identity);
// ["Now","is","the","time","for","all","..."]
```

**Совет:** Аналогичную роль здесь может выполнять встроенная `Boolean(..)`.

Другой пример — использование `identity(..)` по умолчанию для преобразования:

```js
function output(msg, formatFn = identity) {
    msg = formatFn(msg);
    console.log(msg);
}

function upper(txt) {
    return txt.toUpperCase();
}

output("Hello World", upper);   // HELLO WORLD
output("Hello World");          // Hello World
```

Также `identity(..)` бывает полезна в качестве трансформера по умолчанию для `map(..)` или начального значения для `reduce(..)` с функциями.

### Неизменяемое одно

Иногда API требуют функцию, хотя вы хотите просто вернуть значение. Например, в промисах:

```js
// не сработает:
p1.then(foo).then(p2).then(bar);

// вместо этого:
p1.then(foo).then(function(){ return p2; }).then(bar);
```

Можно использовать стрелочную функцию:

```js
p1.then(foo).then(() => p2).then(bar);
```

Но FP-утилита для этого — `constant(..)`:

```js
function constant(v) {
    return function value() {
        return v;
    };
}

// или стрелочная версия
var constant = v => () => v;
```

Теперь:

```js
p1.then(foo).then(constant(p2)).then(bar);
```

**Внимание:** Несмотря на то, что `() => p2` короче, лучше использовать явную функцию — это делает намерения понятнее и проще для композиции.

## Адаптация аргументов к параметрам

Существуют разные трюки, как привести сигнатуру функции к нужному виду.

Вспомните пример с деструктуризацией массива:

```js
function foo([x, y, ...args] = []) {
    // ..
}
```

Но не всегда можно изменить объявление функции. Например:

```js
function foo(x, y) {
    console.log(x + y);
}

function bar(fn) {
    fn([3, 9]);
}

bar(foo); // ошибка
```

`foo(..)` ждёт два отдельных аргумента, а получает массив. Поможет хелпер `spreadArgs(..)`:

<a name="spreadargs"></a>

```js
function spreadArgs(fn) {
    return function spreadFn(argsArr){
        return fn(...argsArr);
    };
}

// стрелочная форма
var spreadArgs = fn => argsArr => fn(...argsArr);
```

Теперь:

```js
bar(spreadArgs(foo)); // 12
```

Обратная задача — собрать отдельные аргументы в массив:

```js
function gatherArgs(fn) {
    return function gatheredFn(...argsArr){
        return fn(argsArr);
    };
}

// стрелочная форма
var gatherArgs = fn => (...argsArr) => fn(argsArr);
```

Пример:

```js
function combineFirstTwo([v1, v2]) {
    return v1 + v2;
}

[1,2,3,4,5].reduce(gatherArgs(combineFirstTwo));
// 15
```

## Некоторые сейчас, некоторые потом

Если функция принимает несколько аргументов, иногда удобно часть из них зафиксировать заранее, а остальные передавать позже.

Например:

```js
function ajax(url, data, callback) {
    // ..
}
```

Создадим хелперы для частичного применения:

```js
function getPerson(data, cb) {
    ajax("http://some.api/person", data, cb);
}

function getOrder(data, cb) {
    ajax("http://some.api/order", data, cb);
}
```

Но ручное написание таких функций неудобно. Лучше использовать утилиту частичного применения:

```js
function partial(fn, ...presetArgs) {
    return function partiallyApplied(...laterArgs){
        return fn(...presetArgs, ...laterArgs);
    };
}

// стрелочная форма
var partial = (fn, ...presetArgs) => (...laterArgs) => fn(...presetArgs, ...laterArgs);
```

Теперь:

```js
var getPerson = partial(ajax, "http://some.api/person");
var getOrder = partial(ajax, "http://some.api/order");
```

Можно делать цепочки частичного применения:

```js
// вариант 1
var getCurrentUser = partial(ajax, "http://some.api/person", { user: CURRENT_USER_ID });

// вариант 2
var getCurrentUser = partial(getPerson, { user: CURRENT_USER_ID });
```

**Примечание:** Второй вариант создаёт дополнительный слой обёртки, но это нормально для FP.

Пример с `add(..)`:

```js
function add(x, y) {
    return x + y;
}

[1,2,3,4,5].map(partial(add, 3));
// [4,5,6,7,8]
```

### `bind(..)`

В JS есть встроенный метод `bind(..)`, который позволяет фиксировать контекст и аргументы:

```js
var getPerson = ajax.bind(null, "http://some.api/person");
```

Но большинству FP-программистов больше по душе явные утилиты типа `partial(..)`.

### Переворот аргументов

Если нужно зафиксировать не первые, а последние аргументы, можно использовать `reverseArgs(..)` или `partialRight(..)`:

```js
function reverseArgs(fn) {
    return function argsReversed(...args){
        return fn(...args.reverse());
    };
}

// стрелочная форма
var reverseArgs = fn => (...args) => fn(...args.reverse());
```

`partialRight(..)`:

<a name="partialright"></a>

```js
function partialRight(fn, ...presetArgs) {
    return function partiallyApplied(...laterArgs){
        return fn(...laterArgs, ...presetArgs);
    };
}

// стрелочная форма
var partialRight = (fn, ...presetArgs) => (...laterArgs) => fn(...laterArgs, ...presetArgs);
```

Пример:

```js
function foo(x, y, z, ...rest) {
    console.log(x, y, z, rest);
}

var f = partialRight(foo, "z:last");

f(1, 2);          // 1 2 "z:last" []
f(1);             // 1 "z:last" undefined []
f(1, 2, 3);       // 1 2 3 ["z:last"]
f(1, 2, 3, 4);    // 1 2 3 [4,"z:last"]
```

## По одному за раз

Техника, похожая на частичное применение, — каррирование (currying): функция, принимающая несколько аргументов, разбивается на цепочку вложенных функций, каждая из которых принимает по одному аргументу.

Пример использования каррированной функции:

```js
curriedAjax("http://some.api/person")
    ({ user: CURRENT_USER_ID })
        (function foundUser(user){ /* .. */ });
```

Или пошагово:

```js
var personFetcher = curriedAjax("http://some.api/person");
var getCurrentUser = personFetcher({ user: CURRENT_USER_ID });
getCurrentUser(function foundUser(user){ /* .. */ });
```

Реализация каррирования:

<a name="curry"></a>

```js
function curry(fn, arity = fn.length) {
    return (function nextCurried(prevArgs){
        return function curried(nextArg){
            var args = [ ...prevArgs, nextArg ];
            if (args.length >= arity) {
                return fn(...args);
            } else {
                return nextCurried(args);
            }
        };
    })([]);
}

// стрелочная форма
var curry = (fn, arity = fn.length, nextCurried) =>
    (nextCurried = prevArgs =>
        nextArg => {
            var args = [ ...prevArgs, nextArg ];
            if (args.length >= arity) {
                return fn(...args);
            } else {
                return nextCurried(args);
            }
        }
    )([]);
```

Пример использования:

```js
var curriedAjax = curry(ajax);
var personFetcher = curriedAjax("http://some.api/person");
var getCurrentUser = personFetcher({ user: CURRENT_USER_ID });
getCurrentUser(function foundUser(user){ /* .. */ });
```

Каррирование удобно для композиции функций, поскольку каждая функция унарная.

Пример с суммой:

```js
function sum(...nums) {
    var total = 0;
    for (let num of nums) {
        total += num;
    }
    return total;
}

var curriedSum = curry(sum, 5);
curriedSum(1)(2)(3)(4)(5); // 15
```

В отличие от частичного применения, каррирование позволяет создавать специализированные функции на каждом этапе.

### Визуализация каррированных функций

Вот как вручную реализовать каррированную сумму:

```js
function curriedSum(v1) {
    return function(v2){
        return function(v3){
            return function(v4){
                return function(v5){
                    return sum(v1, v2, v3, v4, v5);
                };
            };
        };
    };
}
```

Или стрелочная запись:

```js
curriedSum = v1 => v2 => v3 => v4 => v5 => sum(v1, v2, v3, v4, v5);
```

### Зачем каррирование и частичное применение?

Они позволяют задавать аргументы в разное время/месте и делают функции более гибкими для композиции.

Кроме того, в случае каррирования композиция упрощается, так как все промежуточные функции — унарные.

Главное — это возможность создавать специализированные функции из более общих, что повышает читаемость и повторное использование.

### Каррирование сразу нескольких аргументов?

В JS часто встречается "слабое" каррирование, когда за один вызов можно передать несколько аргументов:

<a name="loosecurry"></a>

```js
function looseCurry(fn, arity = fn.length) {
    return (function nextCurried(prevArgs){
        return function curried(...nextArgs){
            var args = [ ...prevArgs, ...nextArgs ];
            if (args.length >= arity) {
                return fn(...args);
            } else {
                return nextCurried(args);
            }
        };
    })([]);
}
```

Теперь можно писать:

```js
var curriedSum = looseCurry(sum, 5);
curriedSum(1)(2, 3)(4, 5); // 15
```

### Раскаррирование

Иногда нужно превратить каррированную функцию обратно в обычную:

```js
function uncurry(fn) {
    return function uncurried(...args){
        var ret = fn;
        for (let arg of args) {
            ret = ret(arg);
        }
        return ret;
    };
}

// стрелочная форма
var uncurry = fn => (...args) => {
    var ret = fn;
    for (let arg of args) {
        ret = ret(arg);
    }
    return ret;
};
```

Пример:

```js
function sum(...nums) {
    var sum = 0;
    for (let num of nums) {
        sum += num;
    }
    return sum;
}

var curriedSum = curry(sum, 5);
var uncurriedSum = uncurry(curriedSum);

curriedSum(1)(2)(3)(4)(5);         // 15
uncurriedSum(1, 2, 3, 4, 5);       // 15
uncurriedSum(1, 2, 3)(4)(5);       // 15
```

**Внимание:** Не всегда `uncurry(curry(f))` будет вести себя как оригинальная функция.

## Порядок важен

В [главе 2](ch2.md#named-arguments) мы обсуждали именованные аргументы. Их плюс — не надо следить за порядком.

Каррирование и частичное применение традиционно завязаны на порядок, что не всегда удобно. Можно написать утилиты для работы с объектами-аргументами:

```js
function partialProps(fn, presetArgsObj) {
    return function partiallyApplied(laterArgsObj){
        return fn(Object.assign({}, presetArgsObj, laterArgsObj));
    };
}

function curryProps(fn, arity = 1) {
    return (function nextCurried(prevArgsObj){
        return function curried(nextArgObj = {}){
            var [key] = Object.keys(nextArgObj);
            var allArgsObj = Object.assign({}, prevArgsObj, { [key]: nextArgObj[key] });
            if (Object.keys(allArgsObj).length >= arity) {
                return fn(allArgsObj);
            } else {
                return nextCurried(allArgsObj);
            }
        };
    })({});
}
```

Теперь порядок не важен:

```js
function foo({ x, y, z } = {}) {
    console.log(`x:${x} y:${y} z:${z}`);
}

var f1 = curryProps(foo, 3);
var f2 = partialProps(foo, { y: 2 });

f1({ y: 2 })({ x: 1 })({ z: 3 }); // x:1 y:2 z:3
f2({ z: 3, x: 1 });               // x:1 y:2 z:3
```

**Совет:** Если вам нравится этот стиль, посмотрите [приложение C, FPO](apC.md#bonus-fpo).

### Spreading Properties

Иногда хочется использовать такой подход и для функций с позиционными параметрами. Можно использовать `spreadArgProps(..)`:

```js
function spreadArgProps(
    fn,
    propOrder =
        fn.toString()
        .replace(/^(?:(?:function.*\(([^]*?)\))|(?:([^\(\)]+?)\s*=>)|(?:\(([^]*?)\)\s*=>))[^]+$/, "$1$2$3")
        .split(/\s*,\s*/)
        .map(v => v.replace(/[=\s].*$/, ""))
) {
    return function spreadFn(argsObj){
        return fn(...propOrder.map(k => argsObj[k]));
    };
}
```

Пример:

```js
function bar(x, y, z) {
    console.log(`x:${x} y:${y} z:${z}`);
}

var f3 = curryProps(spreadArgProps(bar), 3);
var f4 = partialProps(spreadArgProps(bar), { y: 2 });

f3({ y: 2 })({ x: 1 })({ z: 3 }); // x:1 y:2 z:3
f4({ z: 3, x: 1 });               // x:1 y:2 z:3
```

Имейте в виду, что вам нужно знать точные имена параметров функции.

## Без точек

В FP популярен стиль кодирования, где лишние параметры-аргументы убираются. Это называется tacit programming или point-free style.

**Внимание:** Не стоит стремиться к point-free на 100% — это не всегда полезно.

Пример:

```js
function double(x) {
    return x * 2;
}

[1,2,3,4,5].map(function mapper(v){
    return double(v);
});
// [2,4,6,8,10]

// point-free:
[1,2,3,4,5].map(double);
// [2,4,6,8,10]
```

Рассмотрим пример с parseInt:

```js
["1","2","3"].map(unary(parseInt));
// [1,2,3]
```

Пример с помощью not/complement:

```js
function output(txt) {
    console.log(txt);
}

function printIf(predicate, msg) {
    if (predicate(msg)) {
        output(msg);
    }
}

function isShortEnough(str) {
    return str.length <= 5;
}

var msg1 = "Hello";
var msg2 = msg1 + " World";

printIf(isShortEnough, msg1); // Hello
printIf(isShortEnough, msg2);

function not(predicate) {
    return function negated(...args){
        return !predicate(...args);
    };
}

var isLongEnough = not(isShortEnough);
printIf(isLongEnough, msg1);
printIf(isLongEnough, msg2); // Hello World
```

Можно сделать `printIf` point-free с помощью when и uncurry:

```js
function when(predicate, fn) {
    return function conditional(...args){
        if (predicate(...args)) {
            return fn(...args);
        }
    };
}

var printIf = uncurry(partialRight(when, output));

printIf(isShortEnough, msg1); // Hello
printIf(isLongEnough, msg2);  // Hello World
```

**Заметка:** Больше практики с point-free — в [главе 4](ch4.md#revisiting-points).

## Итоги

Частичное применение уменьшает арность функции, фиксируя часть аргументов.

Каррирование — частный случай частичного применения, когда функция превращается в цепочку унарных функций.

Хелперы вроде `unary(..)`, `identity(..)`, `constant(..)` — основа FP-инструментария.

Point-free — стиль, уменьшающий лишнюю "болтовню" между параметрами и аргументами.

Все эти техники помогают согласовать функции, чтобы их было проще комбинировать. В следующей главе мы научимся их объединять для моделирования потока данных!
