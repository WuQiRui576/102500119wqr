# BPE 实现所需的知识储备总结





## 一、 Python 进阶与高级数据结构 （Programming Fundamentals）



要高效且优雅地实现 BPE，您需要超越 Python 基础语法的知识。

| **知识领域**      | **核心概念**                                                 | **在 BPE 代码中的应用**                                      |
| ----------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| **高级数据结构**  | **`Collections` 模块**：特别是 和 。`defaultdict``Counter`   | **`defaultdict（int）：`** 用于在 中高效地统计符号对频率，无需检查键是否存在。**`Counter`**： 用于在 开始时高效统计单词频率。`get_stats``train` |
| **字典操作**      | **字典推导式 （Dictionary Comprehension）** 和 。`dict.items()` | 用于快速构建 ID 和 Token 之间的映射（例如在 中创建反向词汇表）。`decode` |
| **列表操作**      | **列表切片 （List Slicing）** 和列表拼接。                   | 在  方法中，用于高效地将符号对替换成新的合并符号 ()。`tokenize``word_tokens[:i] + [merged] + word_tokens[i+2:]` |
| **函数式编程**    | **`max（）` 函数的 `key` 参数**。                            | 在  中，使用  快速找到字典中**值最大**的键（即频率最高的符号对）。`train``max(pairs, key=pairs.get)` |
| **迭代器/生成器** | 知道  和  返回的是视图对象，以及  强制转换。`keys()``items()``list()` | 了解在  中迭代  是如何工作的。`get_stats``vocab.items()`     |



## 二、 正则表达式 （Regular Expressions - 模块）`re`



`re`模块是 BPE 代码中最关键的“魔法”所在，它使得全局合并作既准确又高效。

| **知识领域**          | **核心概念**                                                 | **在 BPE 代码中的应用**                                      |
| --------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| **模块和编译**        | **`import re`** 和 **`re.compile（）`**                      | `re.compile()`用于预编译正则表达式模式，这是提高  在训练循环中执行效率的关键。`merge_vocab` |
| **基本匹配**          | **`re.sub(pattern, repl, string)`**                          | 用于在  中执行实际的文本替换操作，将所有匹配的符号对替换成合并后的新符号。`merge_vocab` |
| **特殊字符处理**      | **`re.escape(string)`**                                      | 确保符号对中的字符（例如，如果符号本身包含  或  等正则元字符）被当作字面文本来匹配，而不是正则指令。`.``*` |
| **零宽断言 （环顾）** | **负向后顾断言 `（？<！\S）`** 和 **负向先行断言 `（？！\s）。`** | **最重要且最难理解的部分。它确保** 只有在前后都是**空白字符**或**字符串边界**时才被匹配和替换。这保证了我们只合并由空格分隔的**完整符号**，防止误合并。`bigram` |



## 三、 自然语言处理基础 （NLP Fundamentals）



要理解 BPE 的目标和意义，您需要以下 NLP 基础知识。

| **知识领域** | **核心概念**                                 | **在 BPE 代码中的应用**                                      |
| ------------ | -------------------------------------------- | ------------------------------------------------------------ |
| **分词问题** | **OOV （Out-of-Vocabulary） 问题**           | 理解 BPE 诞生的原因：通过创建子词来解决词典中没有罕见词的问题。 |
| **子词分词** | BPE/WordPiece/SentencePiece 算法的核心思想。 | BPE 的核心思想：从字符开始，迭代合并最频繁的符号对。         |
| **特殊标记** | **词尾标记 `</w>`**                          | 理解  的作用：区分一个子词是词缀还是完整的词，使  和  的前缀  得到不同的处理。`</w>``run``running``run` |
| **令牌 ID**  | **词汇表 （Vocabulary） 映射**               | 理解将文本（Token）映射为整数 ID 是所有机器学习模型处理文本的必要步骤。 |

------



## 学习路线建议



如果您想系统学习，建议按照以下路线进行：

1. **深入 Python 进阶**：重点掌握 模块、字典推导式、列表切片以及 函数的 用法。`collections``max()``key`
2. **精通正则表达式**：花时间理解 、，以及最关键的**零宽断言**。这是将您的 BPE 实现与其他简单字符串作区分开来的核心技术。`re.compile()``re.sub()`
3. **理解 BPE 流程**：理解 如何统计， 如何替换，以及 如何应用规则（尤其是**贪婪和优先级**）。`get_stats``merge_vocab``tokenize`

通过掌握这些知识，您不仅能完成这段代码，还能完全理解现代 NLP 模型中分词器的工作原理。

------



# 详细 BPE 算法（）构建教程`SimpleBPE`





## 核心概念解释



