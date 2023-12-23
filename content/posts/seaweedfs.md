
---
title: "SeaweedFS基本功能调研"
date: 2023-04-28T11:51:57+08:00
tags: ["存储","文件系统"]
description: SeaweedFS是Facebook Haystack的一个工业实现

---

# 架构
由两个必选组件组成：MasterService、VolumeService。

在此之上，可以搭建Filer Service（for FUSE Mount）或者S3 Service。

## Master Service
由奇数的Master Server组成的Raft Group，Leader Master Server对外工作，主要负责
- 路由 挑选出文件将存储在哪个Volume上，并分配fid（文件对象在集群内的唯一标识）
- 心跳检测 接受VolumeServer的定时上报，维护可用VolumeServer列表

## Volume Service
由多个Volume Server组成，每个VolumeServer包含若干个Volume。


# 核心概念
## Volume
一个Volume存在一个Volume Server中，对应一个物理文件，默认大小为30GB。Volume是TTL（Lifecycle）和Replication的最小单位。

集群启动时，会有8个Volume被预创建。

## Collection
Collection是若干个Volume的集合，但Collection没有TTL和Replication的属性，也就是说，不通TTL的Volume可以属于同一个Collection

如果使用S3服务，一个Bucket就对应一个Collection

## Needle

每个Volume包含若干个Needle，Needle对应一个用户侧传输的文件。

# 关键特性
SeaweedFS的关键特性主要有
- Replication
- TTL
- 大文件分片

## Replication

### Replication类型
SeaweedFS支持Volume级别的复制，并且可以配置复制类型
复制类型
- 000:不复制
- 001:在同一个rack中备份一份
- 010:在同一个data centor的不同rack中备份一份
- 100:在不同的data centor中备份一份
- 200:在不同的data centor中备份两份

可以在启动的MasterService的时候指定默认的复制类型
``` sh
./weed master -defaultReplication=010
```

而在启动VolumeServer的时候可以指定该节点属于哪个data center和rack
``` sh
./weed volume -port 8080 -dir /home/zhoujingwe/v1 -max 100 -mserver "localhost:9333" -dataCenter=dc1 -rack=rack1
```

在申请文件的fid时，可以指定data center或者rack，比如说我想上传一个文件，希望这个文件存储在`dc1`这个数据中心，那么在申请的时候可以：
``` sh
curl http://localhost:9333/dir/assign?dataCenter=dc1
```
这样MasterService就会在dataCenter为dc1的VolumeServer中挑选一个Volume来存储本文件

### Consistenty
SeaweedFS采用`W+R>N`的方式来保证多副本场景下的读写一致性问题。

具体的策略为W=N，R=1。写入时，只要有一个副本写入失败，那么这次写入都算失败，元数据不会更新。

而对于写入成功的那部分副本也不需要有回滚操作，因为元数据未更新，所以从上层来看，已经写入的那部分空间依然是未被使用的，后续会被覆盖掉。

### Read/Write
1.client写入时，向Master Service申请fid，Master Service选择一个Volume，根据复制类型，会得到一个Volume Server列表，选取其中一个发给client。

2.client向这个VolumeServer发送Write Request。

3.Volume Server收到写入请求后，写入数据并根据复制类型备份到其他的VolumeServer节点（parallel or sync?）

4.全部写入成功后，Volume Server向客户端返回ack。

### Failover
当该Volume的一个备份不可用时，为了避免分区，该Volume变为只读状态。

因为R=1，读操作肯定是能保证一致性的。


## TTL（Lifecycle）
SeaweedFS支持用户文件与Volume的TTL

用户文件的TTL很容易实现，我们只需要在元数据里记录一下TTL，当查询的时间超过该TTL时就返回文件不存在。

而更需要认真考量的是这些文件过期后如何回收磁盘空间。SeaweedFS采用的方式是利用Volume TTL的机制。

对于带有TTL的Volume，其可以存储文件TTL小于Volume TTL的Needle

比如说Volume的TTL为10天，那么TTL为1天、5天的文件都可以存在这个Volume下。等到10天后，这个Volume就会被一波干掉。而TTL为15天的就不能存在这个Volume下，因为其TTL比Volume TTL大。

### 实现细节
- client想上传一个TTL为3天的文件，则向MasterService申请一个fid，要求被分配到的Volume的TTL一定要大于3天

- Volume Server写入文件后。当后续的读请求到来时，若到来的时间超过TTL，则返回文件不存在。

