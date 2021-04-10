# Making it all Generic
# 全部使用泛型


We've already touched a bit on generics with Option and Box. However so
far we've managed to avoid declaring any new type that is actually generic
over arbitrary elements.

It turns out that's actually really easy. Let's make all of our types generic
right now:

我们已经在Option与Box的使用中接触到了一点泛型。但是【】

这其实蛮简单的，立刻把我们所有的使用的类型改为泛型吧。



```rust ,ignore
pub struct List<T> {
    head: Link<T>,
}

type Link<T> = Option<Box<Node<T>>>;

struct Node<T> {
    elem: T,
    next: Link<T>,
}
```

You just make everything a little more pointy, and suddenly your code is
generic. Of course, we can't *just* do this, or else the compiler's going
to be Super Mad.

仅仅加上了一对尖括号和对应的T，就使代码变得通用了。当然，我们要做的*不止于此*，不然编译器要骂人了。


```text
> cargo test

error[E0107]: wrong number of type arguments: expected 1, found 0
  --> src/second.rs:14:6
   |
14 | impl List {
   |      ^^^^ expected 1 type argument

error[E0107]: wrong number of type arguments: expected 1, found 0
  --> src/second.rs:36:15
   |
36 | impl Drop for List {
   |               ^^^^ expected 1 type argument

```

The problem is pretty clear: we're talking about this `List` thing but that's not
real anymore. Like Option and Box, we now always have to talk about
`List<Something>`.

But what's the Something we use in all these impls? Just like List, we want our
implementations to work with *all* the T's. So, just like List, let's make our
`impl`s pointy:

问题相当明确了：我在对`List`这个对象实现一些方法，但它已经改变了。像Option和Box那样，我需要对`List<Something>`实现相关的方法。

那我们要使用的“Something”是什么呢？就像List那样，我们希望所实现的方法对*所有*的T都能正常工作。因此让我们也给`impl`加上尖括号：


```rust ,ignore
impl<T> List<T> {
    pub fn new() -> Self {
        List { head: None }
    }

    pub fn push(&mut self, elem: T) {
        let new_node = Box::new(Node {
            elem: elem,
            next: self.head.take(),
        });

        self.head = Some(new_node);
    }

    pub fn pop(&mut self) -> Option<T> {
        self.head.take().map(|node| {
            self.head = node.next;
            node.elem
        })
    }
}

impl<T> Drop for List<T> {
    fn drop(&mut self) {
        let mut cur_link = self.head.take();
        while let Some(mut boxed_node) = cur_link {
            cur_link = boxed_node.next.take();
        }
    }
}
```

...and that's it!
对，就是这样！


```
> cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 2 tests
test first::test::basics ... ok
test second::test::basics ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured

```

All of our code is now completely generic over arbitrary values of T. Dang,
Rust is *easy*. I'd like to make a particular shout-out to `new` which didn't
even change:
我们所有的代码现在无论T是什么类型的情况下都通用了。当 当 当，Rust*容易*吧。我要特别的说一下`new`方法，它没有任何的修改：

```rust ,ignore
pub fn new() -> Self {
    List { head: None }
}
```

Bask in the Glory that is Self, guardian of refactoring and copy-pasta coding.
Also of interest, we don't write `List<T>` when we construct an instance of
list. That part's inferred for us based on the fact that we're returning it
from a function that expects a `List<T>`.

Alright, let's move on to totally new *behaviour*!

沉醉在其中吧，重构与CV（复制粘贴）代码的工程师。同样有趣的是，我们不需要在创建list实例对象时写`List<T>`。这个T是根据我们在实际定义时推断出来的。

行了，让我们继续向下走吧！
