# <b>Topic in This Section</b>
<div style="float:left; text-align:left">
    <b>A. Parallel Streams</b></br>
    - <span style="font-size:75%">Why stream approach is often better than traditional loops</span></br>
    - <span style="font-size:75%">Best practices: verifying speedup and “same” answer</span></br>
    - <span style="font-size:75%">Knowing when parallel reduce is safe</span></br>
    B. <b>Infinite streams</b></br>
    - <span style="font-size:75%">Really unbounded streams with values that are calculated on the fly</span></br>
    C. <b>Grouping stream elements</b></br>
    - <span style="font-size:75%">Fancy uses of collect</span>
</div>
---

## <b>A. Parallel Streams</b>

- <span style="font-weight:bold">Big idea</span>
    - <span style="font-size:75%">By designating that a Stream be parallel, the operations are automatically done in parallel, without any need for explicit fork/join or threading code</span>
- <b>Designating streams as parallel</b>
    - <span style="font-size:75%">anyStream.<span style="color:red">parallel()</span></span>
    - <span style="font-size:75%">anyList.<span style="color:red">parallelStream()</span></span>
- <b>Quick examples</b>
    - <span style="font-size:75%">Do side effects serially</span>
    - <span style="font-size:75%">Do side effects in parallel</span>

---

### <b>Parallel Streams vs Concurrent Programming with Threads</b>

--
#### <b>Parallel streams</b>

* <span style="font-size:75%">Use fork/join framework internally (see separate lecture)</span>
* <span style="font-size:75%">Have one thread per core</span>
* <span style="font-size:75%">Are beneficial even when you never wait for I/O</span>
* <span style="font-size:75%">Have no benefit on single-core computers</span>
* <span style="font-size:75%">Can be used with minimal changes to serial code</span>

--
#### <b>Concurrent programming with explicit threads (see separate lecture)</b>

* <span style="font-size:75%">Are usually beneficial only when you wait for I/O (disk, screen, network, user, etc.)</span>
* <span style="font-size:75%">Are often beneficial even on single-core computers</span>
* <span style="font-size:75%">May have a lot more threads than there are cores</span>
* <span style="font-size:75%">Require very large changes to serial code</span>

--
#### <b>Bottom line</b>
* <span style="font-size:75%">Parallel streams are often <i>far</i> easier to use than explicit threads</span>
* <span style="font-size:75%">Explicit threads apply in more situations</span>
---

### <b>Advantage of Using Streams vs. Traditional Loops: Parallel Code </b>

--

* <b>Computing sum with normal loop</b>

```java
int[] nums = …;
int sum = 0;
for(int num: nums) {
    sum += num;
}
```
* <span style="font-size:75%"><i>Cannot</i> be easily parallelized</span>

--

* <b>Computing sum with reduction operation</b>

```java
    int[] nums = …;
    int sum = IntStream.of(nums).sum();
```
* <span style="font-size:75%"><i>Can</i> be easily parallelized</span>

```java
    int sum2 = IntStream.of(nums).parallel().sum();
```

---

### <b>Best Practices: Both Fork/Join and Parallel Streams</b>

--

##### <b>Check that you get the same answer</b>
* <span style="font-size:75%">Verify that sequential and parallel versions yield the same (or close enough to the
same) results.</span>
--

##### <b>Check that the parallel version is faster</b>
* <span style="font-size:75%">At the least, no slower. Test in real-life environment.</span>
* <span style="font-size:75%">If running on app server, this analysis might be harder than it looks. Servers
automatically handle requests concurrently, and if you have heavy server load and
many of those requests use parallel streams, all the cores are likely to already be in
use, and parallel execution might have little or no speedup.</span>
    * <span style="font-size:75%">Another way of saying this is that if the CPU of your server is already consistently
maxed out, then parallel streams will not benefit you (and could even slow you down).</span>
---

### <b>Will You Always Get Same Answer in Parallel?</b>

--
* <b>sorted, min, max</b>
    * <span style="font-size:75%">No. (Why not? And do you care?)</span>
