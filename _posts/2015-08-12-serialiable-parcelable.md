---
layout: post_layout
title: Activity 之间的数据传递之Serializable、Parcelable
time: 2015年08月12日
location: 北京
pulished: true
excerpt_separator: "Syntax"
---

Activity 之间的跳转，往往要附带专递各种数据，大多数情况下，我们值传递一些简单的基本类型，如int、String 等，但是在实际应用中我们也会传递一些类对象，他们往往夹带着比较大的数据，比如 bitmap，list 等，那我们是用什么？答案是 Serializable和Parcelable。

Intent 都支持对这两种数据形式的专递，先看一下基本用法：

**Serializable**

    import java.io.Serializable;
    public class SerializablePerson implements Serializable {
        private static final long serialVersionUID = -7060210544600464481L; 
        private String name;
        private int age;
        public String getName() {
            return name;
        }
        public void setName(String name) {
            this.name = name;
        }
        public int getAge() {
            return age;
        }
        public void setAge(int age) {
            this.age = age;
        }
    }

还是熟悉的味道还是熟悉的用法，serialVersionUID： Java的序列化机制是通过在运行时判断类的serialVersionUID来验证版本一致性的。在进行反序列化时，JVM会把传来的字节流中的serialVersionUID与本地相应实体（类）的serialVersionUID进行比较，如果相同就认为是一致的，可以进行反序列化，否则就会出现序列化版本不一致的异常(InvalidCastException)。serialVersionUID 这个值即可自己定义值，也可以让java通过计算为你生成唯一的一个值。

**Parcelable**

        public class ParcelablePerson implements Parcelable {
         private String name;
        private int age;

         public int describeContents() {
             return 0;
         }

         public void writeToParcel(Parcel out, int flags) {
             out.writeInt(age);
             out.writeString(name);
         }

         public static final Parcelable.Creator<ParcelablePerson> CREATOR
                 = new Parcelable.Creator<ParcelablePerson>() {
             public ParcelablePerson createFromParcel(Parcel in) {
                 return new ParcelablePerson(in);
             }

             public ParcelablePerson[] newArray(int size) {
                 return new ParcelablePerson[size];
             }
         };

         private ParcelablePerson(Parcel in) {
             age = in.readInt();
             name = in.readString();
         }
    }

Parcelable 使用格式基本固定，你可以使用网上的一些库来自动为你生成这些代码，另外需要注意的是 Parcelable 读出和读入的顺序是一致的，并且当Parcelable 中包含list 的集合时，在读出时应该按如下如下操作，否则会报空指针异常的错哦：

    list = new ArrayList<String>();
    in.readStringList(list);	
    
## 两者的区别

那么既然Android 使用java开发，java中对于Serializable的使用有很好的支持，那为什么Android还要引入Parcelable，那就不得不谈谈两者的区别了：Serializalbe使用反射，序列化和反序列化过程需要大量I/O操作，Parcelable自已实现封送和解封（marshalled &unmarshalled）操作不需要用反射，数据也存放在Native内存中，效率要快很多。你也可以认为两者的区别在于使用场景的不同：Serializable的作用是为了保存对象的属性到本地文件、数据库、网络流、rmi以方便数据传输，当然这种传输可以是程序内的也可以是两个程序间的。而Android的Parcelable的设计初衷是因为Serializable效率过慢，为了在程序内不同组件间以及不同Android程序间(AIDL)高效的传输数据而设计，这些数据仅在内存中存在，Parcelable是通过IBinder通信的消息的载体。Parcelable的性能比Serializable好，在内存开销方面较小，所以在内存间数据传输时推荐使用Parcelable，如activity间传输数据，而Serializable可将数据持久化方便保存，所以在需要保存或网络传输数据时选择Serializable，因为android不同版本Parcelable可能不同，所以不推荐使用Parcelable进行数据持久化，举一个例子，之前小米手机 在MIUI 5 上把联系人等备份，然后手机升级到MIUI 6，发现联系人不能很好的恢复，臆断有可能是使用了Parcelable持久化了。
那么既然谈到了Parcelable，我觉得有必要顺带说一下Parcle：

如果说Parcelable 是提供开发人员的上层java接口，那Parcle就是Android底层parcelable的内部的实现方式，它的绝大部分方法都是native的方法，提供了最快速的序列化和反序列化的方法，这也是为什么Parcelable 在Android使用比Serializable更高效的原因。	如果感兴趣的话，大家可以阅读一下Parcel的源码哦。

**PS**使用Serializable、Parcelable都是存在一定风险的WHY？因为intent 的Bundle在数据传输过程中是有大小限制的，Bundle使用Binder机制传递数据，而Binder 存在于Android系统的底层，他有自己固定大小的缓冲区，	在Android源码的framework\base\libs\binder\ProcessState.cpp 文件中定义了它的大小：

    #define BINDER_VM_SIZE ((1*1024*1024) - (4096 *2))

不过这个数值很多android 手机厂商修改过（不建议修改，最好优化自己的应用），但是范围一般在1M在2M之间，而一个进程默认有15个binder线程，这个同样在framework\base\libs\binder\ProcessState.cpp 文件可以找到：

        static int open_driver()
    {
        int fd = open("/dev/binder", O_RDWR);
        if (fd >= 0) {
            fcntl(fd, F_SETFD, FD_CLOEXEC);
            int vers = 0;
            status_t result = ioctl(fd, BINDER_VERSION, &vers);
            if (result == -1) {
                ALOGE("Binder ioctl to obtain version failed: %s", strerror(errno));
                close(fd);
                fd = -1;
            }
            if (result != 0 || vers != BINDER_CURRENT_PROTOCOL_VERSION) {
                ALOGE("Binder driver protocol does not match user space protocol!");
                close(fd);
                fd = -1;
            }
            size_t maxThreads = 15;//最多15个binder线程
            result = ioctl(fd, BINDER_SET_MAX_THREADS, &maxThreads);
            if (result == -1) {
                ALOGE("Binder ioctl to set max threads failed: %s", strerror(errno));
            }
        } else {
            ALOGW("Opening '/dev/binder' failed: %s\n", strerror(errno));
        }
        return fd;
    }

所分配到每一个binder线程缓冲区大小也不会很大，我在其他博客中看到有说具体数值的，逼着并不这样认为，因为binder缓冲池为进程中所binder线程共同使用，它的大小依赖于当前缓冲池可分配的大小，应该是一个范围。所以在使用bundle传递数据时应对数据进行优化，避免传递过大数据导致异常发生：TransactionTooLargeException：The Binder transaction failed because it was too large.。

Serializable和Parcelable 就先总结到这吧，每次写一篇文章都会联想起大量的其他东东，不写了，否则一会不知道跑哪去了。