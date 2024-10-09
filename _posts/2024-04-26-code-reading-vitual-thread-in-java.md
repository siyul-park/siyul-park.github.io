---
title: "코드로 읽는 가상 스레드"
date: 2024-04-26 12:00:00 +0900
categories: "system"
tags: ["system", "java"]
---

JDK 21에서 가상 스레드가 공식 기능으로 추가되었습니다. 가상 스레드가 추가되기 전에는 OS 스레드를 직접 생성하고 사용자 스레드에 매핑하는 플랫폼 스레드 방식이 사용되었습니다.

전통적으로 서버 애플리케이션에서는 독립적인 요청을 처리하기 위해 개별적인 스레드를 할당했습니다. 이 방식은 애플리케이션의 동시성 단위를 플랫폼의 동시성 단위와 일치시키므로 이해하기 쉽고 분석하기 쉽습니다.

그러나 요청량이 증가함에 따라 스레드 수도 증가되었습니다. OS 스레드는 비용이 많이 들어 CPU나 네트워크 연결과 같은 다른 리소스가 소진되기 전에 스레드 수가 제한되는 경우가 많으며, 스레드 수가 너무 많아져 자원을 얻기 위한 경쟁이 심해져 과도한 컨텍스트 스위칭이 종종 발생했습니다.

그리하여 스레드 수를 최대한 적게 유지하기 위해 스레드 풀을 사용하기 시작했습니다. 하나의 스레드에서 처음부터 끝까지 요청을 처리하는 대신 I/O 작업이 완료될 때까지 해당 스레드를 풀로 반환하여 다른 요청을 처리할 수 있도록 합니다. 그리하여 많은 동시 작업을 수행할 수 있게 되었지만 비동기 프로그래밍 스타일이 필요하게 되었습니다.

비동기식 스타일은 콜백, 반응형 같은 기존과 다른 방식이 필요합니다. 스택 추적이 가능한 컨텍스트를 제공하지 않으며, 디버거를 통해 각 단계를 따라가기 어렵고, 작업 비용을 호출자와 연결할 수 없어 프로파일링에 어려움이 있습니다.

가상 스레드의 등장으로 기존의 경험을 유지하면서도 비용을 줄일 수 있게 되었습니다. 이 방식은 스레드 풀을 사용하는 것과 유사하지만, 플랫폼에서 지원하는 기능과 긴밀하게 통합됩니다.

사실 이러한 가상 스레드는 JVM의 역사에서 처음 등장한 것은 아니였습니다. JVM 1.3 이전에는 일부 OS에서 Green Thread가 이용되기도 했습니다. Green Thread 는 Java 21에서 안정회된 가상 스레드와 달리 단 하나의 플렛폼 스레드에 여러개의 유저 스레드가 할당되었습니다. 프로세스의 코어가 늘어나도 플렛폼 스레드 수는 확장이 되지 않았으며, 가상 스레드와 다르게 유저 스레드가 정지하면 플렛폼 스레드도 정지되었습니다. Green Thread는 플렛폼 스레드를 만들기 어려운 상황에서 주로 사용이 되었으나, 멀티 코어 프로세스가 활성화되며 점차 사라지게 됩니다.

새롭게 나온 가상 스레드는 총 9가지 상태를 오가며 유저 스레드와 마운트되어 작업을 처리합니다. 하나의 유저 스레드는 여러 가상 스레드를 처리할 수 있습니다.

