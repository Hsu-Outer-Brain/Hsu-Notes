# 如何快速构建一个核酸点服务状态查询Web应用？Python实例详解
什么是设计模式？

设计模式实软件中常见问题的典型解决方案。能根据需求进行预制蓝图，可用于解决代码中反复出现的设计问题。高质量应用程序框架设计过程广泛使用设计模式来确保代码可复用和可扩展性。

设计模式有什么用？  

设计模式并非必须使用，我的项目没使用或不注重设计模式的使用也照样运行，的确，项目中不使用任何设计模式并不会影响项目的运行，但项目后期需求变动涉及二次开发时，在全新的上下文中工作，代码的维护和修改的复杂度着实让人头大。

项目中单例模式和工厂模式应该是用的最多的，单例模式应用以前的文章已经分享过。本次分享主要介绍一下简单工厂模式应用实例。

简单工厂模式案例

目录：

1、查询应用效果图

2、简单工厂模式案例

1）项目结构目录  

2）服务服务端 -（Map_Load.py 地图加载显示模式）

3）应用客户端 - Map_server_client.py

4）Web 应用发布 - Map_app.py

**![](https://mmbiz.qpic.cn/sz_mmbiz_jpg/okAIo1O8h7xH864VcuIFQGHG9pPw81HyjlBZ44aW82fIyEzHPzQqdyLM1l9FQo9PMB6n12CP6Pib6Me7VpfA25g/640?wx_fmt=jpeg)\*\***查询应用效果图 \*\*

![](https://mmbiz.qpic.cn/sz_mmbiz_png/okAIo1O8h7wCUNZRhKIGS8sVSibl03zwjpqyWlNc1FSgp1WtyD62wb0libFiaKXr7JPQWR9asf1HulzG5AZicRga7A/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/sz_mmbiz_gif/okAIo1O8h7wCUNZRhKIGS8sVSibl03zwjV1lvzNibW5nykOxlDf1vp0F1TZqquYCib5UnYahK7cYmWy7HXMTjkjzA/640?wx_fmt=gif)

←核酸采样点服务状态查询→

\***\*![](https://mmbiz.qpic.cn/sz_mmbiz_jpg/okAIo1O8h7xH864VcuIFQGHG9pPw81HyjlBZ44aW82fIyEzHPzQqdyLM1l9FQo9PMB6n12CP6Pib6Me7VpfA25g/640?wx_fmt=jpeg)\*\*\*\***简单工厂模式案例 \*\*

**本次分享结合深圳市便民核酸点服务状态查询服务数据示例。** 

**1、项目结构目录**

**假如有这样一个需求，做一个可切换地图加载模式的可视化项目**

![](https://mmbiz.qpic.cn/sz_mmbiz_png/okAIo1O8h7wCUNZRhKIGS8sVSibl03zwje15KW0N8tQiaXkrjFtDUWorzPgFdicpJvGuh8ViaK1ShyQQic2Xc3KeKvg/640?wx_fmt=png)

←程序项目结构→

项目主要由以上几个文件夹组成：

1、.venv-- 虚拟环境；.streamlit-- 网页主题设置

2、apps-- 应用集成文件夹（Mapview 文件夹 -- 应用服务端，logs 文件夹 -- 日志文件，Map_server_client-- 应用客户端；tools 文件夹 -- 其他脚本工具）；

3、resources-- 资源文件夹

4、Map_app.py--web 应用主文件  

**2、应用服务端 -（**地图加载显示模式 - Map_Load.py**）**

**背景：使用地址文件中的经纬度在地图上标记显示，并显示给定地点信息，提供两种加载模式供选择：** Full_Load（地图一次性全加载）和 Zoom_Loading（地图缩放加载）。

\`import pandas as pd  
import folium  
from folium import plugins

pd.set_option("expand_frame_repr", False)

from abc import  ABCMeta,abstractmethod

class BaseAlgorithm(metaclass=ABCMeta):  
    """  
    地图初始化基类  
    """  
    def **init**(self):  
        # 加载地图模式  
        self.Map = folium.Map(location=[22.540477, 114.061226],  
                         tiles='[https://wprd01.is.autonavi.com/appmaptile?x={x}&y={y}&z={z}&lang=zh_cn&size=1&scl=1&style=7'](https://wprd01.is.autonavi.com/appmaptile?x={x}&y={y}&z={z}&lang=zh_cn&size=1&scl=1&style=7'),  
                         attr='高德 - 常规图',  
                         zoom_start=10,  
                         control_scale=True,  
                         width='100%',)  
    @abstractmethod  
    def execute(self):  
        pass

class Full_Load(BaseAlgorithm):  
    """  
   地图一次全加载  
    """  
    def **init**(self,data):  
        # 获取数据源  
        super(Full_Load, self).**init**()  
        # 数据中心  
        self.data = data

    def execute(self):  
        import base64  
        encoded = base64.b64encode(open(r'.\\resources\\qrcode_logo.JPG', 'rb').read()).decode()

        \# data 映射一列数据【color】--- 设置数据过滤层  
        for name, row in self.data.iterrows():  
            html = '<h2> 欢迎关注 Python 数据分析实例</h2><img src="data:image/jpeg;base64,{0}"><br><h3>采样点名称：{1}</h3>' \\  
                   '<h3>当前等待时间：{2}</h3><h3>服务时间：{3}</h3><h3>服务人群：{4}</h3>' \\  
                   '<a href = https://szwj.borycloud.com/wh5/index.html#/>查询服务➡</a>'.format(encoded,row['service_name'],str(row["delay"]),row['service_time'],row['services'])  
            iframe = folium.IFrame(html, width=380, height=560)  
            folium.Marker(  
                \[row["LATITUDE"], row["LONGITUDE"]],  
                # max_width 设置每行显示字符数  
                popup=folium.Popup(iframe,  
                                        max_width='100%'),   
                tooltip=row["service_name"],  
                icon=folium.Icon(icon='info-sign', color=row["color"])).add_to(self.Map)

        \# self.Map.save(r'.\\resources\\folium_map_full.html')  
        print("地图成功生成！")  
        return self.Map

class Zoom_Loading(BaseAlgorithm):  
    """  
    地图缩放加载  
    """  
    def **init**(self,data):  
        super(Zoom_Loading, self).**init**()  
        # 获取数据源  
        self.data = data

    def execute(self):  
        import base64  
        encoded = base64.b64encode(open(r'.\\resources\\qrcode_logo.JPG', 'rb').read()).decode()

        \# 调用 Marker 可以创建标记，传入位置和信息，当鼠标放在标记上会显示出信息。  
        folium.Marker([22.540477,114.061226],  
                      popup=folium.Popup("深圳市", max_width=100)).add_to(self.Map)   # 创建中心标记

        marker_cluster = plugins.MarkerCluster().add_to(self.Map)

        for name, row in self.data.iterrows():  
            html = '<h2> 欢迎关注 Python 数据分析实例</h2><img src="data:image/jpeg;base64,{0}"><br><h3>采样点名称：{1}</h3>' \\  
                   '<h3>当前等待时间：{2}</h3><h3>服务时间：{3}</h3><h3>服务人群：{4}</h3>' \\  
                   '<a href = https://szwj.borycloud.com/wh5/index.html#/>查询服务➡</a>'.format(encoded,row['service_name'],str(row["delay"]),row['service_time'],row['services'])  
            iframe = folium.IFrame(html, width=380, height=560)

            folium.Marker(\[row["LATITUDE"], row["LONGITUDE"]],  
                          popup=folium.Popup(iframe, max_width='100%'),  
                          tooltip=row["service_name"],  
                          icon=folium.Icon(icon='info-sign', color='lightblue')).add_to(marker_cluster)  
        # self.Map.save(r'.\\resources\\folium_map_zoom.html')  
        print("地图成功生成")  
        return self.Map

class ForestFactory(object):

    def make_execute(self,object_type,data):  
        return eval(object_type)(data).execute()

\`

备注：

1、简单工厂模式

简单工厂模式是创建型模式，顾名思义简单工厂模式则是选择创建出需要的对象。实质是由一个工厂类根据传入的参数，动态决定应该创建并且返回哪一个产品类（这些产品类继承自一个父类或接口）的实例。

本文应用均继承初始化 BaseAlgorithm 类方法，选择加载底图模式。这里创建了 Full_Load 和 Zoom_Loading 两种地图显示产品，当然可以根据业务需求拓展产品类。工厂类 ForestFactory 实例化来决定创建哪个产品类，在创建对象上的灵活性高。

如果一个类已经完成开发、测试和审核工作，属于某个框架或可被其他类使用的话，对其代码进行修改是有风险的。创建子类并重写原始类的部分内容以完成不同的行为。

用了设计模式给我的感受就是代码变得优雅简洁、逻辑清晰明了，再注入高大上的算法模型之后，你的 service 层代码看上去很 nice。以上代码可进一步优化，对方法层面的封装，封装变化的内容。

2、Folium 简介

作为 Python 的一个可视化工具包 Folium，它通过 Leaflet 的地图服务，可以在 Jupyter Notebook 上实现可视化的地理位置作图，制作各种各样精美的地图信息。它不仅可以针对某个经纬度进行地理位置的可视化操作，还能够根据实时的人群地理位置信息来构建静态与动态热力图，甚至还能够针对经纬度的数量来进行必要的聚类可视化。

\`  
#生成地图  
# 1. 初始化一个 map 对象  

# zoom_start：地图 zoom 的初始级别，默认为 10。假设改成 11 的话，那么就相当于在默认创建的地图的级别上放大一级。

\# 2. 底图更换  

# tiles：str 型，用于控制绘图调用的地图样式，默认为'OpenStreetMap'，也有一些其他的内建地图样式，

\# 如'Stamen Terrain'、'Stamen Toner'、'Mapbox Bright'、'Mapbox Control Room' 等；  
''' lyrs 可以设置为不同的参数，分别代表不同形式的地图，可以尝试 1-7  
        h = roads only  
        m = standard roadmap  
        p = terrain  
        r = somehow altered roadmap  
        s = satellite only  
        t = terrain only  
        y = hybrid  
'''\`

3、Icon 标记颜色  

`Icon should be one of: {'blue', 'beige', 'lightblue', 'darkred', 'lightgray', 'lightgreen', 'lightred', 'white', 'purple', 'green', 'darkgreen', 'pink', 'darkpurple', 'red', 'gray', 'darkblue', 'orange', 'black', 'cadetblue'}.`

4、简单工厂模式和策略模式区别

我觉得简单工厂模式和策略模式很相似。从以上代码可以看出，工厂模式主要是返回的接口实现类的实例化对象，最后返回的结果是接口实现类中的方法，而策略模式是在实例化策略模式的时候已经创建好了，我们可以在策略模式中随意的拼接重写方法，简单来说，工厂模式只关注最后的结果，而策略模式注重的是过程。后面结合实例写一篇策略模式相关推文进一步讨论。

**3、应用客户端 - Map_server_client.py**

\`  
from apps.tools.Map_Load import ForestFactory

class webservice_client():  
    def **init**(self,BaseAlgorithm,data):  
        # super(webservice_client, self).**init**()  
        # 定义客户端对象  
        self.BaseAlgorithm = BaseAlgorithm  
        self.data = data

    @fail_data(msg='地图加载失败')  
    def get_Map_model(self):  
        f = ForestFactory()  
        Map = f.make_execute(self.BaseAlgorithm,self.data)

        return Map

\`

备注：这里给客户端实例化 get_Map_model 方法添加带参数装饰器，@fail_data（msg='地图加载失败'）添加接口调用失败处理机制。下一篇文章补充。  

**4、web 应用发布 - Map_app.py**  

\`  
import streamlit as st  
import os, sys  
from apps.Map_server_client import webservice_client  
import pandas as pd  
from streamlit_option_menu import option_menu

class Mapview_App():

    def **init**(self, title='', \*\*kwargs):  
        self.**dict**.update(kwargs)  
        self.title = title  
        self.data_sources = pd.read_excel(r".\\apps\\Mapview\\data1.xlsx",index_col=False)  
        # 设置网页标题，以及使用宽屏模式  
        st.set_page_config(  
            page_title="Map_App",  
            page_icon="🧊",   
            layout="wide",  
            initial_sidebar_state="auto"  # 侧边栏  
        )

    def run(self):  
        try:  
            sidebar = self.\_cs_sidebar()  
            self.\_cs_body(sidebar)  
        except Exception as e:  
            import streamlit as st  
            st.image(os.path.join(".", "resources", "failure.png"), width=100, )  
            st.error('An error has occurred,try again.')  
            st.error('Error details: {}'.format(e))

    def \_cs_sidebar(self):  
        from apps.tools.convert_cir_image import circle  
        import streamlit.components.v1 as components  
        import streamlit as st

        components.html("""<marquee bgcolor="#670796" behavior="alternate"><font color='white'>欢迎进入主页面</font></marquee>""",  
                        height=60)

        c1, c2, c3, c4, \_ = st.sidebar.columns([3, 2, 4, 2, 2])  
        # 输入 logo 的基础图片路径  
        logo_path = r".\\resources\\logo.jpg"  
        cir = circle(logo_path)

        c2.image(image=cir, width=120, caption='Python 数据分析实例')

        with st.sidebar:  
            """  
            icons# bootstrap 网站找到 [https://icons.bootcss.com/](https://icons.bootcss.com/)  
            """  
            sidebar = option_menu("导航栏",  
                                  ["首页", "标记数据源", "地图可视化",'关于'],  
                                  icons=['house','cloud-upload', 'geo-alt', 'brightness-high', 'bar-chart',  
                                         'bell', 'info-circle'],  
                                  menu_icon="broadcast", default_index=0)

        st.sidebar.text("欢迎关注 Python 数据分析实例")

        \# 隐藏右边的菜单以及页脚  
        hide_streamlit_style = """  
        <style>  
        #MainMenu {visibility: hidden;}  
        footer {visibility: hidden;}  
        </style>  
        """  
        st.markdown(hide_streamlit_style, unsafe_allow_html=True)

        return sidebar

    def \_cs_body(self,sidebar):

        import streamlit as st  
        if sidebar == "标记数据源":  
            # 侧边栏  
            st.sidebar.subheader("点击数据下载")  
            st.subheader("深圳市便民核酸采样点服务状态查询")  
            Delay_time = st.number_input("请输入选择排队时间", step=1, max_value=60,value=30)

            @st.cache  
            def load_data(data):  
                info_list = \[]  
                for i, row in data.iterrows():  
                    info = {"code": row['code'], "service_name": row['service_name'],  
                            "delay": row['delay'],"service_time":row['service_time'],'services':row['services']}  
                    info_list.append(info)  
                info_list.sort(key=lambda info: info['delay'], reverse=True)  
                # 数据上传发送层  
                DATA = pd.DataFrame(info_list)

                DATA.columns = ['采样点编号', '采样点名称', '等待时间','服务时间','服务人群']  
                return DATA

            def filter_data(Delay_time,data_sources):  
                """  
                # 构建 Web 表格  
                """  
                # 加载数据表  
                data = load_data(data_sources)  
                # 等待时间过滤  
                data = data\[data["等待时间"] &lt;float(Delay_time)]

                return data

            data = filter_data(Delay_time,self.data_sources)

            st.write('符合条件的采样点数量', data.shape[0])  
            # 交互式表格  
            # st.dataframe(data)  
            st.table(data)

            @st.cache  
            def convert_df(df):  
                return df.to_csv().encode('gbk')

            csv = convert_df(data)

            st.sidebar.download_button(  
                label="Download data as CSV",  
                data=csv,  
                file_name='large_df.csv',  
                mime='text/csv', )

            st.sidebar.success('Success!')

            \# 隐藏底部按钮  
            hide_st_style = "<style># MainMenu {visibility: hidden;} footer {visibility: hidden;}</style>"  
            st.markdown(hide_st_style, unsafe_allow_html=True)  
            st.success('Success!')

        elif sidebar == "地图可视化":

            from streamlit_folium import folium_static  
            def load_Map(BaseAlgorithm,data):  
                # 发布数据应用  
                obj = webservice_client(BaseAlgorithm,data)  
                Map = obj.get_Map_model()  
                return Map

            BaseAlgorithm_selectbox = st.sidebar.selectbox("选择模型", (  
            'Zoom_Loading','Full_Load'))

            Map = load_Map(BaseAlgorithm_selectbox,self.data_sources)

            folium_static(Map,width=1400, height=800)

            \# 隐藏底部按钮  
            hide_st_style = "<style># MainMenu {visibility: hidden;} footer {visibility: hidden;}</style>"  
            st.markdown(hide_st_style, unsafe_allow_html=True)  
            st.success('Success!')

        elif sidebar == "关于":  
            st.subheader("streamlit 开发查询手册")

            from PIL import Image

            image = Image.open(r'.\\resources\\streamlit-cheat-sheet.png')  
            st.image(image, caption='', use_column_width=True)

        else:  
            import streamlit as st  
            st.title("欢迎关注 Python 数据分析实例")  
            st.subheader("号主：Brook")

            st.markdown(  
                "<h2 style='text-align: center;'> "  
                "&lt;a href = [https://mp.weixin.qq.com/mp/profile_ext?action=home&\_\_biz=MzI0NzY2MDA4MA==&scene=124#wechat_redirect>➡](https://mp.weixin.qq.com/mp/profile_ext?action=home&__biz=MzI0NzY2MDA4MA==&scene=124#wechat_redirect>➡)</a>Python 数据分析实例.<h2>",  
                unsafe_allow_html=True)

            col_header_logo_left, col_header_logo_center, col_header_logo_right = st.columns(  
                [2, 2, 2])

            col_header_logo_center.image(os.path.join(".", "resources", "qrcode_logo.jpg"), width=400, )

            MENU_LAYOUT = [1, 1, 1, 7, 2]  
            _, _, col_logo, col_text, _ = st.columns(MENU_LAYOUT)  
            col_logo.image(os.path.join(".", "resources", "data.png"), width=80, )  
            col_text.subheader("这是一个记录数据分析开发路上有趣、有料、实用至上的 Python 项目实战分享库，欢迎关注！一起成长～.")

            st.markdown('<br><br>', unsafe_allow_html=True)

if **name**=="**main**":  
    obj = Mapview_App()  
    obj.run()

\`

备注：

1、低代码开发平台 - streamlit

> Streamlit 网站：[https://streamlit.io/](https://streamlit.io/)  
> GitHub 地址：[https://github.com/streamlit/streamlit/](https://github.com/streamlit/streamlit/)

Pycharm 的 Terminal 中运行：streamlit run **Map_app.py**

运行之后，就会自动生成网页：[http://localhost:8501/, 直接浏览器访问即可。](http://localhost:8501/,直接浏览器访问即可。)

2、convert_cir_image.py - 生成圆形 logo 图片脚本  

3、重要组件  

streamlit 写 markdown 一样写网页，代码快速生成 web 网页。代码较简单易理解，只要会写 Python 脚本，能很快上手，这里不对上面代码详细解释了，下面提供常见组件 api 供查阅使用。

![](https://mmbiz.qpic.cn/sz_mmbiz_png/okAIo1O8h7wCUNZRhKIGS8sVSibl03zwj6Hc9U3mfeXdY8yqwPOWHLMHlrBZyF9Ig6z8huTDmzMmNL5ibAIz6Wiaw/640?wx_fmt=png)

**以上是一个应用基本框架介绍，你可以随意调整代码，拓展组件，创建一个更集成、更全面、更复杂的应用。以上为本次分享的全部内容，文中已包含大部分源代码，各位小伙伴赶快动手实践一下吧！原创不易，欢迎点赞、分享支持。** 

**下一篇将提供项目所有源文件，敬请期待！**

👆点击关注｜设为星标｜干货速递👆 
 [https://mp.weixin.qq.com/s?src=11×tamp=1658737802&ver=3941&signature=wrgQrKoBFNKtOF6hhfqIKYXkH4rH4DJKzaRDPOUSJkMYFqatw7T2q38nVgtHK4ung0jklnBIk_AFlGbIuk9GVxMDGDIYlhP4ySFi6vwPOy_RE8gCCmIlE3Z03Ydyhkok&new=1](https://mp.weixin.qq.com/s?src=11&timestamp=1658737802&ver=3941&signature=wrgQrKoBFNKtOF6hhfqIKYXkH4rH4DJKzaRDPOUSJkMYFqatw7T2q38nVgtHK4ung0jklnBIk*AFlGbIuk9GVxMDGDIYlhP4ySFi6vwPOy*RE8gCCmIlE3Z03Ydyhkok&new=1)
