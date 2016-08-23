#Rewriting Your Test Suite in Clojure in 24 hours

Эта история о том как я написал компилятор для автоматической трансляции сценариев тестирования [CircleCI](https://circleci.com/) состоящих из 14000 строк в другую библиотеку тестирования за 24 часа.

На сегодняшний день набор тестов [CircleCI](https://circleci.com/) возможно один из самых больших в мире Clojure. Наш серверный код на 100% Clojure, включая тесты, состоящие из 14000 строк, в 140 файлах, с 5000 ассертами. Без [распараллеливания](https://circleci.com/docs/how-parallelism-works/), выполнение занимает 40 минут.

На старте этого приключения все тесты были написаны на [Midje](https://github.com/marick/Midje), библиотека для BDD тестирования, что-то похожее на RSpec. Мы не были особо счастливы с Midje, и решили перейти на [clojure.test](http://richhickey.github.io/clojure/clojure.test-api.html), которая, возможно, самая широко используемая библиотека для тестирования. `clojure.test` проще и в ней меньше магии, больше экосистема инструментов и плагинов.

Очевидно, что не практично переписывать 5,000 тестов руками. Вместо этого мы решили использовать Clojure, чтобы переписать их автоматически, используя встроенные в Clojure функции манипулирования языком.

Clojure является гомоиконным, это значит, что все исходные файлы могут быть представленны в виде структуры данных. Наш транслятор переводит каждый тестовый файл в структуру данных Clojure. Затем, мы преобразуем код перед тем как записать его обратно на диск. Как только он записан, мы можем запустить тесты, и даже автоматически добавить файл обратно в систему контроля версий если тесты прошли, и все это не выходя из REPL.

##Чтение

Ключем ко всей этой операции является `read`. `read-string` втроенная в Clojure функция, которая принимает строку содержащую любой Clojure код, и возвращает его как структуру данных Clojure. Эту же самую функцию использует компилятор, когда загружает исходные файлы. Пример: `(read-string "[1 2 3]")` вернет `[1 2 3]`.

Мы используем `read` для превращения кода наших тестов в большой вложенный лист, который может быть изменен обычным кодом Clojure.

##Преобразование

Наши тесты были написаны с использованием `midje`, и мы хотим преобразовать их для использования с `clojure.test`. Пример теста использующего `midje`:

```
(ns circle.foo-test
  (:require [midje.sweet :refer :all]
            [circle.foo :as foo]))
(fact "foo works"
  (foo x) => 42)
```  

и преобразованная версия, использующая `clojure.test`:

```
(ns circle.foo-test
  (:require [clojure.test :refer :all]))

(deftest foo-works
  (is (= 42 (foo x))))
```

Преобразование включает замену:

- `midje.sweet` на `clojure.test` в ns форме

- `(fact "a test name"...)` на `(deftest a-test-name ...)`, потому что имена `clojure.test` переменные, а не строки

- `(foo x) => 42` на `(is (= 42 (foo x)))`

- мелкие детали, которые пока пропустим

Преобразование - это простой depth-first обход дерева:

```
(defn munge-form [form]
  (let [form (-> form
                 (replace-midje-sweet)
                 (replace-foo)
                 ...)]
    (cond
      (or (list? form)
          (vector? form)) (-> form
                              (replace-fact)
                              (replace-arrow)
                              (replace-bar)
                              ...
                              (map munge-form)))
      :else form))
```

Поведение `->` похоже на chaining в Ruby или JQuery, или как Bash’s pipes: передает результат вычисления вызова функции как аргумент в вызов следующей функции

Первая часть `(let [form ...])` берет Clojure form и вызывает в ней каждую функцию преобразования. Вторая часть берет список из forms–representing остальных Clojure выражений и функций–и также рекурсивно преобразует их.

Интересный процесс происходит в функциях замены. They’re all generally of the form:

```
(if (this-form-is-relevant? form)
  (some-transformation form)
  form)
```

i.e., they check to see if the form passed in is relevant to their interests, и если так, преобразует их нужным образом. Поэтому `replace-midje-sweet` выглядит как

```
(defn replace-midje-sweet [form]
  (if (= 'midje.sweet form)
    'clojure.test
    form))
```

##Arrows

Most of Midje’s testing behavior centers around “arrows”, an unidiomatic construct that Midje uses to implement a BDD style declarative test case. A simple example:

```
(foo 42) => 5
```

asserts that `(foo 42)` returns 5.

Depending on the arrow uses, and the types on either side of the arrow, there are a large number of possible behaviours.

```
(foo 42) => map?
```

As as example, if the right-hand side above is a function, this asserts that the result is truthy when passed to the function map?. In standard clojure this would be:

```
(map? (foo 42))
```

Here are a few examples of some midje arrows:

```
(foo 42) => falsey
(foo 42) => map?
(foo 42) => (throws Exception)
(foo 42) =not=> 3
(foo 42) => #"hello world" ;; regex
(foo 42) =not=> "hello"
```

####Replacing Arrows

The actual translation is handled by about 40 [core.match](https://github.com/clojure/core.match) rules. They look like

```
(match [actual arrow expected]
  [actual '=> 'truthy] `(is ~actual)
  [actual '=> expected] `(is (= ~expected ~actual)
  [actual '=> (_ :guard regex?)] `(is (re-find ~contents ~actual))
  [actual '=> nil] `(is (nil? ~actual)))
```

(For Clojure experts, I’ve elided a lot of ~’ characters in the macros above to improve readability. Read the actual source to see what this really looks like.)

Most of these translations are straightforward. However, things get significantly more complicated with the `contains` form:

```
(foo 42) => (contains {:a 1})
(foo 42) => (contains [:a :b] :gaps-ok)
(foo 42) => (contains [:a :b] :in-any-order)
(foo 42) => (contains "hello")
```

The last case is particularly interesting. In the expression

```
(foo 42) => (contains "hello")
```

There are two completely different values that could make this test pass. `(foo 42)` could be a list that contains the item “hello”, or it could be a string that contains the substring “hello”:

```
"hello world" => (contains "hello")
["foo" "hello" "bar"] => (contains "hello")
```

In general, the `contains` forms are difficult to translate automatically. Some cases require runtime information (like the last example), and because there’s no existing implementation for many of the `contains` cases in standard Clojure, such as `(contains [:a :b] :in-any-order)`, we decided to punt on all `contains` cases. The “fall through” rule looked like:

```
[actual arrow expected] (is (~arrow ~expected ~actual))
```

which turns `(foo 42) => (contains bar)` into `(is (=> (contains bar) (foo 42)))`. This intentionally doesn’t compile because midje’s arrow function definitions aren’t loaded, and so we can fix these up by hand instead.

####Runtime Type Information

There was one extra complication with automatic translation. If I have two expressions:

```
(let [bar 3]
  (foo) => bar
```

and

```
(let [bar clojure.core/map?]
  (foo) => bar
```

Midje’s arrow dispatch depends on the type of the right-hand expression, which can only (easily) be determined at runtime. If `bar` resolves to data, like a string, number, list or map, midje tests for equality. But if `bar` resolves to a function, midje instead _calls_ the function, i.e. `(is (= bar (foo)))` vs `(is (bar (foo)))`. Our 90% solution `require`s the original test namespace, and `resolve`s functions during the translation process:

```
(defn form-is-fn? [ns f]
  (let [resolved (ns-resolve ns f)]
    (and resolved (or (fn? resolved)
                      (and (var? resolved)
                           (fn? @resolved)))))))
```

This works great in most cases, but fails when a local variable shadows a global variable name:

```
(let [s [1 2 3]
      count (count s)]
  (foo s) => count)
```

In this case, we want `(is (= count (foo s)))`, but instead get `(is (count (foo s)))`, which fails because in the local scope, count is a number, and `(3 [1 2 3])` is an error. Thankfully there weren’t many of these situations, because solving this correctly would have required writing a full blown compiler with an understanding of the local variables in scope.

##Running the tests

Once the translation code was written, we needed a way to figure out if it worked. Since we run the code in a REPL at runtime, it’s trivial to instantly run tests after translation using `clojure.test`’s built-in functions.

`clojure.test`’s design decisions made it easy to tie together the translation and evaluation process. All test functions are callable directly from the REPL without shelling out, and `(clojure.test/run-all-tests)` even returns a meaningful return value, a map containing the number of tests, passes and fails:

```
{:pass 61, :test 21, :error 0, :fail 0}
```

Being able to run the tests from the REPL made it vey convenient to modify the compiler and retest in a very tight feedback loop.

####The Reader

Not everything worked quite so simply, however.

The “reader” (Clojure’s term for the part of the compiler that implement’s the `read` function) is designed to turn source files into data structures, primarily for consumption by the compiler. The reader strips comments, and expands macros, requiring us to review all diffs manually, and to revert lines that removed comments or contained macro definitions. Thankfully, there are only few of those in the tests. Our coding style tends to prefer docstrings over comments, and the macros tend to be isolated to a small set of utility files, so this didn’t affect us too much.

####Code Indenting

We didn’t find a very good library for indenting our new code idiomatically. We used `clojure.pprint`’s code mode, which although probably the best library out there, doesn’t do that great a job. We didn’t feel like writing that library during this project, so some files were written back to disk with non-idiomatic spacing and indentation. Now, when we work in a file with bad indention, we touch those up by hand. Really fixing this would have required an indenter that knows idiomatic formatting, and possibly respects file and line metadata on reader data.

There was a long delay between actually rewriting the test suite, and the publishing of this blog post. In the meantime, [rewrite-clj](https://github.com/xsc/rewrite-clj) has been released. I haven’t used it, but it looks like exactly what we were missing.

##Outcomes

About 40% of our test files passed without manual intervention, which is pretty amazing given how quickly we threw this solution together. In the remaining files, about 90% of test assertions translated and passed. So 94% of the assertions in all files could be translated automatically, a great result.

Our code is up on GitHub [here](https://github.com/circleci/translate-midje). Let [us](https://twitter.com/arohner) know if you end up using it. While we wouldn’t recommend it for unsupervised translation, especially because of the comment and macro issues, it worked great for [CircleCI](https://circleci.com/) as part of a supervised process.
