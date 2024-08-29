+++
title = 'rust中的所有权机制和变量的可变不可变'
date = 2024-07-31
draft = false
toc = false
tags = ['rust', '难点分析']
categories = ['Rust']
+++

所有权是rust中特有的概念，理解这个是使用rust的基础，这个机制加上控制变量的可变不可变来避免了数据竞争和无GC的内存管理。

<!--more-->

在Rust中所有权是排他唯一的，也就是说在一个范围域内（这个范围域值得讨论，不是简简单单的一个代码块内）只有一个变量拥有所有权，所有权的转移发生在赋值操作，所有权失效则这个变量不可以再使用：
```rust
fn main() {
    let s1 = String::from("hello"); // 赋值会给所有权
    let s2 = s1;  // 所有权从 s1 移动到 s2
    
    // 不能再使用 s1，因为它的所有权已经转移了，s1 失效
    // println!("{}", s1); // 这会导致编译错误
    println!("{}", s2); // 可以使用 s2
}
```

有所有权后也不能直接修改变量，在Rust中变量默认是不可修改的：
```rust
fn main() {
    let s1 = String::from("hello");
	s1 = String::from("word"); // error， cannot assign twice to immutable variable s1
	println!("{}", s1)
}
```

即默认所有变量都是immutable。

如果需要修改一个变量，需要使用 `mut` 关键字：
```rust
fn main() {
    let mut s1 = String::from("hello");
	s1 = String::from("word");
	println!("{}", s1)
}
```
\
可以使用引用来获得多个读权限，一写多读不会产生数据竞争问题。使用 `&` 来创建拥有所有权变量的 **引用**，这个创建引用的行为叫做 **借用**。
```rust
fn main() {
    let s1 = String::from("hello");
	let s2 = &s1;
	println!("{}", s2);
	let s3 = &s1;
	println!("{}", s3);
}
```
如上所示，可以创建多个引用来读，当然这个引用也是无法重新赋值的，是**不可变引用**。

如果想通过引用来修改，可以创建一个可变引用，使用 `&mut` 来创建一个可变引用。
```rust
fn main() {
	let mut x = String::from("hello"); // 需要原本拥有所有权的变量是mut的
	let x_mut1 = &mut x;
	// let x_mut2 = &mut x; 不可以同时创建多个可变引用
	// let x_mut2 = &x; cannot borrow `x` as immutable because it is also borrowed as mutable
	
	*x_mut1 = String::from("word"); // * 来解引用，修改x的值为 “word”
	println!("{}", x_mut1); // word
	println!("{}", x); // word
}
```

在rust中同一个作用域内不可以创建多个可变引用，可变引用和不可变引用也不能同时存在。
但从上面的代码中可以看到拥有所有权的变量 x 和 它的可变引用 x_mut1 貌似都可以修改：
```rust
fn main() {
	let mut x = String::from("hello");
	let x_mut1 = &mut x;  

	*x_mut1 = String::from("word");
	x = String::from("word2");
}
```
上面的代码并不会报错，但是如果调换最后2行的顺序或者是最后加入 `println` 都会导致错误。

调换最后2行的顺序：
```rust
fn main() {
	let mut x = String::from("hello");
	let x_mut1 = &mut x;  

	x = String::from("word2");
	*x_mut1 = String::from("word");	
}
// error[E0506]: cannot assign to `x` because it is borrowed
//  --> src/main.rs:5:2
//   |
// 3 |     let x_mut1 = &mut x;  
//   |                  ------ `x` is borrowed here
// 4 |
// 5 |     x = String::from("word2");
//   |     ^ `x` is assigned to here but it was already borrowed
// 6 |     *x_mut1 = String::from("word");    
//   |     ------- borrow later used here
```
错误表明在x被借出期间不可以修改它的值，一开始没报错是因为对x的赋值在最后，此时引用 x_mut1 已经被编译器判断为失效了（实际上也应该失效，因为从 `*x_mut1 = String::from("word");` 后就没用过 `x_mut1` 了），这表明rust编译器对引用的检查并不是简单的检查一个代码块，而是能一定程度判断出引用在具体多少行失效。

