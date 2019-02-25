1. 基本使用
    - SQLiteOpenHelper
    - SQLiteDatabase
2. ORM
    - greenDAO
    - Room
3. 进程与线程并发
    1. 多进程并发:
        - Shared读锁
        - Exclusive写锁
        - SQLite 默认是支持多进程并发操作的，它通过**文件锁**来控制多进程的并发。多进程可以同时获取 SHARED 锁来读取数据，但是只有一个进程可以获取 EXCLUSIVE 锁来写数据库。在 EXCLUSIVE 模式下，数据库连接在断开前都不会释放 SQLite 文件的锁，从而避免不必要的冲突，提高数据库访问的速度。
    2. 多线程并发
        - 默认开启 Multi-thread模式
        - Busy Retry方案解决写互斥
        - 跟多进程的锁机制一样，为了实现简单，SQLite 锁的粒度都是数据库文件级别，并没有实现表级甚至行级的锁。还有需要说明的是，同一个句柄同一时间只有一个线程在操作，这个时候我们需要打开连接池 Connection Pool
        - 跟多进程类似，多线程可以同时读取数据库数据，但是写数据依然是互斥的。SQLite 提供了 Busy Retry 的方案，即发生阻塞时会触发 Busy Handler，此时可以让线程休眠一段时间后，重新尝试操作
        - 为了进一步提高并发性能，我们还可以打开 WAL（Write-Ahead Logging）模式。WAL 模式会将修改的数据单独写到一个 WAL 文件中，同时也会引入 WAL 日志文件锁。通过 WAL 模式读和写可以完全的并发执行，不会相互阻塞。db.enableWriteAheadLogging();	//返回 false / true;如果开启了 WAL 模式，开启事务要使用 benginTransactionNonExclusive()，注意捕获异常，
        - 但是需要注意的是，**写之间仍然不能并发。**如果出现多个写并发的情况，依然有可能出现 SQLiteDatabaseLockedException，这个时候可以让应用捕获这个异常，然后等待一段时间后重试。
        - 总的来说，通过连接池与 WAL 模式，可以很大程度上增加 SQLite 的读写并发，大大减少由于并发导致的等待耗时，建议在应用中尝试开启。
     
    3. 文件锁

4. 常见问题
    1. 数据库升级之更改表名，增删改字段
        - SQLite 仅仅支持 ALTER TABLE 语句的一部分功能，可以用 ALTER TABLE 语句来更改一个表的名字，也可以向表中增加一个字段，但是我们不能删除一个已经存在的字段，或更改一个已经存在的字段的名称、数据类型、限定符等。
           - 修改表名以及增加字段都是相同的操作：
           - 修改字段：思路是先由原表创建一个临时表，然后创建一个新表把需要的信息从临时表取出即可。
           ```
                //1. 创建临时表
                ALTER TABLE "Student" RENAME TO "_Student_old_20140409";
                //2. 创建新表，把需要修改的字段或者删除的字段更新上去
                CREATE TABLE "Student" (
                "Id"  INTEGER PRIMARY KEY AUTOINCREMENT,
                "Name"  Text);
                //3. 从临时表拿所需数据
                INSERT INTO "Student" ("Id", "Name") SELECT "Id", "Title" FROM "_Student_old_20140409";
                //4. 删除临时表（可选）
           ```

    2. SQLiteDatabaseLockedException
5. 优化建议：
    1. 事物
        - 优点：加快数据库检索的速度,使用事务的两大好处是原子提交和更优性能
        - 缺点：索引的创建和维护存在消耗，会占用物理空间
        - 适用场景：读多写少
    2. 建立索引
        - 原子提交:原子提交意味着同一事务内的所有修改要么都完成要么都不做，如果某个修改失败，会自动回滚使得所有修改不生效。
        - 更优性能:SQLite 默认会为每个插入、更新操作创建一个事务，并且在每次插入、更新后立即提交。
    3. 页大小和缓存大小
        - pagesize指定4kb
        - 对于 SQLite 的 DB 文件来说，页是最小的存储单位。跟文件系统的页缓存一样，SQLite 会将读过的页缓存起来，用来加快下一次读取速度，页大小默认是 1024 Byte，缓存大小默认是 1000 页。如果使用 4KB 的 pageSize 性能能提升 5%～10% 左右，所以在建数据库的时候，就提前选择 4KB 作为默认的 page size 以获得更好的性能。
    4. 
        - sql语句拼接适用StringBuilder
        - getReadableDatabase，getWritableDatabase合理使用
        - 只查询所需的结果集
        - 少用Cursor.getColumnIndex
        - 异步线程

6. 数据库升级
- 当我们SQLiteOpenHelper创建对象db的时候如果传入的版本号大于之前的版本号，该方法就会被调用，通过判断oldVersion 和 newVersion 就可以决定如何升级数据库。在这个函数中把老版本数据库的相应表中增加字段，并给每条记录增加默认值即可。新版本号和老版本号都会作为onUpgrade函数的参数传进来，便于开发者知道数据库应该从哪个版本升级到哪个版本。升级完成后，数据库会自动存储最新的版本号为当前数据库版本号。
- SQLite提供了ALTER TABLE命令，允许用户重命名或添加新的字段到已有表中，但是不能从表中删除字段。并且只能在表的末尾添加字段
7. 数据库迁移
    1. 将表名改成临时表 ALTER TABLE Order RENAME TO _Order;
    2. 创建新表 CREATETABLE Test(Id VARCHAR(32) PRIMARY KEY ,CustomName VARCHAR(32) NOTNULL , Country VARCHAR(16) NOTNULL);
    3. 导入数据 INSERTINTO Order SELECT id, “”, Age FROM _Order;
    4. 删除临时表 DROPTABLE _Order;
