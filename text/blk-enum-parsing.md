% Парсинг Enum 
 
```rust
macro_rules! parse_unitary_variants {
    (@as_expr $e:expr) => {$e};
    (@as_item $($i:item)+) => {$($i)+};
    
    // Правила выхода.
    (
        @collect_unitary_variants ($callback:ident ( $($args:tt)* )),
        ($(,)*) -> ($($var_names:ident,)*)
    ) => {
        parse_unitary_variants! {
            @as_expr
            $callback!{ $($args)* ($($var_names),*) }
        }
    };

    (
        @collect_unitary_variants ($callback:ident { $($args:tt)* }),
        ($(,)*) -> ($($var_names:ident,)*)
    ) => {
        parse_unitary_variants! {
            @as_item
            $callback!{ $($args)* ($($var_names),*) }
        }
    };

    // Поглощение атрибута.
    (
        @collect_unitary_variants $fixed:tt,
        (#[$_attr:meta] $($tail:tt)*) -> ($($var_names:tt)*)
    ) => {
        parse_unitary_variants! {
            @collect_unitary_variants $fixed,
            ($($tail)*) -> ($($var_names)*)
        }
    };

    // Обработка варианта, дополнительно: с инициализацией 
    (
        @collect_unitary_variants $fixed:tt,
        ($var:ident $(= $_val:expr)*, $($tail:tt)*) -> ($($var_names:tt)*)
    ) => {
        parse_unitary_variants! {
            @collect_unitary_variants $fixed,
            ($($tail)*) -> ($($var_names)* $var,)
        }
    };

    // Отмена варианта с полезной нагрузкой (payload) 
    (
        @collect_unitary_variants $fixed:tt,
        ($var:ident $_struct:tt, $($tail:tt)*) -> ($($var_names:tt)*)
    ) => {
        const _error: () = "cannot parse unitary variants from enum with non-unitary variants";
    };
    
    // Правило входа.
    (enum $name:ident {$($body:tt)*} => $callback:ident $arg:tt) => {
        parse_unitary_variants! {
            @collect_unitary_variants
            ($callback $arg), ($($body)*,) -> ()
        }
    };
}
# 
# fn main() {
#     assert_eq!(
#         parse_unitary_variants!(
#             enum Dummy { A, B, C }
#             => stringify(variants:)
#         ),
#         "variants : ( A , B , C )"
#     );
# }
```

Этот макрос показывает, как вы можете использовать [incremental tt muncher] и [push-down accumulation] для парсинга вариантов `enum` , в котором все варианты унитарны (*т.e.* не имеют полезной нагрузки (payload)).  После завершения, `parse_unitary_variants!` вызываеи макрос [callback] со списком вариантов (плюс поддерживает любые другие произвольные аргументы).

Его можно изменить, чтобы также парсить поля  `struct` , вычислять дополнительные значения для вариантов, или даже выделять имена  *всех* вариантов в произвольный  `enum`.

[incremental tt muncher]: pat-incremental-tt-munchers.html
[push-down accumulation]: pat-push-down-accumulation.html
[callback]: pat-callbacks.html