- `NEW`: 스레드가 생성되었지만 아직 시작되지 않은 초기 상태입니다. `Thread.start` 메소드를 호출하여 `STARTED` 상태로 전이됩니다.
- `STARTED`: 스레드가 시작되었지만 아직 실행되지 않은 상태를 나타냅니다. 실행이 실패하면 `TERMINATED` 상태로 전이되고, 실행이 성공하면 `RUNNING` 상태로 전이됩니다.
- `RUNNING`: 스레드가 실행되는 상태를 나타냅니다. 유저 스레드를 언마운트하려면 `PARKING` 상태로 전이됩니다. `Thread.yield` 메소드를 호출하여 `YIELDING` 상태로 전이될 수 있습니다.
- `PARKING`: 언마운트 중인 상태를 나타냅니다. `Thread.yield` 호출이 성공하면 `PARKED` 상태로, 실패하면 `PINNED` 상태로 전이됩니다.
- `PARKED`: 언마운트된 상태를 나타냅니다. `unpark` 메소드 호출이 발생하거나 인터럽트가 발생하면 `RUNNABLE` 상태로 전이됩니다.
- `PINNED`: 마운트된 상태이지만 실행되지 않는 상태를 나타냅니다. `unpark` 메소드 호출이 발생하거나 인터럽트가 발생하면 `RUNNABLE` 상태로 전이됩니다.
- `RUNNABLE`: 실행 가능한 상태이지만 아직 유저 스레드에 마운트되지 않아 실행되지 않는 상태를 나타냅니다. 실행되면 `RUNNING` 상태로 전이됩니다.
- `YIELDING`: 스레드가 대기를 원하는 상태를 나타냅니다. `Thread.yield` 호출이 성공하면 `RUNNABLE` 상태로, 실패하면 `RUNNING` 상태로 유지됩니다.
- `TERMINATED`: 스레드가 종료되었음을 나타냅니다.

가상 스레드가 실행되면 `NEW`에서 `STARTED`로 상태를 전이하고 실행할 작업을 유저 스레드의 작업 큐에 넣습니다.

```java
void start(ThreadContainer container) {
    if (!compareAndSetState(NEW, STARTED)) {
        throw new IllegalThreadStateException("Already started");
    }

    // bind thread to container
    assert threadContainer() == null;
    setThreadContainer(container);

    // start thread
    boolean addedToContainer = false;
    boolean started = false;
    try {
        container.onStart(this);  // may throw
        addedToContainer = true;

        // scoped values may be inherited
        inheritScopedValueBindings(container);

        // submit task to run thread
        submitRunContinuation();
        started = true;
    } finally {
        if (!started) {
            setState(TERMINATED);
            afterTerminate(addedToContainer, /*executed*/false);
        }
    }
}
```

스케줄러는 기본적으로 `ForkJoinPool`을 사용하며 모든 가상 스레드가 이를 공유합니다. `ForkJoinPool`은 하나의 작업 큐를 여러 유저 스레드가 함께 공유하며, 공용 큐에 등록된 작업을 각 스레드가 자신의 내부 큐로 가져가서 처리합니다. 그리고 처리하고 있는 작업이 없는 경우에는 서로의 큐에 접근하여 작업을 가져와 처리합니다. 또한, 하나의 스레드에서 처리하기 어려운 큰 작업은 여러 작은 작업으로 분할하여 처리하고 결과를 합칩니다.

```java
private static final ForkJoinPool DEFAULT_SCHEDULER = createDefaultScheduler();
    
private void submitRunContinuation() {
    try {
        scheduler.execute(runContinuation);
    } catch (RejectedExecutionException ree) {
        submitFailed(ree);
        throw ree;
    }
}
```

`ForkJoinPool`에 등록된 `runContinuation`은 스레드의 상태를 `RUNNING`으로 변경하고 실행할 작업이 등록된 `VThreadContinuation`를 실행합니다. `VThreadContinuation`은 실행할 작업 뿐만 아니라 실행에 필요한 컨텍스트 정보를 함께 저장하고 작업의 영속성을 관리합니다.

```java
private void runContinuation() {
    // the carrier must be a platform thread
    if (Thread.currentThread().isVirtual()) {
        throw new WrongThreadException();
    }

    // set state to RUNNING
    int initialState = state();
    if (initialState == STARTED && compareAndSetState(STARTED, RUNNING)) {
        // first run
    } else if (initialState == RUNNABLE && compareAndSetState(RUNNABLE, RUNNING)) {
        // consume parking permit
        setParkPermit(false);
    } else {
        // not runnable
        return;
    }

    // notify JVMTI before mount
    notifyJvmtiMount(/*hide*/true);

    try {
        cont.run(); // cont: VThreadContinuation
    } finally {
        if (cont.isDone()) {
            afterTerminate();
        } else {
            afterYield();
        }
    }
}
```

실제 작업 실행은 `VirtualThread`에 위임되어 있으며, `run` 메소드를 통해 처리됩니다. `run` 메소드는 플렛폼 스레드를 가상 스레드를 마운트하고 필요한 컨텍스트를 바인딩한 후 작업을 실행합니다. 마운트된 플렛폼 스레드는 `carrier thread`로 불립니다.