| **代码结构**                   | **解释**                                                     |           |
| ------------------------------ | ------------------------------------------------------------ | --------- |
| `re`模块                       | Python 内置的正则表达式模块，用于文本模式匹配和替换。在 BPE 中用于高效的全局合并。 | 进阶      |
| `defaultdict(int)`             | 继承自 ，用于创建带有默认值（）的字典。在统计频率时，无需先检查键是否存在。`dict``0` | 进阶      |
| `Counter(list)`                | 继承自 ，用于快速统计列表中元素或可迭代对象中元素的频率。`dict` | 基础/进阶 |
| `re.compile()`                 | 预编译正则表达式模式，提高在循环中重复使用时的效率。         | 进阶      |
| `re.escape()`                  | 对字符串中所有特殊字符进行转义，确保它们被视为普通字符。在构造模式时使用，以匹配符号对。 | 进阶      |
| `r'(?<!\S)' + ... + r'(?!\S)'` | 正则表达式模式： 是负向后顾断言，表示前面不能是任何非空白字符（即前面是空白或字符串开头）。是负向先行断言，表示后面不能是任何非空白字符（即后面是空白或字符串结尾）。**用于确保只替换完整的、由空格分隔的符号。**`(?<!\S)``(?!\S)` | 进阶      |

------



## 第 1 步：初始化与数据准备



我们从  开始，并在  方法中准备初始数据。`__init__``train`



### 代码实现



python

```
import re
from collections import defaultdict, Counter

class SimpleBPE:
    def __init__(self, vocab_size=1000):
        self.vocab_size = vocab_size
        self.vocab = {}    # 最终的词汇表：{token: id}
        self.merges = {}   # 记录所有合并规则：{(pair_1, pair_2): merged_token}
    
    def train(self, text):
        text = text.lower()
        words = text.split()
        word_counts = Counter(words) # 统计单词频率
        
        vocab = {}
        for word, freq in word_counts.items():
            # 将每个单词拆分为字符，并在末尾加上词尾标记 '</w>'
            tokenized_word = ' '.join(list(word)) + ' </w>'
            vocab[tokenized_word] = freq
            
        # ... (后续训练步骤)
        # ...
```



### 解释



- **`self.vocab_size`**： 决定了我们最多创建多少个子词符号（包括初始字符）。
- **`self.merges`**： 存储了 BPE 算法的“秘籍”，即每次合并的符号对和生成的新符号。
- **`' '.join（list（word））：`**
  - `list(word)`： 将单词（如 “hello”）拆成字符列表 。`['h', 'e', 'l', 'l', 'o']`
  - `' '.join(...)`: 用空格将字符列表连接成字符串 。`"h e l l o"`
- **`Vocalb` 初始化**： 存储当前状态下的**符号序列及其频率**，这是 过程中的主要作对象。`train`

------



## 第 2 步：统计相邻符号对的频率 (`get_stats`)



这是 BPE 算法的核心步骤：找出所有符号对中，哪个对出现的总次数最多。



### 代码实现



python

```
    def get_stats(self, vocab):
        """统计相邻符号对的频率"""
        pairs = defaultdict(int)
        for word, freq in vocab.items():
            symbols = word.split()  # 例如: ['h', 'e', 'l', 'l', 'o', '</w>']
            for i in range(len(symbols) - 1):
                # 统计相邻符号对 (symbols[i], symbols[i+1])
                # 并乘以该单词的频率 (freq)
                pairs[symbols[i], symbols[i+1]] += freq
        return pairs
```



### 解释



- **`defaultdict（int）：`** 确保我们可以在不知道键是否存在的条件下直接使用 来累加频率，如果键不存在，它会自动初始化为 。`pairs[key] += value``0`
- **`symbols = word.split（）`**： 将 中的键（如 ）重新拆分成符号列表，以便迭代相邻对。`vocab``"h e l l o </w>"`
- **`pairs[symbols[i]， symbols[i+1]] += 频率`**：
  - 键是一个元组 ，例如 。`(符号A, 符号B)``('h', 'e')`
  - 值是这对符号在所有单词中出现的总次数（必须乘上单词本身的频率 ）。`freq`

------



## 第 3 步：执行符号对合并 (`merge_vocab`)



选出频率最高的符号对后，我们需要在整个词汇表上执行一次替换。



### 代码实现



python

```
    def merge_vocab(self, pair, vocab):
        """合并指定的符号对"""
        bigram = ' '.join(pair)        # 例如: 't e'
        replacement = ''.join(pair)     # 例如: 'te'
        new_vocab = {}
        
        # 匹配模式：确保只替换独立的符号 't e'，而不是单词中的子字符串 'the' 中的 't e'
        pattern = re.compile(r'(?<!\S)' + re.escape(bigram) + r'(?!\S)')
        
        for word, freq in vocab.items():
            # 使用预编译的正则模式进行替换
            new_word = pattern.sub(replacement, word)
            new_vocab[new_word] = freq
        return new_vocab
```



### 解释



