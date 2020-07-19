---
title: HashMap中的红黑树
category:
  - 计算机科学
  - 算法
tags:
  - 计算机科学
  - 算法
keywords: '算法,红黑树,HashMap'
abbrlink: d253f339
date: 2019-04-14 00:00:00
updated: 2019-04-14 00:00:00
---

HashMap 预期以链表数组的形式存储数据，即以 key 键的散列码计算索引，然后将元素插入到作为数组项的链表中（每个数组项称为桶）。为了提升查询的效率，HashMap 中存在一个阈值，当桶中的元素量超过这个阈值时，桶的数据结构就会从链表转变成红黑树。与红宝书中基于 2-3 树实现的红黑树不同，HashMap 中的红黑树基于 2-3-4 树实现。补充说明的是，Java 中的 TreeMap 也是基于 2-3-4 树实现的。

HashMap 中的红黑树节点通过 TreeNode 类构造，并按照按 key 键的散列码由大到小排列，这样就保证了红黑树的有序性。在链表中，相同的散列码只能存储一个元素；在红黑树中却能存储多个元素。当散列码相同时，HashMap 会通过 compareComparables 比较 key 键乃至 tieBreakOrder 实例方法比较内存地址的散列码，以决定元素在红黑树中的位置。因此，HashMap 以静态方法形式实现了两个辅助函数：comparableClassFor 用于判断 key 键的构造器是否实现了 Comparable 接口；compareComparables 静态方法用于比较 key 键。

HashMap 中的红黑树需要解决以下问题：查询节点、插入节点、删除节点、以及红黑树和链表数据结构的相互转换。为了保证红黑树和链表数据结构的高效转换，TreeNode 实例包含 prev, next 属性指向上一个或下一个节点，因此 TreeNode 既携带着红黑树的结构信息，又携带着双向链表的结构信息。

1. 插入节点：HashMap 首先会通过 putTreeVal 方法根据散列码顺序插入节点，然后通过 balanceInsertion 调整红黑树的平衡性。当树中存在相同的 key 键时，putTreeVal 方法会返回已插入的节点。
2. 删除节点：HashMap 首先会通过 removeTreeNode 方法删除节点；特定情况下，删除节点后需要 balanceDeletion 调整红黑树的平衡性。
3. 查询节点：依次通过比较 key 键的散列码、key 键、key 键内存地址的散列码在左右子树中查找节点。
4. 红黑树和链表转换：在转换之前，无论红黑树和链表都保证了 key 键的唯一性。因此，当链表转换成红黑树时，只需根据 key 键的散列码或 key 键内存地址的散列码将链表节点插入到红黑树中的特定位置，然后使用 balanceInsertion 调整树的平衡性。红黑树转换成链表时，只需顺序遍历 treeNode 节点的 next 属性即可。

以下是 TreeNode 的基本模型，同链表一样，红黑树以引用的方式构建整棵树：

```java
// 判断对象 x 的类是否实现了 Comparable 接口
static Class<?> comparableClassFor(Object x) {
    if (x instanceof Comparable) {
        Class<?> c; Type[] ts, as; Type t; ParameterizedType p;
        // getClass 方法用于在运行时获取对象的类
        if ((c = x.getClass()) == String.class) // bypass checks
            return c;
        // getGenericInterfaces 方法以 Type[] 形式获取类直接实现的接口，包含泛型参数信息
        if ((ts = c.getGenericInterfaces()) != null) {
            for (int i = 0; i < ts.length; ++i) {
                if (((t = ts[i]) instanceof ParameterizedType) &&
                    ((p = (ParameterizedType)t).getRawType() ==
                        Comparable.class) &&
                    (as = p.getActualTypeArguments()) != null &&
                    as.length == 1 && as[0] == c) // type arg is c
                    return c;
            }
        }
    }
    return null;
}

// 无法使用 Comparable 接口比较 key 键返回 0，否则返回对比结果
@SuppressWarnings({"rawtypes","unchecked"}) // for cast to Comparable
static int compareComparables(Class<?> kc, Object k, Object x) {
    return (x == null || x.getClass() != kc ? 0 :
            ((Comparable)k).compareTo(x));
}

static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V> {
    TreeNode<K,V> parent;// 父节点
    TreeNode<K,V> left;// 左子节点
    TreeNode<K,V> right;// 右子节点
    TreeNode<K,V> prev;// 上一个节点
    boolean red;
    TreeNode(int hash, K key, V val, Node<K,V> next) {
        super(hash, key, val, next);
    }

    // 当散列码等值且 key 键比较结果为 0 时，使用内存地址的散列码进行比较
    static int tieBreakOrder(Object a, Object b) {
        int d;
        if (a == null || b == null ||
            (d = a.getClass().getName().
                compareTo(b.getClass().getName())) == 0)
            // System.identityHashCode 根据对象在内存中的地址计算出散列码
            d = (System.identityHashCode(a) <= System.identityHashCode(b) ?
                    -1 : 1);
        return d;
    }
}
```

