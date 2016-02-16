% Макросы, Практическое введение

Эта глава расскажет о системе Rust макрос-по-примеру, используя относительно простой, практичный пример. Мы *не* будем пытаться объяснить всю сложность системы; нашей целью является сделать так, чтобы вы чувствовали себя комфортно с макросами и понимали как и почему они пишутся.

Есть также [глава по макросам в Rust Book](http://doc.rust-lang.org/book/macros.html),в которой объяснения даются на более высоком уровне и [методическое введение](mbe-README.html) - глава в этой книге, которая объясняет систему макросов подробно.

## Маленький контекст

> **Заметка**: не паникуйте!  Дальше пойдет разговор о математике. Вы можете спокойно пропустить этот раздел, если хотите добраться до самого мяса этой статьи.

Если вы не знали, рекурентное соотношение - это последовательность, в которой каждое значение определяется в терминах одного или нескольких *предыдущих*, с одним или несколькими начальными значениями. Например, [последовательность Фибоначчи](https://en.wikipedia.org/wiki/Fibonacci_number) можно описать такой связью:

<!-- Начало математики: $F_n = 0, 1, \ldots, F_n-1 + F_n - 2$ -->
<style type="text/css">
    .katex {
        font: 400 1.21em/1.2 KaTeX_Main;
        white-space: nowrap;
        font-family: "Cambria Math", "Cambria", serif;
    }

    .katex .vlist > span > {
        display: inline-block;
    }

    .mathit {
        font-style: italic;
    }

    .katex .reset-textstyle.scriptstyle {
        font-size: 0.7em;
    }

    .katex .reset-textstyle.textstyle {
        font-size: 1em;
    }

    .katex .textstyle > .mord + .mrel {
        margin-left: 0.27778em;
    }

    .katex .textstyle > .mrel + .minner, .katex .textstyle > .mrel + .mop, .katex .textstyle > .mrel + .mopen, .katex .textstyle > .mrel + .mord {
        margin-left: 0.27778em;
    }

    .katex .textstyle > .mclose + .minner, .katex .textstyle > .minner + .mop, .katex .textstyle > .minner + .mord, .katex .textstyle > .mpunct + .mclose, .katex .textstyle > .mpunct + .minner, .katex .textstyle > .mpunct + .mop, .katex .textstyle > .mpunct + .mopen, .katex .textstyle > .mpunct + .mord, .katex .textstyle > .mpunct + .mpunct, .katex .textstyle > .mpunct + .mrel {
        margin-left: 0.16667em;
    }

    .katex .textstyle > .mord + .mbin {
        margin-left: 0.22222em;
    }

    .katex .textstyle > .mbin + .minner, .katex .textstyle > .mbin + .mop, .katex .textstyle > .mbin + .mopen, .katex .textstyle > .mbin + .mord {
        margin-left: 0.22222em;
    }
</style>

<div class="katex" style="font-size: 100%; text-align: center;">
    <span class="katex"><span class="katex-inner"><span style="height: 0.68333em;" class="strut"></span><span style="height: 0.891661em; vertical-align: -0.208331em;" class="strut bottom"></span><span class="base textstyle uncramped"><span class="reset-textstyle displaystyle textstyle uncramped"><span class="mord displaystyle textstyle uncramped"><span class="mord"><span class="mord mathit" style="margin-right: 0.13889em;">F</span><span class="vlist"><span style="top: 0.15em; margin-right: 0.05em; margin-left: -0.13889em;" class=""><span class="fontsize-ensurer reset-size5 size5"><span style="font-size: 0em;" class="">​</span></span><span class="reset-textstyle scriptstyle cramped"><span class="mord mathit">n</span></span></span><span class="baseline-fix"><span class="fontsize-ensurer reset-size5 size5"><span style="font-size: 0em;" class="">​</span></span>​</span></span></span><span class="mrel">=</span><span class="mord">0</span><span class="mpunct">,</span><span class="mord">1</span><span class="mpunct">,</span><span class="mpunct">…</span><span class="mpunct">,</span><span class="mord"><span class="mord mathit" style="margin-right: 0.13889em;">F</span><span class="vlist"><span style="top: 0.15em; margin-right: 0.05em; margin-left: -0.13889em;" class=""><span class="fontsize-ensurer reset-size5 size5"><span style="font-size: 0em;" class="">​</span></span><span class="reset-textstyle scriptstyle cramped"><span class="mord scriptstyle cramped"><span class="mord mathit">n</span><span class="mbin">−</span><span class="mord">1</span></span></span></span><span class="baseline-fix"><span class="fontsize-ensurer reset-size5 size5"><span style="font-size: 0em;" class="">​</span></span>​</span></span></span><span class="mbin">+</span><span class="mord"><span class="mord mathit" style="margin-right: 0.13889em;">F</span><span class="vlist"><span style="top: 0.15em; margin-right: 0.05em; margin-left: -0.13889em;" class=""><span class="fontsize-ensurer reset-size5 size5"><span style="font-size: 0em;" class="">​</span></span><span class="reset-textstyle scriptstyle cramped"><span class="mord scriptstyle cramped"><span class="mord mathit">n</span><span class="mbin">-</span><span class="mord">2</span></span></span></span><span class="baseline-fix"><span class="fontsize-ensurer reset-size5 size5"><span style="font-size: 0em;" class="">​</span></span>​</span></span></span></span></span></span></span></span>
</div>
<!-- Конец Математики -->

Итак, первые два числа в последовательности - 0 и 1, а третье - <em>F<sub>0</sub></em> + <em>F<sub>1</sub></em> = 0 + 1 = 1, четвертое - <em>F<sub>1</sub></em> + <em>F<sub>2</sub></em> = 1 + 1 = 2, и так далее.

Итак, *из-за того, что* такая последовательность может продолжаться бесконечно, описание функции `fibonacci` получилось довольно хитрым, ведь вам же не нужно возвращать полный список элементов. Все, что вы *хотите* - это вернуть что-то, лениво просчитав столько элементов, сколько надо.

В Rust, это называется создать `Iterator`. Это не особо *трудно*, но тут потребуется довольно много рутинных действий: нужно определить свой тип, понять, какое состояние хранить в нем, и затем реализовать трейт `Iterator` для него.

Получается, рекурентное соотношение - это настолько просто, что почти от всех конкретных деталей можно абстрагироваться и создать маленький генератор кода на базе макроса.

Итак, сказав все, что нужно, давайте начнем.

## Создание

Обычно, если я работаю над новым макросом, первое, что я решаю - это то, как будет выглядеть вызов макроса. В данном конкретном случае в первом приближении получится следующее:

```ignore
let fib = recurrence![a[n] = 0, 1, ..., a[n-1] + a[n-2]];

for e in fib.take(10) { println!("{}", e) }
```

После этого мы можем поставить заглушку в определении макроса, даже если мы не уверены во что, он должен разворачиваться. Это полезно, потому что если вам пока непонятно, как парсить входящий синтаксис, то, *возможно*, вам нужно поменять его.

```rust
macro_rules! recurrence {
    ( a[n] = $($inits:expr),+ , ... , $recur:expr ) => { /* ... */ };
}
# fn main() {}
```

Подразумевая, что вы не знакомы с синтаксисом, позвольте мне объяснить. Здесь представлено определение макроса, выполняемоу через систему `macro_rules!`, названное `recurrence!`.  У этого макроса одно правило парсинга. Это правило гооврит, что вход данного макроса должен совпадать с:

- последовательностью литеральных токенов `a` `[` `n` `]` `=`,
- повторяющейся  (`$( ... )`) последовательностью, используя `,` как разделитель, и одно или больше (`+`) повторений:
    - валидного *выражения*, захваченногоcaptured в переменные `inits` (`$inits:expr`)
- последовательностью литеральных токенов `,` `...` `,`,
- вылидным *выражением*, захваченным в переменную `recur` (`$recur:expr`).

Наконец, правило говорит, *если* вход совпадает с этим правилом, то вызов макроса нужно заменить на последовательность токенов `/* ... */`.

Стоит отметить, что `inits`, как понятно из названия, на самом деле содержит *все* выражения, которые совпадают на этой позиции, а не только первое или последнее. Более того, оно захватывает их *как последовательность* в отличие от, скажем, необратимой вставки их всех вместе. Также помните, что вы можете изменить повторение на "ноль и больше", используя `*` вместо `+`.  Поддержки "одного или нескольких" или еще более конкретного числа повторений тут нет.

В качестве упражнения, давайте возьмем прелагаемый вход и пропустим его через правило, чтобы посмотреть, как оно выполняется. Колонка "позиция", показывающая, какая часть паттерна должна совпасть следующей, отмечается "⌂". Помните, что в некоторых случаях, может быть больше одного возможного "следующего" элемента, с которым найдется совпадение.  "Вход" будет содержить все токены, которые еще *не* были обработаны. `inits` и `recur` будут содержать содержимое их связей.

<style type="text/css">
    /* Customisations. */

    .small-code code {
        font-size: 60%;
    }

    table pre.rust {
        margin: 0;
        border: 0;
    }

    table.parse-table code {
        white-space: pre-wrap;
        background-color: transparent;
        border: none;
    }

    table.parse-table tbody > tr > td:nth-child(1) > code:nth-of-type(2) {
        color: red;
        margin-top: -0.7em;
        margin-bottom: -0.6em;
    }

    table.parse-table tbody > tr > td:nth-child(1) > code {
        display: block;
    }

    table.parse-table tbody > tr > td:nth-child(2) > code {
        display: block;
    }
</style>

<table class="parse-table">
    <thead>
        <tr>
            <th>Позиция</th>
            <th>Вход</th>
            <th><code>inits</code></th>
            <th><code>recur</code></th>
        </tr>
    </thead>
    <tbody class="small-code">
        <tr>
            <td><code>a[n] = $($inits:expr),+ , ... , $recur:expr</code>
                <code>⌂</code></td>
            <td><code>a[n] = 0, 1, ..., a[n-1] + a[n-2]</code></td>
            <td></td>
            <td></td>
        </tr>
        <tr>
            <td><code>a[n] = $($inits:expr),+ , ... , $recur:expr</code>
                <code> ⌂</code></td>
            <td><code>[n] = 0, 1, ..., a[n-1] + a[n-2]</code></td>
            <td></td>
            <td></td>
        </tr>
        <tr>
            <td><code>a[n] = $($inits:expr),+ , ... , $recur:expr</code>
                <code>  ⌂</code></td>
            <td><code>n] = 0, 1, ..., a[n-1] + a[n-2]</code></td>
            <td></td>
            <td></td>
        </tr>
        <tr>
            <td><code>a[n] = $($inits:expr),+ , ... , $recur:expr</code>
                <code>   ⌂</code></td>
            <td><code>] = 0, 1, ..., a[n-1] + a[n-2]</code></td>
            <td></td>
            <td></td>
        </tr>
        <tr>
            <td><code>a[n] = $($inits:expr),+ , ... , $recur:expr</code>
                <code>     ⌂</code></td>
            <td><code>= 0, 1, ..., a[n-1] + a[n-2]</code></td>
            <td></td>
            <td></td>
        </tr>
        <tr>
            <td><code>a[n] = $($inits:expr),+ , ... , $recur:expr</code>
                <code>       ⌂</code></td>
            <td><code>0, 1, ..., a[n-1] + a[n-2]</code></td>
            <td></td>
            <td></td>
        </tr>
        <tr>
            <td><code>a[n] = $($inits:expr),+ , ... , $recur:expr</code>
                <code>         ⌂</code></td>
            <td><code>0, 1, ..., a[n-1] + a[n-2]</code></td>
            <td></td>
            <td></td>
        </tr>
        <tr>
            <td><code>a[n] = $($inits:expr),+ , ... , $recur:expr</code>
                <code>                     ⌂  ⌂</code></td>
            <td><code>, 1, ..., a[n-1] + a[n-2]</code></td>
            <td><code>0</code></td>
            <td></td>
        </tr>
        <tr>
            <td colspan="4" style="font-size:.7em;">

