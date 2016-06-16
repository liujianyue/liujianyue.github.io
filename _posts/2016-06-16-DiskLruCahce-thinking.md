---
layout: post_layout
title: 解析Android攻城狮几个必须要读的源码(三)
time: 2016年06月16日
location: 北京
pulished: true
excerpt_separator: "~~~"
---

前两天总结了一下LruCache，很简单。现在很多应用，尤其是实时拉取内容的应用大多都会使用双缓存，即内存缓存以及磁盘缓存，以及时下流行的图片加载库也是用了双缓存，虽然他们对代码要不进行改动要不重写代码，但是原理类似，先推荐两篇关于DiskLruCache的使用以及分析文章：

鸿洋的 [Android DiskLruCache 源码解析 硬盘缓存的绝佳方案][1]

郭神的 [Android DiskLruCache完全解析，硬盘缓存的最佳方案][2]

~~~ 网上关于DiskLruCache的源码解析也有，我不过我看了一下大多数都是使用讲解偏多，深入解析偏少，所以我决定来一最详细的。同样我们先从获取DiskLruCache实例入手：

    public static DiskLruCache open(File directory, int appVersion, int valueCount, long maxSize) 

至于这四个参数，上两篇文章讲解的很清楚，我们主要看一下其内部流程：

        DiskLruCache cache = new DiskLruCache(directory, appVersion, valueCount, maxSize);
            if (cache.journalFile.exists()) {
                try {
                    cache.readJournal();//verify,current app is a mathchacble version
                    cache.processJournal();
                    cache.journalWriter = new BufferedWriter(new FileWriter(cache.journalFile, true),
                            IO_BUFFER_SIZE);
                    return cache;
                } catch (IOException journalIsCorrupt) {
    //                System.logW("DiskLruCache " + directory + " is corrupt: "
    //                        + journalIsCorrupt.getMessage() + ", removing");
                    cache.delete();
                }
            }

            // create a new empty cache
            directory.mkdirs();
            cache = new DiskLruCache(directory, appVersion, valueCount, maxSize);
            cache.rebuildJournal();
            return cache;

首先会判断日志文件存不存在，如果存在他会做三件事：
1.cache.readJournal() 这个函数中使用readAsciiLine(in)分别读取日志文件的头部的五个信息，并根据这五个信息去判断当前日志文件是否与当前app版本以及各个规格相符合，不符合会抛出异常。如果符合则会调用readJournalLine(readAsciiLine(in))去恢复lruEntries，虽然我们把要缓存的内容保存到磁盘中，但是在内存中我们仍然需要通过“一张表格”来快速查找我们需要的内存信息，lruEntries就只这张内存中“表格”。至于如何恢复lruEntries，我把注释加到了代码中：
	
      private void readJournalLine(String line) throws IOException {
          String[] parts = line.split(" ");
          if (parts.length < 2) {//每一行记录最少有两个字段，一个记录属性，一个key
              throw new IOException("unexpected journal line: " + line);
          }

          String key = parts[1];//读取记录属性
          if (parts[0].equals(REMOVE) && parts.length == 2) {
              lruEntries.remove(key);
              return;
          }
			//如果不是删除操作，接下来...
          Entry entry = lruEntries.get(key);
          if (entry == null) {
              entry = new Entry(key);
              lruEntries.put(key, entry);
          }

          if (parts[0].equals(CLEAN) && parts.length == 2 + valueCount) {//如果是成功插入数据那么，Entry中最多应该有2+valueCount个字段
              entry.readable = true;//只有Entry中含有缓存值时，此只为true，否则为false
              entry.currentEditor = null;
              entry.setLengths(copyOfRange(parts, 2, parts.length));//将缓存信息的大小放入到entry中
          } else if (parts[0].equals(DIRTY) && parts.length == 2) {//加入是一条脏数据，也可以理解为正在编辑的数据
              entry.currentEditor = new Editor(entry);
          } else if (parts[0].equals(READ) && parts.length == 2) {//读操作，对lruEntries无影响
              // this work was already done by calling lruEntries.get()
          } else {
              throw new IOException("unexpected journal line: " + line);
          }
      }

这样一张关于磁盘中缓存新的映射表就建立好了，我们实际的的读写操作都会直接或间接的操作它使其保持和缓存信息的一致。

  2.cache.processJournal() 重新统计当前缓冲使用的空间size，每一个key值对应的Entry其中可能会保存多个缓存单元，比如图片，所以需要将所有缓存单元大小都加以计算，另外需要注意，这个函数还做了另外一个操作，在 cache.readJournal()并没有将记录属性为DIRTY的记录删除，而在这个步骤中，他会将entry.currentEditor ！= null 的DIRTY数据清除掉实际上删除缓存单元文件；

  3.初始化cache.journalWriter，用于随时刷新日志文件。

