# 学习 Rust 标准库

## 智能指针

### Cow

std: [Cow](https://doc.rust-lang.org/stable/std/borrow/enum.Cow.html)

> 一种“写时克隆”（clone-on-write）智能指针。
>
> 类型 `Cow` 是一个智能指针，提供写时克隆功能：它可以封装并对借用的数据提供不可变访问，只有在需要修改或获取所有权时才会延迟地对数据进行克隆。该类型通过 `Borrow` 特征设计为可与通用的借用数据一起使用。
>
> `Cow` 实现了 `Deref`，这意味着可以直接在其封装的数据上调用非修改性（non-mutating）的方法。如果需要进行修改，调用 `to_mut` 会获取一个对已拥有值的可变引用，并在必要时进行克隆。
>
> 如果你需要引用计数指针，注意 `Rc::make_mut` 和 `Arc::make_mut` 同样可以提供写时克隆的功能。

官方例子

```rust
use std::borrow::Cow;

struct Items<'a, X>
where
    [X]: ToOwned<Owned = Vec<X>>,
{
    values: Cow<'a, [X]>,
}

impl<'a, X: Clone + 'a> Items<'a, X>
where
    [X]: ToOwned<Owned = Vec<X>>,
{
    fn new(v: Cow<'a, [X]>) -> Self {
        Items { values: v }
    }
}

fn main() {
    // Creates a container from borrowed values of a slice
    let readonly = [1, 2];
    let borrowed = Items::new((&readonly[..]).into());
    match borrowed {
        Items {
            values: Cow::Borrowed(b),
        } => println!("borrowed {b:?}"),
        _ => panic!("expect borrowed value"),
    }

    let mut clone_on_write = borrowed;
    // Mutates the data from slice into owned vec and pushes a new value on top
    clone_on_write.values.to_mut().push(3);
    println!("clone_on_write = {:?}", clone_on_write.values);

    // The data was mutated. Let's check it out.
    match clone_on_write {
        Items {
            values: Cow::Owned(_),
        } => println!("clone_on_write contains owned data"),
        _ => panic!("expect owned data"),
    }
}
```

#### QA

- 如何理解 ``where [X]: ToOwned<Owned = Vec<X>>``

`[X]` 类型实现了 `ToOwned` trait，且它的关联类型 `Owned` 是 `Vec<X>`。

##### 背景知识：ToOwned 有何作用？

Rust 中 `ToOwned` 是一个 trait，用于从借用数据创建拥有数据（owned copy），可以看作是 `Clone` 的通用版本：

```rust
trait ToOwned {
    type Owned;
    fn to_owned(&self) -> Self::Owned;
}
```

举个例子：

- `str`（字符串切片）实现了 `ToOwned`，其 `Owned` 是 `String`
- `[T]`（切片）实现了 `ToOwned`，其 `Owned` 是 `Vec<T>`

也就是说：

```rust
let s: &str = "hi";
let s_owned: String = s.to_owned(); // ✅

let arr: &[i32] = &[1, 2, 3];
let arr_owned: Vec<i32> = arr.to_owned(); // ✅
```

所以 `where [X]: ToOwned<Owned = Vec<X>>` 是在说：

> 只要 `X` 的切片 `[X]` 实现了 `ToOwned`（返回 `Vec<X>`），我们就可以把 `Cow<[X]>` 从 Borrowed 转换成 Owned。



- 如何理解 ``(&readonly[..]).into()``

这行是 Rust 中非常常见的一种写法，把 slice 转换成 `Cow`。

```rust
let readonly = [1, 2];
let borrowed = Items::new((&readonly[..]).into());
```

逐步拆解：

- `readonly[..]`：从数组 `[1, 2]` 获取一个切片（类型是 `&[i32]`）
- `&readonly[..]`：这已经是个切片了，但加括号是为了清晰
- `.into()`：Rust 有一个从 `&[T]` 到 `Cow<[T]>` 的 `From` 实现，调用 `.into()` 会自动得到 `Cow::Borrowed`

等价于：

```rust
let borrowed: Cow<[i32]> = Cow::Borrowed(&readonly[..]);
```



- 但是我没有实现 (&readonly[..]).into() 功能，怎么会自动转呢?

确实你**没有显式实现** `(&readonly[..]).into()`，但它**“自动转”**的原因在于：Rust 标准库**已经为我们提供了这个类型的实现**，它是通过 `From` 和 `Into` 的 trait 系统完成的。

##### 💡 TL;DR（简要回答）

你能写 `.into()` 是因为：

```
rust


CopyEdit
impl<'a, T: ToOwned + ?Sized> From<&'a T> for Cow<'a, T>
```

这个 trait **已经在标准库里为你实现了**！

------

##### 🧠 深入解释：Rust 的 `From` / `Into` 机制

Rust 有两个紧密关联的 trait：

```
rust


CopyEdit
trait From<T> {
    fn from(value: T) -> Self;
}

trait Into<T> {
    fn into(self) -> T;
}
```

并且有个很重要的默认规则：

> 如果你实现了 `From<T> for U`，你就**自动获得了** `Into<U> for T`！

------

##### ✅ 回到你的代码：

你写的是：

```rust
(&readonly[..]).into()
```

类型是：

```rust
&[i32] → Cow<'_, [i32]>
```

##### 📦 在标准库中：

```rust
impl<'a, B> From<&'a B> for Cow<'a, B>
where
    B: ToOwned + ?Sized,
```

这是 Rust 标准库里的实现，你可以在 官方文档 中看到。

对于 `&[i32]`：

- 它是 `B = [i32]`
- `[i32]: ToOwned<Owned = Vec<i32>>`
- 所以可以 `From<&'a [i32]> for Cow<'a, [i32]>`

于是 `.into()` 自动生效！

------

##### 👇 换个写法：用 `.from()` 显式写出来

你这句：

```rust
let borrowed = Items::new((&readonly[..]).into());
```

等价于下面这两种写法：

```rust
let borrowed = Items::new(Cow::from(&readonly[..]));
let borrowed: Items<i32> = Items::new(Cow::Borrowed(&readonly[..]));
```

所以 `.into()` 只是调用了 `Cow::from(...)` 的语法糖。

---

> Author: [asthetik](https://github.com/asthetik)  
> URL: https://asthetik.github.io/posts/rust/learn-the-rust-std/  

