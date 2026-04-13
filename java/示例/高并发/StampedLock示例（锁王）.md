# 1. StampedLock概述

### 1.1 什么是StampedLock？

StampedLock是一个多模式的同步控制组件，支持写锁、悲观读锁和乐观读三种模式。与传统的ReadWriteLock不同，它通过"戳"（stamp）的概念来标识锁的状态，并提供了乐观读的机制，在特定场景下能够大幅提升系统性能。

### 1.2 核心特性

- 支持三种模式：写锁、悲观读锁、乐观读

- 基于"戳"（stamp）的状态控制

- 不支持重入

- 不支持Condition条件

- 支持读写锁的升级和降级

## 2. StampedLock的三种模式详解

### 2.1 写锁（Write Lock）

写锁是一个排他锁，当一个线程获取写锁时，其他线程无法获取任何类型的锁。

```java
StampedLock lock = new StampedLock();


long stamp = lock.writeLock(); // 获取写锁
try {
    // 写入共享变量
} finally {
    lock.unlockWrite(stamp); // 释放写锁
}
```

### 2.2 悲观读锁（Pessimistic Read Lock）

悲观读锁类似于ReadWriteLock中的读锁，允许多个线程同时获取读锁，但与写锁互斥。

```java
long stamp = lock.readLock(); // 获取悲观读锁
try {
    // 读取共享变量
} finally {
    lock.unlockRead(stamp); // 释放读锁
}
```

### 2.3 乐观读（Optimistic Read）

乐观读是StampedLock最具特色的模式，它不是一个真正的锁，而是一种基于版本号的无锁机制。

```java
long stamp = lock.tryOptimisticRead(); // 获取乐观读戳记
// 读取共享变量
if (!lock.validate(stamp)) { // 验证戳记是否有效
    // 升级为悲观读锁
    stamp = lock.readLock();
    try {
        // 重新读取共享变量
    } finally {
        lock.unlockRead(stamp);
    }
}
```

## 3. 性能优势

### 3.1 与ReadWriteLock的对比

- 读多写少场景：性能提升约10倍

- 读写均衡场景：性能提升约1倍

- 写多读少场景：性能相当

### 3.2 性能优势的原因

1. 乐观读机制避免了不必要的加锁操作

1. 底层实现使用了更多的CPU指令级别的优化

1. 采用了无锁算法

1. 内部实现了自旋机制

### 4.1 基本使用示例

```java
public class Point {
    private double x, y;
    private final StampedLock sl = new StampedLock();

    // 写入方法
    void move(double deltaX, double deltaY) {
        long stamp = sl.writeLock();
        try {
            x += deltaX;
            y += deltaY;
        } finally {
            sl.unlockWrite(stamp);
        }
    }

    // 乐观读方法
    double distanceFromOrigin() {
        long stamp = sl.tryOptimisticRead();
        double currentX = x, currentY = y;
        if (!sl.validate(stamp)) {
            stamp = sl.readLock();
            try {
                currentX = x;
                currentY = y;
            } finally {
                sl.unlockRead(stamp);
            }
        }
        return Math.sqrt(currentX * currentX + currentY * currentY);
    }
}
```

### 4.2 锁升级示例

```java
public class DataContainer {
    private final StampedLock lock = new StampedLock();
    private double data;

    public void transformData() {
        long stamp = lock.tryOptimisticRead();
        double currentData = data;
        // 检查是否需要更新
        if (needsUpdate(currentData)) {
            // 升级为写锁
            long writeStamp = lock.tryConvertToWriteLock(stamp);
            if (writeStamp != 0L) {
                try {
                    data = computeNewValue(currentData);
                } finally {
                    lock.unlockWrite(writeStamp);
                }
            } else {
                // 升级失败，回退到普通的写锁获取
                stamp = lock.writeLock();
                try {
                    data = computeNewValue(data);
                } finally {
                    lock.unlockWrite(stamp);
                }
            }
        }
    }

    private boolean needsUpdate(double currentData) {
        // 判断是否需要更新
        return false;
    }

    private double computeNewValue(double currentData) {
        // 计算新值
        return 0.0;
    }
}
```

## 5. 使用注意事项

### 5.1 不支持重入

StampedLock不支持重入特性，同一个线程多次获取锁会导致死锁。

### 5.2 中断处理

在使用悲观读锁和写锁时，需要注意处理中断情况：

```java
try {
    long stamp = lock.readLockInterruptibly();
    try {
        // 处理数据
    } finally {
        lock.unlockRead(stamp);
    }
} catch (InterruptedException e) {
    // 处理中断
}
```

### 5.3 乐观读的使用建议

- 适用于读多写少的场景

- 读取的共享变量数量较少

- 读取操作的执行时间较短

- 需要做好版本验证和失败后的补偿措施

### 5.4** 缺点与限制**

- **不可重入**：StampedLock的锁不是可重入的，用习惯了ReentrantLock的同学别搞懵了。

- **代码复杂度高**：乐观读锁用不好就容易出问题，尤其是版本号校验没做好，直接踩坑。

- **写线程饥饿**：读多写少场景下，写线程可能等很久，饿到脱发。

### **5.5 实战与优化**

StampedLock的核心在于

1. **读多写少，用乐观读锁**

```java
StampedLock lock = new StampedLock();
long stamp = lock.tryOptimisticRead();
try {
    if (lock.validate(stamp)) {
     // 乐观读成功，读数据
    } else {
        // 回退到悲观读
        stamp = lock.readLock();
         try {
             // 读数据
         } finally {
            lock.unlockRead(stamp);
         }
    }
} finally {
    if (lock.isReadLockHeld()) {
        lock.unlockRead(stamp);
    }
}
```

**关键点：一定要校验**

1. **写操作用写锁，避免写饥饿**

```java
long stamp = lock.writeLock();
try {
    // 写数据
} finally {
    lock.unlockWrite(stamp);
}
```

**写锁确保线程安全，但在读多的场景下可能性能受限。**

1. **高并发场景下，监控性能**