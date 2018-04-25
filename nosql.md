- 流存储
- 管道
- Amazon S3
- 


# A Functional Interface for Key/Value Store
>**键/值存储的功能接口**

Stew O'Connor, Sr Principal SWE • Mengxi Lu, Sr Principal SWE • Rolando Manrique, Principal SWE • 2018-04-06

At LendUp, we have the need to run multiple pipelines that transfer batched data across different systems via SFTP and Amazon S3 and, very commonly, storing temporary data transformations in local filesystems; all these storage mechanisms behave very similar and can be modeled by a common interface.

>译:在LendUp,我们需要运行多个管道,这些管道通过SFTP(Secure File Transfer Protocol)和Amazon S3在不同系统间传输批量数据,并且非常常见的是将临时数据转换存储在本地文件系统中;所有这些存储机制的行为非常的相似,并可以通过通用结构来建模.

Initial implementations using specific client libraries for each storage mechanism gradually gravitated towards bad software development practices that made code hard to test (i.e. required S3/SFTP credentials) and reduced performance and scalability limited by host memory or filesystem.

>译:针对每种存储机制使用特定的客户端库的初始实现逐渐倾向于使代码难以测试(即需要S3/SFTP整数)的不良软件开发实践,并降低了受主机内存或文件系统限制的性能和可扩展性.

By providing a generic Store implementation, abstracted from specific storage mechanisms, we are able to interchange specific implementations depending on the context calling the functions. This makes it very easy to provide mocks in unit tests to replace SFTP/S3 stores with a filesystem store in a controlled environment.

>译:通过提供一个从特定存储机制中抽象出来的通用接口,我们可以根据调用函数的上下文来交换具体的实现.这使得在单元测试中很容易的提供模拟,可以在一个受控环境中将文件系统存储替换SFTP/S3存储

fs2-blobstore is a minimal, idiomatic, stream-based Scala interface for key/value store implementations with the assumption that any key/value store must provide these 6 basic functions:

>译:fs2-blobstore 是一个最小的,惯用的,基于数据流的Scala的接口,用于key/value存储实现与任何key/value存储必须提供者些6种基本功能的假设:

- List keys (列出键)
- Get key’s value (得到键值)
- Put value in key (将值放入键中)
- Copy key value to some other key (将键值复制给其他键)
- Move key value to some other key (将键值移至其他键)
- Remove key (删除键)

fs2-blobstore provides scalability by relying on fs2 streams resource safety and concurrency primitives while providing a flexible abstraction that applies to different storage mechanisms.

>译:fs2-blobstore 通过依赖fs2流资源安全和并发原语提供了可扩展性,同时提供了一个适用于不同存储机制的灵活抽象.

### Stream-based Store
>**基于流的存储**

One of the main goals for fs2-blobstore is to provide scalable code that would allow us to grow our business from thousands to millions of users. With users growth, our data pipeline will need to be able to process larger and larger files.

>译:fs2-blobstore的主要目标之一是提供可扩展的代码,使我们能够将业务从数千人增加到数百万用户.随着用户的增长,我们的数据管道将需要能够处理越来越大的文件.

The first way to provide scale in a data pipeline is to avoid loading incoming data files in memory, off course, but also avoid writing to disk as much as possible, especially for intermediate steps.

>译:第一种方式是提供scala在一个数据管道中来避免在内存中加载传入的数据文件,(笔误的当然 还是 偏离轨道?),但也要避免写如磁盘等等,特别是一些中间步骤.

The initial pipeline implementation would make heavy use of temporary local disk storage before uploading files to the permanent S3 location for archiving. While yes, disks are cheap these days, this approach of using local storage as a staging area for all files transferred through the pipeline would eventually limit the ability to process files concurrently, as this temporary storage would get filled with intermediate transformations to the data and increases processing time as it reads and write data to filesystem.

