title:  保存数据
---

以下四种方法可用于将数据写入野狗云端：

方法 |  说明 
----|------
setValue() | 将数据写入到指定的路径，如果指定路径已存在数据，那么数据将会被覆盖。 
push() | 在当前路径下新增一个子节点, 并返回子节点的引用。这个子节点的 key 是利用服务端的当前时间生成的随机字符串, 与 setValue() 配合使用，用于将数据新增到此路径下。
updateChildren() | 对子节点进行合并操作。不存在的子节点将会被新增，存在的子节点将会被替换。
runTransaction() | 更新可能因并发更新而损坏的复杂数据。 

**特别注意** ：写入数据及其他操作，都要先确保与 Wilddog 云端长连接建立成功。如果你要复制本文档中的代码示例体验，最简单的方式就是加个 Thread.sleep() 线程等待，以确保长连接的成功建立，请自行在每个代码片段下面加上：
```java
//线程等待，以确保长连接的成功建立
try {
  Thread.sleep(6 * 1000);
} catch (InterruptedException e) {
  e.printStackTrace();
}
```

## 写入数据

`setValue()` 是最基本的写数据操作，它会将数据写入当前引用指向的节点。该节点如果已有数据，任何原有数据都将被删除和覆盖，包括其子节点的数据。
`setValue()` 可以传入几种数据类型 `string`, `number`, `boolean`, `null`, `map`或满足 JavaBean 规范的实体做为参数。
为了更好地理解该方法，我们建立一个[示例应用](https://samplechat.wilddogio.com)举例说明。我们打算将定义一个 User 对象保存在下面引用对应的路径中：

```java
Wilddog ref = new Wilddog("https://samplechat.wilddogio.com/android/saving-data/wildblog");
```
我们添加一些用户，为每个用户保存唯一的用户名，同时保存全名和出生日期。由于每个用户的用户名都是独一无二的，所以最好使用  `setValue()`方法，而不是 `push()` 方法，因为我们已经有了独一无二的用户名作为 key 值，不需要在添加的时候重新生成唯一标识。

首先，我们编写 User 类代码，将 User 对象以用户名作为 key 值添加到 Map 中。然后，为用户数据所在路径创建引用，调用 `setValue()` 方法将 Map 中的每个用户添加到数据库中。

```java
public class User {
    private int birthYear;
    private String fullName;

    public User() {}

    public User(String fullName, int birthYear) {
        this.fullName = fullName;
        this.birthYear = birthYear;
    }

    public long getBirthYear() {
        return birthYear;
    }

    public String getFullName() {
        return fullName;
    }
}

User alanisawesome = new User("Alan Turing", 1912);
User gracehop = new User("Grace Hopper", 1906);

Map<String, User> users = new HashMap<String, User>();
users.put("alanisawesome", alanisawesome);
users.put("gracehop", gracehop);

Wilddog usersRef = ref.child("users");

usersRef.setValue(users);
```
我们可以向setValue()方法传入自定义的Java对象作为参数，但需要**满足**如下条件：

- 对象所属的类中存在默认的构造方法;
- 类中所有需要写入数据库的属性都有getter方法。

我们使用 Map 将数据保存到数据库中，因为 Map 中的元素会自动映射成为 JSON 对象，并保存到指定路径。现在，我们再次访问示例应用的[数据预览](https://samplechat.wilddogio.com/android/saving-data/wildblog/users)来预览数据，就可以看到上面示例代码中保存的数据。

还有一种等价的操作方式，即直接保存数据到指定的路径，如下：
```java
Wilddog usersRef = new Wilddog("https://samplechat.wilddogio.com/android/saving-data/wildblog/users");
// 使用父节点引用的child()方法，获得指向子数据节点的引用。
usersRef.child("alanisawesome").child("fullName").setValue("Alan Turing");
usersRef.child("alanisawesome").child("birthYear").setValue(1912);
// 也可以在child()方法的参数中使用 '/' 分隔多个子节点路径。
usersRef.child("gracehop/name").setValue("Grace Hopper");
usersRef.child("gracehop/birthYear").setValue(1906);

try {
  Thread.sleep(6 * 1000);
} catch (InterruptedException e) {
  e.printStackTrace();
}
```
上面介绍的两种保存数据的方式，一种是使用Map将所有数据一次性保存到数据库，另一种是将数据分别保存到数据库的指定路径，最终的效果都是一样的：
```json
{
  "users": {
    "alanisawesome": {
      "birthYear": "1912",
      "fullName": "Alan Turing"
    },
    "gracehop": {
      "birthYear": "1906",
      "fullName": "Grace Hopper"
    }
  }
}
```
我们也可以不使用 User 对象，而使用 Map 来实现与上面相同的功能：
```java
Wilddog usersRef = new Wilddog("https://samplechat.wilddogio.com/android/saving-data/wildblog/users");

Map<String, String> alanisawesomeMap = new HashMap<String, String>();
alanisawesomeMap.put("birthYear", "1912");
alanisawesomeMap.put("fullName", "Alan Turing");

Map<String, String> gracehopMap = new HashMap<String, String>();
gracehopMap.put("birthYear", "1906");
gracehopMap.put("fullName", "Grace Hopper");

Map<String, Map<String, String>> users = new HashMap<String, Map<String, String>>();
users.put("alanisawesome", alanisawesomeMap);
users.put("gracehop", gracehopMap);

usersRef.setValue(users);
```
野狗采用的是一个“数据同步”的架构。本地拥有数据副本。对数据的写入操作，首先写入本地副本，然后 SDK 去将数据与云端进行同步。
也就是说，当 `setValue()` 方法执行完的时候，数据可能还没有同步到云端。
若要确保同步到云端完成，需要使用 `setValue()` 方法的第二个参数，该参数是一个回调函数，代码示例如下：

```java
var ref = new Wilddog("https://samplechat.wilddogio.com/android/saving-data");
ref.child("setValue").setValue("hello", new Wilddog.CompletionListener() {
  public void onComplete(WilddogError error, Wilddog ref) {
    if (error == null) {
      System.out.println("数据已成功保存到云端");
    }
  }
});
```
绝大多数操作都可设置回调函数来确保操作的完成，具体使用参见 [API文档](/docs/guide/database/android/api.html)。


## 更新数据

如果只更新指定子节点，而不覆盖其它的子节点，可以使用 `updateChildren()` 方法:

```java
//原数据如下
{
    "gracehop": {
        "nickname": "Nice Grace",
        "date_of_birth": "December 9, 1906",
        "full_name ": "Grace Lee"
    }
}
```
```js
// 只更新 gracehop 的 nickname
Wilddog hopperRef = usersRef.child("gracehop");
Map<String, Object> children = new HashMap<String, Object>();
children.put("name", "Amazing grace");
hopperRef.updateChildren(children);
```

这样会更新 gracehop 的 nickname 字段。如果我们用 `setValue()` 而不是 `updateChildren()`，那么 date_of_birth 和 full_name 都会被删除。

`updateChildren()` 也支持**多路径更新**，即同时更新不同路径下的数据且不影响其他数据，但用法上有些特殊，举例如下:
```json
//原数据如下
{
    "a": {
        "b": {
            "c": "cc",
            "d": "dd"
        },
        "x": {
            "y": "yy",
            "z": "zz"
        }
    }
}
```
```java
// 同时更新 b 节点下的 d，和 x 节点下的 z
Wilddog ref = new Wilddog("https://<appId>.wilddogio.com/a");
Map<String, Object> children = new HashMap<String, Object>();

children.put("b/d", "vvv");
children.put("x/z", "vvv");

ref.updateChildren(children);
```
可以看到，标识路径的时候，这里必须要用 `b/d`, 和 `x/z` ,而**不能**这样写：
```java
// 错误的多路径更新写法！！！
Wilddog ref = new Wilddog("https://<appId>.wilddogio.com/a");

Map<String, Object> children = new HashMap<String, Object>();
Map<String, Object> children1 = new HashMap<String, Object>();
Map<String, Object> children2 = new HashMap<String, Object>();

children1.put("d", "vvv");
children2.put("z", "vvv");

children.put("b", children1);
children.put("x", children2);

ref.updateChildren(children);
```
这样相当于 `setValue()` 操作，会把之前的数据覆盖掉。

## 追加新节点

当多个用户同时试图在同一个节点下新增一个子节点的时候，这时，数据就会被重写覆盖。
为了解决这个问题，Wilddog `push()` 采用了生成唯一ID 作为 key 的方式。通过这种方式，多个用户同时在一个节点下面 push 数据，他们的 key 一定是不同的。这个 key 是通过一个基于时间戳和随机算法生成的，即使在一毫秒内也不会相同，并且表明了时间的先后(可用来排序)，Wilddog 采用了足够多的位数保证唯一性。

用户可以用 `push` 向[博客应用](https://docs-examples.wilddogio.com/android/saving-data/wildblog)中写新内容：

```java
Wilddog ref = new Wilddog("https://docs-examples.wilddogio.com/android/saving-data/wildblog/posts");

Wilddog newRef1 = ref.push();
newRef1.child("author").setValue("gracehop");
newRef1.child("title").setValue("Announcing COBOL, a New Programming Language");

Wilddog newRef2 = ref.push();
newRef2.child("author").setValue("alanisawesome");
newRef2.child("title").setValue("The Turing Machine");
```

产生的数据都有一个唯一ID,如下:
```json
{
  "posts": {
    "-JRHTHaIs-jNPLXO": {
      "author": "gracehop",
      "title": "Announcing COBOL, a New Programming Language"
    },

    "-JRHTHaKuITFIhnj": {
      "author": "alanisawesome",
      "title": "The Turing Machine"
    }
  }
}
```

**获取唯一ID**
调用 `push()` 会返回一个引用，这个引用指向新增数据所在的节点。你可以通过调用 `getKey()` 来获取这个唯一ID

```java
// 通过push()来获得一个新的数据库地址
Wilddog newRef1 = ref.push();

// 获取push()生成的唯一ID
String postID = newRef1.getKey();
```
## 删除数据
删除数据最简单的方法是在引用上对这些数据所处的位置调用 `removeValue()`,这会把该路径下的所有数据删除:
```java
Wilddog ref = new Wilddog("https://<appId>.wilddogio.com/removeValue");
// 不带回调    
ref.removeValue();

// 或带回调，效果同上。
ref.removeValue(new Wilddog.CompletionListener(){

        public void onComplete(WilddogError error, Wilddog ref) {
            if(error != null) {
                System.out.println(error.getCode());
            }
            System.out.println("Remove Success!");                
        }
    
});
```
此外，还可以通过将 null 指定为另一个写入操作（例如，`setValue()` 或 `updateChildren()`）的值来删除数据。 您可以结合使用此方法与 `updateChildren()`，在单一 API 调用中来删除多个子节点。

**注意**：Wilddog 不会保存空路径，即如果 /a/b/c 节点下的值被设为 null，这条路径下又没其他的含有非空值的子节点存在，那么云端就会把这条路径删除。

## 事务操作
处理可能因并发修改而损坏的数据（例如，增量计数器）时，可以使用事务处理操作。你可以为此操作提供更新函数和完成后的回调（可选）。

更新函数会获取当前值作为参数，当你的数据提交到服务端时，会判断你调用的更新函数传递的当前值是否与实际当前值相等，如果相等，则更新数据为你提交的数据，如果不相等，则返回新的当前值，更新函数使用新的当前值和你提交的数据重复尝试更新，直到成功为止。

举例说明，如果我们想在一个的博文上计算点赞的数量，可以这样写一个事务：
```java
var upvotesRef = new Wilddog('https://docs-examples.wilddogio.com/saving-data/wildblog/posts/-JRHTHaIs-jNPLXOQivY/upvotes');

upvotesRef.runTransaction(new Transaction.Handler() {
    public Transaction.Result doTransaction(MutableData currentData) {
        if(currentData.getValue() == null) {
            currentData.setValue(1);
        } else {
            currentData.setValue((Long) currentData.getValue() + 1);
        }

        return Transaction.success(currentData); // 也可以中止事务 Transaction.abort()
    }

    public void onComplete(WilddogError wilddogError, boolean committed, DataSnapshot currentData) {
        // 事务完成后调用一次，获取事务完成的结果
    }
});
```
我们使用 `currentData.getValue() != null` 来判断计数器是否为空或者是自增加。 如果上面的代码没有使用事务, 当两个客户端在同时试图累加，那结果可能是为数字 1 而非数字 2。

注意：doTransaction() 可能被多次被调用，必须处理currentData.getValue()为 null 的情况。当执行事务时，云端有数据存在，但是本地可能没有缓存，此时currentData.getValue()为 null。
更多使用参见 [transaction()](/docs/guide/database/android/api.html#runTransaction-Transaction-Handler)。


