# Functional-Light JavaScript
# Глава 4: Композиция функций

К этому моменту, надеюсь, вы чувствуете себя гораздо увереннее в использовании функций в функциональном программировании.

Функциональный программист смотрит на каждую функцию в своей программе как на небольшой элемент конструктора Lego. Он узнаёт синий кирпичик 2х2 с первого взгляда, знает, как он работает и что с ним можно сделать. Но иногда вы соединяете синий 2х2 и серый 4х1, и вдруг понимаете: "это полезная конструкция, она мне часто нужна!"

Теперь у вас появляется новый "кирпичик" — комбинация двух других, и вы можете брать его всякий раз, когда он вам понадобился. Это эффективнее — распознавать и использовать такие составные блоки, чем каждый раз собирать их заново.

Функции бывают разных форм и размеров. И мы можем комбинировать их определённым образом, чтобы получить новую составную функцию, полезную в разных частях программы. Этот процесс создания новых функций из комбинаций других называется композицией.

Композиция — это то, как FP-разработчик моделирует поток данных в программе. В каком-то смысле, это самое фундаментальное понятие во всём FP, ведь без него нельзя декларативно моделировать данные и состояние.

## Выход в вход

Мы уже видели несколько примеров композиции. Например, в [обсуждении `unary(..)` в главе 3](ch3.md/#user-content-unary) был такой пример: [`[..].map(unary(parseInt))`](ch3.md/#mapunary).

Чтобы скомбинировать две функции, передайте результат вызова первой как аргумент для второй. В `map(unary(parseInt))` вызов `unary(parseInt)` возвращает функцию; эта функция передаётся в `map(..)`. Результат `map(..)` — это новый массив.

Чтобы визуализировать поток данных, представьте:

```txt
arrayValue <-- map <-- unary <-- parseInt
```

`parseInt` — вход для `unary(..)`. Результат `unary(..)` — вход для `map(..)`. Результат `map(..)` — `arrayValue`. Это и есть композиция.

**Примечание:** Здесь направление справа-налево выбрано специально, хотя поначалу это может казаться странным. Мы вернёмся к этому позже.

Думайте о потоке данных как о конвейере на фабрике конфет: каждая операция — это этап охлаждения, нарезки, упаковки сладостей. Мы будем использовать аналогию с фабрикой конфет и дальше.

<p align="center">
    <img src="images/fig2.png">
</p>

Рассмотрим композицию на примере. Такие две утилиты могут быть у вас в программе:

```js
function words(str) {
    return String(str)
        .toLowerCase()
        .split(/\s|\b/)
        .filter(function alpha(v){
            return /^[\w]+$/.test(v);
        });
}

function unique(list) {
    var uniqList = [];
    for (let v of list) {
        if (uniqList.indexOf(v) === -1) {
            uniqList.push(v);
        }
    }
    return uniqList;
}
```

`words(..)` разбивает строку на массив слов. `unique(..)` фильтрует этот массив, чтобы остались только уникальные слова.

Чтобы использовать их для анализа текста:

```js
var text = "To compose two functions together, pass the \
output of the first function call as the input of the \
second function call.";

var wordsFound = words(text);
var wordsUsed = unique(wordsFound);

wordsUsed;
// ["to","compose","two","functions","together","pass",
// "the","output","of","first","function","call","as",
// "input","second"]
```

Массив, который возвращает `words(..)`, мы называем `wordsFound`. Это же передаём в `unique(..)`.

Вернёмся к фабрике: первая машина берёт расплавленный шоколад и на выходе даёт охлаждённую плитку. Следующая — обрабатывает её дальше, и так далее.

<img src="images/fig3.png" align="right" width="9%" hspace="20">

Фабрика успешно работает, но руководство хочет увеличить производительность и решает убрать конвейер, просто поставив все три машины друг на друга — так, чтобы выход одной соединялся с входом следующей.

Такое нововведение экономит место, и теперь фабрика делает больше конфет в день!

Аналог в коде — мы пропускаем промежуточный шаг (переменную `wordsFound`) и просто объединяем вызовы функций:

```js
var wordsUsed = unique(words(text));
```

**Примечание:** Обычно мы читаем вызовы функций слева направо — сначала `unique(..)`, потом `words(..)`, — но на самом деле порядок вычислений идёт справа налево: сначала `words(..)`, потом `unique(..)`.

Составленные машины работают, но провода мешают. Чем больше комбинаций, тем сложнее разбираться.

<img src="images/fig4.png" align="left" width="15%" hspace="20">

Однажды инженер предлагает обернуть всё в единую коробку: "давайте все три машины спрячем внутрь, а снаружи будет только вход и выход".

Такую коробку удобно перемещать и использовать где угодно. Работникам теперь не нужно возиться с кнопками и проводами.

В коде это выглядит так: мы создаём функцию, которая объединяет `words(..)` и `unique(..)` в нужном порядке:

```js
function uniqueWords(str) {
    return unique(words(str));
}
```

`uniqueWords(..)` — композиция двух функций; поток данных:

```txt
wordsUsed <-- unique <-- words <-- text
```

Вы уже узнаёте: революция на фабрике — это композиция функций.

### Машины, строящие машины

Фабрика успешно работает и теперь может делать новые виды конфет. Но инженеры тратят много времени на создание новых "коробок", комбинируя разные машины.

Они обращаются к поставщику и узнают, что тот может продать им **машину для создания новых машин**! Они покупают устройство, которое автоматически строит составные машины из любых других.

<p align="center">
    <img src="images/fig5.png" width="50%">
</p>

В коде аналог — утилита `compose2(..)`, автоматически создающая композицию двух функций:

```js
function compose2(fn2, fn1) {
    return function composed(origValue){
        return fn2(fn1(origValue));
    };
}

// или стрелочная запись
var compose2 =
    (fn2, fn1) =>
        origValue =>
            fn2(fn1(origValue));
```

Обратите внимание: параметры идут как `fn2, fn1`, и сначала вызывается `fn1`, затем `fn2`. Это принято во многих FP-библиотеках: композиция идёт справа налево.

Это удобно, потому что функции перечислены в том порядке, в каком они идут при обычном (ручном) вызове: `unique(words(str))` — значит, в `compose2` — `unique, words`.

Теперь более эффективная версия нашей машины:

```js
var uniqueWords = compose2(unique, words);
```

### Вариации композиции

Комбинировать функции можно и в другом порядке. Например:

```js
var letters = compose2(words, unique);

var chars = letters("How are you Henry?");
chars;
// ["h","o","w","a","r","e","y","u","n"]
```

Это работает потому, что `words(..)` сначала приводит вход к строке. Но не всегда композиции бывают односторонними!

Фабрика должна быть осторожна: если попробовать "запихнуть" уже упакованные конфеты в машину для замешивания шоколада — получится катастрофа.

## Общая композиция

Если можно объединять две функции, можно и больше. Поток данных для нескольких функций:

```txt
finalValue <-- func1 <-- func2 <-- ... <-- funcN <-- origValue
```

<p align="center">
    <img src="images/fig6.png" width="50%">
</p>

Теперь у фабрики есть супер-машина, которая комбинирует сколько угодно этапов.

Утилита для такой композиции:

<a name="generalcompose"></a>

```js
function compose(...fns) {
    return function composed(result){
        var list = [...fns];
        while (list.length > 0) {
            result = list.pop()(result);
        }
        return result;
    };
}

// или стрелочная запись
var compose =
    (...fns) =>
        result => {
            var list = [...fns];
            while (list.length > 0) {
                result = list.pop()(result);
            }
            return result;
        };
```

**Внимание:** `[...fns]` нужен, чтобы не изменять исходный массив функций.

Пример композиции трёх функций:

```js
function skipShortWords(words) {
    var filteredWords = [];
    for (let word of words) {
        if (word.length > 4) {
            filteredWords.push(word);
        }
    }
    return filteredWords;
}
```

```js
var text = "To compose two functions together, pass the \
output of the first function call as the input of the \
second function call.";

var biggerWords = compose(skipShortWords, unique, words);

var wordsUsed = biggerWords(text);
wordsUsed;
// ["compose","functions","together","output","first",
// "function","input","second"]
```

Можно добавить частичное применение (см. `partialRight(..)`):

```js
function skipLongWords(list) { /* .. */ }

var filterWords = partialRight(compose, unique, words);

var biggerWords = filterWords(skipShortWords);
var shorterWords = filterWords(skipLongWords);

biggerWords(text);
// ["compose","functions","together","output","first",
// "function","input","second"]

shorterWords(text);
// ["to","two","pass","the","of","call","as"]
```

Частичное применение позволяет заранее зафиксировать часть композиции, а потом создавать вариации.

Также можно использовать `curry(..)`, иногда с `reverseArgs(compose)`.

**Примечание:** Так как `curry(..)` определяет арность по `length`, с композициями иногда нужно явно указывать количество аргументов.

### Альтернативные реализации

Понимание разных реализаций `compose(..)` важно для общего понимания FP.

Можно реализовать через `reduce(..)` (подробнее о нем в главе 9):

<a name="composereduce"></a>

```js
function compose(...fns) {
    return function composed(result){
        return [...fns].reverse().reduce(function reducer(result, fn){
            return fn(result);
        }, result);
    };
}

// или стрелочная форма
var compose = (...fns) =>
    result =>
        [...fns].reverse().reduce(
            (result, fn) => fn(result),
            result
        );
```

В этой версии первый аргумент принимает только один параметр. Чтобы исправить это, используем обёртку:

```js
function compose(...fns) {
    return fns.reverse().reduce(function reducer(fn1, fn2){
        return function composed(...args){
            return fn2(fn1(...args));
        };
    });
}
```

В этой реализации `reduce(..)` вызывается один раз, а все вычисления — при вызове итоговой функции.

Можно реализовать и рекурсивно (подробнее — в главе 8):

```js
function compose(...fns) {
    var [fn1, fn2, ...rest] = fns.reverse();
    var composedFn = function composed(...args){
        return fn2(fn1(...args));
    };
    if (rest.length == 0) return composedFn;
    return compose(...rest.reverse(), composedFn);
}
```

Каждая реализация имеет свои плюсы и минусы; выбирайте по ситуации.

## Композиция слева направо

Стандартные реализации `compose(..)` работают справа налево. Это удобно для соответствия обычному порядку записи вложенных вызовов, но иногда читается не так очевидно.

Компоновка слева направо называется `pipe(..)`. Это термин из Unix, где программы соединяются через "|".

```js
function pipe(...fns) {
    return function piped(result){
        var list = [...fns];
        while (list.length > 0) {
            result = list.shift()(result);
        }
        return result;
    };
}
```

Или так:

```js
var pipe = reverseArgs(compose);
```

Например:

```js
var biggerWords = compose(skipShortWords, unique, words);
// то же самое через pipe:
var biggerWords = pipe(words, unique, skipShortWords);
```

Преимущество — функции перечислены в порядке исполнения.

Полезно, если хочется частично применить первую функцию — тогда используем `partial(pipe, words, unique)`.

## Абстракция

Абстракция — ключевой момент при работе с композицией.

Как и частичное применение/каррирование (см. [главу 3](ch3.md/#some-now-some-later)), композиция позволяет выделить общее, спрятать детали, сделать код читаемее и переиспользуемым.

Пример:

```js
function saveComment(txt) {
    if (txt != "") {
        comments[comments.length] = txt;
    }
}

function trackEvent(evt) {
    if (evt.name !== undefined) {
        events[evt.name] = evt;
    }
}
```

Обе функции сохраняют данные в хранилище — различается только "куда" и "как". Вынесем общее:

```js
function storeData(store, location, value) {
    store[location] = value;
}

function saveComment(txt) {
    if (txt != "") {
        storeData(comments, comments.length, txt);
    }
}

function trackEvent(evt) {
    if (evt.name !== undefined) {
        storeData(events, evt.name, evt);
    }
}
```

Так мы избегаем дублирования (DRY). Но абстракция может быть чрезмерной:

```js
function conditionallyStoreData(store, location, value, checkFn) {
    if (checkFn(value, store, location)) {
        store[location] = value;
    }
}
```

Иногда чрезмерная общность усложняет код больше, чем упрощает.

**Главное:** абстракция — не способ скрыть детали, а способ отделить их, чтобы было проще думать о каждом слое по отдельности.

Пример декларативной абстракции:

```js
function getData() {
    return [1,2,3,4,5];
}

// императивно
var tmp = getData();
var a = tmp[0];
var b = tmp[3];

// декларативно
var [a,,,b] = getData();
```

Здесь мы явно показываем *что* хотим получить, а не *как* это делается.

### Композиция как абстракция

Композиция — это тоже декларативная абстракция.

Пример:

```js
// императивно
function shorterWords(text) {
    return skipLongWords(unique(words(text)));
}

// декларативно
var shorterWords = compose(skipLongWords, unique, words);
```

В декларативном варианте мы видим только *что* делаем с данными, не вникая в детали.

Даже если композиция используется только в одном месте, она делает код понятнее, отделяя *как* от *что*.

## Возвращаемся к point-free

Пора снова взглянуть на point-free-стиль из [главы 3](ch3.md/#no-points):

```js
// дано: ajax(url, data, cb)

var getPerson = partial(ajax, "http://some.api/person");
var getLastOrder = partial(ajax, "http://some.api/order", { id: -1 });

getLastOrder(function orderFound(order){
    getPerson({ id: order.personId }, function personFound(person){
        output(person.name);
    });
});
```

Хотим избавиться от промежуточных переменных `order` и `person`.

Для этого определим:

```js
function extractName(person) {
    return person.name;
}
```

Аналогично, можно создать универсальную функцию извлечения свойства:

```js
function prop(name, obj) {
    return obj[name];
}
// стрелочная форма
var prop = (name, obj) => obj[name];
```

Чтобы не мутировать объект, для установки свойства используем клон:

<a name="setprop"></a>

```js
function setProp(name, obj, val) {
    var o = Object.assign({}, obj);
    o[name] = val;
    return o;
}
```

Теперь можем сделать функцию `extractName` через частичное применение:

```js
var extractName = partial(prop, "name");
```

Следующий шаг — создать функцию, которая будет принимать объект, извлекать `personId`, оборачивать его в `{ id: .. }`, и вызывать `getPerson`, а результат передавать в `output`.

Для этого определим:

```js
var extractPersonId = partial(prop, "personId");

function makeObjProp(name, value) {
    return setProp(name, {}, value);
}
var personData = partial(makeObjProp, "id");
```

Соберём композицию:

```js
var outputPersonName = compose(output, extractName);
var processPerson = partialRight(getPerson, outputPersonName);
var lookupPerson = compose(processPerson, personData, extractPersonId);
```

Всё вместе:

```js
var getPerson = partial(ajax, "http://some.api/person");
var getLastOrder = partial(ajax, "http://some.api/order", { id: -1 });

var extractName = partial(prop, "name");
var outputPersonName = compose(output, extractName);
var processPerson = partialRight(getPerson, outputPersonName);
var personData = partial(makeObjProp, "id");
var extractPersonId = partial(prop, "personId");
var lookupPerson = compose(processPerson, personData, extractPersonId);

getLastOrder(lookupPerson);
```

Вот это да! Point-free. Композиция помогла нам избавиться от "точек" и сделать код декларативным и читаемым.

Можно сделать короче, но менее понятно:

```js
partial(ajax, "http://some.api/order", { id: -1 })(
    compose(
        partialRight(
            partial(ajax, "http://some.api/person"),
            compose(output, partial(prop, "name"))
        ),
        partial(makeObjProp, "id"),
        partial(prop, "personId")
    )
);
```

## Итог

Композиция функций — это способ связать вывод одной функции со входом другой, и так далее.

Поскольку функции JS возвращают только одно значение, все функции в композиции (кроме, возможно, первой) должны быть унарными.

Вместо того чтобы детализировать каждый шаг, мы можем использовать утилиту типа `compose(..)` или `pipe(..)` и сделать код более читаемым и декларативным.

Композиция — это декларативный поток данных: наш код явно и прозрачно описывает, как движутся данные.

Во многом композиция — самый важный паттерн FP, ведь это единственный способ управлять потоком данных без побочных эффектов. В следующей главе мы подробнее поговорим о чистоте функций.

----

<a name="footnote-1"><sup>1</sup></a>Scott, Michael L. “Chapter 3: Names, Scopes, and Bindings.” Programming Language Pragmatics, 4th ed., Morgan Kaufmann, 2015, pp. 115.
