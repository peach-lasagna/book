## Замыкания: анонимные функции, которые могут захватывать окружение

Замыкания Rust - это анонимные функции, которые вы можете сохранить в переменной или передать в качестве аргументов другим функциям. Вы можете создать замыкание в одном месте, а затем вызвать замыкание, чтобы вычислить его в другом контексте. В отличие от функций, замыкания могут захватывать значения из области видимости где они определены. Мы продемонстрируем, как эти возможности замыканий позволяют повторно использовать код и настраивать поведение.

### Создание обобщённого поведения используя замыкания

Рассмотрим пример демонстрирующий ситуацию, где сохранение замыкания удобно для его более позднего выполнения. Мы также поговорим про синтаксис замыканий, выведение типов и типажи.

Рассмотрим гипотетическую ситуацию, что мы работаем в стартапе, где создаём приложение для генерации индивидуальных планов тренировок. Серверная часть приложения создаётся на Rust и алгоритм генерирующий план тренировки, учитывает многие различные факторы, такие как возраст пользователя приложения, индекс массы тела, предпочтительные задания, последние тренировки и индекс интенсивности, которые указываются. При проектировании приложения конкретные алгоритмы реализаций не важны. Важно, чтобы различные расчёты не занимали много времени. Мы хотим использовать этот алгоритм только когда нам нужно, и делать это только один раз, чтобы не заставлять пользователя ждать больше, чем требуется.

Мы будем эмулировать работу алгоритма расчёта параметров с помощью функции `simulated_expensive_calculation` листинга 13-1, которая печатает `calculating slowly...`, ждёт две секунды и возвращает любое переданное ему число как результат эмулированного расчёта.

<span class="filename">Файл: src/main.rs</span>

```rust
{{#rustdoc_include ../listings/ch13-functional-features/listing-13-01/src/main.rs:here}}
```

<span class="caption">Листинг 13-1: Функция которая выполняет гипотетические расчёты длительностью в 2 секунды</span>

Теперь рассмотрим содержание функции `main`, которая содержит важные для этого примера части нашего приложения. Данная функция представляет код, который будет вызван, когда пользователь запросит свой план занятий. Так как взаимодействие с клиентской частью программы не связано с использованием замыканий, мы жёстко закодируем входные данные и выводимые на печать результаты.

Требуемые входные данные следующие:

- индекс интенсивности (intensity) пользователя, которое указывается пользователем, когда он запрашивает тренировку: показывает хотят ли они тренировку низкой интенсивности или тренировку высокой интенсивности,
- случайное число, которое будет генерировать разнообразие в планах тренировок.

В результате программа напечатает рекомендованный план занятий.  Листинг 13-2 показывает код использованной функции `main`.

<span class="filename">Файл: src/main.rs</span>

```rust
{{#rustdoc_include ../listings/ch13-functional-features/listing-13-02/src/main.rs:here}}
```

<span class="caption">Листинг 13-2: Функция <code>main</code> содержащая симуляцию пользовательского ввода данных и генерацию случайного числа</span>

Мы для простоты жёстко закодировали в коде значение переменной `simulated_user_specified_value` равным 10 и переменной `simulated_random_number` равным 7. В реальном приложении, мы бы получали значение интенсивности от пользователя и использовали бы пакет `rand` для генерации случайного числа, как мы делали в примере игры «Угадай число» из Главы 2. Функция `main` вызывает функцию`generate_workout` с эмулированными входными значениями.

Теперь когда есть контекст в котором мы будем работать, давайте займёмся алгоритмом. Функция `generate_workout` в листинге 13-3 содержит всю основную логику работу программы, которая наиболее важна в примере. Остальные изменения в коде будут сделаны внутри этой функции:

<span class="filename">Файл: src/main.rs</span>

```rust
{{#rustdoc_include ../listings/ch13-functional-features/listing-13-03/src/main.rs:here}}
```

<span class="caption">Листинг 13-3: Логика программы печатающей план тренировки на основании введённых данных и вызова функции <code>simulated_expensive_calculation</code></span>

