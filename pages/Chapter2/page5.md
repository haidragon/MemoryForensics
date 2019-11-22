# linux_arp插件
码及注释如下：
```
def calculate(self):
    linux_common.set_plugin_members(self)

    # 获取邻接表偏移地址
    neigh_tables_addr = self.addr_space.profile.get_symbol("neigh_tables")

    # 邻接表是链表么？
    # 此处的neigh_table是内核头文件里 include/net/neighbour.h里的结构
    # linux 3.19.0-25中neigh_table非链表
    if hasattr("neigh_table", "next"):
        # 是，获取内存映射文件的邻接表起始地址，随后遍历链表，生成表的list
        ntables_ptr = obj.Object("Pointer", offset = neigh_tables_addr, vm = self.addr_space)
        tables = linux_common.walk_internal_list("neigh_table", "next", ntables_ptr)
    else:
        # 否，创建一个Pointer的数组
        # 内存映射的分布
        # vm            ===>    +---------------+
        #                       |               |
        #                       |               |
        # ntables_addr  ===>    |---------------|  <=== Array.origin_offset
        # Array.current --->    |////pointer////|  <----+
        #                       |---------------|       |
        #                       |////pointer////|       |
        #                       |---------------|       +-- Array.count = 4
        #                       |////pointer////|       |
        #                       |---------------|       |
        #                       |////pointer////|  <----+
        #                       |---------------|
        #                       |               |
        #                       +---------------+
        # 然后以neigh_table来解析创建table数组，因为我们可能有多张网卡，所以会有多个arp表

        tables_arr = obj.Object(theType="Array", targetType="Pointer", offset = neigh_tables_addr, vm = self.addr_space, count = 4)
        tables = [t.dereference_as("neigh_table") for t in tables_arr]

    for ntable in tables:
        # 对每个邻接表进行相应的处理
        for aent in self.handle_table(ntable):
            yield aent
```
3.19内核头文件里的neigh_table结构：
```
struct neigh_table {
    int         family;
    int         entry_size;
    int         key_len;
    __u32           (*hash)(const void *pkey,
                    const struct net_device *dev,
                    __u32 *hash_rnd);
    int         (*constructor)(struct neighbour *);
    int         (*pconstructor)(struct pneigh_entry *);
    void            (*pdestructor)(struct pneigh_entry *);
    void            (*proxy_redo)(struct sk_buff *skb);
    char            *id;
    struct neigh_parms  parms;
    struct list_head    parms_list;
    int         gc_interval;
    int         gc_thresh1;
    int         gc_thresh2;
    int         gc_thresh3;
    unsigned long       last_flush;
    struct delayed_work gc_work;
    struct timer_list   proxy_timer;
    struct sk_buff_head proxy_queue;
    atomic_t        entries;
    rwlock_t        lock;
    unsigned long       last_rand;
    struct neigh_statistics __percpu *stats;
    struct neigh_hash_table __rcu *nht;
    struct pneigh_entry **phash_buckets;
};
```
如代码所示，该结构体不含名为next的变量，则calculate函数中第一个分支进入‘否’分支，获得邻接表的起始地址。由于我们的主机可能有多块网卡，故可能有多个邻接表。

随后将内存映像里neigh_table_addr指向的内存区域转变为Pointer的数组，数组长为4.接着对该数组中的元素，即Pointer对象，进行解引用，即获取其指向的内存区域，并组织成neigh_table对象，最后将这些表放入一个列表之中。

接着对列表中的每个邻接表进行处理，返回邻接表上邻居的集合。
handle_table函数

