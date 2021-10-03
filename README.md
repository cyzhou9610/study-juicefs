blocks的命名规则定义，也就是最终存储在对象存储系统中的对象key名称。

```go
func (c *rChunk) key(indx int) string {
if c.store.conf.Partitions > 1 {
return fmt.Sprintf("chunks/%02X/%v/%v_%v_%v", c.id%256, c.id/1000/1000, c.id, indx, c.blockSize(indx))
}
return fmt.Sprintf("chunks/%v/%v/%v_%v_%v", c.id/1000/1000, c.id/1000, c.id, indx, c.blockSize(indx))
}
```

文件系统定义核心数据结构

```go
type FileSystem struct {
conf   *vfs.Config
reader vfs.DataReader
writer vfs.DataWriter
m      meta.Meta
logBuffer chan string
}
```

ChunkStore的抽象接口定义

```go
type ChunkStore interface {
NewReader(chunkid uint64, length int) Reader
NewWriter(chunkid uint64) Writer
Remove(chunkid uint64, length int) error
FillCache(chunkid uint64, length uint32) error
UsedMemory() int64
}

```


https://cloud.tencent.com/developer/article/1849514
