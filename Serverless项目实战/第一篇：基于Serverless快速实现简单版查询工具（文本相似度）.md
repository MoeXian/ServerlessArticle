# 第一篇：基于Serverless快速实现简单版查询工具（文本相似度）

## 需求背景

朋友的单位，有一个小型的图书室，图书室中摆放了很多的书，每本书都被编号放在对应的区域，为了让大家更快，更容易找到这些书，他联系我，让我帮他弄一个图书查询系统。可以通过用户输入，模糊匹配到对应的结果，并且提供书籍对应的地点。

## 功能设计

  * 让朋友把书籍整理并存储到一个Excel表格中；
  * 将Excel表放到对象存储中，云函数读取这个文件，并且解析；
  * 根据词语的相似寻找相似的图书；
  * 前端页面通过MUI制作，放在对象存储中，并且使用对象存储的Website功能；

## 整体实现

### 数据形态

Excel样式主要包括书名和编号，同时下面包括分类的tab：

![](https://others-1256773370.cos.ap-chengdu.myqcloud.com/article/material/3-1-1.png)

### 基于函数的搜索功能

核心代码实现：

    
```python
import jieba
import openpyxl
from gensim import corpora, models, similarities
from collections import defaultdict
import urllib.request

with open("/tmp/book.xlsx", "wb") as f:
    f.write(
        urllib.request.urlopen("https://********").read()
    )


top_str = "abcdefghijklmn"
book_dict = {}
book_list = []
wb = openpyxl.load_workbook('/tmp/book.xlsx')
sheets = wb.sheetnames
for eve_sheet in sheets:
    print(eve_sheet)
    sheet = wb.get_sheet_by_name(eve_sheet)
    this_book_name_index = None
    this_book_number_index = None
    for eve_header in top_str:
        if sheet[eve_header][0].value == "书名":
            this_book_name_index = eve_header
        if sheet[eve_header][0].value == "编号":
            this_book_number_index = eve_header
    print(this_book_name_index, this_book_number_index)
    if this_book_name_index and this_book_number_index:
        this_book_list_len = len(sheet[this_book_name_index])
        for i in range(1, this_book_list_len):
            add_key = "%s_%s_%s" % (
                sheet[this_book_name_index][i].value, eve_sheet, sheet[this_book_number_index][i].value)
            add_value = {
                "category": eve_sheet,
                "name": sheet[this_book_name_index][i].value,
                "number": sheet[this_book_number_index][i].value
            }
            book_dict[add_key] = add_value
            book_list.append(add_key)


def getBookList(book, book_list):
    documents = []
    for eve_sentence in book_list:
        tempData = " ".join(jieba.cut(eve_sentence))
        documents.append(tempData)
    texts = [[word for word in document.split()] for document in documents]
    frequency = defaultdict(int)
    for text in texts:
        for word in text:
            frequency[word] += 1
    dictionary = corpora.Dictionary(texts)
    new_xs = dictionary.doc2bow(jieba.cut(book))
    corpus = [dictionary.doc2bow(text) for text in texts]
    tfidf = models.TfidfModel(corpus)
    featurenum = len(dictionary.token2id.keys())
    sim = similarities.SparseMatrixSimilarity(
        tfidf[corpus],
        num_features=featurenum
    )[tfidf[new_xs]]
    book_result_list = [(sim[i], book_list[i]) for i in range(0, len(book_list))]
    book_result_list.sort(key=lambda x: x[0], reverse=True)
    result = []
    for eve in book_result_list:
        if eve[0] >= 0.25:
            result.append(eve)
    return result


def main_handler(event, context):
    try:
        print(event)
        name = event["body"]
        print(name)
        base_html = '''<div class='mui-card'><div class='mui-card-header'>{{book_name}}</div><div class='mui-card-content'><div class='mui-card-content-inner'>分类：{{book_category}}<br>编号：{{book_number}}</div></div></div>'''
        result_str = ""
        for eve_book in getBookList(name, book_list):
            book_infor = book_dict[eve_book[1]]
            result_str = result_str + base_html.replace("{{book_name}}", book_infor['name']) \
                .replace("{{book_category}}", book_infor['category']) \
                .replace("{{book_number}}", book_infor['number'] if book_infor['number'] else "")
        if result_str:
            return result_str
    except Exception as e:
        print(e)
    return '''<div class='mui-card' style='margin-top: 25px'><div class='mui-card-content'><div class='mui-card-content-inner'>未找到图书信息，请您重新搜索。</div></div></div>'''
    
```

同时配置APIGW：

![](https://others-1256773370.cos.ap-chengdu.myqcloud.com/article/material/3-1-2.png)


### 功能页面

```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8">
    <title>图书检索系统</title>
    <meta name="viewport" content="width=device-width, initial-scale=1,maximum-scale=1,user-scalable=no">
    <meta name="apple-mobile-web-app-capable" content="yes">
    <meta name="apple-mobile-web-app-status-bar-style" content="black">

    <link rel="stylesheet" href="https://others-1256773370.cos.ap-chengdu.myqcloud.com/booksearch/css/mui.min.css">
    <style>
        html,
        body {
            background-color: #efeff4;
        }
    </style>
    <script>
        function getResult() {
            var UTFTranslate = {
                Change: function (pValue) {
                    return pValue.replace(/[^\u0000-\u00FF]/g, function ($0) {
                        return escape($0).replace(/(%u)(\w{4})/gi, "&#x$2;")
                    });
                },
                ReChange: function (pValue) {
                    return unescape(pValue.replace(/&#x/g, '%u').replace(/\\u/g, '%u').replace(/;/g, ''));
                }
            };

            var xmlhttp;
            if (window.XMLHttpRequest) {
                // IE7+, Firefox, Chrome, Opera, Safari 浏览器执行代码
                xmlhttp = new XMLHttpRequest();
            } else {
                // IE6, IE5 浏览器执行代码
                xmlhttp = new ActiveXObject("Microsoft.XMLHTTP");
            }
            xmlhttp.onreadystatechange = function () {
                if (xmlhttp.readyState == 4 && xmlhttp.status == 200 && xmlhttp.responseText) {
                    document.getElementById("result").innerHTML = UTFTranslate.ReChange(xmlhttp.responseText).slice(1, -1).replace("\"",'"');
                }
            }
            xmlhttp.open("POST", "https://********", true);
            xmlhttp.setRequestHeader("Content-type", "application/x-www-form-urlencoded");
            xmlhttp.send(document.getElementById("book").value);
        }
    </script>
</head>
<body>
<div class="mui-content" style="margin-top: 50px">
    <h3 style="text-align: center">图书检索系统</h3>
    <div class="mui-content-padded" style="margin: 10px; margin-top: 20px">
        <div class="mui-input-row mui-search">
            <input type="search" class="mui-input-clear" placeholder="请输入图书名" id="book">
        </div>
        <div class="mui-button-row">
            <button type="button" class="mui-btn mui-btn-numbox-plus" style="width: 100%" onclick="getResult()">检索
            </button>&nbsp;&nbsp;
        </div>
    </div>
    <div id="result">
        <div class="mui-card" style="margin-top: 25px">
            <div class="mui-card-content">
                <div class="mui-card-content-inner">
                    可以在搜索框内输入书籍的全称，或者书籍的简称，系统支持智能检索功能。
                </div>
            </div>
        </div>
    </div>
</div>
<script src="https://others-1256773370.cos.ap-chengdu.myqcloud.com/booksearch/js/mui.min.js"></script>
</body>
</html>
```    

## 效果展示

为了便于朋友使用，将这个页面用过Webview封装成一个APP，整体效果如下：

![](https://others-1256773370.cos.ap-chengdu.myqcloud.com/article/material/3-1-3.png)

## 总结

这个APP是一个低频使用APP，可以这样认为，如果做在一个传统服务器上，这应该不是一个明智的选择，云函数的按量付费，对象存储与APIGW的融合，完美解决了资源浪费的问题，同时借用云函数的APIGW触发器，很简单轻松的替代传统的Web框架和部分服务器软件的安装和使用、维护等。这个例子非常小，但是确是一个有趣的小工具，除了图书查询之外，还可以考虑做成成绩查询等。
