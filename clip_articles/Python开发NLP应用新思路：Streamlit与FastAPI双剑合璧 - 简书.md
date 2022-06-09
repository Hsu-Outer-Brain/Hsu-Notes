# Python开发NLP应用新思路：Streamlit与FastAPI双剑合璧 - 简书
[![](https://upload.jianshu.io/users/upload_avatars/14270006/035d532e-68ab-4b86-9701-f9bae3b0b6cf.jpg?imageMogr2/auto-orient/strip|imageView2/1/w/96/h/96/format/webp)
](https://www.jianshu.com/u/7e4039d5a162)

0.5852020.03.28 17:59:41 字数 2,628 阅读 3,119

## Ⅰ. Hanlp

HanLP 是一系列模型与算法组成的 NLP 工具包，目前 HanLP 2.0 版本正处于 alpha 测试阶段。我们可以使用该工具包快速构建分词、词性标注、命名实体识别、依存句法分析、语义依存分析等功能。

Hanlp 2.0 是直接支持 Java 和 Python 接口的，这点与 Hanlp 1.x 版本不同。1.x 版本主要支持 java，使用 python 调用实际调用的是 pyhanlp 相关接口。具体项目详情可以登录相关项目主页：

[https://github.com/hankcs/HanLP](https://links.jianshu.com/go?to=https%3A%2F%2Fgithub.com%2Fhankcs%2FHanLP)

由于 Hanlp 2.0 基于 TensorFlow 2.1，因此需要先安装最新版 TensorFlow：  
**使用虚拟环境**

    python3 -m venv .venv
    source .venv/bin/activate 

    pip install tensorflow==2.1 

我用的是云服务器，没有安装 GPU 版本，直接安装会安装 TensorFlow 1.x 的版本，可以使用以下命令升级：

    pip install tensorflow --upgrade 

完成后可以测试一下是否安装成功：

    import tensorflow as tf 
    print(tf.__version__) 

显示了正确的版本号，就可以继续安装 Hanlp：

Hanlp 会自动根据已经安装的 tensorflow 版本进行安装，安装完成后同样测试一下：

    import hanlp 
    print(hanlp.__version__) 

打印 `hanlp.pretrained.ALL` 可以列出 HanLP 中的所有预训练模型，一共有 41 个：

     print(hanlp.pretrained.ALL) 

     {'CHNSENTICORP_BERT_BASE_ZH': 'https://file.hankcs.com/hanlp/classification/chnsenticorp_bert_base_20200104_164655.zip',
    ...
      'TENCENT_AI_LAB_EMBEDDING': 'https://ai.tencent.com/ailab/nlp/data/Tencent_AILab_ChineseEmbedding.tar.gz#Tencent_AILab_ChineseEmbedding.txt'} 

这里只列出本文中要用到的四个模型：

| 模型名称                               | 类型       |
| ---------------------------------- | -------- |
| PKU_NAME_MERGED_SIX_MONTHS_CONVSEG | 中文分词     |
| rules.tokenize_english             | 英文分词     |
| MSRA_NER_BERT_BASE_ZH              | 中文命名实体识别 |
| CTB7_BIAFFINE_DEP_ZH               | 中文依存句法分析 |

以下逐一说明：

### 1. 中文分词

引入模型：

     import hanlp 
    tokenizer = hanlp.load('PKU_NAME_MERGED_SIX_MONTHS_CONVSEG') 

单句分词：

批量并行分词：

    tokenizer(['萨哈夫说，伊拉克将同联合国销毁伊拉克大规模杀伤性武器特别委员会继续保持合作。',              
     '上海华安工业（集团）公司董事长谭旭光和秘书张晚霞来到美国纽约现代艺术博物馆参观。',               
    'HanLP支援臺灣正體、香港繁體，具有新詞辨識能力的中文斷詞系統']) 

    [['萨哈夫', '说', '，', '伊拉克', '将', '同', '联合国', '销毁', '伊拉克', '大', '规模', '杀伤性', '武器', '特别', '委员会', '继续', '保持', '合作', '。'], 
     ['上海', '华安', '工业', '（', '集团', '）', '公司', '董事长', '谭旭光', '和', '秘书', '张晚霞', '来到', '美国', '纽约', '现代', '艺术', '博物馆', '参观', '。'], 
     ['HanLP', '支援', '臺灣', '正體', '、', '香港', '繁體', '，', '具有', '新詞', '辨識', '能力', '的', '中文', '斷詞', '系統']] 

### 2. 英文分词

英文本身是不需要分词的，因此这里直接使用基于规则的普通函数：

    tokenizer = hanlp.utils.rules.tokenize_english
    tokenizer("Don't go gentle into that good night.") 

    ['Do', "n't", 'go', 'gentle', 'into', 'that', 'good', 'night', '.'] 

### 3. 中文命名实体识别

中文命名实体识别是字符级模型，输入使用 `list`将字符串转换为字符列表。输出格式为 `(entity, type, begin, end)`。

    recognizer = hanlp.load(hanlp.pretrained.ner.MSRA_NER_BERT_BASE_ZH)
    recognizer([list('上海华安工业（集团）公司董事长谭旭光和秘书张晚霞来到美国纽约现代艺术博物馆参观。'),            
    list('萨哈夫说，伊拉克将同联合国销毁伊拉克大规模杀伤性武器特别委员会继续保持合作。')]) 

    [[('上海华安工业（集团）公司', 'NT', 0, 12),
      ('谭旭光', 'NR', 15, 18),
      ('张晚霞', 'NR', 21, 24),
      ('美国', 'NS', 26, 28),
      ('纽约现代艺术博物馆', 'NS', 28, 37)],
     [('萨哈夫', 'NR', 0, 3),
      ('伊拉克', 'NS', 5, 8),
      ('联合国销毁伊拉克大规模杀伤性武器特别委员会', 'NT', 10, 31)]] 

### 4. 中文依存句法分析

句法分析器的输入是单词列表及词性列表，输出是 CoNLL-X 格式 `[^conllx]` 的句法树。

    syntactic_parser = hanlp.load(hanlp.pretrained.dep.CTB7_BIAFFINE_DEP_ZH)
    print(syntactic_parser([('蜡烛', 'NN'), ('两', 'CD'), ('头', 'NN'), ('烧', 'VV')])) 

    1  蜡烛  _  NN  _  _  4  nsubj  _  _
    2  两  _  CD  _  _  3  nummod  _  _
    3  头  _  NN  _  _  4  dep  _  _
    4  烧  _  VV  _  _  0  root  _  _ 

由于预训练模型都较大，需要预留好足够的存储空间。

个人经验是最好在云服务器上下载，如果是下载到本地最好提前下载，并且晚上下载速度会快一些。

下面介绍 FastAPI 以及对 NLP 功能的接口封装。

## Ⅱ. FastAPI

2005 年就有人说过：“Python 是一门 web 框架比关键字还多的语言 “。近几年主要流行的 Python Web 框架是 Django 和 Flask。不过 Django 较为繁重，Flask 常常被人诟病其异步性能。

自从 Python3.5 中正式将协程做为底层技术引入之后，关于如何构建标准异步 web 框架的讨论就源源不绝。之后便诞生了一些基于 ASGI 的 web 框架，如：API Star、Uvicorn、Starlette、FastAPI 等等。

其中 FastAPI 是个比较有趣的项目：它并非一个从零开始的项目，而是基于 Starlette 和 Pydantic，其中 Starlette 又是基于 Uvicorn 的。

这里我们可以看一下 FastAPI 的源码，在 dependencies 文件夹下的 `utils.py` 引用部分：

     from fastapi.dependencies.models import Dependant, SecurityRequirement
    from fastapi.security.base import SecurityBase
    from fastapi.security.oauth2 import OAuth2, SecurityScopes
    from fastapi.security.open_id_connect_url import OpenIdConnect
    from fastapi.utils import PYDANTIC_1, get_field_info, get_path_param_names
    from pydantic import BaseConfig, BaseModel, create_model
    from pydantic.error_wrappers import ErrorWrapper
    from pydantic.errors import MissingError
    from pydantic.utils import lenient_issubclass
    from starlette.background import BackgroundTasks
    from starlette.concurrency import run_in_threadpool
    from starlette.datastructures import FormData, Headers, QueryParams, UploadFile
    from starlette.requests import Request
    from starlette.responses import Response
    from starlette.websockets import WebSocket 

因此可以将 FastAPI 视做对 Starlette 的高级封装，并引入了 Pydantic 来对数据进行数据验证和设置管理。

单从性能上来说应该是 [Uvicorn > Starlette > FastAPI](https://links.jianshu.com/go?to=https%3A%2F%2Fwww.techempower.com%2Fbenchmarks%2F%23section%3Dtest%26runid%3D7464e520-0dc2-473d-bd34-dbdfd7e85911%26hw%3Dph%26test%3Dquery%26l%3Dzijzen-7)：  

![](https://upload-images.jianshu.io/upload_images/14270006-7663696c0df8a21c.png)

![](https://upload-images.jianshu.io/upload_images/14270006-f0bbffb3ba888ad8.png)

项目官网列出了 FastAPI 的主要优点：

-   **性能快**：高性能，可以和**NodeJS**和**Go**相提并论；
-   **快速开发**：开发功能速度提高约 200% 至 300%；
-   **更少的 Bug**：减少 40% 开发人员容易引发的错误；
-   **直观**：完美的编辑支持，补全功能缩减了 debugging 的时间；
-   **简单**: 易于使用和学习，减少阅读文档的时间；
-   **代码简洁**：很大程度上减少代码重复。每个参数可以声明多个功能，减少 bug 的发生；
-   **标准化**：基于并完全兼容 API 的开发标准：`OpenAPI`（以前称为 `Swagger`）和 `JSON Schema`。

以下简单介绍以下 FastAPI 的使用。

### 1. 安装

直接使用 `pip`安装：

如果用于生产，那么你还需要一个 ASGI 服务器，如 Uvicorn 或 Hypercorn：

### 2. 创建 FastAPI 项目

创建 main.py 文件

     from fastapi import FastAPI


    app = FastAPI()

    @app.get("/")
    def first_fuc():
        return {"Hello": "World"}

    @app.get("/any/{item_id}")
    def second_fuc(item_id: int, q: str = None):
        return {"item_id": item_id, "q": q} 

> 如果代码需要用到异步 async/await，使用 async def，如下所示：

     from fastapi import FastAPI

    app = FastAPI()

    @app.get("/")
    async def first_fuc():
        return {"Hello": "World"}

    @app.get("/any/{item_id}")
    async def second_fuc(item_id: int, q: str = None):
        return {"item_id": item_id, "q": q} 

### 3. 运行项目

运行服务器：

    uvicorn main:app --reload 

> 命令 `uvicorn main:app --reload` 指的是：
>
> -   main：main.py 文件
> -   app：app = FastAPI() 在 main.py 内创建的对象。
> -   \--reload：在代码更改后重新启动服务器。只有在开发时才使用这个参数。

在 Terminal 中启动成功后便会显示如下：

![](https://upload-images.jianshu.io/upload_images/14270006-619d8ec3801b76e9.png)

### 4. 检查项目

在浏览器中打开网址：[http://127.0.0.1:8000](http://127.0.0.1:8000)

可以看见 json 格式的响应数据：

![](https://upload-images.jianshu.io/upload_images/14270006-a6773c45251fcd98.png)

![](https://upload-images.jianshu.io/upload_images/14270006-02613fc560f00bef.png)

> **这样便创建了一个 API：** 
>
> -   url：/ 和 / any / {item_id}，两个 url 都可以接收 HTTP 请求。
> -   / 和 / any / {item_id} 都采用 GET 方式的 HTTP 请求方法
> -   / any / {item_id} 包含路径参数 item_id，格式为 int
> -   / any / {item_id} 还包含一个可选的参数 q，格式为 str

### 5. 交互的 API 文档

现在进入 [http://127.0.0.1:8000/docs](https://links.jianshu.com/go?to=http%3A%2F%2F127.0.0.1%3A8000%2Fdocs)

便会得到自动的交互式 API 文档，该文档由 Swagger UI 提供。

![](https://upload-images.jianshu.io/upload_images/14270006-ca1ea0953de60c46.png)

**6. 备用 API 文档**

现在，转到 [http://127.0.0.1:8000/redoc](https://links.jianshu.com/go?to=http%3A%2F%2F127.0.0.1%3A8000%2Fredoc)

这里是备用自动文档（由 ReDoc 提供）。

![](https://upload-images.jianshu.io/upload_images/14270006-d61add26a8fa9fdc.png)

### 7. 下载 API 接口文档

如果需要提供 API 接口，那么只需要一行命令，即可下载 api 文件，一般保存为 api.json

    curl -o api.json http://127.0.0.1:8000/openapi.json 

### 8. 对 hanlp 进行接口封装

我们需要调用 Hanl p 的四个模型功能中，两个模型的输入是字符串，两个模型的输入是列表，此处统一为列表。

设计接口的输入为以下格式的 JSON 字符串：

    {
        "model_name": str, 
       "input": list,
    } 

接口的返回输出为：

    {
        "success": bool，
        "rlt": list
    } 

下面直接贴出全部的后端代码：

     from fastapi import FastAPI
    from pydantic import BaseModel
    import hanlp


    app = FastAPI()

    tokenizer_zh = hanlp.load('PKU_NAME_MERGED_SIX_MONTHS_CONVSEG')
    tokenizer_en = hanlp.utils.rules.tokenize_english
    recognizer = hanlp.load(hanlp.pretrained.ner.MSRA_NER_BERT_BASE_ZH)
    syntactic_parser = hanlp.load(hanlp.pretrained.dep.CTB7_BIAFFINE_DEP_ZH)

    class Data(BaseModel):
        model_name: str
        input: list

    @app.post('/tok_zh')
    def split_cn(data_zh: Data):
        msg = data_zh.input
        rlt = [tokenizer_zh(x) for x in msg]
        return {'success': True, 'rlt': rlt}

    @app.post('/tok_en')
    def split_en(data_en: Data):
        msg = data_en.input
        rlt = [tokenizer_en(x) for x in msg]
        return {'success': True, 'rlt': rlt}

    @app.post('/ner')
    def ner(data_zh: Data):
        msg = data_zh.input
        rlt = [recognizer(x) for x in msg]
        return {'success': True, 'rlt': rlt}

    @app.post('/parser')
    def parser(data_zh: Data):
        msg = data_zh.input
        rlt = syntactic_parser(msg)
        return {'success': True, 'rlt': rlt} 

如果希望程序支持异步执行，只需要将以上代码中 `def` 修改为 `async def` 即可。

### 9. 在服务器上启动项目

由于使用了云服务器，所以在运行时要设置 ip 以及端口号：

    uvicorn main:app --reload --host 0.0.0.0 --port  80 

其中 `--host` 是设置 ip 地址，`0.0.0.0`意思就是使用本机的公网 ip， 然后 `--port:80` 是指将端口号设置为 80。

根据云服务器的安全策略，使用其他端口需要进入控制台，开放相应端口的外网访问权限。

更多的设置可以通过 `uvicorn --help` 进行查询。项目启动后如果显示以下内容则表示启动成功。

![](https://upload-images.jianshu.io/upload_images/14270006-ebb66a159a4cef15.png)

然后在浏览器中输入 `ip 地址 / docs` 进入到如下接口页面便可以对接口进行调试：

![](https://upload-images.jianshu.io/upload_images/14270006-8e396e833aa7d06e.png)

从图中可以看出一共有四个接口，都是 `POST` 类型的，后面有相对路径和函数名。具体测试方法与上文相同，不再赘述。

## Ⅲ. Streamlit

要说 2019 年开源社区的宝藏项目，Streamlit 绝对排的上号。官方宣称它是：**The fastest way to build custom ML tools**。

具体有多块呢？可见下图展示的例子，使用两百多行代码就制作了一个用于自动驾驶的车辆检测应用：

![](https://upload-images.jianshu.io/upload_images/14270006-1178550a8ca109d8.png)

当然 Streamlit 相对于传统前端还是有很多的问题，比如：控件数量较少；可修改的自由度有限；没有传统的递归，每次请求都相当于重新运行整个脚本或者要使用 `cache` 改进性能。

不过将其作为快速应用的开发，以及机器学习工具的配置式前端，都是十分不错的。毕竟相比花费一、两周制作一个完美的页面，一个小时就能展示模型能力更加吸引 AI 开发者。

同样直接贴出使用 Streamlit 搭建前端的代码：

     import streamlit as st
    import requests

    def send2back(data_bin):
        rlt = requests.post('http://111.229.217.153:80/ner', json=data_bin).json()

    st.title("自然语言处理APP")
    html_tmp = """
        <div style="background-color:tomato;padding:10px">
        <h2 style="color:white;text-align:center;">基于Hanlp与FastAPI制作</h2>
        </div>
        """
    st.markdown(html_tmp, unsafe_allow_html=True)

    st.markdown('---')

    option = st.selectbox('请选择你想要使用的功能：',
                          ('', '中文分词', '英文分词', '中文命名实体识别', '中文依存句法分析'))
    content = st.text_input('请输入待分析的内容：')

    if option == '中文分词':
        if st.button("中文分词") & (content != ''):
            data_bin = {'model_name': 'tok_zh', 'input': [content]}
            rlt = requests.post('http://111.229.217.153:80/tok_zh', json=data_bin).json()
            st.text(rlt['rlt'][0])
    elif option == '英文分词':
        if st.button("英文分词") & (content != ''):
            data_bin = {'model_name': 'tok_en', 'input': [content]}
            rlt = requests.post('http://111.229.217.153:80/tok_en', json=data_bin).json()
            st.text(rlt['rlt'][0])
    elif option == '中文命名实体识别':
        if st.button("命名实体识别") & (content != ''):
            data_bin = {'model_name': 'ner', 'input': [list(content)]}
            rlt = requests.post('http://111.229.217.153:80/ner', json=data_bin).json()
            st.write(rlt)
    elif option == '中文依存句法分析':
        if st.button("依存句法分析") & (content != ''):
            data_bin = {'model_name': 'parser', 'input': [content]}
            rlt = requests.post('http://111.229.217.153:80/parser', json=data_bin).json()
            st.write(rlt)
    else:
        pass 

调用后台接口的是 Python 的 HTTP 库 `requests`。如果后台使用了 `async def`异步函数，那么建议将 `requests` 替换为同为异步的 `aiohttp`。

将以上代码保存为文件 `front.py`，在终端中运行：

运行成功后终端便会显示：

![](https://upload-images.jianshu.io/upload_images/14270006-2726409b5495af1c.png)

因为我是在本地电脑上运行，默认端口是 8501，如果在服务器上运行，需要配置 ip 地址和端口号。详见官方文档：[https://docs.streamlit.io/](https://links.jianshu.com/go?to=https%3A%2F%2Fdocs.streamlit.io%2F)

在浏览器中输入：[http://localhost:8501/](https://links.jianshu.com/go?to=http%3A%2F%2Flocalhost%3A8501%2F)

便可看到我们 NLP 应用的界面：

![](https://upload-images.jianshu.io/upload_images/14270006-4abbea886d416c48.png)

然后我们测试一下：

![](https://upload-images.jianshu.io/upload_images/14270006-dae964064b9f7a6b.png)

Streamlit 的另外一个优点是：开发出来的应用可以基于设备自适应，因此在手机端浏览也会有较好的体验效果。有兴趣的小伙伴可以自行尝试。

## Ⅳ. 进阶

以上简单介绍了使用 FastAPI 以及 Streamlit 开发一款基于 Hanlp 的自然语言处理应用的方法。

完整的开发流程还应包括反向代理设置，以及使用容器工具打包。这些在本文中不是重点，就不一一展开了。

本文提到的三种框架中：

-   Hanlp 基于 Tensorflow；
-   FastAPI 基于 Scarlett 和 P ydantic；
-   Streamlit 依赖于 tornado。

这揭示了一种 APP 开发的趋势：**使用更高级的框架，尽可能地忽略掉底层的细节，以此达到快速开发和迭代的目的**。

不过这并不意味着以后开发人员就不用弄懂 web 的各种细节。

相反，**只有明白底层细节才能快速上手高级框架，并且不至于跌入高度抽象的陷阱当中**。想要去个性化地定制系统或者提高性能，就必须深入了解其底层架构。

这里只是对这种思路的一种尝试，后面我会继续探索将更多新的框架纳入生产中的 pipeline。比方说 AllenNLp、FastAI 以及流计算框架 Flink 等等。

### [学习来源](https://links.jianshu.com/go?to=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzIyMDkxMjAyOA%3D%3D%26mid%3D2247483710%26idx%3D1%26sn%3Dfdc4686c18507b81b34a5313635f9ad3%26chksm%3D97c581cda0b208db926a53c01cbd90eb4782161004e13144f88ba3a461593bcc0509165c9896%26mpshare%3D1%26scene%3D1%26srcid%3D%26sharer_sharetime%3D1583212366178%26sharer_shareid%3D438cc9281d104554c9b24b4acfc9de05%26key%3D895ceea09f03347446b7d9c833895aca3615ab56032c91e924de30cef830f3e3c57bdd613ae39888ce7ab577f1aeffc3be8862961918c6aac59f9560dbe291234c2592e07e61c312c98c6dd78ae2f979%26ascene%3D1%26uin%3DMjUxNDAwODcwNA%253D%253D%26devicetype%3DWindows%2B10%26version%3D62080079%26lang%3Dzh_CN%26exportkey%3DAQ3y3JVElIFNJtIQYKMK9TY%253D%26pass_ticket%3DCZQD57nZo%252Ffjdc%252FupyeVPA7ppi6qYrSxriiSQsHg8r7LSZR3cAXriimhNeXWGelJ)

更多精彩内容，就在简书 APP

"小礼物走一走，来简书关注我"

还没有人赞赏，支持一下

[![](https://upload.jianshu.io/users/upload_avatars/14270006/035d532e-68ab-4b86-9701-f9bae3b0b6cf.jpg?imageMogr2/auto-orient/strip|imageView2/1/w/100/h/100/format/webp)
](https://www.jianshu.com/u/7e4039d5a162)

[羋虹光](https://www.jianshu.com/u/7e4039d5a162 "羋虹光")苦学僧<br><br>文章记录自己的学习过程，如果涉及到版权问题请告知。

总资产 102 共写了 60.0W 字获得 1,252 个赞共 730 个粉丝

### 推荐阅读[更多精彩内容](https://www.jianshu.com/)

-   常用概念： 自然语言处理（NLP） 数据挖掘 推荐算法 用户画像 知识图谱 信息检索 文本分类 常用技术： 词级别...

    [![](https://cdn2.jianshu.io/assets/default_avatar/7-0993d41a595d6ab6ef17b19496eb2f21.jpg)
    御风之星](https://www.jianshu.com/u/e6ae6d978f3d)阅读 7,773 评论 1 赞 24

-   前言 从本文开始，我们进入实战部分。首先，我们按照中文自然语言处理流程的第一步获取语料，然后重点进行中文分词的学习...

    [![](https://upload-images.jianshu.io/upload_images/5214391-82a8abc7875fa82c.png?imageMogr2/auto-orient/strip|imageView2/1/w/300/h/240/format/webp)
    ](https://www.jianshu.com/p/c18b937e1cf1)

-   古时有侠客， 胸中装着正义， 一剑刺穿喉咙， 一剑割下头颅， 电光火石闪耀， 划出夺目的弧。 江河涌流， 黄沙掀土...

    [![](https://upload.jianshu.io/users/upload_avatars/4907682/da6a3265-ded5-4a4b-b340-c72ea988fac2.jpg?imageMogr2/auto-orient/strip|imageView2/1/w/48/h/48/format/webp)
    山河一梦](https://www.jianshu.com/u/43a7c5680407)阅读 331 评论 11 赞 14

-   Dear Ryan dear, 大约是在 8 月底的某一天。 今天麻麻用婴儿车推着你，走在黄昏的小区里面。即将下线的落... 
    [https://www.jianshu.com/p/ef4802c6edc5](https://www.jianshu.com/p/ef4802c6edc5)
