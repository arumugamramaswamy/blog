+++
date = '2026-08-05T22:59:17+08:00'
draft = false
title = 'Rust: Thread per Core Executor'
+++


- doing round of optimizations at work
- we use C# + a thread per core model
- realized that we were allocating a lot when running async tasks 
- our custom statemachine builder was making a lot of allocations - mainly in allocating delegates for the `OnCompleted(Action)` on the awaiter
- remembered reading somewhere that rust was supposed to have zero cost async


- [Phil Opp](https://os.phil-opp.com/async-await/)
	- one of best resources for async/ await + pinning I've seen
- read through this to get some inspiration (a lot of my code started off as just copying over code from the post)
- goal: build a thread per core rust executor - powered by io_uring
- resources I kept coming back to:
	- the phil-opp blog post
	- monoio - popular thread per core async runtime
	- chatgpt

- goals for the executor:
	- to be able to spawn tasks from anywhere and have them run to completion in the background
	- any task with `.await` inside it behaves as expected

- I'll assume you're already comfortable with `Pin`.
- If not, I'd strongly recommend:
	- the phil-opp blog post
	- [std::pin](https://doc.rust-lang.org/std/pin/) docs

- declare task:
```rust
struct Task(Pin<Box<dyn Future<Output = ()>>>);
```

- declare executor:
```rust
struct Executor;

impl Executor {
    fn new() -> Self{
        Self
    }

    fn spawn(&mut self, future: impl Future<Output = ()> + 'static){
        todo!()
    }
    
    fn run_to_completion(&mut self) {
        todo!()
    }
}
```

- basic main:
```rust

fn main() {
    let mut e = Executor::new();
    e.spawn(async {
        println!("Should run zeroth!");
    });
    e.run_to_completion();
}

```

- this obviously errors on `spawn`
- let's implement `spawn` (the easy part)
```rust
struct Executor {
    tasks: VecDeque<Task>
};

impl Executor {
    fn new() -> Self {
        Self {
            tasks: vec![].into(),
        }
    }

    fn spawn(&mut self, future: impl Future<Output = ()> + 'static){
        self.tasks.push_back(Task(Box::pin(future)));
    }
    
    ...
}
```

- now this works, but errors on `run_to_completion`
- implement `run_to_completion` (naively)

```rust
impl Executor {
    ...
    
    fn run_to_completion(&mut self) {
        while let Some(mut task) = self.tasks.pop_front() {
            let waker = Waker::noop();
            let mut cx = Context::from_waker(&waker);
            match task.0.as_mut().poll(&mut cx) {
                std::task::Poll::Ready(_) => {}
                std::task::Poll::Pending => self.tasks.push_back(task),
            }
        }
    }
}
```

- this now works!
- But ...
- this does nothing useful

- let's add cooperative multitasking using `yield_now`
- goal: add current executing task to the back of the queue

```rust
/// Add the current executing task to the back of the queue
pub fn yield_now() -> impl Future<Output = ()> {
    Yield::default()
}

#[derive(Default)]
struct Yield;

impl Future for Yield {
    type Output = ();

    fn poll(
        mut self: std::pin::Pin<&mut Self>,
        cx: &mut std::task::Context<'_>,
    ) -> std::task::Poll<Self::Output> {
        todo!()
    }
}

fn main() {
    let mut e = Executor::new();
    e.spawn(async {
        println!("Should run zeroth!");
	yield_now().await;
        println!("Should run second!");
    });
    e.spawn(async {
        println!("Should run first!");
    });
    e.run_to_completion();
}
```

- scaffolding done
- now when we poll `Yield` for the first time, we should return pending, then return done
```rust
#[derive(Default)]
struct Yield {
    done: bool
}

impl Future for Yield {
    type Output = ();

    fn poll(
        mut self: std::pin::Pin<&mut Self>,
        cx: &mut std::task::Context<'_>,
    ) -> std::task::Poll<Self::Output> {
        if self.done {
	    Poll::Ready(())
	} else {
	    self.done = true;
	    Poll::Pending
	}
    }
}
```

- and this now works!
- caveat:
    - normally, a future returning `Poll::Pending` is responsible for arranging to be woken again
    - we don't do this yet because our executor polls each future until it is complete
    - it is something we will add in the future though

- Stay tuned for the next part: "Timers & IO URING"
