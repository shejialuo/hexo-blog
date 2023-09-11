---
title: MIT6.858-SUNDR的实现
tags:
  - 学习
categories:
  - 安全
date: 2023-09-11 15:09:18
---


MIT6.858最后一个实验是复现论文[Secure Untrusted Data Repository](https://www.usenix.org/legacy/event/osdi04/tech/full_papers/li_j/li_j.pdf)的串行化实现。在开始做这个实验之前，你应该仔细地阅读这篇论文。

## 源码阅读

MIT6.858提供了基本的文件系统代码，了解这个文件系统是如何构建的，对于代码实现的相当重要。

### 基本的类型定义

在SUNDR中，有用户和群组的划分。同时每一个文件都是由`<principal, i-number>`对应的。显然，每个用户和群组都可以有多个文件，所以只需要再定义一个类包含用户（群组）和所有的文件个数。故源代码中定义了三个类型：

+ `User`：继承于`Principal`。
+ `Group`：继承于`Principal`。
+ `I`：包含`Principal`和`n`。

### 基本的存储实现

在文件系统中，最基本的存储单元是`Block`类，其可以存储数据、inode等信息。由于SUNDR基于快照的机制保障安全，故其与git的实现机制相似，基于content-hash机制对每一个Block进行哈希。故`Block`类提供了两个函数：

+ `store(blob)`：`blob`是存储的数据，其为二进制数据。其返回值为二进制内容的sha1sum。
+ `load(chash)`：根据哈希值找到相应的block，进而读取block中的二进制数据。

同时，文件系统定义了`Inode`类，用于作为元数据。其包含了某一个文件或者目录的基本信息，例如类型，创建的时间，修改的时间及其对应的所有的block的哈希值。

```py
class Inode:
    def __init__(self):
        self.size = 0
        self.kind = 0 # 0 is dir, 1 is file
        self.ex = False
        self.ctime = 0
        self.mtime = 0
        self.blocks = []
```

显然，`Inode`类也是存储在block中的，所以我们需要基于`ihash`通过调用`block.load(chash)`函数得到其二进制数据，然后进行反序列化得到其字段值。当我们想要获得文件的值时，我们直接使用如下的函数即可：

```py
def read(self):
    return b"".join([secfs.store.block.load(b) for b in self.blocks])
```

然而，我们必须考虑目录这种情况。在SUNDR中，目录的每一项元素都是一个元组`(filename, I)`。其中其定义了`add`函数来实现在一个目录里面添加一个新项。同时定义了`find_under`函数用来查找目录。

### Itable

每个用户和群组都拥有一个`i-table`。对于用户而言，`i-table`保存的是`i-number`到`ihash`的映射。对于群组而言，`i-table`保存的是`i-number`到`I`的映射。因此对于`Itable`类而言，源码利用了python的动态性，直接定义了一个dict。

```py
class Itable:
    def __init__(self):
        self.mapping = {}

    def load(ihandle):
        b = secfs.store.block.load(ihandle)
        if b == None:
            return None

        t = Itable()
        t.mapping = pickle.loads(b)
        return t

    def bytes(self):
        return pickle.dumps(self.mapping)
```

然后，其定义了一个关键的函数`resolve`，基于`i-table`解析得到`ihash`或者`I`。同时，该函数支持递归地解析`I`最终得到`ihash`。其思路实现比较简单。

最关键的在于`modmap(mod_as, i, ihash)`函数，对于某个用户`mod_as`而言，给定一个`I`类，需要修改或者创建其在`i-table`中的映射。然而关键在于`I`有两类情况，一种是用户，另一种是群组。

+ 对于用户而言，其`mod_as`和`i.p`必须相等。我们需要修改或者创建其在`i-table`中的映射，这个过程相当简单。
+ 对于群组而言，情况就会比较复杂了。我们首先需要基于提供的`i`找到其对应的`I`，也就是调用`resolve`函数并进行一层的解析。如果`I.p = mod_as`，证明群组的修改和用户的修改是一致的，我们一定会最先对用户的修改进行处理，所以此处不需要进行任何操作。如果不是那么证明我们需要创建一个新的`I`，于是调用`modmap(mod_as, I(mod_as), ihash)`在`mod_as`用户的`i-table`中添加映射。

### 文件系统API接口

源码已经给出了某些文件系统接口的实现，例如`link`函数，`link`函数的作用即将一个`I`添加到当前目录中，其原理很简单，首先需要基于当前目录构建一个`Directory`。然后调用`add`函数添加目录，并使用`modmap`修改映射关系。

```py
# Omit syntax check and write check
def link(def link(link_as: User, i: I, parent_i: I, name: str):
    parent_ihash = secfs.store.tree.add(parent_i, name, i)
    secfs.tables.modmap(link_as, parent_i, parent_ihash))
```

在SUNDR论文中，文件系统的初始化是root用户生成一对私钥/公钥，同时创建`.users`和`.groups`文件，存储相应的公钥及印映射信息。文件系统使用`init`函数提供了接口。其首先需要创建一个类型为目录的inode，然后添加`.`和`..`目录。同时需要创建`.users`和`.groups`文件，调用`link`函数添加到根目录中。

在了解了基本的数据结构以及映射关系后，我们就可以开始编写自己的代码。

## 实现

### 实现创建文件和目录的功能

我们需要所做的第一个工作就是实现创建文件和目录。无论是文件还是目录我们都需要首先创建一个`Inode`。然后我们需要将其存储在文件系统中，并得到其hash值`ihash`。然后我们需要调用`modmap`修改itable的映射关系。对于目录而言，我们需要做更多额外的工作。我们需要添加在当前目录创建`.`和`..`，并调用`modmap`修改itable的映射关系。然后我们调用`link`维护目录结构关系。

然而，我们需要考虑如果是为一个群组创建文件和目录呢？实际上面的流程都是一样的，因为对于群组来说，`itable`是存储的映射，所以我们仅仅只需要更新这个映射关系即可。

```py
def _create(parent_i: I, name: str, create_as: User, create_for: Principal, isdir: bool):
    ...
    ihash = secfs.store.block.store(node.bytes())
    store_i = secfs.tables.modmap(create_as, I(create_for), ihash)

    if isdir:
        new_ihash = secfs.store.tree.add(store_i, b'.', store_i)
        secfs.tables.modmap(create_as, store_i, new_ihash)
        new_ihash = secfs.store.tree.add(store_i, b'..', parent_i)
        secfs.tables.modmap(create_as, store_i, new_ihash)

    if create_for.is_group():
        secfs.tables.modmap(create_as, I(create_for), store_i)

    link(create_as, store_i, parent_i, name)

    return store_i
```

### Version Structure List实现

目前所有的映射关系都是存储在`current_table`中的，其是存储在内存中的，当有其他客户端时，其访问服务端，得不到`current_table`的信息，所以需要对`current_table`进行修改，其需要存储在文件系统中。在这个过程中，我们不考虑任何安全方面的实现。

我们首先需要认识到`current_table`保存的是`Principal`到`Itable`的映射。由于函数`resolve`以及`modmap`严重依赖于`current_table`，所以应该尽可能合理地设计数据结构，减少代码的更改。

在SUNDR原论文中，其提供的数据结构`VersionStructure`包含每一个用户的`i-handle`，所属的群组的`g-handle`以及`version_vector`。这是与当前的逻辑矛盾的，当前的映射单独保存了用户和群组的，为了简便起见，我没有采取这样的数据结构。我仍然按照当前的逻辑进行实现。因此，我定义了如下的数据结构。

```py
class VersionStructure:
    """
    `VersionStructure` is the most important data structure in this lab. In
    the original SUNDR paper, the version structure is specified only for
    the `User`. Each user could have one and more group handles. However,
    it's a bad idea for the current code. Because the `current_itables`
    mapping the `I` to the inode. The `I.p` could be user or the group.
    """

    version_vector : dict[Principal, int]
    # It could be the user or the group, it is only the hash, should
    # later read from
    i_handle: str

    def __init__(self):
        self.version_vector = {}
        self.i_handle = ""
```

根据SUNDR论文，每次每个客户端对文件系统进行操作之间，都需要从服务器下载最新的VSL，当进行了修改需要对VSL进行验证，如果验证通过，则进行数字签名，上传VSL。从性能的角度来说，传递单独的`VersionStructure`是节省带宽的，然而我认为此处不是这个lab的核心，因此我才用了最粗暴的方式，即上传整个VSL。故定义了如下的数据结构：

```py
class VersionStructureList:
    """
    `VersionStructureList` is the core data structure here, It
    contains the the user or the group `VersionStructure`.
    """

    version_structures: dict[Principal, VersionStructure]

    def __init__(self) -> None:
        self.version_structures = {}

    def download(self) -> None:
        """
        Download the `version_structure_list` from the server, it should
        be called every time it operates on the file system. It's may
        be a bad idea, but it is simple.
        """

        global server
        blob = server.read_version_structure_list()

        if blob == None:
            return

        if "data" in blob:
            import base64
            blob = base64.b64decode(blob["data"])

        self.version_structures = pickle.loads(blob)

    def upload(self) -> None:

        global server
        blob = pickle.dumps(self.version_structures)
        server.store_version_structure_list(blob)
```

我们要在server端添加`store_version_structure_list`RPC调用，以及`read_version_structure_list`RPC调用。

```py
class SecFSRPC():
    ...
    @Pyro4.expose
    def read_version_structure_list(self):
        chash = self.version_structure_list_hash
        if chash != None and chash in self.blocks:
            return self.blocks[chash]
        return None

    @Pyro4.expose
    def store_version_structure_list(self, blob):
        if "data" in blob:
            import base64
            blob = base64.b64decode(blob["data"])

        import hashlib
        chash = hashlib.sha224(blob).hexdigest()
        self.blocks[chash] = blob
        self.version_structure_list_hash = chash
```

现在到达了最关键的一步，也就是我们需要修改`resolve`和`modmap`中的代码，我们需要替换掉`current_table`。我们需要一点即可，即`VersionStructure`存储的是`i-handle`我们必须进行磁盘的读写，而不是类似源码中直接存储`Itable`（实际上这一步是可以优化的，如果其`i-handle`没有发生变化，我们可以实现缓存，而不是从磁盘中实现读写，然而我忽略了因为这并不是这个lab的核心）。

```py
def resolve(i: I, resolve_groups = True):
    ...
    global version_structure_list
    if principal not in version_structure_list.version_structures:
        # User does not yet have an itable
        return None

    i_handle = version_structure_list.version_structures[principal].i_handle
    t = Itable.load(i_handle)
    ...
```

```py
def modmap(mod_as: User, i: I, ihash) -> I:

    ...
    # find (or create) the principal's itable
    t = None
    global version_structure_list
    if i.p not in version_structure_list.version_structures:
        if i.allocated():
            # this was unexpected;
            # user did not have an itable, but an inumber was given
            raise ReferenceError("itable not available")
        t = Itable()
        version_structure_list.version_structures[i.p] = VersionStructure()
        print("no current list for principal", i.p, "; creating empty table", t.mapping)
    else:
        i_handle = version_structure_list.version_structures[i.p].i_handle
        t = Itable.load(i_handle)

    ...

    i_handle = secfs.store.block.store(t.bytes())
    version_structure_list.version_structures[i.p].i_handle = i_handle

    return i
```

到了此处，我们就可以通过百分之87的测试。可见，我们已经完成了大部分的工作。剩下的就是实现安全方面的功能，这个lab最难的并不在于代码，而是你如何进行数据结构的设计。

### 安全方面实现

要实现安全方面就很简单了，需要考虑如下两个（由于时间关系，此处我并没有进行验证）：

1. 验证服务器端的身份
2. 实现SUNDR协议