Код листинга 13-3 несколько раз вызывает функцию с медленными расчётами. Первый блок `if` дважды вызывает `simulated_expensive_calculation`, блок `if` внутри внешнего `else` вообще не вызывает её, а код внутри второго `else` вызывает её один раз.

Желаемое поведение функции `generate_workout` состоит в том, чтобы сначала проверить, хочет ли пользователь тренировку с низкой интенсивностью (обозначается числом менее 25) или тренировку с высокой интенсивностью (число от 25 или более).

Планы тренировок низкой интенсивности будут рекомендовать несколько отжиманий и приседаний на основе сложного алгоритма, который мы моделируем.

Если пользователь хочет высокую интенсивность тренировок, то выполняется дополнительная логика. Если случайный образом выбираемое число равно 3, то предлагается сделать перерыв и освежиться. Иначе на основании сложного алгоритма пользователь получит задание бегать несколько минут.

Данный код работает, так как этого хочет заказчик сейчас. Но допустим, что команда специалистов по анализу данных в будущем решит, что нам нужно внести некоторые изменения в то, как мы вызываем функцию `simulated_expensive_calculation`. Чтобы упростить обновление кода, когда возникнут подобного рода желания, нам стоит переделать код так, чтобы функцию `simulated_expensive_calculation` вызывали только раз. Мы также хотим избавиться от места, где мы вызываем эту функцию дважды и при этом не добавлять какие-либо другие вызовы этой функции в коде. Иными словами, мы не хотим вызывать функцию, если её результат не нужен, и одновременно мы всё равно хотим вызвать её только один раз.

#### Рефакторинг используя функции

Можно было бы реструктуризовать нашу программу разными способами. Сначала, мы попробуем извлечь повторные вызовы функции  `simulated_expensive_calculation` в переменную, как показано в листинге 13-4.

<span class="filename">Файл: src/main.rs</span>

```rust
{{#rustdoc_include ../listings/ch13-functional-features/listing-13-04/src/main.rs:here}}
```

<span class="caption">Листинг 13-4: Извлечение вызова функции <code>simulated_expensive_calculation</code> в одно место и сохранение результата в переменной <code>expensive_result</code></span>

Это изменение объединяет все вызовы `simulated_expensive_calculation` и решает проблему первого `if` блока, который вызывает функцию дважды без необходимости. К сожалению, сейчас мы вызываем эту функцию и ждём результат во всех случаях, включая внутренний блок `if`, который вообще не использует значение результата.

Мы хотим определить код в одном месте нашей программы, но *выполнять* этот код, только там где нам действительно нужен результат. Это случай для замыканий!

#### Рефакторинг с помощью замыкания для сохранение кода, который может быть запущен позднее

Вместо того, чтобы всегда выполнять функцию `simulated_expensive_calculation` перед блоком `if`, мы может определить замыкание и сохранить это *замыкание* в переменной вместо того, чтобы сохранять результат вызова функции, как показано в листинге 13-5. Можно переместить все тело `simulated_expensive_calculation` в замыкание представленное здесь.

<span class="filename">Файл: src/main.rs</span>

```rust
{{#rustdoc_include ../listings/ch13-functional-features/listing-13-05/src/main.rs:here}}
```

<span class="caption">Листинг 13-5: Определение замыкания и сохранение его в переменной <code>expensive_closure</code></span>

Определение замыкания начинается после символа `=`, который присваивает его переменной `expensive_closure`. Замыкание мы начинаем с пары палочек (vertical pipes (`|`)). Внутри этой конструкции мы определяем входные параметры замыкания. Такой синтаксис был выбран под влиянием языков Ruby и Smalltalk. Данное замыкание имеет параметр `num`. Если нужно несколько параметров, то они разделяются запятыми: `|param1, param2|`.

После параметров замыкания, в фигурных скобках идёт тело функции замыкания. Фигурные скобки могут не использоваться, если код функции состоит только из одной строчки кода. После закрытия фигурных скобок необходим символ `;` для завершения выражения. Значение возвращаемое последней строчкой тела замыкания (`num`) будет являться значением, которое будет возвращено из замыкания, когда оно будет вызвано, поэтому данная строка не содержит точку с запятой (`;`) как и в теле любой функции.