### 左旋、右旋

左旋、右旋操作的功能点在于：

1. 左旋：将作为右链接的红节点置为左链接。当父节点为左链接红节点、子节点为右链接红节点，通过左旋可以将红节点集中在左侧。
2. 右旋：将作为左链接的红节点置为右链接。当左侧父子节点同时为红节点时，通过右旋可以将其转变成 4- 节点。

#### 左旋

红宝书中的左旋操作会返回子树的根节点，以便于向上递归；HashMap 中的左旋操作不会返回子树的根节点，因此在左旋操作仍需要将新添加为子树根节点的 r 节点挂到祖父节点 pp 上（当 pp 为 null 时，则 r 为红黑树的根节点）。右旋操作同此。

![image](hhs1.gif)

```java
// p 图示中的 E，r 图示中的 S
static <K,V> TreeNode<K,V> rotateLeft(TreeNode<K,V> root, TreeNode<K,V> p) {
    TreeNode<K,V> r, pp, rl;
    if (p != null && (r = p.right) != null) {
        // 将 3 结点的中间部分挂在左节点 p（原始父节点）下
        if ((rl = p.right = r.left) != null)
            rl.parent = p;

        // 将红链接中的右节点 r（原始子节点）上移，左节点（原始父节点）下移为右节点的子节点
        // 子树（可能是包含根节点的完整二叉树）在父节点 pp 中的位置保持不变
        if ((pp = r.parent = p.parent) == null)
            (root = r).red = false;// 原始父节点即为根节点
        else if (pp.left == p)
            pp.left = r;
        else
            pp.right = r;

        // 父子节点反转
        r.left = p;
        p.parent = r;
    }
    return root;
}
```

#### 右旋

![image](hhs2.gif)

```java
// p 图示中的 S，l 图示中的 E
static <K,V> TreeNode<K,V> rotateRight(TreeNode<K,V> root, TreeNode<K,V> p) {
    TreeNode<K,V> l, pp, lr;
    if (p != null && (l = p.left) != null) {
        // 将 3 结点的中间部分挂在右节点 p（原始父节点）下
        if ((lr = p.left = l.right) != null)
            lr.parent = p;

        // 将红链接中的左节点 l（原始子节点）上移，右节点（原始父节点）下移为左节点的子节点
        // 子树（可能是包含根节点的完整二叉树）在父节点 pp 中的位置保持不变
        if ((pp = l.parent = p.parent) == null)
            (root = l).red = false;
        else if (pp.right == p)
            pp.right = l;
        else
            pp.left = l;

        // 父子节点反转
        l.right = p;
        p.parent = l;
    }
    return root;
}
```

### 插入节点

插入节点包含两个步骤：

1. 通过 putTreeVal 方法根据散列码将节点插入树的底部。
2. 通过 balanceInsertion 方法调整树的平衡性。
3. 通过 moveRootToFront 方法将根节点置于链表的首位。

#### putTreeVal

putTreeVal 基于以下逻辑插入节点：

1. 首先比较 key 键的散列码，若不同，就通过比较值将节点插入到左子树或右子树中。
2. 其次比较 key 键与树中节点是否等值，若等值，返回树中已存在的节点。
3. 其次使用 Comparable 接口比较 key 键，根据比较结果将节点插入左子树或右子树中。
4. 其次使用 tieBreakOrder 方法比较内存地址的散列码，再根据比较结果将节点插入左子树或右子树中。