>译:在将文件上传到永久S3位置进行归档之前,初始管道实施将大量使用临时本地磁盘存储.虽然是的,现在磁盘价格便宜,这种使用本地存储作为通过管道传输的所有文件的临时区域的方法最终会限制并发处理文件的能力,因为这种临时存储会被数据的中间转换填满.在读取和写入数据到文件系统时增加处理时间.

In a stream-based store interface built with Functional Streams for Scala (fs2) we can transfer the stream of bytes coming from SFTP into the stream of bytes going into S3 without ever storing these bytes in the local filesystem. There are times when it is required to use local filesystem for intermediate storage, but, by avoiding unnecessary writes we allow for larger scale overall.

>译:用`Functional Streams for Scala(fs2)`构建的基于流的存储接口中我们可以将来自SFTP的字节流传递给S3而不会将这些字节存储在本地文件系统.时需要使用本地文件系统进行中间存储,但是,通过避免不必要的写入,我们可以在整体上扩大规模.

### Modeling a Key/Value Store
>**建立一个键值存储模型**

As mentioned above, fs2-blobstore models a very specific type of store from Path to Bytes, and guarantees a very strong contract and expectation when providing store implementations.

>译:如上所述,fs2-blobstore 建立了一个从Path到Bytes的一个非常具体的存储方式,并在提供存储实现时确保有一个非常强的*合同*和*期望*.

The key is modeled as:

>key的建模为:

```scala
case class Path( // 类似于构造函数 "()"里面的是参数 VariableName : DataType
  root: String,  
  key: String,
  size: Option[Long], //Option 是选项类型
  isDir: Boolean,
  lastModified: Option[Date] 
)
```

Functions in the Store interface that receive a Path (get, put, copy, move, remove) assume that only root and key values are mandatory, there is no guarantee that the other attributes of Path would be accurate: size, isDir, lastModified. On the other hand, when a Store implements the list function, it is expected that all 3 fields will be present in the response.

>译:Store的接口方法接受`Path(get, put, copy, move, remove)`确保只有`root`和`key values`是的,不能保证Path中的其他属性`(size, isDir, lastModified)`也是准确的.换句话说,当存储实现列表方法是,估计这3个字段全部present在响应中.

Given that the key is modeled as Path and value is modeled as bytes, we can define the Store interface with the six basic functions listed above like this:

>译: 假设key被建模为Path,并且值被建模为字节,我们可以定义存储接口如下列六个寄出方法.

```scala
trait Store[F[_]] {  //trait 类似于Java中的接口,但比接口更强大 ·def 函数名(参数):返回值类型·
  def list(path: Path): fs2.Stream[F, Path]
  def get(path: Path, chunkSize: Int): fs2.Stream[F, Byte]
  def put(path: Path): fs2.Sink[F, Byte]
  def move(src: Path, dst: Path): F[Unit]
  def copy(src: Path, dst: Path): F[Unit]
  def remove(path: Path): F[Unit]
}
```

These six functions can be composed to provide any functionality required when manipulating data across stores. Note that the get function requires the “chunk size” in the basic interface but by importing Store syntax you can use functions like these that default to 4k buffer size:

>译:在存储键操作数据时这六个方法被组合起来能够提供任何所需的功能.注意在基础接口中get方法需要"chunk size"但是通过提供Store语法你可以使用默认缓冲区大小为4k的方法.

```scala
// get Stream with default 4k chunk size
def get(path: Path): Stream[F, Byte]

// consume contents stream and return as UTF-8 String
def getContents(path: Path)(implicit F: Effect[F]): F[String]
```

It is also highly recommended to provide the optional path size when calling Store.put function because of Amazon’s S3 upload requirements; S3Store.put will use the provided path size when starting the data upload to S3. If no size is provided, the Amazon S3 client will warn that all data will be loaded into memory first, compute the size, and then start transferring data to S3 which would cause out of memory exception when attempting to transfer large amounts of data.

>译:由于Amazon's S3上传了需求,强烈建议在调用`Store.put`方法时提供自选路径大小.开始上传数据到S3时,`S3Store.put`将使用提供的路径大小.如果没有提供(路径大小),`Amazon S3`客户端将警告,所有的数据先被加载到内存,计算大小,然后开始传输数据到S3,这是因为尝试传大量数据是可能造成超内存的异常.