<em>Внимание</em>: здесь два ⌂, потому что следующий входной токен можно сопоставить и <em>с</em> запятой-разделителем <em>между</em>em> элементами в повторении, <em>и с</em> запятой <em>после</em> повторения. Система макроса будет иметь ввиду обе возможности, до тех пор пока не сможет определить, какую выбрать.

            </td>
        </tr>
        <tr>
            <td><code>a[n] = $($inits:expr),+ , ... , $recur:expr</code>
                <code>         ⌂                ⌂</code></td>
            <td><code>1, ..., a[n-1] + a[n-2]</code></td>
            <td><code>0</code></td>
            <td></td>
        </tr>
        <tr>
            <td><code>a[n] = $($inits:expr),+ , ... , $recur:expr</code>
                <code>                     ⌂  ⌂ <s>⌂</s></code></td>
            <td><code>, ..., a[n-1] + a[n-2]</code></td>
            <td><code>0</code>, <code>1</code></td>
            <td></td>
        </tr>
        <tr>
            <td colspan="4" style="font-size:.7em;">

<em>Внимание</em>: третий, подчеркнутый маркер показывает, что система макроса после обработки последнего токена удалила одну из предыдующих возможных веток.

            </td>
        </tr>
        <tr>
            <td><code>a[n] = $($inits:expr),+ , ... , $recur:expr</code>
                <code>         ⌂                ⌂</code></td>
            <td><code>..., a[n-1] + a[n-2]</code></td>
            <td><code>0</code>, <code>1</code></td>
            <td></td>
        </tr>
        <tr>
            <td><code>a[n] = $($inits:expr),+ , ... , $recur:expr</code>
                <code>         <s>⌂</s>                    ⌂</code></td>
            <td><code>, a[n-1] + a[n-2]</code></td>
            <td><code>0</code>, <code>1</code></td>
            <td></td>
        </tr>
        <tr>
            <td><code>a[n] = $($inits:expr),+ , ... , $recur:expr</code>
                <code>                                ⌂</code></td>
            <td><code>a[n-1] + a[n-2]</code></td>
            <td><code>0</code>, <code>1</code></td>
            <td></td>
        </tr>
        <tr>
            <td><code>a[n] = $($inits:expr),+ , ... , $recur:expr</code>
                <code>                                           ⌂</code></td>
            <td></td>
            <td><code>0</code>, <code>1</code></td>
            <td><code>a[n-1] + a[n-2]</code></td>
        </tr>
        <tr>
            <td colspan="4" style="font-size:.7em;">

