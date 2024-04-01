## 模块

crate 是关于项目间代码共享的，而模块是关于项目内代码组织的。它们扮演着 Rust 命名空间的角色。是构成 Rust 程序或库的函数、类型、常量等的容器。

```rust
mod spores {
  use cells::{ Cell, Gene };

  pub struct Spore {}

  pub fn produce_spore(factory: &mut Sporangium) -> Spore {}

  pub (crate) fn genes(spore: &Spore) -> Vec<Gene> {}

  fn recombine(parent: &mut Cell) {}
}
```

模块是一组语法项的集合，这些语法项具有命名的特性，比如上面的 Spore 结构体和 3 个函数。pub 关键字会使某个语法项声明为公共项，这样它就可以从模块外部访问了。

如果把一个函数标记为 `pub(crate)`，那么就意味着它可以在这个 crate 中的任何地方使用，但不会作为外部接口的一部分公开。他不能被其他 crate 使用，也不会出现在这个 crate 的文档中。

任何未标记为 pub 的内容都是私有的，只能在定义它的模块及其任意子模块中使用。

```rust
let s = spores::produce_spore(&mut factory); // 正确
spores::recombine(&mut cell); // 错误， recombine 是私有的。
```

### 嵌套模块

模块可以嵌套，通常可以看到某个模块仅仅是一组子模块集合：

```rust
mod plant_structures {
  pub mod roots {}
  pub mod stems {}
  pub mod leaves {}
}
```

也可以指定 `pub(super)`，让语法项只对其父模块可见。还可以指定 `pub(in <path>)`，让语法项在特定的父模块及其后代中可见。这对于深度嵌套的模块特别有用：

```rust
mod plant_structures {
    pub mod roots {
        pub mod products {
            pub(in crate::plant_structures::roots) struct Cytokinin {}
        }
        use products::Cytokinin;
    }
    use roots::products::Cytokinin; //error: struct `Cytokinin` is private
}

use plant_structures::roots::products::Cytokinin; //error: struct `Cytokinin` is private
```

### 单独文件中的模块

模块还可以这样写：

```rust
mod spores;
```

前面我们一直把 spores 模块的主体代码包裹在花括号中。在这里，我们告诉 Rust 编译器， spores 模块保存在一个单独的名为 spores.rs 的文件中。

```rust
// spores.rs 文件
pub struct Spore {}
pub fn produce_spore(factory: &mut Sporangium) -> Spore {}
pub (crate) fn genes(spore: &Spore) -> Vec<Gene> {}
fn recombine(parent: &mut Cell) {}
```

spores.rs 文件仅包含构成该模块的那些语法项，它不需要任何样板代码来声明自己是一个模块。

模块可以有自己的目录。当 Rust 看到 `mod spore`时，会同时检查 spores.rs 和 spores/mod.r。如果两个文件都不存在，或者都存在，就会报错。对于上面的例子来说，我们使用了 `spores.rs`，因为 spores 模块没有任何子模块。但是考虑一下我们之前编写的 plant_structures 模块。如果将该模块及其 3 个子模块拆分到它们自己的文件中，则会生成如下的项目：

```
fern_sim/
├── Cargo.toml
└── src/
    ├── main.rs
    ├── spores.rs
    └── plant_structures/
        ├── mod.rs
        ├── leaves.rs
        ├── roots.rs
        └── stems.rs
```

在 main.rs 中，我们声明了 plant_structures 模块

```rust
pub mod plant_structures;
```

这行语句会导致 Rust 加载 plant_structures/mod.rs 该文件声明了 3 个子模块。

```rust
// 在plant_structures/mod.rs中 pub mod roots;
pub mod stems;
pub mod leaves;
```

这 3 个模块的内容存储在 leaves.rs、roots.rs、stems.rs 这 3 个单独的文件中，与 mod.rs 一样位于 plant_structures 目录下。
也可以使用同名的文件和目录来组成模块。如果 stems 需要包含为 xylem 和 phloem 模块，，那么可以选择将 stems 保留在 plant_structures/stems.rs 中并添加一个 stems 目录

```
fern_sim/
├── Cargo.toml
└── src/
    ├── main.rs
    ├── spores.rs
    └── plant_structures/
        ├── mod.rs
        ├── leaves.rs
        ├── roots.rs
        ├── stems/
        │   ├── phloem.rs
        │   └── xylem.rs
        └── stems.rs
```

然后在 stems.rs 中，我们声明了两个新的子模块

```rust
// plant_structrues/stems.rs 文件
pub mod xylem;
pub mod phloem;
```

总结模块的目录结构可以有以下三种方式。