* <b>findFirst</b>
    * <span style="font-size:75%">No. Use findAny if you do not care.</span>
* <b>map, filter</b>
    * <span style="font-size:75%">No, but you do not care about what the stream looks like in intermediate stages (You only care about the terminal operation) </span>

--
* <b>allMatch, anyMatch, noneMatch, count</b>
    * <span style="font-size:75%">Yes</span>
* <b>reduce (and sum and average)</b>
    * <span style="font-size:75%">It depends!</span>
---

### <b>Equivalence of Parallel Reduction Operations</b>

--
* <b>sum, average are the same if</b>
    * <span style="font-size:75%">You use IntStream or LongStream</span>
* <b>sum, average could be different if</b>
    * <span style="font-size:75%">You use DoubleStream. Reording the additions could yield slightly different answers due to roundoff error.</span>
        * <span style="font-size:65%">“Almost the same” may or may not be acceptable</span>
        * <span style="font-size:65%">If the answers differ, it is not clear which one is “right”</span>

--
* <b>reduce is the same if</b>
    * <span style="font-size:75%">No side effects on global data are performed</span>
    * <span style="font-size:75%">The combining operation is associative (i.e., where reordering the operations does not matter).</span>
        * <span style="font-size:50%">Reordering addition or multiplication of doubles does not necessarily yield exactly the same answer. You may or may not care.</span>
---

### <b>Parallel reduce: No Global Data</b>

--
* <b>Binary operator itself should be stateless</b>
    * <span style="font-size:75%">Guaranteed if an explicit lambda is used, but not guaranteed if you directly build an
instance of a class that implements BinaryOperator, or if you use a method reference
that refers to a statefull class</span>

--
* <b>The operator does not modify global data</b>
    * <span style="font-size:75%">The body of a lambda is allowed to mutate instance variables of the surrounding
class or call setter methods that modify instance variables. If you do so, there is no
guarantee that parallel reduce will be safe.</span>
---

### <b>Parallel reduce: Associative Operation</b>

--
* <b>Reordering ops should have no effect</b>
    * <span style="font-size:75%"> I.e., (a op b) op c == a op (b op c)</span>

--
* <b>are these associative?</b>
    * <span style="font-size:75%"> Division? No.</span>
    * <span style="font-size:75%"> Subtraction? No.</span>
    * <span style="font-size:75%"> Addition or multiplication of ints? Yes.</span>
    * <span style="font-size:75%"> Addition or multiplication of doubles? No.</span>
        * <span style="font-size:50%"> Not guaranteed to get exactly the same result. This may or may not be tolerable in
your application, so you should define an acceptable delta and test. If your answers
differ by too much, you also have to decide which is the “right” answer.</span>
---

### <b>Example: Parallel Sum of Square Roots of Doubled Vals</b>
```java
public class MathUtils {
    public static double fancySum1(double[] nums) {
        return DoubleStream.of(nums)
            .map(d -> Math.sqrt(2*d))
                .sum();
    }
    public static double fancySum2(double[] nums) {
        return DoubleStream.of(nums)
            .parallel()
                .map(d -> Math.sqrt(2*d))
                    .sum();
}
```

--

### <b>Helper Method</b>
```java
public static double[] randomNums(int length) {
    double[] nums = new double[length];
    for(int i=0; i<length; i++) {
        nums[i] = Math.random() * 3;
    }
    return(nums);
}
```

--
### <b>Verifying “Same” Result: Test Code</b>

```java
public static void compareOutput() {
    double[] nums = MathUtils.randomNums(10_000_000);
    double result1 = MathUtils.fancySum1(nums);
    System.out.printf("Serial result = %,.12f%n", result1);
    double result2 = MathUtils.fancySum2(nums);
    System.out.printf("Parallel result = %,.12f%n", result2);
}
```
<div style="float: right">
    <small style="text-align: left;">
        <u>Representative output</u> </br>
        Serial result = 16,328,996.08110622<span style="color: red">3</span>000 </br>
        Parallel result = 16,328,996.08110622<span style="color: red">5</span>000
    </small>
