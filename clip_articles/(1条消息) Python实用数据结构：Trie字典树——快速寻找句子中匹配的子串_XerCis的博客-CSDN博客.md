# (1条消息) Python实用数据结构：Trie字典树——快速寻找句子中匹配的子串_XerCis的博客-CSDN博客
![](https://img-blog.csdnimg.cn/20200918095137975.png)

Trie，字典树，又称单词查找树，前缀树，是哈希树的变种。

应用：

1.  用于统计，排序和保存大量的字符串（但不仅限于字符串）
2.  经常被搜索引擎系统用于文本词频统计，可以快速找到句子中匹配的子串。

优点：

1.  利用字符串的公共前缀减少查询时间，最大限度地减少无谓字符串比较，查询效率比哈希树高。

**Trie 字典树的简单实现查阅文末**

本文将使用 Google 编写的 [`pygtrie`](https://github.com/google/pygtrie) ，特性如下：

-   完整的映射实现（类似 dict）
-   支持迭代和删除子 trie
-   支持前缀查找、最短前缀查找、最长前缀查找
-   可扩展为任意类型的用户定义键
-   `PrefixSet`支持 “所有以给定前缀开始的键” 逻辑
-   可以存储任意值，包括 None

```shell
pip install pygtrie

```

![](https://img-blog.csdnimg.cn/20210518114136251.png)

默认分隔符为 `/`

```python
import pygtrie

t = pygtrie.StringTrie(separator='/')
t['foo'] = 'Foo'
t['foo/bar'] = 'Bar'
t['foo/bar/baz'] = 'Baz'
del t['foo/bar']
print(t.keys())  
del t['foo':]
print(t.keys())  

```

![](https://img-blog.csdnimg.cn/20210518114919641.png)

前缀无值取不了

```python
import pygtrie

t = pygtrie.StringTrie()
t['foo/bar'] = 'Bar'
t['foo/baz'] = 'Baz'
t['qux'] = 'Qux'
print(t['foo/bar'])  
print(sorted(t['foo':]))  




```

![](https://img-blog.csdnimg.cn/20210518115649826.png)

切片将清除条目

```python
import pygtrie

t = pygtrie.StringTrie()
t['foo/bar'] = 'Bar'
t['foo/baz'] = 'Baz'
print(sorted(t.keys()))


t['foo':] = 'Foo'
print(t.keys())


```

![](https://img-blog.csdnimg.cn/20210518115816409.png)

存在节点：`has_node(key)`

节点有值：`HAS_VALUE` 或 `has_key(key)`

节点有子树：`HAS_SUBTRIE` 或 `has_subtrie(key)`

```python
import pygtrie

t = pygtrie.StringTrie()
t['foo/bar'] = 'Bar'
t['foo/bar/baz'] = 'Baz'
print(t.has_node('qux') == 0)  
print(t.has_node('foo/bar/baz') == pygtrie.Trie.HAS_VALUE)  
print(t.has_node('foo') == pygtrie.Trie.HAS_SUBTRIE)  
print(t.has_node('foo/bar') == (pygtrie.Trie.HAS_VALUE | pygtrie.Trie.HAS_SUBTRIE))  





```

可直接使用更方便的封装

```python
import pygtrie

t = pygtrie.StringTrie()
t['foo/bar'] = 'Bar'
t['foo/bar/baz'] = 'Baz'
print(t.has_key('qux'), t.has_subtrie('qux'))  
print(t.has_key('foo/bar/baz'), t.has_subtrie('foo/bar/baz'))  
print(t.has_key('foo'), t.has_subtrie('foo'))  
print(t.has_key('foo/bar'), t.has_subtrie('foo/bar'))  

```

![](https://img-blog.csdnimg.cn/2021051812072222.png)

`items()`：只输出有值的节点

`iteritems()`：只输出有值的节点（生成器）

`prefix` 参数：只生成具有指定前缀的项

`shallow` 参数：如果一个节点有值，则不遍历子节点

```python
import pygtrie

t = pygtrie.StringTrie()
t['foo'] = 'Foo'
t['foo/bar/baz'] = 'Baz'
t['qux'] = 'Qux'

print(sorted(t.items()))  


print(t.items(prefix='foo'))  


print(sorted(t.items(shallow=True)))


```

items 按拓扑顺序生成，兄弟姐妹顺序是不确定的

可以牺牲效率为代价，调用 `Trie.enable_sorting()` 确保顺序

`traverse()` 也可以遍历，和 `iteritems()` 相比有两个优点：

1.  基于父节点属性遍历节点列表时，允许完全跳过子节点
2.  它直接表示为 trie 结构，易于构造成不同的树

打印当前目录的所有文件，统计 HTML 文件数，忽略隐藏文件

```python
import os
import pygtrie

t = pygtrie.StringTrie(separator=os.sep)

for root, _, files in os.walk('.'):
    for name in files:
        t[os.path.join(root, name)] = True


def traverse_callback(path_conv, path, children, is_file=False):
    if path and path[-1] != '.' and path[-1][0] == '.':
        
        return 0
    elif is_file:
        
        print(path_conv(path))
        return int(path[-1].endswith('.html'))
    else:
        
        return sum(children)


print(t.traverse(traverse_callback))

```

![](https://img-blog.csdnimg.cn/20210518122028188.png)

`longest_prefix(key)`

```python
import pygtrie

t = pygtrie.StringTrie()
t['foo'] = 'Foo'
t['foo/bar/baz'] = 'Baz'
print(t.longest_prefix('foo/bar/baz/qux'))  
print(t.longest_prefix('foo/bar/baz/qux').key)  
print(t.longest_prefix('foo/bar/baz/qux').value)  
print(t.longest_prefix('does/not/exist'))  
print(bool(t.longest_prefix('does/not/exist')))  

```

![](https://img-blog.csdnimg.cn/20210518122028188.png)

`shortest_prefix(key)`

```python
import pygtrie

t = pygtrie.StringTrie()
t['foo'] = 'Foo'
t['foo/bar/baz'] = 'Baz'
print(t.shortest_prefix('foo/bar/baz/qux'))  
print(t.shortest_prefix('foo/bar/baz/qux').key)  
print(t.shortest_prefix('foo/bar/baz/qux').value)  
print(t.shortest_prefix('does/not/exist'))  
print(bool(t.shortest_prefix('does/not/exist')))  

```

![](https://img-blog.csdnimg.cn/20210518122028188.png)

`prefixes(key)`

```python
import pygtrie

t = pygtrie.StringTrie()
t['foo'] = 'Foo'
t['foo/bar/baz'] = 'Baz'
print(list(t.prefixes('foo/bar/baz/qux')))  
print(list(t.prefixes('does/not/exist')))  

```

![](https://img-blog.csdnimg.cn/20210518153222843.png)

`pygtrie.CharTrie` 接受字符串作为键，与 `pygtrie.Trie` 的不同之处在于调用 `.keys()` 时返回字符串的键

常见例子是自然语言中的单词字典（如图，每个橙色点代表一个单词）

```python
import pygtrie

t = pygtrie.CharTrie()
t['wombat'] = True
t['woman'] = True
t['man'] = True
t['manhole'] = True

print(t)  
print(t.has_subtrie('wo'))  
print(t.has_key('man'))  
print(t.has_subtrie('man'))  
print(t.has_subtrie('manhole'))  

```

`pygtrie.StringTrie` 接受字符串作为分隔符，默认为 `/`

常见例子是路径映射到一个请求

```python
import pygtrie


def handle_root():
    print('root')


def handle_admin():
    print('admin')


def handle_admin_images():
    print('admin_images')


handlers = pygtrie.StringTrie()
handlers[''] = handle_root
handlers['/admin'] = handle_admin
handlers['/admin/images'] = handle_admin_images
print(handlers.keys())


request_path = '/admin/images/foo'
handler = handlers.longest_prefix(request_path).value
handler()  


```

```python
import collections


class TrieNode:
    def __init__(self):
        self.children = collections.defaultdict(TrieNode)
        self.is_word = False

    def __repr__(self):
        s = ''
        first = True
        for k, v in self.children.items():
            if first:
                if v.is_word:
                    s += '{} -> {}\n'.format(k, v)
                else:
                    s += '{} -> {}'.format(k, v)
                first = False
                continue
            if v.is_word:
                s += '{}\n'.format(k)
            else:
                s += '{} -> {}'.format(k, v)
        return s


class Trie:
    def __init__(self):
        self.root = TrieNode()

    def insert(self, word):
        current = self.root
        for letter in word:
            current = current.children[letter]
        current.is_word = True

    def search(self, word):
        current = self.root
        for letter in word:
            current = current.children.get(letter)
            if current is None:
                return False
        return current.is_word

    def starts_with(self, prefix):
        current = self.root
        for letter in prefix:
            current = current.children.get(letter)
            if current is None:
                return False
        return True

    def __repr__(self):
        return repr(self.root).replace('\n\n', '\n').replace('\n\n', '\n')

    def find_one(self, word):
        '''找到第一个匹配的词

        :param word: str
        :return: 第一个匹配的词 or None

        >>> a = Trie()
        >>> a.insert('感冒')
        >>> a.find_one('我感冒了好难受怎么办')
        '感冒'
        '''
        for i in range(len(word)):
            c = word[i]
            node = self.root.children.get(c)
            if node:
                for j in range(i + 1, len(word)):
                    _c = word[j]
                    node = node.children.get(_c)
                    if node:
                        if node.is_word:
                            return word[i:j + 1]
                    else:
                        break
        return None


if __name__ == '__main__':
    a = Trie()
    a.insert('张三')
    a.insert('张')
    a.insert('李四')
    a.insert('王五五')
    print(a)
    print(a)
    print(a.find_one('同学有张三、李四'))
    
    
    
    
    
    

```

1.  [Python 库 FlashText——比正则表达式快得多的匹配和替换库](https://xercis.blog.csdn.net/article/details/111170748)

2.  [数据结构与算法（十一）Trie 字典树](https://blog.csdn.net/yuzhiqiang666/article/details/80711441)

3.  [trie 的 Python 实现](https://github.com/keon/algorithms/blob/master/algorithms/tree/trie/trie.py)

4.  [pygtrie GitHub](https://github.com/google/pygtrie)

5.  [pygtrie Documentation](https://pygtrie.readthedocs.io/en/latest/) 
    [https://blog.csdn.net/lly1122334/article/details/108649593](https://blog.csdn.net/lly1122334/article/details/108649593)