```java
private static class VThreadContinuation extends Continuation {
    VThreadContinuation(VirtualThread vthread, Runnable task) {
        super(VTHREAD_SCOPE, wrap(vthread, task));
    }
    @Override
    protected void onPinned(Continuation.Pinned reason) {
        if (TRACE_PINNING_MODE > 0) {
            boolean printAll = (TRACE_PINNING_MODE == 1);
            PinnedThreadPrinter.printStackTrace(System.out, printAll);
        }
    }
    private static Runnable wrap(VirtualThread vthread, Runnable task) {
        return new Runnable() {
            @Hidden
            public void run() {
                vthread.run(task);
            }
        };
    }
}
```

```java
private void run(Runnable task) {
    assert state == RUNNING;

    // first mount
    mount();
    notifyJvmtiStart();

    // emit JFR event if enabled
    if (VirtualThreadStartEvent.isTurnedOn()) {
        var event = new VirtualThreadStartEvent();
        event.javaThreadId = threadId();
        event.commit();
    }

    Object bindings = scopedValueBindings();
    try {
        runWith(bindings, task);
    } catch (Throwable exc) {
        dispatchUncaughtException(exc);
    } finally {
        try {
            // pop any remaining scopes from the stack, this may block
            StackableScope.popAll();

            // emit JFR event if enabled
            if (VirtualThreadEndEvent.isTurnedOn()) {
                var event = new VirtualThreadEndEvent();
                event.javaThreadId = threadId();
                event.commit();
            }

        } finally {
            // last unmount
            notifyJvmtiEnd();
            unmount();

            // final state
            setState(TERMINATED);
        }
    }
}
```

가상 스레드에서 `Thread.yield`나 `Thread.sleep`와 같은 인터럽트가 발생하면 `VirtualThread`가 해당 연산을 실행합니다. 

```java
public static void yield() {
    if (currentThread() instanceof VirtualThread vthread) {
        vthread.tryYield();
    } else {
        yield0();
    }
}
```

`tryYield`는 `yieldContinuation`을 호출하여 스레드를 일시 중단하고 언마운트합니다. 

```java
void tryYield() {
    assert Thread.currentThread() == this;
    setState(YIELDING);
    boolean yielded = false;
    try {
        yielded = yieldContinuation();  // may throw
    } finally {
        assert (Thread.currentThread() == this) && (yielded == (state() == RUNNING));
        if (!yielded) {
            assert state() == YIELDING;
            setState(RUNNING);
        }
    }
}
```

마찬가지로 인터럽트가 발생하여 가상 스레드가 일시 중단되면 `park`가 호출되어 스레드가 언마운트되고 상태가 `PARKING` 또는 `PINNED`으로 전이됩니다.

```java
void park() {
    assert Thread.currentThread() == this;

    // complete immediately if parking permit available or interrupted
    if (getAndSetParkPermit(false) || interrupted)
        return;

    // park the thread
    boolean yielded = false;
    setState(PARKING);
    try {
        yielded = yieldContinuation();  // may throw
    } finally {
        assert (Thread.currentThread() == this) && (yielded == (state() == RUNNING));
        if (!yielded) {
            assert state() == PARKING;
            setState(RUNNING);
        }
    }

    // park on the carrier thread when pinned
    if (!yielded) {
        parkOnCarrierThread(false, 0);
    }
}

private void parkOnCarrierThread(boolean timed, long nanos) {
    assert state() == RUNNING;

    VirtualThreadPinnedEvent event;
    try {
        event = new VirtualThreadPinnedEvent();
        event.begin();
    } catch (OutOfMemoryError e) {
        event = null;
    }

    setState(PINNED);
    try {
        if (!parkPermit) {
            if (!timed) {
                U.park(false, 0);
            } else if (nanos > 0) {
                U.park(false, nanos);
            }
        }
    } finally {
        setState(RUNNING);
    }

    // consume parking permit
    setParkPermit(false);

    if (event != null) {
        try {
            event.commit();
        } catch (OutOfMemoryError e) {
            // ignore
        }
    }
}
```

