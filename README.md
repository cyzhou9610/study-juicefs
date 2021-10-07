```
任何存入 JuiceFS 的文件都会被拆分成固定大小的 "Chunk"，默认的容量上限是 64 MiB
每个 Chunk 由一个或多个 "Slice" 组成，Slice 的长度不固定，取决于文件写入的方式。
每个 Slice 又会被进一步拆分成固定大小的 "Block"，默认为 4 MiB。
最后，这些 Block 会被存储到对象存储。
与此同时，JuiceFS 会将每个文件以及它的 Chunks、Slices、Blocks 等元数据信息存储在元数据引擎中。
```

1. blocks的命名规则定义，S3中key的名称。


```go
func (c *rChunk) key(indx int) string {
if c.store.conf.Partitions > 1 { //支持使用多个bucket存储
return fmt.Sprintf("chunks/%02X/%v/%v_%v_%v", c.id%256, c.id/1000/1000, c.id, indx, c.blockSize(indx))
}
return fmt.Sprintf("chunks/%v/%v/%v_%v_%v", c.id/1000/1000, c.id/1000, c.id, indx, c.blockSize(indx))
}
```

2. 文件系统定义核心数据结构

```go
type FileSystem struct {
    conf   *vfs.Config      //本地缓存、元数据引擎连接信息等相关的配置
    reader vfs.DataReader   //数据处理相关 读取
    writer vfs.DataWriter   //数据处理相关 写入
    m      meta.Meta        //数据库相关的接口 redis
    logBuffer chan string
}
```

存储的三个层级：ChunkStore 

cacheStore基于内存缓存 diskStore本地文件系统

3. ChunkStore的抽象接口定义

```go
type ChunkStore interface {
    NewReader(chunkid uint64, length int) Reader //实现数据的读写相关原子接口
    NewWriter(chunkid uint64) Writer //实现数据的读写相关原子接口
    Remove(chunkid uint64, length int) error
    FillCache(chunkid uint64, length uint32) error
    UsedMemory() int64
}

```

实现ObjectStore接口，与JuiceFS对接

```go
type ObjectStorage interface {
	// Description of the object storage.
	String() string
	// Create the bucket if not existed.
	Create() error
	// Get the data for the given object specified by key.
	Get(key string, off, limit int64) (io.ReadCloser, error)
	// Put data read from a reader to an object specified by key.
	Put(key string, in io.Reader) error
	// Delete a object.
	Delete(key string) error

	// Head returns some information about the object or an error if not found.
	Head(key string) (Object, error)
	// List returns a list of objects.
	List(prefix, marker string, limit int64) ([]Object, error)
	// ListAll returns all the objects as an channel.
	ListAll(prefix, marker string) (<-chan Object, error)

	// CreateMultipartUpload starts to upload a large object part by part.
	CreateMultipartUpload(key string) (*MultipartUpload, error)
	// UploadPart upload a part of an object.
	UploadPart(key string, uploadID string, num int, body []byte) (*Part, error)
	// AbortUpload abort a multipart upload.
	AbortUpload(key string, uploadID string)
	// CompleteUpload finish an multipart upload.
	CompleteUpload(key string, uploadID string, parts []*Part) error
	// ListUploads lists existing multipart uploads.
	ListUploads(marker string) ([]*PendingPart, string, error)
}
```

文件操作流程：redis<–Meta<–data writer< -->file writer< -->chunk writer< -->slice writer，先写数据再更新元数据

```
func (r *redisMeta) inodeKey(inode Ino) string {
    return "i" + inode.String()
}

func (r *redisMeta) entryKey(parent Ino) string {
    return "d" + parent.String()
}

func (r *redisMeta) chunkKey(inode Ino, indx uint32) string {
    return "c" + inode.String() + "_" + strconv.FormatInt(int64(indx), 10)
}

func (r *redisMeta) sliceKey(chunkid uint64, size uint32) string {
    return "k" + strconv.FormatUint(chunkid, 10) + "_" + strconv.FormatUint(uint64(size), 10)
} // 这个从代码里看起来是合并chunk的时候为了区分slice所属chunk才用到的，我测试环境的redis里面没看到有这个key

// 用法示例：
pipe.Set(ctx, r.inodeKey(inode), r.marshal(&attr), 0)
pipe.HSet(ctx, r.entryKey(parent), name, r.packEntry(_type, ino))
pipe.RPush(ctx, r.chunkKey(inode, uint32(old/ChunkSize)), w.Bytes())  // inode->chunk-->slices
```
https://cloud.tencent.com/developer/article/1849514
