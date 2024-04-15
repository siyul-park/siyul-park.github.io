---
title: "다시 돌아온 가상 스레드"
date: 2024-04-26 12:00:00 +0900
categories: "system"
tags: ["system", "java"]
---

JDK 21에서 경량 스레드가 공식 기능으로 추가되었습니다. 이전에는 OS 스레드를 직접 생성하고 사용자 스레드에 매핑하는 플랫폼 스레드 방식이 사용되었습니다.

보통 서버 애플리케이션에서는 독립적인 요청을 처리하기 위해 개별적인 스레드를 할당합니다. 이는 애플리케이션의 동시성 단위를 플랫폼의 동시성 단위와 일치시키므로 이해하기 쉽고 분석하기 쉽습니다.

그러나 이 방식은 요청 처리량이 증가함에 따라 스레드 수도 증가해야 합니다. 하지만 OS 스레드는 비용이 많이 들기 때문에 너무 많이 생성할 수 없어 요청별 스레드 스타일에 적합하지 않습니다. CPU나 네트워크 연결과 같은 다른 리소스가 소진되기 전에 스레드 수가 제한되는 경우가 많으며, 스레드 수가 너무 많아지면 스레드 간에 자원을 얻기 위한 경쟁이 심해져 과도한 컨텍스트 스위칭이 발생할 수 있습니다.

이러한 제약으로 하드웨어를 최대한 활용하기 위해 스레드 풀을 사용하기 시작했습니다. 스레드 풀은 하나의 스레드에서 처음부터 끝까지 요청을 처리하는 대신, 다른 I/O 작업이 완료될 때까지 기다릴 때 해당 스레드를 풀로 반환하여 다른 요청을 처리할 수 있도록 합니다. 이렇게 함으로써 많은 수의 동시 작업을 수행할 수 있지만, 계산을 수행할 때에만 여러 서브 루틴들이 스레드를 사용하여 많은 수의 스레드를 소비하지 않아도 됩니다. 그러나 이에는 비동기 프로그래밍 스타일이 필요합니다.

비동기식 스타일은 콜백, 반응형 같은 기존과 다른 방식이 필요합니다. 스택 추적이 가능한 컨텍스트를 제공하지 않으며, 디버거를 통해 각 단계를 따라가기 어렵고, 작업 비용을 호출자와 연결할 수 없어 프로파일링에 어려움이 있습니다.

이에 따라 OS 스레드보다 저렴한 가상 스레드가 등장하였습니다. 이 방식은 스레드 풀을 사용하는 것과 유사하지만, 플랫폼에서 지원하는 기능과 긴밀하게 통합됩니다. 

사실 이러한 가상 스레드는 JVM의 역사에서 처음 등장한 것은 아니였습니다. JVM 1.3 이전에는 일부 OS에서 Green Thread가 이용되기도 했습니다. Green Thread는 플렛폼 스레드를 만들기 어려운 상황에서 주로 사용이 되었으나, 멀티 코어 프로세스가 활성화되며 점차 사라지게 됩니다.

Green Thread 는 Java 21에서 안정회된 가상 스레드와 달리 단 하나의 플렛폼 스레드에 여러개의 유저 스레드가 할당되었습니다. 프로세스의 코어가 늘어나도 플렛폼 스레드 수는 확장이 되지 않았으며, 가상 스레드와 다르게 유저 스레드가 정지하면 플렛폼 스레드도 정지되었습니다.

새롭게 나온 가상 스레드는 총 9가지 상태를 오가며 유저 스레드와 마운트되어 작업을 처리합니다. 하나의 유저 스레드는 여러 가상 스레드를 처리할 수 있습니다.

- `NEW`: 스레드가 생성되었지만 아직 시작되지 않은 초기 상태입니다. `Thread.start` 메서드를 호출하여 `STARTED` 상태로 전이됩니다.
- `STARTED`: 스레드가 시작되었지만 아직 실행되지 않은 상태를 나타냅니다. 실행이 실패하면 `TERMINATED` 상태로 전이되고, 실행이 성공하면 `RUNNING` 상태로 전이됩니다.
- `RUNNING`: 스레드가 실행되는 상태를 나타냅니다. 유저 스레드를 언마운트하려면 `PARKING` 상태로 전이됩니다. `Thread.yield` 메서드를 호출하여 `YIELDING` 상태로 전이될 수 있습니다.
- `PARKING`: 언마운트 중인 상태를 나타냅니다. `Thread.yield` 호출이 성공하면 `PARKED` 상태로, 실패하면 `PINNED` 상태로 전이됩니다.
- `PARKED`: 언마운트된 상태를 나타냅니다. `unpark` 메서드 호출이 발생하거나 인터럽트가 발생하면 `RUNNABLE` 상태로 전이됩니다.
- `PINNED`: 마운트된 상태이지만 실행되지 않는 상태를 나타냅니다. `unpark` 메서드 호출이 발생하거나 인터럽트가 발생하면 `RUNNABLE` 상태로 전이됩니다.
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