- 模块位于自己的文件中
- 模块位于自己的带有 mod.rs 的目录中
- 模块在自己的文件中，并带有包含子模块的补充目录

### 路径和导入

`::` 运算符用于访问模块中的各项特性。项目中任何位置的代码都可以通过写出其路径来引用标准库特性

```rust
if s1 > s2 {
  std::mem::swap(&mut s1, &mut s2);
}
```

`std`是标准库的名称。路径 std 指的是标准库的顶层模块。std::mem 是标准库中的子模块，而 std::mem::swap 是该模块中的公共函数。

可以用这种方式编写所有代码：如果你想要一个园或字典，就明确写出 `std::f64::consts::PI` 或 `std::collections::HashMap::new`。但这样做很繁琐。另一种方法是将这些特性导入使用它们的模块中：

```rust
use std::mem;
if s1 > s2 {
  mem::swap(&mut s1, &mut s2);
}
```

`use std::mem`表示名称 `mem`在整个封闭快或模块中成了 `std::mem` 的本地别名。

可以通过 use std::mem::swap 来导入 swap 函数本身，而不是 mem 模块。然而，我们之前的编写风格通常被认为是最好的：**导入类型、特型和模块**，然后使用相对路径访问其中的函数、常量和其他成员。

```rust
use std::collections::{HashMap, HashSet}; // 同时导入两个模块
use std::fs::{self, File}; // 同时导入 std::fs 和 std::fs::File。 self表示 fs 本身。
use std::io::prelude::*; // 导入所有语法项。
```

可以使用 as 导入一个语法项，但在本地赋予它一个不同的名称

```rust
use std::io::Result as IOResult;

// 这个返回类型只是 std::io::Result<()> 的另一种写法
fn save_spore(spore: &Spore) -> IOResult<()>
```

- 默认情况下，路径是相对于当前模块的。
- self 也是当前模块的同义词。
- super 表示父模块
- crate 指的是当前 crate

### 标准库的预导入

标准库 std 会自动链接到每个项目。这意味着我们可以使用可以 使用 `use std::whatever`。另一方面，还有一些特别的便捷名称入`Vec`和`Result`会包含在标准库预导入中并自动导入。Rust 的行为就好像每个模块（包括根模块）有用以下导入语句开头一样。

```rust
use std::prelude::v1::*;
```

### 公开 use 声明

虽然 use 声明只是个别名，但也可以公开它们

```rust
// 在 plant_structures/mod.rs
pub use self::leaves::Leaf;
pub use self::roots::Root;
```

这意味着 Leaf 和 Root 是 plant_structures 模块的公共语法项。它们还是 plant_structures::leaves::Leaf 和 plant_structures::roots::Root 的简单别名。

### 公开结构体字段

结构体的字段，甚至是私有字段，都可以在声明该结构体的整个模块及其子模块中访问。在模块之外，只能访问公共字段。

### 静态变量与常量

除了函数、类型和嵌套模块，模块还可以定义常量和静态变量。常量有点儿像 C++ 的 #define: 该值在每个使用了它的地方都会编译到你的代码中。静态变量是在程序开始运行之前设置并持续到程序退出的变量。

```rust
// 定义常量
pub const ROOM_TEMPERATURE: f64 = 20.0;

// 定义静态变量
pub static ROOM_TEMPERATURE: f64 = 68.0;
```

## 属性

Rust 程序中的任何语法都可以用属性进行装饰。属性是 Rust 的通用语法，用于向编译器提供各种指令和建议。

```rust
// 使用#[allow]允许某种警告。
#[allow(non_camel_case_types)]
pub struct git_revspec {}
```

条件编译是使用名为 #[cfg] 的属性编写的另一项特性

```rust
#[cfg(target_os = "android")]
mod mobile;
```

在对函数进行优化时，可以使用 `#[inline]`将函数内联。建议只用在函数特别小和简单的场景使用，能减少函数调用的开销。

要将属性附着到整个 crate 上，请将其添加到 `main.rs`和`lib.rs`文件到顶部，放在任何语法项之前，并写成`#!`, 而不是 `#`。

```rust
// libgit2_sys/lib.rs

#![allow(non_camel_case_types)] // 会对整个 crate 生效。

pub struct git_revspec {
  // ...
}
pub struct git_error {
  // ...
}
```

`#![feature]` 属性用于启用 Rust 语言和库的不稳定性，这些特性是实验性的。因此可能有 bug 或者未来可能会被更改或移出。Rust 团队有时会将实验性特性下来，使其成为语言标准的一部分。那时这个 `#![feature]` 属性就会变得多余，因此 Rust 会生成一个警告，建议你将其移除。