</div>

--
### <b>Comparing Performance: Test Code</b>

```java
public static void compareTiming() {
    for(int i=5; i<9; i++) {
        int size = (int)Math.pow(10, i);
        double[] nums = MathUtils.randomNums(size);
        Op serialSum = () -> MathUtils.fancySum1(nums); // non parallel
        Op parallelSum = () -> MathUtils.fancySum2(nums); // parallel
        System.out.printf("Serial sum for length %,d.%n", size);
        Op.timeOp(serialSum);
        System.out.printf("Parallel sum for length %,d.%n", size);
        Op.timeOp(parallelSum);
    }
}
```

--
### <b>Comparing Performance: Representative Output</b>

<pre>
    Serial sum for length 100,000.
    Elapsed time: <span style="color: blue">0.012</span> seconds.
    Parallel sum for length 100,000.
    Elapsed time: <span style="color: red">0.008</span> seconds.
    Serial sum for length 1,000,000.
    Elapsed time: <span style="color: blue">0.005</span> seconds.
    Parallel sum for length 1,000,000.
    Elapsed time: <span style="color: red">0.003</span> seconds.
    Serial sum for length 10,000,000.
    Elapsed time: <span style="color: blue">0.047</span> seconds.
    Parallel sum for length 10,000,000.
    Elapsed time: <span style="color: red">0.024</span> seconds.
    Serial sum for length 100,000,000.
    Elapsed time: <span style="color: blue">0.461</span> seconds.
    Parallel sum for length 100,000,000.
    Elapsed time: <span style="color: red">0.176</span> seconds.
</pre>
---

## <b>B. Infinite (Unbounded On-the-Fly) Streams</b>
* <b>Usage</b>
    * <span style="font-size:75%">The values are not calculated until they are needed</span>
    * <span style="font-size:75%">To avoid unterminated processing, you must eventually use a size-limiting operation like
limit or findFirst (but not skip alone)</span>
        * <span style="font-size:50%">The point is not really that this is an “infinite” Stream, but that it is an unbounded “on the fly”
Stream – one with no fixed size, where the values are calculated as you need them.</span>

---
#### <b>Stream.generate(valueGenerator)<b>
* <span style="font-size:75%">Stream.generate lets you specify a Supplier. This Supplier is invoked each time the system
needs a Stream element.</span>
    * <span style="font-size:50%">Powerful when Supplier maintains state, but then parallel version will give wrong answer</span>

--
### <b>Big ideas</b>
* You supply a function (Supplier) to Stream.generate. Whenever the system needs
stream elements, it invokes the function to get them.
    * <span style="font-size:75%">You must limit the Stream size.</span>
        * <span style="font-size:50%">Usually with limit or findFirst (or findAny for parallel streams). skip alone is not enough,
since the size is still unbounded</span>
    * <span style="font-size:75%">By using a real class instead of a lambda, the function can maintain state so that new
values are based on any or all of the previous values</span>

--
### <b>Quick example</b>

```java
List<Employee> emps =
    Stream.generate(() -> randomEmployee()) // generate
        .limit(someRuntimeValue)
            .collect(Collectors.toList());
```

--
### <b>Stateless generate Example: Random Numbers</b>
- Code <!-- .element: class="fa-code" -->

```java
Supplier<Double> random = Math::random;
System.out.println("2 Random numbers:");
Stream.generate(random).limit(2).forEach(System.out::println); //generate(random)
System.out.println("4 Random numbers:");
Stream.generate(random).limit(4).forEach(System.out::println); //generate(random)
```
- Result <!-- .element: class="fa-code" -->

```java
2 Random numbers:
0.00608980775038892
0.2696067924164013
4 Random numbers:
0.7761651889987567
0.818313574113532
0.07824375091607816
0.7154788145391667
```

--

### <b>Stateful generate Example: Supplier Code</b>