<em>Внимание</em>: этот конкретный шаг должен объяснить, что такая связь, как <tt>$recur:expr</tt>, заменит <em>все выражение</em>, используя знания компилятора о том, что считать валидным выражением. Как будет рассказано дальше, вы можете так делать и с другими конструкциями языка.

            </td>
        </tr>
    </tbody>
</table>

<p></p>

Что нужно понять из этого всего, это то, что система макроса будет *пытаться* последовательно сопоставить предложенные на входе токены с представленным правилом. Мы еще вернемся к части "попыток".

Теперь, давайте напишем последнюю, полностью развернутую форму. Для этого развертывания, я искал что-то вроде этого:

```ignore
let fib = {
    struct Recurrence {
        mem: [u64; 2],
        pos: usize,
    }
```

Это будет тип итератора.  `mem` будет буфером в памяти, который будет содержать несколько последних значений, достаточных для продолжения рекурентных вычислений.  `pos` должен следить за значением `n`.

> **В сторону**: Я выбрал `u64` как "достаточно большой" тип для элементов этой последовательности.  Не волнуйтесь о том, как это будет работать для *других* последовательностей; мы еще вернемся к этому.

```ignore
    impl Iterator for Recurrence {
        type Item = u64;

        #[inline]
        fn next(&mut self) -> Option<u64> {
            if self.pos < 2 {
                let next_val = self.mem[self.pos];
                self.pos += 1;
                Some(next_val)
```

Нам нужна ветка, которая будет заполнять начальные значения последовательности; ничего необычного.

```ignore
            } else {
                let a = /* something */;
                let n = self.pos;
                let next_val = (a[n-1] + a[n-2]);

                self.mem.TODO_shuffle_down_and_append(next_val);

                self.pos += 1;
                Some(next_val)
            }
        }
    }
```

Тут все немножко посложнее; мы еще вернемся и посмотрим, *как* именно определить `a`.  Также, `TODO_shuffle_down_and_append` это еще один временный заполнитель; Мне нужна что-то, что заменит `next_val` в конце массива, сдвинув все остальное вниз на одну позицию и удалив 0й элемент.

```ignore

    Recurrence { mem: [0, 1], pos: 0 }
};

for e in fib.take(10) { println!("{}", e) }
```

Наконец, вернем экземпляр нашей новой структуры, по которому затем можно выполнять итерации. Объединяя все, полное развертывание будет таким:

```ignore
let fib = {
    struct Recurrence {
        mem: [u64; 2],
        pos: usize,
    }

    impl Iterator for Recurrence {
        type Item = u64;

        #[inline]
        fn next(&mut self) -> Option<u64> {
            if self.pos < 2 {
                let next_val = self.mem[self.pos];
                self.pos += 1;
                Some(next_val)
            } else {
                let a = /* something */;
                let n = self.pos;
                let next_val = (a[n-1] + a[n-2]);

                self.mem.TODO_shuffle_down_and_append(next_val.clone());

                self.pos += 1;
                Some(next_val)
            }
        }
    }

    Recurrence { mem: [0, 1], pos: 0 }
};

for e in fib.take(10) { println!("{}", e) }
```

> **Aside**: Yes, this *does* mean we're defining a different `Recurrence` struct and its implementation for each macro invocation.  Most of this will optimise away in the final binary, with some judicious use of `#[inline]` attributes.

It's also useful to check your expansion as you're writing it.  If you see anything in the expansion that needs to vary with the invocation, but *isn't* in the actual macro syntax, you should work out where to introduce it.  In this case, we've added `u64`, but that's not neccesarily what the user wants, nor is it in the macro syntax.  So let's fix that.

```rust
macro_rules! recurrence {
    ( a[n]: $sty:ty = $($inits:expr),+ , ... , $recur:expr ) => { /* ... */ };
}

/*
let fib = recurrence![a[n]: u64 = 0, 1, ..., a[n-1] + a[n-2]];

for e in fib.take(10) { println!("{}", e) }
*/
# fn main() {}
```

Here, I've added a new capture: `sty` which should be a type.

> **Aside**: if you're wondering, the bit after the colon in a capture can be one of several kinds of syntax matchers.  The most common ones are `item`, `expr`, and `ty`.  A complete explanation can be found in [Macros, A Methodical Introduction; `macro_rules!` (Captures)](mbe-macro-rules.html#captures).
>
> There's one other thing to be aware of: in the interests of future-proofing the language, the compiler restricts what tokens you're allowed to put *after* a matcher, depending on what kind it is.  Typically, this comes up when trying to match expressions or statements; those can *only* be followed by one of `=>`, `,`, and `;`.
>
> A complete list can be found in [Macros, A Methodical Introduction; Minutiae; Captures and Expansion Redux](mbe-min-captures-and-expansion-redux.html).

## Indexing and Shuffling

I will skim a bit over this part, since it's effectively tangential to the macro stuff.  We want to make it so that the user can access previous values in the sequence by indexing `a`; we want it to act as a sliding window keeping the last few (in this case, 2) elements of the sequence.

We can do this pretty easily with a wrapper type:

```ignore
struct IndexOffset<'a> {
    slice: &'a [u64; 2],
    offset: usize,
}

impl<'a> Index<usize> for IndexOffset<'a> {
    type Output = u64;

    #[inline(always)]
    fn index<'b>(&'b self, index: usize) -> &'b u64 {
        use std::num::Wrapping;

        let index = Wrapping(index);
        let offset = Wrapping(self.offset);
        let window = Wrapping(2);

        let real_index = index - offset + window;
        &self.slice[real_index.0]
    }
}
```

> **В сторону**: из-за того, что время жизни *вгоняет в ступор* новичков в Rust, по-быстрму объясню: `'a` и `'b` - это параметры времени жизни, которые используются для слежения за ссылками (*т.e.* захваченными указателями на какие-то данные).  В этом случае, `IndexOffset` захватывает ссылку на наши данные итератора, поэтому нам надо следить, как долго ей можно удерживать их в себе, и для этого используется `'a`.
>
> `'b` используется, потому что функция `Index::index`  (то, как на самом деле реализуется синтаксис суб-скрипта) *также* параметризирована временем жизни, в расчете на возврат захваченной ссылки.  `'a` и `'b` не обязательно совпадают во всех случаях. Анализатор заимствований должен убедиться, что даже если мы явно не сопоставим  `'a` и `'b` друг с другом, мы на самом деле по неосторожности не нарушим целостность памяти.

Это меняет определение `a` на:

```ignore
let a = IndexOffset { slice: &self.mem, offset: n };
```