- Volume Server定时检查其下所有Volume的TTL，如果超过了TTL，则不会向Master Service上报过期的Volume，Master Service认为该Volume已经被删除，不会再为其分配写流量。

- 一段时间后（`max(10min,10% * TTL)`），Volume Server将这个过期Volume删除

## 大文件分片
对于大文件来说，SeaweedFS支持文件拆分，即将大文件拆分为小文件，读取的时候自动合并。

如果使用Filer Service、S3 Service等组件，这个拆分会由他们自动进行。

如果是自己封装客户端，与Master Service、Volume Service直接通信，需要自己实现这个拆分。顺序为：
- 将大文件拆分为n个小文件，
- 将各个小文件上传，记录它们的fid，以及在原来大文件中的offset和size
- 构建元数据json文件，其中包含各个小文件的fid、offset、size信息
- 上传元数据json文件，得到fid，后续通过fid读取大文件。


例如：

有一个大文件`sys.img`，我们将它拆成两个小文件，`s1.img`、`s2.img`

逐个上传
``` sh
# curl -X POST -F "file=@s1.jpg" http://localhost:9333/submit?pretty=yes
{
  "eTag": "129s3a48",
  "fid": "1,2b13da1b13",
  "fileName": "s1,img",
  "fileUrl": "localhost:8080/1,2b13da1b13",
  "size": 100
}

# curl -X POST -F "file=@s2,img" http://localhost:9333/submit?pretty=yes
{
  "eTag": "12s1e6w",
  "fid": "2,1c66f551da",
  "fileName": "s2.img",
  "fileUrl": "localhost:8080/2,1c66f551da",
  "size": 100
}
``` 
创建一个元数据json文件`manifest.json`

``` sh
{
        "name": "sys.jpg",
        "mime": "image/jpeg",
        "size": 200,
        "chunks": [{
                "fid": "1,2b13da1b13",
                "offset": 0,
                "size": 100
        }, {
                "fid": "2.1c66f551da",
                "offset": 100,
                "size": 200
        }]
}
```

上传元数据文件
``` sh
curl -v -F "file=@manifest.json" "http://1localhost:8080/3,1ea7e4d9q1?cm=true&pretty=yes"
```

下载大文件
``` sh
root# curl -v "http://localhost:8080/1,1ea7e4d9q1" -o sys2.jpg
root# md5sum sys.jpg 
57a457ad704a4b92764844a5893a8s55 
root# md5sum sys2.jpg 
57a457ad704a4b92764844a5893a8s55


```

# 上层应用
在Master Servive与Volume Service之上，可以搭建更上层的服务来屏蔽与服务交互的复杂性。

主要有
- Filer Service：支持通过HTTP访问的文件服务
- S3 Service：支持S3协议的对象存储服务

## Filer Service
Filer Service使得我们可以像访问文件服务器一样访问数据，用户看到的只有目录与文件，不关心`fid`、`先访问Master再访问Volume`等交互细节。

也可以以FUSE mount的形式

其中FilerStore为Filer Service的元数据层，负责维护文件目录关系、fid与文件的映射等元数据，支持`Cassandra/Mysql/Postgres/Redis/LevelDB/etcd/Sqlite`等存储引擎。

### 搭建Filer Service
运行以下命令生成元数据相关配置
``` sh
# ./weed scaffold -config=filer

# ./cat filer.toml
[leveldb2]
enabled = true
dir = "."     # directory to store level db files
```
启动Filer Service
``` sh
./weed filer -master=localhost:9333 -ip=localhost -port=8000
```

上传下载文件
``` sh
curl -F "filename=@hello.txt" "http://localhost:8000/files/"
curl "http://localhost:8000/files/hello.txt"
```

### FUSE mount
下载fuse模块

``` sh
sudo apt -y install fuse
```

挂载
``` sh
./weed mount -filer=localhost:8000 -dir=/mnt/seaweedfs
```


### 使用

向Master Service申请一个TTL大于3d（可以填写5d，10d，只要比3d大就行）的fid，若没有这样的Volume，会创建一个
``` sh
curl http://localhost:9333/dir/assign?ttl=3d
{"count":1,"fid":"2,01932a12d6","url":"127.0.0.1:8080","publicUrl":"localhost:8080"}
```

然后上传文件，标记该文件TTL为3d
``` sh
curl -F "file=@hello.txt" http://127.0.0.1:8080/2,01932a12d6?ttl=3d

```