```java
public class FibonacciMaker implements Supplier<Long> { //implements Supplier<Long>
    private long previous = 0;
    private long current = 1;
    @Override
    public Long get() {
        long next = current + previous;
        previous = current;
        current = next;
        return(previous);
    }
}
```

<div style="float: right; font-size: 75%">
    <small style="text-align: left">
        Lambdas cannot define instance variables, so we use a regular class instead of a</br>
        lambda to define the Supplier. Also, see the section on lambda-related methods in</br>
        Lists and Maps for a Fibonacci-generation method.
    </small>
</div>

--

### <b>Helper Code: Simple Methods to Get Any Amount of Fibonacci Numbers</b>

```java
public static Stream<Long> makeFibStream() {
    return(Stream.generate(new FibonacciMaker())); //Stream.generate(new FibonacciMaker())
}
public static Stream<Long> makeFibStream(int numFibs) {
    return(makeFibStream().limit(numFibs)); //makeFibStream().limit(numFibs)
}
public static List<Long> makeFibList(int numFibs) {
    return(makeFibStream(numFibs).collect(Collectors.toList()));
}
public static Long[] makeFibArray(int numFibs) {
    return(makeFibStream(numFibs).toArray(Long[]::new));
}
```

--
### <b>Stateful generate Example</b>

- Main Code <!-- .element: class="fa-code" -->

```java
System.out.println("5 Fibonacci numbers:");
FibStream.makeFibStream(5).forEach(System.out::println);
System.out.println("25 Fibonacci numbers:");
FibStream.makeFibStream(25).forEach(System.out::println);
```

- Result <!-- .element: class="fa-code" -->

```java
5 Fibonacci numbers:
1
1
2
3
5
25 Fibonacci numbers:
1
1
...
75025
```

---
#### <b>Stream.iterate(initialValue, valueTransformer)</b>
* <span style="font-size:75%">Stream.iterate lets you specify a seed and a UnaryOperator f. The seed becomes the first
  element of the Stream, f(seed) becomes the second element, f(second) becomes third
  element, etc</span>

--
### <b>Big Ideas</b>
* You specify a seed value and a UnaryOperator f. The seed becomes the first element
of the Stream, f(seed) becomes the second element, f(second) [i.e., f(f(seed))]
becomes third element, etc.
    * <span style="font-size:75%">You must limit the Stream size. </br></span>
         <span style="font-size:65%">( Usually with limit. skip alone is not enough, since the size is still unbounded)</span>
    * <span style="font-size:75%">Will not yield the same result in parallel</span>

--
### <b>Quick example</b>

```java
List<Integer> powersOfTwo = Stream.iterate(1, n -> n * 2) //iterate
    .limit(…)
        .collect(Collectors.toList());
```

--
### <b>Simple Example: Twitter Messages</b>
* <b>Idea</b>
    * <span style="font-size:75%">Generate a series of Twitter messages</span>
* <b>Approach</b>
    * <span style="font-size:75%">Start with a very short String as the first message</span>
    * <span style="font-size:75%">Append exclamation points on the end</span>
    * <span style="font-size:75%">Continue to 140-character limit</span>
* Core code <!-- .element: class="fa-code" -->

```java
Stream.iterate("Base Msg",msg -> msg + "Suffix")
    .limit(someCutoff)
```

--
### <b>Twitter Messages: Example</b>
* Code <!-- .element: class="fa-code" -->

```java
System.out.println("14 Twitter messages:");
Stream.iterate("Big News!!", msg -> msg + "!!!!!!!!!!") //iterate
    .limit(14)
        .forEach(System.out::println);
```

* Result <!-- .element: class="fa-code" -->

```java
Big News!!
Big News!!!!!!!!!!!!
Big News!!!!!!!!!!!!!!!!!!!!!!
Big News!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
```

--
### <b>More Complex Example: Consecutive Large Prime Numbers</b>

* <b>Idea</b>
    * <span style="font-size:75%">Generate a series of very large consecutive prime numbers (e.g., 100 or 150 digits or more)</span>
    * <span style="font-size:75%">Large primes are used extensively in cryptography</span>
