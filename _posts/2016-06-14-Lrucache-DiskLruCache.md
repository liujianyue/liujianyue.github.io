---
layout: post_layout
title: 解析Android攻城狮几个必须要读的源码(二)
time: 2016年06月14日
location: 北京
pulished: true
excerpt_separator: "~~~"
---

今天总结LruCache吧，因为时下最热的图片加载库picasso图片缓存也使用了LruCache，不过我看了一下源码，有改动哦。并且总结它是为了以后总结DiskLruCache，光看代码量，后者是前者的两倍，不过这对于庞大的动不动一个类就几千行代码的Android源码来说，简直是九牛一毛。废话不多说没直接主题。读LruCache源码，我觉得从他的构造函数接着在是常用方法来分析，这样思路清楚。

## 构造函数

    /**
     * @param maxSize for caches that do not override {@link #sizeOf}, this is
     *     the maximum number of entries in the cache. For all other caches,
     *     this is the maximum sum of the sizes of the entries in this cache.
     */
    public LruCache(int maxSize) {
        if (maxSize <= 0) {
            throw new IllegalArgumentException("maxSize <= 0");
        }
        this.maxSize = maxSize;
        /*
         * 构造一个带指定初始容量、加载因子和排序模式的空 LinkedHashMap 实例。 
		        参数：
		        initialCapacity - 初始容量
		        loadFactor - 加载因子
		        accessOrder - 排序模式 - 对于访问顺序，为 true；对于插入顺序，则为 false 
		        抛出： 
		        IllegalArgumentException - 如果初始容量为负或者加载因子为非正
         */
		 this.map = new LinkedHashMap<K, V>(0, 0.75f, true);
    }
    
~~~这是我们使用LruCache时最常使用额构造方法，我们可以清楚的看到其中不仅可以指定我们缓存需要的空间大小，并且实际的缓存机制是对LinkedHashMap的封装。看到“linked” 字眼，不用犹豫，这肯定是带有链表性质的集合类。LinkedHashMap<K,V> 继承 HashMap<K,V> 并实现了implements Map<K,V> 接口，可以这么说它是对HashMap的重新封装，创造除了带有链表性质的集合。链表什么性质，高效的插入删除和排序操作。我们再来联想一下缓存的特点，缓存由于空间大小有限，所以会时刻保持最新访问的内容，当控件不足使，剔除最不常访问的内容，这个特点也正好复合LinkedHashMap的特性。
好，我们接着分析LinkedHashMap的构造函数，initialCapacity代表其初始的容量，刚开始其中为空所以为0；loadFactor 翻译为负载因子，它是做什么的呢？我们知道，析LinkedHashMap为元素的查找提供了很好的支持，但是LinkedHashMap的越大，我们的查找时间越长，为了权衡空间大小与查找时间，Oracle的设计们时间和空间成本上做了折衷：增大负载因子可以减少 Hash 表（就是那个 Entry 数组）所占用的内存空间，但会增加查询数据的时间开销，而查询是最频繁的的操作（HashMap 的 get() 与 put() 方法都要用到查询）；减小负载因子会提高数据查询的性能，但会增加 Hash 表所占用的内存空间。估计这还不能说服你，这根本也没说服我，因为我既没有弄清楚负载因子的工作工程，并且也提出了质疑：负载因子越小，HashMap的利用空间越少，这不是空间上的浪费吗？我们最好还是不去纠结这个。还有有一个参数accessOrder，如果表示是true表示LinkedHashMap以后元素将是按照访问的顺序排序，否则将是按照插入的顺序排序，因为要实现缓存的效果，所以这个值要设置成true。
    
看一下LruCache的最常用法：
    
    private LruCache<String, Bitmap> mMemoryCache; 
    @Override 
    protected void onCreate(Bundle savedInstanceState) { 
        ...  
        // Get max available VM memory, exceeding this amount will throw an   
        // OutOfMemory exception. Stored in kilobytes as LruCache takes an   
        // int in its constructor.   
        final int maxMemory = (int) (Runtime.getRuntime().maxMemory() / 1024);    
        // Use 1/8th of the available memory for this memory cache.  
        final int cacheSize = maxMemory / 8;    
        mMemoryCache = new LruCache<String, Bitmap>(cacheSize) {    
            @Override   
            protected int sizeOf(String key, Bitmap bitmap) {      
            // The cache size will be measured in kilobytes rather than     
            // number of items.    
            return bitmap.getByteCount() / 1024;     
            }     
        };  
        ... 
    }
    
cacheSize 计算方法几乎都是这个样子，而sizeOf（） 方法也几乎使我们必须要重写的，源码中，直接返回1，其实它的意思是，没添加一组键值对，都会占用总大小的1/N，但是我们保存的value 是以空间大小的计算的，这样更加准确，这里需要注意cacheSize的单位和sizeOf（）返回值的单位必须一致，是KB或是B等.以上就是构造函数的解析，接着我们看一下mMemoryCache的两个最常用的函数，put(),get().
    
