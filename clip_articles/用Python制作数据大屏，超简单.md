# 用Python制作数据大屏，超简单
今天我们用`Streamlit`模块来制作一个数据面板，将数据更加直观地呈现给别人观看，整个页面大致如下图所示  

![](https://mmbiz.qpic.cn/mmbiz_png/Jibw7n291dTwpRFkScFYCURNQUhvQBcxfYbibOIoKgPoJh18F1o7z8KnfticUwsaiaIx9YIic2sqJpwNRdJvKJjia96Q/640?wx_fmt=png)

### 制作工具栏

在页面的左侧是一个工具栏，工具栏中有多个按钮，分别是 “About”、“Demo”、“App” 以及 "Contact" 这几个，用来切换到不同的页面

![](https://mmbiz.qpic.cn/mmbiz_gif/Jibw7n291dTwpRFkScFYCURNQUhvQBcxfSFib61lg8Uia5LTuMymaDkBnTY6qFutn7wz02KVf5TojuXRZicC8CoPMg/640?wx_fmt=gif)

这里主要是通过`streamlit_option_menu`模块来实现的，我们来调用其中的`option_menu`函数，我们需要明确里面的几个参数

-   `menu_title`：工具栏的标题，必填
-   `options`: 规定要有哪几个选项栏，必填
-   `menu_icon`: 每一个选项卡的图标，非必填
-   `default_index`: 默认勾选的选项按钮，一般默认勾选的都是第一个选项按钮
-   `styles`: 每个选项按钮的样式

因此我们要制作的数据面板，工具栏部分的代码如下

`with st.sidebar:  
    choose = option_menu("Main Menu", ["About", "Demo","App", "Contact"],  
                         icons=['house', 'file-slides','app-indicator','person lines fill'],  
                         menu_icon="list", default_index=0,  
                         styles={  
        "container": {"padding": "5!important", "background-color": "#fafafa"},  
        "icon": {"color": "orange", "font-size": "25px"},  
        "nav-link": {"font-size": "16px", "text-align": "left", "margin":"0px", "--hover-color": "#eee"},  
        "nav-link-selected": {"background-color": "#02ab21"},  
    })  
`

### 主页面的设计

`About`页面的功能主要是对整个网页的内容、用途做一个简单的介绍，代码逻辑主要是通过`if else`来判断，例如当我们点击`About`这个选项的时候

\`logo = Image.open(r'952.png')  
profile = Image.open(r'5052.png')  
if choose == "About":  
    col1, col2 = st.columns([0.8, 0.2])  
    with col1:  # To display the header text using css style  
        st.markdown(""" <style> .font {  
        font-size:35px ; font-family: 'Cooper Black'; color: #FF9633;}   
        </style> """, unsafe_allow_html=True)  
        st.markdown('<p class="font">简介</p>', unsafe_allow_html=True)  
    with col2:  # To display brand log  
        st.image(logo, width=130)

    st.write(  
        "介绍......")  
    st.image(profile, width=700)

\`

而当我们点击 “Demo” 这个按钮的时候，该页面的功能主要是通过视频来展示一下该网页的主要功能，播放一段`Demo`视频，代码如下

`elif choose=='Demo':  
    st.markdown(""" <style> .font {  
    font-size:25px ; font-family: 'Cooper Black'; color: #FF9633;}   
    </style> """, unsafe_allow_html=True)  
    st.markdown('<p class="font">Watch a short demo of the app...</p>', unsafe_allow_html=True)  
    video_file = open('Demo.mp4', 'rb')  
    video_bytes = video_file.read()  
    st.video(video_bytes)  
`

而当我们点击 “App” 的时候，则主要展示出来的是整个网页的主要功能了，本案例是通过调用`raceplotly`模块来绘制动态可交互的柱状图，如下图所示

![](https://mmbiz.qpic.cn/mmbiz_gif/Jibw7n291dTwpRFkScFYCURNQUhvQBcxfhZuDicnibxhxbrh6B8ic15yIpE8aHpmWPu9p8527d9bELkmANwmOZXoaA/640?wx_fmt=gif)

我们首先需要上传数据集，然后设置好呈现出来的图表的属性，例如图表的标题、柱状图的柱间距等等，如下图所示

![](https://mmbiz.qpic.cn/mmbiz_gif/Jibw7n291dTwpRFkScFYCURNQUhvQBcxfo3ic6a3LzcvaVZia4CznaiaH30kElgxeMYibBwiaM583oquYCADETyia1w4w/640?wx_fmt=gif)

最后我们来看一下代码，因为篇幅整体有限，这里就先展示一部分代码

`elif choose=='App':  
    #Add a file uploader to allow users to upload their csv file  
    st.markdown(""" <style> .font {  
    font-size:25px ; font-family: 'Cooper Black'; color: #FF9633;}   
    </style> """, unsafe_allow_html=True)  
    st.markdown('<p class="font">Upload your data...</p>', unsafe_allow_html=True) #use st.markdown() with CSS style to create a nice-formatted header/text  
    uploaded_file = st.file_uploader('',type=['csv']) #Only accepts csv file format  
    if uploaded_file is not None:  
        df=pd.read_csv(uploaded_file)  #use AgGrid to create a aggrid table that's more visually appealing than plain pandas datafame  
        grid_response = AgGrid(  
            df,editable=False,  
            height=300,fit_columns_on_grid_load=True,  
            theme='blue',width=100,  
            allow_unsafe_jscode=True,  
            )  
        updated = grid_response['data']  
        df = pd.DataFrame(updated)  
        st.write('---')  
        st.markdown('<p class="font">Set Parameters...</p>', unsafe_allow_html=True)  
        column_list = list(df)  
        column_list = deque(column_list)  
        column_list.appendleft('-')  
        with st.form(key='columns_in_form'):  
            text_style = '<p style="font-family:sans-serif; color:red; font-size: 15px;">***These input fields are required***</p>'  
            st.markdown(text_style, unsafe_allow_html=True)  
            col1, col2, col3 = st.columns([1, 1, 1])  
            ......  
            col4, col5, col6 = st.columns([1, 1, 1])  
            ......  
            col7, col8, col9 = st.columns([1, 1, 1])  
            ......  
            col10, col11, col12 = st.columns([1, 1, 1])  
            ......  
            submitted = st.form_submit_button('Submit')  
            st.write('---')  
            if submitted:  
                raceplot = barplot(df, item_column=item_column, value_column=value_column, time_column=time_column,  
                                   top_entries=num_items)  
                fig = raceplot.plot(item_label=item_label, value_label=value_label, frame_duration=frame_duration,  
                                    date_format=date_format, orientation=orientation)  
                fig.update_layout(......)  
                st.plotly_chart(fig, use_container_width=True)  
`

当我们对于该应用的功能有什么不满、有什么建议想要联系开发者的话，点击 “Contact” 按钮，页面如下图所示

![](https://mmbiz.qpic.cn/mmbiz_png/Jibw7n291dTwpRFkScFYCURNQUhvQBcxf7kJCtUPR4To7nekO3xINb1fq8EJGp1iaHCyjzdGtRV7XZBgNJqTb0Sw/640?wx_fmt=png)

代码如下

`elif choose == "Contact":  
    st.markdown(""" <style> .font {  
    font-size:35px ; font-family: 'Cooper Black'; color: #FF9633;}   
    </style> """, unsafe_allow_html=True)  
    st.markdown('<p class="font">Contact Form</p>', unsafe_allow_html=True)  
    with st.form(key='columns_in_form2',clear_on_submit=True):   
        Name=st.text_input(label='姓名')   
        Email=st.text_input(label='联系方式')   
        Message=st.text_input(label='您想要说的是')   
        submitted = st.form_submit_button('提交')  
        if submitted:  
            st.write('感谢!')  
`

至此整个网站就都完成了，大家可以依次来作为模板制作自己的数据大屏，将数据更加直观地展示出来。

### 获取代码

公众号后台回复【**20220309**】即可获取代码

**NO.\*\***1\*\*

往期推荐

Historical articles

[Pandas 数据挖掘与分析时的常用方法](http://mp.weixin.qq.com/s?__biz=Mzk0NzI3ODMyMA==&mid=2247497526&idx=1&sn=251f1e50030d5f32848428a5d44dd7f9&chksm=c37befa9f40c66bfe125507a091f7fc34536cbef0dd5b25cd58a01168a0203438ed426ba004c&scene=21#wechat_redirect)  

[20 个 Pandas 数据实战案例，干货多多](http://mp.weixin.qq.com/s?__biz=Mzk0NzI3ODMyMA==&mid=2247497429&idx=1&sn=c88170ed4e55a93c2cfa7bfcc9f0f359&chksm=c37bee4af40c675c0dcdbca1a1841283f9488e2ad3ae80fe9b9d7bdfa882d99b62ab29fa48d7&scene=21#wechat_redirect)  

[10 张图带你看清俄乌冲突的始末](http://mp.weixin.qq.com/s?__biz=Mzk0NzI3ODMyMA==&mid=2247497358&idx=1&sn=3db45737d6717651caceb3d829730df7&chksm=c37bee11f40c6707e9fba935093dde35e51476d484c4d1d1bd086d81d6393c18d4621c73902e&scene=21#wechat_redirect)  

[年轻人为什么会猝死？这篇 Python 数据分析报告不可错过！](http://mp.weixin.qq.com/s?__biz=Mzk0NzI3ODMyMA==&mid=2247497175&idx=1&sn=a9ed84d34a8e27db4269b4e78079e6eb&chksm=c37bed48f40c645e67171e2472cc9ce11d9657120e57ef1c4896f5cab9ccd18f06f246074dcf&scene=21#wechat_redirect)  

分享、收藏、点赞、在看安排一下？

![](https://mmbiz.qpic.cn/mmbiz_gif/Jibw7n291dTwpRFkScFYCURNQUhvQBcxfb9588Nt0FHLBxoukibw94Q6cFPsBCNMiagTjWnHzbfut9AuCwRcx6JwQ/640?wx_fmt=gif)

![](https://mmbiz.qpic.cn/mmbiz_gif/Jibw7n291dTwpRFkScFYCURNQUhvQBcxfb9588Nt0FHLBxoukibw94Q6cFPsBCNMiagTjWnHzbfut9AuCwRcx6JwQ/640?wx_fmt=gif)

![](https://mmbiz.qpic.cn/mmbiz_gif/Jibw7n291dTwpRFkScFYCURNQUhvQBcxfb9588Nt0FHLBxoukibw94Q6cFPsBCNMiagTjWnHzbfut9AuCwRcx6JwQ/640?wx_fmt=gif)

![](https://mmbiz.qpic.cn/mmbiz_gif/Jibw7n291dTwpRFkScFYCURNQUhvQBcxfb9588Nt0FHLBxoukibw94Q6cFPsBCNMiagTjWnHzbfut9AuCwRcx6JwQ/640?wx_fmt=gif) 
 [https://mp.weixin.qq.com/s?src=11×tamp=1658737802&ver=3941&signature=1q8dPrlID361lfN-Ijm6leFGiXTUyKsZdlq7mt1T4tDkcoOnXDslUs5g7rYVf7AGKAN6Qufwvl_m9x8Kzhy-6HfjS0EzBgr2sm6RiXkkMF_x8s-7PJnyC3JXY09orAaj&new=1](https://mp.weixin.qq.com/s?src=11&timestamp=1658737802&ver=3941&signature=1q8dPrlID361lfN-Ijm6leFGiXTUyKsZdlq7mt1T4tDkcoOnXDslUs5g7rYVf7AGKAN6Qufwvl*m9x8Kzhy-6HfjS0EzBgr2sm6RiXkkMF*x8s-7PJnyC3JXY09orAaj&new=1)