```java
// h 插入节点的散列码；k 插入节点的 key；pk 红黑二叉树节点的 key
final TreeNode<K,V> putTreeVal(HashMap<K,V> map, Node<K,V>[] tab,
                                int h, K k, V v) {
    Class<?> kc = null;
    boolean searched = false;
    TreeNode<K,V> root = (parent != null) ? root() : this;
    for (TreeNode<K,V> p = root;;) {
        int dir, ph; K pk;
        // 散列码大的放在右侧，小的放在左侧
        if ((ph = p.hash) > h)
            dir = -1;
        else if (ph < h)
            dir = 1;

        // 散列码相等，比较 key 键
        else if ((pk = p.key) == k || (k != null && k.equals(pk)))
            return p;
        else if ((kc == null &&
                    (kc = comparableClassFor(k)) == null) ||
                    (dir = compareComparables(kc, k, pk)) == 0) {
            if (!searched) {
                TreeNode<K,V> q, ch;
                searched = true;
                if (((ch = p.left) != null &&
                        (q = ch.find(h, k, kc)) != null) ||
                    ((ch = p.right) != null &&
                        (q = ch.find(h, k, kc)) != null))
                    return q;
            }
            dir = tieBreakOrder(k, pk);
        }

        TreeNode<K,V> xp = p;
        if ((p = (dir <= 0) ? p.left : p.right) == null) {
            Node<K,V> xpn = xp.next;
            TreeNode<K,V> x = map.newTreeNode(h, k, v, xpn);
            if (dir <= 0)
                xp.left = x;
            else
                xp.right = x;
            xp.next = x;
            x.parent = x.prev = xp;
            // 新插入的节点置于链表的左侧
            if (xpn != null)
                ((TreeNode<K,V>)xpn).prev = x;

            moveRootToFront(tab, balanceInsertion(root, x));
            return null;
        }
    }
}
```

#### balanceInsertion

balanceInsertion 方法在向上递归的过程中需要处理的情况有以下几种：

1. 插入根节点，只需将根节点转变成黑链接即可。
2. 黑节点下插入子节点，左右两侧都是红链接，构成 4- 节点。
3. 4- 节点下插入子节点，将 4- 节点转变成 3 个 2- 节点子树。
4. 单侧插入两个红节点，通过左旋、右旋转变成 3 个 2- 节点子树。

![image](hhs3.png)

```java
// 1. 首层插入根节点，黑链接
// 2. 第二层插入的两个节点通常情况下均为红链接
static <K,V> TreeNode<K,V> balanceInsertion(TreeNode<K,V> root, TreeNode<K,V> x) {
    x.red = true;
    for (TreeNode<K,V> xp, xpp, xppl, xppr;;) {
        // x 作为根节点
        if ((xp = x.parent) == null) {
            x.red = false;
            return x;
        // 父节点为普通节点或者父节点作为根节点
        } else if (!xp.red || (xpp = xp.parent) == null)
            return root;

        // 以下均基于父子节点同时为红链接，且必然存在祖父节点的情况
        if (xp == (xppl = xpp.left)) {
            // 祖父节点下两侧节点均为红链接，构成 4- 节点，通过颜色转换将左右两侧节点置为黑链接
            // 此时将祖父节点置为红链接，以便于向上递归
            if ((xppr = xpp.right) != null && xppr.red) {
                xppr.red = false;
                xp.red = false;
                xpp.red = true;
                x = xpp;

            // 祖父节点下单侧出现两个红链接，通过旋转调整
            } else {
                // 右节点左旋，使红链接集中在左侧，同时反转父子节点
                if (x == xp.right) {
                    root = rotateLeft(root, x = xp);
                    xpp = (xp = x.parent) == null ? null : xp.parent;
                }
                // 将集中在左侧的父子节点右旋成 4- 节点
                if (xp != null) {
                    xp.red = false;
                    if (xpp != null) {
                        xpp.red = true;
                        root = rotateRight(root, xpp);
                    }
                }
            }
        } else {
            if (xppl != null && xppl.red) {
                xppl.red = false;
                xp.red = false;
                xpp.red = true;
                x = xpp;
            } else {
                if (x == xp.left) {
                    root = rotateRight(root, x = xp);
                    xpp = (xp = x.parent) == null ? null : xp.parent;
                }
                if (xp != null) {
                    xp.red = false;
                    if (xpp != null) {
                        xpp.red = true;
                        root = rotateLeft(root, xpp);
                    }
                }
            }
        }
    }
}
```