Единственный оставшийся вопрос - что насчет `TODO_shuffle_down_and_append`? Я не смог найти метод в стандартной библиотеке, совпадающий по семантике с тем, что мне нужно, но его и нетрудно сделать ручками самому.

```ignore
{
    use std::mem::swap;

    let mut swap_tmp = next_val;
    for i in (0..2).rev() {
        swap(&mut swap_tmp, &mut self.mem[i]);
    }
}
```

Здесь новое значение перемещается в конец массива, сдвигая остальные элементы на один вниз.

> **В сторону**: делая так, знайте, что этот код будет работать также и для типов, не поддерживающих копирование.

Рабочий код теперь выглядит так:

```rust
macro_rules! recurrence {
    ( a[n]: $sty:ty = $($inits:expr),+ , ... , $recur:expr ) => { /* ... */ };
}

fn main() {
    /*
    let fib = recurrence![a[n]: u64 = 0, 1, ..., a[n-1] + a[n-2]];

    for e in fib.take(10) { println!("{}", e) }
    */
    let fib = {
        use std::ops::Index;

        struct Recurrence {
            mem: [u64; 2],
            pos: usize,
        }

        struct IndexOffset<'a> {
            slice: &'a [u64; 2],
            offset: usize,
        }

        impl<'a> Index<usize> for IndexOffset<'a> {
            type Output = u64;

            #[inline(always)]
            fn index<'b>(&'b self, index: usize) -> &'b u64 {
                use std::num::Wrapping;

                let index = Wrapping(index);
                let offset = Wrapping(self.offset);
                let window = Wrapping(2);

                let real_index = index - offset + window;
                &self.slice[real_index.0]
            }
        }

        impl Iterator for Recurrence {
            type Item = u64;

            #[inline]
            fn next(&mut self) -> Option<u64> {
                if self.pos < 2 {
                    let next_val = self.mem[self.pos];
                    self.pos += 1;
                    Some(next_val)
                } else {
                    let next_val = {
                        let n = self.pos;
                        let a = IndexOffset { slice: &self.mem, offset: n };
                        (a[n-1] + a[n-2])
                    };

                    {
                        use std::mem::swap;

                        let mut swap_tmp = next_val;
                        for i in (0..2).rev() {
                            swap(&mut swap_tmp, &mut self.mem[i]);
                        }
                    }

                    self.pos += 1;
                    Some(next_val)
                }
            }
        }

        Recurrence { mem: [0, 1], pos: 0 }
    };

    for e in fib.take(10) { println!("{}", e) }
}
```

Заметьте, что я поменял порядок объявления `n` и `a`, а также обернул их (вместе с реккурентным выражением) в блок. Причина для первого довольна тривиальна (`n` должна быть определена раньше, чтобы я мог использовать его для `a`). Причина второго - это то, что заимствованная ссылка `&self.mem` не дает произойти дальнейшим сдвигам ( вы не можете изменить то, что связано в другом месте). Блок гарантирует, что заимствование `&self.mem` заканчивается до него.

Между прочим, единственной причиной, по которой код, делающий сдвиг `mem`, находится внутри блока, является желание приблизить область видимости, в которой доступен `std::mem::swap`, просто ради аккуратности.

Если мы выполним этот код, мы получим:

```text
0
1
2
3
5
8
13
21
34
```

Это успех! Теперь, давайте скопируем и вставим код в развертывание макроса, и заменим развернутый код на его вызов. Получаем:

```ignore
macro_rules! recurrence {
    ( a[n]: $sty:ty = $($inits:expr),+ , ... , $recur:expr ) => {
        {
            /*
                Идущий дальше код, это *буквально* код выше,
                вырезанный и вставленный на новую позицию. Никаких изменений
                больше не выполнялось.
            */

            use std::ops::Index;

            struct Recurrence {
                mem: [u64; 2],
                pos: usize,
            }

            struct IndexOffset<'a> {
                slice: &'a [u64; 2],
                offset: usize,
            }

            impl<'a> Index<usize> for IndexOffset<'a> {
                type Output = u64;

                #[inline(always)]
                fn index<'b>(&'b self, index: usize) -> &'b u64 {
                    use std::num::Wrapping;

                    let index = Wrapping(index);
                    let offset = Wrapping(self.offset);
                    let window = Wrapping(2);

                    let real_index = index - offset + window;
                    &self.slice[real_index.0]
                }
            }

            impl Iterator for Recurrence {
                type Item = u64;

                #[inline]
                fn next(&mut self) -> Option<u64> {
                    if self.pos < 2 {
                        let next_val = self.mem[self.pos];
                        self.pos += 1;
                        Some(next_val)
                    } else {
                        let next_val = {
                            let n = self.pos;
                            let a = IndexOffset { slice: &self.mem, offset: n };
                            (a[n-1] + a[n-2])
                        };

                        {
                            use std::mem::swap;

                            let mut swap_tmp = next_val;
                            for i in (0..2).rev() {
                                swap(&mut swap_tmp, &mut self.mem[i]);
                            }
                        }

                        self.pos += 1;
                        Some(next_val)
                    }
                }
            }

            Recurrence { mem: [0, 1], pos: 0 }
        }
    };
}

fn main() {
    let fib = recurrence![a[n]: u64 = 0, 1, ..., a[n-1] + a[n-2]];

    for e in fib.take(10) { println!("{}", e) }
}
```

Очевидно, что мы не *используем* еще захват в метапеременные, но мы можем поменять это довольно просто. Однако, если мы попытаемся скомпилировать код, `rustc` вернет ошибку, говорящую нам:

```text
recurrence.rs:69:45: 69:48 error: local ambiguity: multiple parsing options: built-in NTs expr ('inits') or 1 other options.
recurrence.rs:69     let fib = recurrence![a[n]: u64 = 0, 1, ..., a[n-1] + a[n-2]];
                                                             ^~~
```

Вот тут мы и попались в ограничение `macro_rules`. Проблемой является вторая запятая. Когда `macro_rules`  видит ее во время развертывания, он не может решить парсить ему *следующее* выражение как `inits` или как `...`. К сожалению, он не такой умный, чтобы понять, что `...` не является валидным выражением, и поэтому он сдается. Теоретически, все должно работать, но нас самом деле не работает.

> **В сторону**: Я *сделал* этот раздел таким, каким он должен быть. В общем, он *должен был бы* работать как описано, но не в этом случае. Устройство `macro_rules`, как оно сейчас есть, имеет свои слабости, и всегда стоит помнить, что в некоторых случаях вам придется исказить код немного, чтобы заставить его работать.
>
> В этом *конкретном* случае две проблемы.  Первая - система макроса не знает, что составляет некоторые грамматические элементы, а что нет (*например*, выражения); это работа парсера. Поэтому, он не знает, что `...` не является выражением. Вторая - она не может попытаться захватить составной грамматический элемент(такой, как выражение), без его 100% захвата.
>
> Другими словами, она может попросить парсер попытаться и распарсить вход как выражение, а парсер ответит на любую проблему отменой с ошибкой. Единственным способом, как система макроса может работать с этим - просто попытаться избегать ситуаций, в которых это может стать проблем.
>
> К положительному моменту можно отнести то, что *никто* не в восторге от этого.  Ключевое слово `macro` уже зарезервировано для будущего более строго определения системы макроса. А пока, страдаем..

