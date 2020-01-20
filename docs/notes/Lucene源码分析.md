

# 1.  索引，搜索示例

## 1.1 索引

```java
 Analyzer analyzer = new StandardAnalyzer();

        // Store the index in memory:
//        Directory directory = new RAMDirectory();
        // To store an index on disk, use this instead:
        Directory directory = FSDirectory.open(Path.of("./data"));
        IndexWriterConfig config = new IndexWriterConfig(analyzer);
        IndexWriter iwriter = new IndexWriter(directory, config);
        Document doc = new Document();
        String text = "This is the text to be indexed.";
        doc.add(new Field("fieldname", text, TextField.TYPE_STORED));
        iwriter.addDocument(doc);
        iwriter.close();
```

## 1.2 搜索

```java
DirectoryReader ireader = DirectoryReader.open(directory);
        IndexSearcher isearcher = new IndexSearcher(ireader);
        // Parse a simple query that searches for "text":
        QueryParser parser = new QueryParser("fieldname", analyzer);
        Query query = parser.parse("text");
        ScoreDoc[] hits = isearcher.search(query, 1000, null).scoreDocs;
//        assertEquals(1, hits.length);
        // Iterate through the results:
        for (int i = 0; i < hits.length; i++) {
            Document hitDoc = isearcher.doc(hits[i].doc);
//            assertEquals("This is the text to be indexed.", hitDoc.get("fieldname"));
        }
        ireader.close();
        directory.close();
```

# 2. 索引源码分析 1 - Directory 初始化

首先，创建索引的目录，存放索引的本地文件

```java
Directory directory = FSDirectory.open(Path.of("./data"));
```

FSLockFactory.getDefault() 方法返回的是 NativeFSLockFactory 的实例。

```java
// open 静态方法
public static FSDirectory open(Path path) throws IOException {
    return open(path, FSLockFactory.getDefault());
  }
// 获得默认 FSLockFactory
public static final FSLockFactory getDefault() {
    return NativeFSLockFactory.INSTANCE;
  }
// 返回 Directory
public static FSDirectory open(Path path, LockFactory lockFactory) throws IOException {
    // 判断是否是64位JRE ，UNMAP_SUPPORTED初始化为 true
    if (Constants.JRE_IS_64BIT && MMapDirectory.UNMAP_SUPPORTED) {
      return new MMapDirectory(path, lockFactory);
    } else if (Constants.WINDOWS) {
      return new SimpleFSDirectory(path, lockFactory);
    } else {
      return new NIOFSDirectory(path, lockFactory);
    }
  }
```

![](pics\MMapDirectory.png)