스케줄러는 기본적으로 `ForkJoinPool`을 사용하며 모든 가상 스레드가 이를 공유합니다. `ForkJoinPool`은 하나의 작업 큐를 여러 유저 스레드가 함께 공유하며, 공용 큐에 등록된 작업을 각 스레드가 자신의 내부 큐로 가져가서 처리합니다. 그리고 처리하고 있는 작업이 없는 경우에는 서로의 큐에 접근하여 작업을 가져와 처리합니다. 또한, 하나의 스레드에서 처리하기 어려운 큰 작업은 여러 작은 작업으로 분할하여 처리하고 결과를 합칩니다. 이를 통해 스레드들을 최대한 효율적으로 사용하게 됩니다.

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
        cont.run();
    } finally {
        if (cont.isDone()) {
            afterTerminate();
        } else {
            afterYield();
        }
    }
}
```

실제 작업 실행은 `VirtualThread`에 위임되어 있으며, `run` 메서드를 통해 처리됩니다. `run` 메서드는 플렛폼 스레드를 가상 스레드를 마운트하고 필요한 컨텍스트를 바인딩한 후 작업을 실행합니다. 마운트된 플렛폼 스레드는 `carrier thread`로 불립니다.

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

가상 스레드가 일시 중단되면 `park`가 호출되어 스레드가 언마운트되고 상태가 `PARKING` 또는 `PINNED`으로 전이됩니다.

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

`Continuation.yield`는 스레드를 일시 중단 시키며 JNI(Java Native Interface)를 통해 JVM과 연결된 네이티브 메서드인 `doYield`를 호출하게 됩니다.

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

다시 가상 스레드가 실행되면 `unpark` 메서드가 호출되어 유저 스레드에 마운트되고 컨텍스트가 로딩됩니다. 그리고 `submitRunContinuation`를 호출하여 상태를 `RUNNING`으로 변경하고 등록된 작업을 다시 시작합니다. 만약 상태가 `PINNED`이면 이미 스레드가 마운트되어 있으므로 스레드를 재개합니다.

```java
void unpark() {
    switch (state()) {
        case PARKING:
            // attempt to transition to RUNNABLE
            if (compareAndSetState(PARKING, RUNNABLE)) {
                // completed
            } else {
                // already completed
                assert state() == RUNNABLE;
            }
            break;

        case PINNED:
            // attempt to transition to RUNNING
            if (compareAndSetState(PINNED, RUNNING)) {
                // completed
            } else {
                // already completed
                assert state() == RUNNING;
            }
            break;

        default:
            throw new IllegalStateException("Not parked");
    }
}
```

많은 동기 연산은 언마운트를 지원하지만 일부 작업은 가상 스레드와 플렛폼 스레드를 같이 중지시킵니다. 이 경우에 일시적으로 스레드 풀에 있는 플렛폼 스레드 수를 일시적으로 늘여 플렛폼 스레드의 고길을 방지힙니다.

시스템 콜 외에도 `synchronized`를 사용하거나 네이티브 함수를 호출하면 플렛폼 스레드가 중단되게 됩니다.그래서 `synchronized`를 많이 이용하던 `Spring`이나 `MySQL` 같은 라이브러리들은 대신 `ReentrantLock`을 사용하도록 마이그레이션이 이루어지고 있습니다.

## 참고 문서
- [openjdk](https://github.com/openjdk/jdk)
- [JEP 444: Virtual Threads](https://openjdk.org/jeps/444)
- [Java의 미래, Virtual Thread](https://techblog.woowahan.com/15398/)
- [Virtual Thread의 기본 개념 이해하기](https://d2.naver.com/helloworld/1203723)
- [Green Threads vs Non Green Threads](https://stackoverflow.com/questions/5713142/green-threads-vs-non-green-threads)
