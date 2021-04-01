# GC Roots

In Java, GC roots can be four types of objects:

- Objects referenced in the virtual machine (VM) stack, that is the local variable table in the stack frame
- Objects referenced by class static attributes in the method area
- Objects referenced by constants in the method area
- Objects referenced by JNI (the Native method) in the native method stack

GC roots这组引用是tracing GC的起点。要实现语义正确的tracing GC，就必须要能完整枚举出所有的GC roots，否则就可能会漏扫描应该存活的对象，导致GC错误回收了这些被漏扫的活对象。