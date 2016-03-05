---
layout: post
title: "Hadoop Secondary Sorting"
date: 2015-11-22 17:27
comments: true
category: Big Data
tags: [hadoop, sort]
---

Hadoop MapReduce的神奇之处发生在mapper和reducer之间，将所有相同key的map输出记录聚集在一块，使得用户可以方便的处理聚合在一起的数据。Hadoop内部使用了partition、sort和merge（shuffle的一部分），在每个reducer中流式地得到排序后的key和value集合。在MapReduce Sorting中有个特别的部分是secondary sort，也就是对value进行排序。

<!--more-->

Secondary sort在两种情况下特别有用：

* 需要某一部分的数据比其他数据更快的到达reducer。
* 希望job的输出按照两个key进行排序。

实现Secondary sort需要对MapReduce中的数据流和处理有一定的了解，下图展示了对reducer中出现的数据有影响的三个部分。

<img src="http://i5.tietuku.com/7ad3ad872415c4b6.png" width="600px" />

`partitioner`决定哪个reducer接收该mapper数据记录；`sorting RawComparator`用于在各自的分片中排序输出的结果，map和reduce阶段都有它，其中map阶段的sorting是对reduce阶段sorting的一个优化，让reducer的sorting更高效；最后，`grouping RawComparator`用于决定reducer处理排序后记录的边界，发生在reducer从本地磁盘读取数据的时候，也就是说，你可以用这个方法决定数据记录是如何聚集起来调用一个reduce方法的。MapReduce默认把这个三个方法作用于map方法输出的key上。

要实现Secondary sorting，我们需要重写partitioner、sort comparator和grouping comparator。

下面，通过对人名的排序来说明如何使用Secondary sorting。使用primary sort排序last name，secondary sort排序first name。

我们需要构建一个由map方法输出的Composite key，这个key由两部分组成：

*    Natural Key
*    Secondary Key

<img src="http://i5.tietuku.com/25eedf0319e92775.png" width="430px" />

``` java
public class Person implements WritableComparable<Person> {

  private String firstName;
  private String lastName;

  @Override
  public void readFields(DataInput in) throws IOException {
    this.firstName = in.readUTF();
    this.lastName = in.readUTF();
  }

  @Override
  public void write(DataOutput out) throws IOException {
    out.writeUTF(firstName);
    out.writeUTF(lastName);
  }
...
```

下图说明hadoop框架配置中用于设置partitioning、sorting和grouping类的名字和方法。

<img src="http://i5.tietuku.com/520e7242cd6ecc43.png" width="530px" />

###Partitioner

默认的partitioner使用对key进行hash后取reducer个数的模。但是默认的partitioner使用整个key，会导致相同的natural key发往不同的reducer。所以需要实现自己的partitioner。

``` java
public class PersonNamePartitioner extends
    Partitioner<Person, Text> {

  @Override
  public int getPartition(Person key, Text value, int numPartitions) {
    return Math.abs(key.getLastName().hashCode() * 127) %
        numPartitions;
  }
} 
```

###Sorting

``` java
public class PersonComparator extends WritableComparator {
  protected PersonComparator() {
    super(Person.class, true);
  }

  @Override
  public int compare(WritableComparable w1, WritableComparable w2) {

    Person p1 = (Person) w1;
    Person p2 = (Person) w2;


    int cmp = p1.getLastName().compareTo(p2.getLastName());
    if (cmp != 0) {
      return cmp;
    }

    return p1.getFirstName().compareTo(p2.getFirstName());
  }
}
```

###grouping

grouping阶段所有的数据记录已经是secondary sort了，grouping comparator需要将相同的last name聚合在一起。

``` java
public class PersonNameComparator extends WritableComparator {

  protected PersonNameComparator() {
    super(Person.class, true);
  }

  @Override
  public int compare(WritableComparable o1, WritableComparable o2) {

    Person p1 = (Person) o1;
    Person p2 = (Person) o2;

    return p1.getLastName().compareTo(p2.getLastName());

  }
}
```

###MapReduce

最后在driver中，需要设置上文提到的三个类：

``` java
job.setPartitionerClass(PersonNamePartitioner.class);
job.setSortComparatorClass(PersonComparator.class);
job.setGroupingComparatorClass(PersonNameComparator.class);

public static class Map extends Mapper<Text, Text, Person, Text> {
  private Person outputKey = new Person();

  @Override
  protected void map(Text lastName, Text firstName, Context context)
      throws IOException, InterruptedException {
    outputKey.set(lastName.toString(), firstName.toString());
    context.write(outputKey, firstName);
  }
}

public static class Reduce extends Reducer<Person, Text, Text, Text> {

  Text lastName = new Text();
  @Override
  public void reduce(Person key, Iterable<Text> values,
                     Context context)
      throws IOException, InterruptedException {
    lastName.set(key.getLastName());
    for (Text firstName : values) {
      context.write(lastName, firstName);
    }
  }
}   
```

Secondary sort涉及到的自定义的partitioner、sorter和grouper，还是比较复杂的。可以考虑[htuple](http://htuple.org)对简单类型进行secondary sort。