加入 `println`：
```rust
fn main() {
	let mut x = String::from("hello");
	let x_mut1 = &mut x;  

	*x_mut1 = String::from("word");
	x = String::from("word2");
	
	println!("{}", x_mut1); 
	println!("{}", x); 
}
// error[E0506]: cannot assign to `x` because it is borrowed
//  --> src/main.rs:6:2
//   |
// 3 |     let x_mut1 = &mut x;  
//   |                  ------ `x` is borrowed here
// ...
// 6 |     x = String::from("word2");
//   |     ^ `x` is assigned to here but it was already borrowed
// 7 |     
// 8 |     println!("{}", x_mut1); 
//   |                    ------ borrow later used here
```
这个错误的原因是一样的，最后面使用到了 `x_mut1` 导致它的有效期持续到了最后，而在 `x_mut1` 有效期间又对 x 进行了修改操作。

总结Rust中的所有权和引用的原则就是：
1. 变量默认不可变
2. 所有权唯一，赋值操作发生所有权转移，作用域结束所有权失效，无法再使用
3. 通过借用可以创建对单一所有权变量的多个引用，可用于多个变量来读
4. 所有权失效，其所有的引用也会失效
5. 通过 mut 关键字来区分可修改/不可修改，即使拥有所有权也需要通过 mut 关键字来声明变量是可以修改的
6. 可变引用可以修改所有权变量的值，但需要2个都用`mut`来声明，借出期间拥有所有权的变量不能对其修改
7. 可变引用在一个作用域范围内最多只能有一个，且不能和一般引用（不可变引用）同时存在，即不允许同时多写和同时读写

一个需要注意的是函数传参时也会发生所有权的转移，因为传参是个赋值过程，通过借用行为可以避免这种所有权丢失。

```rust
fn main() {
    let s1 = String::from("hello");
    let s2 = s1;  // 所有权从 s1 移动到 s2

    // 因为use_string的参数是String，所以这里也会发生所有权转移，之后不能再使用变量s2
    use_string(s2); // 所有权转移到use_string函数

    // println!("{}", s2); // 编译错误，s2的所有权已经转移到了use_string函数中, s2 失效

    let s3 = String::from("hello");
    // 使用引用来避免所有权转移，将创建一个引用的行为称为 借用（borrowing），
    use_string_by_refer(&s3);
    // 可以继续使用s3
    println!("{}", s3);
}

fn use_string(s: String) {
    println!("{}", s);
    // 这里s的所有权会在函数结束时被释放
}

fn use_string_by_refer(s: &str) {
    println!("{}", s);
}
```
\
另一个要理解的是集合类型变量和结构体的所有权，在 Rust 中，集合类型变量和结构体类似，外部只能使用其引用来访问和修改其内部内容，集合变量拥有其内部所有成员的所有权。
```rust
use std::collections::HashMap;

fn main() {
    let mut scores = HashMap::new();

	// 这里 insert 方法的key的参数类型是String
    scores.insert(String::from("Blue"), 10);
    
    let team_name = String::from("Blue");
    // 这里 get 方法的key的参数类型是&str，get 直接返回的也是引用
    let score = scores.get(&team_name).copied().unwrap_or(0);
    println!("Score for {}: {}", team_name, score);

	// key: &String, value: &i32
    for (key, value) in &scores {
        println!("{}: {}", key, value);
    }
}
```
从上面的代码可以看到，当使用hashmap的insert方法时key的参数类型是String，但是get 方法则是引用，这很容易理解，因为 insert 里面会有写操作，就是把外部一对 key, value 写入到hashmap里，而在Rust中集合类变量需要拥有其成员的所有权，所以在 insert 的时候就应该把外部变量的所有权给转移给 hashmap，而 get 方法是按照 key 来匹配查找 value，参数 key 是用来读的，所以是引用就可以了，不需要所有权的转移，这个可以启发在我们写自己的函数时函数参数需不需要设置为引用。

