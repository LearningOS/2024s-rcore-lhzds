# 从哪获取 Task 信息

- TaskManager 模块负责管理模块，current_task 指定当前 task index，通过 current_task 可以索引当前 task 的 control block
    
    ```rust
    // os/src/task/mod.rs
    
    pub struct TaskManager {
        num_app: usize,
        inner: UPSafeCell<TaskManagerInner>,
    }
    
    struct TaskManagerInner {
        tasks: [TaskControlBlock; MAX_APP_NUM],
        current_task: usize,
    }
    ```
    

# Task 控制块要添加的数据

```rust
pub struct TaskControlBlock {
    ...
    /// Whether the task has been scheduled
    pub been_sched: bool,
    /// the time when the task was first scheduled
    pub start_time_ms: usize,
    /// The numbers of syscall called by task
    pub syscall_times: [u32; MAX_SYSCALL_NUM]
}
```

# 维护运行时间

思路：task 首次状态变为 running 时记录其 `start_time_ms`，每次调用 `sys_task_info` 时将当前时间减去 `start_time_ms` 记录过了多少时间 

- 记录 start_time_ms 需要修改两处

```rust
fn run_first_task(&self) -> ! {
    ...
    task0.been_sched = true;
    task0.start_time_ms = get_time_ms();
    ...
}
...
fn run_next_task(&self) {
    if let Some(next) = self.find_next_task() {
        let mut inner = self.inner.exclusive_access();
        let current = inner.current_task;
        inner.tasks[next].task_status = TaskStatus::Running;
        if !inner.tasks[next].been_sched {
            inner.tasks[next].been_sched = true;
            inner.tasks[next].start_time_ms = get_time_ms();
        }
        ...
}
```

- 计算过了多少时间

```rust
/// YOUR JOB: Finish sys_task_info to pass testcases
pub fn sys_task_info(_ti: *mut TaskInfo) -> isize {
    trace!("kernel: sys_task_info");
    let cb = get_current_task_cb();
    unsafe { 
        (*_ti).status = cb.task_status;
        (*_ti).syscall_times = cb.syscall_times;
        (*_ti).time = get_time_ms() - cb.start_time_ms;
    }
    0
}
```

# 维护系统调用次数

- 系统调用入口函数处根据 syscall_id 增加调用次数

```rust
pub fn syscall(syscall_id: usize, args: [usize; 3]) -> isize {
    add_syscall_time(syscall_id);
    match syscall_id {
        ...
    }
}
```

- 内部实现：桶数组计数

```rust
fn add_syscall_time(&self, syscall_id: usize) {
    let mut inner = self.inner.exclusive_access();
    let current = inner.current_task;
    inner.tasks[current].syscall_times[syscall_id] += 1;
}
```