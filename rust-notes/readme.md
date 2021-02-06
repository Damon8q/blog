# Rust学习笔记

## 语法部分

### 变量与可变性
* 声明变量使用let关键字
* 默认情况下，变量是不可变的(Immutable)
* 声明变量时，在变量前面加上mut，就可以使变量可变

### 变量与常量
* 常量(constant)，常量在绑定值以后也是不可变的，但是与不可变变量有很多区别：
  - 不可以使用mut，常量永远都是不可变的
  - 声明常量使用const关键字，并且类型必须标注清楚
  - 常量可以在任意作用域内声明，包括全局作用域
  - 常量只可以绑定到常量表达式，无法绑定到函数的调用结果或只能在运行时才能计算出的值
* 在程序运行期间，常量在其声明的作用域内一直有效
* 命名规范：Rust里常量使用全大写字母，每个单词之间用下划线分开，例如：
  - MAX_POINTS
  - 例子：const MAX_POINTS:u32 = 100_OOO;

### 变量隐藏（shadow）
* 可以使用相同的名字声明新的变量，新的变量会shadow（隐藏）之前的同名变量
* shadow和把变量标记为mut是不一样的：
  - 如果不使用let关键字，重新给非mut的变量赋值会导致编译时错误
  - 而使用let声明的同名新变量，也是不可变的    
  - 使用let声明的同名新变量，它的类型可以与之前不同

示例：
```rust
fn main() {
    let mut guess = String::new();
    // do sth...

    // shadow
    let guess:u32 = guess.trim().parse().expect("Please type a number!"); // 编译器可以根据声明的u32类型，自动推断需要将字符串解析为u32类型
    // do sth...
}
```
这一特性在需要类型转换的场景特别有用，可以复用同一个变量名，这样就不用定义另外一个变量名，比如guess_int之类。

### 数据类型
* 标量和复合类型
* Rust是静态编译语言，在编译时必须知道所有变量的类型
  - 基于使用的值，编译器通常能够推断出它的具体类型
  - 但是如果可能的类型比较多（例如上面把String转为整数的parse方法），就必须添加类型的标注，否则编译会报错

#### 标量类型
* 一个标量类型代表一个单个的值
* Rust有四个主要的标量类型：
  - 整数类型（i8, i16, i32, i64, i128, isize, u8, u16, u32, u64, u128, usize）
  - 浮点类型（f32, f64）
  - 布尔类型（bool: true, false）
  - 字符类型（char, 字面值使用单引号）

isize 和 usize 类型：
* 其位数由程序运行的计算机的架构所决定，如64位计算机，那就是64位的
* 使用isize或usize的主要场景是对某种集合进行索引操作

#### 整数字面值
* 除了byte类型外，所有的数值字面值都允许使用类型后缀
  - 例如 57u8
* 如果不确定使用哪种类型，可以使用Rust相应的默认类型
* 整数的默认类型是i32
  - 总体上来说速度很快，即使在64位系统中

| Number literals | Example  |
|  ----  | ----  |
| Decimal      | 98_222 |
| Hex          | 0xff |
| Octal        | 0o77 |
| Binary       | 0b1111_0000 |
| Byte(u8 only)| b'A' |

#### 整数溢出
* 例如：u8的范围是0-255, 如果把一个u8变量的值设置为256,那么：
  - 调试模式下编译：Rust会检查整数溢出，如果发生溢出，程序在运行时就会panic
  - 发布模式下(--release)编译：Rust不会检查可能导致panic的整数溢出
    * 如果溢出发生：Rust会执行“环绕”操作：256变为0, 257变为1...
    * 但程序不会panic

#### 浮点类型
* Rust有两种基础的浮点类型，也就是包含小数部分的类型
  - f32, 32位  单精度
  - f64, 64位  双精度
* Rust的浮点类型使用了IEEE-754标准来表述
* f64是默认的类型，因为在现代CPU上f64和f32的速度差不多，而且精度更高

