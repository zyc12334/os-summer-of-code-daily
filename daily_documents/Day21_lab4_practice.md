# 实验四（上）：线程

1. 原理：线程切换之中，页表是何时切换的？页表的切换会不会影响程序 / 操作系统的运行？为什么？

    这里实际上和实验指导中是不同的；

    在实验指导中：

    os/src/process/thread.rs: impl Thread

    ```rs
    /// 准备执行一个线程
    ///
    /// 激活对应进程的页表，并返回其 Context
    pub fn run(&self) -> *mut Context {
        // 激活页表
        self.process.read().memory_set.activate();
        // 取出 Context
        let parked_frame = self.context.lock().take().unwrap();

        if self.process.read().is_user {
            // 用户线程则将 Context 放至内核栈顶
            KERNEL_STACK.push_context(parked_frame)
        } else {
            // 内核线程则将 Context 放至 sp 下
            let context = (parked_frame.sp() - size_of::<Context>()) as *mut Context;
            unsafe { *context = parked_frame };
            context
        }
    }
    ```

    在 lab4 的分支代码中，应当是在 Process::prepare_next_thread() 中调用 Thread::prepare():

    ```rs
    /// 准备执行一个线程
    ///
    /// 激活对应进程的页表，并返回其 Context
    pub fn prepare(&self) -> *mut Context {
        // 激活页表
        self.process.write().memory_set.activate();
        // 取出 Context
        let parked_frame = self.inner().context.take().unwrap();
        // 将 Context 放至内核栈顶
        unsafe { KERNEL_STACK.push_context(parked_frame) }
    }
    ```

    它不会影响执行，因为在中断期间是操作系统正在执行，而操作系统所用到的内核线性映射是存在于每个页表中的。（这我记得重复了
    
    上下文 Context 的保存和取出：

    >思考：在 run 函数中，我们在一开始就激活了页表，会不会导致后续流程无法正常执行？
    >
    >不会，因为每一个进程的 MemorySet 都会映射操作系统的空间，否则在遇到中断的时候，将无法执行异常处理。

2. 设计：如果不使用 sscratch 提供内核栈，而是像原来一样，遇到中断就直接将上下文压栈，请举出（思路即可，无需代码）：

- 一种情况不会出现问题
- 一种情况导致异常无法处理（指无法进入 handle_interrupt）
- 一种情况导致产生嵌套异常（指第二个异常能够进行到调用 handle_interrupt，不考虑后续执行情况）
- 一种情况导致一个用户进程（先不考虑是怎么来的）可以将自己变为内核进程，或以内核态执行自己的代码

解答：

由于中断可能是多种原因导致的，也包含了不少错误等内容，因此可以从这些方向考虑：

- 只运行一个非常善意的线程，比如 loop {}：这种情况下和内核进程类似
- 线程把自己的 sp 搞丢了，比如 mv sp, x0。此时无法保存寄存器，也没有能够支持操作系统正常运行的栈：这种情况下出现错误之后就没有办法处理异常和恢复操作系统的状态，操作系统也会崩溃
- 运行两个线程。在两个线程切换的时候，会需要切换页表。但是此时操作系统运行在前一个线程的栈上，一旦切换，再访问栈就会导致缺页，因为每个线程的栈只在自己的页表中；
- 用户进程巧妙地设计 sp，使得它恰好落在内核的某些变量附近，于是在保存寄存器时就修改了变量的值。这相当于任意修改操作系统的控制信息；（这个回答我觉得和题目有点小关联不紧，不过可以提供思路）



1. 实验：当键盘按下 Ctrl + C 时，操作系统应该能够捕捉到中断。实现操作系统捕获该信号并结束当前运行的线程（你可能需要阅读一点在实验指导中没有提到的代码）

    这边有一个qemu特有的属性，需要按下 Ctrl + Alt + C 来在虚拟机中导入 Ctrl + C ，这个要注意一下；

    具体实现比较简单：

    在 `interrupt/handler.rs` 中，修改 `supervisor_external` 函数：

    ```rs
    /// 处理外部中断，只实现了键盘输入
    fn supervisor_external(context: &mut Context) -> Result<*mut Context, String> {
        let mut c = console_getchar();
        if c <= 255 {
            if c == 3 {
                PROCESSOR.get().kill_current_thread();
                PROCESSOR.get().prepare_next_thread();
            }else{
                if c == '\r' as usize {
                    c = '\n' as usize;
                }
                STDIN.push(c as u8);
            }
        }
        Ok(context)
    }


    ```

