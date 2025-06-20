## 整体架构设计理念
Reth 存储层采用了经典的分层架构模式，通过 Rust 的 trait 系统实现了清晰的抽象层次。整个架构从上到下分为五个层次：应用层、抽象层、实现层、封装层和存储引擎层，每层都有明确的职责边界和接口定义

### 分层概览
1. **应用业务层**
    - 实际的区块链业务逻辑
2. **Provider实现层** 
    - `reth-provider`：主要的数据访问提供者，包含 `ProviderFactory` 和 `DatabaseProvider`
3. **存储业务抽象层**
    - `reth-storage-api`：定义业务级存储接口
4. **数据库抽象层**
    - `reth-db-api`：数据库统一接口（`Database`、`DbTx`、`DbTxMut`）
    - `reth-db-models`：数据模型定义
5. **MDBX实现层**
    - `reth-db`：MDBX的具体实现，包含 `DatabaseEnv`、`Tx`、`Cursor` 等
6. **底层封装层**
    - `reth-libmdbx`：对libmdbx C库的类型安全Rust封装
7. **存储引擎层**
    - **libmdbx C库**：底层LMDB存储引擎

![](https://cdn.nlark.com/yuque/__mermaid_v3/df2ad7d455c17dc4446beb6d54a82bbf.svg)

### 文件夹与业务分层对应关系
| **文件夹** | **层次** | **主要职责** | **核心组件** |
| --- | --- | --- | --- |
| `provider/` | 应用层 | 业务逻辑、数据访问 | `DatabaseProvider`, `ProviderFactory` |
| `db-common/` | 应用层 | 公共工具、初始化 | Database utilities, ETL, configurations |
| `db-api/` | 抽象层 | 接口定义 | `Database`, `DbTx`, `DbTxMut` traits |
| `db-models/` | 抽象层 | 数据模型 | 数据结构定义、编解码 |
| `db/` | 实现层 | MDBX适配实现 | `DatabaseEnv`, `Tx<K>`, `Cursor<K,T>` |
| `libmdbx-rs/` | 封装层 | 类型安全封装 | `Environment`, `Transaction`, `Cursor` |


这种设计的核心思想是将业务逻辑与具体的存储实现解耦，使得 Reth 可以在不修改上层代码的情况下轻松切换底层存储引擎。同时，通过 Rust 的类型系统和 trait 抽象，确保了编译时的类型安全和运行时的内存安全

## 核心抽象层设计（reth-db-api）
### Database Trait：统一数据库抽象
Database trait 是整个存储层的核心接口，使用了 Rust 稳定版的泛型关联类型 (GATs) 来实现强大的抽象能力。

```rust
pub trait Database: Send + Sync + Debug {
    /// 只读数据库事务类型
    type TX: DbTx + Send + Sync + Debug + 'static;
    /// 读写数据库事务类型
    type TXMut: DbTxMut + DbTx + TableImporter + Send + Sync + Debug + 'static;

    /// 创建只读事务
    #[track_caller]
    fn tx(&self) -> Result<Self::TX, DatabaseError>;

    /// 创建读写事务（仅在数据库以写访问模式打开时可用）
    #[track_caller]
    fn tx_mut(&self) -> Result<Self::TXMut, DatabaseError>;

    /// 便捷方法：传入函数并确保事务正确关闭
    fn view<T, F>(&self, f: F) -> Result<T, DatabaseError>
    where
        F: FnOnce(&Self::TX) -> T,
    {
        let tx = self.tx()?;
        let res = f(&tx);
        tx.commit()?;
        Ok(res)
    }

    /// 便捷方法：传入函数并确保事务正确提交
    fn update<T, F>(&self, f: F) -> Result<T, DatabaseError>
    where
        F: FnOnce(&Self::TXMut) -> T,
    {
        let tx = self.tx_mut()?;
        let res = f(&tx);
        tx.commit()?;
        Ok(res)
    }
}
```

这种设计允许不同的数据库实现提供自己的事务类型，同时保持统一的接口规范. 

<font style="color:#117CEE;">GATs 是 Rust 语言中的一项重要特性，允许为 trait 的关联类型添加泛型参数。这项特性在 Rust 1.65 版本中稳定化，为 reth-db 的灵活设计提供了基础。</font>

#### <font style="color:#117CEE;">普通关联类型</font>
<font style="color:#117CEE;">在 Rust 中，很多 trait 会定义关联类型，比如：</font>

```plain
trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
}

```

<font style="color:#117CEE;">这里的 </font>`**<font style="color:#117CEE;">Item</font>**`<font style="color:#117CEE;"> 是一个固定的类型，每个实现了 </font>`**<font style="color:#117CEE;">Iterator</font>**`<font style="color:#117CEE;"> 的类型都必须指定一个具体的 </font>`**<font style="color:#117CEE;">Item</font>**`<font style="color:#117CEE;"> 类型。</font>

#### **<font style="color:#117CEE;">泛型关联类型（GATs）</font>**
<font style="color:#117CEE;">GATs 的优势在于可以让关联类型包含生命周期参数，使得迭代器可以返回借用自身的数据，这在普通关联类型中是无法实现的。</font>

```plain
trait LendingIterator {
    type Item<'a> where Self: 'a;
    fn next<'a>(&'a mut self) -> Option<Self::Item<'a>>;
}

```

<font style="color:#117CEE;">这里的 </font>`**<font style="color:#117CEE;">Item<'a></font>**`<font style="color:#117CEE;"> 是一个可以带生命周期的关联类型，允许 </font>`**<font style="color:#117CEE;">next</font>**`<font style="color:#117CEE;"> 方法返回一个与 </font>`**<font style="color:#117CEE;">self</font>**`<font style="color:#117CEE;"> 生命周期</font>_<font style="color:#117CEE;">(标记引用的“存活时间”)</font>_<font style="color:#117CEE;">绑定的项。这在实现“借出迭代器”等模式时非常有用，因为你可以让迭代器返回一个借用自迭代器自身的项，而普通迭代器做不到这一点</font>

### 事务抽象：DbTx 和 DbTxMut
事务抽象是 Reth 存储层的另一个关键设计。`DbTx` trait 定义了只读事务的操作，而 `DbTxMut` trait 继承了 `DbTx` 并添加了写操作功能。

#### 只读事务
```rust
pub trait DbTx: Debug + Send + Sync {
    /// 只读事务的游标类型
    type Cursor<T: Table>: DbCursorRO<T> + Send + Sync;
    /// 只读事务的重复排序游标类型
    type DupCursor<T: DupSort>: DbDupCursorRO<T> + DbCursorRO<T> + Send + Sync;

    /// 通过键获取值
    fn get<T: Table>(&self, key: T::Key) -> Result<Option<T::Value>, DatabaseError>;
    
    /// 通过编码键的引用获取值（优化性能，避免克隆）
    fn get_by_encoded_key<T: Table>(
        &self,
        key: &<T::Key as Encode>::Encoded,
    ) -> Result<Option<T::Value>, DatabaseError>;
    
    /// 提交只读事务并释放内存页
    fn commit(self) -> Result<bool, DatabaseError>;
    
    /// 中断事务
    fn abort(self);
    
    /// 创建只读游标
    fn cursor_read<T: Table>(&self) -> Result<Self::Cursor<T>, DatabaseError>;
}
```

#### 读写事务
```rust
pub trait DbTxMut: Send + Sync {
    /// 读写游标类型
    type CursorMut<T: Table>: DbCursorRW<T> + DbCursorRO<T> + Send + Sync;

    /// 向数据库写入值
    fn put<T: Table>(&self, key: T::Key, value: T::Value) -> Result<(), DatabaseError>;
    
    /// 从数据库删除值
    fn delete<T: Table>(&self, key: T::Key, value: Option<T::Value>) 
        -> Result<bool, DatabaseError>;
    
    /// 清空表
    fn clear<T: Table>(&self) -> Result<(), DatabaseError>;
    
    /// 创建读写游标
    fn cursor_write<T: Table>(&self) -> Result<Self::CursorMut<T>, DatabaseError>;
}
```

这种继承关系确保了读写事务包含所有只读操作，同时通过类型系统在编译期防止在只读事务中执行写操作。

### 游标抽象：高效数据遍历
游标是数据库中用于遍历记录的重要概念，Reth 通过 `DbCursorRO` 和 `DbCursorRW` traits 提供了类型安全的游标抽象。只读游标支持查找、定位和遍历操作，而读写游标还支持插入、更新和删除操作。

```rust
pub trait DbCursorRO<T: Table> {
    /// 定位到表中第一个条目
    fn first(&mut self) -> PairResult<T>;

    /// 查找精确匹配的键值对
    fn seek_exact(&mut self, key: T::Key) -> PairResult<T>;

    /// 查找大于等于指定键的键值对
    fn seek(&mut self, key: T::Key) -> PairResult<T>;

    /// 移动到下一个键值对
    fn next(&mut self) -> PairResult<T>;

    /// 移动到上一个键值对
    fn prev(&mut self) -> PairResult<T>;

    /// 定位到表中最后一个条目
    fn last(&mut self) -> PairResult<T>;

    /// 获取游标当前位置的键值对
    fn current(&mut self) -> PairResult<T>;

    /// 创建遍历迭代器
    fn walk(&mut self, start_key: Option<T::Key>) -> Result<Walker<'_, T, Self>, DatabaseError>
    where
        Self: Sized;
}
```

![](https://cdn.nlark.com/yuque/0/2025/png/693539/1749399357343-d3f6ee5b-f7f1-4d0e-b8d0-a6ac48706164.png)

Illustration of a database fetch operation using a cursor

### Table Trait：编解码抽象
Table trait 定义了数据库表的结构，包括表名、键类型和值类型。每个表都必须实现此 trait，指定其键值对的具体类型以及相应的编解码方式。

```rust
/// 定义数据压缩相关操作，用于数据库存储前的数据处理
pub trait Compress: Send + Sync + Sized + Debug {
    /// 压缩后的数据类型，需要满足以下特性：
    /// - 支持缓冲区操作 (BufMut)
    /// - 支持字节切片转换 (AsRef/AsMut)
    /// - 可转换为Vec<u8>
    /// - 线程安全 (Send/Sync)
    type Compressed: bytes::BufMut
        + AsRef<[u8]>
        + AsMut<[u8]>
        + Into<Vec<u8>>
        + Default
        + Send
        + Sync
        + Debug;

    /// 获取未压缩数据的引用（适用于无需压缩的情况）
    /// 默认返回None，需要压缩的类型应重写此方法
    fn uncompressable_ref(&self) -> Option<&[u8]> {
        None
    }

    /// 执行数据压缩，返回压缩后的缓冲区
    /// 使用默认实现时内部会创建新缓冲区
    fn compress(self) -> Self::Compressed {
        let mut buf = Self::Compressed::default();
        self.compress_to_buf(&mut buf);
        buf
    }

    /// 核心压缩方法，将数据写入现有缓冲区
    /// 参数：
    /// - buf: 可变的缓冲区引用，需要实现BufMut和AsMut特性
    fn compress_to_buf<B: bytes::BufMut + AsMut<[u8]>>(&self, buf: &mut B);
}

/// 定义数据解压操作，用于从数据库读取数据的处理
pub trait Decompress: Send + Sync + Sized + Debug {
    /// 从字节切片解压数据
    /// 参数：
    /// - value: 包含压缩数据的字节切片
    /// 返回解压后的数据实例或错误
    fn decompress(value: &[u8]) -> Result<Self, DatabaseError>;

    /// 从Vec<u8>解压数据（便捷方法）
    /// 默认实现直接调用slice版本
    fn decompress_owned(value: Vec<u8>) -> Result<Self, DatabaseError> {
        Self::decompress(&value)
    }
}

/// 定义数据编码操作，用于数据库存储前的格式转换
pub trait Encode: Send + Sync + Sized + Debug {
    /// 编码后的数据类型，需要满足：
    /// - 字节切片转换 (AsRef)
    /// - 可转换为Vec<u8>
    /// - 支持排序和调试
    type Encoded: AsRef<[u8]> + Into<Vec<u8>> + Send + Sync + Ord + Debug;

    /// 将数据编码为数据库存储格式
    /// 消耗self所有权以支持高效编码
    fn encode(self) -> Self::Encoded;
}

/// 定义数据解码操作，用于从数据库读取数据的转换
pub trait Decode: Send + Sync + Sized + Debug {
    /// 从字节切片解码数据
    /// 参数：
    /// - value: 包含编码数据的字节切片
    /// 返回解码后的数据实例或错误
    fn decode(value: &[u8]) -> Result<Self, DatabaseError>;

    /// 从Vec<u8>解码数据（便捷方法）
    /// 默认实现直接调用slice版本
    fn decode_owned(value: Vec<u8>) -> Result<Self, DatabaseError> {
        Self::decode(&value)
    }
}

```

## MDBX 适配实现层（reth-db）
### DatabaseEnv：MDBX 环境封装
`DatabaseEnv` 是 `Database` trait 的 MDBX 具体实现，它封装了底层的 MDBX 环境并提供了类型安全的接口。该实现负责管理 MDBX 的环境配置、事务创建和表映射。

```rust
// 为数据库环境（DatabaseEnv）实现核心的Database trait
impl Database for DatabaseEnv {
    // 定义只读事务（TX）的具体类型为Tx<RO>
    type TX = tx::Tx<RO>;
    // 定义读写事务（TXMut）的具体类型为Tx<RW>
    type TXMut = tx::Tx<RW>;

    /// 创建一个只读事务
    fn tx(&self) -> Result<Self::TX, DatabaseError> {
        // 使用指标监控功能初始化事务封装器
        Tx::new_with_metrics(
            // 启动底层的MDBX只读事务，并在失败时将错误转换为DatabaseError
            self.inner.begin_ro_txn().map_err(|e| DatabaseError::InitTx(e.into()))?,
            // 传入指标收集器的克隆，用于监控此事务的性能
            self.metrics.clone(),
        )
        // 捕获并转换封装器初始化时可能发生的错误
        .map_err(|e| DatabaseError::InitTx(e.into()))
    }

    /// 创建一个读写事务
    fn tx_mut(&self) -> Result<Self::TXMut, DatabaseError> {
        // 使用指标监控功能初始化事务封装器
        Tx::new_with_metrics(
            // 启动底层的MDBX读写事务，并在失败时将错误转换为DatabaseError
            self.inner.begin_rw_txn().map_err(|e| DatabaseError::InitTx(e.into()))?,
            // 传入指标收集器的克隆，用于监控此事务的性能
            self.metrics.clone(),
        )
        // 捕获并转换封装器初始化时可能发生的错误
        .map_err(|e| DatabaseError::InitTx(e.into()))
    }
}

```

这种实现方式将 MDBX 的 C API 调用封装在安全的 Rust 接口之后，确保内存安全和错误处理。

### 事务实现：Tx<K>
`Tx<K>` 结构体是事务抽象的具体实现，其中 `K` 是一个类型参数，用于区分只读（RO）和读写（RW）事务。这种设计利用了 Rust 的类型系统在编译时就能确定事务的读写性质。

```rust
// K: TransactionKind 是一个泛型参数，代表事务的模式（如：RO-只读, RW-读写）。
impl<K: TransactionKind> Tx<K> {
    /// 创建一个新的事务封装器。
    ///
    /// # 参数
    /// * `inner`: 从 MDBX 库获取的底层事务对象。
    #[inline]
    pub const fn new(inner: Transaction<K>) -> Self {
        Self { inner }
    }

    /// 获取当前事务的唯一ID。
    ///
    /// 直接调用底层 MDBX 事务的 `id()` 方法。
    pub fn id(&self) -> reth_libmdbx::Result<u64> {
        self.inner.id()
    }

    /// 获取或创建一个表（在MDBX中称为DBI）的句柄。
    ///
    /// 表是数据库中用于组织数据的命名空间。此函数是执行任何表操作前必须调用的。
    ///
    /// # 参数
    /// * `T`: 一个实现了 `Table` trait 的类型，它定义了表的名称和编解码格式。
    pub fn get_dbi<T: Table>(&self) -> Result<MDBX_dbi, DatabaseError> {
        self.inner
            // 尝试打开名为 T::NAME 的表
            .open_db(Some(T::NAME))
            // 如果成功，获取其内部句柄 (DBI)
            .map(|db| db.dbi())
            // 将 MDBX 的错误类型转换为项目自定义的 DatabaseError
            .map_err(|e| DatabaseError::Open(e.into()))
    }

    /// 为指定的表创建一个新的游标（Cursor）。
    ///
    /// 游标是遍历表中键值对的主要方式。
    ///
    /// # 参数
    /// * `T`: 需要操作的表，必须实现 `Table` trait。
    pub fn new_cursor<T: Table>(&self) -> Result<Cursor<K, T>, DatabaseError> {
        // 1. 首先获取表的句柄 (DBI)
        let dbi = self.get_dbi::<T>()?;
        
        // 2. 使用该句柄在底层事务中创建 MDBX 游标
        let inner_cursor = self
            .inner
            .cursor_with_dbi(dbi)
            .map_err(|e| DatabaseError::InitCursor(e.into()))?;

        // 3. 将底层游标封装成项目自定义的 Cursor 类型并返回
        Ok(Cursor::new(inner_cursor))
    }

}

```

### 游标实现：Cursor<K,T>
游标的实现同样使用了泛型参数，`K` 表示事务类型，`T` 表示表类型。这种设计确保了游标只能操作与其关联的表，并且遵循事务的读写限制。

可以把 `**T**` 想象成一个具体的表，比如 `**Transactions**` 表，那么 `**T::Key**` 就是交易哈希，`**T::Value**` 就是交易内容。

```plain
// K 代表事务类型（只读/读写），T 代表操作的表（Table）
impl<K, T> Cursor<K, T> 
where 
    K: TransactionKind, // 事务类型
    T: Table,           // 表定义
{
    // --- 1. 定位操作 (Positioning) ---

    /// 将游标移动到表的第一个条目，并返回该条目。
    fn first(&mut self) -> Result<Option<(T::Key, T::Value)>>;

    /// 将游标移动到表的最后一个条目，并返回该条目。
    fn last(&mut self) -> Result<Option<(T::Key, T::Value)>>;

    /// 精确查找指定的 `key`。如果找到，游标会定位到该条目并返回它。
    /// 这是最常用的查找方式。
    fn seek_exact(&mut self, key: T::Key) -> Result<Option<(T::Key, T::Value)>>;

    /// 查找第一个大于或等于指定 `key` 的条目。
    /// 这对于范围查询非常有用。
    fn seek(&mut self, key: T::Key) -> Result<Option<(T::Key, T::Value)>>;


    // --- 2. 步进操作 (Stepping) ---

    /// 从当前位置移动到下一个条目，并返回它。
    fn next(&mut self) -> Result<Option<(T::Key, T::Value)>>;

    /// 从当前位置移动到上一个条目，并返回它。
    fn prev(&mut self) -> Result<Option<(T::Key, T::Value)>>;


    // --- 3. 数据读取 (Data Retrieval) ---

    /// 获取游标当前位置的键值对，游标位置不移动。
    fn current(&mut self) -> Result<Option<(T::Key, T::Value)>>;


    // --- 4. 迭代器 (Iteration) ---

    /// 创建一个“遍历器”(Walker)，从表的开头或指定 `start_key` 开始，
    /// 按顺序遍历所有条目。这是最强大和最常用的遍历方式。
    ///
    /// `Walker` 自身实现了 Rust 的 `Iterator` trait，所以你可以用 for 循环来处理。
    fn walk(&mut self, start_key: Option<T::Key>) -> Result<Walker<T>>;

    /// 创建一个“范围遍历器”，只遍历指定范围内的数据。
    fn walk_range(&mut self, range: impl RangeBounds<T::Key>) -> Result<RangeWalker<T>>;
}

```

典型的 `**Cursor**` 使用流程:

**获取事务**: 首先，你需要从数据库环境 `**db**` 中获取一个事务 `**tx**`。

```plain
let tx = db.tx()?; 
```

**创建游标**: 然后，通过事务为某个具体的表（例如 `**Headers**` 表）创建一个游标。

```plain
let mut cursor = tx.cursor_read::<tables::Headers>()?;
```

**遍历数据**: 现在你可以使用游标的迭代器功能来遍历数据。例如，使用 `**walk**` 创建一个遍历器，然后用 `**for**` 循环处理每一个键值对。

```plain
// 从头开始遍历 Headers 表
for result in cursor.walk(None)? {
    let (block_number, header) = result?;
    println!("Block {}: {:?}", block_number, header);
}
```

**精确查找**: 或者，你可以用 `**seek_exact**` 来查找一个特定的块头。

```plain
let block_number: u64 = 1_000_000;
if let Some((num, header)) = cursor.seek_exact(block_number)? {
    println!("Found header for block {}: {:?}", num, header);
}
```



## 底层封装层（reth-libmdbx）
reth-libmdbx 是对 MDBX C 库的 Rust 封装，提供了类型安全和内存安全的接口。该层的主要职责是将 C 库的原始指针和函数调用转换为 Rust 的类型安全接口。

MDBX 作为一个高性能的键值存储引擎，支持 ACID 事务和多版本并发控制。Reth 选择 MDBX 而不是其他数据库引擎，主要是因为其优秀的性能特性和内存映射机制。

crates/storage/libmdbx-rs

```plain
// 关键API
impl Environment {
    pub fn builder() -> EnvironmentBuilder
    pub fn begin_ro_txn(&self) -> Result<Transaction<RO>>
    pub fn begin_rw_txn(&self) -> Result<Transaction<RW>>
    pub fn stat(&self) -> Result<Stat>
    pub fn info(&self) -> Result<Info>
}
// Transaction（事务）
impl<K: TransactionKind> Transaction<K> {
    pub fn get<Key>(&self, dbi: MDBX_dbi, key: &[u8]) -> Result<Option<Key>>
    pub fn cursor(&self, db: &Database) -> Result<Cursor<K>>
    pub fn commit(self) -> Result<(bool, CommitLatency)>
}

impl Transaction<RW> {
    pub fn put(&self, dbi: MDBX_dbi, key: impl AsRef<[u8]>, data: impl AsRef<[u8]>, flags: WriteFlags) -> Result<()>
    pub fn del(&self, dbi: MDBX_dbi, key: impl AsRef<[u8]>, data: Option<&[u8]>) -> Result<bool>
}

//Cursor 
impl<K: TransactionKind> Cursor<K> {
    pub fn first<Key, Value>(&mut self) -> Result<Option<(Key, Value)>>
    pub fn next<Key, Value>(&mut self) -> Result<Option<(Key, Value)>>
    pub fn set<Value>(&mut self, key: &[u8]) -> Result<Option<Value>>
    pub fn iter<Key, Value>(&mut self) -> Iter<'_, K, Key, Value>
}
```

## 应用层实现（reth-provider）
### DatabaseProvider：业务逻辑封装
`DatabaseProvider` 是应用层的核心组件，它封装了具体的业务逻辑，如账户查询、存储访问等以太坊相关操作。该组件使用底层的数据库抽象接口，不直接依赖 MDBX 的具体实现。

可以看到代码中已经在实现区块链相关的业务了

```plain
impl<TX: DbTx + 'static, N: NodeTypes> DatabaseProvider<TX, N> {
    /// State provider for latest state
    pub fn latest<'a>(&'a self) -> Box<dyn StateProvider + 'a> {
        trace!(target: "providers::db", "Returning latest state provider");
        Box::new(LatestStateProviderRef::new(self))
    }

    /// Storage provider for state at that given block hash
    pub fn history_by_block_hash<'a>(
        &'a self,
        block_hash: BlockHash,
    ) -> ProviderResult<Box<dyn StateProvider + 'a>> {
        let mut block_number =
            self.block_number(block_hash)?.ok_or(ProviderError::BlockHashNotFound(block_hash))?;
        if block_number == self.best_block_number().unwrap_or_default() &&
            block_number == self.last_block_number().unwrap_or_default()
        {
            return Ok(Box::new(LatestStateProviderRef::new(self)))
        }

        // +1 as the changeset that we want is the one that was applied after this block.
        block_number += 1;

        let account_history_prune_checkpoint =
            self.get_prune_checkpoint(PruneSegment::AccountHistory)?;
        let storage_history_prune_checkpoint =
            self.get_prune_checkpoint(PruneSegment::StorageHistory)?;

        let mut state_provider = HistoricalStateProviderRef::new(self, block_number);

        // If we pruned account or storage history, we can't return state on every historical block.
        // Instead, we should cap it at the latest prune checkpoint for corresponding prune segment.
        if let Some(prune_checkpoint_block_number) =
            account_history_prune_checkpoint.and_then(|checkpoint| checkpoint.block_number)
        {
            state_provider = state_provider.with_lowest_available_account_history_block_number(
                prune_checkpoint_block_number + 1,
            );
        }
        if let Some(prune_checkpoint_block_number) =
            storage_history_prune_checkpoint.and_then(|checkpoint| checkpoint.block_number)
        {
            state_provider = state_provider.with_lowest_available_storage_history_block_number(
                prune_checkpoint_block_number + 1,
            );
        }

        Ok(Box::new(state_provider))
    }

    #[cfg(feature = "test-utils")]
    /// Sets the prune modes for provider.
    pub fn set_prune_modes(&mut self, prune_modes: PruneModes) {
        self.prune_modes = prune_modes;
    }
}

```

### ProviderFactory：连接管理
它作为数据库访问的工厂类，负责创建和管理数据库相关的提供者（Provider）

```plain

pub trait DatabaseProviderFactory: Send + Sync {
    /// Database this factory produces providers for.
    type DB: Database;

    /// Provider type returned by the factory.
    type Provider: DBProvider<Tx = <Self::DB as Database>::TX>;

    /// Read-write provider type returned by the factory.
    type ProviderRW: DBProvider<Tx = <Self::DB as Database>::TXMut>;

    /// Create new read-only database provider.
    fn database_provider_ro(&self) -> ProviderResult<Self::Provider>;

    /// Create new read-write database provider.
    fn database_provider_rw(&self) -> ProviderResult<Self::ProviderRW>;
}
----------------------------------------------------------------------------
pub fn provider(&self) -> ProviderResult<DatabaseProviderRO<N::DB, N>> {
    Ok(DatabaseProvider::new(
        self.db.tx()?,  // 创建只读事务
        self.chain_spec.clone(),
        self.static_file_provider.clone(),
        self.prune_modes.clone(),
        self.storage.clone(),
    ))
}

pub fn provider_rw(&self) -> ProviderResult<DatabaseProviderRW<N::DB, N>> {
    Ok(DatabaseProviderRW(DatabaseProvider::new_rw(
        self.db.tx_mut()?,  // 创建读写事务
        self.chain_spec.clone(),
        self.static_file_provider.clone(),
        self.prune_modes.clone(),
        self.storage.clone(),
    )))
}

```

ProviderFactory 将数据库和静态文件结合

```plain
fn transaction_by_id(&self, id: TxNumber) -> ProviderResult<Option<Self::Transaction>> {
    self.static_file_provider.get_with_static_file_or_database(
        StaticFileSegment::Transactions,
        id,
        |static_file| static_file.transaction_by_id(id),  // 优先从静态文件读取
        || self.provider()?.transaction_by_id(id),        // 回退到数据库
    )
}
```

## 编解码系统设计
Reth 实现了灵活的编解码系统，通过 `Encode`、`Decode`、`Compress` 和 `Decompress` traits 支持多种序列化格式。这种设计允许根据不同的使用场景选择最优的编码方式，在存储空间和性能之间取得平衡。

系统支持多种编码格式，包括 Ethereum 特定的紧凑编码、Scale 编码、Postcard 编码等。通过派生宏 `reth_codec`，可以轻松为自定义类型实现编解码功能。

变长整数编码

```plain
fn encode_varuint<B>(mut n: usize, buf: &mut B);
fn decode_varuint(buf: &[u8]) -> (usize, &[u8]);
```

区块头编解码 - header.rs

```plain
#[derive(Compact)]
pub struct Header {
    pub parent_hash: B256,
    pub ommers_hash: B256,
    pub beneficiary: Address,
    pub state_root: B256,
    // ... 其他字段
}
```

交易编解码 - transaction

**主要交易类型**：

+ TxLegacy - 传统交易
+ TxEip1559 - EIP-1559费用市场交易
+ TxEip2930 - EIP-2930访问列表交易
+ TxEip4844 - EIP-4844 blob交易
+ TxEip7702 - EIP-7702授权交易

```plain
impl<Eip4844> Compact for EthereumTypedTransaction<Eip4844> {
    fn to_compact<B>(&self, buf: &mut B) -> usize {
        let identifier = self.tx_type().to_compact(buf);
        match self {
            Self::Legacy(tx) => tx.to_compact(buf),
            Self::Eip1559(tx) => tx.to_compact(buf),
            // ... 其他类型
        }
    }
}
```

## 表结构设计
Reth 定义了完整的以太坊数据表结构，包括区块头、交易、收据、账户状态等。每个表都有明确的键值类型定义和用途说明。

主要表包括：

+ `PlainAccountState`：存储账户的当前状态
+ `PlainStorageState`：存储账户的存储值
+ `Transactions`：存储规范交易数据
+ `Receipts`：存储交易收据
+ `Headers`：存储区块头信息

```plain
    table CanonicalHeaders {
        type Key = BlockNumber;
        type Value = HeaderHash;
    }

    /// Stores the total difficulty from a block header.
    table HeaderTerminalDifficulties {
        type Key = BlockNumber;
        type Value = CompactU256;
    }

    /// Stores the block number corresponding to a header.
    table HeaderNumbers {
        type Key = BlockHash;
        type Value = BlockNumber;
    }

    /// Stores header bodies.
    table Headers<H = Header> {
        type Key = BlockNumber;
        type Value = H;
    }

    /// Stores block indices that contains indexes of transaction and the count of them.
    ///
    /// More information about stored indices can be found in the [`StoredBlockBodyIndices`] struct.
    table BlockBodyIndices {
        type Key = BlockNumber;
        type Value = StoredBlockBodyIndices;
    }

    /// Stores the uncles/ommers of the block.
    table BlockOmmers<H = Header> {
        type Key = BlockNumber;
        type Value = StoredBlockOmmers<H>;
    }

    /// Stores the block withdrawals.
    table BlockWithdrawals {
        type Key = BlockNumber;
        type Value = StoredBlockWithdrawals;
    }

    /// Canonical only Stores the transaction body for canonical transactions.
    table Transactions<T = TransactionSigned> {
        type Key = TxNumber;
        type Value = T;
    }

    /// Stores the mapping of the transaction hash to the transaction number.
    table TransactionHashNumbers {
        type Key = TxHash;
        type Value = TxNumber;
    }

    /// Stores the mapping of transaction number to the blocks number.
    ///
    /// The key is the highest transaction ID in the block.
    table TransactionBlocks {
        type Key = TxNumber;
        type Value = BlockNumber;
    }

    /// Canonical only Stores transaction receipts.
    table Receipts<R = Receipt> {
        type Key = TxNumber;
        type Value = R;
    }

    /// Stores all smart contract bytecodes.
    /// There will be multiple accounts that have same bytecode
    /// So we would need to introduce reference counter.
    /// This will be small optimization on state.
    table Bytecodes {
        type Key = B256;
        type Value = Bytecode;
    }

    /// Stores the current state of an [`Account`].
    table PlainAccountState {
        type Key = Address;
        type Value = Account;
    }

    /// Stores the current value of a storage key.
    table PlainStorageState {
        type Key = Address;
        type Value = StorageEntry;
        type SubKey = B256;
    }

    /// Stores pointers to block changeset with changes for each account key.
    ///
    /// Last shard key of the storage will contain `u64::MAX` `BlockNumber`,
    /// this would allows us small optimization on db access when change is in plain state.
    ///
    /// Imagine having shards as:
    /// * `Address | 100`
    /// * `Address | u64::MAX`
    ///
    /// What we need to find is number that is one greater than N. Db `seek` function allows us to fetch
    /// the shard that equal or more than asked. For example:
    /// * For N=50 we would get first shard.
    /// * for N=150 we would get second shard.
    /// * If max block number is 200 and we ask for N=250 we would fetch last shard and know that needed entry is in `AccountPlainState`.
    /// * If there were no shard we would get `None` entry or entry of different storage key.
    ///
    /// Code example can be found in `reth_provider::HistoricalStateProviderRef`
    table AccountsHistory {
        type Key = ShardedKey<Address>;
        type Value = BlockNumberList;
    }

    /// Stores pointers to block number changeset with changes for each storage key.
    ///
    /// Last shard key of the storage will contain `u64::MAX` `BlockNumber`,
    /// this would allows us small optimization on db access when change is in plain state.
    ///
    /// Imagine having shards as:
    /// * `Address | StorageKey | 100`
    /// * `Address | StorageKey | u64::MAX`
    ///
    /// What we need to find is number that is one greater than N. Db `seek` function allows us to fetch
    /// the shard that equal or more than asked. For example:
    /// * For N=50 we would get first shard.
    /// * for N=150 we would get second shard.
    /// * If max block number is 200 and we ask for N=250 we would fetch last shard and know that needed entry is in `StoragePlainState`.
    /// * If there were no shard we would get `None` entry or entry of different storage key.
    ///
    /// Code example can be found in `reth_provider::HistoricalStateProviderRef`
    table StoragesHistory {
        type Key = StorageShardedKey;
        type Value = BlockNumberList;
    }

    /// Stores the state of an account before a certain transaction changed it.
    /// Change on state can be: account is created, selfdestructed, touched while empty
    /// or changed balance,nonce.
    table AccountChangeSets {
        type Key = BlockNumber;
        type Value = AccountBeforeTx;
        type SubKey = Address;
    }

    /// Stores the state of a storage key before a certain transaction changed it.
    /// If [`StorageEntry::value`] is zero, this means storage was not existing
    /// and needs to be removed.
    table StorageChangeSets {
        type Key = BlockNumberAddress;
        type Value = StorageEntry;
        type SubKey = B256;
    }

    /// Stores the current state of an [`Account`] indexed with `keccak256Address`
    /// This table is in preparation for merklization and calculation of state root.
    /// We are saving whole account data as it is needed for partial update when
    /// part of storage is changed. Benefit for merklization is that hashed addresses are sorted.
    table HashedAccounts {
        type Key = B256;
        type Value = Account;
    }

    /// Stores the current storage values indexed with `keccak256Address` and
    /// hash of storage key `keccak256key`.
    /// This table is in preparation for merklization and calculation of state root.
    /// Benefit for merklization is that hashed addresses/keys are sorted.
    table HashedStorages {
        type Key = B256;
        type Value = StorageEntry;
        type SubKey = B256;
    }
```

这种表设计支持高效的历史状态查询，通过块号索引可以快速访问任意时刻的状态信息。

