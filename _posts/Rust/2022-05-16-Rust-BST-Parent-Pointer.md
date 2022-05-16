---
layout: post
title: "부모 포인터가 있는 BST의 Rust 구현 1편 - 스마트 포인터로는 안 된다"
date: 2022-05-16 17:58:15 +0900
categories: Rust
use_math: true
---

Rust는 독특하고 엄격한 메모리 관리 구조를 갖고 있습니다. 덕분에 가비지 콜렉터(GC) 없이도 누수되는 메모리가 없게끔 메모리를 정확하게 통제하는 프로그램을 만들 수 있습니다.

하지만, 이러한 엄격함 때문에 간혹 다른 언어에서는 구현하기 쉬웠던 것이 Rust에서는 매우 어려운 경우가 있습니다. 대표적으로 이진 탐색 트리와 같이, 노드를 동적 할당한 후 트리 구조를 만든 형태의 자료구조가 그러합니다. 특히, 트리의 노드가 부모를 향한 포인터(parent pointer)터를 갖고 있을 경우 유독 Rust의 규칙 아래에서 구현하기 매우 어렵습니다.

이러한 트리를 구현하는 방법에 대한 자료는 인터넷에도 거의 없습니다. 구글에 검색해보면 대부분의 경우 부모 포인터가 아예 없게 구현해놨거나, 질문글인데 답변을 다는 사람들도 다른 방법으로 접근해보라고 권하는 수준입니다. 이만큼이나 Rust에선 트리 자료구조가 구현하기 어렵습니다.

이 포스팅에서는 Rust를 사용해서 부모 포인터가 있는 이진 탐색 트리를 구현하는 과정을 설명합니다. 구체적인 구현에 대한 설명에 앞서 Rust에선 이게 왜 이리 어려운지, 동적 할당을 하지 않고 구현하는 다른 선택지는 없는지 먼저 논의한 후, 구현을 시작해보겠습니다. 구현하는 과정에서 unsafe Rust의 기초적인 내용을 알아보고, 이를 활용해서 이진 탐색 트리 구현을 완성할 겁니다.