Обратите внимание, что выражение `let` означает, что  `expensive_closure` содержит *определение* анонимной функции, а не *значение результата* выполнения анонимной функции. Напомним, что мы используем замыкание, потому что хотим определить код для вызова в одной точке, сохранить этот код и фактически вызвать его на более позднем этапе. Код, который мы хотим вызвать, теперь хранится в переменной `expensive_closure`.

Теперь, после определения замыкания можно изменить код в блоках `if`: вызвать код замыкания чтобы его выполнить и получить результирующее значение. Вызов замыкания очень похож на вызов функции. Мы определяем имя переменной, которая содержит определение замыкания и в скобках указываем аргументы, которые мы хотим использовать для вызова, как показано в листинге 13-6.

<span class="filename">Файл: src/main.rs</span>

```rust
{{#rustdoc_include ../listings/ch13-functional-features/listing-13-06/src/main.rs:here}}
```

<span class="caption">Листинг 13-6: Вызов объявленного замыкания <code>expensive_closure</code></span>

Теперь дорогостоящий расчёт вызывается только в одном месте, и мы выполняем этот код только там, где нам нужны результаты.

Тем не менее, у нас осталась ещё одна проблема в листинге 13-3: мы все ещё дважды вызываем замыкание в первом блоке `if`, что дважды вызовет медленный код и заставит пользователя ждать дважды выполнение функции. Мы могли бы решить эту проблему, создав локальную переменную для этого блока `if`, чтобы хранить результат вызова замыкания, но замыкания предоставляют нам другое решение. Мы немного поговорим об этом решении. Но сначала давайте поговорим о том, почему в определении замыкания нет аннотаций типов и признаков, связанных с замыканиями.

### Выведение типа замыкания и аннотация

Замыкания не требуют аннотирования типов параметров или возвращаемого значения, как это делают функции `fn`. Аннотации типов требуются для функций, потому что они являются частью явного интерфейса, предоставляемого пользователям. Жёсткое определение интерфейса функций вытекает из их открытости и важно для обеспечения того, чтобы все понимали какие типы значений использует и возвращает функция. Но замыкания, в отличие от функций, не используются в открытых интерфейсах: они хранятся в переменных и используются без их наименования и предоставления пользователям нашей библиотеки.

Замыкания являются обычно короткими и актуальны только в узком контексте, а не в любом произвольном сценарии. В этих ограниченных контекстах компилятор может надёжно определить типы входных параметров и возвращаемый тип, подобно тому, как он может выводить типы большинства переменных.

Заставлять программистов аннотировать типы в этих небольших анонимных функциях было бы раздражающим и в значительной степени излишним при наличии информации уже имеющейся у компилятора.

Как и в случае с переменными, можно добавить аннотации типов, если мы хотим повысить явность и ясность кода за счёт большей детализации, чем это строго необходимо. Аннотирование типов для замыкания, которое мы определили в листинге 13-5, будет выглядеть как определение, показанное в листинге 13-7.

<span class="filename">Файл: src/main.rs</span>

```rust
{{#rustdoc_include ../listings/ch13-functional-features/listing-13-07/src/main.rs:here}}
```

<span class="caption">Листинг 13-7: Добавление необязательных аннотаций типов для параметров и типов возвращаемых значений у замыкания</span>

С добавленными аннотациями типов синтаксис замыканий выглядит более похожим на синтаксис функций. Ниже приводится сравнение синтаксиса функции, которая добавляет 1 к своему параметру и несколько эквивалентных замыканий, которые имеют аналогичное поведение. Мы добавили несколько пробелов для выравнивания соответствующих частей. Это показывает, как синтаксис замыкания похож на синтаксис функции, за исключением использования вертикальных палочек и необязательного синтаксиса:

```rust,ignore
fn  add_one_v1   (x: u32) -> u32 { x + 1 }
let add_one_v2 = |x: u32| -> u32 { x + 1 };
let add_one_v3 = |x|             { x + 1 };
let add_one_v4 = |x|               x + 1  ;
```

