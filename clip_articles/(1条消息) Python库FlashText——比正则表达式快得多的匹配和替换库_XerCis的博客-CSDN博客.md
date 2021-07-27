# (1条消息) Python库FlashText——比正则表达式快得多的匹配和替换库_XerCis的博客-CSDN博客
![](https://csdnimg.cn/release/blogv2/dist/pc/img/original.png)

[XerCis](https://xercis.blog.csdn.net/) 2020-12-14 16:59:51 ![](https://csdnimg.cn/release/blogv2/dist/pc/img/articleReadEyes.png)
 355 ![](https://csdnimg.cn/release/blogv2/dist/pc/img/tobarCollect.png)
 收藏  1 

版权声明：本文为博主原创文章，遵循 [CC 4.0 BY-SA](http://creativecommons.org/licenses/by-sa/4.0/) 版权协议，转载请附上原文出处链接和本声明。

`FlashText` 可用于匹配或替换句子中的关键词。

`FlashText` 基于[FlashText 算法](https://arxiv.org/abs/1711.00046)，使用[Aho-Corasick 自动机算法](https://en.wikipedia.org/wiki/Aho%E2%80%93Corasick_algorithm)和[Trie 字典树](https://en.wikipedia.org/wiki/Trie)。

AC 自动机算法用于模式匹配，Trie 字典树用于快速寻找句子中匹配的子串。

一万个词中匹配一千个关键词，`FlashText`比编译后的正则表达式快 28 倍，而且是纯 Python 实现。

[匹配耗时](https://gist.github.com/vi3k6i5/604eefd92866d081cfa19f862224e4a0)  
![](https://img-blog.csdnimg.cn/20201214150757348.png)

[替换耗时](https://gist.github.com/vi3k6i5/dc3335ee46ab9f650b19885e8ade6c7a)  
![](https://img-blog.csdnimg.cn/20201214150836514.png)

```shell
pip install flashtext

```

匹配

```python
from flashtext import KeywordProcessor

kp = KeywordProcessor()
kp.add_keyword(keyword='my city')  
kp.add_keyword(keyword='my country', clean_name='China')  
print(len(kp))  
print(kp.extract_keywords('I love my city and my country.'))  
print(kp.extract_keywords('I love my city and my country.', span_info=True))  




kp.add_keyword(keyword='love', clean_name=['like', 'favor'])  
print(kp.extract_keywords('I love my city and my country.'))


kp1 = KeywordProcessor(case_sensitive=True)  
kp1.add_keyword(keyword='my city')  
print(kp1.extract_keywords('I love my city.'))  
print(kp1.extract_keywords('I love My City.'))  



```

替换

```python
from flashtext import KeywordProcessor

kp = KeywordProcessor()
kp.add_keyword(keyword='my country', clean_name='China')  
result = kp.replace_keywords('I love my city and my country.')
print(result)


```

```python
from flashtext import KeywordProcessor


kp = KeywordProcessor()
keyword_dict = {
    "java": ["java_2e", "java programing"],
    "product management": ["PM", "product manager"]
}
kp.add_keywords_from_dict(keyword_dict)  
kp.add_keywords_from_list(["java", "python"])  
print(kp.extract_keywords('I am a product manager for a java_2e platform'))



kp.remove_keyword('java_2e')
kp.remove_keywords_from_dict({"product management": ["PM"]})
kp.remove_keywords_from_list(["java programing"])
print(kp.extract_keywords('I am a product manager for a java_2e platform'))


```

| 方法                                                    | 描述             |
| ----------------------------------------------------- | -------------- |
| **extract_keywords(sentence, span_info=False)**       | 提取关键词          |
| replace_keywords(sentence)                            | 替换关键词          |
| add_keyword(keyword, clean_name=None)                 | 添加关键词          |
| add_keyword_from_file(keyword_file, encoding=“utf-8”) | 从文件添加关键词       |
| add_keywords_from_dict(keyword_dict)                  | 从字典添加关键词       |
| add_keywords_from_list(keyword_list)                  | 从列表添加关键词       |
| remove_keyword(keyword)                               | 删除关键词          |
| remove_keywords_from_dict(keyword_dict)               | 从字典删除关键词       |
| remove_keywords_from_list(keyword_list)               | 从列表删除关键词       |
| get_all_keywords(term_so_far=’’, current_dict=None)   | 递归构建当前关键词字典    |
| get_keyword(word)                                     | 返回关键词的干净词      |
| add_non_word_boundary(character)                      | 将被视为单词的一部分的字符  |
| set_non_word_boundaries(non_word_boundaries)          | 被认为是单词一部分的一组字符 |

1.  [FlashText GitHub](https://github.com/vi3k6i5/flashtext)
2.  [FlashText Documentation](https://flashtext.readthedocs.io/en/latest/) 
    [https://xercis.blog.csdn.net/article/details/111170748](https://xercis.blog.csdn.net/article/details/111170748)
