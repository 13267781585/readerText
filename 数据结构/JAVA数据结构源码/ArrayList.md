##  ArrayList 源码解读

```java
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable
```

1. RandomAccess 随机访问接口 
* ArrayList 是基于 数组 数据结构实现的，可以通过索引直接访问元素，实现常量级的随机访问，复杂度为O(1)；   
* 相比之下 LinkedList 是基于 链表 数据结构实现的，所以没有实现 RandomAccess 接口；  
* 实现了这个接口 使用 for 遍历列表 比 使用 迭代器性好

2. 创建列表  
* 在创建对象不指定长度默认创建空列表
```java
  /**
     * Constructs an empty list with an initial capacity of ten.
     */
    public ArrayList() {
        this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
    }
    private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};

```


3. 内存扩容  
* 在向列表中添加元素时，会先检查列表中数组是否还有空间，没有则申请一个  原来数组长度 + 1/2 * 原来数组长度 的数组，并把之前的数据拷贝过去。以 add() 函数为例   

```java
 public boolean add(E e) {
     //判断数组空间是否足够
        ensureCapacityInternal(size + 1);  // Increments modCount!!
        elementData[size++] = e;
        return true;
    }
    //系统设定最小扩容量
     private static final int DEFAULT_CAPACITY = 10;

     private void ensureCapacityInternal(int minCapacity) {
         //判断列表是否为系统默认空数组
         //在 minCapacity 和 DEFAULT_CAPACITY中取一个最大数作为申请的长度，这里取最大数有两种情况 1.DEFAULT_CAPACITY小于minCapacity时说明需要的空间大于系统设定的空间大小，应该按照实际需要的申请
         //2. DEFAULT_CAPACITY大于minCapacity，说明实际需要的空间很小，避免频繁扩容，选取系统设定最小扩容量
        if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
            minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
        }

        ensureExplicitCapacity(minCapacity);
    }

    private void ensureExplicitCapacity(int minCapacity) {
        modCount++;

        // overflow-conscious code
        if (minCapacity - elementData.length > 0)
            grow(minCapacity);
    }
    //系统默认最大扩容量
    private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;

       private void grow(int minCapacity) {
        // 原来空间大小
        int oldCapacity = elementData.length;
        //将实际需要的空间大小适量增大，防止多次扩容
        int newCapacity = oldCapacity + (oldCapacity >> 1);
        //判断增大后是否溢出
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
        //需要的空间大小超过数值表示
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
        // minCapacity is usually close to size, so this is a win:
        elementData = Arrays.copyOf(elementData, newCapacity);
    }

      private static int hugeCapacity(int minCapacity) {
          //溢出 抛出异常
        if (minCapacity < 0) // overflow
            throw new OutOfMemoryError();
        return (minCapacity > MAX_ARRAY_SIZE) ?
            Integer.MAX_VALUE :
            MAX_ARRAY_SIZE;
    }
```

4. 数值复制函数  
*  public static native void arraycopy(Object src,  int  srcPos,
                                        Object dest, int destPos,
                                        int length); 
   浅复制(值复制)

* Arrays.copyOf
    浅复制(值复制)

5. ArrayList 是线程不安全的，在多线程的情况下不能使用该结构。

6. ArrayList 中有一些利用 接口函数式 参数进行操作简化代码的函数   
i.  forEach
```java
 @Override
    public void forEach(Consumer<? super E> action) {
        Objects.requireNonNull(action);

        final int expectedModCount = modCount;

        @SuppressWarnings("unchecked")
        final E[] elementData = (E[]) this.elementData;
        final int size = this.size;

        for (int i=0; modCount == expectedModCount && i < size; i++) {
            action.accept(elementData[i]);
        }
        
        if (modCount != expectedModCount) {
            throw new ConcurrentModificationException();
        }
    }
```

* 测试
```java
  @Test
    public void test2(){
        ArrayList<Integer> list = new ArrayList<>();
        list.add(1);
        list.add(3);
        list.add(2);
        list.forEach((i)->{i += 1;});
        list.forEach(System.out::println);
    }

//结果
1
3
2
```

ii. ArrayList内部迭代器Itr的forEachRemaining->用于遍历迭代器还未遍历的元素
```java
@Override
//modCount记录了列表修改的次数，用于判断是否发生并发异常
        int expectedModCount = modCount;
        @SuppressWarnings("unchecked")
        public void forEachRemaining(Consumer<? super E> consumer) {
            Objects.requireNonNull(consumer);
            //数组长度
            final int size = ArrayList.this.size;
            //迭代器当前指向元素索引
            int i = cursor;
            if (i >= size) {
                return;
            }
            final Object[] elementData = ArrayList.this.elementData;
            //发生并发异常
            if (i >= elementData.length) {
                throw new ConcurrentModificationException();
            }
            while (i != size && modCount == expectedModCount) {
                consumer.accept((E) elementData[i++]);
            }
            // update once at end of iteration to reduce heap write traffic
            cursor = i;
            lastRet = i - 1;
            checkForComodification();
        }
```
ArrayList是不适合在并发的环境中使用的，该类中迭代器也利用modCount(记录列表修改次数)做简单的判断识别并抛出并发异常。

