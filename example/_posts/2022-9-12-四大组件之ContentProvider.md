---
layout: post
description: > 
  本文介绍了四大组件之ContentProvider的相关内容。
image: 
  path: /assets/img/blog/blogs_content_provider_cover.png
  srcset: 
    1920w: /assets/img/blog/blogs_content_provider_cover.png
    960w:  /assets/img/blog/blogs_content_provider_cover.png
    480w:  /assets/img/blog/blogs_content_provider_cover.png
accent_image: /assets/img/blog/blogs_content_provider_cover.png
excerpt_separator: <!--more-->
sitemap: false
---
# 四大组件之ContentProvider

本文中源码来自郭神的《第一行代码》（第三版）。

## 概念
ContentProvider，即内容提供者，主要用于在不同应用之间共享数据。它提供了一种标准的方式来访问和操作应用程序中的数据。ContentProvider可以被多个应用程序共享，并且可以在不同的应用程序之间进行数据的共享。

## ContentProvider的创建与使用

ContentP rovider 的用法一般有两种：

* 一种是使用现有的ContentProvider 读取和操作相应程序中的数据；

* 另一种是创建自己的ContentProvider ，给程序的数据提供外部访问接口。

### 使用现有的ContentP rovider

对于每一个应用程序来说，如果想要访问ContentProvider 中共享的数据，就一定要借助 ```ContentResolver``` 类，可以通过 ```Context``` 中的 ```getContentResolver()``` 方法获取该类的实例。 ```ContentResolver``` 中提供了一系列的方法用于对数据进行增删改查操作，其中insert()方法用于添加数据，update()方法用于更新数据，delete()方法用于删除数据，query()方法用于查询数据。

ContentResolver 中的增删改查方法都是不接收表名参数的，而是使用一个Uri参数代替，这个参数被称为内容URI。内容URI给ContentProvider 中的数据建立了唯一标识符，它主要由两部分组成：authority 和path 。authority 是用于对不同的应用程序做区分的，一般为了避免冲突，会采用应用包名的方式进行命名。path 则是用于对同一应用程序中不同的表做区分的，通常会添
加到authority 的后面。

内容URI最标准的格式如下：

```
content://com.example.app.provider/table1
content://com.example.app.provider/table2
```

在得到了内容URI字符串之后，我们还需要将它解析成Uri对象才可以作为参数传入。解析的方法也相当简单，代码如下所示：

```kotlin
val uri = Uri.parse("content://com.example.app.provider/table1")
```

只需要调用Uri.parse()方法，就可以将内容URI字符串解析成Uri对象了。

拿到uri对象之后，就可以调用ContentResolver中的增删改查方法了。

使用cursor对象读取数据时，需要使用moveToFirst()方法将游标移动到第一行数据的位置，然后使用getColumnIndex()方法获取每一列数据的列索引，最后使用getString()方法获取指定列索引对应的数据。

代码如下：
```kotlin
/** uri from table_name 指定查询某个应用程序下的某一张表
projection select column1, column2 指定查询的列名
selection where column = value 指定where的约束条件
selectionArgs - 为where中的占位符提供具体的值
sortOrder order by column1, column2 指定查询结果的排序方式
*/
val cursor = contentResolver.query(
 uri,
 projection,
 selection,
 selectionArgs,
 sortOrder) 

while (cursor.moveToNext()) {
 val column1 = cursor.getString(cursor.getColumnIndex("column1"))
 val column2 = cursor.getInt(cursor.getColumnIndex("column2"))
}
cursor.close() 

// 更新数据
val values = ContentValues()
values.put("column1", "value1")
values.put("column2", 1)
val rows = contentResolver.update(
 uri,
 values,
 selection, 
)

// 删除数据
val rows = contentResolver.delete(
 uri,
 selection,
 selectionArgs 
)
// 插入数据
val values = ContentValues()
values.put("column1", "value1")
values.put("column2", 1)
val uri = contentResolver.insert(
 uri,
 values
)

```

以读取系统联系人为例，代码如下：

```kotlin
private fun readContacts() {
    // 查询联系人数据
    contentResolver.query(ContactsContract.CommonDataKinds.Phone.CONTENT_URI,
        null, null, null, null)?.apply {
        while (moveToNext()) {
            // 获取联系人姓名
            val displayName = getString(getColumnIndex(
                ContactsContract.CommonDataKinds.Phone.DISPLAY_NAME))
            // 获取联系人手机号
            val number = getString(getColumnIndex(
                ContactsContract.CommonDataKinds.Phone.NUMBER))
            contactsList.add("$displayName\n$number")
        }
        adapter.notifyDataSetChanged()
        close()
    }
} 
```

