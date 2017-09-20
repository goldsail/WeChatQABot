# WeChatQABot
A guide of builing a Q&amp;A bot in WeChat using Microsoft QnA Maker Service  
This document is written in Chinese since most WeChat users speak Chinese. 

# 使用微软 QnA Maker 搭建微信问答机器人

# 0. 简介

- 微软的 QnA Maker 是一个定制化的免费问答机器人服务，用户可以提供一系列的“问题-答案”对（QA pair），然后 QnA Maker 服务会生成一个 API，用户输入询问的问题，API 会返回匹配的答案。值得注意的是，这个 API 采用了一些自然语言处理技术，能够应对语言表达上的模糊：变换着句式提问，甚至将一些词语用同义词替代，都能得到很好的匹配正确率。

- 微信公众号具有一个叫做微信开放平台的功能。作为公众号的运营者，可以手动指定一个服务器的URL，按照微信提供的 API 接口，用来自动回复用户发送的消息。一个典型的应用便是问答机器人：用户进行提问，询问一些常见问题，这些常见问题的答案事先由运营者编写好，只要匹配到正确的问题，就能做出相应的回答。

- 由于 QnA Maker 和微信公众号的 API 是不同的，因此需要我们自行在服务器上搭建一个 web service 做中转。本文以微软 ASP.NET 框架为例，使用 C# 快速搭建一个 API，并部署在服务器上。

## 0.1 系统框图

