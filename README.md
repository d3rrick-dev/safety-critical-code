### Practical examples using java
[Based off Rules for Developing Safety Critical Code](https://spinroot.com/gerard/pdf/P10.pdf)


### Rule 1: Avoid Complex Flow (No Recursion)
In a safety-critical system, you cannot risk a StackOverflowError. Every path must be predictable.

Control flow must be:
 * Explicit
 * Acyclic
 * Predictable
 * Statistically analyzable

Recursion introduces:
 * Unbounded stack growth
 * Hard-to-prove termination
 * Hidden memory consumption


 Using recursion to find a file in a directory.
 Using a simple for or while loop with an iterative approach.

```java
// ex-1
public void processData(int[] sensorReadings) {
    for (int i = 0; i < sensorReadings.length; i++) {
        // Simple, predictable flow
        doWork(sensorReadings[i]);
    }
}

// ex-2
// dangerous
public void scan(File dir) {
    for (File f : dir.listFiles()) {
        if (f.isDirectory()) {
            scan(f); // unbounded recursion depth
        }
    }
}

//  safer
public void scan(File root) {
    Deque<File> stack = new ArrayDeque<>();
    stack.push(root);

    int maxDepth = 10_000; // Explicit bound
    int processed = 0;

    while (!stack.isEmpty() && processed < maxDepth) {
        File current = stack.pop();
        processed++;

        File[] files = current.listFiles();
        if (files != null) {
            for (File f : files) {
                if (f.isDirectory()) {
                    stack.push(f);
                }
            }
        }
    }

    if (processed >= maxDepth) {
        throw new IllegalStateException("Traversal exceeded safe bounds");
    }
}
```

### Rule 2: All Loops Must Have Fixed Bounds
Every loop must have a statically provable upper bound.

A loop that "runs until done" might run forever if a sensor breaks. You must provide a "circuit breaker."

No:
 * while(true)
 * while(condition from external system)
 * open-ended retries

Because:
 * External systems fail (IO)
 * Sensors freeze
 * Network partitions happen


 while (sensor.isActive()) { ... }
 Setting a maximum "sane" limit for iterations.

```java
//ex-1
// dangerous
while (sensor.isActive()) {
    process();
}

// safer
public void calibrate() {
    final int MAX_ITERATIONS = 1000;
    final long MAX_TIME_MS = 2000;

    int iterations = 0;
    long start = System.currentTimeMillis();

    while (sensor.needsCalibration()
            && iterations < MAX_ITERATIONS
            && System.currentTimeMillis() - start < MAX_TIME_MS) {

        applyAdjustment();
        iterations++;
    }

    if (iterations >= MAX_ITERATIONS) {
        throw new IllegalStateException("Calibration iteration limit exceeded");
    }
}


// ex-2
// dangerous
while (true) {
    processQueue();
}

// safer - always boud retries and timeouts etc.
for (int i = 0; i < MAX_RETRIES; i++) {
    if (tryProcess()) {
        break;
    }
}

// ex-3
// dangerous
while (true) {
    kafkaTemplate.send(...);
}

// safer
for (int attempt = 1; attempt <= MAX_RETRIES; attempt++) {
    try {
        kafkaTemplate.send(...).get(2, TimeUnit.SECONDS);
        return;
    } catch (Exception ex) {
        log.warn("Attempt {} failed", attempt);
    }
}

throw new IllegalStateException("Kafka publish failed after ${MAX_RETRIES} retries");
```

### Rule 3: No Dynamic Memory After Initialization
Java always allocates objects.

Do not allocate memory in uncontrolled or unbounded execution paths.

Because:
 * GC pauses can freeze your system
 * Memory growth can crash the JVM
 * Latency becomes unpredictable

 Creating a `new Point()` 60 times a second in a game loop.
 Object Pooling. Pre-allocate objects at startup and reuse them.

```java
// ex-1
// Pre-allocate at the start
private final List<DataPacket> packetPool = new ArrayList<>(100);

public void onMessageReceived(byte[] raw) {
    // Reuse an existing object instead of 'new DataPacket()'
    DataPacket packet = packetPool.get(nextAvailableIndex);
    packet.update(raw);
}

// ex-2
// keeps growing forever
List<String> cache = new ArrayList<>();



Cache<String, User> cache =
    Caffeine.newBuilder()
        .maximumSize(10_000)
        .build();

// ex-3
// Object pool
private final BlockingQueue<DataPacket> pool = new ArrayBlockingQueue<>(100);

public DataPacket acquire() {
    DataPacket packet = pool.poll();
    if (packet == null) {
        throw new IllegalStateException("Pool exhausted");
    }
    return packet;
}
```