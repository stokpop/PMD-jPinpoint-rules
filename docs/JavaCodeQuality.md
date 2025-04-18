
Java code quality - pitfalls and best practices
=============================================
By Jeroen Borgers ([jPinpoint](https://www.jpinpoint.com)) and Peter Paul Bakker ([Stokpop](https://www.stokpop.com)), sponsored by Rabobank

# Table of contents

- [Introduction](#introduction)
- [Improper use of BigDecimal](#improper-use-of-bigdecimal)
- [Improper amount representation](#improper-amount-representation)
- [Incorrect equals and hashCode](#incorrect-equals-and-hashcode)
- [Potential Session Data Mix-up](#potential-session-data-mix-up)
- [Suspicious code constructs](#suspicious-code-constructs)
- [Ineffective Lambdas and Streams](#ineffective-lambdas-and-streams)
- [Maintainability](#maintainability)
- [Improved Sonar rules](#improved-sonar-rules)

Introduction
------------
We categorized many performance and quality issues based on performance code reviews, load tests, heap analyses, profiling and production problems of various applications. Several of these items are automated into PMD-jPinpoint-rules code checks.
The next items do have performance implications. However, they are actually more concerning general software quality or security.   
Disclaimer: these best practices come without warranty of any kind. Use at your own risk and always (load)test in your own situation.

Improper use of BigDecimal
--------------------------

#### IUOB01

**Observation: the constructor BigDecimal(double) is used.** 

[Javadoc of this constructor:](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/math/BigDecimal.html#%3Cinit%3E(double)) 

```text
Translates a double into a BigDecimal which is the exact decimal representation of the double's binary floating-point value. The scale of the returned BigDecimal is the smallest value such that (10scale × val) is an integer.

Notes:

1. The results of this constructor can be somewhat unpredictable. One might assume that writing new BigDecimal(0.1) in Java creates a BigDecimal which is exactly equal to 0.1 (an unscaled value of 1, with a scale of 1), but it is actually equal to 0.1000000000000000055511151231257827021181583404541015625. This is because 0.1 cannot be represented exactly as a double (or, for that matter, as a binary fraction of any finite length). Thus, the value that is being passed in to the constructor is not exactly equal to 0.1, appearances notwithstanding.
2. The String constructor, on the other hand, is perfectly predictable: writing new BigDecimal("0.1") creates a BigDecimal which is exactly equal to 0.1, as one would expect. Therefore, it is generally recommended that the String constructor be used in preference to this one.
3. When a double must be used as a source for a BigDecimal, note that this constructor provides an exact conversion; it does not give the same result as converting the double to a String using the Double.toString(double) method and then using the BigDecimal(String) constructor. To get that result, use the static valueOf(double) method.
```
**Problem:** BigDecimal is intended to be used for amounts of e.g. euro’s with cents. It should e.g. represent `0,11` exactly. However, with the used constructor, one easily gets unexpected rounding, e.g.:
```java
System.out.println("result = " + new BigDecimal(0.105).setScale(2, RoundingMode.HALF_UP));
```

Results unexpectedly in:
````text
result = 0.10
````
**Solution:** Use the factory method `BigDecimal.valueOf(double)` instead. E.g.:
````java
System.out.println("result = " + BigDecimal.valueOf(0.105).setScale(2, RoundingMode.HALF_UP));
````
Results as expected in:
````text
result = 0.11
````
Or use long for cents instead of BigDecimal.

#### IUOB02

**Observation: BigDecimal is instantiated with `0`,`1`,`2` or `10`.**  
**Problem:** These instances are already available as `BigDecimal.ZERO`, `BigDecimal.ONE`, `BigDecimal.TWO` (since Java 19!) and `BigDecimal.TEN`  
**Solution:** Use the static instances.

Improper amount representation
------------------------------

Incorrect equals and hashCode
-----------------------------

#### IEAH01

**Observation: Object does not implement equals nor hashCode method.** Often the equals and/or the hashCode methods are missing or incompatible. If equals and hashCode are not well-defined, this means that comparing (with equals) references to these objects is equivalent to finding out whether they refer to the same object, instead of finding out whether the objects are logically equivalent.  
**Problem:** Does not meet programmer expectations and when instances of the class are used as map keys, set elements or certain List operations, this results in unpredictable, undesirable behavior.  
Example where Amount does not define equals nor hashCode method:
````java
Set set = new HashSet();  
set.add(new Amount("0.10"));  
set.add(new Amount("0.10"));  
System.out.println("result = " + set);  
  
Map map = new HashMap();  
map.put(new Amount("0.10"), new Amount("0.10"));  
System.out.println("result = " + map.containsKey(new Amount("0.10")));
````
Results unexpectedly in:
````java
result = [0.10, 0.10]
result = false 
````
**Solution:** Add proper equals and hashCode methods that meet the general contract to all objects which might anyhow be put in a Map, Set or other collection by someone, e.g. the domain objects. See [Effective Java, Chapter 3](http://www.slideshare.net/ibrahimkurce/effective-java-chapter-3-methods-common-to-all-objects). Options to use are:

Three ways to generate the boilerplate code:
1. Use [Project Lombok](https://projectlombok.org/features/index.html) annotation to generate the boilerplate code:
   * @EqualsAndHashCode, or better 
   * @Data, or best 
   * @Value with @Builder and @Singular for collections, for immutability, 

2. Use [Google AutoValue framework](https://github.com/google/auto/blob/master/value/userguide/why.md) (Promoted by Effective Java)
3. Use Immutables framework [Article that compares Lombok, AutoValue and Immutables](https://codeburst.io/lombok-autovalue-and-immutables-or-how-to-write-less-and-better-code-returns-2b2e9273f877)

````
import lombok.*;
class Getters { // bad - equals and hashCode missing
    private String someState1 = "some1";
    private String someState2 = "some2";

    public String getSomeState1() {
        return someState1;
    }
    public String getSomeState2() {
        return someState2;
    }
}

@Getter
class LombokGetterBad { // bad - equals and hashCode missing
    private String someState1 = "some1";
    private String someState2 = "some2";
}

@Getter
@EqualsAndHashCode
class LombokGetterGood { // good  
    private String someState1 = "some1";
    private String someState2 = "some2";
}

@Builder
@Value
class LombokImmutableBetter { // better  
    private String someState1 = "some1";
    private String someState2 = "some2";
}

````
More traditional ways:
1.  [EqualsBuilder](http://commons.apache.org/lang/api-2.4/org/apache/commons/lang/builder/EqualsBuilder.html) and [HashCodeBuilder](http://commons.apache.org/lang/api-2.4/org/apache/commons/lang/builder/HashCodeBuilder.html), without reflection, see [UUOR01](http://wiki.rabobank.nl/wiki/JavaCodePerformance#UUOR01) above.
2.  The more concise [Google Guava Objects](http://guava-libraries.googlecode.com/svn/trunk/javadoc/com/google/common/base/Objects.html) as discussed at [StackOverflow](http://stackoverflow.com/questions/5038204/apache-commons-equals-hashcode-builder).
3.  Java 7 version using the Objects class, see example below.
4.  Your IDE to generate those methods, possibly using one of the above.

Since we use Java 7+, we recommend the Java 7+ version, example:
````java
public boolean equals(Object obj) {  
    if (obj == this) {  
        return true;  
    }  
    if (obj == null || getClass() != obj.getClass()) {  
        return false;  
    }  
    final AddressO7 other = (AddressO7) obj;  
    return Objects.equals(this.houseNumber, other.houseNumber)  
        && Objects.equals(this.street, other.street)  
        && Objects.equals(this.city, other.city)  
        && Objects.equals(this.country, other.country);  
    }  
}  
  
public int hashCode() {  
    return Objects.hash(houseNumber, street, city, country);  
}
````
**Note:** _Always remember to maintain the equals and hashCode on field changes!_

If you are still not convinced, please read: [Effective Java, Chapter 3](http://www.slideshare.net/ibrahimkurce/effective-java-chapter-3-methods-common-to-all-objects) and recent Dzone article: [why-should-you-care-about-equals-and-hashcode](https://dzone.com/articles/why-should-you-care-about-equals-and-hashcode), or more details in [JavaRanch article](http://www.javaranch.com/journal/2002/10/equalhash.html).

##### Equals and hashCode not designed

In the exceptional case that instances of the class will or should never be checked for equality nor be used in any collection type, or to ease the transition period to properly implement equals and hashCode, we recommend to implement equals and hashCode like:
````java
@SuppressFBWarnings("EQ_UNUSUAL")  
public final boolean equals(Object other) {  
    throw new UnsupportedOperationException("equals not designed.");  
}  
  
public final int hashCode() {  
    throw new UnsupportedOperationException("hashCode not designed.");  
}
````
Added advantage is that you don't need to maintain these methods on field changes. The used annotation suppresses FindBugs violations which otherwise show up in Sonar. You need a dependency in your pom, see next mentioned tool.

**Note**: we created a tool that generates this failing equals and hashcode in your classes and takes care of possible complications.

##### Motivation

[Effective Java, Chapter 3](http://www.slideshare.net/ibrahimkurce/effective-java-chapter-3-methods-common-to-all-objects) explains that equals and hashCode of Object usually don't make sense, they are designed to be overridden. It is very easy to introduce bugs by not overriding them. With the UnsupportedException version you get fail-fast in stead of unexpected, undesired behaviour, yet you don't need to create code which needs maintenance and which is not meant to be used, in fact is just waste.

Tip 1: with the [MeanBean](http://meanbean.sourceforge.net/) test library and [Equals Verifier](http://jqno.nl/equalsverifier/) of Jan Ouwens it is easy to test the equals and hashCode contract. These tests can easily be integrated in the existing UnitTests.

Tip 2: [Project Lombok](https://projectlombok.org/features/index.html) provides annotations for boilerplate code like equals and hashCode. Often no need to maintain the methods on field changes.

**Rule-name:** [ImplementEqualsHashCodeOnValueObjects](https://jira.rabobank.nl/browse/JPCC-21).

#### IEAH02

**Observation: Comparable classes don't override equals()**  
**Problem:** If equals() is not overridden, the equals() implementation is not consistent with the compareTo() implementation. If an object of such a class is added to a collection such as java.util.SortedSet, this collection will violate the contract of java.util.Set, which is defined in terms of equals().  
**Solution:** Implement equals consistent with compareTo.

#### IEAH03

**Observation: Equals and hashCode are inconsistent: they are not based on the same fields or use inconsistent conversion.**  
**Problem:** 
Actually, two problem cases: 
1. Equal objects can have different hashCodes and end-up in different buckets of a Map/Set. Strange things can happen like adding an object to a Set and not being able to find it back.
2. Two unequal objects can have the same hashCode and end up in the same bucket of a Map. This may result in bad performance, O(n) lookup instead of O(1).   

**Solution:** When objects are equal, hashCode needs to be equal, for correctness. When objects are not equal, hashCodes should also not be equal for good performance. 
Use the same fields in equals and hashCode and if conversions are needed, use identical conversions in both. So don't use equalsIgnoreCase.  
**Rule-names:** MissingFieldInHashCode, FieldOfHashCodeMissingInEquals, EqualsOperationInconsistentWithHashCode, HashCodeOnlyCallsSuper   
**Examples:**  
````java
class Good {
    String field1, field2;

    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        Good that = (Good) o;
        return Objects.equals(field1, that.field1) &&
                Objects.equals(field2, that.field2);
    }
    public int hashCode() {
        return Objects.hash(field1, field2);
    }
}

class Bad1 {
    String field1;
    String field2; //bad

    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        Bad1 that = (Bad1) o;
        return Objects.equals(field1, that.field1); // field2 missing in equals
    }
    public int hashCode() {
        return Objects.hash(field1, field2); 
    }
}

class Bad2 {
    String field1;
    String field2; //bad

    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        Bad2 that = (Bad2) o;
        if (field1 != null ? !field1.equals(that.field1) : that.field1 != null) return false;
        return field2 != null ? field2.equalsIgnoreCase(that.field2) : that.field2 == null; // bad: ignore case - inconsistent conversion
    }
    public int hashCode() {
        return Objects.hash(field1, field2);
    }
}

class Bad3 {
    String field1;
    String field2; //bad - missing in hashCode

    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        EqHashTryout2 that = (EqHashTryout2) o;
        return Objects.equals(field1, that.field1) &&
                Objects.equals(field2, that.field2);
    }
    public int hashCode() {
        int result = field1 != null ? field1.hashCode() : 0;
        return result;
    }
}

class Bad4 {
    String field1;
    String field2;

    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        Bad4 that = (Bad4) o;
        return Objects.equals(field1.toUpperCase(), that.field1.toUpperCase()) && // bad
                Objects.equals((field2.toLowerCase()), that.field2.toLowerCase()); // bad
    }
    public int hashCode() {
        return Objects.hash(field1, field2);
    }
}

class Bad5 {
    String field1;
    String field2;

    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        Bad5 that = (Bad5) o;
        return Objects.equals(field1.toUpperCase(), that.field1.toUpperCase()) && // bad
                Objects.equals((field2.toLowerCase()), that.field2.toLowerCase());
    }
    public int hashCode() {
        return Objects.hash(field1.toLowerCase(), field2.toLowerCase());
    }
}

class Bad6 {
    String field1;

    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        Bad6 that = (Bad6) o;
        return Objects.equals(field1, that.field1);
    }
    public int hashCode() {
        return super.hashCode(); // bad
    }
}

````


#### IEAH04

**Observation: A field simply assigned to is missing in the equals method**  
**Problem:**  If a field which can be assigned separately (independent of other fields) is missing in the equals method, then changing that field in one object has no effect on the equality with another object.
However, if a field of one of two equal objects is changed, the expectation is that they are no longer equal. In other words, objects are typically only considered equal when all their fields have equal values. 
**Solution:** include the missing field in the equals and hashCode method.  
**Examples:**
````java
class Bad1 {
    String field1;
    final String field2; // bad, missing in equals

    public Bad1(String arg2) {
        field2 = arg2;
    }
    public void setField1(String arg1) {
        field1 = arg1;
    }
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        Bad1 that = (Bad1) o;
        return Objects.equals(field1, that.field1);
    }
    public int hashCode() {
        return Objects.hash(field1);
    }
}
````
**Rule-name:** MissingFieldInEquals

Potential Session Data Mix-up
-----------------------------

Session data mixup is one of the worst problems that can occur. Customers seeing data of other customers is bad, for the users and for the reputation of the company. Therefore, we want to have defence mechanisms to protect against these problems. There can be several causes of session data mixup, like:

*   a response over a connection being bound to the wrong request, so request-response mixup.
*   shared variables like singleton fields, e.g. in a servlet, Spring @Component, JavaEE @Singleton.
*   cache key mixup: when two users use the same key in a shared cache. See [caching pitfalls](https://github.com/jborgers/PMD-jPinpoint-rules/blob/master/docs/JavaCodePerformance.md#improper-caching).

#### PSDM01

**Observation: a response from a service call is not validated on business level with the request**   
**Problem:** Software infrastructure (pool, queue) with problems like misconfiguration or bugs, can cause a response being returned for a different request than the original. So a request-response mixup.   
**Solution:** Check if a business value like customerId or accountNumber is equal for the request and the response. If not, log an error and do *not* show the data to the user.   

Suspicious code constructs
--------------------------

### SSC01

**Observation: Multiple switch cases contain the same assignment**  
**Problem:** Identical assignments to the same variable are very likely a bug. It lead to a production incident in a project.  
**Solution:** Each case block of a switch should contain unique assignments. Common assignments should be taken out of the switch construct. 
Exceptional case: a duplicate assignment to a boolean is considered safe since it can only hold 2 values.    
**Rule name:** AvoidDuplicateAssignmentsInCases

### SSC02

**Observation: Improper combination of annotations.**   
**Problem:** These annotations are not meant to be combined and may cause unexpected and unwanted behavior, e.g. data mix-up.
Don't combine:
  * 2+ of [@Component, @Service, @Configuration, @Controller, @RestController, @Repository, @Entity] (Spring/JPA)   
  * @Aspect with one of [@Service, @Configuration, @Controller, @RestController, @Repository, @Entity] (Spring/AspectJ)   
  * [@Data with @Value]; and [@Data or @Value] with any of [@ToString, @EqualsHashCode, @Getter, @Setter, @RequiredArgsConstructor] (Lombok)   
  * @Data with any of [@Component, @Service, @Controller, @RestController, @Repository], it may cause user data mix-up: 
@Data is meant for passing around, to put in collections, etc. while 
@Component is typically meant for singletons, preferably stateless; and if not: made thread-safe. 
    * Note that @Data combined with @Configuration is not on this list since the setters are called only once by Spring to initialize (assumed in a thread-safe manner) and are thread-safe afterward.     

**Solution:** Don't combine the annotations. Understand what they do and choose and keep only the right one.     
**Rule name:** AvoidImproperAnnotationCombinations

### SSC03

**Observation: A shared object has a field with a name which indicates user related data.**  
**Problem:** A shared/singleton object like Spring @Component is shared among users and so are its fields. 
User related data fields in such a component will be shared among users accessing it at about the same time. 
Therefore, this data can mix-up: a user can access data of another user, which is really bad.  
**Solution:** 
* Do *not* put the user related data in a shared component. Use a POJO, think about scope.   
* If the field does not actually reference user data, change to a name which does not indicate user data e.g.   
``private Scheduler schedulerOrderReference;`` rename to: ``private Scheduler orderReferenceScheduler;``   
``private OrderHandler handleOrders;`` rename to: ``private OrderHandler orderHandler;``   
The last or second last word as noun is assumed to be the entity, so handleOrders is assumes to be order objects which typically is user related data. A handler is not.   
**Rule name:** AvoidUserDataInSharedObjects   
**Details:** The regular expression for matching the name of the field assumed to be user data: 
``User[Id|Ref|Reference]*$|Customer[Id|Ref|Reference]*$|Session[Id|Ref|Reference]*$|Order[Id|Ref|Reference|List]*$|Account[Id|Ref|Reference|List]*$|Transaction[Id|Ref|Reference|List]*$|Contract[Id|Ref|Reference|List]*$``   
**Example:**
````java
@Component // one instance shared among requests and users
@Data
class VMRDataBad {
    private List<OrderDetails> orderList; // bad 1
    private String authUser; // bad 2
    private String sessionId; // bad 3
    private OrderHandler handleOrders; // bad 4, improper name
}

// POJO, not shared
@Data
class VMRDataGood {
    private List<OrderDetails> orderList; 
    private String authUser; 
    private String sessionId; 
    private OrderHandler orderHandler; // proper field name
}
````

Ineffective Lambdas and Streams
-------------------------------

### ILS01

**Observation: forEach is used to perform a stream computation.**  
**Problem:** Java Streams is a paradigm based on functional programming: the result should depend only on its input, not on any mutable state nor should it update any state.
Use of forEach in a stream is actually iterative code masquerading as streams code. It is typically harder to read and less maintainable than the iterative form.  
**Solution:** Use the for-each (enhanced-for) loop, or the pure functional form. The forEach operation should only be used to report (i.e. log) the result of a stream computation.    
**See:** Effective Java 3rd Ed. Item 46: Prefer Side-Effect-Free Functions In Streams.   
**Rule name:** AvoidForEachInStreams   
**Example:**
````java
class AvoidForEachInStreams {
    List<String> letters = Arrays.asList("a", "b", "c");
    Map<String, Integer> map;
    void forEachInStream() {
        map = new HashMap<>();
        letters.forEach(l -> map.put(l, 0)); // bad, side effect, modifies map
    }
    void forEachInLogging() {
        letters.forEach(Log::info); // good, logging is okay
    }
    void forEachLoop() {
        map = new HashMap<>();
        for (String l : letters) {  // good, meant for modifying state
            map.put(l, 0);
        }
    }
    Map pureFunctional() {
        return letters.stream().collect(toMap(l -> l, v -> 0)); // good, no side effects, no state used nor modified
    }
    void forEachInRange() {
        map = new HashMap<>();
        IntStream.range(0, letters.size()).forEach(i -> map.put(letters.get(i), 0)); // bad, side effect, modifies map
    }
}
````

Maintainability
---------------

### M01

**Observation: Variable does not have a meaningful name, like 'var3' and fields like 'FOUR = 4'**  
**Problem:** Variable does not express what it is used for. This is bad for maintainability.  
**Solution:** Let variable names express what they are used for, like 'key' and 'MAX_KEYS = 4'.    
**Rule name:** ImproperVariableName
**Example:**
````java
class Foo {
private static final int FOUR_ZERO_NINE_SIX = 4096; // bad
private static int six = 6; // bad
private int five = 6; // really bad
private static final int SIXTIES_START = 1960; // good

    void bar() {
        String var1 = "baz"; // bad
    }
}
````

Improved Sonar rules
--------------------
We found that several Sonar rules do not meet our needs. We need an improved version.

### ISR01
**Issue:** [202](https://github.com/jborgers/PMD-jPinpoint-rules/issues/202)   
**Observation: The number of static imports with a wildcard exceeds the limit (default = 3)**  
**Problem:** If you import the public static members of too many classes, your code can become confusing and difficult to maintain.  
**Solution:** Import class members individually.     
**Rule name:** LimitWildcardStaticImports   
**Sonar rule(s):** java:S3030 - Classes should not have too many "static" imports (inadequate rule)    
java:S2208 - Wildcard imports should not be used - static imports are ignored by this rule.

### ISR02
**Issue:** [201](https://github.com/jborgers/PMD-jPinpoint-rules/issues/201)   
**Observation: Field in a Serializable class is not serializable nor transient.**  
**Problem:** When (de)serialization happens, a RuntimeException will be thrown and (de)serialization fails.   
**Solution:** make the field either transient or make its class implement Serializable.     
**Note:** Classes extending Throwable do, by inheritance, implement Serializable, yet are excluded in this rule, since they are typically never actually serialized. 
An exception to this exception is when extending RemoteException, then fields should be transient or serializable.
**Rule name:** LetFieldsMeetSerializable   
**Sonar rule(s):** S1948 - Fields in a "Serializable" class should either be transient or serializable (inadequate rule).   

### ISR03
**Issue:** [200](https://github.com/jborgers/PMD-jPinpoint-rules/issues/200)   
**Observation: A lambda expression has many statements: it exceeds the limit (default = 5).**  
**Problem:** lambda expressions with many statements are hard to understand and maintain.  
**Solution:** extract the lambda expression code block into one or more separate method(s).     
**Rule name:** LimitStatementsInLambdas   
**Sonar rule(s):** java:S5612 - Lambdas should not have too many lines (inadequate rule).   

### ISR04
**Issue:** [220](https://github.com/jborgers/PMD-jPinpoint-rules/issues/220)   
**Observation: A value assigned to a local variable is never used.**   
**Problem:** Assignments to variables for which the assigned value is not used because a new value is assigned before actual use, is unnecessary work and may indicate a bug.  
**Solution:** Remove the first assignment and make sure that is as intended.     
**Rule name:** AvoidUnusedAssignments   
**Sonar rule(s):** java:S1854 - Unused assignments should be removed (inadequate rule).   

### ISR05
**Issue:** [200](https://github.com/jborgers/PMD-jPinpoint-rules/issues/200)   
**Observation: A lambda expression has deep nesting of lambdas.** 
The depth of lambda-with-code-block nesting exceeds (by default) 1,
or the depth of lambda-single-expression in lambda nesting exceeds (by default) 4.    
**Problem:** lambda expressions with deep nested lambdas are hard to understand and maintain.  
**Solution:** extract the lambda expression code block into one or more separate method(s).     
**Rule name:** LimitNestingInLambdas   
**Sonar rule(s):** java:S5612 - Lambdas should not have too many lines (inadequate rule).   