![系统框图](https://i.loli.net/2017/08/25/59a0296bde092.png)

# 1. 注册并使用微软 QnA Maker 服务

## 1.1 创建和训练

QnA Maker 服务的网址是 [](https://qnamaker.ai)，使用微软账号登录。在页面上方点击“Create new service”，在弹出的页面中，只需输入问答机器人的名字，其他选项均是可选的：  

![创建 QnA Service](https://i.loli.net/2017/08/25/59a02633d6350.png)  

创建之后进入该服务，在左侧会出现三个选项卡：Knowledge Base、Test、Settings。首先点击 Knowledge Base，其中已经预置了一个问答对（Hi - Hello），可以手动添加更多的问答对。或者，也可以在 Settings 选项卡下，选择从文件批量导入。  

添加完成后点击 Save and retrain 按钮，可以进行训练。然后在 Test 中可以手动输入一些问题进行测试。可以发现，变换提问的方式，QnA Maker 仍然能够识别出正确的问题，并找到答案：  

![训练的问题](https://i.loli.net/2017/08/25/59a027b02f573.png)  
![变换问法后，仍能识别正确](https://i.loli.net/2017/08/25/59a027ed2d800.png)  

## 1.2 发布并使用 API

一切就绪后，点击 Publish 按钮，即可发布问答机器人。之后会给出一个 API，向这个 API 发送 POST 请求，就能收到相应回答。  

![POST 请求示例](https://i.loli.net/2017/08/25/59a02907bcd01.png)  

在稍后，我们会介绍如何在 ASP.NET 网页中使用这个 API。

# 2. 注册微信公众号的开发者接口

![启用开发者服务器配置](https://i.loli.net/2017/08/25/59a02a8e2c5d7.png)  

按照微信官方的要求，我们的 URL（即我们中间服务器部署的地址）必须能够对微信服务器发送过来的一种指定格式的 GET 请求做出指定格式的回应（参考资料：微信官方文档），否则这个 URL 是无效的。因此我们必须首先搭建出这样一个服务器。 

# 3. 使用 ASP.NET 快速搭建自己的 Web API 中转服务器

## 3.1 创建 ASP.NET 项目

在 Visual Studio 中创建一个 ASP.NET 项目：  

![](https://i.loli.net/2017/08/25/59a02ba52892c.png)  
![](https://i.loli.net/2017/08/25/59a02c3e6f55e.png)

## 3.2 添加一个 API 控制器

点击菜单栏“项目”->“添加新项”，选择“Web API 控制器类”（如图），文件名称选择为```TokenController.cs```。
![](https://i.loli.net/2017/08/26/59a0cd845591f.png)

将以下代码粘贴并覆盖该文件中的原有代码：

```C#
using System;
using System.Collections.Generic;
using System.Linq;
using System.Net;
using System.Net.Http;
using System.Web.Http;
using System.Web;

using System.Xml;
using System.IO;
using System.Collections;
using System.Xml.Serialization;
using System.Threading.Tasks;
using System.Text;
using Newtonsoft.Json;

namespace WeChatApi.Controllers
{

    public class TokenController : ApiController
    {

        private class QnAMakerResult
        {
            /// <summary>
            /// The top answer found in the QnA Service.
            /// </summary>
            [JsonProperty(PropertyName = "answer")]
            public string Answer { get; set; }

            /// <summary>
            /// The score in range [0, 100] corresponding to the top answer found in the QnA    Service.
            /// </summary>
            [JsonProperty(PropertyName = "score")]
            public double Score { get; set; }
        }
        

        private async Task<string> QueryBotApi(string query)
        {
            string responseString = string.Empty;
            
            var knowledgebaseId = "52c3e0a5-3f3d-4927-xxxx-xxxxxxxxxxxx"; // Use knowledge base id created.
            var qnamakerSubscriptionKey = "e0e88c1aaae44d6eaxxxxxxxxxxxxxxx"; //Use subscription key assigned to you.

            //Build the URI
            Uri qnamakerUriBase = new Uri("https://westus.api.cognitive.microsoft.com/qnamaker/v1.0");
            var builder = new UriBuilder($"{qnamakerUriBase}/knowledgebases/{knowledgebaseId}/generateAnswer");

            //Add the question as part of the body
            var postBody = $"{{\"question\": \"{query}\"}}";

            //Send the POST request
            using (WebClient client = new WebClient())
            {
                //Set the encoding to UTF8
                client.Encoding = System.Text.Encoding.UTF8;

                //Add the subscription key header
                client.Headers.Add("Ocp-Apim-Subscription-Key", qnamakerSubscriptionKey);
                client.Headers.Add("Content-Type", "application/json");
                responseString = client.UploadString(builder.Uri, postBody);
            }

            QnAMakerResult response;
            string responseText = null;
            try
            {
                response = JsonConvert.DeserializeObject<QnAMakerResult>(responseString);
                if (response.Answer == "No good match found in the KB" || response.Score < 0.2)
                {
                    responseText = "我还在学习回答这个问题。回复「 你会回答什么问题 」试试？";
                }
                else
                {
                    responseText = response.Answer;
                }
            }
            catch
            {
                responseText = null;
            }

            return responseText;
        }

        // GET: api/Token
        public HttpResponseMessage Get()
        {

            var parameters = Request.GetQueryNameValuePairs();

            Dictionary<string, string> para = new Dictionary<string, string>();
            foreach (var t in parameters)
            {
                para.Add(t.Key, t.Value);
            }
            string result = para["echostr"];
            var resp = new HttpResponseMessage(HttpStatusCode.OK);
            resp.Content = new StringContent(result, System.Text.Encoding.UTF8, "text/plain");
            return resp;
            
        }

        
        // POST: api/Token
        public async Task<HttpResponseMessage> Post()
        {
            // read

            var inputBytes = await Request.Content.ReadAsByteArrayAsync();
            string input = Encoding.UTF8.GetString(inputBytes);

            string strFromUserName = "0";
            string strCreateTime = "0";

            XmlDocument doc = new XmlDocument();
            doc.LoadXml(input);
            XmlNodeList listFromUserName = doc.GetElementsByTagName("FromUserName");
            strFromUserName = listFromUserName[0].InnerText;

            XmlNodeList listCreateTime = doc.GetElementsByTagName("CreateTime");
            strCreateTime = listCreateTime[0].InnerText;

            XmlNodeList listContent = doc.GetElementsByTagName("Content");

            string answer = await QueryBotApi(listContent[0].InnerText); // API query

            // write

            StringWriter stream = new StringWriter();
            XmlWriter writer = XmlWriter.Create(stream);

            writer.WriteStartElement("xml");

            writer.WriteStartElement("ToUserName");
            writer.WriteCData(strFromUserName);
            writer.WriteEndElement();

            writer.WriteStartElement("FromUserName");
            writer.WriteCData("THU-FIT");
            writer.WriteEndElement();

            writer.WriteStartElement("MsgType");
            writer.WriteCData("text");
            writer.WriteEndElement();

            writer.WriteStartElement("CreateTime");
            writer.WriteString(strCreateTime);
            writer.WriteEndElement();

            writer.WriteStartElement("Content");
            writer.WriteCData(answer);
            writer.WriteEndElement();

            writer.WriteEndElement();
            writer.Flush();

            string result = stream.ToString();
            string header = "<?xmlversion=\"1.0\"encoding=\"utf - 16\"?>";

            result = result.Substring(header.Length);


            var resp = new HttpResponseMessage(HttpStatusCode.OK);
            resp.Content = new StringContent(result, System.Text.Encoding.UTF8, "text/plain");
            // resp.Content = new StringContent(input, System.Text.Encoding.UTF8, "text/plain");



            return resp;
        }
        

        
    }
}
```

将其中的变量```knowledgebaseId```和```qnamakerSubscriptionKey```修改成你在发布微软 QnA Maker 机器人的时候，页面上显示的参数。这两个参数就像你的 QnA Maker API 的用户名和密码，必须正确提供。

## 3.3 将该网站发布到你的服务器

在解决方案资源管理器中，右击该项目（不是右击解决方案），选择“发布”，选择“自定义”，随便输入一个配置名称，点击下一步。  

在发布方法中选择“文件系统”，目标位置可以选在本地电脑上。然后一路next即可完成发布。

## 3.4 配置 Windows Server 服务器

无论是自己的 Windows Server 服务器，还是在云端租用的 Windows Server 虚拟机，都可以很方便地进行网站部署。 

### 3.4.1 安装 IIS

在服务器端，使用“服务器管理器”，点击“添加角色和功能”，然后勾选“Web 服务器 IIS”（如图）

![](https://i.loli.net/2017/08/26/59a0d1ab53e9c.png)

### 3.4.2 安装 ASP.NET

首先在[官方网站](https://www.microsoft.com/web/downloads/platform.aspx)下载 Web Platform Intaller 5.0 。  

安装完并运行，然后搜索“ASP.NET”，安装图示的两个模块：

![](https://i.loli.net/2017/08/26/59a0d324c7984.png)

### 3.4.3 创建 IIS 网站

在服务器端的开始菜单里打开“IIS 管理器”，在左侧点开自己的服务器（名字是 IP 地址），右击“网站”，选择添加网站。  

网站名称可以自己填写，在服务器上找一个目录作为网站的物理路径。  

点击“Connect as”按钮，身份选择“指定用户”，点击“设置”，输入服务器的管理员账户的用户名（默认是Administrator）和密码。

![](https://i.loli.net/2017/08/26/59a0d452953d7.png)

### 3.4.5 将发布好的网站复制到服务器端

在本地端，刚刚我们在 Visual Studio 中发布了该网站。现在找到刚刚的目标发布路径（在本地），将其中的所有文件复制到服务器端刚刚设定的网站物理路径下（在服务器端）。

### 3.4.6 测试网站部署

在本地端的浏览器中，访问以下链接：[](http://xx.xx.xx.xx/api/token?echostr=Hello%20World)，其中```xx.xx.xx.xx```替换为你的服务器的公网 IP 地址。  

如果浏览器能够回显字符串“Hello World”，则说明网站已部署成功。

# 4. 完成微信公众平台配置

回到微信公众平台，在“开发”->“基本配置”中，设置服务器配置，将服务器地址（URL）设置成“http://你的服务器的公网IP地址/api/token”。

之后在手机上对着你的公众号发送“Hi”，如果你的公众号能够回复“Hello”，则说明整个配置就成功了。此时你可以变换着问法，看看你的问答机器人的精彩（或者哭笑不得）的表现吧。

> 注：推荐将服务器部署在香港，这样你的服务器无论是访问 QnA Maker 服务，还是被微信访问，都会比较快。（微信公众平台要求你的服务器必须在5秒内做出应答）


# 5. 参考资料

这些都是我在开发中参阅的文档和回答。

1. https://qnamaker.ai/Documentation/ApiReference
2. https://mp.weixin.qq.com/wiki?t=resource/res_main&id=mp1421140453
3. https://mp.weixin.qq.com/wiki?t=resource/res_main&id=mp1421140543
4. https://stackoverflow.com/questions/14046417/how-to-return-raw-string-with-apicontroller
5. http://www.cnblogs.com/beginor/archive/2012/03/19/2406624.html
6. https://docs.microsoft.com/en-us/aspnet/web-api/overview/getting-started-with-aspnet-web-api/tutorial-your-first-web-api