Первая строка показывает определение функции, а вторая строка показывает полностью аннотированное определение замыкания. В третьей строке удаляются аннотации типов из определения замыкания, а в четвёртой строке удаляются скобки, которые являются необязательными, поскольку тело замыкания имеет только одно выражение. Все это допустимые определения, которые будут вызывать одинаковое поведение при вызове. Вызов замыканий `add_one_v3` и `add_one_v4` требует, чтобы они могли скомпилироваться, поэтому при компиляции типы будут выводиться на основе их использования.

Определения замыканий будут иметь конкретные типы выведенные только один раз - для каждого из её параметров и для возвращаемого значения. Например, в листинге 13-8 показано определение короткого замыкания, которое просто возвращает значение, которое оно получает в качестве параметра. Это замыкание не очень полезно, за исключением целей этого примера. Обратите внимание, что мы не добавили никаких аннотаций типов в определение: если мы затем попытаемся дважды вызвать замыкание, используя тип `String` в качестве аргумента в первый раз и тип `u32` во второй раз, то мы получим ошибку.

<span class="filename">Файл: src/main.rs</span>

```rust,ignore,does_not_compile
{{#rustdoc_include ../listings/ch13-functional-features/listing-13-08/src/main.rs:here}}
```

<span class="caption">Листинг 13-8: Попытка вызвать замыкание, типы у которого выводятся двумя разными типами</span>

Компилятор вернёт нам вот такую ошибку:

```console
{{#include ../listings/ch13-functional-features/listing-13-08/output.txt}}
```

Когда мы в первый раз вызываем `example_closure` со значением типа `String`, компилятор выводит тип для  `x` и возвращаемого значения как `String`. Эти типы затем привязываются к замыканию в `example_closure` и мы получаем ошибку типа, если пытаемся использовать другой тип с тем же замыканием.

### Сохранение замыканий используя обобщённые типы и типажи `Fn`

Вернёмся к нашему приложению для создания тренировок. В листинге 13-6 наш код по-прежнему вызывал замыкание с дорогостоящим вычислением больше, чем это требовалось. Один из вариантов решения этой проблемы - сохранить результат дорогостоящего замыкания в переменной для повторного использования и использовать переменную в каждом месте, где нам нужен результат, вместо повторного вызова замыкания. Однако этот метод может привести к многократному повторению кода.

К счастью, нам доступно другое решение. Можно создать структуру, которая будет содержать замыкание и значение вызова замыкания. Структура будет выполнять замыкание только если нам понадобится результирующее значение, а результирующее значение будет кэшировать, поэтому остальной части нашего кода не понадобится отвечать за сохранение и повторное использование результата. Вы можете знать этот шаблон как *memoization (запоминание)* или *lazy evaluation (ленивое вычисление)*.

Чтобы создать структуру, которая содержит замыкание, нам нужно указать тип замыкания, потому что определение структуры должно описывать типы каждого из его полей. Каждый экземпляр замыкания имеет свой уникальный анонимный тип: то есть, даже если два замыкания имеют одну и ту же сигнатуру, их типы по-прежнему считаются разными. Для определения структур, перечислений или параметров функций, которые используют замыкания, мы используем обобщённые типы и ограничения типажей, как мы обсуждали в Главе 10.