#### 字符类型
* Rust中char类型被用来描述语言中最基础的单个字符
* 字符类型的字面值使用单引号
* 占用4字节大小
* 是Unicode标量值，可以表示比ASCII多得多的字符内容：拼音、中日韩文、零长度空白字符、emoji表情等
  - U+0000 到 U+D7FF
  - U+E000 到 U+10FFFF

示例：
```rust
fn main() {
    let x = 'z';
    let y: char = '∑';
    let z = '李️';
}
```    

#### 复合类型
* 复合类型可以将多个值放在一个类型里
* Rust提供了两种基础的复合类型：元组(Tuple)、数组

#### Tuple
* Tuple可以将多个类型的多个值放在一个类型里
* Tuple的长度是固定的，一旦声明就无法改变

示例：
```rust
fn main() {
    // 创建Tuple
    // 在小括号里，将值用逗号分开
    // Tuple中的每个位置都对应一个类型，Tuple中各元素的类型不必相同
    let tup: (i32, f64, u8) = (500, 6.4, 1);
    
    // 获取Tuple的元素值
    // 可以使用模式匹配来解构（destructure）一个Tuple来获取元素的值
    let (x, y, z) = tup;
    println!("{}, {}, {}", x, y, z);
    
    // 访问Tuple的元素
    // 在Tuple变量使用点标记法，后接元素的索引号
    println!("{}, {}, {}", tup.0, tup.1, tup.2);
}
```

#### 数组
* 数组也可以将多个值放在一个类型里
* 数组中每个元素的类型必须相同
* 数组的长度也是固定的

示例：
```rust
fn main() {
    // 声明一个数组
    // 在中括号里，各值用逗号分开
    let a = [1, 2, 3, 4, 5];
    let months = [
        "January",
        "February",
        "March",
        "April",
        "May",
        "June",
        "July",
        "August",
        "September",
        "October",
        "November",
        "December",
    ];
}
```

数组的用处：
* 如果想让数据存放在stack(栈)上而不是heap(堆)上，或者想保证有固定数量的元素，这时使用数组更有好处
* 数组没有Vector灵活
  - Vector和数组类似，它由标准库提供
  - Vector的长度可以改变
  - 如果不确定应该用数组还是Vector，那么估计应该使用Vector

数组的类型：
* 数组的类型以这周形式表示：[类型; 长度]
  - 例如： let a:[i32; 5] = [1, 2, 3, 4, 5];

示例：
```rust
fn main() {
    // 指定类型和长度
    let a:[i32; 5] = [1, 2, 3, 4, 5];
    
    // 另一种声明数组的方法
    // 如果数组的每个元素值都相同，那么可以在：
    // 在中括号里指定初始值，然后是一个“;”，最后是数组的长度
    let a = [3; 5];
    // 等价于
    let a = [3, 3, 3, 3, 3];
    
    // 访问数组的元素
    // 数组是Stack上分配的单个块内存
    // 可以使用索引来访问数组的元素
    let first = a[0];
    let second = a[1];
    
    // 如果访问的索引超出了数组的范围，那么
    // 编译会通过(不是绝对，会做最简单的检查)，运行会报错(runtime时会panic)
    // Rust不会允许其继续访问相应地址的内存
    let last = a[5] // 报错，简单的编译器检查不通过
    
    let index = [6, 7];
    let last = a[index[1]]; // 虽然是非法的，但是编译会通过，运行时将panic
}
```

### 函数
* 声明函数使用fn关键字
* 依照惯例，针对函数和变量名，Rust使用snake case命名规范
  - 所有的字母都是小写，单词之间使用下划线分开
