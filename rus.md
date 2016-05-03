# Осваиваем трансдьюсеры в JavaScript

Я нашёл очень хорошую статью о трансдьюсерах: [«Осваиваем трансдьюсеры»][1].
Если вы знакомы с Clojure, советую прочитать её. Если же вам в диковинку
lisp-подобный код, не отчаивайтесь — я переписал примеры на языке JavaScript,
так что вы всё равно сможете прочесть статью, обращаясь сюда за понятными
вам примерами.

## Что такое трансдьюсеры?
Быстрое введение для чайников: трансдьюсеры — это эффективные компонуемые
функции для преобразования данных, не создающие промежуточных коллекций.

Взгляните на разницу между встроенными методами массивов и трансдьюсерами.
Встроенные трансформации, объединённые в цепочки, создают промежуточные
коллекции:

![Иллюстрация][Визуализация работы встроенных трансформаций массивов]

Трансдьюсеры последовательно обрабатывают каждый элемент и кладут его в итоговый
массив:

![Иллюстрация][Визуализация работы трансдьюсеров]

## Зачем их использовать?
Последняя визуализация показывает, что мы хотим скомпоновать трансформации вроде
`map`, `filter` и `reduce` и пропускать кусочки данных последовательно через
каждую из них. Но это не должно выглядеть так:


    array
      .map(fn1)
      .filter(fn2)
      .reduce(fn3);

Вместо этого мы должны получить что-то вроде:

    const transformation = compose(map(fn1), filter(fn2), reduce(fn3));

    transformation(array);

В таком случае мы сможем повторно использовать полученную трансформацию, а также
компоновать её с другими трансформациями. Проблема в том, что `map`, `filter` и `reduce` имеют
разную сигнатуру. Поэтому их нужно обобщить, и помочь в этом может `reduce`.

## Примеры кода из статьи

