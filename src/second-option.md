# 使用Option

特别细心的读者们也许已经注意到我们实现了一个差劲版本的Option:

```rust ,ignore
enum Link {
    Empty,
    More(Box<Node>),
}
```

Link与`Option<Box<Node>>`等价。这样，我们就不用到处写`Option<Box<Node>>`了，并且不像`pop`方法，我们不用将它暴露给外界，这样也许是不错的。然而Option有一些我们之前自己实现的一些*Really nice*的方法。我们不再自己实现而是改用Option吧。首先，我们只需要简单的用Some与None来做个替换就可以了：


```rust ,ignore
use std::mem;

pub struct List {
    head: Link,
}

// yay type aliases!
type Link = Option<Box<Node>>;

struct Node {
    elem: i32,
    next: Link,
}

impl List {
    pub fn new() -> Self {
        List { head: None }
    }

    pub fn push(&mut self, elem: i32) {
        let new_node = Box::new(Node {
            elem: elem,
            next: mem::replace(&mut self.head, None),
        });

        self.head = Some(new_node);
    }

    pub fn pop(&mut self) -> Option<i32> {
        match mem::replace(&mut self.head, None) {
            None => None,
            Some(node) => {
                self.head = node.next;
                Some(node.elem)
            }
        }
    }
}

impl Drop for List {
    fn drop(&mut self) {
        let mut cur_link = mem::replace(&mut self.head, None);
        while let Some(mut boxed_node) = cur_link {
            cur_link = mem::replace(&mut boxed_node.next, None);
        }
    }
}
```

这样稍微好了一些，但是最大的胜利还是来源于Option的相关方法。

首先，`mem::replace(&mut option, None)`这个常用操作的在Option中也继续被使用并且成为了一个方法：`take`。


```rust ,ignore
pub struct List {
    head: Link,
}

type Link = Option<Box<Node>>;

struct Node {
    elem: i32,
    next: Link,
}

impl List {
    pub fn new() -> Self {
        List { head: None }
    }

    pub fn push(&mut self, elem: i32) {
        let new_node = Box::new(Node {
            elem: elem,
            next: self.head.take(),
        });

        self.head = Some(new_node);
    }

    pub fn pop(&mut self) -> Option<i32> {
        match self.head.take() {
            None => None,
            Some(node) => {
                self.head = node.next;
                Some(node.elem)
            }
        }
    }
}

impl Drop for List {
    fn drop(&mut self) {
        let mut cur_link = self.head.take();
        while let Some(mut boxed_node) = cur_link {
            cur_link = boxed_node.next.take();
        }
    }
}
```

第二点，`match option { None => None, Some(x) => Some(y) }`这个常用的用法被称作`map`，`map`接受`Some(x)`中的`x`来产生`Some(y)`中的`y`。我们将写一个合适的`fn`来将它传给`pop`方法，并且使用内联的方式来重写。

实现这一功能的方式叫做*闭包(closure)*，闭包是一个具有超级权限的匿名函数：它可以访问闭包之外的局部变量！这一特性使得它在执行各种条件逻辑中非常有用。`match`操作只需要在`pop`中做就可以了，那就让我们来重写这个`pop`方法吧：

```rust ,ignore
pub fn pop(&mut self) -> Option<i32> {
    self.head.take().map(|node| {
        self.head = node.next;
        node.elem
    })
}
```

嗯，这下好多了。来跑一遍测试以确认我们没有破坏其他的功能：


```text
> cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 2 tests
test first::test::basics ... ok
test second::test::basics ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured

```

成了！继续对代码的一些*行为*做改造吧。
