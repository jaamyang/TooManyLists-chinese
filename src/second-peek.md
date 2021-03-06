# Peek
# Peek方法

One thing we didn't even bother to implement last time was peeking. Let's go
ahead and do that. All we need to do is return a reference to the element in
the head of the list, if it exists. Sounds easy, let's try:

上一章中我们没有提到与实现Peek方法，现在就让我们来实现它吧。要实现Peek方法只需要在list头元素存在的情况下返回其引用便可以了。貌似很容易，来试一试：

```rust ,ignore
pub fn peek(&self) -> Option<&T> {
    self.head.map(|node| {
        &node.elem
    })
}
```


```text
> cargo build

error[E0515]: cannot return reference to local data `node.elem`
  --> src/second.rs:37:13
   |
37 |             &node.elem
   |             ^^^^^^^^^^ returns a reference to data owned by the current function

error[E0507]: cannot move out of borrowed content
  --> src/second.rs:36:9
   |
36 |         self.head.map(|node| {
   |         ^^^^^^^^^ cannot move out of borrowed content


```

*Sigh*. What now, Rust?
诶，什么情况？

Map takes `self` by value, which would move the Option out of the thing it's in.
Previously this was fine because we had just `take`n it out, but now we actually
want to leave it where it was. The *correct* way to handle this is with the
`as_ref` method on Option, which has the following definition:
map通过传值拿到了`self`，它将self.head中Option里存放的数据拿出来。之前的Ok的原因是我们仅仅是将其中的数据拿出来，但现在我们却想要离开它的作用域。*正确*的做法是通过`as_ref`方法来处理Option，它的定义如下：


```rust ,ignore
impl<T> Option<T> {
    pub fn as_ref(&self) -> Option<&T>;
}
```

It demotes the Option<T> to an Option to a reference to its internals. We could
do this ourselves with an explicit match but *ugh no*. It does mean that we
need to do an extra dereference to cut through the extra indirection, but
thankfully the `.` operator handles that for us.
这一步的操作使Option<T>内部的T变为一个引用。我们也可以自己通过显式的match来实现，但还是*算了吧*。使用`as_ref`意味着我们需要做一些额外的操作，但幸好我们有`.`这一操作符帮我们处理了这一繁琐的工作。



```rust ,ignore
pub fn peek(&self) -> Option<&T> {
    self.head.as_ref().map(|node| {
        &node.elem
    })
}
```

```text
cargo build

    Finished dev [unoptimized + debuginfo] target(s) in 0.32s
```

Nailed it.

We can also make a *mutable* version of this method using `as_mut`:

搞定。

这个方法我们同样也可以通过`as_mut`搞一个*mutable*版本的：



```rust ,ignore
pub fn peek_mut(&mut self) -> Option<&mut T> {
    self.head.as_mut().map(|node| {
        &mut node.elem
    })
}
```

```text
lists::cargo build

```

EZ

Don't forget to test it:

小意思
不要忘记测试：

```rust ,ignore
#[test]
fn peek() {
    let mut list = List::new();
    assert_eq!(list.peek(), None);
    assert_eq!(list.peek_mut(), None);
    list.push(1); list.push(2); list.push(3);

    assert_eq!(list.peek(), Some(&3));
    assert_eq!(list.peek_mut(), Some(&mut 3));
}
```

```text
cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 3 tests
test first::test::basics ... ok
test second::test::basics ... ok
test second::test::peek ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured

```

That's nice, but we didn't really test to see if we could mutate that `peek_mut` return value, did we?  If a reference is mutable but nobody mutates it, have we really tested the mutability?  Let's try using `map` on this `Option<&mut T>` to put a profound value in:

不错，但我们并没有测试是否能够改变`peek_mut`的返回值。如果一个引用可变但没人改变它，怎么能够说我们真的测过它的可变性？让我们在`Option<&mut T>`上通过`map`方法来一探究竟：


```rust ,ignore
#[test]
fn peek() {
    let mut list = List::new();
    assert_eq!(list.peek(), None);
    assert_eq!(list.peek_mut(), None);
    list.push(1); list.push(2); list.push(3);

    assert_eq!(list.peek(), Some(&3));
    assert_eq!(list.peek_mut(), Some(&mut 3));
    list.peek_mut().map(|&mut value| {
        value = 42
    });

    assert_eq!(list.peek(), Some(&42));
    assert_eq!(list.pop(), Some(42));
}
```

```text
> cargo test

error[E0384]: cannot assign twice to immutable variable `value`
   --> src/second.rs:100:13
    |
99  |         list.peek_mut().map(|&mut value| {
    |                                   -----
    |                                   |
    |                                   first assignment to `value`
    |                                   help: make this binding mutable: `mut value`
100 |             value = 42
    |             ^^^^^^^^^^ cannot assign twice to immutable variable          ^~~~~
```

The compiler is complaining that `value` is immutable, but we pretty clearly wrote `&mut value`; what gives? It turns out that writing the argument of the closure that way doesn't specify that `value` is a mutable reference. Instead, it creates a pattern that will be matched against the argument to the closure; `|&mut value|` means "the argument is a mutable reference, but just copy the value it points to into `value`, please."  If we just use `|value|`, the type of `value` will be `&mut i32` and we can actually mutate the head:

编译器抱怨道`value`是不可变的，但我们确实写的是`&mut value`；什么鬼？原来是闭包中参数的定义没有特别指出`value`是可变引用。它创建了一个模式来匹配闭包所对应的参数；`|&mut value|`的意思是“该参数是一个可变引用，但请将它指向的值复制到`value`中。”如果改用`|value|`，`value`的类型将是`&mut i32`并且我们可以改变它：



```rust ,ignore
    #[test]
    fn peek() {
        let mut list = List::new();
        assert_eq!(list.peek(), None);
        assert_eq!(list.peek_mut(), None);
        list.push(1); list.push(2); list.push(3);

        assert_eq!(list.peek(), Some(&3));
        assert_eq!(list.peek_mut(), Some(&mut 3));

        list.peek_mut().map(|value| {
            *value = 42
        });

        assert_eq!(list.peek(), Some(&42));
        assert_eq!(list.pop(), Some(42));
    }
```

```text
cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 3 tests
test first::test::basics ... ok
test second::test::basics ... ok
test second::test::peek ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured

```

Much better!
更棒了！
