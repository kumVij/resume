Here are all 20 answered with examples.

**1. Why is `String` immutable in Java?**
Once created, a `String`'s internal character array can never be changed — any operation that seems to modify a string actually creates a new object.
```java
String s = "hello";
s.concat(" world"); // creates a new String, doesn't change s
System.out.println(s); // still "hello"
```
Reasons for this design:
- **String pool / caching**: literals are interned and reused; if strings were mutable, changing one reference would corrupt every other variable pointing to the same pooled value.
- **Thread safety**: immutable objects are inherently safe to share across threads with no synchronization.
- **Security**: strings are used for things like file paths, network URLs, class names, and DB credentials — if mutable, a reference could be changed after a security check but before it's used (a classic TOCTOU vulnerability).
- **Hashcode caching**: since the value never changes, the hashCode can be computed once and cached, making `String` a very efficient `HashMap` key.

**2. How does `HashMap` work internally?**
It's backed by an array of "buckets." When you call `put(key, value)`, Java computes `key.hashCode()`, runs it through an internal spreading function, then reduces it via `hash & (capacity - 1)` to pick a bucket index. If the bucket is empty, the entry goes straight in. If it already has entries (a collision), it's appended to a linked list there (or inserted into a red-black tree if that bucket has more than 8 entries). `get(key)` does the same hash calculation, then walks that bucket's list/tree comparing keys with `equals()` until it finds a match.
```java
Map<String, Integer> map = new HashMap<>();
map.put("apple", 1); // hash("apple") -> bucket index -> stored there
```

**3. What happens when two keys have the same hashcode?**
That's a **collision**. Both keys land in the same bucket. `HashMap` doesn't overwrite or lose data — it stores both entries as a chain (linked list, or tree once the chain exceeds 8 entries) within that bucket. On lookup, Java compares hashCodes first (fast filter), then falls back to `equals()` to find the exact matching key within the bucket.
```java
// Two different keys CAN share a hashCode without being equal —
// they'll both live in the same bucket, distinguished by equals()
```
Note this is different from two keys being `.equals()` — that would mean it's the *same* key, and `put()` would just overwrite the value.