2. 实验：实现线程的 fork()。目前的内核线程不能进行系统调用，所以我们先简化地实现为“按 F 进入 fork”。fork 后应当为目前的线程复制一份几乎一样的拷贝，新线程与旧线程同属一个进程，公用页表和大部分内存空间，而新线程的栈是一份拷贝。

代码：

handler.rs

```rs

/// 处理外部中断，只实现了键盘输入
fn supervisor_external(context: &mut Context) -> Result<*mut Context, String> {
    let mut c = console_getchar();
    if c <= 255 {
        if c == 3 {
            PROCESSOR.get().kill_current_thread();
            PROCESSOR.get().prepare_next_thread();
        }
        else if c == 'f' as usize {
            PROCESSOR.get().fork_current_thread(context);
        }
        else{
            if c == '\r' as usize {
                c = '\n' as usize;
            }
            STDIN.push(c as u8);
        }
    }
    Ok(context)
}

```

processsor.rs

```rs

    /// 线程的 fork()
    /// fork 后应当为目前的线程复制一份几乎一样的拷贝，新线程与旧线程同属一个进程，公用页表和大部分内存空间，而新线程的栈是一份拷贝。
    pub fn fork_current_thread(&mut self, context: &Context){
        let thread = self.current_thread().fork(*context).unwrap();
        self.add_thread(thread);
    }

```

thread.rs

```rs

pub fn fork(&self, current_context: Context) -> MemoryResult<Arc<Thread>> {

        println!("enter fork");

        // 让所属进程分配并映射一段空间，作为线程的栈
        let stack = self.process
            .write()
            .alloc_page_range(STACK_SIZE, Flags::READABLE | Flags::WRITABLE)?;

        for i in 0..STACK_SIZE {
            *VirtualAddress(stack.start.0 + i).deref::<u8>() = *VirtualAddress(self.stack.start.0 + i).deref::<u8>()
        }

        let mut context = current_context.clone();

        context.set_sp( usize::from(stack.start) -  usize::from(self.stack.start) + current_context.sp()  );

        // 打包成线程
        let thread = Arc::new(Thread {
            id: unsafe {
                THREAD_COUNTER += 1;
                THREAD_COUNTER
            },
            stack,
            process: Arc::clone(&self.process),
            inner: Mutex::new(ThreadInner {
                context: Some(context),
                sleeping: false,
                descriptors: vec![STDIN.clone(), STDOUT.clone()],
            }),
        });

        Ok(thread)
    }

```

测试用户程序：

```rs

pub fn main() -> usize {
    println!("Hello world from user mode program!");
    for i in 0..10000000{
        if i%1000000 == 0 {
            println!("Hello world from user mode program!{}",i);
        }
    }
    0
}

```

输出：

```sh

Hello world from user mode program!6000000
Hello world from user mode program!7000000
Hello world from user mode program!6000000
Hello world from user mode program!8000000
Hello world from user mode program!7000000
Hello world from user mode program!9000000
Thread 3 exit with code 0
Hello world from user mode program!8000000
Hello world from user mode program!9000000
Thread 2 exit with code 0
src/process/processor.rs:101: 'all threads terminated, shutting down'
make[1]: Leaving directory '/home/yunwei/rCore-Tutorial-lab-4/os'

```

# 实验四（下）：线程调度

1. 实验：了解并实现 Stride Scheduling 调度算法，为不同线程设置不同优先级，使得其获得与优先级成正比的运行时间。

os/src/algorithm/src/scheduler/stride_scheduler.rs