Как можно объединить `map` и `filter`:

    [0, 1, 2, 3, 4, 5, 6, 7, 8, 9].map((x) => x + 1);
    // [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

    [1, 2, 3, 4, 5, 6, 7, 8, 9, 10].filter((x) => x % 2 === 0);
    // [2, 4, 6, 8, 10]

    [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
      .map((x) => x + 1)
      .filter((x) => x % 2 === 0);
    // [2, 4, 6, 8, 10]

`map` и `filter` могут быть реализованы с помощью `reduce`. Реализация `map`:

    const mapIncReducer = (result, input) => {
      result.push(input + 1);
      return result;
    };

    [0, 1, 2, 3, 4, 5, 6, 7, 8, 9].reduce(mapIncReducer, []);
    // [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

Давайте извлечём функцию изменения значения, чтобы её можно было передать в редьюсер:

    const mapReducer = (f) => (result, input) => {
      result.push(f(input));
      return result;
    };

    [0, 1, 2, 3, 4, 5, 6, 7, 8, 9].reduce(mapReducer((x) => x + 1), []);
    // [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

Больше примеров использования `mapReducer`:

    [0, 1, 2, 3, 4, 5, 6, 7, 8, 9].reduce(mapReducer((x) => x - 1), []);
    // [-1, 0, 1, 2, 3, 4, 5, 6, 7, 8]

    [0, 1, 2, 3, 4, 5, 6, 7, 8, 9].reduce(mapReducer((x) => x * x), []);
    // [0, 1, 4, 9, 16, 25, 36, 49, 64, 81]

Реализация `filter` с помощью `reduce`:

    const filterEvenReducer = (result, input) => {
      if (input % 2 === 0) {
        result.push(input);
      }
      return result;
    };

    [1, 2, 3, 4, 5, 6, 7, 8, 9, 10].reduce(filterEvenReducer, []);
    // [2, 4, 6, 8, 10]

А теперь извлечём функцию-предикат, чтобы её можно было передавать снаружи:

    const filterReducer = (predicate) => (result, input) => {
      if (predicate(input)) {
        result.push(input);
      }
      return result;
    };

    [1, 2, 3, 4, 5, 6, 7, 8, 9, 10].reduce(filterReducer((x) => x % 2 === 0), []);
    // [2, 4, 6, 8, 10]

Комбинируем оба редьюсера:

    [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
      .reduce(mapReducer((x) => x + 1), [])
      .reduce(filterReducer((x) => x % 2 === 0), []);
    // [2, 4, 6, 8, 10]

Получилось практически так же, как если бы вы использовали нативные методы массивов:

    [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
      .map((x) => x + 1)
      .filter((x) => x % 2 === 0);
    // [2, 4, 6, 8, 10]

Давайте снова взглянем на наши редьюсеры. Они оба используют `Array.push()` в качестве
*сокращающей функции*:

    const mapReducer = (f) => (result, input) => {
      result.push(f(input));
      return result;
    };
    const filterReducer = (predicate) => (result, input) => {
      if (predicate(input)) {
        result.push(input);
      }
      return result;
    }

*Cокращающие функции* — это функции наподобие `push` и `+`: они принимают начальное значение
и входные данные, а затем сокращают их до одного выходного значения:

    array.push(4);
    // [1, 2, 3, 4]

    10 + 1
    // 11

Давайте извлечём нашу *сокращающую функцию*, чтобы её можно было передавать извне:

    const mapping = (f) => (reducing) => (result, input) => reducing(result, f(input));
    const filtering = (predicate) => (reducing) => (result, input) => predicate(input) ? reducing(result, input) : result;

Теперь наши редьюсеры можно использовать так:

    [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
      .reduce(mapping((x) => x + 1)((xs, x) => {
        xs.push(x);
        return xs;
      }), [])
      .reduce(filtering((x) => x % 2 === 0)((xs, x) => {
        xs.push(x);
        return xs;
      }), []);
    // [2, 4, 6, 8, 10]

Редьюсеры работают по принципу *результат, начальное значение → новый результат*:

    mapping((x) => x + 1)((xs, x) => {
      xs.push(x);
      return xs;
    })([], 1);
    // [2]

    mapping((x) => x + 1)((xs, x) => {
      xs.push(x);
      return xs;
    })([2], 2);
    // [2, 3]

    mapping((x) => x + 1)((xs, x) => {
      xs.push(x);
      return xs;
    })([2, 3], 3);
    // [2, 3, 4]

    filtering((x) => x % 2 === 0)((xs, x) => {
      xs.push(x);
      return xs;
    })([2, 4], 5);
    // [2, 4]

    filtering((x) => x % 2 === 0)((xs, x) => {
      xs.push(x);
      return xs;
    })([2, 4], 6);
    // [2, 4, 6] 

Композиция редьюсеров работает по тому же принципу:

    mapping((x) => x + 1)(filtering((x) => x % 2 === 0)((xs, x) => {
      xs.push(x);
      return xs;
    }));

…и она также может быть использована в качестве редьюсера:

    [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
      .reduce(
        mapping((x) => x + 1)(filtering((x) => x % 2 === 0)((xs, x) => {
          xs.push(x);
          return xs;
        })),
        []);
    // [2, 4, 6, 8, 10]

Давайте воспользуемся функцией `R.compose` из библиотеки Ramda для лучшей читабельности кода:

    const xform = R.compose(
      mapping((x) => x + 1),
      filtering((x) => x % 2 === 0));

    [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
      .reduce(xform((xs, x) => {
        xs.push(x);
        return xs;
      }), []);
    // [2, 4, 6, 8, 10]

Более сложный пример:

const square = (x) => x * x;

    const xform = R.compose(
      filtering((x) => x % 2 === 0),
      filtering((x) => x < 10),
      mapping(square),
      mapping((x) => x + 1));

    [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
      .reduce(xform((xs, x) => {
        xs.push(x);
        return xs;
      }), []);
    // [1, 5, 17, 37, 65]

Наконец, давайте обернём это в вспомогательную функцию `transduce`:

    const transduce = (xform, reducing, initial, input) => input.reduce(xform(reducing), initial);

Пример использования:

    const xform = R.compose(
      mapping((x) => x + 1),
      filtering((x) => x % 2 === 0));
      
    transduce(xform, (xs, x) => {
      xs.push(x);
      return xs;
    }, [], [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]);
    // [2, 4, 6, 8, 10]

    transduce(xform, (sum, x) => sum + x, 0, [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]);
    // 30

Взгляните на библиотеку [transducers-js][2], если вам нужна полная и производительная реализация
трансдьюсеров в JavaScript. Также советую почитать о [протоколе «Трансдьюсер»][3], позволяющем
взаимодействовать разным библиотекам (вроде Lodash, Underscore и Immutable.js). 


 [Визуализация работы встроенных трансформаций массивов]: img/array-methods-visually.gif "Визуализация работы встроенных трансформаций массивов"
 [Визуализация работы трансдьюсеров]: img/tranducers-visually.gif "Визуализация работы трансдьюсеров"
 [1]: http://elbenshira.com/blog/understanding-transducers/
 [2]: https://github.com/cognitect-labs/transducers-js
 [3]: https://github.com/cognitect-labs/transducers-js#the-transducer-protocol
