# 知识点小demo  


* 字符串常量池放的是运行时常量池还是堆   
```java
/*
-XX:+PrintGCDetails
-Xms50M
-Xmx100M
-Xmn5M
-XX:SurvivorRatio=8
-XX:MetaspaceSize=10m
-XX:MaxMetaspaceSize=10m
-XX:+UseCompressedClassPointers
-XX:+UseCompressedOops
-XX:CompressedClassSpaceSize=5M
-XX:-UseGCOverheadLimit
*/
       ArrayList<String> list = new ArrayList<>();
        while(true){
            String str= (UUID.randomUUID().toString() + System.currentTimeMillis()).intern();
            list.add(str);
        }
```

* java循环引用
```java
//        A a = new A();
//        B b = new B();
//        a.b = b;
//        b.a = a;
```

* 强引用、软引用、弱引用、虚引用
```java
        //强引用
        Object o = new Object();

       //软引用
        //-Xmx200m -XX:+PrintGC
        byte[] data = new byte[100*1024*1024];
        Object o = new Object();
        System.out.println(o);
        SoftReference<Object> softReference = new SoftReference<Object>(o);
        o=null;
        System.out.println("gc前:" + softReference.get());

        System.gc();
        Thread.sleep(2000);

        byte[] data1 = new byte[40*1024*1024];
        System.out.println("gc后:" + softReference.get());

        //弱引用
        Object o2 = new Object();
        System.out.println(o2);
        WeakReference<Object> weakReference = new WeakReference<>(o2);
        o2 = null;
        System.out.println(weakReference.get());
        System.gc();
        //Thread.sleep(2000);
        //很难捕捉到
        System.out.println("是否被标记:" + weakReference.isEnqueued());
        System.out.println(weakReference.get());

        //虚引用
        ReferenceQueue<Object> referenceQueue = new ReferenceQueue<>();
        Object o3 = new Object();
        PhantomReference<Object> phantomReference = new PhantomReference<>(o3,referenceQueue);
        System.out.println("垃圾回收前:" + phantomReference.isEnqueued());
        System.gc();
        Thread.sleep(2000);
        System.out.println("垃圾回收前:" + phantomReference.isEnqueued());
```

* 垃圾回收调用覆盖finalize
```java
public class B{
    @Override
    protected void finalize() throws Throwable {
        System.out.println("B回收调用。。。。。。。。");
        super.finalize();
    }

    public static void main(String[] args) throws InterruptedException {
        //虚引用
        B b = new B();
        b = null;
        System.gc();
    }
}

```

* hashCode()
```java
       /*
        -XX:hashCode=n
        hashCode=0，使用系统生成的随机数作为hashcode
        hashCode=1，对对象地址做移位和逻辑操作，生成hashcode(默认)
        hashCode=2，所有的hashcode都等于1
        hashCode=3，用一个自增序列给hashcode赋值
        hashCode=4，以对象地址作为hashcode
         */
        B b = new B();
        B b1 = new B();
        System.out.println(b);
        System.out.println(b.hashCode());
        System.out.println(b1);
        System.out.println(b1.hashCode());
```