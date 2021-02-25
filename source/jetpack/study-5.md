---
title: Room架构组件原理解析
---

<!--more-->

## 目录
- 什么是Room
- Room高频用法
- Room数据库创建实现原理
- Room与LiveData的巧妙结合,数据变更监听



### 什么是Room

- 轻量级orm数据库,本质上是一个SQLite抽象层。使用更加简单（类似于Retrofit库）

- 开发阶段通过注解的方式标记相关功能。 编译时自动生成响应的impl实现类

- 丰富的编译时校验，错误提示

  

![room](/imgs/jetpack/room.png)

### Room高频用法

- sqlite数据库创建
  - SQL语句的正确性及安全性都没有保证，问题被延迟到了运行时才能被发现
  - 极容易在主线程对数据库进行操作
  - 从数据库数据到我们所需要的类数据之间的转换繁琐

```kotlin
//表名、列名
const val TABLE_NAME = "table_cache"
const val COLUMN_NAME_KEY = "cache_key"
const val COLUMN_NAME_DATA = "cache_data"

//建表 SQL，是不是很容易犯错，如果使用Java就更容易犯错了
//即使你说这些我信手拈来，但是依然很繁琐
private const val SQL_CREATE_TABLE_CACHE =
        "CREATE TABLE $TABLE_NAME (" +
                "$ID INTEGER PRIMARY KEY," +
                "$COLUMN_NAME_TITLE TEXT," +
                "$COLUMN_NAME_SUBTITLE TEXT)"

class CacheDbHelper(context: Context) : SQLiteOpenHelper(context, DATABASE_NAME, null, DATABASE_VERSION) {
    companion object {
        const val DATABASE_VERSION = 1
        const val DATABASE_NAME = "cache.db"
    }
    
    override fun onCreate(db: SQLiteDatabase) {
        db.execSQL(SQL_CREATE_TABLE_CACHE)
    }
    
    override fun onUpgrade(db: SQLiteDatabase, oldVersion: Int, newVersion: Int) {
        //根据oldVersion和newVersion编写相应的SQL语句对数据库进行升级
        //在实际应用中，这是比较难处理的一部分，因为缺少必要的验证，极容易出错
    }
    
    //基类中包含了两个重要的方法 getWritableDatabase() 和 getReadableDatabase()
    //这是我们操作数据库的入口
}
```



- Room数据库创建
  - 添加依赖

```groovy
api "android.arch.persistence.room:runtime:2.2。0"
kapt "android.arch.persistence.room:compiler:2.2.0" 
```

-  创建Room数据库,必备三大件

   - @Entity：表示数据库中的表

   - @DAO：数据操作对象
   - @Database数据库：必须是扩展 RoomDatabase 的抽象类。在注解中添加与数据库关联的数据表。包含使用 @Dao 注解标记的的类的抽象方法。

```kotlin
// 1.定义 数据表
@Entity(tableName = "table_cache")
class Cache {
    @PrimaryKey(autoGenerate = false)
    @NonNull
    @ColumnInfo("cache_key")
    var key: String = ""

    @ColumnInfo("cache_data")
    var data: String? = null
}

// 2.定义数据库数据操作对象
@Dao(data access object)
interface CacheDao {

    @Insert(entity = Cache::class, onConflict = OnConflictStrategy.REPLACE)
    fun saveCache(cache: Cache): Long

    @Query("select * from cache where `key`=:key")
    fun getCache(key: String): Cache?

    @Delete(entity = Cache::class)
    fun deleteCache(cache: Cache)
}

// 3.定义数据库，并关联上表 和 数据操作实体
@Database(entities = [Cache::class], version = 1)
abstract class CacheDatabase : RoomDatabase() {
   val database =
                Room.databaseBuilder(context, CacheDatabase::class.java, "howow_cache").build() 
   abstract val cacheDao: CacheDao
}
```



### 数据库创建实现原理

- Room数据抽象设计

![room_arc](/imgs/jetpack/room_arc.png)

- 数据库创建流程

![room_database](/imgs/jetpack/room_database.png)

### Room与LiveData的巧妙结合,数据变更监听

- Room与LiveData的巧妙结合
  - 第一次向livedata注册observer时触发onActive,从而触发首次数据加载，并向InvalidationTracker注册表数据变更监听

![room_livedata](/imgs/jetpack/room_livedata.png)

- 数据变更监听
  - 增删改三种操作开始之前会向一张表中写入本次操作的表的名称,状态置为1，操作完成后会触发InvalidationTracker的endTranstions。进而调用 refreshRunnable查询出 所有数据变更了的表。然后回调给每一个RoomTracklingLiveData再次执行refreshRunnable 重新加载数据，并发送到UI层的observer刷新页面。

![room_livedata_trigger](/imgs/jetpack/room_livedata_trigger.png)
