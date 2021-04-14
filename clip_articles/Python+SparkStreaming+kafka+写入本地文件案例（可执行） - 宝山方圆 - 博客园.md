# Python+SparkStreaming+kafka+写入本地文件案例（可执行） - 宝山方圆 - 博客园
从 kafka 中读取指定的 topic，根据中间内容的不同，写入不同的文件中。

文件按照日期区分。

![](https://common.cnblogs.com/images/copycode.gif)

\#!/usr/bin/env python # -\*- coding: utf-8 -\*- # @Time    : 2018/4/9 11:49 # @Author  : baoshan # @Site    : 

# @File    : readTraceFromKafkaStreamingToJson.py # @Software: PyCharm Community Edition

from pyspark import SparkContext from pyspark.streaming import StreamingContext from pyspark.streaming.kafka import KafkaUtils import datetime import json import time from collections import defaultdict import subprocess class KafkaMessageParse: def extractFromKafka(self, kafkainfo): if type(kafkainfo) is tuple and len(kafkainfo) == 2: return kafkainfo\[1] def lineFromLines(self, lines): if lines is not None and len(lines) > 0: return lines.strip().split("\\n") def messageFromLine(self, line): if line is not None and"message"in line.keys(): return line.get("message") def extractFromMessage(self, message): try:
            jline \\= json.loads(message)
            result \\= defaultdict()
            name \\= jline.get("name") if "speech" in name:
                trace_id \\= jline.get("trace_id")
                parent_id \\= jline.get("parent_id")
                span_id \\= jline.get("span_id")
                sa \\= jline.get("sa")
                sr \\= jline.get("sr")
                ss \\= jline.get("ss")
                ret \\= jline.get("ret")

                result\['trace\_id'\] = trace\_id
                result\['parent\_id'\] = parent\_id
                result\['span\_id'\] = span\_id
                result\['name'\] = name
                result\['sa'\] = sa
                result\['sr'\] = sr
                result\['ss'\] = ss
                result\['ret'\] = ret

                annotation \= jline.get("annotation") try: for anno in annotation: if anno.get("name") == "nlp":
                            debug\_log \= anno.get("debug\_log")
                            debug\_log\_anno \= debug\_log\[0\]
                            asr \= debug\_log\_anno.get("asr")  # asr
                            nlp = debug\_log\_anno.get("nlp")

                            action \= debug\_log\_anno.get("action")
                            jaction \= json.loads(action)
                            response \= jaction.get("response")
                            tts \= response.get("action").get("directives")\[0\].get("item").get("tts")
                            result\['tts'\] = tts

                            jnlp \= json.loads(nlp)
                            intent \= jnlp.get('intent')
                            app\_id \= jnlp.get('appId')
                            cloud \= jnlp.get("cloud")
                            slots \= jnlp.get("slots")

                            result\['app\_id'\] = app\_id
                            result\['intent'\] = intent
                            result\['cloud'\] = cloud
                            result\['asr'\] = asr
                            result\['nlp'\] = nlp
                            result\['slots'\] = slots
                    debug\_log \= jline.get("debug\_log")
                    debug\_log0 \= debug\_log\[0\]
                    session\_id \= debug\_log0.get("session\_id")
                    codec \= debug\_log0.get("codec") if not session\_id:
                        session\_id \= ""  # 超级无敌重要
                    wavfile = session\_id + ".wav" codecfile \= session\_id + "." + codec

                    asrtimestr \= session\_id.split("\-")\[-1\] try:
                        st \= time.localtime(float(asrtimestr)) except:
                        st \= time.localtime()
                    asrtime \= time.strftime("%Y-%m-%d %H:%M:%S", st)
                    asrthedate \= time.strftime("%Y%m%d", st)

                    asrdeviceid \= debug\_log0.get("device\_id")
                    asrdevicetype \= debug\_log0.get("device\_type")
                    asrdevicekey \= debug\_log0.get("device\_key")

                    result\['session\_id'\] = session\_id
                    result\['device\_id'\] = asrdeviceid
                    result\['device\_key'\] = asrdevicekey
                    result\['device\_type'\] = asrdevicetype
                    result\['thedate'\] = asrtime
                    result\['wavfile'\] = wavfile
                    result\['codecfile'\] = codecfile
                    result\['asrthedate'\] = asrthedate

                    strmessage \= json.dumps(result, ensure\_ascii=False) return strmessage except: return strmessage elif "tts" in name: # tts
                try:
                    trace\_id \= jline.get("trace\_id")
                    parent\_id \= jline.get("parent\_id")
                    span\_id \= jline.get("span\_id")
                    name \= jline.get("name")
                    sa \= jline.get("sa")
                    sr \= jline.get("sr")
                    ss \= jline.get("ss")
                    ret \= jline.get("ret")

                    result\['trace\_id'\] = trace\_id
                    result\['parent\_id'\] = parent\_id
                    result\['span\_id'\] = span\_id
                    result\['name'\] = name
                    result\['sa'\] = sa
                    result\['sr'\] = sr
                    result\['ss'\] = ss
                    result\['ret'\] = ret

                    debug\_log \= jline.get("debug\_log")
                    debug\_log\_tts \= debug\_log\[0\]
                    text \= debug\_log\_tts.get("text")
                    codec \= debug\_log\_tts.get("codec")
                    declaimer \= debug\_log\_tts.get("declaimer")
                    logs \= debug\_log\_tts.get("logs")
                    params \= debug\_log\_tts.get("params")

                    result\['text'\] = text
                    result\['codec'\] = codec
                    result\['declaimer'\] = declaimer
                    result\['logs'\] = logs
                    result\['params'\] = params

                    strresult \= json.dumps(result, ensure\_ascii=False) return strresult except: return None except: return ''

def tpprint(val, num=10000): """ Print the first num elements of each RDD generated in this DStream.
    @param num: the number of elements from the first will be printed. """
    def takeAndPrint(time, rdd):
        taken \\= rdd.take(num + 1) print("########################") print("Time: %s" % time) print("########################")
        DATEFORMAT \\= '%Y%m%d' today \\= datetime.datetime.now().strftime(DATEFORMAT)

        speechfile \= open("/mnt/data/trace/trace.rt.speech." + today, "a")
        ttsfile \= open("/mnt/data/trace/trace.rt.tts." + today, "a")
        otherfile \= open("/mnt/data/trace/trace.rt.other." + today, "a") for record in taken\[:num\]: if record is not None and len(record) > 2: # None 不打印
                print(record)
                jrecord \= json.loads(record)
                name \= jrecord.get("name") if "speech" in name:
                    speechfile.write(str(record) \+ "\\n") elif "tts" in name:
                    ttsfile.write(str(record) \+ "\\n") else:
                    otherfile.write(str(record) \+ "\\n")

        speechfile.close()
        ttsfile.close()
        otherfile.close() if len(taken) > num: print("...")

    val.foreachRDD(takeAndPrint) if \_\_name\_\_ == '\_\_main\_\_':
    zkQuorum \= 'datacollect-1:2181,datacollect-2:2181,datacollect-3:2181' topic \= {'trace-open-gw-5': 1, 'trace-open-gw-6': 1, 'trace-open-gw-7': 1, 'trace-open-gw-8': 1, 'trace-open-gw-9': 1}
    groupid \= "rokid-trace-rt" master \= "local\[\*\]" appName \= "SparkStreamingRokidTrace" timecell \= 5 sc \= SparkContext(master=master, appName=appName)
    ssc \= StreamingContext(sc, timecell)

    kvs \= KafkaUtils.createStream(ssc, zkQuorum, groupid, topic)
    kmp \= KafkaMessageParse()
    lines \= kvs.map(lambda x: kmp.extractFromKafka(x))
    lines1 \= lines.flatMap(lambda x: kmp.lineFromLines(x))
    valuedict \= lines1.map(lambda x: eval(x))
    message \= valuedict.map(lambda x: kmp.messageFromLine(x))
    rdd2 \= message.map(lambda x: kmp.extractFromMessage(x))  # result is a json str

 tpprint(rdd2)

    ssc.start()
    ssc.awaitTermination()

![](https://common.cnblogs.com/images/copycode.gif)

还请各位大仙不吝赐教！ 
 [https://www.cnblogs.com/zhzhang/p/8762908.html](https://www.cnblogs.com/zhzhang/p/8762908.html)
