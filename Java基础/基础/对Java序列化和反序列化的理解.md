
# 什么是序列化和反序列化

序列化：就是将对象转化成字节序列的过程。

反序列化：就是将字节序列转化成对象的过程。

对象序列化成的字节序列会包含描述这个对象的所有信息，比如对象的类型信息、对象的数据等，根据这些信息能“复刻”出一个和原来一模一样的对象。


# 为什么要序列化

* **持久化**：对象是存储在JVM中内存中的（堆区），但是如果JVM停止运行了，对象也不存在了。序列化可以将对象转化成字节序列，然后写进硬盘文件中实现持久化。在新开启的JVM中可以将读取到的字节序列反序列化成对象。  
* **网络传输**：网络可以直接传输数据，但是无法直接传输对象。可在传输前进行序列化，传输完成后反序列化成对象。所以所有可在网络上传输的对象都必须是可序列化的。

## 什么场景需要序列化

* 当你想把的内存中的对象状态保存到一个文件中或者数据库中时候。
* 当你想用套接字在网络上传送对象的时候。
* 当你想通过RMI传输对象的时候。  

# 如何实现序列化和反序列化

1. 对于要序列化对象的类要去实现 `Serializable` 接口或者 `Externalizable` 接口（Serializable和Externalizable都是标记接口，不包含任何方法）  
2. JDK提供的ObjectOutputStream和ObjectInputStream来实现序列化和反序列化（对象序列化是基于字节的，因此使用InputStream和OutputStream继承的类）

## Serializable 接口实现序列化和反序列化

定义需要序列化的类

```java
public class TestBean implements Serializable {

    private Integer id;

    private String name;

    private Date date;
    //省去getter和setter方法和toString
}
```

序列化

```java
public static void main(String[] args) {
    TestBean testBean = new TestBean();
    testBean.setDate(new Date());
    testBean.setId(1);
    testBean.setName("zll1");
    //使用ObjectOutputStream序列化testBean对象并将其序列化成的字节序列写入test.txt文件
    try (FileOutputStream fileOutputStream = new FileOutputStream("test.txt");
         ObjectOutputStream objectOutputStream = new ObjectOutputStream(fileOutputStream);) {
        objectOutputStream.writeObject(testBean);
    } catch (FileNotFoundException e) {
        e.printStackTrace();
    } catch (IOException e) {
        e.printStackTrace();
    }
}
```

反序列化 test.txt 文件的内容

```java
public static void main(String[] args) {
    try (FileInputStream fileInputStream = new FileInputStream("D:\\test.txt");
         ObjectInputStream objectInputStream=new ObjectInputStream(fileInputStream)) {
        TestBean testBean = (TestBean) objectInputStream.readObject();
        System.out.println(testBean);
    } catch (FileNotFoundException e) {
        e.printStackTrace();
    } catch (IOException e) {
        e.printStackTrace();
    } catch (ClassNotFoundException e) {
        e.printStackTrace();
    }
}
```

> **注意：**
> 1. 一个对象要进行序列化，如果该对象成员变量是引用类型的，那这个引用类型也一定要是可序列化的，否则会报错
> 2. 同一个对象多次序列化成字节序列，这多个字节序列通过反序列化成的对象还是同一个（使用==判断为true）。因为所有序列化保存的对象都会生成一个序列化编号，当再次序列化时，会去检查此对象是否已经序列化了，如果是，那序列化只会输出上个序列化的编号。
> 3. 如果序列化一个可变对象，序列化之后，修改对象属性值，再次序列化，只会保存上次序列化的编号（这是个坑注意下）
> 4. 对于不想序列化的字段可以再字段类型之前加上 `transient` 关键字修饰（反序列化时会被赋予默认值）

## serialVersionUID的作用

在进行序列化时，会把当前类的 serialVersionUID 写入到字节序列中（也会写入序列化的文件中），在反序列化时会将字节流中的 serialVersionUID 同本地对象中的 serialVersionUID 进行对比，一致的话进行反序列化，不一致则失败报错（报InvalidCastException异常）

serialVersionUID的生成有三种方式（private static final long serialVersionUID= XXXL ）：    
* 显式声明：默认的1L
* 显式声明：根据包名、类名、继承关系、非私有的方法和属性以及参数、返回值等诸多因素计算出的64位的hash值
* 隐式声明：未显式的声明serialVersionUID时java序列化机制会根据Class自动生成一个serialVersionUID（最好不要这样，因为如果Class发生变化，自动生成的serialVersionUID可能会随之发生变化，导致匹配不上）

序列化类增加属性时，最好不要修改serialVersionUID，避免反序列化失败

# 其他序列化方式

其实对于对象转化成json字符串和json字符串转化成对象，也是属于序列化和反序列化的范畴，相对于JDK提供的序列化机制，各有各的优缺点：

* JDK序列化/反序列化：原生方法不依赖其他类库、但是不能跨平台使用、字节数较大
* json序列化/反序列化：json字符串可读性高、可跨平台使用无语言限制、扩展性好、但是需要第三方类库、字节数较大