```java
//测试
  @Test
    public void test3() throws InterruptedException {
        ArrayList<Integer> list = new ArrayList<>();
        list.add(1);
        list.add(3);
        list.add(2);
        list.add(4);

        //不停地加入元素
        Thread t1 = new Thread(()->{for(int i = 0; i < 1000; i++)list.add(i);});
        //利用迭代器遍历
        Thread t2 = new Thread(()->list.iterator().forEachRemaining(System.out::println));
        t1.start();
        t2.start();
        t1.join();;
        t2.join();
    }

//结果->抛出异常
Exception in thread "Thread-1" java.util.ConcurrentModificationException
	at java.util.ArrayList$Itr.checkForComodification(ArrayList.java:901)
	at java.util.ArrayList$Itr.forEachRemaining(ArrayList.java:896)
	at com.springboottest.DataStructTest.lambda$test3$1(DataStructTest.java:74)
	at java.lang.Thread.run(Thread.java:748)

```
iii. replaceAll 自定义替换元素
```java
 @Override
    @SuppressWarnings("unchecked")
    public void replaceAll(UnaryOperator<E> operator) {
        Objects.requireNonNull(operator);
        final int expectedModCount = modCount;
        final int size = this.size;
        for (int i=0; modCount == expectedModCount && i < size; i++) {
            //自定义函数处理元素并返回新的元素
            elementData[i] = operator.apply((E) elementData[i]);
        }
        if (modCount != expectedModCount) {
            throw new ConcurrentModificationException();
        }
        modCount++;
    }
```

iv. removeIf  自定义函数删除满足条件元素
```java
@Override
    public boolean removeIf(Predicate<? super E> filter) {
        Objects.requireNonNull(filter);
        // figure out which elements are to be removed
        // any exception thrown from the filter predicate at this stage
        // will leave the collection unmodified
        int removeCount = 0;
        //存放符合删除条件的元素
        final BitSet removeSet = new BitSet(size);
        final int expectedModCount = modCount;
        final int size = this.size;
        for (int i=0; modCount == expectedModCount && i < size; i++) {
            @SuppressWarnings("unchecked")
            final E element = (E) elementData[i];
            if (filter.test(element)) {
                removeSet.set(i);
                removeCount++;
            }
        }
        if (modCount != expectedModCount) {
            throw new ConcurrentModificationException();
        }

        // shift surviving elements left over the spaces left by removed elements
        //判断是否有需要删除的元素
        final boolean anyToRemove = removeCount > 0;
        if (anyToRemove) {
            final int newSize = size - removeCount;
            for (int i=0, j=0; (i < size) && (j < newSize); i++, j++) {
                i = removeSet.nextClearBit(i);
                elementData[j] = elementData[i];
            }
            for (int k=newSize; k < size; k++) {
                elementData[k] = null;  // Let gc do its work
            }
            this.size = newSize;
            if (modCount != expectedModCount) {
                throw new ConcurrentModificationException();
            }
            modCount++;
        }

        return anyToRemove;
    }
```

7. ArrayList 中的迭代器 
* private class Itr implements Iterator<E>    
比较简单，提供了简单的遍历相关操作  
```java
//是否还有元素
public boolean hasNext();

//返回下一个元素，没有抛出NoSuchElementException()异常
public E next();

//删除元素
public void remove();

//遍历剩下的元素
public void forEachRemaining(Consumer<? super E> consumer);
```

* private class ListItr extends Itr implements ListIterator<E>  
在 Itr 的基础上增加了一些操作方法
```java
//是否有前一个元素
public boolean hasPrevious();

//下一个元素的索引
public int nextIndex();

//前一个元素的索引
public int previousIndex();

//返回前一个元素
public E previous();

//设置最后遍历元素的值
public void set(E e);

//在最后遍历元素后一位增加一个元素
 public void add(E e);
```

8. ArrayList 中实现了一个并行读取的迭代器保证在多线程环境中读取的安全
```java
static final class ArrayListSpliterator<E> implements Spliterator<E>
```

* 测试
```java
@Test
    public void test3() throws InterruptedException {
        ArrayList<Integer> list = new ArrayList<>();
        list.add(1);
        list.add(5);
        list.add(2);
        list.add(4);
        Spliterator<Integer> sr = list.spliterator();

        Thread t1 = new Thread(()->sr.tryAdvance(System.out::println));
        Thread t2 = new Thread(()->sr.tryAdvance(System.out::println));
        Thread t3 = new Thread(()->sr.tryAdvance(System.out::println));
        Thread t4 = new Thread(()->sr.tryAdvance(System.out::println));

        t1.start();
        t2.start();
        t3.start();
        t4.start();
        t1.join();;
        t2.join();
        t3.join();;
        t4.join();
    }

//结果
1
5
2
4
```

9. Vector和ArrayList实现类时，区别是Vector在方法前加了 synchronized 限制，是线程安全的。