Типажи `Fn` входят в состав стандартной библиотеки. Все замыкания реализуют один из типажей: `Fn`, `FnMut` или `FnOnce`. Мы поговорим о различиях между ними в разделе ["Захват переменных окружения замыканиями"](#capturing-the-environment-with-closures)<!--  -->; в данном примере мы можем использовать типаж `Fn`.

Мы добавляем типы в описание ограничений типажа `Fn` для описания типов параметров и возвращаемого значения, которое замыкания должны иметь для того, чтобы соответствовать данному ограничению типажа. В данном случае, наше замыкание имеет тип параметра `u32` и возвращает тип `u32`, поэтому сигнатуру ограничения типажа мы описываем как `Fn(u32) -> u32`.

Листинг 13-9 показывает определение структуры `Cacher` содержащей замыкание и необязательное значение результата:

<span class="filename">Файл: src/main.rs</span>

```rust
{{#rustdoc_include ../listings/ch13-functional-features/listing-13-09/src/main.rs:here}}
```

<span class="caption">Листинг 13-9: Определение структуры <code>Cacher</code> содержащей замыкание в поле <code>calculation</code> и дополнительный результат в поле <code>value</code></span>

Структура `Cacher` имеет поле `calculation` обобщённого типа `T`. Ограничение типажа для `T` требует, чтобы обобщённый тип соответствовал замыканию: требование определяется типажом `Fn`. Любое замыкание, которые мы хотим сохранить в поле `calculation` должно иметь один параметр типа  `u32` (указанный внутри круглых скобок после `Fn`) и должно возвращать тип `u32` (указанный после `->`).

> Примечание. Функции также могут реализовывать все три типажа `Fn`. Если то, что мы хотим сделать, не требует захвата значения из среды, мы можем использовать функцию, а не замыкание, где нам нужно что-то, что реализует типаж `Fn`.

Поле `value` имеет тип `Option<u32>`. Перед выполнением замыкания, значение `value` будет `None`. Когда код, использующий `Cacher`, запрашивает  *результат* выполнения замыкания, `Cacher` выполнит код замыкания и сохранит результат внутри варианта `Some` в поле `value`. Затем, если код снова запросит результат замыкания, вместо повторного выполнения замыкания, `Cacher` вернёт результат, хранящийся в варианте `Some`.

Логика вычисления поля `value`, которую мы только что описали определена в листинге 13-10.

<span class="filename">Файл: src/main.rs</span>

```rust
{{#rustdoc_include ../listings/ch13-functional-features/listing-13-10/src/main.rs:here}}
```

<span class="caption">Листинг 13-10: Логика кэширования в <code>Cacher</code></span>

Мы хотим, чтобы `Cacher` управлял значениями полей структуры, а не позволял бы вызывающему коду потенциально изменять значения в этих полях напрямую, поэтому эти поля являются закрытыми.

Функция `Cacher::new` принимает обобщённый параметр `T`, который мы определили как имеющий то же ограничения типажа, что и структура `Cacher`. Затем `Cacher::new` возвращает экземпляр `Cacher` содержащий замыкание, указанное в поле `calculation` и значение `None` в поле `value`, потому что мы ещё не выполнили замыкание.

Когда вызывающему коду требуется результат выполнения замыкания, то вместо непосредственного вызова замыкания, он вызовет метод `value`. Этот метод проверяет, есть ли у нас уже готовое значение `self.value` в `Some`; если есть, то метод возвращает значение в `Some` без повторного выполнения замыкания.

Если же поле `self.value` имеет значение `None`, то код вызывает замыкание сохранённое в поле `self.calculation` и результат работы записывается в поле `self.value` для будущего использования и, затем, полученное значение возвращается вызывающему коду.

Листинг 13-11 демонстрирует использование структуры `Cacher` в функции `generate_workout` из листинга 13-6.

<span class="filename">Файл: src/main.rs</span>

```rust
{{#rustdoc_include ../listings/ch13-functional-features/listing-13-11/src/main.rs:here}}
```

<span class="caption">Листинг 13-11: Использование <code>Cacher</code> в функции <code>generate_workout</code> для абстрагирования логики кэширования</span>

Вместо непосредственного сохранения замыкания в переменной мы сохраняем новый экземпляр `Cacher`, который содержит замыкание. Затем в каждом месте, где мы хотим получить результат, мы вызываем метод `value` у экземпляра `Cacher`. Мы можем вызывать метод `value` столько раз, сколько захотим или не вызывать его вообще, а дорогостоящие вычисления будут выполняться максимум один раз.

Попробуйте запустить эту программу с помощью функции `main` из листинга 13-2. Измените значения в переменных `simulated_user_specified_value` и `simulated_random_number`, чтобы убедиться, что во всех случаях в различных блоках `if` и `else` текст `calculating slowly...` появляется только один раз и только при необходимости. `Cacher` заботится о логике, необходимой для обеспечения того, чтобы мы не вызывали дорогостоящие вычисления больше, чем нужно, чтобы мы могли сосредоточиться на бизнес-логике в `generate_workout`.

### Ограничения реализации `Cacher`

Кэширование значений - это обычно полезное поведение, которое мы могли бы использовать в других частях нашего кода с другими замыканиями. Однако в текущей реализации `Cacher` есть две проблемы, которые затрудняют его повторное использование в различных контекстах.

Первая проблема заключается в том, что экземпляр `Cacher` предполагает, что он всегда получит одно и то же значение параметра `arg` для метода `value`. То есть этот тест `Cacher` не пройдёт:

```rust,ignore,panics
{{#rustdoc_include ../listings/ch13-functional-features/no-listing-01-failing-cacher-test/src/lib.rs:here}}
```

Этот тест создаёт новый экземпляр `Cacher` с замыканием, которое возвращает переданное в него значение. Мы вызываем метод `value` для этого экземпляра `Cacher` со значением `arg` равным 1 и затем значением `arg` равным 2 и ожидаем, что вызов `value` со значением `arg` равным 2 вернёт 2.

Запустите этот тест с реализацией `Cacher` в листинге 13-9 и листинге 13-10, тест `assert_eq!` завершится неудачно с данным сообщением:

```console
{{#include ../listings/ch13-functional-features/no-listing-01-failing-cacher-test/output.txt}}
```

Проблема в том, что при первом вызове `c.value` с аргументом 1, экземпляр `Cacher` сохранит значение `Some(1)` в `self.value`. После этого, неважно какие будут входные параметры метода  `value`, он всегда будет возвращать 1.

Попробуйте изменить `Cacher` так, чтобы он сохранял HashMap, а не единственное значение. Ключами HashMap будут значения `arg`, которые передаются, а значениями будут результаты вызова замыкания для соответствующего ключа. Вместо того, чтобы проверять имеет ли `self.value` значение `Some` или `None`, функция `value` будет делать поиск `arg` в HashMap и возвращать значение, если оно присутствует. Если оно отсутствует, то `Cacher` вызовет замыкание и сохранит полученное значение в HashMap связанное со значением из `arg`.

Вторая проблема с текущей реализацией `Cacher` заключается в том, что она принимает только замыкания, которые имеют один входной параметр типа `u32` и возвращают `u32`. Мы могли бы захотеть, к примеру, кэшировать результаты замыканий, которые берут строковый срез и возвращают значение типа `usize`. Чтобы решить эту проблему, попробуйте ввести обобщённые параметры, чтобы повысить гибкость функциональности `Cacher`.

### Захват переменных окружения с помощью замыкания

В примере генератора тренировок мы использовали только замыкания в качестве встроенных анонимных функций. Однако у замыканий есть дополнительная возможность, которой нет у функций: они могут захватывать своё окружение и получать доступ к переменным из области видимости, в которой они определены.

В листинге 13-12 приведён пример замыкания, хранящегося в переменной `equal_to_x`, в которой используется переменная `x` из ближайшего окружения замыкания.

<span class="filename">Файл: src/main.rs</span>

```rust
{{#rustdoc_include ../listings/ch13-functional-features/listing-13-12/src/main.rs}}
```

<span class="caption">Листинг 13-12: Пример замыкания, которое ссылается на переменную в области видимости</span>

Здесь, хоть `x` и не является одним из параметров `equal_to_x`, замыканию `equal_to_x` разрешено использовать  переменную `x`, которая определена в той же области видимости что и `equal_to_x`.

Мы не можем сделать то же самое с функциями; если мы попробуем использовать следующий пример, наш код не скомпилируется:

<span class="filename">Файл: src/main.rs</span>

```rust,ignore,does_not_compile
{{#rustdoc_include ../listings/ch13-functional-features/no-listing-02-functions-cant-capture/src/main.rs}}
```

Описание ошибки:

```console
{{#include ../listings/ch13-functional-features/no-listing-02-functions-cant-capture/output.txt}}
```

Компилятор даже напоминает нам, что это работает только с замыканиями!

В случае когда замыкание захватывает значение из своего окружения, оно использует дополнительную память для хранения значений используемых в теле замыкания. Такое использование памяти является накладными расходами, которые мы не хотим платить в общих случаях, там где мы хотим выполнить код, который не захватывает переменные окружения. Поскольку функциям никогда не разрешается захватывать их окружение, то определение и использование функций никогда не повлечёт за собой таких издержек.

Замыкания могут захватывать значения из своего окружения тремя способами, которые напрямую соответствуют трём способам, которыми функция может принимать параметр: забирать во владение, получать изменяемое заимствование и получать неизменяемое заимствование. Эти способы закодированы в трёх типажах `Fn` следующим образом:

- замыкания типажа `FnOnce` потребляют переменные, которые они захватывают из окружающего контекста, известного как *окружение (environment)* замыкания. Чтобы использовать захваченные переменные, замыкание должно стать владельцем этих переменных и переместить их в замыкание, когда оно определено. Часть имени `Once` отражает тот факт, что замыкание не может владеть одними и теми же переменными более одного раза, поэтому его можно вызывать только один раз.
- замыкания типажа `FnMut` могут изменять значения переменных из окружения, поскольку они заимствуют изменяемые значения.
- замыкания типажа `Fn` заимствуют значения из окружения без их изменения.

Когда вы создаёте замыкание Rust определяет какой типаж использовать, основываясь на том как замыкание использует значения из окружения. Все замыкания реализуют `FnOnce`, потому что все они могут быть вызваны хотя бы один раз. Замыкания, которые не перемещают захваченные переменные, также реализуют `FnMut`, а замыкания которым не требуется изменяемый доступ к захваченным переменным, также реализуют `Fn`. В листинге 13-12 замыкание `equal_to_x` заимствует `x` как неизменяемый (поэтому `equal_to_x` имеет типаж `Fn`), поскольку тело замыкания должно только читать значение в `x`.

Если вы хотите, чтобы замыкание стало владельцем значений, которые оно использует в окружении, то вы можете использовать ключевое слово `move` перед списком параметров. Этот метод в основном полезен при передаче замыкания в новый поток для перемещения данных, чтобы они принадлежали новому потоку.

> Примечание: замыкания с `move` могут по-прежнему реализовывать `Fn` или `FnMut`, даже если они захватывают переменные при перемещении. Это связано с тем, что типаж, реализуемый типом замыкания, определяется тем, что замыкание делает с захваченными значениями, а не тем, как оно их захватывает. Ключевое слово `move` указывает только как замыкание захватывает значения.

У нас будет больше примеров замыканий с `move` в Главе 16, когда мы поговорим про одновременное выполнение множества задач (concurrency). А пока будем довольствоваться кодом из листинга 13-12 с ключевым словом `move`, добавленным в определение замыкания и использующим векторы вместо целых чисел, поскольку целые числа можно копировать, а не перемещать. Обратите внимание, что этот код ещё не компилируется.

<span class="filename">Файл: src/main.rs</span>

```rust,ignore,does_not_compile
{{#rustdoc_include ../listings/ch13-functional-features/no-listing-03-move-closures/src/main.rs}}
```

Мы получаем следующую ошибку:

```console
{{#include ../listings/ch13-functional-features/no-listing-03-move-closures/output.txt}}
```

Значение `x` перемещается в замыкание, когда замыкание определено, потому что мы добавили ключевое слово `move`. Замыкание становится владельцем `x` и `main` больше не может использовать `x` в макросе `println!`. Удаление `println!` исправит ошибку в этом примере.

В большинстве случаев при указании одного из ограничений типажа `Fn` можно начать с типажа `Fn` и компилятор сообщит нужен ли вам `FnMut` или `FnOnce` в зависимости от того, что происходит в теле замыкания.

Для иллюстрации ситуаций, когда замыкания захватывающие своё окружение (как параметры функций) являются полезными, давайте перейдём к следующей теме: итераторы.
