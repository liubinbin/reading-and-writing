# ThreadMap源码阅读



wiki: <https://zh.wikipedia.org/wiki/%E7%BA%A2%E9%BB%91%E6%A0%91>



##基本结构

	static final class Entry<K, V> implements Map.Entry<K, V> {
		K key;
		V value;
		Entry<K, V> left;
		Entry<K, V> right;
		Entry<K, V> parent;
		boolean color = BLACK;
		...
	}
Entry 成员变量有 key，value，左子节点，右子节点和颜色（默认为黑色）。



## 查看源码小窍门

由于 TreeMap 的代码比较干净和独立，我从源码包 src.zip 中拷贝出 TreeMap.java 重命名加入自己的工程内。就你可以对源码直接编辑了。



##核心思想

* 把红节点往上推或者往另一个子树推。





##写入数据

public V put(K key, V value) ：

1. 如果 root 为空，则先设置 root。

2. 不停的按树的结构寻找需要放 key 的 Entry。

   2.1 如果数中有对应 key 的 Entry，直接设置并返回。

   2.2 如果没有，则找到 key 对应的 parent。

3. 新建 key 和 value 的 Entry，加入到 parent 的子树中。

4. 执行 fixAfterInsertion。

5. 设置 size 和 modCount 值。


fixAfterInsertion代码解读注释如下：

```java
	private void fixAfterInsertion(Entry<K, V> x) {
		x.color = RED;

		while (x != null && x != root && x.parent.color == RED) {
			// 如果是父节点是黑色的话，则不会改变红黑树性质，不做调整。如果父节点则说明无法满足性质，需要调整。
			if (parentOf(x) == leftOf(parentOf(parentOf(x)))) {
				// 父节点是祖父节点做子节点。else里和此对应。
				// 获取叔父节点
				Entry<K, V> y = rightOf(parentOf(parentOf(x)));
				if (colorOf(y) == RED) {
					// 情形3
					setColor(parentOf(x), BLACK);
					setColor(y, BLACK);
					setColor(parentOf(parentOf(x)), RED);
					x = parentOf(parentOf(x));
				} else {
					if (x == rightOf(parentOf(x))) {
						// 情形4（此情形是为5服务的）
						x = parentOf(x);
						// 左旋
						rotateLeft(x);
					}
					// 情形5，
					setColor(parentOf(x), BLACK);
					setColor(parentOf(parentOf(x)), RED);
					// 右旋，右旋的意义在于不改变平衡树和红黑树性质的情况下，把一个红色转移到另一子树中。应为子树不会影响性质。
					rotateRight(parentOf(parentOf(x)));
				}
			} else {
				// 与if内容类似，只是更改了左右方向。
				Entry<K, V> y = leftOf(parentOf(parentOf(x)));
				if (colorOf(y) == RED) {
					setColor(parentOf(x), BLACK);
					setColor(y, BLACK);
					setColor(parentOf(parentOf(x)), RED);
					x = parentOf(parentOf(x));
				} else {
					if (x == leftOf(parentOf(x))) {
						x = parentOf(x);
						rotateRight(x);
					}
					setColor(parentOf(x), BLACK);
					setColor(parentOf(parentOf(x)), RED);
					rotateLeft(parentOf(parentOf(x)));
				}
			}
		}
		root.color = BLACK;
	}
```



## 读取数据

和二叉树的搜索一样，根据大小的比较不停的往左子树和右子树搜索。知道找到对应的值或到叶子节点。



## 删除数据

删除逻辑的思路其实比较清晰，就是一步步去构造情形6，因为6可以通过旋转达到我们改变路径中黑色节点的个数。

deleteEntry代码解读注释如下：

```java
	private void deleteEntry(Entry<K, V> p) {
		modCount++;
		size--;

		// If strictly internal, copy successor's element to p and then make p
		// point to successor.
		// 如果有两个子节点，则获取后继，然后将后继的节点的内容拷贝至p中，接下来要处理的就变成后继节点。
		if (p.left != null && p.right != null) {
			Entry<K, V> s = successor(p);
			p.key = s.key;
			p.value = s.value;
			p = s;
		} // p has 2 children

		// Start fixup at replacement node, if it exists.
		/**
		 *  在这个时候我们可以保证的是，p节点最多只有一个子节点。分为三种情况。
		 *  1. 有一个节点
		 *  2. 此节点为ROOT
		 *  3. 没有子节点
		 *  此外，我们需要做当节点是黑色时 ，我们需要进行旋转，因为删除黑色会改变红黑树的性质。
		 */
		
		Entry<K, V> replacement = (p.left != null ? p.left : p.right);

		if (replacement != null) {
			// 1. 有一个节点
			// Link replacement to parent
			// 将replacement替换给节点p，替换之后，我们就少了一个节点，如果的节点p是黑色，我们需要做些调整。
			replacement.parent = p.parent;
			if (p.parent == null)
				root = replacement;
			else if (p == p.parent.left)
				p.parent.left = replacement;
			else
				p.parent.right = replacement;

			// Null out links so they are OK to use by fixAfterDeletion.
			p.left = p.right = p.parent = null;

			// Fix replacement
			if (p.color == BLACK)
				fixAfterDeletion(replacement);
		} else if (p.parent == null) { // return if we are the only node.
			// 2. 此节点为ROOT
			root = null;
		} else { // No children. Use self as phantom replacement and unlink.
			// 3. 没有子节点
			// 节点p是的黑色的话，需要做调整，然后才能把节点p移除。
			if (p.color == BLACK)
				fixAfterDeletion(p);
			// 解除掉节点p
			if (p.parent != null) {
				if (p == p.parent.left)
					p.parent.left = null;
				else if (p == p.parent.right)
					p.parent.right = null;
				p.parent = null;
			}
		}
	}
```

fixAfterDeletion代码解读注释如下：

```java
	/** From CLR
	 * 
	 *  x 节点需要调整，x节点的路径少了个黑色节点，需要平衡。
	 *  	1. x下面的有一个黑色节点删除。
	 *  	2. x会被删除，x是黑色。
	 *  
	 *  */
	private void fixAfterDeletion(Entry<K, V> x) {
		while (x != root && colorOf(x) == BLACK) {
			if (x == leftOf(parentOf(x))) {
				Entry<K, V> sib = rightOf(parentOf(x));

				if (colorOf(sib) == RED) {
					// 情形2
					setColor(sib, BLACK);
					setColor(parentOf(x), RED);
					rotateLeft(parentOf(x));
					sib = rightOf(parentOf(x));
				}
				// 到此为止，sib变为黑色
				if (colorOf(leftOf(sib)) == BLACK && colorOf(rightOf(sib)) == BLACK) {
					// 情形3
					setColor(sib, RED);
					x = parentOf(x);
				} else {
					if (colorOf(rightOf(sib)) == BLACK) {
						// 情形5
						setColor(leftOf(sib), BLACK);
						setColor(sib, RED);
						rotateRight(sib);
						sib = rightOf(parentOf(x));
					}
					// 情形6
					setColor(sib, colorOf(parentOf(x)));
					setColor(parentOf(x), BLACK);
					setColor(rightOf(sib), BLACK);
					rotateLeft(parentOf(x));
					x = root;  // 可以终止算法，说明这种情形是我们最终的想要的。
				}
			} else { // symmetric
				...
			}
		}

		setColor(x, BLACK);
	}
```