#### moveRootToFront

moveRootToFront 方法用于将红黑树的根节点置于链表的顶部。因为通过 balanceInsertion 等方法调整树平衡性时，原本子节点可能成为根节点，这样新的根节点在链表中的位置就需要得到调整。针对这种情况，moveRootToFront 方法先从链表中剔除这个新的根节点，然后将这个根节点置于链表的首位。

```java
static <K,V> void moveRootToFront(Node<K,V>[] tab, TreeNode<K,V> root) {
    int n;
    if (root != null && tab != null && (n = tab.length) > 0) {
        int index = (n - 1) & root.hash;
        TreeNode<K,V> first = (TreeNode<K,V>)tab[index];
        if (root != first) {
            Node<K,V> rn;
            tab[index] = root;

            // 将新根节点 root 从原始链表中剔除，再插入为根节点
            TreeNode<K,V> rp = root.prev;
            if ((rn = root.next) != null)
                ((TreeNode<K,V>)rn).prev = rp;
            if (rp != null)
                rp.next = rn;
            
            // 将链表的原始首节点 first 置于新的根节点 root 后
            if (first != null)
                first.prev = root;
            root.next = first;

            root.prev = null;
        }
        assert checkInvariants(root);
    }
}
```

### 删除节点

删除节点包含两个步骤：

1. 通过 removeTreeNode 方法删除当前节点。当待删除节点存在左右子节点时，删除过程中需要使用后继节点替换当前节点。
2. 移位后的待删除节点为 3- 节点的父节点，通过 balanceInsertion 方法调整树的平衡性。
3. 通过 moveRootToFront 方法将根节点置于链表的首位。

#### removeTreeNode

removeTreeNode 基于以下逻辑删除当前节点：

1. 通过重置 prev, next 属性调整双向链表的结构信息。
2. 重置根节点；当树中节点过少时，将树转换成链表。
3. 若待删除节点包含左右子节点，交换待删除节点和其后继节点的位置。可想而知的是，在完美平衡树中，这一操作会将待删除节点移到 2-3-4 树的底部。如果待删除节点为红节点，那么就构成了 4- 节点；否则构成了 3- 节点或普通 2- 节点。
4. 若待删除节点在移位后为 3- 节点中的父节点，删除节点，然后通过 balanceDeletion 调整树的平衡性。若待删除节点在移位后为 4- 节点中的子节点，直接删除。若待删除节点在移位后为普通 2- 节点，通过 balanceDeletion 调整树的平衡性，然后删除。