假如日志文件不存在，可能由于用户删除文件，第一次使用应用或是认为清除缓存等，这时候会重新恢复日志文件：首先创建日志临时文件，把头部信息呢写入，接着再根据我们的“表格”lruEntries，来恢复日志文件，当然刚开始lruEntries中什么都没有，并且lruEntries使用final修饰，所以这一步几乎不会做任何事，我思来想去既然这样部分代码没用为什么还要写呢，后来我想，如果我们在源码的的DiskLruCache基础上加以定制，我们不确保不会在初始化lruEntries时，不对其加入些缓存内容这样它也能自我恢复日志文件。不过在继续读代码我发现我错了，还有地方会用到rebuildJournal(). 最后将临时文件改名为日志文件。
open函数使用了static 关键字修饰，所以我们也可以将其理解为工厂模式的一种是用。

写入缓存：

	

        new Thread(new Runnable() {
        @Override
        public void run() {
            try {
                String imageUrl = "xxxxxxxxxxx";
                String key = hashKeyForDisk(imageUrl);
                DiskLruCache.Editor editor = mDiskLruCache.edit(key);
                if (editor != null) {
                    OutputStream outputStream = editor.newOutputStream(0);
                    if (downloadUrlToStream(imageUrl, outputStream)) {
                        editor.commit();
                    } else {
                        editor.abort();
                    }
                }
                mDiskLruCache.flush();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }).start();

读缓存：

    try {
	String imageUrl = "xxxxxxxxxxxxx";
	String key = hashKeyForDisk(imageUrl);
	DiskLruCache.Snapshot snapShot = mDiskLruCache.get(key);
	if (snapShot != null) {
		InputStream is = snapShot.getInputStream(0);
		Bitmap bitmap = BitmapFactory.decodeStream(is);
		mImage.setImageBitmap(bitmap);
	}
} catch (IOException e) {
	e.printStackTrace();
}

上述写缓存与读缓存各用到了一个关键类：DiskLruCache.Editor、DiskLruCache.Snapshot，在讲解这两个类之前，有必要先说明一下Entry类：
	
      private final class Entry {
          private final String key;

          /** Lengths of this entry's files. */
          private final long[] lengths;

          /** True if this entry has ever been published */
          private boolean readable;

          /** The ongoing edit or null if this entry is not being edited. */
          private Editor currentEditor;

          /** The sequence number of the most recently committed edit to this entry. */
          private long sequenceNumber;

          private Entry(String key) {
              this.key = key;
              this.lengths = new long[valueCount];
          }
			/**
			*读取没有缓存单元的长度大小
			*
           */
          public String getLengths() throws IOException {
              StringBuilder result = new StringBuilder();
              for (long size : lengths) {
                  result.append(' ').append(size);
              }
              return result.toString();
          }

          /**
           * Set lengths using decimal numbers like "10123".
			*设置没有缓存单元的长度大小
			*
           */
          private void setLengths(String[] strings) throws IOException {
              if (strings.length != valueCount) {
                  throw invalidLengths(strings);
              }

              try {
                  for (int i = 0; i < strings.length; i++) {
                      lengths[i] = Long.parseLong(strings[i]);
                  }
              } catch (NumberFormatException e) {
                  throw invalidLengths(strings);
              }
          }

          private IOException invalidLengths(String[] strings) throws IOException {
              throw new IOException("unexpected journal line: " + Arrays.toString(strings));
          }
			/**
			获取指定的文件，通过这我们了解了它的文件存储规则
			*
           */
          public File getCleanFile(int i) {
              return new File(directory, key + "." + i);
          }
			/**
			获取记录属性为dirty指定的文件，这些文件一般没有commit或是abort，也就是说说的			*正在操作的文件
           */
          public File getDirtyFile(int i) {
              return new File(directory, key + "." + i + ".tmp");
          }
      }

首先我们可以明白一点，一个Entry中可保存多个缓存单元的路径。它把每个缓存单元的的空间大小保存在lengths中，key与lruEntries中对应的是一个key，Entry类提供几个方法，在注释中加以说明吧。

Editor类：

      /**
       * Edits the values for an entry.
       */
      public final class Editor {
          private final Entry entry;
          private boolean hasErrors;

          private Editor(Entry entry) {
              this.entry = entry;
          }

          /**
           * Returns an unbuffered input stream to read the last committed value,
           * or null if no value has been committed.
           */
          public InputStream newInputStream(int index) throws IOException {
              synchronized (DiskLruCache.this) {
                  if (entry.currentEditor != this) {
                      throw new IllegalStateException();
                  }
                  if (!entry.readable) {
                      return null;
                  }
                  return new FileInputStream(entry.getCleanFile(index));
              }
          }

          /**
           * Returns the last committed value as a string, or null if no value
           * has been committed.
           */
          public String getString(int index) throws IOException {
              InputStream in = newInputStream(index);
              return in != null ? inputStreamToString(in) : null;
          }

          /**
           * Returns a new unbuffered output stream to write the value at
           * {@code index}. If the underlying output stream encounters errors
           * when writing to the filesystem, this edit will be aborted when
           * {@link #commit} is called. The returned output stream does not throw
           * IOExceptions.
           */
          public OutputStream newOutputStream(int index) throws IOException {
              synchronized (DiskLruCache.this) {
                  if (entry.currentEditor != this) {
                      throw new IllegalStateException();
                  }
                  return new FaultHidingOutputStream(new FileOutputStream(entry.getDirtyFile(index)));
              }
          }

          /**
           * Sets the value at {@code index} to {@code value}.
           */
          public void set(int index, String value) throws IOException {
              Writer writer = null;
              try {
                  writer = new OutputStreamWriter(newOutputStream(index), UTF_8);
                  writer.write(value);
              } finally {
                  closeQuietly(writer);
              }
          }

          /**
           * Commits this edit so it is visible to readers.  This releases the
           * edit lock so another edit may be started on the same key.
           */
          public void commit() throws IOException {
              if (hasErrors) {
                  completeEdit(this, false);
                  remove(entry.key); // the previous entry is stale
              } else {
                  completeEdit(this, true);
              }
          }

          /**
           * Aborts this edit. This releases the edit lock so another edit may be
           * started on the same key.
           */
          public void abort() throws IOException {
              completeEdit(this, false);
          }

          private class FaultHidingOutputStream extends FilterOutputStream {
              ...
      }

通过mDiskLruCache.edit(key)来返回Eidtor对象，edit(key)函数中主要做了几件事，1.验证key的有效性，上边两篇博客都提到了这掉，我们需要对key进行编码加密等操作。2.根据expectedSequenceNumber来判断当前key值是否已经提交过缓存，相当于验证当前key是否存在，如存在返回null，3.生成entry，放入到lruEntries，并将currentEditor置当前editor，防止其他editor操作，产生错误。

Eidtor为我们提供重要的操作Entry中缓存值的函数，两个重载newOutputStream函数分别返回两种文件输入流，不过FaultHidingOutputStream是对FilterOutputStream几个方法的简单重写，主要增加了对IO异常的处理，若有异常hasErrors置为true，当发生异常时，我们应该把异常缓存的残缺内容删除，这里已经在commit函数中实现，看一下这个函数不管有无异常都会执行completeEdit函数，他是做什么的呢？首先他会判断currentEditor是都另有其人，如若有说明可能不止一个editor在编辑当前Entry，这时抛出异常，然后会去查看是否期望缓存的文件都创建并保存下来了，如果有没有的则仍然判处异常。再然后它会把dirty的临时魂村文件改为clean属性的缓存文件，并更新size。倘若简历dirty属性文件的过程中存在异常，即hasErrors为true则会这些文件删掉。接下来redundantOpCount++，这一个一会讲，在然后更新日志文件，并置Entry sequenceNumber 为当前的值，此值记录了当前是总的第几次提交的缓存，同样，入股欧放生过异常，则删除这个key并记录日志。最后的操作就和前边所说的redundantOpCount的有关，先判断当前缓冲占用空间size是否超出最大限制，以及日志文件操作的记录条数，是否超过规定限制，两者满足其中则会通过线程池的方式执行一个实行callable接口的cleanupCallable类对象，目的做两件事，如果当前journalWriter不为空，即缓存日志仍然允许访问，则会执行函数trimToSize()，这个Lrucache中trimToSize()效果类似，将最不常使用的Entry移除，只不过这里两种方法实现，因为LinkedHashMap将最新的Entry放到链表尾部，旧的都在头部，使用iterator删除的恰好也是头部Entry。另外还可能清除精简日志，这里又用到了rebuildJournal(),哈哈。

PS：当我们试图去写缓存时，写入的数量应该和Entry中valuecount的数值相等，否则会产生错误。

而当我们读缓存是使用了Snapshot类，这个类主要是获取Entry中缓存单元的数据流，比较简单，不再赘述。

另外DiskLruCache还提供了其他的几个函数共使用：

    public synchronized long size() {
        return size;
    }

返回当前缓存共使用的空间。

     /**
     * Closes the cache and deletes all of its stored values. This will delete
     * all files in the cache directory including files that weren't created by
     * the cache.
     */
    public void delete() throws IOException {
        close();
        deleteContents(directory);
    }

用来清空当前缓存，方便吧。

      /**
     * Force buffered operations to the filesystem.
     */
    public synchronized void flush() throws IOException {
        checkNotClosed();
        trimToSize();
        journalWriter.flush();
    }

当我们吧缓存内容写入到磁盘中时，别忘加执行flush()，他即会重新确保我们的缓存总大小不会超过限制，又会确保将日志在缓存中的的内容强制输入到文件中，不过这个操作我们只需要执行一次，的过多执行只会消耗资源，建议在onpause中执行该操作。

DiskLruCache的源码解析就到这，解析这个源码的目的时明白其原理，在我们做开发时，假如出现错误也能够发现错误及时改正，另外介入原生的代码不嗯给你满足我们的需求，我们也可以在它的基础上加以定制。



  [1]: http://blog.csdn.net/lmj623565791/article/details/47251585
  [2]: http://blog.csdn.net/guolin_blog/article/details/28863651