* <b>Approach</b>
    * <span style="font-size:75%">Start with a prime BigInteger as the seed</span>
    * <span style="font-size:75%">Supply a UnaryOperator that finds the first prime number higher than the given one</span>
* Core code <!-- .element: class="fa-code" -->

```java
Stream.iterate(Primes.findPrime(numDigits), Primes::nextPrime)
    .limit(someCutoff)
```
--
### <b>Helper Methods: Idea</b>

* <b>Idea</b>
    * <span style="font-size:75%">Generate a random odd BigInteger of the requested size, check if prime, keep adding 2 until you find a match.</span>
* <b>Why this is feasible</b>
    * <span style="font-size:75%">The BigInteger class has a builtin probabilistic algorithm (Miller-Rabin test) for
    determining if a number is prime without attempting to factor it. It is ultra-fast even
    for 100-digit or 200-digit numbers.</span>
    * <span style="font-size:75%">Technically, there is a 2100 chance that this falsely identifies a prime, but since 2100
    is about the number of particles in the universe, that’s not a very big risk</span></br>
        <span style="font-size:50%">(Algorithm is not fooled by Carmichael numbers)</span>

--
### <b>Helper Methods: Code</b>

```java
public static BigInteger nextPrime(BigInteger start) {
    if (isEven(start)) {
        start = start.add(ONE);
    } else {
        start = start.add(TWO);
    }
    if (start.isProbablePrime(ERR_VAL)) {
        return(start);
    } else {
        return(nextPrime(start));
    }
}
public static BigInteger findPrime(int numDigits) {
    if (numDigits < 1) {
        numDigits = 1;
    }
    return(nextPrime(randomNum(numDigits)));
}
```
<small style="float: right; font-size:30%">
    Complete source code can be downloaded from http://www.coreservlets.com/java-8-tutorial/
</small>

--
### <b>Making Stream of Primes</b>

```java
public static Stream<BigInteger> makePrimeStream(int numDigits) {
    return(Stream.iterate(Primes.findPrime(numDigits), Primes::nextPrime));//iterate
}
public static Stream<BigInteger> makePrimeStream(int numDigits, int numPrimes) {
    return(makePrimeStream(numDigits).limit(numPrimes));
}
public static List<BigInteger> makePrimeList(int numDigits, int numPrimes) {
    return(makePrimeStream(numDigits, numPrimes).collect(Collectors.toList()));
}
public static BigInteger[] makePrimeArray(int numDigits, int numPrimes) {
    return(makePrimeStream(numDigits, numPrimes).toArray(BigInteger[]::new));
}
```

--
### <b>Primes</b>
* Code <!-- .element: class="fa-code" -->

```java
System.out.println("10 100-digit primes:");
PrimeStream.makePrimeStream(100, 10).forEach(System.out::println);
```

* Result <!-- .element: class="fa-code" -->

```java
10 100-digit primes:
3484894489805924599033259501599559961057903572743870105972345438458556253531271262848463552867976353
3484894489805924599033259501599559961057903572743870105972345438458556253531271262848463552867976647
3484894489805924599033259501599559961057903572743870105972345438458556253531271262848463552867976663
3484894489805924599033259501599559961057903572743870105972345438458556253531271262848463552867976689
3484894489805924599033259501599559961057903572743870105972345438458556253531271262848463552867977233
3484894489805924599033259501599559961057903572743870105972345438458556253531271262848463552867977859
3484894489805924599033259501599559961057903572743870105972345438458556253531271262848463552867977889
3484894489805924599033259501599559961057903572743870105972345438458556253531271262848463552867977989
3484894489805924599033259501599559961057903572743870105972345438458556253531271262848463552867978031
3484894489805924599033259501599559961057903572743870105972345438458556253531271262848463552867978103
```

---
## <b>C. Grouping Stream Elements</b>
* <b>Big idea</b>
    * <span style="font-size:75%">Using methods in the Collectors class, you can output a Stream as many types</span>