**4. Why must `equals()` and `hashCode()` follow a contract?**
The rule: if two objects are equal via `equals()`, they **must** return the same `hashCode()`. (The reverse isn't required — different objects can share a hashCode, which is just a collision.)

Why it matters: hash-based collections (`HashMap`, `HashSet`) use `hashCode()` to pick a bucket, then `equals()` to confirm the match within that bucket. If you override `equals()` but not `hashCode()` (or implement them inconsistently), two "equal" objects could compute different hashCodes and land in *different* buckets — meaning a `HashMap` would treat them as distinct keys, and `map.get(key)` could fail to find a value you clearly stored under an "equal" key.
```java
class Point {
    int x, y;
    @Override
    public boolean equals(Object o) {
        if (!(o instanceof Point p)) return false;
        return x == p.x && y == p.y;
    }
    @Override
    public int hashCode() {
        return Objects.hash(x, y); // MUST be consistent with equals()
    }
}
```

**5. `ArrayList` vs `LinkedList` — when would you use each?**
- `ArrayList`: backed by a dynamic array — O(1) random access (`get(i)`), but O(n) insert/delete in the middle (shifting elements).
- `LinkedList`: backed by a doubly-linked list — O(n) random access, but O(1) insert/delete at a known node (no shifting), and it also implements `Deque` (usable as a stack/queue).

**Use `ArrayList`** when you mostly read/iterate and rarely insert/delete in the middle — this covers the vast majority of real-world use cases.
**Use `LinkedList`** when you're frequently adding/removing from the head or tail (e.g., implementing a queue or an undo stack) and rarely need random access.
```java
Deque<Integer> stack = new LinkedList<>();
stack.push(1); stack.push(2); // O(1) at head
```
In practice, `ArrayDeque` often beats `LinkedList` even for queue/stack use cases due to better cache locality — `LinkedList` is used less often than people assume.

**6. `HashMap` vs `ConcurrentHashMap` — what's the real difference?**
`HashMap` is not thread-safe at all — concurrent writes can corrupt its internal structure (even cause infinite loops during resize in old JVM versions) or silently lose data.

`ConcurrentHashMap` is designed for concurrent access:
- Reads are lock-free (`volatile` reads).
- Writes to an empty bucket use lock-free CAS.
- Writes to a colliding bucket lock **only that bucket's head node**, not the whole map — so threads writing to different buckets never block each other.
- Its iterator is weakly consistent (won't throw `ConcurrentModificationException`, but might not reflect very recent updates).

```java
Map<String, Integer> map = new ConcurrentHashMap<>();
// safe to call from multiple threads without external synchronization
```
Compare this to `Collections.synchronizedMap(new HashMap<>())`, which is thread-safe but uses one **global lock** — every operation blocks every other thread, effectively serializing all access. `ConcurrentHashMap` is almost always the better choice.

**7. What happens if you modify a collection while iterating over it?**
Most standard collections (`ArrayList`, `HashMap`, `HashSet`) are **fail-fast**: they track an internal `modCount`. If you structurally modify the collection (add/remove) during iteration through any means other than the iterator's own `remove()`, the next call to `iterator.next()` throws `ConcurrentModificationException`.
```java
List<String> list = new ArrayList<>(List.of("a", "b", "c"));
for (String s : list) {
    if (s.equals("b")) list.remove(s); // throws ConcurrentModificationException
}
```
Correct approach — use the iterator's own remove:
```java
Iterator<String> it = list.iterator();
while (it.hasNext()) {
    if (it.next().equals("b")) it.remove(); // safe
}
```
Or use `removeIf()`:
```java
list.removeIf(s -> s.equals("b"));
```
Note: `ConcurrentHashMap` and `CopyOnWriteArrayList` are fail-safe and won't throw this — they iterate over a snapshot instead.

**8. `synchronized` vs `volatile` — when would you use each?**
- `volatile` guarantees **visibility** (every thread sees the latest write immediately, bypassing CPU caches) and prevents instruction reordering around that variable — but it does **not** provide atomicity for compound operations.
```java
private volatile boolean running = true; // simple flag, safe with volatile
public void stop() { running = false; }
public void run() { while (running) { ... } }
```
- `synchronized` provides both visibility **and** mutual exclusion (atomicity) — needed when you're doing a read-modify-write operation that must be atomic.
```java
private int count = 0;
public synchronized void increment() { count++; } // NOT safe with just volatile — read+write isn't atomic
```
**Rule of thumb**: use `volatile` for simple flags/status variables read by many threads and written by one. Use `synchronized` (or `AtomicInteger`, `AtomicLong`, etc.) when multiple threads need to perform compound read-modify-write operations.

**9. How would you identify a deadlock in a Java application?**
1. **Symptoms**: the application hangs — certain threads stop making progress, CPU usage may drop even though the app is "running," requests time out.
2. **Take a thread dump**: `jstack <pid>` or `kill -3 <pid>` (prints to stdout). Modern JVMs actually detect deadlocks automatically and print a clear section like:
```
Found one Java-level deadlock:
=============================
"Thread-1":
  waiting to lock monitor 0x... (object B),
  which is held by "Thread-2"
"Thread-2":
  waiting to lock monitor 0x... (object A),
  which is held by "Thread-1"
```
3. Tools like **VisualVM** or **JConsole** also have a built-in "Detect Deadlock" button that parses thread state for you.
4. Fix by ensuring **consistent lock ordering** across all threads, reducing lock scope, or switching to `tryLock()` with a timeout so threads back off instead of blocking forever.

**10. `Runnable` vs `Callable` — when would you choose each?**
- `Runnable.run()` takes no arguments, returns `void`, and cannot throw checked exceptions.
- `Callable<V>.call()` returns a value of type `V` and can throw a checked exception.

**Use `Runnable`** for fire-and-forget tasks where you don't need a result — e.g., logging, sending a notification.
```java
Runnable task = () -> System.out.println("Task done");
executor.execute(task);
```
**Use `Callable`** when you need the outcome of the computation or need to propagate a checked exception cleanly — e.g., fetching data from a DB or an API call.
```java
Callable<Integer> task = () -> fetchOrderCount(); // might throw SQLException
Future<Integer> future = executor.submit(task);
int count = future.get();
```

**11. Why would you use `ExecutorService` instead of creating threads manually?**
Manually creating a `new Thread()` for every task is expensive (OS thread creation overhead) and gives you no control over how many threads run concurrently — under heavy load this can exhaust system resources and crash the app.

`ExecutorService` gives you a managed **pool** of reusable worker threads:
```java
ExecutorService executor = Executors.newFixedThreadPool(10);
executor.submit(() -> processOrder(order));
executor.shutdown();
```
Benefits: threads are reused (no per-task creation cost), the number of concurrent threads is bounded (prevents resource exhaustion), it provides task queuing when all threads are busy, and it offers clean lifecycle management (`shutdown()`, `awaitTermination()`) instead of manually tracking every `Thread` object.

**12. What problem does `CompletableFuture` solve?**
A plain `Future.get()` **blocks** the calling thread until the result is ready — you can't easily chain more work after it completes, combine multiple async results, or handle errors without writing clunky blocking code.

`CompletableFuture` (Java 8+) enables **non-blocking, composable asynchronous pipelines**:
```java
CompletableFuture.supplyAsync(() -> fetchUser(id))
    .thenApply(user -> user.getName())
    .thenAccept(name -> System.out.println("User: " + name))
    .exceptionally(ex -> { System.out.println("Failed: " + ex.getMessage()); return null; });
```
You can also combine independent async calls:
```java
CompletableFuture<Integer> priceFuture = CompletableFuture.supplyAsync(this::getPrice);
CompletableFuture<Integer> taxFuture = CompletableFuture.supplyAsync(this::getTax);
priceFuture.thenCombine(taxFuture, (price, tax) -> price + tax)
           .thenAccept(total -> System.out.println("Total: " + total));
```
This is especially useful for orchestrating multiple microservice calls in parallel without blocking threads.

**13. How can a Java application have a memory leak despite having Garbage Collection?**
GC only reclaims objects that are **unreachable**. A "leak" in Java means objects are still technically reachable (so GC can't touch them) but are no longer actually needed by the application — they're just being held onto unintentionally. Common causes:
- **Static collections** that keep growing (e.g., a `static Map` cache with no eviction/TTL) — the class loader keeps them alive for the app's whole lifetime.
- **`ThreadLocal`s not cleared** in a thread-pool environment — since pool threads are long-lived and reused, an unclear `ThreadLocal` value lingers on that thread indefinitely.
- **Unclosed resources** (streams, DB connections) that indirectly retain large object graphs.
- **Listener/callback registration without deregistration** — an object that registers itself as a listener but never unregisters stays reachable via the listener list forever.
```java
static List<byte[]> cache = new ArrayList<>(); // never cleared -> classic leak
public void handle(byte[] data) {
    cache.add(data); // grows forever, never removed
}
```
Diagnosis: watch the Old Gen heap trend in an APM tool — a "rising baseline" even after Full GC signals a leak. Confirm with a heap dump analyzed in Eclipse MAT, tracing the "Path to GC Roots" of the bloated objects.

**14. Heap vs Stack — what is stored where?**
- **Heap**: shared across all threads. Stores all objects created with `new` (instance fields, arrays, etc.) and is managed by the Garbage Collector. Divided into Young Generation (Eden + Survivor spaces) and Old Generation.
- **Stack**: each thread has its own stack. Stores method call frames — local (primitive) variables, method parameters, and **references** to heap objects (not the objects themselves). When a method returns, its stack frame is popped and instantly reclaimed — no GC involved for stack memory.
```java
void method() {
    int x = 5;              // primitive -> on the stack
    Person p = new Person(); // reference 'p' on the stack, actual Person object on the heap
}
```
A `StackOverflowError` happens when the stack runs out of space (e.g., unbounded recursion); an `OutOfMemoryError` happens when the heap can't allocate more objects.

**15. What happens during a Full GC, and why can it affect API latency?**
A Full GC collects the **entire heap** — both Young and Old Generation — reclaiming objects that survived multiple minor GCs and became "old." Depending on the collector (e.g., older CMS, or a stressed G1), a Full GC can require a **"stop-the-world" pause**, where all application threads are frozen while the GC does its work.

For a live API, this means: every in-flight request is paused mid-execution, response times spike (sometimes to seconds), and if the pause exceeds a load balancer's health-check timeout, the instance can be pulled out of rotation, causing cascading failures. Full GCs on large heaps (tens of GB) are especially dangerous, since pause time roughly scales with heap size for older collectors.

Mitigation: use a low-pause collector (G1, ZGC, or Shenandoah), size the heap appropriately, avoid triggering Full GC frequently (usually a sign of a memory leak or undersized heap), and monitor GC logs to catch this pattern before it hits production users.

**16. What happens if an exception is thrown inside a thread?**
If an exception is thrown inside `run()` and isn't caught, the thread simply **terminates** — it doesn't crash the JVM or other threads. By default, the exception's stack trace is printed via the thread's **uncaught exception handler** (default implementation just prints to stderr), but the calling thread (e.g., `main`) is never notified and continues running normally, unaware anything went wrong.
```java
Thread t = new Thread(() -> {
    throw new RuntimeException("Oops");
});
t.start();
// main thread continues unaffected; exception just gets printed
```
You can attach a custom handler to catch and log this properly:
```java
t.setUncaughtExceptionHandler((thread, ex) -> System.out.println("Thread " + thread.getName() + " failed: " + ex));
```
This is one reason `Callable` + `Future` is often preferred over raw `Runnable` — the exception is captured and re-thrown (wrapped in `ExecutionException`) when you call `future.get()`, so it's not silently swallowed.

**17. Why can `HashSet` detect duplicate objects incorrectly if `equals()` and `hashCode()` are implemented badly?**
`HashSet` is backed internally by a `HashMap` — it checks for duplicates by computing an object's `hashCode()` to find the right bucket, then uses `equals()` to compare against existing entries in that bucket.

If `hashCode()` isn't overridden (or is inconsistent with `equals()`), two objects that are logically "equal" per your `equals()` might return **different hashCodes**, so they land in different buckets and `HashSet` never even compares them with `equals()` — it treats them as distinct, letting a "duplicate" slip through.
```java
class Employee {
    String id;
    @Override
    public boolean equals(Object o) {
        return o instanceof Employee e && id.equals(e.id);
    }
    // forgot to override hashCode()! Uses default Object.hashCode() (identity-based)
}

Set<Employee> set = new HashSet<>();
set.add(new Employee("E1"));
set.add(new Employee("E1")); // different object, same "id" — but hashCode differs, so BOTH get added!
System.out.println(set.size()); // prints 2, not 1 — bug!
```
The fix is always to override both together, and to base them on the same fields.

**18. When would you use `Optional`, and when should you avoid it?**
**Use it** as a **method return type**, to explicitly signal to callers that a value might be absent, replacing ambiguous `null` returns:
```java
public Optional<User> findUserById(String id) {
    return Optional.ofNullable(userRepository.get(id));
}

// caller
findUserById("123").ifPresentOrElse(
    user -> System.out.println(user.getName()),
    () -> System.out.println("User not found")
);
```
**Avoid it**:
- As a **class field** — it's not `Serializable`, and it adds indirection/overhead to something that should just be a nullable reference (or better, a properly-initialized default).
- As a **method parameter** — forces every caller to wrap arguments in `Optional.ofNullable()`, adding unnecessary boilerplate. Use method overloading or plain null checks instead.
- Don't call `.get()` without checking `.isPresent()` first — that just reintroduces the `NullPointerException` risk `Optional` was meant to prevent, via `NoSuchElementException` instead.

**19. Streams vs traditional loops — when would you prefer one over the other?**
**Use Streams** when the operation is a clear, declarative pipeline of transformations — filter, map, sort, collect — since it's more readable and expresses *what* you want rather than *how* to do it step by step:
```java
List<String> names = employees.stream()
    .filter(e -> e.getSalary() > 50000)
    .map(Employee::getName)
    .sorted()
    .collect(Collectors.toList());
```
**Use traditional loops** when:
- You need **early exit** with complex control flow (`break`/`continue` with multiple conditions) — streams can do this but it's often more awkward (`takeWhile`/`anyMatch` cover some cases, but not all).
- You're doing something with **side effects** on external state — streams (especially parallel ones) discourage mutating shared variables, and doing so is a common source of bugs.
- **Performance-critical hot paths** — a simple `for` loop has less overhead than the lambda/functional-interface machinery of streams, which matters in tight, high-throughput loops.
- The logic involves multiple, tangled steps that are genuinely clearer as imperative code than as a chained pipeline.

**20. Your Java API suddenly becomes slow in production. What JVM-level metrics would you check first?**
1. **GC behavior** — check GC frequency and pause times (via GC logs or an APM tool). Frequent Full GCs or long stop-the-world pauses are a very common cause of sudden latency spikes; check whether Old Gen has a "rising baseline" (possible leak) versus a healthy sawtooth pattern.
2. **Heap usage** — is the heap near `-Xmx`? Is it undersized for current load?
3. **Thread state** — take a thread dump (`jstack`) to check for **deadlocks**, thread pool exhaustion (all worker threads blocked/busy), or excessive threads waiting on a lock (contention).
4. **CPU usage** — high CPU could mean excessive GC activity, a runaway loop, or genuine load increase; low CPU with high latency often points to threads blocked on I/O or locks rather than compute.
5. **Connection pools** — DB or HTTP client connection pool exhaustion is a very common "mystery slowdown" cause outside the JVM's own memory/CPU metrics but still visible as thread blocking.
6. **Class loading / JIT warm-up** — relevant right after a deploy: cold JIT compilation can cause temporary slowness until hot paths are optimized.

In practice, the fastest triage is usually: check GC logs first (rules out memory pressure), then take a thread dump if GC looks healthy (reveals blocking/deadlock/pool exhaustion issues).
