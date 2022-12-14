# Demo 3
## `Pin` 、 `UnPin`

自引用类型：
```rust
struct Test {
    a: String,
    p2a: *const String,
}

impl Test {
    fn new(txt: &str) -> Self {
        Test {
            a: String::from(txt),
            p2a: std::ptr::null(),
        }
    }

    fn init(&mut self) {
        let self_ref: *const String = &self.a;
        self.p2a = self_ref;
    }

    fn a(&self) -> &str {
        &self.a
    }

    fn p2a(&self) -> &String {
        assert!(!self.p2a.is_null(), "Call Test::init first");
        unsafe { &*(self.p2a)}
    }
}

fn main() {
    let mut test1 = Test::new("test1");
    let mut test2 = Test::new("test2");
    test1.init();
    test2.init();

    println!("a: {}, p2a: {}", test1.a(), test1.p2a());
    std::mem::swap(&mut test1, &mut test2);
    println!("a: {}, p2a: {}", test2.a(), test2.p2a());

    //
    //      0x11
    //      ---------   ---------
    //      | test1 |   | 0x11  |
    //      ---------   ---------
    //      0x22
    //      ---------   ---------
    //      | test2 |   | 0x22  |
    //      ---------   ---------
    // swap:
    //      0x11
    //      ---------   ---------
    //      | test2 |   | 0x22  |
    //      ---------   ---------
    //      0x22
    //      ---------   ---------
    //      | test1 |   | 0x11  |
    //      ---------   ---------
    //
}

```

Rust 中大部分类型有默认实现了 `UnPin`，这是一个 trait，表示该类型的值是可以移动的（move）。

实现了 `!Unpined` 的类型，表示 **没有实现 `UnPin`**，不是固定、无法移动的意思（负负得正），而是表示 **可以固定**，需要显式的通过 `Pin` 将其固定，无法移动。

```rust
use std::pin::Pin;
use std::marker::PhantomPinned;

struct Test {
    a: String,
    p2a: *const String,
    _marker: PhantomPinned,
}

impl Test {
    fn new(txt: &str) -> Self {
        Test {
            a: String::from(txt),
            p2a: std::ptr::null(),
            _marker: PhantomPinned,
        }
    }

    fn init(self: Pin<&mut Self>) {
        let self_ptr: *const String = &self.a;
        let this = unsafe {self.get_unchecked_mut()};
        this.p2a = self_ptr;
    }

    fn a(self: Pin<&Self>) -> &str {
        &self.get_ref().a
    }

    fn p2a(self: Pin<&Self>) -> &String {
        assert!(!self.p2a.is_null(), "Call Test::init first");
        unsafe { &*(self.p2a)}
    }
}

fn main() {
    let mut test1 = Test::new("test1");
    let mut test1 = unsafe { Pin::new_unchecked(&mut test1)};
    Test::init(test1.as_mut());

    let mut test2 = Test::new("test2");
    let mut test2 = unsafe { Pin::new_unchecked(&mut test2)};
    Test::init(test2.as_mut());

    println!("a: {}, p2a: {}", Test::a(test1.as_ref()), Test::p2a(test1.as_ref()));
    std::mem::swap(test1.get_mut(), test2.get_mut());
    println!("a: {}, p2a: {}", Test::a(test2.as_ref()), Test::p2a(test2.as_ref()));
}
```

`Future` 通过 `poll` `.await` 被执行时，是在另外一个状态机里运行，如果期间有引用的数据，当数据源被移动（或销毁），就会造成类似悬垂引用的内存错误，所以数据源一定要被 `Pin` 住。