### Composing Streams of Bytes
> **构成字节流**

Now that we have defined the Store interface we can get to more complex functions to demonstrate the capabilities of the composable Streams. Going back to LendUp’s data pipelines, a very common use case is to transfer files across stores, regardless of the Store implementation, transfering a file can be achieved with:

>译:现在已经定义了Store接口,我们可以使用更复杂的方法来演示组合Streams的能力.回到**LendUp**的数据管道,一个很普通的用例是通过存储来传输文件,无论`Store`怎么实现,传输一个文件都可以通过以下方式实现:

```scala
def transferFile[F[_]](
  srcStore: Store[F],
  dstStore: Store[F],
  path: Path): fs2.Stream[F, Unit] = {
  srcStore.list(path).filter(!_.isDir).evalMap { p =>
    val contents: fs2.Stream[F, Byte] = store.get(p, 4096)
    val sink: fs2.Sink[F, Byte] = dstStore.put(p)
    val transfer: fs2.Stream[F, Unit] = contents to sink
    transfer.compile.drain
  }
}    
```

Notice that this transfer implementation does not assume that provided path has a the correct size of the file to transfer. Instead it makes the best effort to compute the size by calling list before starting data transfer. This implementation also guarantees that data is not stored in memory or local file system at any point, not even if the destination store bandwidth is much slower than the source store bandwidth. Data will be pulled from the source store socket only as fast as we can write into the destination store. This is because fs2 is naturally “pull based”, and will only demand bytes from a source when they are needed. This eliminates the need for explicit back-pressure, as would be required from a “push-based” implementation.

>译:注意这个传输的实现不确保提供的路径有要传输文件的正确大小.相反它尽力通过在传输开始前估计列表计算大小.这个实现也保证数据在任何时刻都不被存储在内存或者本地文件系统中.及时目标存储带宽比源存储带宽慢的多.数据只有我们能够一样快的写入目标存储才能从源存储socket(插槽 or 套接字?)中提取.

Transferring data across stores is such a common use case that it is already provided by importing Store syntax and it is a nicer version that allows to transfer nested directories recursively:

>译:跨存储传输数据是一个常见的用例,它通过导入`Store`语法提供,并且是允许以地柜方式传输嵌套目录的更高的版本.

```scala
import blobstore.implicits._
srcStore.transferTo(dstStore, srcPath, dstPath)
```

### Next Steps
>**接下来**

There are some features to complete that while not difficult to implement we just have not had the time to do so. It would be very simple to implement an in-memory store that would be even better mock Store than a filesystem based Store. Also features like listing recursively would be easy to implement in the three current implementations of Store.

>还有一些功能要完成,虽然不能完成,但我们没有时间那么做了.这是一个非常简单的方式去实现一个基于内存的存储,*甚至比mock Store 一个基于文件系统的存储更好*.同样的像递归列表在三个当前实现下很容易的就能实现.

Other features are not so straight forward, allowing stores to implement specific behaviors related to their underlying system are not trivial, especially when putting content as each storage mechanism provides custom features that may not be available for all, for example, S3 allows to write data with server side encryption, different ACLs, among many options, while a filesystem allows to append data to an existing file; these specific features may prove difficult to implement while preserving the abstraction.

>译:其他的功能不是那么直接,允许存储实现基本有关底层系统的行为是微不足道的.特别的在将内容作为每个存储机制提供可能不适用所有人的自定义功能时,例如`S3`允许写入数据与服务端紧密相连,不同的`ACLs`,许多选项,当一个文件系统允许向一个已存在的文件追加数据是,在保留抽象的同时,这些特定的功能可能难以实现.

If this looks like something you could use in your projects, head over to our git repo for instructions and samples and join the conversation in our gitter channel.

>译:如果这看起来像是能够用在您的项目中的东西,请前往我的`git repo`获取说明和示例,并在我们的`gitter`频道中与我们联系.