--
#### <b>Quick examples</b>
* <span style="font-size:75%">List (shown in previous section)</span></br>
    <span style="font-size:65%"><pre>       anyStream.collect(<span style="color:red">toList</span>())</pre></span>
* <span style="font-size:75%">String</span></br>
    <span style="font-size:65%"><pre>       stringStream.collect(<span style="color:red">joining</span>(delimiter)).toString()<pre></span>
* <span style="font-size:75%">Set</span>
    <span style="font-size:65%"><pre>       anyStream.collect(<span style="color:red">toSet</span>())<pre></span>
* <span style="font-size:75%">Other collection</span>
    <span style="font-size:65%"><pre>       anyStream.collect(<span style="color:red">toCollection</span>(CollectionType::new))<pre></span>
* <span style="font-size:75%">Map</span></br>
    <span style="font-size:65%"><pre>       strm.collect(<span style="color:red">partioningBy</span>(...)), strm.collect(<span style="color:red">groupingBy</span>(...))(toCollection(CollectionType::new))<pre></span>

Note:
For brevity, the examples here assume you have done
“import static java.util.stream.Collectors.*”,
so that toList() really means Collectors.toList()

--
#### <b>Most Common Output: Building Lists</b>

* Code <!-- .element: class="fa-code" -->

```java
    List<Integer> ids = Arrays.asList(2, 4, 6, 8);
    List<Employee> emps = ids.stream()
        .map(EmployeeSamples::findGoogler)
            .collect(Collectors.toList());
    System.out.printf("Googlers with ids %s: %s.%n", ids, emps);
```
* Results <!-- .element: class="fa-code" -->

```java
    Googlers with ids [2, 4, 6, 8]:
    [Sergey Brin [Employee#2 $8,888,888],
    Nikesh Arora [Employee#4 $6,666,666],
    Patrick Pichette [Employee#6 $4,444,444],
    Peter Norvig [Employee#8 $900,000]].
```

Note:
Remember that you can do a static import on
java.util.stream.Collectors.* so that you can use toList()
instead of Collectors.toList().
Also recall that List has a builtin toString method that
prints the entries comma-separated inside square brackets.
Here and elsewhere, line breaks and whitespace added to
printouts for readability

--
#### <b>Aside: The StringJoiner Class</b>
* <b>Big idea</b>
    * <span style="font-size:75%">Java 8 added new StringJoiner class that builds delimiter-separated Strings, with optional prefix and suffix</span>
    * <span style="font-size:75%">Java 8 also added static “join” method to the String class; it uses StringJoiner</span>
* <b>Quick examples (result: "Java, Lisp, Ruby")</b>
    * <span style="font-size:75%">Explicit StringJoiner with no prefix or suffix</span>
        <pre>
            StringJoiner joiner1 = new StringJoiner(", ");
            String result1 = joiner1.add("Java").add("Lisp").add("Ruby").toString();
        </pre>
    * <span style="font-size:75%">Usually easier: String.join convenience method</span>
        <pre>
            String result2 = String.join(", ", "Java", "Lisp", "Ruby");
        </pre>

--
#### <b>Building Strings</b>
* Code <!-- .element: class="fa-code" -->

```java
List<Integer> ids = Arrays.asList(2, 4, 6, 8);
String lastNames = ids.stream().map(EmployeeSamples::findGoogler)
    .filter(e -> e != null)
        .map(Employee::getLastName)
            .collect(Collectors.joining(", "))
                .toString();
System.out.printf("Last names of Googlers with ids %s: %s.%n", ids, lastNames);
```

* Result <!-- .element: class="fa-code" -->

```java
Last names of Googlers with ids [2, 4, 6, 8]:
Brin, Arora, Pichette, Norvig.
```

--
#### <b>Building Sets</b>
* Code <!-- .element: class="fa-code" -->