注意需要提前检查和申请权限：

```xml
<uses-permission android:name="android.permission.READ_CONTACTS" />
```

运行时权限申请：

```kotlin
public class MainActivity : AppCompatActivity() {
  override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    setContentView(R.layout.activity_main) 
      if (ContextCompat.checkSelfPermission(this, Manifest.permission.READ_CONTACTS)
            != PackageManager.PERMISSION_GRANTED) {
            ActivityCompat.requestPermissions(this,
                arrayOf(Manifest.permission.READ_CONTACTS), 1)
        } else {
            readContacts()
        }
  }
}
```

### 创建自己的ContentProvider
首先在包名文件夹上右键New->Other->Content Provider。设置好ContentProvider的名称和authority，然后点击Finish。就可以在Manifest里看到对应的类和authority了。

然后在ContentProvider类中重写onCreate()、query()、insert()、update()、delete()方法。

```xml
<provider
    android:name=".MyContentProvider"
    android:authorities="com.example.app.provider"
    android:exported="true" />
```

Kotlin代码实现类如下：

```kotlin

class DatabaseProvider : ContentProvider() {
    private val bookDir = 0
    private val bookItem = 1
    private val categoryDir = 2
    private val categoryItem = 3
    private val authority = "com.example.databasetest.provider"
    private var dbHelper: MyDatabaseHelper? = null
    private val uriMatcher by lazy {
        val matcher = UriMatcher(UriMatcher.NO_MATCH)
        matcher.addURI(authority, "book", bookDir)
        matcher.addURI(authority, "book/#", bookItem)
        matcher.addURI(authority, "category", categoryDir)
        matcher.addURI(authority, "category/#", categoryItem)
        matcher
    }

    override fun onCreate() = context?.let {
        dbHelper = MyDatabaseHelper(it, "BookStore.db", 2)
        true
    } ?: false

    override fun query(
        uri: Uri, projection: Array<String>?, selection: String?,
        selectionArgs: Array<String>?, sortOrder: String?
    ) = dbHelper?.let {
        // 查询数据
        val db = it.readableDatabase
        val cursor = when (uriMatcher.match(uri)) {
            bookDir -> db.query(
                "Book", projection, selection, selectionArgs,
                null, null, sortOrder
            )

            bookItem -> {
                val bookId = uri.pathSegments[1]
                db.query(
                    "Book", projection, "id = ?", arrayOf(bookId), null, null,
                    sortOrder
                )
            }

            categoryDir -> db.query(
                "Category", projection, selection, selectionArgs,
                null, null, sortOrder
            )

            categoryItem -> {
                val categoryId = uri.pathSegments[1]
                db.query(
                    "Category", projection, "id = ?", arrayOf(categoryId),
                    null, null, sortOrder
                )
            }

            else -> null
        }
        cursor
    }

    override fun insert(uri: Uri, values: ContentValues?) = dbHelper?.let {
        // 添加数据
        val db = it.writableDatabase
        val uriReturn = when (uriMatcher.match(uri)) {
            bookDir, bookItem -> {
                val newBookId = db.insert("Book", null, values)
                Uri.parse("content://$authority/book/$newBookId")
            }

            categoryDir, categoryItem -> {
                val newCategoryId = db.insert("Category", null, values)
                Uri.parse("content://$authority/category/$newCategoryId")
            }

            else -> null
        }
        uriReturn
    }

    override fun update(
        uri: Uri, values: ContentValues?, selection: String?,
        selectionArgs: Array<String>?
    ) = dbHelper?.let {
        // 更新数据
        val db = it.writableDatabase
        val updatedRows = when (uriMatcher.match(uri)) {
            bookDir -> db.update("Book", values, selection, selectionArgs)
            bookItem -> {
                val bookId = uri.pathSegments[1]
                db.update("Book", values, "id = ?", arrayOf(bookId))
            }

            categoryDir -> db.update("Category", values, selection, selectionArgs)
            categoryItem -> {
                val categoryId = uri.pathSegments[1]
                db.update("Category", values, "id = ?", arrayOf(categoryId))
            }

            else -> 0
        }
        updatedRows
    } ?: 0

    override fun delete(uri: Uri, selection: String?, selectionArgs: Array<String>?) =
        dbHelper?.let {
            // 删除数据
            val db = it.writableDatabase
            val deletedRows = when (uriMatcher.match(uri)) {
                bookDir -> db.delete("Book", selection, selectionArgs)
                bookItem -> {
                    val bookId = uri.pathSegments[1]
                    db.delete("Book", "id = ?", arrayOf(bookId))
                }

                categoryDir -> db.delete("Category", selection, selectionArgs)
                categoryItem -> {
                    val categoryId = uri.pathSegments[1]
                    db.delete("Category", "id = ?", arrayOf(categoryId))
                }

                else -> 0
            }
            deletedRows
        } ?: 0

    override fun getType(uri: Uri) = when (uriMatcher.match(uri)) {
        bookDir -> "vnd.android.cursor.dir/vnd.com.example.databasetest.provider.book"
        bookItem -> "vnd.android.cursor.item/vnd.com.example.databasetest.provider.book"
        categoryDir -> "vnd.android.cursor.dir/vnd.com.example.databasetest.provider.category"
            categoryItem -> "vnd.android.cursor.item/vnd.com.example.databasetest.provider.category"
        else -> null
    }
} 
```

