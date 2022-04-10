---
title: PingCAP 故障注入利器 fail-rs
categories: rust
toc: true
---

![](/images/inject-fault-cover.jpg)

年初分享过[聊聊 Go failpoint 使用](https://mp.weixin.qq.com/s/R6LYv20bM9hsZjlDHYYnBQ)，感兴趣的可以看看看这篇文章

`Failpoints` 是一个允许在运行时注入错误或是其它行为的工具，主要用于测试目的，包括 ut 单测，集成压测等等。测试的内容包括状态机错误，磁盘错误，网络 IO 延迟

可以注入的行为有：`panic`, `early returns`, `sleeping` 等等，注入的形为可以通过环境变量或代码进行控制。一般推荐用 http 或集成公司的配置平台，触发规则可以是次数，概率或是两种的结合

### 入门案例
首先配置依赖，Cargo.toml
```rust
[dependencies]
fail = "0.4"
```
我们依赖 0.4 版本
```rust
use fail::{fail_point, FailScenario};

fn do_fallible_work() {
    fail_point!("read-dir");
    println!("mock working now");
}

fn main() {
    let scenario = FailScenario::setup();
    do_fallible_work();
    scenario.teardown();
    println!("done");
}
```
`do_fallible_work` 函数只做两件事情，执行 read-dir 注入点，打印消息用于模拟函数处理请求
```rust
$ FAILPOINTS=read-dir="panic" cargo run
mock working now
done
```
通过环境变量注入 panic 语句，条件编译默认没有开启，所以正常输出
```rust
$ FAILPOINTS=read-dir="panic" cargo run --features fail/failpoints
mock working now
thread 'main' panicked at 'failpoint read-dir panic', /Users/zerun.dong/.cargo/registry/src/github.com-1ecc6299db9ec823/fail-0.4.0/src/lib.rs:488:25
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```
cargo 指定 `--features fail/failpoints`, 发生 Panic 符合预期
```rust
FAILPOINTS=read-dir="sleep(2000)" cargo run --features fail/failpoints
```
当然我们也可以指定其它行为，比如 `sleep(2000)` 休眠 2 秒
```rust
use fail::{fail_point, FailScenario};
use std::io;

fn do_fallible_work() -> io::Result<()>{
    println!("mock working now");
    fail_point!("read-dir", |_| {
        Err(io::Error::new(io::ErrorKind::PermissionDenied, "error"))
    });
    Ok(())
}

fn main() -> io::Result<()> {
    let scenario = FailScenario::setup();
    do_fallible_work()?;
    do_fallible_work()?;
    scenario.teardown();
    println!("done");
    Ok(())
}
```
这是测试提前返回 early return 的案例，需要使用闭包来封装 error
```rust
$ FAILPOINTS=read-dir=return cargo run --features fail/failpoints
mock working now
Error: Custom { kind: PermissionDenied, error: "error" }
```
上面是普通用法，也可以指定多个 action
```rust
$ FAILPOINTS=read-dir="1*sleep(2000)->return" cargo run --features fail/failpoints
mock working now
mock working now
Error: Custom { kind: PermissionDenied, error: "error" }
```
`"1*sleep(2000)->return"` 表示第一次休眠 2 秒，然后第二次时提前返回。关于更多高级用法，请参考官网 https://docs.rs/fail
### 零性能消耗
最重要的要求是：**集成 `Failpoint` 的代码，在线上正式环境运行时，要做到零性能消耗**

```go
func test() {
    failpoint.Inject("testValue", func(v failpoint.Value) {
        fmt.Println(v)
    })
}
```
这是 go 测试代码，`failpoint.Inject` 是 `marker` 函数，参数是名称和闭包
```go
// failpoint.Inject("fail-point-name", func(_ failpoint.Value) (...){}
func Inject(fpname string, fpbody interface{}) {}
```
由于 `Inject` 是空函数体，编译时会被优化掉，所以运行时零性能消耗。当线下测试时，需要执行 `failpoint-ctl` 将所有 marker 函数转化成注入函数
```go
func test() {
 if v, _err_ := failpoint.Eval(_curpkg_("testValue")); _err_ == nil {
  fmt.Println(v)
 }
}
```
上面是转换后的代码，原理不难，解析 AST 替换语法树。**那么 rust 如何实现呢？答案是 macro 宏 + 条件编译**
```rust
/// Define a fail point (disabled, see `failpoints` feature).
#[macro_export]
#[cfg(not(feature = "failpoints"))]
macro_rules! fail_point {
    ($name:expr, $e:expr) => {{}};
    ($name:expr) => {{}};
    ($name:expr, $cond:expr, $e:expr) => {{}};
}
```
当 cargo build 编译时未指定 `failpints` feature, `fail_point` 宏对应空实现
```rust
#[cfg(feature = "failpoints")]
macro_rules! fail_point {
    ($name:expr) => {{
        $crate::eval($name, |_| {
            panic!("Return is not supported for the fail point \"{}\"", $name);
        });
    }};
    ($name:expr, $e:expr) => {{
        if let Some(res) = $crate::eval($name, $e) {
            return res;
        }
    }};
    ($name:expr, $cond:expr, $e:expr) => {{
        if $cond {
            fail_point!($name, $e);
        }
    }};
}
```
指定 feature 时，对应上面的宏实现，编译期展开成相应的逻辑代码。`fail_point` 宏有三种形式，模式匹配到不同的参数表达式 (designators) 对应不同代码块

* 单个参数 name 字符串，可以执行 panic, print, sleep, pause 四种行为
* 两个参数 name, e 这里面 e 是闭包，可以做到 early return 提前返回
* 三个参数 name, cond, e, 其中 cond 是条件表达式，应该返回 bool 值，e 是闭包。根据条件来执行对应注入

### 实现原理
#### 1.注册中心
```rust
/// Registry with failpoints configuration.
type Registry = HashMap<String, Arc<FailPoint>>;

#[derive(Debug, Default)]
struct FailPointRegistry {
    // TODO: remove rwlock or store *mut FailPoint
    registry: RwLock<Registry>,
}

lazy_static::lazy_static! {
    static ref REGISTRY: FailPointRegistry = FailPointRegistry::default();
    static ref SCENARIO: Mutex<&'static FailPointRegistry> = Mutex::new(&REGISTRY);
}
```
注册中心 `Registry` 是 HashMap 类型，key 是上面测试例子的 `name`, value 是 `Arc<Failpoint>` 类型，[Arc 用于并发环境下共享所有权](https://mp.weixin.qq.com/s/hjOLVK2FTZsyaNkJXJB2QQ)
```rust
struct FailPoint {
    pause: Mutex<bool>,
    pause_notifier: Condvar,
    actions: RwLock<Vec<Action>>,
    actions_str: RwLock<String>,
}
```
`pause` 表示是否暂停，`pause_notifier` 用于暂停通知，`actions` 是一个数组，因为一个 fail_point 注入可以有多个动作，`actions_str` 是表示任务的字符串，通过 `from_str` 转化成 `action` 结构体
#### 2.生成任务
`FailScenario::setup()` 通过获取 `FAILPOINTS` 环境变量来初始化注入动作，暂时不支持通过 http 方式

解析后通过 `set` 函数将多个注入动作解析，注册到上文提到的 `Registry`
```rust
fn set(
    registry: &mut HashMap<String, Arc<FailPoint>>,
    name: String,
    actions: &str,
) -> Result<(), String> {
    let actions_str = actions;
    // `actions` are in the format of `failpoint[->failpoint...]`.
    let actions = actions
        .split("->")
        .map(Action::from_str)
        .collect::<Result<_, _>>()?;
    // Please note that we can't figure out whether there is a failpoint named `name`,
    // so we may insert a failpoint that doesn't exist at all.
    let p = registry
        .entry(name)
        .or_insert_with(|| Arc::new(FailPoint::new()));
    p.set_actions(actions_str, actions);
    Ok(())
}
```
这里面用 `Action::from_str` 将字符串解析成 `Action`
```rust
#[derive(Clone, Debug, PartialEq)]
enum Task {
    /// Do nothing.
    Off,
    /// Return the value.
    Return(Option<String>),
    /// Sleep for some milliseconds.
    Sleep(u64),
    /// Panic with the message.
    Panic(Option<String>),
    /// Print the message.
    Print(Option<String>),
    /// Sleep until other action is set.
    Pause,
    /// Yield the CPU.
    Yield,
    /// Busy waiting for some milliseconds.
    Delay(u64),
    /// Call callback function.
    Callback(SyncCallback),
}

#[derive(Debug)]
struct Action {
    task: Task,
    freq: f32,
    count: Option<AtomicUsize>,
}
```
`Action` 类型都不一样，`freq` 控制频率，`count` 控制触发次数
#### 3.触发任务
大前提肯定是条件编译打开了 failpoint, 直接看 macro 实现
```rust
pub fn eval<R, F: FnOnce(Option<String>) -> R>(name: &str, f: F) -> Option<R> {
    let p = {
        let registry = REGISTRY.registry.read().unwrap();
        match registry.get(name) {
            None => return None,
            Some(p) => p.clone(),
        }
    };
    p.eval(name).map(f)
}
```
逻辑比较简单，从 `Registry` 注册中心 map 找到对应 `failpoint`, 然后调用 `failpoint.eval` 函数，并且针对所有返回值执行闭句 f (如果有值)
```rust
#[cfg_attr(feature = "cargo-clippy", allow(clippy::option_option))]
fn eval(&self, name: &str) -> Option<Option<String>> {
    let task = {
        let actions = self.actions.read().unwrap();
        match actions.iter().filter_map(Action::get_task).next() {
            Some(Task::Pause) => {
                let mut guard = self.pause.lock().unwrap();
                *guard = true;
                loop {
                    guard = self.pause_notifier.wait(guard).unwrap();
                    if !*guard {
                        break;
                    }
                }
                return None;
            }
            Some(t) => t,
            None => return None,
        }
    };

    match task {
        Task::Off => {}
        Task::Return(s) => return Some(s),
        Task::Sleep(t) => thread::sleep(Duration::from_millis(t)),
        Task::Panic(msg) => match msg {
            Some(ref msg) => panic!("{}", msg),
            None => panic!("failpoint {} panic", name),
        },
        Task::Print(msg) => match msg {
            Some(ref msg) => log::info!("{}", msg),
            None => log::info!("failpoint {} executed.", name),
        },
        Task::Pause => unreachable!(),
        Task::Yield => thread::yield_now(),
        Task::Delay(t) => {
            let timer = Instant::now();
            let timeout = Duration::from_millis(t);
            while timer.elapsed() < timeout {}
        }
        Task::Callback(f) => {
            f.run();
        }
    }
    None
}
```
`eval` 函数不难，首先调用 `get_task` 获取要执行的 `Action`, 这里 `Pause` 动作单独处理，其它的通过 match 模式匹配。**同时也能看到，如果 Return 不指定闭包 f, 那么返回值是 Some(""), 触发 macro 的默认 panic 闭包**
```rust
fn get_task(&self) -> Option<Task> {
  use rand::Rng;

  if let Some(ref cnt) = self.count {
      let c = cnt.load(Ordering::Acquire);
      if c == 0 {
          return None;
      }
  }
  if self.freq < 1f32 && !rand::thread_rng().gen_bool(f64::from(self.freq)) {
      return None;
  }
  if let Some(ref ref_cnt) = self.count {
      let mut cnt = ref_cnt.load(Ordering::Acquire);
      loop {
          if cnt == 0 {
              return None;
          }
          let new_cnt = cnt - 1;
          match ref_cnt.compare_exchange_weak(
              cnt,
              new_cnt,
              Ordering::AcqRel,
              Ordering::Acquire,
          ) {
              Ok(_) => break,
              Err(c) => cnt = c,
          }
      }
  }
  Some(self.task.clone())
}
```
`get_task` 先判断执行次数，如果为 0 返回空。然后判断频率，如果没有触发返回空，最后再判断一次计数，并 cas 更新。这里 `count` 计数字段类型是 `Option<AtomicUsize>`, 如果不指定次数默认无限制
### 小结 
写文章不容易，如果对大家有所帮助和启发，请大家帮忙点击`在看`，`点赞`，`分享` 三连

关于 `Failpoint` 大家有什么看法，欢迎留言一起讨论，大牛多留言 ^_^

![](/images/dongzerun-weixin-code.png)