```java
List<Employee> googlers = EmployeeSamples.getGooglers();
Set<String> firstNames = googlers.stream().map(Employee::getFirstName)
    .collect(Collectors.toSet());     //Collectors.toSet()
Stream.of("Larry", "Harry", "Peter", "Deiter", "Eric", "Barack")
    .forEach(s -> System.out.printf ("%s is a Googler? %s.%n", s,
        firstNames.contains(s) ? "Yes" : "No"));
```

* Result <!-- .element: class="fa-code" -->

```java
Larry is a Googler? Yes.
Harry is a Googler? No.
Peter is a Googler? Yes.
Deiter is a Googler? No.
Eric is a Googler? Yes.
Barack is a Googler? No.
```

--
#### <b>Building Other Collections</b>
* <b>Big idea</b>
    * <span style="font-size:75%">You provide a Supplier<Collection> to collect. Java takes the resultant Collection
      and then calls “add” on each element of the Stream.</span>

--
* <b>Quick examples (result: "Java, Lisp, Ruby")</b>
    * <span style="font-size:75%">ArrayList</span>
        <pre>
            someStream.collect(toCollection(ArrayList::new))
        </pre>
    * <span style="font-size:75%">TreeSet</span>
        <pre>
            someStream.collect(toCollection(TreeSet::new))
        </pre>
    * <span style="font-size:75%">Stack</span>
        <pre>
            someStream.collect(toCollection(Stack::new))
        </pre>
    * <span style="font-size:75%">Vector</span>
        <pre>
            someStream.collect(toCollection(Vector::new))
        </pre>

Note:
For brevity, the examples here assume you have done
“import static java.util.stream.Collectors.*”,
so that toCollection(…) really means Collectors.toCollection(…).

--
#### <b>Building Other Collections: TreeSet</b>
* Code <!-- .element: class="fa-code" -->

```java
TreeSet<String> firstNames2 = googlers.stream()
    .map(Employee::getFirstName)
        .collect(Collectors.toCollection(TreeSet::new)); //Collectors.toCollection
Stream.of("Larry", "Harry", "Peter", "Deiter", "Eric", "Barack")
    .forEach(s -> System.out.printf("%s is a Googler? %s.%n",
        s, firstNames2.contains(s) ? "Yes" : "No"));
```

* Result <!-- .element: class="fa-code" -->

```java
Larry is a Googler? Yes.
Harry is a Googler? No.
Peter is a Googler? Yes.
Deiter is a Googler? No.
Eric is a Googler? Yes.
Barack is a Googler? No.
```

--
#### <b>partioningBy: Building Maps</b>
* <b>Big idea</b>
    * <span style="font-size:75%">You provide a Predicate. It builds a Map where true maps to a List of entries that
      passed the Predicate, and false maps to a List that failed the Predicate.</span>
* <b>Quick example</b>
    * <span style="font-size:75%">ArrayList</span>
        <pre>
            Map<Boolean,List<Employee>> oldTimersMap =
            employeeStream().collect(<span style="color:red">partitioningBy(e -> e.getEmployeeId() < 10)</span>);
        </pre>
    * <span style="font-size:75%">Now, oldTimersMap.get(true) returns a List<Employee> of employees whose ID’s are
      less than 10, and oldTimersMap.get(false) returns a List<Employee> of everyone else.</span>

--
#### <b>partitioningBy: Example</b>

* Code <!-- .element: class="fa-code" -->

```java
Map<Boolean,List<Employee>> richTable =
    googlers.stream()
        .collect(partitioningBy(e -> e.getSalary() > 1_000_000)); //partitioningBy
System.out.printf("Googlers with salaries over $1M: %s.%n", richTable.get(true));
System.out.printf("Destitute Googlers: %s.%n", richTable.get(false));
```

* Result <!-- .element: class="fa-code" -->

```java
Googlers with salaries over $1M: [Larry Page [Employee#1 $9,999,999],
Sergey Brin [Employee#2 $8,888,888], Eric Schmidt [Employee#3 $7,777,777],
Nikesh Arora [Employee#4 $6,666,666], David Drummond [Employee#5 $5,555,555],
Patrick Pichette [Employee#6 $4,444,444], Susan Wojcicki [Employee#7 $3,333,333]].
Destitute Googlers: [Peter Norvig [Employee#8 $900,000],
Jeffrey Dean [Employee#9 $800,000], Sanjay Ghemawat [Employee#10 $700,000],
Gilad Bracha [Employee#11 $600,000]].
```