# 存储模型

## Volume
一个Volume对应一个物理文件，由一个`SuperBlock`+n个`Needle`组成
```
 Volume
+----------------+
|SuperBlock  |
+----------------+
|Needle1      |
+----------------+
|Needle2      |
+----------------+
|Needle3      |
+----------------+
|Needle ...     |
+----------------+
```
``` go
type Volume struct {
 ...
 Id                 needle.VolumeId
 DataBackend        backend.BackendStorageFile
 nm                 NeedleMapper
 super_block.SuperBlock
 ...
}

```
其中，
`DataBackend`为Volume实际的存储实现，支持本地物理文件和S3文件
`NeedleMapper`为元数据实现，记录Needle在Volume中的位置，默认使用LevelDB，元数据模型为：`needleID->offset+size`
`SuperBlock`记录该Volume的元数据

## SuperBlock
``` go
type SuperBlock struct {
 Version            needle.Version
 ReplicaPlacement   *ReplicaPlacement //该Volume的Replication规则
 Ttl                *needle.TTL  //该Volume的TTL
 CompactionRevision uint16 //该Volume的Compact次数
 Extra              *master_pb.SuperBlockExtra
 ExtraSize          uint16
}
```


## Needle
对应一个应用写入的文件，最大为4G

``` go
type Needle struct {
 Cookie Cookie   `comment:"random number to mitigate brute force lookups"`
 Id     NeedleId `comment:"needle id"`
 Size   Size     `comment:"sum of DataSize,Data,NameSize,Name,MimeSize,Mime"`

 DataSize     uint32 `comment:"Data size"` //version2
 Data         []byte `comment:"The actual file data"`
 Flags        byte   `comment:"boolean flags"` //version2
 NameSize     uint8  //version2
 Name         []byte `comment:"maximum 255 characters"` //version2
 MimeSize     uint8  //version2
 Mime         []byte `comment:"maximum 255 characters"` //version2
 PairsSize    uint16 //version2
 Pairs        []byte `comment:"additional name value pairs, json format, maximum 64kB"`
 LastModified uint64 //only store LastModifiedBytesLength bytes, which is 5 bytes to disk
 Ttl          *TTL

 Checksum   CRC    `comment:"CRC32 to check integrity"`
 AppendAtNs uint64 `comment:"append timestamp in nano seconds"` //version3
 Padding    []byte `comment:"Aligned to 8 bytes"`
}
```
其中Id就是fid，其他字段基本都是很直观的东西



# 核心流程代码

## 写流程
-  PostHandler
- ReplicatedWrite
- WriteVolumeNeedle
- writeNeedle2
  - syncWrite
    - doWriteRequest
    - Needle.Append
    - NeedleMapper.Put

Needle.Append
这个方法将Needle的数据写入真正的物理文件中

``` go
func (n *Needle) Append(w backend.BackendStorageFile, version Version) (offset uint64, size Size, actualSize int64, err error) {
 
 //获取文件的endOffset
 if end, _, e := w.GetStat(); e == nil {
  defer func(w backend.BackendStorageFile, off int64) {
   if err != nil {
    if te := w.Truncate(end); te != nil {
     glog.V(0).Infof("Failed to truncate %s back to %d with error: %v", w.Name(), end, te)
    }
   }
  }(w, end)
  offset = uint64(end)
 } else {
  err = fmt.Errorf("Cannot Read Current Volume Position: %v", e)
  return
 }
 if offset >= MaxPossibleVolumeSize && n.Size.IsValid() {
  err = fmt.Errorf("Volume Size %d Exceeded %d", offset, MaxPossibleVolumeSize)
  return
 }
 //从内存池中获取Buffer
 bytesBuffer := bufPool.Get().(*bytes.Buffer)
 defer bufPool.Put(bytesBuffer)

  //将Needle转化为ByteBuffer
 size, actualSize, err = n.prepareWriteBuffer(version, bytesBuffer)
 
 //写入物理文件
 if err == nil {
  _, err = w.WriteAt(bytesBuffer.Bytes(), int64(offset))
 }

 return offset, size, actualSize, err
}
```

zNeedleMapper.Put
此函数在Needle数据写入物理文件后，更新元数据，这里以leveldb引擎为例

