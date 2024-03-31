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