--
#### <b>groupingBy: Another Way of Building Maps</b>
* <b>Big idea</b>
    * <span style="font-size:75%">You provide a Function. It builds a Map where each output value of the Function
      maps to a List of entries that gave that value.</span>
        </br><span style="font-size:65%">E.g., if you supply Employee::getFirstName, it builds a Map where supplying a first
        name yields a List of employees that have that first name.</span>

--
* <b>Quick example</b>
    * <span style="font-size:75%">ArrayList</span>
        <pre>
            Map<Department,List<Employee>> deptTable = employeeStream()
                .collect(<span style="color:red">Collectors.groupingBy(Employee::getDepartment)</span>);
        </pre>
    * <span style="font-size:75%">Now, deptTable.get(someDepartment) returns a List<Employee> of everyone in that department.</span>

--
#### <b>groupingBy: Supporting Code</b>
* <b>Idea</b>
    * <span style="font-size:75%">Make a class called Emp that is a simplified Employee that has a first name, last
     name, and office/location name, all as Strings.</span>
* <b>Sample Emps</b>

```java
private static Emp[] sampleEmps = {
   new Emp("Larry", "Page", "Mountain View"),
   new Emp("Sergey", "Brin", "Mountain View"),
   new Emp("Lindsay", "Hall", "New York"),
   new Emp("Hesky", "Fisher", "New York"),
   new Emp("Reto", "Strobl", "Zurich"),
   new Emp("Fork", "Guy", "Zurich"),
};
public static List<Emp> getSampleEmps() {
    return(Arrays.asList(sampleEmps));
}
```

--
#### <b>groupingBy: Example</b>
* Code <!-- .element: class="fa-code" -->

```java
Map<String,List<Emp>> officeTable = EmpSamples.getSampleEmps().stream()
    .collect(Collectors.groupingBy(Emp::getOffice)); //groupingBy
System.out.printf("Emps in Mountain View: %s.%n", officeTable.get("Mountain View"));
System.out.printf("Emps in NY: %s.%n", officeTable.get("New York"));
System.out.printf("Emps in Zurich: %s.%n", officeTable.get("Zurich"));
```

* Result <!-- .element: class="fa-code" -->

```java
Emps in Mountain View:
[Larry Page [Mountain View], Sergey Brin [Mountain View]].
Emps in NY: [Lindsay Hall [New York], Hesky Fisher [New York]].
Emps in Zurich: [Reto Strobl [Zurich], Fork Guy [Zurich]].
```

---
## <b>Summary</b>
<div style="float:left; text-align:left">
    <b>Parallel streams</b></br>
    <span style="margin-left:1em; font-size:75%">- anyStream.<span style="color:red">parallel()</span>.normalStreamOps(…)</span></br>
        <span style="margin-left:3em; font-size:65%">- For reduce, be sure there is no global data and that operator is associative</span></br>
        <span style="margin-left:3em; font-size:65%">- Test to verify answers are the same both ways, or (with doubles) at least close enough</span></br>
        <span style="margin-left:3em; font-size:65%">- Compare timing</span></br>
    <b>Infinite (really unbounded) streams</b></br>
    <span style="margin-left:1em; font-size:75%">- Stream.generate(someStatelessSupplier).limit(…)</span></br>
    <span style="margin-left:1em; font-size:75%">- Stream.generate(someStatefullSupplier).limit(…)</span></br>
    <span style="margin-left:1em; font-size:75%">- Stream.iterate(seedValue, operatorOnSeed).limit(…)</span></br>
    <b>Fancy uses of collect</b></br>
    <span style="margin-left:1em; font-size:75%">- You can build many collection types from streams</span>
</div>

---
## <b>QUESTIONS?</b>