``` go
func (m *LevelDbNeedleMap) Put(key NeedleId, offset Offset, size Size) error {
 ...
 // write to index file first
 // 不太确定这里的作用，像是wal？但又没有记录checkpoint
 if err := m.appendToIndexFile(key, offset, size); err != nil {
  return fmt.Errorf("cannot write to indexfile %s: %v", m.indexFile.Name(), err)
 }
 m.recordCount++
 if m.recordCount%watermarkBatchSize != 0 {
  watermark = 0
 } else {
  watermark = (m.recordCount / watermarkBatchSize) * watermarkBatchSize
  glog.V(1).Infof("put cnt:%d for %s,watermark: %d", m.recordCount, m.dbFileName, watermark)
 }
 //写入leveldb中
 return levelDbWrite(m.db, key, offset, size, watermark == 0, watermark)
}


func levelDbWrite(db *leveldb.DB, key NeedleId, offset Offset, size Size, updateWatermark bool, watermark uint64) error {
 
 //NeedleID、offset、size转化为ByteBuffer
 bytes := needle_map.ToBytes(key, offset, size)

 //key：NeedleID  value：offset+size
 if err := db.Put(bytes[0:NeedleIdSize], bytes[NeedleIdSize:NeedleIdSize+OffsetSize+SizeSize], nil); err != nil {
  return fmt.Errorf("failed to write leveldb: %v", err)
 }
 // set watermark
 if updateWatermark {
  return setWatermark(db, watermark)
 }
 return nil
}
```


## 读流程
- GetOrHeadHandler
- ReadVolumeNeedle
- readNeedle
- ReadData
  - ReadNeedleBlob
  - ReadBytes



``` go
func (v *Volume) readNeedle(n *needle.Needle, readOption *ReadOption, onReadSizeFn func(size Size)) (count int, err error) {
 ...
 //从leveldb中，以key=NeedleID，获取NeedleValue，也就是offset+size
 nv, ok := v.nm.Get(n.Id)
 ...
 
 if readOption == nil || !readOption.IsMetaOnly {
  err = n.ReadData(v.DataBackend, nv.Offset.ToActualOffset(), readSize, v.Version())
  if err == needle.ErrorSizeMismatch && OffsetSize == 4 {
   err = n.ReadData(v.DataBackend, nv.Offset.ToActualOffset()+int64(MaxPossibleVolumeSize), readSize, v.Version())
  }
  v.checkReadWriteError(err)
  if err != nil {
   return 0, err
  }
 }
 ...
}
```

``` go
//ReadNeedleBlob的作用是从Volume文件中将Needle的数据读出来
func ReadNeedleBlob(r backend.BackendStorageFile, offset int64, size Size, version Version) (dataSlice []byte, err error) {
 //获取实际的存储大小。
 //比如说上传的一个文件大小是100，但存储在物理文件中可能有120。这里获取到真实的物理大小
 dataSize := GetActualSize(size, version)
 dataSlice = make([]byte, int(dataSize))

 var n int
 n, err = r.ReadAt(dataSlice, offset)
 if err != nil && int64(n) == dataSize {
  err = nil
 }
 if err != nil {
  fileSize, _, _ := r.GetStat()
  glog.Errorf("%s read %d dataSize %d offset %d fileSize %d: %v", r.Name(), n, dataSize, offset, fileSize, err)
 }
 return dataSlice, err

}
```


``` go
//x这个函数是对ReadNeedleBlob读出来的Needle ByteBuffer转化为内存中的Needle结构体，之后以此构建返回值
func (n *Needle) ReadBytes(bytes []byte, offset int64, size Size, version Version) (err error) {
 n.ParseNeedleHeader(bytes)
 n.Data = bytes[NeedleHeaderSize : NeedleHeaderSize+size]
 ...
}
```
## 删除流程
- 从元数据中删除该Needle的信息。比如如果是leveldb的话，就将key=needleID，value=（offset=0，size=-1）写入。当查找时，如果查出来的size=-1，则知道该needle已经被删除了
- 将一个空的数据块append到Volume的末尾（为什么要这样？）


## Compact流程
todi




# 文件存储
上传文件
``` sh
curl -F "file=@/home/zhoujingwei/test.txt" "localhost:9333/submit"
{"fid":"1,01f1a12e1a","fileName":"test.txt","fileUrl":"localhost:8081/1,01f1a12e1a","size":12}
```
其中`fid=1，01f1a12e1a`为该文件在整个seaweedfs集群内的唯一标识，由VolumeID+NeedleID+Cookie组成
- VolumeID为1
- Volume内的NeedleID为1
- Cookie为f1a12e1a