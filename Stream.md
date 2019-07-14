# Stream
Stream提供了数据视图，让人可以在比集合更高的概念层上进行操作。使用流，只需要指定做什么，而不是怎么做。

## 流的创建
```
		  List<String> words = new ArrayList<>();
        for (int i = 0; i < 100; i ++) {
            words.add(i + "");
        }
        words.stream().forEach(v -> Log.d("test", v));
```

方法stream()是生成串行流，比如上面的例子中，串行的打印数据
生成并行流的的方法如下:
```
words.parallelStream()
```
这样可以并行的处理数据。
```
Stream<String> words = Stream.of("i", "am", "summer");
Stream.generate(() -> "aa");
```
创建空的流：
```
Stream<String> words = Stream.empty();
```

## API
forEach方法
```
words.stream().forEach(v -> Log.d("test", v));
words.parallelStream().forEach(v -> Log.d("test", v));
```

filter方法，filter参数是一个Predicate<T>对象，也就是说从T到boolean的函数。
```
long count = words.stream().filter(w -> w.length() > 3).count();
words.parallelStream().filter(w -> w.length() > 3).count();
```

map方法，函数将作用于每一个元素，这样就产生了一个包含最终结果的新流。flapMap方法，可以将结果展开。
```
words.stream().map(String::toLowerCase);
```

limit(n)方法，返回一个包含n个元素的新流。
```
Object[] powers = Stream.iterate(1.0, p -> p * 2).peek(e -> Log.d("zdz", e + "")).limit(10).toArray();
```

skip(n)方法，会丢弃前n个元素。
```
words.stream().skip(1);
```

concat方法，将两个流连接起来。
```
Stream.concat(words1, words2);
```

distinct方法，会根据原始流中的元素返回一个具有相同顺序、去除了重复元素的新流。

sorted方法，生成一个其元素是原先流中元素按照排序顺序重新排列的新流。

peek方法，生成一个与原先流有着相同元素的另一个流。

## 归约
归约是终止操作，它们可以将流归约为一个可以在程序中使用的非流值。
归约方法，具备此功能的方法，count方法，max/min方法
findFirst方法，返回非空集合中的第一个值。
findAny方法，返回任何一个匹配的元素，如果有。
anyMatch方法，接受一个predicate参数，返回boolean值，true表示流中含有匹配的元素。

## Optional类型
Optional<T>对象是一个T类型对象或者空对象的封装。
有值则返回值；否则，则返回默认值，或调用代码来计算默认值，或抛出异常。
```
Optional<String> optionalS;
optionalS.orElse("");

optionalS.orElseGet(() -> System.getProperty("user.dir"));

optionalS.orElseThrow(IllegalStateException::new);
```

## 收集数据
toArray()方法，返回一个Object[]类型数组；stream.toArray(String[]::new)则返回String[]类型数据。

### Collectors类
collect方法，接受一个Collector接口的实例。Collectors类为普通集合提供了大量工厂方法。

```
Map<Integer, String> idToName = people.collect(Collectors.toMap(Person::getId, Person::getName));
```
toMap方法、groupingBy方法、partitioningBy方法