```java
final void removeTreeNode(HashMap<K,V> map, Node<K,V>[] tab,
                            boolean movable) {
    int n;
    if (tab == null || (n = tab.length) == 0)
        return;
    int index = (n - 1) & hash;
    TreeNode<K,V> first = (TreeNode<K,V>)tab[index], root = first, rl;
    TreeNode<K,V> succ = (TreeNode<K,V>)next, pred = prev;
    // 调整双向链表的结构信息
    if (pred == null)
        tab[index] = first = succ;
    else
        pred.next = succ;

    if (succ != null)
        succ.prev = pred;
    if (first == null)
        return;

    // 重置根节点
    if (root.parent != null)
        root = root.root();

    // 树中节点过少，将树转化成链表
    if (root == null || root.right == null ||
        (rl = root.left) == null || rl.left == null) {
        tab[index] = first.untreeify(map);  // too small
        return;
    }

    TreeNode<K,V> p = this, pl = left, pr = right, replacement;
    if (pl != null && pr != null) {
        // 查找后继节点 s
        TreeNode<K,V> s = pr, sl;
        while ((sl = s.left) != null) // find successor
            s = sl;

        boolean c = s.red; s.red = p.red; p.red = c; // swap colors
        TreeNode<K,V> sr = s.right;
        TreeNode<K,V> pp = p.parent;

        // 交换待删除节点 p 和后继节点 s 的位置
        if (s == pr) { // p was s's direct parent
            p.parent = s;
            s.right = p;
        }
        else {
            TreeNode<K,V> sp = s.parent;
            if ((p.parent = sp) != null) {
                if (s == sp.left)
                    sp.left = p;
                else
                    sp.right = p;
            }
            if ((s.right = pr) != null)
                pr.parent = s;
        }
        p.left = null;
        if ((p.right = sr) != null)
            sr.parent = p;
        if ((s.left = pl) != null)
            pl.parent = s;
        if ((s.parent = pp) == null)
            root = s;
        else if (p == pp.left)
            pp.left = s;
        else
            pp.right = s;

        if (sr != null)
            replacement = sr;
        else
            replacement = p;
    }
    else if (pl != null)
        replacement = pl;
    else if (pr != null)
        replacement = pr;
    else
        replacement = p;

    // 若 p 下还有子节点 replacement，将 replacement 挂在祖父节点下，并剔除 p
    if (replacement != p) {
        TreeNode<K,V> pp = replacement.parent = p.parent;
        if (pp == null)
            root = replacement;
        else if (p == pp.left)
            pp.left = replacement;
        else
            pp.right = replacement;
        p.left = p.right = p.parent = null;
    }

    // 首先 p 已经移到了 2-3-4 树的底部，与其他节点构成 3- 节点或普通 2- 节点、或 4- 节点
    // 当 p 为红节点时，即作为 4- 节点的子节点，无需通过 balanceDeletion 调整树的平衡性
    // 当 p 为黑节点，即作为 3- 节点的父节点或普通 2- 节点，调整树的平衡性
    // 当构成 3- 节点时，replacement 作为 p 的子节点，必为红节点
    // 当构成 2- 节点时，replacement 即为 p
    TreeNode<K,V> r = p.red ? root : balanceDeletion(root, replacement);

    // 若 p 下没有子节点（p 的位置可能经过 balanceDeletion 调整），剔除 p
    if (replacement == p) {  // detach
        TreeNode<K,V> pp = p.parent;
        p.parent = null;
        if (pp != null) {
            if (p == pp.left)
                pp.left = null;
            else if (p == pp.right)
                pp.right = null;
        }
    }

    if (movable)
        moveRootToFront(tab, r);
}
```

#### balanceDeletion

removeTreeNode 方法保证了 balanceDeletion 的执行时机，即 x 在树的底部，且 x 只可能是 3- 节点或普通 2- 节点，不可能是 4- 节点中的子红节点。在完美平衡树的机制下，x 的兄弟节点也只可能包含一级子节点。

balanceDeletion 在向上递归的过程中需要处理的情况有以下几种：

1. 当 x 为根节点或空节点，无需调整树的平衡性。
2. 当 x 为 3- 节点中的子红节点，将其转换成普通 2- 节点，无需调整树的平衡性。向上递归过程也可能导致 x 为红节点，这是也只需转换颜色即可。
3. 当 x 为 2- 节点，这时需要保障兄弟节点树的平衡性。若兄弟节点两侧都不是红节点或都不存在时，这时兄弟节点树的平衡性有所保障，只需将兄弟节点标红即可。若兄弟节点单侧包含红节点时，需要保障从父节点起的树的平衡性，这时可以按条件先将红节点右旋至右侧，再通过左旋将父节点下移、兄弟节点上移、左侄子节点挂在父节点下，以保障树的平衡性。

