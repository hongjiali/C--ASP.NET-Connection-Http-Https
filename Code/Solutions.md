
## 分析：
- 在 调试中 (HttpWebResponse)req.GetResponse() 会执行中断，报错 "asp.net Unable to connect to the remote server"

## 猜想：
- 请求数据方式由 http 协议转变成了 https 协议.

  1.是否是SSL安全协议的拦截问题？

## 验证和了解：
- 未加密的http prot为80，而https的加密端口为443. 

  1.在本机调试中获取http请求就可以获取数据.
  
  2.而数据发布到服务器之后无法得到数据.
  
  
- [该博客中指正了一个问题，当通过访问https或请求数据时,针对SSL协议的安全性我们要对Https进行特殊处理](http://blog.51cto.com/zhoufoxcn/561934)

  1.但是又有说明其实C#是可以自动处理协议，无论是https还是http

## 尝试和解决
- 在涉及到利用**WebRequest**处理**https**请求时,因为涉及到SSL安全证书的原因,我们还是需要在请求之前加一个验证.


```
private bool CheckValidationResult(object sender,
        X509Certificate certificate, X509Chain chain, SslPolicyErrors errors)
{
    return true;// Always accept
}
```

```
if(url.StartsWith("https", StringComparison.OrdinalIgnoreCase))
    {
        ServicePointManager.ServerCertificateValidationCallback =
                new RemoteCertificateValidationCallback(CheckValidationResult);
    }
```

### 全部代码
```
 private bool CheckValidationResult(object sender,
        X509Certificate certificate, X509Chain chain, SslPolicyErrors errors)
        {
            return true;// Always accept
        }


        private SAES_M_comChatResult GetResult(string url)
        {
            HttpWebRequest request = (HttpWebRequest)WebRequest.Create(url);
            request.Method = "GET";

            if (url.StartsWith("https", StringComparison.OrdinalIgnoreCase))
            {
                ServicePointManager.ServerCertificateValidationCallback =
                    new RemoteCertificateValidationCallback(CheckValidationResult);
            }

            HttpWebResponse response = (HttpWebResponse)request.GetResponse();

            // Get the stream associated with the response.
            Stream receiveStream = response.GetResponseStream();

            // Pipes the stream to a higher level stream reader with the required encoding format. 
            StreamReader readStream = new StreamReader(receiveStream, Encoding.UTF8);
            string received = readStream.ReadToEnd();
            // Console.WriteLine("Response stream received.");
            //  Console.WriteLine (readStream.ReadToEnd ());
            //  Response.Write("<script>window.alert('" + readStream.ReadToEnd() + "');</script>");
            response.Close();
            readStream.Close();

            if (received == null && received == "") return null;

            SAES_M_comChatResult result = null;
            using (var ms = new MemoryStream(Encoding.Unicode.GetBytes(received)))
            {
                DataContractJsonSerializer deseralizer = new DataContractJsonSerializer(typeof(SAES_M_comChatResult));
                result = (SAES_M_comChatResult)deseralizer.ReadObject(ms);//反序列化ReadObject
            }

            return result;
        }
```

[参考博客地址（传送）](https://www.crifan.com/access_https_type_url_in_csharp/)