使用方法：    

		public void addBitmapToMemoryCache(String key, Bitmap bitmap) {   
          if (getBitmapFromMemCache(key) == null) {     
          mMemoryCache.put(key, bitmap);  
          }
        }  
        public Bitmap getBitmapFromMemCache(String key) {   
        	return mMemoryCache.get(key); 
     	}  
    
get()源码：

    /**
     * Returns the value for {@code key} if it exists in the cache or can be
     * created by {@code #create}. If a value was returned, it is moved to the
     * head of the queue. This returns null if a value is not cached and cannot
     * be created.
     */
    public final V get(K key) {
        if (key == null) {
            throw new NullPointerException("key == null");
        }

        V mapValue;
        synchronized (this) {
            mapValue = map.get(key);
            if (mapValue != null) {
                hitCount++;
                return mapValue;
            }
            missCount++;
        }

        /*
         * Attempt to create a value. This may take a long time, and the map
         * may be different when create() returns. If a conflicting value was
         * added to the map while create() was working, we leave that value in
         * the map and release the created value.
         */

        V createdValue = create(key);
        if (createdValue == null) {
            return null;
        }

        synchronized (this) {
            createCount++;
            mapValue = map.put(key, createdValue);

            if (mapValue != null) {
                // There was a conflict so undo that last put
                map.put(key, mapValue);
            } else {
                size += safeSizeOf(key, createdValue);
            }
        }

        if (mapValue != null) {
            entryRemoved(false, key, createdValue, mapValue);
            return mapValue;
        } else {
            trimToSize(maxSize);
            return createdValue;
        }
    }
  
    
首先判断键Key是否为空，为空则抛出NullPointerException("key==null")，这个没有疑问；接着定义了一个V类型的mapValue，在使用synchronized关键字保证对LinkedHashMap访问的同步，试图去获得mapValue，如果mapValue不为空说明获取成功，hiCount加1，表示我们成功读取了一次值，并返回值，如果为空missCount加1，表示获取失败或键值对不存在的次数加1.当没有成功获取到值的时候，它接下来会使用create()函数去创建一个value，这里我们很多情况下并没有重写create()方法，这其实为我们手动创建一个value提供了扩展性，源码中返回null，多以一般也会返回null；倘若我们的确重写了create，那么接下仍然同步操作，createCount++，表示人为创建的value的个数，接下来，他会试图将创建的的createdValue保存到LinkedHashMap中，map.put(key,createdValue)如返回一个值不为空，这说明之前已经包含了键为key的value，然后把刚才插入的createdValue剔除，换位之前的值，如果为空，表示插入成功，size的大小增加。接下来当mapValue不为空即插入失败时，有一个entryRemoved(false, key, createdValue,mapValue)，倘若你想要对上述情况做处理，那你需要重写这个函数，源码中为空实现。倘若mapValue为空，则需要做极为重要的事：trimToSize(maxSize)。
    
    /**
         * Remove the eldest entries until the total of remaining entries is at or
         * below the requested size.
         *
         * @param maxSize the maximum size of the cache before returning. May be -1
         *            to evict even 0-sized elements.
         */
        public void trimToSize(int maxSize) {
            while (true) {
                K key;
                V value;
                synchronized (this) {
                    if (size < 0 || (map.isEmpty() && size != 0)) {
                        throw new IllegalStateException(getClass().getName()
                                + ".sizeOf() is reporting inconsistent results!");
                    }
                    if (size <= maxSize) {
                        break;
                    }
                    Map.Entry<K, V> toEvict = map.eldest();
                    if (toEvict == null) {
                        break;
                    }
                    key = toEvict.getKey();
                    value = toEvict.getValue();
                    map.remove(key);
                    size -= safeSizeOf(key, value);
                    evictionCount++;
                }

                entryRemoved(true, key, value, null);
            }
        }
    
 这段代码是把最新的键值对保存下来并把老旧的值剔除，并且他还有另外一个功能，就是通过evictAll() 或trimToSize(-1)清空map.

put源码：

      /**
       * Caches {@code value} for {@code key}. The value is moved to the head of
       * the queue.
       *
       * @return the previous value mapped by {@code key}.
       */
      public final V put(K key, V value) {
          if (key == null || value == null) {
              throw new NullPointerException("key == null || value == null");
          }

          V previous;
          synchronized (this) {
              putCount++;
              size += safeSizeOf(key, value);
              previous = map.put(key, value);
              if (previous != null) {
                  size -= safeSizeOf(key, previous);
              }
          }

          if (previous != null) {
              entryRemoved(false, key, previous, value);
          }

          trimToSize(maxSize);
          return previous;
      }


首先它会判断key ，value两者是否为空，为空则抛出空指针异常，接着声明一个V类型的previous，接着仍然是在同步块中执行插入操作，size增加，并把返回值付给previous，加入previous不为空仍然说已经有一个key相同的键值对存在，size需要复原，在此像get中供用户处理这种情况一样，通过entryRemoved(false,key,previous,value)函数做处理，同时更新缓存中内容的新旧度，返回值。

源码中不仅提供了put get 方法也提供了remove方法，它的原理很简单，不再赘述。下一篇我们将要总结一DiskLruCache哦，好兴奋。