- **`re.compile（...）`**： 预编译正则模式，这是性能优化的好习惯。
- **`re.escape（bigram）：`** 如果 中包含 、 等正则特殊字符， 会将其转义，确保它被匹配为字面意义上的符号。`bigram``.``*``re.escape()`
- **`r'（？<！\S）' + ... + r'（？！\S）'`**： **这是最关键的进阶点。它保证**了替换的精确性：
  - 它要求匹配到的  (如 ) 前后必须是**非非空白字符**（即空白字符或字符串边界）。在  中，符号总是被空格分隔，这个模式确保了  不会被替换成 ，而  会被替换成 。`bigram``"t e"``vocab``"t e"``"te"``"t e s t"``"te s t"`
- **`pattern.sub（replacement， word）：`** 执行替换，将旧的 状态转换为新的 状态。`vocab``new_vocab`

------



## 第 4 步：构建最终词汇表 ( 的收尾部分)`train`



回到 方法，完成训练循环并建立最终的符号 ID 映射。`train`



### 代码实现（仅展示关键更新部分）



python

```
    def train(self, text):
        # ... (初始化和循环找到 best_pair) ...

        while len(self.vocab) < self.vocab_size:
            # ... (统计并找到 best_pair, best_freq) ...
            
            # 合并符号对
            vocab = self.merge_vocab(best_pair, vocab)
            
            # 记录合并操作和新符号
            new_token = ''.join(best_pair)
            self.merges[best_pair] = new_token
            
            # 赋予新符号 ID
            self.vocab[new_token] = len(self.vocab)
            
            # ... (打印结果) ...
        
        # ... (返回最终 vocab) ...
```



### 解释



- 每次成功的合并都会产生一个 。`new_token`
- 我们将  ->  的关系记录到  中，供后续  使用。`(pair_1, pair_2)``new_token``self.merges``tokenize`
- 将 加入 并赋予一个递增的 ID （）。 训练循环就是不断增加  的大小，直到达到 。`new_token``self.vocab``len(self.vocab)``self.vocab``vocab_size`

------



## 第 5 步：使用合并规则进行分词 (`tokenize`)



分词器接收新文本，并根据训练过程中记录的  规则来拆分单词。`self.merges`



### 代码实现



python

```
    def tokenize(self, text):
        """使用训练好的BPE模型对文本进行分词"""
        text = text.lower()
        words = text.split()
        tokens = []
        
        for word in words:
            word_tokens = list(word) + ['</w>']
            
            while True:
                best_pair_in_word = None
                best_pair_index = -1
                
                # 遍历所有合并规则，优先使用最新的/最长的规则（即 self.merges.keys() 的逆序）
                for merge_pair in reversed(list(self.merges.keys())):
                    # 查找当前 word_tokens 列表中是否存在该符号对
                    for i in range(len(word_tokens) - 1):
                        current_pair = (word_tokens[i], word_tokens[i+1])
                        if current_pair == merge_pair:
                            # 找到最高优先级的合并对，记录位置并跳出查找
                            best_pair_in_word = current_pair
                            best_pair_index = i
                            break
                    if best_pair_in_word:
                        break # 跳出 merge_pair 循环
                
                if best_pair_in_word:
                    # 执行合并：将两个符号替换为一个新符号
                    merged = self.merges[best_pair_in_word]
                    i = best_pair_index
                    word_tokens = (word_tokens[:i] + 
                                   [merged] + 
                                   word_tokens[i+2:])
                else:
                    # 无法再合并，结束循环
                    break
            
            tokens.extend(word_tokens)
        return tokens
```



### 解释



- **分词的核心逻辑**：对于每个单词，它都从其字符序列开始，然后**贪婪地**、**迭代地**应用已学习到的合并规则。
- **合并优先级**：代码中通过 遍历合并规则。这保证了**最新**（即最长、频率最高）的合并规则会被优先考虑，这是 BPE 算法在分词阶段常用的策略。`reversed(list(self.merges.keys()))`
- **列表切片合并**： 是 Python 中高效地在列表中进行插入和删除作的简洁写法。它将符号对  替换为 。`word_tokens[:i] + [merged] + word_tokens[i+2:]``(i, i+1)``merged`

------



## 6. 编码和解码（ 和 ）`encode``decode`



这是最终将符号转换为 ID，以及从 ID 恢复文本的工具方法。



### 代码实现



python

```
    def encode(self, text):
        """编码文本为token IDs"""
        tokens = self.tokenize(text)
        # 将子词符号映射为 ID
        return [self.vocab.get(token, 0) for token in tokens]
    
    def decode(self, token_ids):
        """将token IDs解码回文本"""
        # 创建反向映射 {id: token}
        id_to_token = {id: token for token, id in self.vocab.items()}
        
        tokens = [id_to_token.get(token_id, '<?>') for token_id in token_ids]
        
        # 1. 拼接符号序列
        text = ''.join(tokens)
        # 2. 处理词尾标记，将其替换为空格，恢复单词结构
        text = text.replace('</w>', ' ')
        
        return text.strip()
```