首先，在类的一开始，同样是定义了4个变量，分别用于表示访问Book 表中的所有数据、访问Book 表中的单条数据、访问Category 表中的所有数据和访问Category 表中的单条数据。然后在一个by lazy代码块里对UriMatcher进行了初始化操作，将期望匹配的几种URI格式添加了进去。by lazy代码块是Kotlin 提供的一种懒加载技术，代码块中的代码一开始并不会执行，只有当uriMatcher变量首次被调用的时候才会执行，并且会将代码块中最后一行代码的返回值赋给uriMatcher。

使用方的调用：

```kotlin
class MainActivity : AppCompatActivity() {
    var bookId: String? = null
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        addData.setOnClickListener {
            // 添加数据
            val uri = Uri.parse("content://com.example.databasetest.provider/book")
            val values = contentValuesOf("name" to "A Clash of Kings",
                "author" to "George Martin", "pages" to 1040, "price" to 22.85)
            val newUri = contentResolver.insert(uri, values)
            bookId = newUri?.pathSegments?.get(1)
        }
        queryData.setOnClickListener {
            // 查询数据
            val uri = Uri.parse("content://com.example.databasetest.provider/book")
            contentResolver.query(uri, null, null, null, null)?.apply {
                while (moveToNext()) {
                    val name = getString(getColumnIndex("name"))
                    val author = getString(getColumnIndex("author"))
                    val pages = getInt(getColumnIndex("pages"))
                    val price = getDouble(getColumnIndex("price"))
                    Log.d("MainActivity", "book name is $name")
                    Log.d("MainActivity", "book author is $author")
                    Log.d("MainActivity", "book pages is $pages")
                    Log.d("MainActivity", "book price is $price")
                }
                close()
            }
        }
        updateData.setOnClickListener {
            // 更新数据
            bookId?.let {
                val uri = Uri.parse("content://com.example.databasetest.provider/
                        book/$it")
                val values = contentValuesOf("name" to "A Storm of Swords",
                    "pages" to 1216, "price" to 24.05)
                contentResolver.update(uri, values, null, null)
            }
        }
        deleteData.setOnClickListener {
            // 删除数据
            bookId?.let {
                val uri = Uri.parse("content://com.example.databasetest.provider/
                        book/$it")
                contentResolver.delete(uri, null, null)
            }
        }
    }
}
```

添加数据的时候，首先调用了Uri.parse()方法将一个内容URI解析成Uri对象，然后把要添加的数据都存放到ContentValues对象中，接着调用ContentResolver的insert()方法执行添加操作就可以了。注意，insert()方法会返回一个Uri对象，这个对象中包含了新增数据的id，我们通过getPathSegments()方法将这个id取出，稍后会用到它。

查询数据的时候，同样是调用了Uri.parse()方法将一个内容URI解析成Uri对象，然后调用ContentResolver的query()方法查询数据，查询的结果当然还是存放在Cursor对象中。之后对Cursor进行遍历，从中取出查询结果，并一一打印出来。

更新数据的时候，也是先将内容URI解析成Uri对象，然后把想要更新的数据存放到ContentValues对象中，再调用ContentResolver的update()方法执行更新操作就可以了。注意，这里我们为了不想让Book 表中的其他行受到影响，在调用Uri.parse()方法时，给内容URI的尾部增加了一个id，而这个id正是添加数据时所返回的。这就表示我们只希望更新刚刚添加的那条数据，Book 表中的其他行都不会受影响。

删除数据的时候，也是使用同样的方法解析了一个以id结尾的内容URI，然后调用
ContentResolver的delete()方法执行删除操作就可以了。由于我们在内容URI里指定了一个id，因此只会删掉拥有相应id的那行数据，Book 表中的其他数据都不会受影响。