代码及注释如下。
```
def handle_table(self, ntable):

    ret = []

    # FIXME: Consider using kernel version metadata rather than checking hasattr
    if hasattr(ntable, 'hash_mask'):
        # 3.19没有该结构体
        hash_size = ntable.hash_mask
        hash_table = ntable.hash_buckets
    elif hasattr(ntable.nht, 'hash_mask'):
        # nht == neighbour hash table
        # 3.19也没有这个结构体
        hash_size = ntable.nht.hash_mask
        hash_table = ntable.nht.hash_buckets
    else:
        # 3.19有这个结构体
        hash_size = (1 << ntable.nht.hash_shift)
        hash_table = ntable.nht.hash_buckets

    # param:
    # hash_table 为 hash桶头指针数组的起始地址，即 neighbour **p
    # addr_space 视为 内存镜像，即一整片内存
    # hash_size 为 hash桶的个数
    # 本步骤则是将内存映象里的头指针数组转换为python里Pointer的列表
    buckets = obj.Object(theType = 'Array', offset = hash_table, vm = self.addr_space, targetType = 'Pointer', count = hash_size)

    for i in range(hash_size):
        if buckets[i]:
            # 取出其中的一个头指针，找出内存映像中偏移位置的内容，形成一个neighbour对象
            neighbor = obj.Object("neighbour", offset = buckets[i], vm = self.addr_space)

            # 以neighbor为起点遍历桶，并加入ret中
            ret.append(self.walk_neighbor(neighbor))

    # 整合桶集合成为一体
    return sum(ret, [])
```
首先是ntable是一个neigh_table的结构体。在linux内核中该结构体在上文中列过，并不存在hash_mask变量，且观察neigh_hash_table结构体中也不存在hash_mask变量，故handle_table函数的第一个分支进入子分支三，hash桶的大小为2的hash_shift次方，而hash桶的头指针数组起始地址为hash_table。其中hash_buckets即hash桶头指针数组的首地址，格式为neighbour **p。结构体neigh_hash_table的结构如下：
```
struct neigh_hash_table {
    struct neighbour __rcu  **hash_buckets;
    unsigned int        hash_shift;
    __u32           hash_rnd[NEIGH_NUM_HASH_RND];
    struct rcu_head     rcu;
};
```
随后以hash桶头指针数组的起始地址为出发点，找寻hash_size个指针，并构建为长度为hash_size的Pointer数组。

接着对数组中的元素，即hash桶头指针，取出并找出内存映象中头指针位置上的内容，构造成一个neighbour对象。

再以该对象为起点遍历整个桶，将便利结果放入ret中。最后将结果集合为一个集合返回。
walk_neighbor函数

代码及注释如下。
```
def walk_neighbor(self, neighbor):
    ret = []

    # 迭代器遍历以neighbor结构为头部的链表
    for n in linux_common.walk_internal_list("neighbour", "next", neighbor):

        # 解析
        family = n.tbl.family

        if family == socket.AF_INET:
            ip = obj.Object("IpAddress", offset = n.primary_key.obj_offset, vm = self.addr_space).v()
        elif family == socket.AF_INET6:
            ip = obj.Object("Ipv6Address", offset = n.primary_key.obj_offset, vm = self.addr_space).v()
        else:
            ip = '?'

        mac = ":".join(["{0:02x}".format(x) for x in n.ha][:n.dev.addr_len])
        devname = n.dev.name

        ret.append(a_ent(ip, mac, devname))

    return ret
```
首先遍历以neighbor结构为头部的链表。walk_internal_list("neighbour", "next", neighbor)返回一个迭代器，第一次迭代时会返回neighbor自身，即头节点。其代码如下：
```
# list_start = neighbor
# struct_name = neighbour
# list_member = next

def walk_internal_list(struct_name, list_member, list_start, addr_space = None):
    if not addr_space:
        addr_space = list_start.obj_vm

    while list_start:
        list_struct = obj.Object(struct_name, vm = addr_space, offset = list_start.v())
        yield list_struct
        list_start = getattr(list_struct, list_member)
```
如代码所示，会先取出start对应的内容先生成一个neighbour对象。

结合以上三个函数，最后就可以输出数据，即a_ent的集合。