К счастью, решение очень простое: мы удаляем запятую из синтаксиса. Чтобы удержать равновесие, мы удалим  *обе* запятые вокруг `...`:

```rust
macro_rules! recurrence {
    ( a[n]: $sty:ty = $($inits:expr),+ ... $recur:expr ) => {
//                                     ^~~ changed
        /* ... */
#         // Cheat :D
#         (vec![0u64, 1, 2, 3, 5, 8, 13, 21, 34]).into_iter()
    };
}

fn main() {
    let fib = recurrence![a[n]: u64 = 0, 1 ... a[n-1] + a[n-2]];
//                                         ^~~ changed

    for e in fib.take(10) { println!("{}", e) }
}
```

Это успех! Теперь можем заменить вещи из *развертывания* на *захваченные метапеременные*.

### Замена

Замена того, что вы захватили в метапеременную, очень проста; вы заменяете содержимое захваченного в `$sty:ty` на `$sty`.  Итак, давайте пройдемая и исправим все `u64`:

```rust
macro_rules! recurrence {
    ( a[n]: $sty:ty = $($inits:expr),+ ... $recur:expr ) => {
        {
            use std::ops::Index;

            struct Recurrence {
                mem: [$sty; 2],
//                    ^~~~ changed
                pos: usize,
            }

            struct IndexOffset<'a> {
                slice: &'a [$sty; 2],
//                          ^~~~ changed
                offset: usize,
            }

            impl<'a> Index<usize> for IndexOffset<'a> {
                type Output = $sty;
//                            ^~~~ changed

                #[inline(always)]
                fn index<'b>(&'b self, index: usize) -> &'b $sty {
//                                                          ^~~~ changed
                    use std::num::Wrapping;

                    let index = Wrapping(index);
                    let offset = Wrapping(self.offset);
                    let window = Wrapping(2);

                    let real_index = index - offset + window;
                    &self.slice[real_index.0]
                }
            }

            impl Iterator for Recurrence {
                type Item = $sty;
//                          ^~~~ changed

                #[inline]
                fn next(&mut self) -> Option<$sty> {
//                                           ^~~~ changed
                    /* ... */
#                     if self.pos < 2 {
#                         let next_val = self.mem[self.pos];
#                         self.pos += 1;
#                         Some(next_val)
#                     } else {
#                         let next_val = {
#                             let n = self.pos;
#                             let a = IndexOffset { slice: &self.mem, offset: n };
#                             (a[n-1] + a[n-2])
#                         };
#     
#                         {
#                             use std::mem::swap;
#     
#                             let mut swap_tmp = next_val;
#                             for i in (0..2).rev() {
#                                 swap(&mut swap_tmp, &mut self.mem[i]);
#                             }
#                         }
#     
#                         self.pos += 1;
#                         Some(next_val)
#                     }
                }
            }

            Recurrence { mem: [1, 1], pos: 0 }
        }
    };
}

fn main() {
    let fib = recurrence![a[n]: u64 = 0, 1 ... a[n-1] + a[n-2]];

    for e in fib.take(10) { println!("{}", e) }
}
```

Давайте решим вопрос посложнее: как превратить `inits` в массив литералов `[0, 1]` *и* в массив типов, `[$sty; 2]`.  Для первого мы можем сделать так:

```ignore
            Recurrence { mem: [$($inits),+], pos: 0 }
//                             ^~~~~~~~~~~ изменено
```

Это полная противопложность захвату в метапеременные: повтор `inits` один или несколько раз, каждый отделяется запятой.  Это развернется в ожидаемую последовательность токенов: `0, 1`.

Почем-то превратить `inits` в литерал `2` немного сложнее. Оказывается, нет прямого способа это сделать, но мы *можем* исправить это, написав второй макрос. Сделаем оба преобразованием за один ход.

```rust
macro_rules! count_exprs {
    /* ??? */
#     () => {}
}
# fn main() {}
```

Очевидно: получив ноль выражений, вы ожидаете, что `count_exprs` развернется в литерал `0`.

```rust
macro_rules! count_exprs {
    () => (0);
//  ^~~~~~~~~~ добавлено
}
# fn main() {
#     const _0: usize = count_exprs!();
#     assert_eq!(_0, 0);
# }
```

> **В сторону**: Вы должно быть заметили, что я использую круглые скоби вместо фигурных для развертывания.  `macro_rules` все равно, *какие* скобки вы используете, если это любые из пар: `( )`, `{ }` или `[ ]`. На самом деле, вы можете использовать любые у самого макроса (*т.е.* скобки сразу после имени макроса), скрбки вокруг синтаксического правила или скобки вокруг соответствующего развертывания.
>
> Вы можете также использовать любые скобки при *вызове* макроса, но в более ограниченном количестве: макрос, вызываемый как  `{ ... }` или `( ... );` будет *всегда* парситься как *элемент* (*т.e.* как объявление `struct` или `fn`).  Это важно учитывать при использовании макросов внутри тела функций; это помогает устранить неоднозначность между "парсить как выражение" и "парсить как утверждение".

Что если у вас *одно* выражение? Этому должен соответствовать литерал `1`.

```rust
macro_rules! count_exprs {
    () => (0);
    ($e:expr) => (1);
//  ^~~~~~~~~~~~~~~~~ добавлено
}
# fn main() {
#     const _0: usize = count_exprs!();
#     const _1: usize = count_exprs!(x);
#     assert_eq!(_0, 0);
#     assert_eq!(_1, 1);
# }
```

Two?

```rust
macro_rules! count_exprs {
    () => (0);
    ($e:expr) => (1);
    ($e0:expr, $e1:expr) => (2);
//  ^~~~~~~~~~~~~~~~~~~~~~~~~~~~ добавлено
}
# fn main() {
#     const _0: usize = count_exprs!();
#     const _1: usize = count_exprs!(x);
#     const _2: usize = count_exprs!(x, y);
#     assert_eq!(_0, 0);
#     assert_eq!(_1, 1);
#     assert_eq!(_2, 2);
# }
```

Мы можем "упростить" это, по-другому выразив случай с двумя выражениями через рекурсию.

```rust
macro_rules! count_exprs {
    () => (0);
    ($e:expr) => (1);
    ($e0:expr, $e1:expr) => (1 + count_exprs!($e1));
//                           ^~~~~~~~~~~~~~~~~~~~~ изменено
}
# fn main() {
#     const _0: usize = count_exprs!();
#     const _1: usize = count_exprs!(x);
#     const _2: usize = count_exprs!(x, y);
#     assert_eq!(_0, 0);
#     assert_eq!(_1, 1);
#     assert_eq!(_2, 2);
# }
```

Это работает, Rust сможет преобразовать `1 + 1` в константное значение. Что если у нас три выражения?