```java
static <K,V> TreeNode<K,V> balanceDeletion(TreeNode<K,V> root, TreeNode<K,V> x) {
    for (TreeNode<K,V> xp, xpl, xpr;;)  {
        // x 作为空节点或根节点，无需调整
        if (x == null || x == root)
            return root;
        else if ((xp = x.parent) == null) {
            x.red = false;
            return x;
        }

        // x 作为 3- 节点中的子红节点，将其转换为普通 2- 节点
        else if (x.red) {
            x.red = false;
            return root;
        }

        // x 作为普通 2-节点，且位于左侧
        else if ((xpl = xp.left) == x) {
            // 兄弟节点是红节点，左旋将父节点转换为红节点
            if ((xpr = xp.right) != null && xpr.red) {
                xpr.red = false;
                xp.red = true;
                root = rotateLeft(root, xp);
                // 原兄弟节点的左子节点作为新的 xpr 兄弟节点
                xpr = (xp = x.parent) == null ? null : xp.right;
            }
            if (xpr == null)
                x = xp;
            else {
                TreeNode<K,V> sl = xpr.left, sr = xpr.right;
                // 兄弟节点的两个子节点都不是红节点或都不存在时，兄弟节点树的平衡性有所保障，只需将兄弟节点标红
                if ((sr == null || !sr.red) &&
                    (sl == null || !sl.red)) {
                    xpr.red = true;
                    x = xp;
                }
                else {
                    // 只兄弟节点的左子节点为红节点，右旋将红节点挂于右侧
                    if (sr == null || !sr.red) {
                        if (sl != null)
                            sl.red = false;
                        xpr.red = true;
                        root = rotateRight(root, xpr);
                        xpr = (xp = x.parent) == null ?
                            null : xp.right;
                    }
                    // 只兄弟节点的右子节点为红节点，将颜色标黑
                    if (xpr != null) {
                        xpr.red = (xp == null) ? false : xp.red;
                        if ((sr = xpr.right) != null)
                            sr.red = false;
                    }
                    // 左旋将父节点下移，兄弟节点上移
                    if (xp != null) {
                        xp.red = false;
                        root = rotateLeft(root, xp);
                    }
                    x = root;
                }
            }
        }
        else { // symmetric
            if (xpl != null && xpl.red) {
                xpl.red = false;
                xp.red = true;
                root = rotateRight(root, xp);
                xpl = (xp = x.parent) == null ? null : xp.left;
            }
            if (xpl == null)
                x = xp;
            else {
                TreeNode<K,V> sl = xpl.left, sr = xpl.right;
                if ((sl == null || !sl.red) &&
                    (sr == null || !sr.red)) {
                    xpl.red = true;
                    x = xp;
                }
                else {
                    if (sl == null || !sl.red) {
                        if (sr != null)
                            sr.red = false;
                        xpl.red = true;
                        root = rotateLeft(root, xpl);
                        xpl = (xp = x.parent) == null ?
                            null : xp.left;
                    }
                    if (xpl != null) {
                        xpl.red = (xp == null) ? false : xp.red;
                        if ((sl = xpl.left) != null)
                            sl.red = false;
                    }
                    if (xp != null) {
                        xp.red = false;
                        root = rotateRight(root, xp);
                    }
                    x = root;
                }
            }
        }
    }
}
```

### 查询节点

查询节点包含如下三个方法：

* root 获取红黑树的根节点。
* find 查找当前子树中的节点。
* getTreeNode 查找完整红黑树中的节点。

```java
// 获取根节点
final TreeNode<K,V> root() {
    for (TreeNode<K,V> r = this, p;;) {
        if ((p = r.parent) == null)
            return r;
        r = p;
    }
}

// 从当前节点起查找节点：h 散列码，k key键
final TreeNode<K,V> find(int h, Object k, Class<?> kc) {
    TreeNode<K,V> p = this;
    do {
        int ph, dir; K pk;
        TreeNode<K,V> pl = p.left, pr = p.right, q;
        // 比较散列码
        if ((ph = p.hash) > h)
            p = pl;
        else if (ph < h)
            p = pr;

        // 散列码相等，判断 key 是否等值
        else if ((pk = p.key) == k || (k != null && k.equals(pk)))
            return p;
        else if (pl == null)
            p = pr;
        else if (pr == null)
            p = pl;
        
        // 使用 compareComparables 比较 key 键
        else if ((kc != null ||
                    (kc = comparableClassFor(k)) != null) &&
                    (dir = compareComparables(kc, k, pk)) != 0)
            p = (dir < 0) ? pl : pr;

        // 节点按 key 键内存地址的散列码决定位置，从左右子树中查找
        // 右子树递归调用 find 方法
        else if ((q = pr.find(h, k, kc)) != null)
            return q;
        // 左子树循环
        else
            p = pl;
    } while (p != null);
    return null;
}

// 从根节点起查找节点
final TreeNode<K,V> getTreeNode(int h, Object k) {
    return ((parent != null) ? root() : this).find(h, k, null);
}
```

### 桶交互

#### 链表 -> 红黑树

将链表转换成红黑树就是节点插入的过程，其特殊性是在这个插入过程中，key 键不存在重复值。因此该过程可分为步骤：

1. 插入根节点。
2. 根据散列码将节点插入二叉树的底部。
3. 通过 balanceInsertion 调整树的平衡性。
4. 通过 moveRootToFront 将根节点置于链表的首位。

