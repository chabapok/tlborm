% Идентификаторы, не являющиеся идентификаторами

Есть два токена, с которыми вы, вероятно, сталкивались когда-либо, которые
*выглядят* как идентификаторы, но ими не являются.  Кроме случаев, когда они
выступают именно в роли идентификаторов.

Первый - `self`. Это  *совершенно точно* ключевое слово. Однако, ему также
случается быть определением идентификатора. В обычном коде на Rust `self` не
может быть интерпретирован как идентификатор, но в макросе такая возможность
*есть*:

```rust
macro_rules! what_is {
    (self) => {"the keyword `self`"};
    ($i:ident) => {concat!("the identifier `", stringify!($i), "`")};
}

macro_rules! call_with_ident {
    ($c:ident($i:ident)) => {$c!($i)};
}

fn main() {
    println!("{}", what_is!(self));
    println!("{}", call_with_ident!(what_is(self)));
}
```

Вывод следующий:

```text
the keyword `self`
the keyword `self`
```

Но в этом нет смысла; `call_with_ident!` требует идентификатор, находит
совпадение с образцом, и заменяет его! Получается `self` в одно и то же время
как является ключевым словом, так им и не является. Возможно, вам непонятно, в
каких случаях это может быть важно. Рассмотрим такой пример:

```ignore
macro_rules! make_mutable {
    ($i:ident) => {let mut $i = $i;};
}

struct Dummy(i32);

impl Dummy {
    fn double(self) -> Dummy {
        make_mutable!(self);
        self.0 *= 2;
        self
    }
}
# 
# fn main() {
#     println!("{:?}", Dummy(4).double().0);
# }
```

Возникнет ошибка при компиляции:

```text
<anon>:2:28: 2:30 error: expected identifier, found keyword `self`
<anon>:2     ($i:ident) => {let mut $i = $i;};
                                    ^~
```

То есть макрос спокойно и успешно сопоставил `self` с идентификатором, позволив
вам использовать его в случае, когда вам на самом деле так его использовать
нельзя. Но, постойте; он каким-то образом помнит еще и что `self` является
ключевым словом, даже если это идентификатор. Поэтому, пример ниже будет
работать, так ведь?

```ignore
macro_rules! make_self_mutable {
    ($i:ident) => {let mut $i = self;};
}

struct Dummy(i32);

impl Dummy {
    fn double(self) -> Dummy {
        make_self_mutable!(mut_self);
        mut_self.0 *= 2;
        mut_self
    }
}
# 
# fn main() {
#     println!("{:?}", Dummy(4).double().0);
# }
```

Здесь выдаётся ошибка:

```text
<anon>:2:33: 2:37 error: `self` is not available in a static method. Maybe a `self` argument is missing? [E0424]
<anon>:2     ($i:ident) => {let mut $i = self;};
                                         ^~~~
```

Это тоже не имеет никакого смысла. `self` и не находится в статическом методе.
Очень похоже, что компилятор жалуется, что `self`, который он пытается
использовать - это не тот же самый `self`... получается у ключевого слова `self`
гигиена, как у... идентификатора.

```ignore
macro_rules! double_method {
    ($body:expr) => {
        fn double(mut self) -> Dummy {
            $body
        }
    };
}

struct Dummy(i32);

impl Dummy {
    double_method! {{
        self.0 *= 2;
        self
    }}
}
# 
# fn main() {
#     println!("{:?}", Dummy(4).double().0);
# }
```

Та же ошибка.  Что насчет...

```rust
macro_rules! double_method {
    ($self_:ident, $body:expr) => {
        fn double(mut $self_) -> Dummy {
            $body
        }
    };
}

struct Dummy(i32);

impl Dummy {
    double_method! {self, {
        self.0 *= 2;
        self
    }}
}
# 
# fn main() {
#     println!("{:?}", Dummy(4).double().0);
# }
```

Наконец-то, *заработало*.  Получается `self` является как ключевым словом, *так
и* идентификатором - как ему захочется. Конечно, это сработает и для
других, похожих конструкций, не так ли?

```ignore
macro_rules! double_method {
    ($self_:ident, $body:expr) => {
        fn double($self_) -> Dummy {
            $body
        }
    };
}

struct Dummy(i32);

impl Dummy {
    double_method! {_, 0}
}
# 
# fn main() {
#     println!("{:?}", Dummy(4).double().0);
# }
```

```text
<anon>:12:21: 12:22 error: expected ident, found _
<anon>:12     double_method! {_, 0}
                              ^
```

Нет, конечно нет.  `_` это ключевое слово, которое правильно в образцах и
выражениях, но при этом *не является* идентификатором в том смысле, как им
является `self`, несмотря на то, что описание идентификатора абсолютно такое же.

Вам может показаться, что это можно обойти, используя  `$self_:pat`; и в таком
случае, `_` подойдет!  За исключением того, что... нет, не подойдет, потому что
`self` не является образцом. Улыбаемся и машем.

Как-то обойти это в данном случае (в случае если вам нужна комбинация этих
токенов) можно, используя совпадения с образцом по `tt` вместо этого.