另一个问题是如果一个变量借用一个结构体里的map或者结构体这样的嵌套结构，会存在借用链吗？
```rust
use std::collections::HashMap;

struct SubSubStruct {
    map: HashMap<String, String>,
}

struct SubStruct {
    sub_sub_struct: SubSubStruct,
}

struct Struct {
    sub_struct: SubStruct,
}

impl SubSubStruct {
    fn get_map(&self) -> &HashMap<String, String> {
        &self.map
    }
}

fn main() {
    let mut main_struct = Struct {
        sub_struct: SubStruct {
            sub_sub_struct: SubSubStruct {
                map: HashMap::new(),
            },
        },
    };

    main_struct.sub_struct.sub_sub_struct.map.insert("key".to_string(), "value".to_string());

    // 获取 map 的不可变引用
    let map_ref = main_struct.sub_struct.sub_sub_struct.get_map();
    println!("{:?}", map_ref.get("key"));

    // 由于 map_ref 的存在，以下代码会报错，因为它试图获取 main_struct 的可变引用
    let sub_struct_mut_ref = &mut main_struct.sub_struct;
    sub_struct_mut_ref.sub_sub_struct.map.insert("another_key".to_string(), "another_value".to_string());
	// 最后没有用到 map_ref 的话则没有错误，map_ref会被释放
    println!("{:?}", map_ref.get("another_key"));
}
// error[E0502]: cannot borrow `main_struct.sub_struct` as mutable because it is also borrowed as immutable
//   --> src/main.rs:37:30
//   |
// 33 |     let map_ref = main_struct.sub_struct.sub_sub_struct.get_map();
//   |                   ------------------------------------- immutable borrow occurs here
// ...
// 37 |     let sub_struct_mut_ref = &mut main_struct.sub_struct;
//   |                              ^^^^^^^^^^^^^^^^^^^^^^^^^^^ mutable borrow occurs here
// ...
// 40 |     println!("{:?}", map_ref.get("another_key"));
//   |                      ------- immutable borrow later used here
```
结论是会存在引用链，从最外面的 main_struct 到里面的 hashmap 都会被引用。
要解决借用链问题，可以尽量缩小借用的范围：
```rust
fn main() {
    let mut container = Container::new();
    container.map.insert(String::from("key1"), String::from("value1"));

    {
        let map_ref = container.get_map(); // 短暂的不可变借用
        println!("{:?}", map_ref.get("key1"));
    }

    {
        let map_mut_ref = container.get_mut_map(); // 短暂的可变借用
        map_mut_ref.insert(String::from("key2"), String::from("value2"));
    }
}
```
\
接下来是一个和所有权机制有关的错误的分析  
这个错误真的非常奇怪，App里有2个方法，switch_mode 和 add_secret, 使用其中一个会报错而另一个则不会，这里奇怪的地方在于我觉得这个2个方法在 &mut self 这里是一样的，所以应该会同时报错或者同时不报错。  
`app.rs`
```rust
pub struct App<'a> {
    pub should_exit: bool,
    pub secret_list: SecretList,
    pub panels: HashMap<PanelName, Panel>,
    pub mode: Mode,
    pub guide: &'a str,
    pub error: AppErr<'a>,
}

....

impl<'a> App<'a> {
.....
    pub fn get_panel(&mut self, panel_name: PanelName) -> &mut Panel {
        self.panels.get_mut(&panel_name).unwrap()
    }
.....
    pub fn switch_mode(&mut self, mode: Mode) {
        self.mode = mode;
        match self.mode {
            Mode::Add => self.guide = GUIDE_ADD,
            Mode::Make => self.guide = GUIDE_MAKE,
            Mode::Delete => self.guide = GUIDE_DELETE,
            Mode::Update => self.guide = GUIDE_UPDATE,
            Mode::Normal => self.guide = GUIDE_NORMAL,
            _ => {},
        }
    }
......
    pub fn add_secret(&mut self, name: String, value: String) -> Result<(), &str> {
        if name.is_empty() || value.is_empty() {
            return Err("Name, value and cannot be empty");
        }
        let mut secrets = self.secret_list.secrets.clone();
        if secrets.iter().any(|s| s.name == name) {
            return Err("Secret already exists");
        }

        secrets.push(SecretItem::new(name, value));
        utils::sync_secrets_to_file(secret_items_to_strings(&secrets), &utils::get_secret_file_path());
        self.secret_list.secrets = secrets;
        Ok(())
    }
........
}
```
在handle_keys.rs里使用上面的方法的话，只有add_secret会报 cannot borrow `*app` as mutable more than once at a time, because app is borrowed here `let panel = _app_.get_panel(PanelName::MakeSecret); `这个错误，但是 switch_mode 则不会，我寻思这2个方法不都是 &mut self 么，怎么一个有错误一个还没有呢？  
`handle_keys.rs`
```rust
pub fn handle_key_in_make_mode(app: &mut App, key: KeyEvent) {
	// first mutable borrow app
    let panel = app.get_panel(PanelName::MakeSecret);

    match key.code {
        KeyCode::Esc => app.switch_mode(Mode::Normal),
        KeyCode::Char(ch) => panel.content[panel.index].push(ch),
        KeyCode::Backspace => _ = panel.content[panel.index].pop(),
        KeyCode::Tab => panel.index = (panel.index + 1) % 3,
        KeyCode::Enter => {
            let length = panel.content[1].trim();
            let n = match length.parse::<usize>() {
                Ok(num) => num,
                Err(_) => {
                    app.error = AppErr{msg: "Length must be number", error_timer: Some(Instant::now())};
                    return;
                }
            };

			// name borrow from app->panel
            let name = panel.content[0].trim();
            let advance = panel.content[2].trim();
            let value = utils::generate_random_string(n, advance == "yes" || advance == "y");
            // name is used here, so will checked by rust
            app.add_secret(name.to_string(), value);

			// this line does not have error
            // app.switch_mode(Mode::Normal); no error
        }
        _ => {}
    }
}
```