```rs

//! 最高响应比优先算法的调度器 [`HrrnScheduler`]

use super::Scheduler;
use alloc::collections::LinkedList;

/// 将线程和调度信息打包
struct StrideThread<ThreadType: Clone + Eq> {
    /// ticket
    ticket: usize,
    /// stride
    pass: usize,
    /// 线程数据
    pub thread: ThreadType,
}

/// 采用 HRRN（最高响应比优先算法）的调度器
pub struct StrideScheduler<ThreadType: Clone + Eq> {
    /// max stride
    big_stride:usize,
    current_min:usize,
    /// 带有调度信息的线程池
    pool: LinkedList<StrideThread<ThreadType>>,
}

/// `Default` 创建一个空的调度器
impl<ThreadType: Clone + Eq> Default for StrideScheduler<ThreadType> {
    fn default() -> Self {
        Self {
            big_stride: 137,
            current_min: 0,
            pool: LinkedList::new(),
        }
    }
}

impl<ThreadType: Clone + Eq> Scheduler<ThreadType> for StrideScheduler<ThreadType> {

    fn add_thread(&mut self, thread: ThreadType, _priority: usize) {
            self.pool.push_back(StrideThread {
                ticket: _priority,
                pass: self.current_min,
                thread,
            })
    }

    fn get_next(&mut self) -> Option<ThreadType> {
        // 计时

        if let Some(best) = self.pool.iter_mut().min_by(|x, y| {
            (x.pass)
                .cmp(&(y.pass))
        }) {
            if best.ticket == 0 {
                best.pass += self.big_stride;
            }else{
                best.pass += self.big_stride / ( best.ticket + 1 );
            }
            self.current_min = best.pass;
            Some(best.thread.clone())
        } else {
            None
        }
    }

    fn remove_thread(&mut self, thread: &ThreadType) {
        // 移除相应的线程并且确认恰移除一个线程
        let mut removed = self.pool.drain_filter(|t| t.thread == *thread);
        assert!(removed.next().is_some() && removed.next().is_none());
    }

    fn set_priority(&mut self, _thread: ThreadType, _priority: usize) {
        for x in self.pool.iter_mut(){
            if x.thread == _thread {
                x.ticket = _priority as usize;
            }
        }
    }
}

```

修改一下 thread 的定义：

```rs
/// 线程中需要可变的部分
pub struct ThreadInner {
    /// 线程执行上下文
    ///
    /// 当且仅当线程被暂停执行时，`context` 为 `Some`
    pub context: Option<Context>,
    /// 是否进入休眠
    pub sleeping: bool,
    /// 是否已经结束
    pub dead: bool,
    /// priority
    pub priority: usize,
}
```

测试代码：

```rs
fn sample_process(id: usize) {
    for i in 0..4000000{
        if i%1000000 == 0 {
            println!("Hello world from kernel mode {} program!{}",id,i);
        }
    }
}
```

结果：

```

Hello world from kernel mode 2 program!0
Hello world from kernel mode 3 program!0
Hello world from kernel mode 4 program!0
Hello world from kernel mode 5 program!0
Hello world from kernel mode 6 program!0
Hello world from kernel mode 7 program!0
Hello world from kernel mode 8 program!0
Hello world from user mode program!
thread 9 exit with code 0
Hello world from kernel mode 1 program!0
Hello world from kernel mode 8 program!1000000
Hello world from kernel mode 7 program!1000000
Hello world from kernel mode 6 program!1000000
Hello world from kernel mode 5 program!1000000
Hello world from kernel mode 4 program!1000000
Hello world from kernel mode 8 program!2000000
Hello world from kernel mode 3 program!1000000
Hello world from kernel mode 7 program!2000000
Hello world from kernel mode 6 program!2000000
Hello world from kernel mode 2 program!1000000
Hello world from kernel mode 5 program!2000000
Hello world from kernel mode 8 program!3000000
Hello world from kernel mode 7 program!3000000
Hello world from kernel mode 4 program!2000000
Hello world from kernel mode 6 program!3000000
thread 8 exit
Hello world from kernel mode 5 program!3000000
Hello world from kernel mode 3 program!2000000
thread 7 exit
Hello world from kernel mode 1 program!1000000
thread 6 exit
Hello world from kernel mode 4 program!3000000
thread 5 exit
Hello world from kernel mode 2 program!2000000
Hello world from kernel mode 3 program!3000000
thread 4 exit
Hello world from kernel mode 2 program!3000000
thread 3 exit
Hello world from kernel mode 1 program!2000000
thread 2 exit
Hello world from kernel mode 1 program!3000000
thread 1 exit
```

2. 分析：
    - 在 Stride Scheduling 算法下，如果一个线程进入了一段时间的等待（例如等待输入，此时它不会被运行），会发生什么？
      - 如果在这种简单的实现下，有可能会出现其他线程等待该线程的情况；比如一个要获取输入的进程的优先级较高，要等它的 pass 经过多个时间片比其他的线程大的时候其他线程才会被调度。
    - 对于两个优先级分别为 9 和 1 的线程，连续 10 个时间片中，前者的运行次数一定更多吗？
     - 并不一定，因为有可能9的线程运行了一下就结束了。
    - 你认为 Stride Scheduling 算法有什么不合理之处？可以怎样改进？
      - 可能会出现等待进程的情况；
      - 也要注意，新加进去的进程的pass不能是0，否则会一直霸占着时间片；