`U.park`은 플랫폼 스레드를 언마운트하지 않고 고정된 상태에서 중단하며, `yieldContinuation`은 실행 중인 플랫폼 스레드를 언마운트한 후 일시 중단시켜 스레드 풀로 플랫폼 스레드를 반환합니다. `U.park`은 마운트 중이던 스레드를 그대로 점유하므로 `yieldContinuation`가 실패했을 경우만 사용됩니다.

```java
@Deprecated(since="22", forRemoval=true)
@ForceInline
public void park(boolean isAbsolute, long time) {
    theInternalUnsafe.park(isAbsolute, time);
}

private boolean yieldContinuation() {
    // unmount
    notifyJvmtiUnmount(/*hide*/true);
    unmount();
    try {
        return Continuation.yield(VTHREAD_SCOPE);
    } finally {
        // re-mount
        mount();
        notifyJvmtiMount(/*hide*/false);
    }
}
```

```java
@ChangesCurrentThread
@ReservedStackAccess
private void unmount() {
    // set Thread.currentThread() to return the platform thread
    Thread carrier = this.carrierThread;
    carrier.setCurrentThread(carrier);

    // break connection to carrier thread, synchronized with interrupt
    synchronized (interruptLock) {
        setCarrierThread(null);
    }
    carrier.clearInterrupt();

    // notify JVMTI after unmount
    notifyJvmtiUnmount(/*hide*/false);
}

private void setCarrierThread(Thread carrier) {
    // U.putReferenceRelease(this, CARRIER_THREAD, carrier);
    this.carrierThread = carrier;
}
```

`yieldContinuation`가 호출하는 `Continuation.yield`는 JNI(Java Native Interface)를 통해 JVM과 연결된 네이티브 메소드인 `doYield`를 통해 스레드를 일시 중단 시킵니다.

```java
public static boolean yield(ContinuationScope scope) {
    Continuation cont = JLA.getContinuation(currentCarrierThread());
    Continuation c;
    for (c = cont; c != null && c.scope != scope; c = c.parent)
        ;
    if (c == null)
        throw new IllegalStateException("Not in scope " + scope);

    return cont.yield0(scope, null);
}

private boolean yield0(ContinuationScope scope, Continuation child) {
    if (scope != this.scope)
        this.yieldInfo = scope;
    int res = doYield();
    U.storeFence(); // needed to prevent certain transformations by the compiler

    assert scope != this.scope || yieldInfo == null : "scope: " + scope + " this.scope: " + this.scope + " yieldInfo: " + yieldInfo + " res: " + res;
    assert yieldInfo == null || scope == this.scope || yieldInfo instanceof Integer : "scope: " + scope + " this.scope: " + this.scope + " yieldInfo: " + yieldInfo + " res: " + res;

    if (child != null) { // TODO: ugly
        if (res != 0) {
            child.yieldInfo = res;
        } else if (yieldInfo != null) {
            assert yieldInfo instanceof Integer;
            child.yieldInfo = yieldInfo;
        } else {
            child.yieldInfo = res;
        }
        this.yieldInfo = null;
    } else {
        if (res == 0 && yieldInfo != null) {
            res = (Integer)yieldInfo;
        }
        this.yieldInfo = null;

        if (res == 0)
            onContinue();
        else
            onPinned0(res);
    }
    assert yieldInfo == null;

    return res == 0;
}
```

`doYield`은 `freeze`를 호출하여 현재 스택을 힙에 복사하여 저장합니다.