```rust
macro_rules! count_exprs {
    () => (0);
    ($e:expr) => (1);
    ($e0:expr, $e1:expr) => (1 + count_exprs!($e1));
    ($e0:expr, $e1:expr, $e2:expr) => (1 + count_exprs!($e1, $e2));
//  ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~ добавлено
}
# fn main() {
#     const _0: usize = count_exprs!();
#     const _1: usize = count_exprs!(x);
#     const _2: usize = count_exprs!(x, y);
#     const _3: usize = count_exprs!(x, y, z);
#     assert_eq!(_0, 0);
#     assert_eq!(_1, 1);
#     assert_eq!(_2, 2);
#     assert_eq!(_3, 3);
# }
```

> **В сторону**: Вам должно быть интересно, можем ли мы поменять порядок следования правил. В данном конкретном случае, *да*, но система макроса может быть иногда требовательна к этому и не пожелает работать. Если вы напишите макрос с несколькими правилами, который вы кленетесь, что должен работать, но выдает ошибки на неожиданных токенах, попробуйте поменять порядок следования правил.

Слава богу, вы можете посмотреть паттерн ниже. Мы всегда можем уменьшить список выражений, выполняя совпадение с одним выражением, за которым следуют ноль и более выражений, разворачивая это в 1 + оставшееся количество выражений.

```rust
macro_rules! count_exprs {
    () => (0);
    ($head:expr) => (1);
    ($head:expr, $($tail:expr),*) => (1 + count_exprs!($($tail),*));
//  ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~ изменено
}
# fn main() {
#     const _0: usize = count_exprs!();
#     const _1: usize = count_exprs!(x);
#     const _2: usize = count_exprs!(x, y);
#     const _3: usize = count_exprs!(x, y, z);
#     assert_eq!(_0, 0);
#     assert_eq!(_1, 1);
#     assert_eq!(_2, 2);
#     assert_eq!(_3, 3);
# }
```

> **<abbr title="Только в этом примере">ТВЭП</abbr>**: это не *единственный*, и даже не *лучший* способ выполнить подсчет. Лучше использовать [Подсчет](blk-counting.html) из следующих глав.

Наконец, теперь мы можем изменить `recurrence` так, чтобы определить необходимый размер `mem`.

```rust
// добавлено:
macro_rules! count_exprs {
    () => (0);
    ($head:expr) => (1);
    ($head:expr, $($tail:expr),*) => (1 + count_exprs!($($tail),*));
}

macro_rules! recurrence {
    ( a[n]: $sty:ty = $($inits:expr),+ ... $recur:expr ) => {
        {
            use std::ops::Index;

            const MEM_SIZE: usize = count_exprs!($($inits),+);
//          ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~ добавлено

            struct Recurrence {
                mem: [$sty; MEM_SIZE],
//                          ^~~~~~~~ изменено
                pos: usize,
            }

            struct IndexOffset<'a> {
                slice: &'a [$sty; MEM_SIZE],
//                                ^~~~~~~~ изменено
                offset: usize,
            }

            impl<'a> Index<usize> for IndexOffset<'a> {
                type Output = $sty;

                #[inline(always)]
                fn index<'b>(&'b self, index: usize) -> &'b $sty {
                    use std::num::Wrapping;

                    let index = Wrapping(index);
                    let offset = Wrapping(self.offset);
                    let window = Wrapping(MEM_SIZE);
//                                        ^~~~~~~~ изменено

                    let real_index = index - offset + window;
                    &self.slice[real_index.0]
                }
            }

            impl Iterator for Recurrence {
                type Item = $sty;

                #[inline]
                fn next(&mut self) -> Option<$sty> {
                    if self.pos < MEM_SIZE {
//                                ^~~~~~~~ изменено
                        let next_val = self.mem[self.pos];
                        self.pos += 1;
                        Some(next_val)
                    } else {
                        let next_val = {
                            let n = self.pos;
                            let a = IndexOffset { slice: &self.mem, offset: n };
                            (a[n-1] + a[n-2])
                        };

                        {
                            use std::mem::swap;

                            let mut swap_tmp = next_val;
                            for i in (0..MEM_SIZE).rev() {
//                                       ^~~~~~~~ изменено
                                swap(&mut swap_tmp, &mut self.mem[i]);
                            }
                        }

                        self.pos += 1;
                        Some(next_val)
                    }
                }
            }

            Recurrence { mem: [$($inits),+], pos: 0 }
        }
    };
}
/* ... */
# 
# fn main() {
#     let fib = recurrence![a[n]: u64 = 0, 1 ... a[n-1] + a[n-2]];
# 
#     for e in fib.take(10) { println!("{}", e) }
# }
```

Сделав это, мы можем заменить последнюю вещь: выражение `recur`.

```ignore
# macro_rules! count_exprs {
#     () => (0);
#     ($head:expr $(, $tail:expr)*) => (1 + count_exprs!($($tail),*));
# }
# macro_rules! recurrence {
#     ( a[n]: $sty:ty = $($inits:expr),+ ... $recur:expr ) => {
#         {
#             const MEMORY: uint = count_exprs!($($inits),+);
#             struct Recurrence {
#                 mem: [$sty; MEMORY],
#                 pos: uint,
#             }
#             struct IndexOffset<'a> {
#                 slice: &'a [$sty; MEMORY],
#                 offset: uint,
#             }
#             impl<'a> Index<uint, $sty> for IndexOffset<'a> {
#                 #[inline(always)]
#                 fn index<'b>(&'b self, index: &uint) -> &'b $sty {
#                     let real_index = *index - self.offset + MEMORY;
#                     &self.slice[real_index]
#                 }
#             }
#             impl Iterator<u64> for Recurrence {
/* ... */
                #[inline]
                fn next(&mut self) -> Option<u64> {
                    if self.pos < MEMORY {
                        let next_val = self.mem[self.pos];
                        self.pos += 1;
                        Some(next_val)
                    } else {
                        let next_val = {
                            let n = self.pos;
                            let a = IndexOffset { slice: &self.mem, offset: n };
                            $recur
//                          ^~~~~~ changed
                        };
                        {
                            use std::mem::swap;
                            let mut swap_tmp = next_val;
                            for i in range(0, MEMORY).rev() {
                                swap(&mut swap_tmp, &mut self.mem[i]);
                            }
                        }
                        self.pos += 1;
                        Some(next_val)
                    }
                }
/* ... */
#             }
#             Recurrence { mem: [$($inits),+], pos: 0 }
#         }
#     };
# }
# fn main() {
#     let fib = recurrence![a[n]: u64 = 1, 1 ... a[n-1] + a[n-2]];
#     for e in fib.take(10) { println!("{}", e) }
# }
```

И, когда мы скомпилируем наш законченный макрос...

