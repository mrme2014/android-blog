---
title: 实战：基于Room封装APP离线缓存框架HiStorage
---

<!--more-->

## 目录

- 需求分析
- 疑难解惑
- Coding

### 需求分析

- 基于Room数据库实现离线缓存框架HiStorage
- 仅仅只对外暴露 save、delete、get三种方法

```kotlin
HiStorage.saveCache(String key,List<GoodsModel>)//GoodsModel,"cache"

HiStorage.getCache(key:String)

HiStorage.deleteCache(key:String)
```



### 疑难解惑

1.添加依赖

```groovy
def room_version = "2.2.0-alpha01"
implementation "androidx.room:room-runtime:$room_version"
kapt "androidx.room:room-compiler:$room_version"
```



2.因为需要缓存的数据结构千变万化，数据模型也不尽相同，所以需要一个通用的数据持久化方案

```kotlin
@Entity(tableName = "cache")
class Cache  {
    @PrimaryKey(autoGenerate = false)
    @NonNull
    var key: String = ""
    //缓存数据的二进制
    var data: ByteArray? = null
}
```

3.持久化时，需要把object数据转换成二进制。body以及内嵌对象需要实现Serializable

```kotlin
    //序列化存储数据需要转换成二进制
    private fun <T> toByteArray(body: T): ByteArray {
        val baos = ByteArrayOutputStream()
        val oos = ObjectOutputStream(baos)
        oos.writeObject(body)
        oos.flush()
        return baos.toByteArray()
   }
```

4.读取缓存,需要反序列化。可自动完成二进制到object的转换

```kotlin
//反序列,把二进制数据转换成java object对象
    private fun toObject(data: ByteArray?): Any? {
        val bais = ByteArrayInputStream(data)
        val ois = ObjectInputStream(bais)
        return ois.readObject()
    }
```

.定义数据库操作对象

```kotlin
@Dao
interface CacheDao {
    @Insert(onConflict = OnConflictStrategy.REPLACE)
    fun saveCache(cache: Cache): Long

    @Query("select *from cache where `key`=:key")
    fun getCache(key: String): Cache?

    @Delete
    fun deleteCache(cache: Cache): Int
}
```

6.定义缓存数据库CacheDatabase

```kotlin
@Database(
    entities = [Cache::class],
    version = 1
) 
abstract class CacheDatabase : RoomDatabase() {
    companion object {
        private var database: CacheDatabase
        fun get(): CacheDatabase {
            return database
        }
        init {
            database = Room.databaseBuilder(
                AppGlobals.get()!!,
                CacheDatabase::class.java, "howow_cache"
            ) .build()
        }
    }
    //获取操作数据库数据的dao对象
    abstract val getCacheDao: CacheDao
}
```

7.HiStorage封装

```kotlin
object HiStorage{
   fun  deleteCache(key: String) {
        val cache = Cache()
        cache.key = key
        CacheDatabase.get().cache.delete(cache)
    }

    fun <T> saveCache(key: String, body: T) {
        val cache = Cache()
        cache.key = key
        cache.data = toByteArray(body)
        CacheDatabase.get().cache.save(cache)
    }

    fun <T> getCache(key: String): T? {
        val cache: Cache? = CacheDatabase.get().cache.getCache(key)
        return (if (cache?.data != null) {
            toObject(cache.data)
        } else null) as? T?
    }
  
}
```