```
Thread-stack layout on freeze/thaw.
See corresponding stack-chunk layout in instanceStackChunkKlass.hpp

            +----------------------------+
            |      .                     |
            |      .                     |
            |      .                     |
            |   carrier frames           |
            |                            |
            |----------------------------|
            |                            |
            |    Continuation.run        |
            |                            |
            |============================|
            |    enterSpecial frame      |
            |  pc                        |
            |  rbp                       |
            |  -----                     |
        ^   |  int argsize               | = ContinuationEntry
        |   |  oopDesc* cont             |
        |   |  oopDesc* chunk            |
        |   |  ContinuationEntry* parent |
        |   |  ...                       |
        |   |============================| <------ JavaThread::_cont_entry = entry->sp()
        |   |  ? alignment word ?        |
        |   |----------------------------| <--\
        |   |                            |    |
        |   |  ? caller stack args ?     |    |   argsize (might not be 2-word aligned) words
Address |   |                            |    |   Caller is still in the chunk.
        |   |----------------------------|    |
        |   |  pc (? return barrier ?)   |    |  This pc contains the return barrier when the bottom-most frame
        |   |  rbp                       |    |  isn't the last one in the continuation.
        |   |                            |    |
        |   |    frame                   |    |
        |   |                            |    |
            +----------------------------|     \__ Continuation frames to be frozen/thawed
            |                            |     /
            |    frame                   |    |
            |                            |    |
            |----------------------------|    |
            |                            |    |
            |    frame                   |    |
            |                            |    |
            |----------------------------| <--/
            |                            |
            |    doYield/safepoint stub  | When preempting forcefully, we could have a safepoint stub
            |                            | instead of a doYield stub
            |============================| <- the sp passed to freeze
            |                            |
            |  Native freeze/thaw frames |
            |      .                     |
            |      .                     |
            |      .                     |
            +----------------------------+
```

```c++
// Entry point to freeze. Transitions are handled manually
// Called from gen_continuation_yield() in sharedRuntime_<cpu>.cpp through Continuation::freeze_entry();
template<typename ConfigT>
static JRT_BLOCK_ENTRY(int, freeze(JavaThread* current, intptr_t* sp))
  assert(sp == current->frame_anchor()->last_Java_sp(), "");

  if (current->raw_cont_fastpath() > current->last_continuation()->entry_sp() || current->raw_cont_fastpath() < sp) {
    current->set_cont_fastpath(nullptr);
  }

  return ConfigT::freeze(current, sp);
JRT_END
```

다시 가상 스레드가 실행되면 `unpark` 메소드가 호출되어 유저 스레드에 마운트되고 컨텍스트가 로딩됩니다. `unpark`는 `submitRunContinuation`를 호출하여 상태를 `RUNNING`으로 변경하고 등록된 작업을 다시 시작합니다. 만약 상태가 `PINNED`이면 이미 스레드가 마운트되어 있으므로 이미 마운트된 스레드를 재개합니다.

```java
void unpark() {
    Thread currentThread = Thread.currentThread();
    if (!getAndSetParkPermit(true) && currentThread != this) {
        int s = state();
        boolean parked = (s == PARKED) || (s == TIMED_PARKED);
        if (parked && compareAndSetState(s, UNPARKED)) {
            if (currentThread instanceof VirtualThread vthread) {
                vthread.switchToCarrierThread();
                try {
                    submitRunContinuation();
                } finally {
                    switchToVirtualThread(vthread);
                }
            } else {
                submitRunContinuation();
            }
        } else if ((s == PINNED) || (s == TIMED_PINNED)) {
            // unpark carrier thread when pinned
            notifyJvmtiDisableSuspend(true);
            try {
                synchronized (carrierThreadAccessLock()) {
                    Thread carrier = carrierThread;
                    if (carrier != null && ((s = state()) == PINNED || s == TIMED_PINNED)) {
                        U.unpark(carrier);
                    }
                }
            } finally {
                notifyJvmtiDisableSuspend(false);
            }
        }
    }
}
```

많은 연산이 가상 스레드를 지원하지만 일부 플렛폼 스레드를 같이 중지시키는 연산을 사용하면 스레드 풀에 있는 플렛폼 스레드 수를 일시적으로 늘여 플렛폼 스레드의 고갈을 방지힙니다. 동기 시스템 콜 외에도 `synchronized`를 사용하거나 네이티브 함수를 호출하면 플렛폼 스레드가 중단되게 됩니다. 그래서 `synchronized`를 많이 이용하던 `Spring`이나 `MySQL` 같은 라이브러리들은 대신 `ReentrantLock`을 사용하도록 마이그레이션이 이루어지고 있습니다.

## 참고 문서
- [openjdk](https://github.com/openjdk/jdk)
- [JEP 444: Virtual Threads](https://openjdk.org/jeps/444)
- [Java의 미래, Virtual Thread](https://techblog.woowahan.com/15398/)
- [Virtual Thread의 기본 개념 이해하기](https://d2.naver.com/helloworld/1203723)
- [Green Threads vs Non Green Threads](https://stackoverflow.com/questions/5713142/green-threads-vs-non-green-threads)
