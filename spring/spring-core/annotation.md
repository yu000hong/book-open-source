# Spring 类型与注解

- org.springframework.core.annotation
- org.springframework.core.type


### Meta Annotations

A meta-annotation is an annotation that is declared on another annotation. An annotation is therefore meta-annotated if it is annotated with another annotation. 

### Composed Annotations

A composed annotation is an annotation that is meta-annotated with one or more annotations with the intent of combining the behavior associated with those meta-annotations into a single custom annotation.

### Annotation Presence

The terms `directly present`, `indirectly present`, and `present` have the same meanings as defined in the class-level Javadoc for `java.lang.reflect.AnnotatedElement` in Java 8.

In Spring, an annotation is considered to be meta-present on an element if the annotation is declared as a meta-annotation on some other annotation which is present on the element. For example, given the aforementioned @TransactionalService, we would say that @Transactional is meta-present on any class that is directly annotated with @TransactionalService.


[MergedAnnotation API internals](https://github.com/spring-projects/spring-framework/wiki/MergedAnnotation-API-internals)

[Spring Annotation Programming Model](https://github.com/spring-projects/spring-framework/wiki/Spring-Annotation-Programming-Model)