### 解释



- **`self.vocab.get（token， 0）`**： 使用字典的 方法是安全的，如果 不在词汇表中（OOV），则返回默认值 。`get``token``0`
- **`{id： token 的 token， self.vocab.items（）} 中的 id： token 的 token， self.vocab.items（）} 中的 id`**：
- **解码**：关键在于将所有子词符号连接起来 （），然后将 BPE 特有的 标记替换为普通空格，完成单词边界的恢复。`''.join(tokens)``</w>`

---

## Python 模块详解`re`





### 1. 核心功能及在 BPE 中的应用



`re`模块主要提供以下功能：

| 功能         | 常用函数                        | BPE 中的应用                                                 |
| ------------ | ------------------------------- | ------------------------------------------------------------ |
| **模式定义** | `re.compile()`                  | 预编译正则表达式，提高在  循环中的替换效率。`train`          |
| **替换**     | `re.sub(pattern, repl, string)` | 将所有匹配到的符号对（如 ）替换成新符号（如 ）。`"t e"``"te"` |
| **转义**     | `re.escape(string)`             | 确保符号对中的字符即使是正则表达式的元字符（如  或 ）也能被字面匹配。`.``*` |
| **复杂匹配** | 先行/后顾断言                   | 确保替换操作只发生在由空格分隔的**完整**符号对上，避免替换到子字符串。 |

导出到工作表



### 2. BPE 代码中用到的关键 函数及概念`re`





#### 2.1.`re.compile(pattern)`



- **作用：** 将正则表达式的字符串模式预先编译成一个正则表达式对象。

- **优点：** 编译后的对象在多次使用时（例如在 BPE 的 循环中），执行速度比直接使用字符串模式要快得多。`train`

- **BPE 应用：**

  蟒

  ```
  pattern = re.compile(r'(?<!\S)' + re.escape(bigram) + r'(?!\S)')
  # pattern 对象随后用于 word.sub(replacement, word)
  ```



#### 2.2.`re.sub(pattern, repl, string)`



- **作用：** 在 中找到所有匹配 的子串，并将它们替换为 。`string``pattern``repl`

- **BPE 应用：** 这是执行符号合并的核心作。

  python

  ```
  new_word = pattern.sub(replacement, word)
  ```

  - `pattern`：预编译的符号对模式（例如匹配 ）。`"t e"`
  - `replacement`：新的合并符号（例如 ）。`"te"`
  - `word`：当前待合并的符号序列（例如 ）。`"t e s t </w>"`



#### 2.3.`re.escape(string)`



- **作用：** 对  中的所有特殊正则表达式字符（如 、、、 等）进行转义，让它们在模式中被视为普通字符。`string``.``+``(``*`
- **BPE 应用：**BPE 的符号可以是任何字符组合。如果一个符号对是 ，那么  就是 。如果不使用 ，正则引擎会将  视为匹配任何字符，导致匹配错误。会变成 ，确保只匹配字面意义的空格和点。`("a", ".")``bigram``"a ."``re.escape()``.``re.escape("a .")``"a\ \."`



#### 2.4. 先行断言和后顾断言（Lookarounds）



在 BPE 代码中，使用了以下复杂的正则结构，这是为了确保替换的**精确性**：

python

```
r'(?<!\S)' + re.escape(bigram) + r'(?!\S)'
```

| 正则部分      | 类型             | 含义                                                         |
| ------------- | ---------------- | ------------------------------------------------------------ |
| **`\S`**      | 元字符           | 匹配**任何非空白字符**（non-whitespace character）。         |
| **`(?<!\S)`** | **负向后顾断言** | 确保匹配的位置**前面**不是一个非空白字符。这意味着匹配的前面必须是**空白字符**或**字符串的开头**。 |
| **`(?!\S)`**  | **负向先行断言** | 确保匹配的位置**后面**不是一个非空白字符。这意味着匹配的后面必须是**空白字符**或**字符串的结尾**。 |

导出到工作表

**为什么 BPE 需要这个结构？** 

在 BPE 的 中，符号是用**空格**分隔的，例如：。`vocab``"t e s t </w>"`

1. 假设要合并 ， 是 。`("t", "e")``bigram``"t e"`
2. 如果不使用断言，直接用  可能会成功。`re.sub("t e", "te", "t e s t")`
3. 但如果符号是 ，而文本中存在 ，如果没有断言，替换可能会出错或不准确。`("st", "e")``"s t e"`
4. 这个结构强制要求 **必须是一个完整的符号对，**前后都被空格界定，从而保证了 BPE 算法的正确性：**只合并由空格分隔的、完整的符号。**`bigram`