* 在函数签名里，必须声明每个参数的类型
* 函数体由一系列语句组成，可选的由一个表达式结束(相当于返回这个表达式的值)
* Rust是一个基于表达式的语言
* 语句是执行一些动作的指令
* 表达式会计算产生一个值(表达式本质可以理解就是一个值)
* 函数的定义也是语句
* 语句不返回值，所以不可以使用let将一个语句赋给一个变量
* 在-> 符号后边声明函数返回值的类型，但是不可以为返回值命名
* 在Rust里面，返回值就是函数体里面最后一个表达式的值
* 若想提前返回，需使用return关键字，并指定一个值
  - 大多数函数都是默认使用最后一个表达式作为返回值

示例：
```rust
// 整个函数的定义是一个语句
fn main() {
    // 这是一个语句
    let y = 6; // 6是一个字面值，所有字面值都是表达式
    let y = 5 + 6; // 5+6也是一个表达式，表达式都对应一个值。 
    
    // 表达式可以作为语句的一部分，调用函数也是一个表达式，调用宏也是表达式，比如println!，一个块也是一个表达式
    let y = {
      let x = 1;
      x + 3 // x + 3,是一个表达式，也是这个块的值，因为是最后一个表达式，因此y的值就是好4
    };
  
    // let x = (let z = 6); // 编译报错，不能够将statement(语句)赋值给变量，期望是一个表达式
    
    let x = plus_five(6); // x == 11
    
}

fn plus_five(x: i32) -> i32 {
    x + 5 // 最后的表达式作为函数的返回值
}
```

### 控制流(if else)
* if表达式允许根据条件来执行不同的代码分支
  - 这个条件必须是bool类型
* if表达式中，与条件相关联的代码块叫做分支(arm)  
* 可选的，可以在后面加上一个else表达式
* 但如果使用了多于一个的else if，那么最好是使用match来重构代码

示例：
```rust
fn main() {
    let number = 6;
    
    // if else if else 的判断顺序是从上往下依次判断，谁在前面就执行对应的逻辑
    if number % 4 == 0 {
      println!("number is divisible by 4");
    } else if number % 3 == 0 {
      println!("number is divisible by 3"); // 执行进入这个块
    } else if number % 2 == 0 {
      println!("number is divisible by 2");
    } else {
      println!("number is not divisible by 4, 3 or 2");
    }
    // match 重构
    match number {
      num if num % 4 == 0 => println!("number is divisible by 4"),
      num if num % 3 == 0 => println!("number is divisible by 3"),
      num if num % 2 == 0 => println!("number is divisible by 2"),
      _ => println!("number is not divisible by 4, 3 or 2"),
    }
}
```

在let语句中使用if：
* 因为if是一个表达式，所以可以将它放在let语句中等号的右边

示例：
```rust
fn main() {
    let cond = true;
    
    let number = if cond { 5 } else { 6 };
    
    println!("The value of number is: {}", number);
}
```

### 控制流(循环)
* Rust提供了3种循环：loop，while和for
* loop关键字告诉Rust反复执行一块代码，直到你喊停
* 可以在loop循环中使用break关键字来告诉程序何时停止循环
* while条件循环，是每次执行循环体之前都判断一次条件
* for循环更适合集合的遍历，更简洁紧凑，它可以针对集合中的每个元素来执行一些代码
* 由于使用for循环的安全、简洁性，所以它在Rust里面用的最多

示例：
```rust
fn main() {
    let mut counter = 0;
    let result = loop {
        counter += 1;
        if counter == 10 {
          break counter * 2
        }
    };
    println!("The result is: {}", result);

    let mut number = 3;
    while number != 0 {
      println!("{}!", number);
      number -= 1;
    }
    println!("LIFTOFF!!!");

    let a = [10, 20, 30, 40, 50];
    for element in a.iter() {
      println!("the value is: {}", element);
    }
}
```

for 配合Range使用：
* 标准库提供
* 指定一个开始数字和一个结束数字，Range 可以生成它们之间的数字(不包含结束)
* rev 方法可以反转Range

示例：
```rust
fn main() {
    for number in (1..4).rev() {
      println!("{}!", number);
    }
    println!("LIFTOFF!");
}
```