이 글은 독자가 Rust의 기초를 어느정도 알고 있고, 이진 탐색 트리가 무엇인지 안다는 전제 하에 작성되었습니다. 여기서 Rust의 기초는 [Rust 기본서](https://rinthel.github.io/rust-lang-book-ko/foreword.html)의 내용 중 10장까지와, 15장 스마트 포인터에 대한 내용까지를 포함합니다. 19-1장 안전하지 않은 Rust도 기본적인 내용은 알고 있는 것이 좋습니다.

이진 탐색 트리가 무엇인지 잘 모르는 분들은 구글링해보면 자료가 쏟아지므로 그쪽을 참고해주시기 바랍니다. 또한, 앞으로 이진 탐색 트리는 BST(binary search tree)라고 줄여 부르겠습니다.

## 부모 포인터를 추가하려는 이유

사실 BST는 사실 부모 포인터가 없어도 구현이 가능합니다. BST를 확장한 balanced BST 역시 부모 포인터 없이 구현이 가능합니다. 예를 들어 splay tree의 경우 top-down splaying을 활용하면 자식 포인터만 가진 splay tree를 구현할 수 있습니다.

그럼에도 불구하고 부모 포인터를 추가하려는 이유는 유연성 때문입니다. 부모 포인터 없이도 하고 싶은 걸 대부분 다 할 수는 있지만, 구현하기 상당히 어렵고 자료구조를 조금만 확장하려 해도 Rust의 엄격한 규칙에 걸려서 수많은 에러가 발생합니다. 부모 포인터만 있었으면 간단하게 해결할 문제를, 자식 포인터만으로 해결하려 하다 보면 구현하기 상당히 어려워지기 때문에 부모 포인터를 추가하려는 것입니다.

부모 포인터 없이도 할 건 다 할 수 있습니다. 사실 특별한 이유 없이 해보려는 것도 큽니다. Rust에선 특히나 난이도가 높아서 시도해보는 맛이 있기도 합니다.

# 스마트 포인터로는 왜 어려울까?

Rust에서 데이터를 힙에 동적 할당하는 방법으로 [Rust 기본서](https://rinthel.github.io/rust-lang-book-ko/ch15-00-smart-pointers.html)에서 소개된 방법은 `Box`, `Rc`, `RefCell` 세 종류가 있습니다. 일단 이 세 가지를 통해 BST를 구현하는 걸 시도해봅시다. 구현 실패를 건너뛰고 싶으면 이 문단의 결론 파트로 바로 넘어가도 괜찮습니다.

한 가지 스포일러하자면, 세 가지 모두 실패할 겁니다. 각각이 실패하는 이유를 한번 하나씩 살펴봅시다.

## Box: 가장 기본적인 동적 할당

`Box`는 힙에 데이터를 동적 할당하는 가장 기초적인 방법으로, 사용하기 편리하고 mutability에 대한 제약도 다른 변수와 다르지 않습니다. 하지만, `Box`의 값을 "소유"하는 것은 한번에 하나의 변수만이 할 수 있으며, 이때문에 BST를 구현하는 데에 애로사항이 생깁니다. 직접 시도해보도록 하죠.

우선 자식 포인터만 있는 트리를 봅시다. 사실 이 경우엔 `Box`를 사용해서 문제 없이 구현할 수 있습니다.

```rust
struct Node<K, V> {
    key: K,
    value: V,
    left: Link<K, V>,
    right: Link<K, V>,
}

type Link<K, V> = Option<Box<Node<K, V>>>;

pub struct BST<K, V> {
    root: Link<K, V>,
}
```

자식 포인터만 있는 경우는 이렇게 하면 끝납니다. 매우 간단합니다. 하지만 여기에 부모 포인터를 추가하는 순간, 모든 것이 꼬입니다. 어느 면에서 꼬이는지 직접 한번 코드를 써봅시다.

시험삼아서 단순히 루트의 왼쪽에 새로운 노드를 붙이는 함수를 만들어봅시다. 단순한 상황을 가정하기 위해 루트의 왼쪽 자식은 언제나 없다고 가정합시다.

```rust
struct Node<K, V> {
    key: K,
    value: V,
    parent: Link<K, V>,
    left: Link<K, V>,
    right: Link<K, V>,
}

type Link<K, V> = Option<Box<Node<K, V>>>;

pub struct BST<K, V> {
    root: Link<K, V>,
}

impl<K, V> Node<K, V> {
    fn new(key: K, value: V) -> Box<Self> {
        Box::new(Self {
            key,
            value,
            parent: None,
            left: None,
            right: None,
        })
    }
}
```

우선 `Node<K, V>`에 `parent: Link<K, V>` 부모 포인터를 추가해줍니다. 그리고, 코드를 간결하게 하기 위해 `Node<K, V>`를 할당해주는 `Node::new(key: K, value: V)` 함수를 정의합니다.

그럼 이제 `BST::insert_left(&mut self, key: K, value: V)`를 구현해줍시다. 이 함수는 BST 루트의 왼쪽 자식을 주어진 `key, value`의 노드로 교체하는 함수입니다. 여기선 중요하지 않지만 루트가 없다면 그 노드를 새 루트로 교체하기까지 해봅시다.

```rust
impl<K, V> BST<K, V> {
    fn insert_left(&mut self, key: K, value: V) {
        let mut new_left = Node::new(key, value);
        match self.root {
            None => {
                self.root = Some(new_left);
            }
            Some(ref mut root) => {
                new_left.parent = Some(*root);
                root.left = Some(new_left);
            }
        }
    }
}
```

크게 어려울 거 없는 코드입니다. `new_left`는 키와 값을 `key`, `value`로 갖고 있는 새로운 노드입니다. BST가 루트가 있다면 `Some(ref mut root)`와 매칭되어, `new_left`의 부모에 `root`를 넘겨주고 `root`의 왼쪽 자식에 `new_left`를 넘겨줍니다. 한번 이대로 놓고 컴파일해봅시다.

```
error[E0507]: cannot move out of `*root` which is behind a mutable reference
  --> src/main.rs:37:40
   |
37 |                 new_left.parent = Some(*root);
   |                                        ^^^^^ move occurs because `*root` has type `Box<Node<K, V>>`, which does not implement the `Copy` trait

For more information about this error, try `rustc --explain E0507`.
error: could not compile `rust` due to previous error
```

컴파일 에러가 발생했습니다. "move occurs"라고 말하는 걸 보니 소유권과 관련된 에러인 것 같습니다. 해석해보자면, `*root`를 가리키는 가변 참조(mutable reference)가 이미 존재하기 때문에, 거기서 값을 꺼내서 `Some` 안으로 "이동"시킨 후 `new_left.parent`에게 넘겨줄 수 없다는 내용입니다.

일리있는 것이, `*root`의 실제 데이터는 `self`가 "소유"하고 있는 값입니다. 이를 `new_left.parent`로 넘겨주게 되면, `new_left` 역시 같은 데이터를 "소유"하게 될 겁니다. Rust에서는 하나의 데이터에 단 하나의 소유자만 있어야 하므로, `new_left.parent = Some(*root)`를 한 순간 `new_left`는 `self.root`를 소유하게 됩니다. 하지만 `self.root`는 `self`에 의해 소유되어 있으므로, 이 값은 이동할 수 없습니다.

이 구현에선 `Node<K, V>`가 `parent`를 "소유"하는 형태로 이루어져있기 때문에 이러한 문제가 발생합니다. 구체적으로, 어떤 노드는 그 자식에 의해 `parent`로써 동시에 소유되고, 그 부모에 의해 `left` 또는 `right`로써 소유됩니다. 그 노드가 루트라면 BST의 루트로써 BST가 소유합니다. 즉, 하나의 노드를 여러 노드가 소유하려고 하다보니, 이러한 문제가 발생합니다.

그러면 각 노드가 `parent`를 소유하는 것이 아니라, 참조만 하게 하는 건 어려울까요? 즉, `parent`의 타입을 `&mut Option<Node<K, V>>`로 할 수는 없는 걸까요? 하지만 실제로 이를 시도해보면, 소유권과 라이프타임과 관련된 에러가 수도 없이 튀어나옵니다. 글이 길어지기 때문에 간단히 넘어가지만, 실제로 시도해보면 참조와 소유의 관계가 이리저리 뒤섞여서 에러가 사라지지 않습니다.

`Box`는 한 번에 하나의 변수만이 소유할 수 있으므로 무리일 것 같군요. `Rc`는 어떨까요?

## Rc: 여러 변수가 동시에 참조할 수 있는 동적 할당

`Rc` 역시 `Box`와 마찬가지로 변수를 동적 할당해줍니다. 하지만, `Box`와 다르게 `Rc`는 여러 변수가 동시에 소유할 수 있습니다. 내부적으로 이 소유는 참조로 구현되지만, 마치 여러 변수가 하나의 `Rc`를 소유하는 것처럼 만들 수 있게 해줍니다. 이를 바탕으로 앞서 시도했던 것을 다시 해봅시다.

```rust
use std::rc::Rc;

struct Node<K, V> {
    key: K,
    value: V,
    parent: Link<K, V>,
    left: Link<K, V>,
    right: Link<K, V>,
}

type Link<K, V> = Option<Rc<Node<K, V>>>;

pub struct BST<K, V> {
    root: Link<K, V>,
}

impl<K, V> Node<K, V> {
    fn new(key: K, value: V) -> Rc<Self> {
        Rc::new(Self {
            key,
            value,
            parent: None,
            left: None,
            right: None,
        })
    }
}
```

`Rc`의 기본적인 사용법은 `Box`와 유사합니다. `use`를 통해 `std::rc`에서 `Rc`를 가져와야 한다는 점만 빼면 거의 동일하므로, 기존 코드의 `Box`를 `Rc`로 교체하는 것만으로도 `Rc`로 변환이 가능합니다. 그럼 `insert_left`도 구현해봅시다.

한 가지 차이가 있다면, 하나의 `Rc`값을 가리키는 또다른 포인터를 만들기 위해선 `Rc::clone()`을 활용하면 됩니다. 예를 들어, `Rc<i32>`형의 변수 `x`가 있을 때, 변수 `y`가 이를 가리키게 하고 싶으면 `let y = x.clone()`이라고 하면 됩니다.

```rust
impl<K, V> BST<K, V> {
    fn insert_left(&mut self, key: K, value: V) {
        let mut new_left = Node::new(key, value);
        match self.root {
            None => {
                self.root = Some(new_left);
            }
            Some(ref mut root) => {
                new_left.parent = Some(root.clone());
                root.left = Some(new_left);
            }
        }
    }
}
```

`Box`를 사용한 예제와 다른 점은 `new_left.parent`에 대입하는 값이 `Some(*root)`에서 `Some(root.clone())`으로 바뀌었다는 점입니다. `root.clone()`은 `root`를 가리키는 또다른 포인터를 만들어줌으로써 `root`의 값 자체가 `new_left.parent`로 이동하는 것을 막아줍니다.

이제 컴파일을 돌려보면...

```
error[E0594]: cannot assign to data in an `Rc`
  --> src/main.rs:39:17
   |
39 |                 new_left.parent = Some(root.clone());
   |                 ^^^^^^^^^^^^^^^ cannot assign
   |
   = help: trait `DerefMut` is required to modify through a dereference, but it is not implemented for `Rc<Node<K, V>>`

error[E0594]: cannot assign to data in an `Rc`
  --> src/main.rs:40:17
   |
40 |                 root.left = Some(new_left);
   |                 ^^^^^^^^^ cannot assign
   |
   = help: trait `DerefMut` is required to modify through a dereference, but it is not implemented for `Rc<Node<K, V>>`

For more information about this error, try `rustc --explain E0594`.
```

또 에러가 발생해버렸네요. 이번엔 에러 내용을 해석해보면 "`DerefMut`가 `Rc`에 대해 구현되지 않았다"가 되네요.

`DeferMut`는 값을 바꿀 수 있는 상태로 역참조를 할 수 있다는 의미입니다. 예를 들어 `x`라는 값의 타입에 `DerefMut`가 구현되어 있으면, `*x = 3`과 같이 `x`의 값을 역참조로 바꿀 수 있음을 의미합니다.

이는 정확히 저희 코드에서 한 행동입니다. `new_left.parent = Some(root.clone())`을 통해 `Rc`타입인 `new_left`의 값을 역참조해서 변경하려고 했으니까요. 하지만 `Rc`는 `DerefMut`가 구현되어 있지 않습니다. 즉, `Rc`타입은 값을 바꿀 수 없습니다!

`Rc`는 여러 변수가 동시에 참조할 수 있는 값입니다. 만약 여러 변수가 `Rc`를 동시에 참조하고 있는데, 하나의 변수가 그 값을 바꿔버리면 나머지 변수들은 그 값이 변했다는 걸 모르는 채로 있을 수 있습니다. Rust에서 하나의 변수에 대한 가변 참조가 하나라도 있으면 그에 대한 불변 참조가 하나도 허용되지 않는 것과 원리가 같죠. 때문에 `Rc`는 값 변경이 불가능합니다.

BST는 그 특성상 노드의 자식, 부모 포인터를 자주 바꿔줘야 하는데, 이러면 `Rc`는 쓰기 어렵습니다. `RefCell`을 써보면 어떨까요?

## RefCell: 불변 참조에 대한 값 변경 가능

어떤 변수가 `RefCell`이면, 그에 대한 가변 참조는 그 변수의 값을 바꿀 수 있습니다.

```rust
let x: i32 = 3;
let y = &x;
*y = 5;
```

위 코드는 에러를 일으킵니다. `y`가 불변 참조이기 때문에 이를 역참조해서 값을 바꾸는 것이 불가능한 건 당연합니다. 'x'가 처음부터 불변으로 선언되었기 때문에 이 값이 바뀌면 안 될 것이므로, `y`를 애초부터 가변 참조로 둘 수도 없습니다. 하지만 `RefCell`을 활용하면 불변인 값을 바꿀 수 있습니다.

```rust
use std::cell::RefCell;

fn main() {
    let x: RefCell<i32> = RefCell::new(3);
    println!("{}", x.borrow());

    {
        let mut y = x.borrow_mut();
        *y = 2;
    }

    println!("{}", x.borrow());
}
```

위 코드에서 처음에 `x`는 `RefCell`로 감싸진 `3`을 들고 있습니다. 때문에 첫 번째 출력되는 줄은 `3`입니다. 여기서 `x`는 불변인 변수입니다.

한편 `y`는 `x`에 대한 가변 참조를 하고 있습니다. `borrow_mut()` 메서드는 `RefCell`에 대한 가변 참조를 만들며, 위 코드에선 그를 통해 `x`의 값을 2로 바꿔버렸습니다. 그리고 마지막 줄에서 `2`가 출력됩니다.

`x`가 불변인데도 `x`의 값이 `3`에서 `2`로 변경되었습니다. 이와 같이 불변 변수에서부터 가변 참조를 가져와 값을 변경하는 것을 "내부 가변성"(inherent mutability)이라고 부릅니다.

BST에서는 각 노드가 여러 번 참조되어야 하므로 다 불변 참조여야 하는데, 그 값을 바꾸기는 해야 하므로 `Rc<RefCell<Node<K, V>>>`가 적절한 타입으로 보입니다.

하지만... 이를 활용해서 직접 구현해보면 `RefCell`의 단점이 나타납니다. 첫째로, `RefCell`에서 `borrow()`나 `borrow_mut()`를 통해 참조를 빌려오면 그 자료형이 `Ref`와 `RefMut`가 됩니다. 이때문에 단순 참조 자료형과는 다른 방식으로 처리해줘야 해서 이 과정에서 코드가 쓸데없이 복잡해집니다.

또한, `RefCell`은 자기 자신의 빌림 규칙을 런타임에 확인합니다. 이때문에 `Rc<RefCell<T>>`를 사용하는 건 사실상 가비지 컬렉터를 사용하는 것과 다름이 없습니다. 런타임에 자신의 참조를 계속 체크하고, 규칙이 위반되지 않았는지 확인해야 하니까요. 이때문에 코드의 퍼포먼스도 느려집니다.

워낙에 구현이 복잡하고, 다다르게 되는 막다른 길이 복잡해서, `RefCell`에 대한 구현 예시는 생략하겠습니다.

## 결론

부모 포인터가 있는 트리는 그 특성상 어떤 노드가 다른 노드를 소유한다는 개념이 애매해집니다. 자식 포인터만 있는 경우 부모가 자식을 소유한다고 하면 되지만, 부모 포인터가 있게 되면 그런 개념이 애매해집니다. 부모 노드를 참조만 하려 해도 라이프타임이 꼬여버립니다.

또한, 각각의 노드를 하나의 변수만이 소유하지 않게 됩니다. 따라서 `Box`는 사용하기 어려우며, 여러 변수가 동시에 하나의 데이터를 참조할 수 있는 `Rc`를 고려해야 합니다.

하지만 `Rc`는 역참조 후 값 변경이 불가능합니다. 즉, 불변 참조자로만 역참조가 가능합니다. 그렇기 때문에 불변 참조자이면서도 값을 바꿀 수 있는 `RefCell`을 활용할 생각을 할 수 있습니다.

다만 `RefCell`은 런타임에 빌림 규칙을 확인하려 하기 때문에 속도가 느리고, `borrow()`와 `borrow_mut()`가 참조자 그 자체를 반환하지 않기 때문에 코드가 매우 장황해집니다.

Safe Rust에서 제공해주는 것으로는 이정도가 한계입니다. 동적 할당이 들어가는 부모 포인터 BST를 제대로 구현하고 싶으면 결국 unsafe Rust를 건드려야 합니다. unsafe Rust를 사용해서 구현하는 과정은 다음 포스트에서 계속하도록 하겠습니다.