```java
// 将链表转化成红黑二叉树，存储结构仍为链表，首节点是红黑二叉树的根节点
final void treeify(Node<K,V>[] tab) {
    TreeNode<K,V> root = null;
    for (TreeNode<K,V> x = this, next; x != null; x = next) {
        next = (TreeNode<K,V>)x.next;
        x.left = x.right = null;
        if (root == null) {// 填充根节点
            x.parent = null;
            x.red = false;
            root = x;
        } else {
            K k = x.key;// 链表节点的 key，待插入红黑二叉树中
            int h = x.hash;
            Class<?> kc = null;
            for (TreeNode<K,V> p = root;;) {
                int dir, ph;
                K pk = p.key;// 红黑二叉树节点的 key

                // 顺序比较 key 键的散列码、key 键、内存地址的散列码，确定节点的位置 dir
                if ((ph = p.hash) > h)
                    dir = -1;
                else if (ph < h)
                    dir = 1;
                else if ((kc == null &&
                            (kc = comparableClassFor(k)) == null) ||
                            (dir = compareComparables(kc, k, pk)) == 0)
                    dir = tieBreakOrder(k, pk);

                TreeNode<K,V> xp = p;

                // 将 x 插入底部，不设置红链接标识，散列码大的位于左侧
                if ((p = (dir <= 0) ? p.left : p.right) == null) {
                    x.parent = xp;
                    if (dir <= 0)
                        xp.left = x;
                    else
                        xp.right = x;
                    root = balanceInsertion(root, x);
                    break;
                }
            }
        }
    }
    moveRootToFront(tab, root);
}
```

#### 红黑树 -> 链表

```java
// 将红黑二叉树转化成链表
final Node<K,V> untreeify(HashMap<K,V> map) {
    Node<K,V> hd = null, tl = null;
    for (Node<K,V> q = this; q != null; q = q.next) {
        // 通过 replacementNode 方法将 TreeNode 转化成 Node
        Node<K,V> p = map.replacementNode(q, null);
        if (tl == null)
            hd = p;// hd 即链表中的首节点
        else
            tl.next = p;
        tl = p;// 记录上一个节点，以绑定上一个节点和当前节点的关联
    }
    return hd;
}
```

#### 扩容

当需要扩容时，调用 split 方法可以将红黑树中的元素分拆到两个桶中。

```java
final void split(HashMap<K,V> map, Node<K,V>[] tab, int index, int bit) {
    TreeNode<K,V> b = this;
    // 将红黑树拆分为两个链表
    TreeNode<K,V> loHead = null, loTail = null;
    TreeNode<K,V> hiHead = null, hiTail = null;
    int lc = 0, hc = 0;
    for (TreeNode<K,V> e = b, next; e != null; e = next) {
        next = (TreeNode<K,V>)e.next;
        e.next = null;
        if ((e.hash & bit) == 0) {
            if ((e.prev = loTail) == null)
                loHead = e;
            else
                loTail.next = e;
            loTail = e;// 记录上一个节点
            ++lc;
        }
        else {
            if ((e.prev = hiTail) == null)
                hiHead = e;
            else
                hiTail.next = e;
            hiTail = e;
            ++hc;
        }
    }

    // 根据元素量转换为红黑树或链表，并分配到不同的桶中
    if (loHead != null) {
        if (lc <= UNTREEIFY_THRESHOLD)
            tab[index] = loHead.untreeify(map);
        else {
            tab[index] = loHead;
            if (hiHead != null) // (else is already treeified)
                loHead.treeify(tab);
        }
    }
    if (hiHead != null) {
        if (hc <= UNTREEIFY_THRESHOLD)
            tab[index + bit] = hiHead.untreeify(map);
        else {
            tab[index + bit] = hiHead;
            if (loHead != null)
                hiHead.treeify(tab);
        }
    }
}
```

### 后记

文章草草整理完，回头想想，我仍感觉自己没法把握红黑树为什么会采用这种方式实现，就好像困惑于红黑树这个主意到底是谁在什么契机下想出来的那般。种种奥妙，尚未窥破，仍需努力。

### 参考

[balanceInsertion 红黑树平衡插入](https://www.cnblogs.com/oldbai/p/9890808.html)
[图解红黑树-算法导论-java实现基于HashMap1.8](https://www.jianshu.com/p/7b38abfc3298)