FSDirectory 最终返回的是 MMapDirectory 的实例。MMapDirectory  的创建过程是 先创建父类并初始化，最后在创建 MMapDirectory   实例并初始化。[参考有符号移动无符号移动](https://blog.csdn.net/u014110320/article/details/83037130)

```java
// 默认 最大 chunk size. JRE_IS_64BIT 为 true, 1 << 30，否则 1 << 28
public static final int DEFAULT_MAX_CHUNK_SIZE = Constants.JRE_IS_64BIT ? (1 << 30) : (1 << 28);

// MMapDirectory 构造函数，初始化 MMapDirectory
public MMapDirectory(Path path, LockFactory lockFactory) throws IOException {
    this(path, lockFactory, DEFAULT_MAX_CHUNK_SIZE);
  }
// 这里的 maxChunkSize 传入 1 << 30
public MMapDirectory(Path path, LockFactory lockFactory, int maxChunkSize) throws IOException {
    // 调用父类的构造函数
    super(path, lockFactory);
    if (maxChunkSize <= 0) {
      throw new IllegalArgumentException("Maximum chunk size for mmap must be >0");
    }
    // chunkSizePower = 30
    this.chunkSizePower = 31 - Integer.numberOfLeadingZeros(maxChunkSize);
    assert this.chunkSizePower >= 0 && this.chunkSizePower <= 30;
  }


// 传入的 i 为 1073741824，>>> 无符号移动
public static int numberOfLeadingZeros(int i) {
        if (i <= 0) {
            return i == 0 ? 32 : 0;
        } else {
            int n = 31;
            if (i >= 65536) { 
                n -= 16;
                i >>>= 16;// 右移动 16位后，i 为 16384
            }

            if (i >= 256) {
                n -= 8;
                i >>>= 8; // 右移动 8 位后， i 为 64
            }

            if (i >= 16) {
                n -= 4;
                i >>>= 4; //右移动 4 位后，i 为 4
            }

            if (i >= 4) {
                n -= 2;
                i >>>= 2;// 右移动 2 位后，i 为 1
            }

            return n - (i >>> 1); //i >>> 1 移动后为 0，返回 1
        }
    } 
```

调用父类 FSDirectory 的构造方法  super(path, lockFactory)；初始化pendingDeletes 指定 在尝试删除该键之前，映射我们要删除（或我们已经尝试但失败）的文件；**opsSinceLastDelete 不知道啥意思？，**nextTempFileCounter  用来创建 temp file name。 directory  构造函数中指定索引目录的 绝对路径；

```java
protected FSDirectory(Path path, LockFactory lockFactory) throws IOException {
    // 接着调用父类的构造方法 
    super(lockFactory);
    // If only read access is permitted, createDirectories fails even if the directory already exists.如果是只读，即使目录已经存在，createDirectories也返回失败
    if (!Files.isDirectory(path)) {
      Files.createDirectories(path);  // create directory, if it doesn't exist 如果不存在，创建目录
    }
    directory = path.toRealPath();
  }
```

调用父类 BaseDirectory 的构造方法  super(lockFactory)；然后初始化BaseDirectory 的成员变量，isOpen = false，lockFactory 在构造函数中初始化为 NativeFSLockFactory  实例。

```java
protected BaseDirectory(LockFactory lockFactory) {
    // 调用父类的无参构造方法
    super();
    if (lockFactory == null) {
      throw new NullPointerException("LockFactory must not be null, use an explicit instance!");
    }
    this.lockFactory = lockFactory;
  }
```

# 3. 索引源码分析 2 - IndexWriterConfig初始化

类图

![](pics\IndexWriterConfig.png)

```java
//使用标准 分词器，这里主要介绍 IndexWriterConfig，analyzer具体用法不深入探讨
Analyzer analyzer = new StandardAnalyzer();
IndexWriterConfig config = new IndexWriterConfig(analyzer);
```

默认在排序的类型，包括以下几种：

```java
private static final EnumSet<SortField.Type> ALLOWED_INDEX_SORT_TYPES = EnumSet.of(SortField.Type.STRING,
                                                                                     SortField.Type.LONG,
                                                                                     SortField.Type.INT,
                                                                                     SortField.Type.DOUBLE,
                                                                                     SortField.Type.FLOAT);
```

调用父类的构造方法：

```java
 public IndexWriterConfig(Analyzer analyzer) {
    super(analyzer);
  }
```

```java
 // True if segment flushes should use compound file format 
protected volatile boolean useCompoundFile = IndexWriterConfig.DEFAULT_USE_COMPOUND_FILE_SYSTEM;

//  True if calls to {@link IndexWriter#close()} should first do a commit
protected boolean commitOnClose = IndexWriterConfig.DEFAULT_COMMIT_ON_CLOSE;

// The sort order to use to write merged segments.
  protected Sort indexSort = null;

  // The field names involved in the index sort
  protected Set<String> indexSortFields = Collections.emptySet();

  // if an indexing thread should check for pending flushes on update in order to help out on a full flush
  protected volatile boolean checkPendingFlushOnUpdate = true;

  // soft deletes field 
  protected String softDeletesField = null;

LiveIndexWriterConfig(Analyzer analyzer) {
    // 初始化为 StandardAnalyzer
    this.analyzer = analyzer;
    // ram缓存大小 16 M
    ramBufferSizeMB = IndexWriterConfig.DEFAULT_RAM_BUFFER_SIZE_MB;
    // 默认是 disabled 的，因为 IndexWriter 默认 按 RAM 的使用情况 进行 flushes
    maxBufferedDocs = IndexWriterConfig.DEFAULT_MAX_BUFFERED_DOCS;
    mergedSegmentWarmer = null;
    // IndexDeletionPolicy 设置 KeepOnlyLastCommitDeletionPolicy
    delPolicy = new KeepOnlyLastCommitDeletionPolicy();
    commit = null;
    // 是否使用合成文件(.cfe、.cfs)，默认为 true
    useCompoundFile = IndexWriterConfig.DEFAULT_USE_COMPOUND_FILE_SYSTEM;
    // OpenMode 初始化 CREATE_OR_APPEND
    openMode = OpenMode.CREATE_OR_APPEND;
    // Similarity 默认使用 BM25Similarity
    similarity = IndexSearcher.getDefaultSimilarity();
    // MergeScheduler 设置为 ConcurrentMergeScheduler
    mergeScheduler = new ConcurrentMergeScheduler();
    // IndexingChain 默认实现 getChain 方法，返回 DefaultIndexingChain
    indexingChain = DocumentsWriterPerThread.defaultIndexingChain;
    // 默认为 Lucene70
    codec = Codec.getDefault();
    if (codec == null) {
      throw new NullPointerException();
    }
    // 默认使用 NoOutput
    infoStream = InfoStream.getDefault();
    // MergePolicy 设置Wie TieredMergePolicy
    mergePolicy = new TieredMergePolicy();
    // FlushPolicy 设置为 FlushByRamOrCountsPolicy
    flushPolicy = new FlushByRamOrCountsPolicy();
    // 默认为 true
    readerPooling = IndexWriterConfig.DEFAULT_READER_POOLING;
    // 设置为 DocumentsWriterPerThreadPool
    indexerThreadPool = new DocumentsWriterPerThreadPool();
    // 默认 1945
    perThreadHardLimitMB = IndexWriterConfig.DEFAULT_RAM_PER_THREAD_HARD_LIMIT_MB;
  }
```

Lucene70 中field 的值：

![](pics\Lucen70-source.png)

# 4.  索引源码分析 3 - IndexWriter初始化

![](pics\IndexWriter.png)

```java
IndexWriter iwriter = new IndexWriter(directory, config);
```

IndexWriter 构造函数

```java
  public IndexWriter(Directory d, IndexWriterConfig conf) throws IOException {
      // Used only for testing
    enableTestPoints = isEnableTestPoints();
      // prevent reuse by other instances 防止其他实例重用
    conf.setIndexWriter(this); 
    config = conf;
    infoStream = config.getInfoStream();
    softDeletesEnabled = config.getSoftDeletesField() != null;
    // obtain the write.lock. If the user configured a timeout,
    // we wrap with a sleeper and this might take some time.
      // 4.1 writeLock 初始化
    writeLock = d.obtainLock(WRITE_LOCK_NAME);// 9.25 看到这里
    
    boolean success = false;
    try {
        //原始的 directory
      directoryOrig = d;
        //使用 writeLock 包装的 directory
      directory = new LockValidatingDirectoryWrapper(d, writeLock);
		// 分词器 使用的是 IndexWriterConf 中的标准分词器
        // StandardAnalyzer
      analyzer = config.getAnalyzer();
        // ConcurrentMergeScheduler
      mergeScheduler = config.getMergeScheduler();
      mergeScheduler.setInfoStream(infoStream);
        //Lucene70Codec
      codec = config.getCodec();
        // config 中的 mode = CREATE_OR_APPEND
      OpenMode mode = config.getOpenMode();
      final boolean indexExists;
      final boolean create;
      if (mode == OpenMode.CREATE) {
        indexExists = DirectoryReader.indexExists(directory);
        create = true;
      } else if (mode == OpenMode.APPEND) {
        indexExists = true;
        create = false;
      } else {
        // CREATE_OR_APPEND - create only if an index does not exist
          // CREATE_OR_APPEND - 仅当索引不存在才创建
          // indexExists 检查的是索引目录中 segments_N 是否存在
        indexExists = DirectoryReader.indexExists(directory);
        create = !indexExists;
      }

      // If index is too old, reading the segments will throw
      // IndexFormatTooOldException.
        // 如果 索引太旧了，读取 segments 将会抛出 IndexFormatTooOldException异常

      String[] files = directory.listAll();

      // Set up our initial SegmentInfos:
      IndexCommit commit = config.getIndexCommit();

      // Set up our initial SegmentInfos:
      StandardDirectoryReader reader;
      if (commit == null) {
        reader = null;
      } else {
        reader = commit.getReader();
      }

        // 4.2 segmentInfos 初始化及 rollbackSegments初始化 begin
      if (create) {

        if (config.getIndexCommit() != null) {
          // We cannot both open from a commit point and create:
          if (mode == OpenMode.CREATE) {
            throw new IllegalArgumentException("cannot use IndexWriterConfig.setIndexCommit() with OpenMode.CREATE");
          } else {
            throw new IllegalArgumentException("cannot use IndexWriterConfig.setIndexCommit() when index has no commit");
          }
        }

        // Try to read first.  This is to allow create
        // against an index that's currently open for
        // searching.  In this case we write the next
        // segments_N file with no segments:
        final SegmentInfos sis = new SegmentInfos(Version.LATEST.major);
          // indexExists 存在的时候 才会读取最后提交的 段 信息，并且更新新建的段 信息。
        if (indexExists) {
          final SegmentInfos previous = SegmentInfos.readLatestCommit(directory);// 9.26 看到这里
          sis.updateGenerationVersionAndCounter(previous);
        }
        segmentInfos = sis;
        rollbackSegments = segmentInfos.createBackupSegmentInfos();

        // Record that we have a change (zero out all
        // segments) pending:
        changed();

      } else if (reader != null) {
        // Init from an existing already opened NRT or non-NRT reader:
      
        if (reader.directory() != commit.getDirectory()) {
          throw new IllegalArgumentException("IndexCommit's reader must have the same directory as the IndexCommit");
        }

        if (reader.directory() != directoryOrig) {
          throw new IllegalArgumentException("IndexCommit's reader must have the same directory passed to IndexWriter");
        }

        if (reader.segmentInfos.getLastGeneration() == 0) {  
          // TODO: maybe we could allow this?  It's tricky...
          throw new IllegalArgumentException("index must already have an initial commit to open from reader");
        }

        // Must clone because we don't want the incoming NRT reader to "see" any changes this writer now makes:
        segmentInfos = reader.segmentInfos.clone();

        SegmentInfos lastCommit;
        try {
          lastCommit = SegmentInfos.readCommit(directoryOrig, segmentInfos.getSegmentsFileName());
        } catch (IOException ioe) {
          throw new IllegalArgumentException("the provided reader is stale: its prior commit file \"" + segmentInfos.getSegmentsFileName() + "\" is missing from index");
        }

        if (reader.writer != null) {

          // The old writer better be closed (we have the write lock now!):
          assert reader.writer.closed;

          // In case the old writer wrote further segments (which we are now dropping),
          // update SIS metadata so we remain write-once:
          segmentInfos.updateGenerationVersionAndCounter(reader.writer.segmentInfos);
          lastCommit.updateGenerationVersionAndCounter(reader.writer.segmentInfos);
        }

        rollbackSegments = lastCommit.createBackupSegmentInfos();
      } else {
        // Init from either the latest commit point, or an explicit prior commit point:

        String lastSegmentsFile = SegmentInfos.getLastCommitSegmentsFileName(files);
        if (lastSegmentsFile == null) {
          throw new IndexNotFoundException("no segments* file found in " + directory + ": files: " + Arrays.toString(files));
        }

        // Do not use SegmentInfos.read(Directory) since the spooky
        // retrying it does is not necessary here (we hold the write lock):
        segmentInfos = SegmentInfos.readCommit(directoryOrig, lastSegmentsFile);

        if (commit != null) {
          // Swap out all segments, but, keep metadata in
          // SegmentInfos, like version & generation, to
          // preserve write-once.  This is important if
          // readers are open against the future commit
          // points.
          if (commit.getDirectory() != directoryOrig) {
            throw new IllegalArgumentException("IndexCommit's directory doesn't match my directory, expected=" + directoryOrig + ", got=" + commit.getDirectory());
          }
          
          SegmentInfos oldInfos = SegmentInfos.readCommit(directoryOrig, commit.getSegmentsFileName());
          segmentInfos.replace(oldInfos);
          changed();

          if (infoStream.isEnabled("IW")) {
            infoStream.message("IW", "init: loaded commit \"" + commit.getSegmentsFileName() + "\"");
          }
        }

        rollbackSegments = segmentInfos.createBackupSegmentInfos();
      }
        // 4.2 segmentInfos 初始化及 rollbackSegments初始化 end

		// IndexWriter.commit 期间，用户所指定的 Map<String,String>
      commitUserData = new HashMap<>(segmentInfos.getUserData()).entrySet();
		// segment 的 maxDocs 的和。注意，不包含删除的。
      pendingNumDocs.set(segmentInfos.totalMaxDoc());

      // start with previous field numbers, but new FieldInfos
      // NOTE: this is correct even for an NRT reader because we'll pull FieldInfos even for the un-committed segments:
        // 从以前的 field number 开始，但是 新的 FieldInfos。
      globalFieldNumberMap = getFieldNumberMap();
		// 确认 传入的 索引排序 与 现有的 索引排序 匹匹配。
      validateIndexSort();
		// 初始化 FlushPolicy
      config.getFlushPolicy().init(config);
        // 初始化 BufferedUpdatesStream
      bufferedUpdatesStream = new BufferedUpdatesStream(infoStream);
        // 初始化 DocumentsWriter
      docWriter = new DocumentsWriter(flushNotifications, segmentInfos.getIndexCreatedVersionMajor(), pendingNumDocs,
          enableTestPoints, this::newSegmentName,
          config, directoryOrig, directory, globalFieldNumberMap);
        // 初始化 ReaderPool
      readerPool = new ReaderPool(directory, directoryOrig, segmentInfos, globalFieldNumberMap,
          bufferedUpdatesStream::getCompletedDelGen, infoStream, conf.getSoftDeletesField(), reader);
      if (config.getReaderPooling()) {
        readerPool.enableReaderPooling();
      }
      // Default deleter (for backwards compatibility) is
      // KeepOnlyLastCommitDeleter:

      // Sync'd is silly here, but IFD asserts we sync'd on the IW instance:
      synchronized(this) {
          // 初始化 IndexFileDeleter
        deleter = new IndexFileDeleter(files, directoryOrig, directory,
                                       config.getIndexDeletionPolicy(),
                                       segmentInfos, infoStream, this,
                                       indexExists, reader != null);

        // We incRef all files when we return an NRT reader from IW, so all files must exist even in the NRT case:
        assert create || filesExist(segmentInfos);
      }

      if (deleter.startingCommitDeleted) {
          // 删除策略 删除 "head"  提交点
        // Deletion policy deleted the "head" commit point.
        // We have to mark ourself as changed so that if we
        // are closed w/o any further changes we write a new
        // segments_N file.
        changed();
      }

      if (reader != null) {
        // We always assume we are carrying over incoming changes when opening from reader:
        segmentInfos.changed();
        changed();
      }

      if (infoStream.isEnabled("IW")) {
        infoStream.message("IW", "init: create=" + create + " reader=" + reader);
        messageState();
      }

      success = true;

    } finally {
      if (!success) {
        if (infoStream.isEnabled("IW")) {
          infoStream.message("IW", "init: hit exception on init; releasing write lock");
        }
        IOUtils.closeWhileHandlingException(writeLock);
        writeLock = null;
      }
    }
  }

```

以下对几个重要的初始化进行跟踪：

## 4.1 writeLock 初始化

 **d.obtainLock(WRITE_LOCK_NAME);**来得到 write.lock 锁，其中，d 是 MMapDirectory ，obtainLock 方法是在其父类 BaseDirectory中。( [索引文件锁LockFactory](https://www.amazingkoala.com.cn/Lucene/Store/2019/0604/62.html))

```java
public final Lock obtainLock(String name) throws IOException {
    return lockFactory.obtainLock(this, name);
  }
```

 lockFactory默认采用的是 NativeFSLockFactory，lockFactory.obtainLock 调用的是 其父类 FSLockFactory 的 obtainLock 方法

```java
public final Lock obtainLock(Directory dir, String lockName) throws IOException {
    if (!(dir instanceof FSDirectory)) {
      throw new UnsupportedOperationException(getClass().getSimpleName() + " can only be used with FSDirectory subclasses, got: " + dir);
    }
    return obtainFSLock((FSDirectory) dir, lockName);
  }
```

obtainFSLock 方法 在 NativeFSLockFactory  中实现。

```java
protected Lock obtainFSLock(FSDirectory dir, String lockName) throws IOException {
    // 索引目录的绝对路径
    Path lockDir = dir.getDirectory();
    
    // Ensure that lockDir exists and is a directory.
    // note: this will fail if lockDir is a symlink
    // 确定 lockDir 存在并且是一个目录，注意，如果 locdir 是一个符号链接将会失败
    Files.createDirectories(lockDir);
    // write.lock 文件的绝对路径
    Path lockFile = lockDir.resolve(lockName);

    IOException creationException = null;
    try {
      Files.createFile(lockFile);// 9.25 看到这里
    } catch (IOException ignore) {
      // we must create the file to have a truly canonical path.
      // if it's already created, we don't care. if it cant be created, it will fail below.
      creationException = ignore;
    }
    
    // fails if the lock file does not exist
    final Path realPath;
    try {
      realPath = lockFile.toRealPath();
    } catch (IOException e) {
      // if we couldn't resolve the lock file, it might be because we couldn't create it.
      // so append any exception from createFile as a suppressed exception, in case its useful
      if (creationException != null) {
        e.addSuppressed(creationException);
      }
      throw e;
    }
    
    // used as a best-effort check, to see if the underlying file has changed
    final FileTime creationTime = Files.readAttributes(realPath, BasicFileAttributes.class).creationTime();
    
    if (LOCK_HELD.add(realPath.toString())) {
      FileChannel channel = null;
      FileLock lock = null;
      try {
          // 这里 抽象类FileChannel 的实现类 是 FileChannelImpl
        channel = FileChannel.open(realPath, StandardOpenOption.CREATE, StandardOpenOption.WRITE);
        lock = channel.tryLock();
        if (lock != null) {
          return new NativeFSLock(lock, channel, realPath, creationTime);
        } else {
          throw new LockObtainFailedException("Lock held by another program: " + realPath);
        }
      } finally {
        if (lock == null) { // not successful - clear up and move out
          IOUtils.closeWhileHandlingException(channel); // TODO: addSuppressed
          clearLockHeld(realPath);  // clear LOCK_HELD last 
        }
      }
    } else {
      throw new LockObtainFailedException("Lock held by this virtual machine: " + realPath);
    }
  }
  
```

**Files.createFile(lockFile);**创建write.lock 文件。

```java
public static Path createFile(Path path, FileAttribute<?>... attrs) throws IOException {
    // 创建 一个 FileChannel将 write.lock 文件写到目录中，创建完成后关闭 channel，返回文件的绝对路径。
        newByteChannel(path, DEFAULT_CREATE_OPTIONS, attrs).close();
        return path;
    
    public static SeekableByteChannel newByteChannel(Path path, Set<? extends OpenOption> options, FileAttribute<?>... attrs) throws IOException {
        return provider(path).newByteChannel(path, options, attrs);
    }
```

由于本机是 Windows 系统，所以这里使用的 provider 是 WindowsFileSystemProvider

```java
public SeekableByteChannel newByteChannel(Path obj, Set<? extends OpenOption> options, FileAttribute<?>... attrs) throws IOException {
        WindowsPath file = WindowsPath.toWindowsPath(obj);
        WindowsSecurityDescriptor sd = WindowsSecurityDescriptor.fromAttribute(attrs);

        Object var7;
        try {
            FileChannel var6 = WindowsChannelFactory.newFileChannel(file.getPathForWin32Calls(), file.getPathForPermissionCheck(), options, sd.address());
            return var6;
        } catch (WindowsException var11) {
            var11.rethrowAsIOException(file);
            var7 = null;
        } finally {
            sd.release();
        }

        return (SeekableByteChannel)var7;
    }
```

调用 WindowsChannelFactory 的静态方法 WindowsChannelFactory

```java
// 调用下面 静态方法
static FileChannel newFileChannel(String pathForWindows, String pathToCheck, Set<? extends OpenOption> options, long pSecurityDescriptor) throws WindowsException {
    // 根据 options 设置 flags 的属性时候为 true。这里的options是WRITE、CREATE_NEW
        WindowsChannelFactory.Flags flags = WindowsChannelFactory.Flags.toFlags(options);
        if (!flags.read && !flags.write) {
            if (flags.append) {
                flags.write = true;
            } else {
                flags.read = true;
            }
        }

        if (flags.read && flags.append) {
            throw new IllegalArgumentException("READ + APPEND not allowed");
        } else if (flags.append && flags.truncateExisting) {
            throw new IllegalArgumentException("APPEND + TRUNCATE_EXISTING not allowed");
        } else {
            //此示例执行以下方法
            FileDescriptor fdObj = open(pathForWindows, pathToCheck, flags, pSecurityDescriptor);
            return FileChannelImpl.open(fdObj, pathForWindows, flags.read, flags.write, flags.direct, (Object)null);
        }
    }


// open(pathForWindows, pathToCheck, flags, pSecurityDescriptor); 方法
private static FileDescriptor open(String pathForWindows, String pathToCheck, WindowsChannelFactory.Flags flags, long pSecurityDescriptor) throws WindowsException {
        boolean truncateAfterOpen = false;
        int dwDesiredAccess = 0;
        if (flags.read) {// false
            dwDesiredAccess |= -2147483648;
        }

        if (flags.write) {//true
            // 1073741824 = 1 <<< 30 
            dwDesiredAccess |= 1073741824;
        }// dwDesiredAccess = 1073741824

        int dwShareMode = 0;
        if (flags.shareRead) {// true
            dwShareMode |= 1;
        }

        if (flags.shareWrite) {//true
            dwShareMode |= 2;
        }

        if (flags.shareDelete) {//true
            dwShareMode |= 4;
        }// dwShareMode = 7 

        int dwFlagsAndAttributes = 128;
        int dwCreationDisposition = 3;
        if (flags.write) {// true
            if (flags.createNew) {//true
                //dwCreationDisposition = 1
                // dwFlagsAndAttributes = 2097280
                dwCreationDisposition = 1;
                dwFlagsAndAttributes |= 2097152;
            } else {
                if (flags.create) {
                    dwCreationDisposition = 4;
                }

                if (flags.truncateExisting) {
                    if (dwCreationDisposition == 4) {
                        truncateAfterOpen = true;
                    } else {
                        dwCreationDisposition = 5;
                    }
                }
            }
        }

        if (flags.dsync || flags.sync) {// dsync = false, sync = false
            dwFlagsAndAttributes |= -2147483648;
        }

        if (flags.overlapped) {//fasle
            dwFlagsAndAttributes |= 1073741824;
        }

        if (flags.deleteOnClose) { //false
            dwFlagsAndAttributes |= 67108864;
        }

        boolean okayToFollowLinks = true;
        if (dwCreationDisposition != 1 && (flags.noFollowLinks || flags.openReparsePoint || flags.deleteOnClose)) {
            if (flags.noFollowLinks || flags.deleteOnClose) {
                okayToFollowLinks = false;
            }

            dwFlagsAndAttributes |= 2097152;
        }

        if (pathToCheck != null) {
            SecurityManager sm = System.getSecurityManager();
            if (sm != null) {
                if (flags.read) {//false
                    sm.checkRead(pathToCheck);
                }

                if (flags.write) {//true
                    sm.checkWrite(pathToCheck);
                }

                if (flags.deleteOnClose) {//true
                    sm.checkDelete(pathToCheck);
                }
            }
        }

    // dwDesiredAccess = 1073741824
    // dwShareMode = 7 
    // pSecurityDescriptor = 0
     //dwCreationDisposition = 1
     // dwFlagsAndAttributes = 2097280
	// 这里在文件中生成了 write.lock 文件
        long handle = WindowsNativeDispatcher.CreateFile(pathForWindows, dwDesiredAccess, dwShareMode, pSecurityDescriptor, dwCreationDisposition, dwFlagsAndAttributes);
        if (!okayToFollowLinks) {// true
            try {
                if (WindowsFileAttributes.readAttributes(handle).isSymbolicLink()) {
                    throw new WindowsException("File is symbolic link");
                }
            } catch (WindowsException var16) {
                WindowsNativeDispatcher.CloseHandle(handle);
                throw var16;
            }
        }

        if (truncateAfterOpen) { // false
            try {
                WindowsNativeDispatcher.SetEndOfFile(handle);
            } catch (WindowsException var15) {
                WindowsNativeDispatcher.CloseHandle(handle);
                throw var15;
            }
        }

    // flags.sparse = fasle,dwCreationDisposition==1
        if (dwCreationDisposition == 1 && flags.sparse) {
            try {
                WindowsNativeDispatcher.DeviceIoControlSetSparse(handle);
            } catch (WindowsException var14) {
            }
        }

        FileDescriptor fdObj = new FileDescriptor();
        fdAccess.setHandle(fdObj, handle);
        fdAccess.setAppend(fdObj, flags.append);
        fdAccess.registerCleanup(fdObj);
        return fdObj;// 返回文件描述类
    }
```

**FileChannelImpl.open(fdObj, pathForWindows, flags.read, flags.write, flags.direct, (Object)null);**方法调用

```java
 //fd 文件描述 为open方法返回实例，此时已经创建 write.lock 文件
// path = "C:\Java\IdeaProjects\study\lucene-demo\lucene75\data\write.lock"
// readable = false
// writable = true
// direct = false
// parent = null
public static FileChannel open(FileDescriptor fd, String path, boolean readable, boolean writable, boolean direct, Object parent) {
        return new FileChannelImpl(fd, path, readable, writable, direct, parent);
    }

private FileChannelImpl(FileDescriptor fd, String path, boolean readable, boolean writable, boolean direct, Object parent) {
        this.fd = fd;
        this.readable = readable;
        this.writable = writable;
        this.parent = parent;
        this.path = path;
        this.direct = direct;
        this.nd = new FileDispatcherImpl();
        if (direct) {
            assert path != null;

            this.alignment = this.nd.setDirectIO(fd, path);
        } else {
            this.alignment = -1;
        }

        this.closer = parent != null ? null : CleanerFactory.cleaner().register(this, new FileChannelImpl.Closer(fd));
    }
```

## 4.2 segmentInfos 初始化及 rollbackSegments初始化

执行完   **d.obtainLock(WRITE_LOCK_NAME);** 回到IndexWriter的构造函数中，接下来会对directoryOrig; directory; analyzer; mergeScheduler; codec; 成员变量赋值。OpenMode 默认是 CREATE_OR_APPEND，第一次创建索引目录的时候 indexExists = fasle, create = true;  第一次进入的时候 config.getIndexCommit() = null; 所以，这时候 不会调用  SegmentInfos.readLatestCommit(directory) 方法。segmentInfos  =  sis; rollbackSegments = segmentInfos. createBackupSegmentInfos(); 做简单赋值；并调用 changed() 同步方法 改变 changeCount 自增，segmentInfos.changed() 方法 自增 version++;

```java
 if (create) {

        if (config.getIndexCommit() != null) {
          // We cannot both open from a commit point and create:
          if (mode == OpenMode.CREATE) {
            throw new IllegalArgumentException("cannot use IndexWriterConfig.setIndexCommit() with OpenMode.CREATE");
          } else {
            throw new IllegalArgumentException("cannot use IndexWriterConfig.setIndexCommit() when index has no commit");
          }
        }

        // Try to read first.  This is to allow create
        // against an index that's currently open for
        // searching.  In this case we write the next
        // segments_N file with no segments:
        final SegmentInfos sis = new SegmentInfos(Version.LATEST.major);
        if (indexExists) {
          final SegmentInfos previous = SegmentInfos.readLatestCommit(directory);
          sis.updateGenerationVersionAndCounter(previous);
        }
        segmentInfos = sis;
        rollbackSegments = segmentInfos.createBackupSegmentInfos();

        // Record that we have a change (zero out all
        // segments) pending:
        changed();

      } 
```

下一次重新建立索引的时候，索引目录下已经有了索引文件，这时会进入

```java
    // Init from either the latest commit point, or an explicit prior commit point:
	// 初始化 最近提交点或者之前的提交点
    String lastSegmentsFile = SegmentInfos.getLastCommitSegmentsFileName(files);
	// 如果未找到文件，抛出 IndexNotFoundException 异常
    if (lastSegmentsFile == null) {
      throw new IndexNotFoundException("no segments* file found in " + directory + ": files: " + Arrays.toString(files));
    }

    // Do not use SegmentInfos.read(Directory) since the spooky
    // retrying it does is not necessary here (we hold the write lock):
	// 不要使用 SegmentInfos.read(Directory) ，由于它那吓人的重试是没有必要的(我们持有 write  锁)
    segmentInfos = SegmentInfos.readCommit(directoryOrig, lastSegmentsFile);

    if (commit != null) {
      // Swap out all segments, but, keep metadata in
      // SegmentInfos, like version & generation, to
      // preserve write-once.  This is important if
      // readers are open against the future commit
      // points.
      if (commit.getDirectory() != directoryOrig) {
        throw new IllegalArgumentException("IndexCommit's directory doesn't match my directory, expected=" + directoryOrig + ", got=" + commit.getDirectory());
      }
      
      SegmentInfos oldInfos = SegmentInfos.readCommit(directoryOrig, commit.getSegmentsFileName());
      segmentInfos.replace(oldInfos);
      changed();

      if (infoStream.isEnabled("IW")) {
        infoStream.message("IW", "init: loaded commit \"" + commit.getSegmentsFileName() + "\"");
      }
    }

    rollbackSegments = segmentInfos.createBackupSegmentInfos();
```
SegmentInfos.readCommit(directoryOrig, lastSegmentsFile);通过最后一个段文件返回SegmentInfos

```java

public static final SegmentInfos readCommit(Directory directory, String segmentFileName) throws IOException {

    long generation = generationFromSegmentsFileName(segmentFileName);
    //System.out.println(Thread.currentThread() + ": SegmentInfos.readCommit " + segmentFileName);
    try (ChecksumIndexInput input = directory.openChecksumInput(segmentFileName, IOContext.READ)) {
      try {
        return readCommit(directory, input, generation);
      } catch (EOFException | NoSuchFileException | FileNotFoundException e) {
        throw new CorruptIndexException("Unexpected file read error while reading index.", input, e);
      }
    }
  }
```

调用重载的 readCommit 方法。

```java
public static final SegmentInfos readCommit(Directory directory, ChecksumIndexInput input, long generation) throws IOException {

    // NOTE: as long as we want to throw indexformattooold (vs corruptindexexception), we need
    // to read the magic ourselves.
    int magic = input.readInt();
    // 如果 magic 不等于 codec header 开始常数 抛出异常
    if (magic != CodecUtil.CODEC_MAGIC) {
      throw new IndexFormatTooOldException(input, magic, CodecUtil.CODEC_MAGIC, CodecUtil.CODEC_MAGIC);
    }
    // 检查 不带 magic 的 codec header
    int format = CodecUtil.checkHeaderNoMagic(input, "segments", VERSION_53, VERSION_CURRENT);
    byte id[] = new byte[StringHelper.ID_LENGTH];
    input.readBytes(id, 0, id.length);
    // 检查 索引头的后缀
    CodecUtil.checkIndexHeaderSuffix(input, Long.toString(generation, Character.MAX_RADIX));

    //读取 Lucene 的版本号
    Version luceneVersion = Version.fromBits(input.readVInt(), input.readVInt(), input.readVInt());
    if (luceneVersion.onOrAfter(Version.LUCENE_6_0_0) == false) {
      // TODO: should we check indexCreatedVersion instead?
      throw new IndexFormatTooOldException(input, "this index is too old (version: " + luceneVersion + ")");
    }

    int indexCreatedVersion = 6;
    if (format >= VERSION_70) { // 如果版本是 7 以后的，则 indexCreatedVersion 从 input 中获取
      indexCreatedVersion = input.readVInt();
    }

    //
    SegmentInfos infos = new SegmentInfos(indexCreatedVersion);
    infos.id = id;
    infos.generation = generation;
    infos.lastGeneration = generation;
    infos.luceneVersion = luceneVersion;

    infos.version = input.readLong();
    //System.out.println("READ sis version=" + infos.version);
    if (format > VERSION_70) {
      infos.counter = input.readVLong();
    } else {
      infos.counter = input.readInt();
    }
    int numSegments = input.readInt();
    if (numSegments < 0) {
      throw new CorruptIndexException("invalid segment count: " + numSegments, input);
    }

    if (numSegments > 0) {
      infos.minSegmentLuceneVersion = Version.fromBits(input.readVInt(), input.readVInt(), input.readVInt());
    } else {
      // else leave as null: no segments
    }

    long totalDocs = 0;
    for (int seg = 0; seg < numSegments; seg++) {
      String segName = input.readString();
      if (format < VERSION_70) {
        byte hasID = input.readByte();
        if (hasID == 0) {
          throw new IndexFormatTooOldException(input, "Segment is from Lucene 4.x");
        } else if (hasID != 1) {
          throw new CorruptIndexException("invalid hasID byte, got: " + hasID, input);
        }
      }
      byte[] segmentID = new byte[StringHelper.ID_LENGTH];
      input.readBytes(segmentID, 0, segmentID.length);
      Codec codec = readCodec(input);
      SegmentInfo info = codec.segmentInfoFormat().read(directory, segName, segmentID, IOContext.READ);
      info.setCodec(codec);
      totalDocs += info.maxDoc();
      long delGen = input.readLong();
      int delCount = input.readInt();
      if (delCount < 0 || delCount > info.maxDoc()) {
        throw new CorruptIndexException("invalid deletion count: " + delCount + " vs maxDoc=" + info.maxDoc(), input);
      }
      long fieldInfosGen = input.readLong();
      long dvGen = input.readLong();
      int softDelCount = format > VERSION_72 ? input.readInt() : 0;
      if (softDelCount < 0 || softDelCount > info.maxDoc()) {
        throw new CorruptIndexException("invalid deletion count: " + softDelCount + " vs maxDoc=" + info.maxDoc(), input);
      }
      if (softDelCount + delCount > info.maxDoc()) {
        throw new CorruptIndexException("invalid deletion count: " + softDelCount + delCount + " vs maxDoc=" + info.maxDoc(), input);
      }
      SegmentCommitInfo siPerCommit = new SegmentCommitInfo(info, delCount, softDelCount, delGen, fieldInfosGen, dvGen);
      siPerCommit.setFieldInfosFiles(input.readSetOfStrings());
      final Map<Integer,Set<String>> dvUpdateFiles;
      final int numDVFields = input.readInt();
      if (numDVFields == 0) {
        dvUpdateFiles = Collections.emptyMap();
      } else {
        Map<Integer,Set<String>> map = new HashMap<>(numDVFields);
        for (int i = 0; i < numDVFields; i++) {
          map.put(input.readInt(), input.readSetOfStrings());
        }
        dvUpdateFiles = Collections.unmodifiableMap(map);
      }
      siPerCommit.setDocValuesUpdatesFiles(dvUpdateFiles);
      infos.add(siPerCommit);

      Version segmentVersion = info.getVersion();

      if (segmentVersion.onOrAfter(infos.minSegmentLuceneVersion) == false) {
        throw new CorruptIndexException("segments file recorded minSegmentLuceneVersion=" + infos.minSegmentLuceneVersion + " but segment=" + info + " has older version=" + segmentVersion, input);
      }

      if (infos.indexCreatedVersionMajor >= 7 && segmentVersion.major < infos.indexCreatedVersionMajor) {
        throw new CorruptIndexException("segments file recorded indexCreatedVersionMajor=" + infos.indexCreatedVersionMajor + " but segment=" + info + " has older version=" + segmentVersion, input);
      }

      if (infos.indexCreatedVersionMajor >= 7 && info.getMinVersion() == null) {
        throw new CorruptIndexException("segments infos must record minVersion with indexCreatedVersionMajor=" + infos.indexCreatedVersionMajor, input);
      }
    }

    infos.userData = input.readMapOfStrings();

    CodecUtil.checkFooter(input);

    // LUCENE-6299: check we are in bounds
    if (totalDocs > IndexWriter.getActualMaxDocs()) {
      throw new CorruptIndexException("Too many documents: an index cannot exceed " + IndexWriter.getActualMaxDocs() + " but readers have total maxDoc=" + totalDocs, input);
    }

    return infos;
  }
```





readCommit函数首先创建一个ChecksumIndexInput，然后通过readCommit函数读取段信息并返回一个SegmentInfos，这里的readCommit函数和具体的segments_*文件格式和协议相关，这里就不往下看了。最后返回的SegmentInfos保存了段信息。

回到IndexWriter的构造函数中，如果readLatestCommit函数返回的SegmentInfos不为空，就调用其clear清空，如果是第一次创建索引，就会构造一个SegmentInfos，SegmentInfos的构造函数为空函数。接下来调用SegmentInfos的createBackupSegmentInfos函数备份其中的SegmentCommitInfo信息列表，该备份主要是为了回滚rollback操作使用。IndexWriter然后调用changed表示段信息发生了变化。

继续往下看IndexWriter的构造函数，pendingNumDocs函数记录了索引记录的文档总数，globalFieldNumberMap记录了该段中Field的相关信息，getFlushPolicy返回在LiveIndexWriterConfig构造函数中创建的FlushByRamOrCountsPolicy，然后通过FlushByRamOrCountsPolicy的init函数进行简单的赋值。再往下创建了一个DocumentsWriter，并获得其事件队列保存在eventQueue中。IndexWriter的构造函数接下来会创建一个IndexFileDeleter，IndexFileDeleter用来管理索引文件，例如添加引用计数，在多线程环境下操作索引文件时可以保持同步性。

# 参考链接：

[lucene源码分析---2](https://blog.csdn.net/conansonic/article/details/51879376)