首先仔细看下错误信息：
```text
error[E0499]: cannot borrow `*app` as mutable more than once at a time            
  --> src/handle_keys.rs:43:13                               
   |                                                                             
23 |     let panel = app.get_panel(PanelName::MakeSecret);                       
   |                 --- first mutable borrow occurs here                       
...                                                                              
43 |             app.add_secret(name.to_string(), value);            
   |             ^^^            ---- first borrow later used here                
   |             |                                 
   |             second mutable borrow occurs here
```

虽然是显示 second mutable borrow 的是app，但是仔细看的话会发现很奇怪 `first borrow later used here` 是 `name.to_string()` 的位置。

想要深入理解这个问题，需要搞清楚rust中变量的生命周期以及引用检查器的工作机制，现在只是简单分析下。  
首先是在23行的 `let panel = app.get_panel(PanelName::MakeSecret);` 这里是第一个对app的可变引用，根据可变引用不能同时存在的原理在第一个分支处 `KeyCode::Esc => app.switch_mode(Mode::Normal),` 就应该报错才对，没有错误的原因是这里 panel 已经exist了，编译器检查代码发现在这个分支里panel没有用到会释放这个引用，这也是除 `KeyCode::Enter` 之外其他 match 分支没有错误的原因。

在43行报错的原因是因为用到了 `name.to_string()` 而 name 是对 first mutable borrow 处panel 的引用（也就是23行第一个对app的引用），而这里的  app.add_secret() 这里又用到了一个app的引用（可以将每次调用app.xxx 方法作为对app的一次引用）从而产生了冲突。
而 `app.switch_mode(Mode::Normal)` 没有错误则是因为没有用到 name，编译器判断23行的第一个app的引用被释放了。

解决方法就是在 `app.add_secret` 之前使用 name.to_string()，复制name得到一个新的拥有所权的变量，让编译器判断到 `app.add_secret` 这行的时候第一个引用已经失效，从而避免引用冲突。
```rust
	let name = name.to_string();
	app.add_secret(name, value);
```
\
最后贴下在reddit上的问题：
[I really got confused why here has cannot borrow `*app` as mutable more than once at a time error : r/rust (reddit.com)](https://www.reddit.com/r/rust/comments/1ea21xg/comment/lelm9rl/?context=3)