```text
recurrence.rs:77:48: 77:49 error: unresolved name `a`
recurrence.rs:77     let fib = recurrence![a[n]: u64 = 0, 1 ... a[n-1] + a[n-2]];
                                                                ^
recurrence.rs:7:1: 74:2 note: in expansion of recurrence!
recurrence.rs:77:15: 77:64 note: expansion site
recurrence.rs:77:50: 77:51 error: unresolved name `n`
recurrence.rs:77     let fib = recurrence![a[n]: u64 = 0, 1 ... a[n-1] + a[n-2]];
                                                                  ^
recurrence.rs:7:1: 74:2 note: in expansion of recurrence!
recurrence.rs:77:15: 77:64 note: expansion site
recurrence.rs:77:57: 77:58 error: unresolved name `a`
recurrence.rs:77     let fib = recurrence![a[n]: u64 = 0, 1 ... a[n-1] + a[n-2]];
                                                                         ^
recurrence.rs:7:1: 74:2 note: in expansion of recurrence!
recurrence.rs:77:15: 77:64 note: expansion site
recurrence.rs:77:59: 77:60 error: unresolved name `n`
recurrence.rs:77     let fib = recurrence![a[n]: u64 = 0, 1 ... a[n-1] + a[n-2]];
                                                                           ^
recurrence.rs:7:1: 74:2 note: in expansion of recurrence!
recurrence.rs:77:15: 77:64 note: expansion site
```

... постойте, что? Так быть не должно... проверим, во что разворачивается макрос.

```shell
$ rustc -Z unstable-options --pretty expanded recurrence.rs
```

Аргумент `--pretty expanded` говорит `rustc` выполнить развертывание макроса, и затем вернуть получившееся AST обратно в исходный код. Эта опция не считается стабильной, поэтому надо указать `-Z unstable-options`.  Выход (после форматирования) показан ниже; в частности, обратите внимание на то место в коде, где `$recur` был заменен:

```ignore
#![feature(no_std)]
#![no_std]
#[prelude_import]
use std::prelude::v1::*;
#[macro_use]
extern crate std as std;
fn main() {
    let fib = {
        use std::ops::Index;
        const MEM_SIZE: usize = 1 + 1;
        struct Recurrence {
            mem: [u64; MEM_SIZE],
            pos: usize,
        }
        struct IndexOffset<'a> {
            slice: &'a [u64; MEM_SIZE],
            offset: usize,
        }
        impl <'a> Index<usize> for IndexOffset<'a> {
            type Output = u64;
            #[inline(always)]
            fn index<'b>(&'b self, index: usize) -> &'b u64 {
                use std::num::Wrapping;
                let index = Wrapping(index);
                let offset = Wrapping(self.offset);
                let window = Wrapping(MEM_SIZE);
                let real_index = index - offset + window;
                &self.slice[real_index.0]
            }
        }
        impl Iterator for Recurrence {
            type Item = u64;
            #[inline]
            fn next(&mut self) -> Option<u64> {
                if self.pos < MEM_SIZE {
                    let next_val = self.mem[self.pos];
                    self.pos += 1;
                    Some(next_val)
                } else {
                    let next_val = {
                        let n = self.pos;
                        let a = IndexOffset{slice: &self.mem, offset: n,};
                        a[n - 1] + a[n - 2]
                    };
                    {
                        use std::mem::swap;
                        let mut swap_tmp = next_val;
                        {
                            let result =
                                match ::std::iter::IntoIterator::into_iter((0..MEM_SIZE).rev()) {
                                    mut iter => loop {
                                        match ::std::iter::Iterator::next(&mut iter) {
                                            ::std::option::Option::Some(i) => {
                                                swap(&mut swap_tmp, &mut self.mem[i]);
                                            }
                                            ::std::option::Option::None => break,
                                        }
                                    },
                                };
                            result
                        }
                    }
                    self.pos += 1;
                    Some(next_val)
                }
            }
        }
        Recurrence{mem: [0, 1], pos: 0,}
    };
    {
        let result =
            match ::std::iter::IntoIterator::into_iter(fib.take(10)) {
                mut iter => loop {
                    match ::std::iter::Iterator::next(&mut iter) {
                        ::std::option::Option::Some(e) => {
                            ::std::io::_print(::std::fmt::Arguments::new_v1(
                                {
                                    static __STATIC_FMTSTR: &'static [&'static str] = &["", "\n"];
                                    __STATIC_FMTSTR
                                },
                                &match (&e,) {
                                    (__arg0,) => [::std::fmt::ArgumentV1::new(__arg0, ::std::fmt::Display::fmt)],
                                }
                            ))
                        }
                        ::std::option::Option::None => break,
                    }
                },
            };
        result
    }
}
```

Но все вроде в порядке! Если мы добавим несколько не хватающих атрибутов `#![feature(...)]` и запустим под ночной сборкой `rustc`, он даже комплиируется!  ... *как?!*

> **В сторону**: У вас не получится скомпилировать это на не-ночной сборке `rustc`.  Происходит это из-за того, что развертывания макроса `println!` зависит от внутренних деталей компилятора, которые еще пубично *не* стабилизированы.

### Соблюдая гигену

Проблема здесь в том, что идентификаторы в макросах Rust обладают *гигиеной*. Поэтому идентификаторы из двух разных контекстов *не могут* сталкиваться. Чтобы объяснить разницу, возьмем простой пример.

```rust
# /*
macro_rules! using_a {
    ($e:expr) => {
        {
            let a = 42i;
            $e
        }
    }
}

let four = using_a!(a / 10);
# */
# fn main() {}
```

Макрос просто принимает выражение, оборачивает его в блок и определяет переменную `a`.  Мы используем его как окольный путь вычисления `4`.  На самом деле здесь *два* контекста синтаксиса, но они невидимы. Поэтомы, для помощи вам, дадим каждому контексту свой цвет. Начнем с неразвернутого кода, в котором только один контекст:

<pre class="rust rust-example-rendered"><span class="synctx-0"><span class="macro">macro_rules</span><span class="macro">!</span> <span class="ident">using_a</span> {&#xa;    (<span class="macro-nonterminal">$</span><span class="macro-nonterminal">e</span>:<span class="ident">expr</span>) <span class="op">=&gt;</span> {&#xa;        {&#xa;            <span class="kw">let</span> <span class="ident">a</span> <span class="op">=</span> <span class="number">42</span>;&#xa;            <span class="macro-nonterminal">$</span><span class="macro-nonterminal">e</span>&#xa;        }&#xa;    }&#xa;}&#xa;&#xa;<span class="kw">let</span> <span class="ident">four</span> <span class="op">=</span> <span class="macro">using_a</span><span class="macro">!</span>(<span class="ident">a</span> <span class="op">/</span> <span class="number">10</span>);</span></pre>

Теперь, развернем вызов.

<pre class="rust rust-example-rendered"><span class="synctx-0"><span class="kw">let</span> <span class="ident">four</span> <span class="op">=</span> </span><span class="synctx-1">{&#xa;    <span class="kw">let</span> <span class="ident">a</span> <span class="op">=</span> <span class="number">42</span>;&#xa;    </span><span class="synctx-0"><span class="ident">a</span> <span class="op">/</span> <span class="number">10</span></span><span class="synctx-1">&#xa;}</span><span class="synctx-0">;</span></pre>

Как видно, <code><span class="synctx-1">a</span></code>, определяемая макросом находится в другом контексте по отношению к <code><span class="synctx-0">a</span></code>, которую мы подсунули в наш вызов.  Поэтому компилятор считает их абсолютно разными идентификаторами, *не принимая во внимания, что у них одинаковое лексическое представление*.

Это то, с чем надо быть *особенно* осторожным при работе с макросами: макросы могут софрмировать AST, которые не будут компилироваться, но, которые, если написать от руки или с использованием `--pretty expanded`, все же *скомпилируются*.

Решением здесь является захватить идентификатор *с подходящим контекстом синтаксиса*. Чтобы сделать это, надо снова улучшить синтаксис нашего макроса. Продолжая с нашим простым примером:

<pre class="rust rust-example-rendered"><span class="synctx-0"><span class="macro">macro_rules</span><span class="macro">!</span> <span class="ident">using_a</span> {&#xa;    (<span class="macro-nonterminal">$</span><span class="macro-nonterminal">a</span>:<span class="ident">ident</span>, <span class="macro-nonterminal">$</span><span class="macro-nonterminal">e</span>:<span class="ident">expr</span>) <span class="op">=&gt;</span> {&#xa;        {&#xa;            <span class="kw">let</span> <span class="macro-nonterminal">$</span><span class="macro-nonterminal">a</span> <span class="op">=</span> <span class="number">42</span>;&#xa;            <span class="macro-nonterminal">$</span><span class="macro-nonterminal">e</span>&#xa;        }&#xa;    }&#xa;}&#xa;&#xa;<span class="kw">let</span> <span class="ident">four</span> <span class="op">=</span> <span class="macro">using_a</span><span class="macro">!</span>(<span class="ident">a</span>, <span class="ident">a</span> <span class="op">/</span> <span class="number">10</span>);</span></pre>

Это развернется в:

<pre class="rust rust-example-rendered"><span class="synctx-0"><span class="kw">let</span> <span class="ident">four</span> <span class="op">=</span> </span><span class="synctx-1">{&#xa;    <span class="kw">let</span> </span><span class="synctx-0"><span class="ident">a</span></span><span class="synctx-1"> <span class="op">=</span> <span class="number">42</span>;&#xa;    </span><span class="synctx-0"><span class="ident">a</span> <span class="op">/</span> <span class="number">10</span></span><span class="synctx-1">&#xa;}</span><span class="synctx-0">;</span></pre>

Сейчас контексты совпадают, и код скомпилируется. Можем аналогичным образом изменить наш `recurrence!`, явно захватывая `a` и `n`.  После изменений, получим:

```rust
macro_rules! count_exprs {
    () => (0);
    ($head:expr) => (1);
    ($head:expr, $($tail:expr),*) => (1 + count_exprs!($($tail),*));
}

macro_rules! recurrence {
    ( $seq:ident [ $ind:ident ]: $sty:ty = $($inits:expr),+ ... $recur:expr ) => {
//    ^~~~~~~~~~   ^~~~~~~~~~ изменено
        {
            use std::ops::Index;

            const MEM_SIZE: usize = count_exprs!($($inits),+);

            struct Recurrence {
                mem: [$sty; MEM_SIZE],
                pos: usize,
            }

            struct IndexOffset<'a> {
                slice: &'a [$sty; MEM_SIZE],
                offset: usize,
            }

            impl<'a> Index<usize> for IndexOffset<'a> {
                type Output = $sty;

                #[inline(always)]
                fn index<'b>(&'b self, index: usize) -> &'b $sty {
                    use std::num::Wrapping;

                    let index = Wrapping(index);
                    let offset = Wrapping(self.offset);
                    let window = Wrapping(MEM_SIZE);

                    let real_index = index - offset + window;
                    &self.slice[real_index.0]
                }
            }

            impl Iterator for Recurrence {
                type Item = $sty;

                #[inline]
                fn next(&mut self) -> Option<$sty> {
                    if self.pos < MEM_SIZE {
                        let next_val = self.mem[self.pos];
                        self.pos += 1;
                        Some(next_val)
                    } else {
                        let next_val = {
                            let $ind = self.pos;
//                              ^~~~ изменено
                            let $seq = IndexOffset { slice: &self.mem, offset: $ind };
//                              ^~~~ изменено
                            $recur
                        };

                        {
                            use std::mem::swap;

                            let mut swap_tmp = next_val;
                            for i in (0..MEM_SIZE).rev() {
                                swap(&mut swap_tmp, &mut self.mem[i]);
                            }
                        }

                        self.pos += 1;
                        Some(next_val)
                    }
                }
            }

            Recurrence { mem: [$($inits),+], pos: 0 }
        }
    };
}

fn main() {
    let fib = recurrence![a[n]: u64 = 0, 1 ... a[n-1] + a[n-2]];

    for e in fib.take(10) { println!("{}", e) }
}
```

И он скомпилировался! Теперь попробуем другую последовательность.

```rust
# macro_rules! count_exprs {
#     () => (0);
#     ($head:expr) => (1);
#     ($head:expr, $($tail:expr),*) => (1 + count_exprs!($($tail),*));
# }
# 
# macro_rules! recurrence {
#     ( $seq:ident [ $ind:ident ]: $sty:ty = $($inits:expr),+ ... $recur:expr ) => {
#         {
#             use std::ops::Index;
#             
#             const MEM_SIZE: usize = count_exprs!($($inits),+);
#     
#             struct Recurrence {
#                 mem: [$sty; MEM_SIZE],
#                 pos: usize,
#             }
#     
#             struct IndexOffset<'a> {
#                 slice: &'a [$sty; MEM_SIZE],
#                 offset: usize,
#             }
#     
#             impl<'a> Index<usize> for IndexOffset<'a> {
#                 type Output = $sty;
#     
#                 #[inline(always)]
#                 fn index<'b>(&'b self, index: usize) -> &'b $sty {
#                     use std::num::Wrapping;
#                     
#                     let index = Wrapping(index);
#                     let offset = Wrapping(self.offset);
#                     let window = Wrapping(MEM_SIZE);
#                     
#                     let real_index = index - offset + window;
#                     &self.slice[real_index.0]
#                 }
#             }
#     
#             impl Iterator for Recurrence {
#                 type Item = $sty;
#     
#                 #[inline]
#                 fn next(&mut self) -> Option<$sty> {
#                     if self.pos < MEM_SIZE {
#                         let next_val = self.mem[self.pos];
#                         self.pos += 1;
#                         Some(next_val)
#                     } else {
#                         let next_val = {
#                             let $ind = self.pos;
#                             let $seq = IndexOffset { slice: &self.mem, offset: $ind };
#                             $recur
#                         };
#     
#                         {
#                             use std::mem::swap;
#     
#                             let mut swap_tmp = next_val;
#                             for i in (0..MEM_SIZE).rev() {
#                                 swap(&mut swap_tmp, &mut self.mem[i]);
#                             }
#                         }
#     
#                         self.pos += 1;
#                         Some(next_val)
#                     }
#                 }
#             }
#     
#             Recurrence { mem: [$($inits),+], pos: 0 }
#         }
#     };
# }
# 
# fn main() {
for e in recurrence!(f[i]: f64 = 1.0 ... f[i-1] * i as f64).take(10) {
    println!("{}", e)
}
# }
```

Получим:

```text
1
1
2
6
24
120
720
5040
40320
362880
```

Это победа!
