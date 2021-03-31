### Answer

Something called the "bridge method" concept related to Java Generics made me stop at a point and think over it.

Btw, I only know that it occurs at the bytecode level and is not available for us to use.

But I am eager to know the concept behind the "bridge method" used by the Java compiler.

What exactly happens behind the scenes and why it is used?

Any help with an example would be greatly appreciated.

### Question

It's a method that allows a class extending a generic class or implementing a generic interface (with a concrete type parameter) to still be used as a raw type.

Imagine this:

```java
public class MyComparator implements Comparator<Integer> {
   public int compare(Integer a, Integer b) {
      //
   }
}
```

This can't be used in its raw form, passing two `Object`s to compare, because the types are compiled in to the compare method (contrary to what would happen were it a generic type parameter T, where the type would be erased). So instead, behind the scenes, the compiler adds a "bridge method", which looks something like this (were it Java source):

```java
public class MyComparator implements Comparator<Integer> {
   public int compare(Integer a, Integer b) {
      //
   }

   //THIS is a "bridge method"
   public int compare(Object a, Object b) {
      return compare((Integer)a, (Integer)b);
   }
}
```

The compiler protects access to the bridge method, enforcing that explicit calls directly to it result in a compile time error. Now the class can be used in its raw form as well:

```java
Object a = 5;
Object b = 6;

Comparator rawComp = new MyComparator();
int comp = rawComp.compare(a, b);
```

**Why else is it needed?**

In addition to adding support for explicit use of raw types (which is mainly for backwards compatability) bridge methods are also required to support type erasure. With type erasure, a method like this:

```java
public <T> T max(List<T> list, Comparator<T> comp) {
   T biggestSoFar = list.get(0);
   for ( T t : list ) {
       if (comp.compare(t, biggestSoFar) > 0) {
          biggestSoFar = t;
       }
   }
   return biggestSoFar;
}
```

is actually compiled into bytecode compatible with this:

```java
public Object max(List list, Comparator comp) {
   Object biggestSoFar = list.get(0);
   for ( Object  t : list ) {
       if (comp.compare(t, biggestSoFar) > 0) {  //IMPORTANT
          biggestSoFar = t;
       }
   }
   return biggestSoFar;
}
```

If the bridge method didn't exist and you passed a List<Integer> and a MyComparator to this function, the call at the line tagged IMPORTANT would fail since MyComparator would have no method called compare that takes two Objects...only one that takes two Integers.

The FAQ below is a good read.

See Also:

[The Generics FAQ - What is a bridge method?](http://www.angelikalanger.com/GenericsFAQ/FAQSections/TechnicalDetails.html#FAQ102)

[Java bridge methods explained (thanks @Bozho)](http://stas-blogspot.blogspot.com/2010/03/java-bridge-methods-explained.html)

[Java Generics - Bridge method](https://stackoverflow.com/questions/5007357/java-generics-bridge-method)