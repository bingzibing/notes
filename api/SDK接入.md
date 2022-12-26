# SDK接入

## 一、接入步骤

### 1、接入sdk

1）静态接入方式：下载jar包放入项目libs目录下即可

下载路径：https://pkgs.d.xiaomi.net/artifactory/webapp/#/artifacts/browse/tree/General/snapshots/com/xiaomi/ota/whippet-updater-sdk/1.0.0-SNAPSHOT

下载路径下有相应的版本更新记录，可根据实际需要选择**最新的版本更新记录**使用即可

2）maven接入方式：

```java
repositories {
    maven {
        url "https://pkgs.d.xiaomi.net:443/artifactory/maven-snapshot-virtual/"
    }
}
  
dependencies {
    //具体版本号更换为最新版本，见最下方的版本记录
    implementation 'com.xiaomi.ota:whippet-updater-sdk:1.0.0-20211028.084151-1'
}
```

同时，sdk 需要在接入方的 manifest 中添加网络访问权限

```xml
<uses-permission android:name="android.permission.INTERNET"/>
```

### 2、调用sdk接口

sdk接口都在**com.xiaomi.sdk.whippetapplication.WhippetBaseApi**类中，具体的内容见后面接口定义。

### 3、接口统一返回结果

所有功能性接口（除初始化的接口）的接口返回结果都是ResponseEntity

```java
class ResponseEntity{
    ResponseHeader header;
    // body是指服务器端返回的数据，用Result或者直接使用String来封装
    T body;        
}

class ResponseHeader{
    String code;        //状态码
    String desc;        //描述
    String msg;         //信息msg
    String location;    //位置
}

class Result{
    int code;        //服务器端业务返回code
    String message;  //服务器端业务返回message
    T data;          //服务器端业务返回接口内容
}
```

示例

```java
RequestEntity {
    header = ResponseHeader {
        code = '200',
        desc = '成功',
        msg = 'null',
        location = 'null'
    },
    body = Result {
        code = 200,
        message = ok,
        data = [{
           groupId = 91.0,
           groupName = griup66
        }]
    }
}
```

## 二、接口定义

### 1、初始化

**所有的接口都必须在初始化后才能使用。**

```java
static void init(String appId, String appKey, String region)
    maven {
        url "https://pkgs.d.xiaomi.net:443/artifactory/maven-snapshot-virtual/"
    }
```

初始化需要三个String类型的参数**appId、appKey、region**，都必须指定才能完成初始化

- **appId、appKey**需要找接口提供方进行分配
- **region**：系统更新服务是全球范围的服务，涉及多个地区的集群，多集群也是数据本地化的基础。由于判断所在地区的能力各APP已经实现，因此需要传入区域code到region属性上，SDK会根据区域判断调用哪个集群。

**区域code和域名的映射关系**

| 区域code   | 集群     | 域名 | 备注     |
| ---------- | -------- | -------- | ---------------- |
| STAGING    | 测试集群   | whippet-updater-staging-cl77800.staging.ingress.mice.cc.d.xiaomi.net  | http://whippet-updater-staging-cl77800.staging.ingress.mice.cc.d.xiaomi.net  |
| CN     | 国内集群   | whippet.bsp.xiaomi.com    |   http://whippet.bsp.xiaomi.com   |
| INDIA    | 印度集群 |  whippet-india.bsp.xiaomi.com |   http://whippet-india.bsp.xiaomi.com   |
| RUSSIA   | 俄罗斯集群     | whippet-russia.bsp.xiaomi.com       |   http://whippet-russia.bsp.xiaomi.com   |
| AMS | 欧洲阿姆斯特丹集群     | whippet-ams.bsp.xiaomi.com       |    http://whippet-ams.bsp.xiaomi.com  |
| SGP | 欧洲区域新加坡集群     | whippet-intl.bsp.xiaomi.com       |   http://whippet-intl.bsp.xiaomi.com     |


示例

```
WhippetBaseApi.init("12345","abcdefg","CN");
```

### 2、checkUpdate

检查更新

```java
static ResponseEntity<Result<UpdaterQueryResponse>> checkUpdate(
                              UpdaterQueryRequest updaterRequest)
```

请求参数UpdaterQueryRequest对象中的属性：

| 字段名称 | 类型   | 是否必传 | 字段描述 | 备注 |
| -------- | ------ | -------- | -------- | ---- |
| product   | 字符串 | 是       | 产品名称     |      |
| device  | 字符串 | 是       | 设备名称   |      |
| modules  | 列表数组 | 是       | 模块以及版本信息   |      |
| identity  | 对象 | 是       | 身份对象标识   |      |
| properties  | 对象 | 否       | 其他属性   |      |


modules中对象的属性：

| 字段名称 | 类型   | 是否必传 | 字段描述 | 备注 |
| -------- | ------ | -------- | -------- | ---- |
| module   | 字符串 | 是       | 模块     |      |
| version  | 字符串 | 是       | 版本号   |      |


identity属性：

| 字段名称 | 类型   | 是否必传 | 字段描述 | 备注 |
| -------- | ------ | -------- | -------- | ---- |
| imei   | 字符串 | 否       | IMEI     |      |
| miid  | 字符串 | 否       | 小米ID   |      |
| sn  | 字符串 | 否       | 序列号   |      |


properties，根据实际需要传入

示例

```java
UpdaterQueryRequest queryRequest = new UpdaterQueryRequest();
queryRequest.setProduct("K11");
queryRequest.setDevice("5NFC");
queryRequest.setLang("zh_CN");

UpdaterModuleCheckParam param = new UpdaterModuleCheckParam();
param.setModule("ROS");
param.setVersion("V21.01.03");
List<UpdaterModuleCheckParam> list = Arrays.asList(param);
queryRequest.setModules(list);

Map<String, String> identity = new HashMap<>();
identity.put("sn", "sn:ab.12.34.55");
identity.put("miid", "hfdkasjflasjdfas");
queryRequest.setIdentity(identity);

ResponseEntity<Result<UpdaterQueryResponse>> responseEntity = WhippetBaseApi.checkUpdate(queryRequest);
```

返回数据

```java
RequestEntity {
    header = ResponseHeader {
        code = '200',
        desc = '成功',
        msg = 'null',
        location = 'null'
    },
    body = {
        code = 200,
        message = ok,
        data = UpdaterQueryResponse{...}
    }
}
```

### 3、getApplyGroups

获取可申请的组列表，示例场景：内测组

```java
static ResponseEntity<Result<List<SimpleGroupResponse>>> getApplyGroups(
                                    QueryGroupForApplyRequest applyRequest)
```

请求参数QueryGroupForApplyRequest对象中的属性：

| 字段名称 | 类型   | 是否必传 | 字段描述 | 备注 |
| -------- | ------ | -------- | -------- | ---- |
| product   | 字符串 | 是       | 产品名称     |      |
| device  | 字符串 | 是       | 设备名称   |      |
| properties  | 对象 | 否       | 其他属性   |      |

properties，根据实际需要传入

示例

```java
QueryGroupForApplyRequest group = new QueryGroupForApplyRequest();
group.setDevice("K91.dog.test");
group.setProduct("K91");
ResponseEntity<Result<List<SimpleGroupResponse>>> responseEntity = WhippetBaseApi.getApplyGroups(group);
```

返回数据

```java
RequestEntity {
    header = ResponseHeader {
        code = '200',
        desc = '成功',
        msg = 'null',
        location = 'null'
    },
    body = {
        code = 200,
        message = ok,
        data = List<SimpleGroupResponse>{...}
    }
}
```

### 4、applyGroup

针对接口getApplyGroups获取的组进行申请

```java
static ResponseEntity<String> applyGroup(CreateGroupApplyRequest applyRequest)
```

请求参数CreateGroupApplyRequest对象中的属性：

| 字段名称 | 类型   | 是否必传 | 字段描述 | 备注 |
| -------- | ------ | -------- | -------- | ---- |
| product   | 字符串 | 是       | 产品名称     |      |
| device  | 字符串 | 是       | 设备名称   |      |
| groupId  | 长整型Long |        | 申请组ID   |      |
| groupName  | 字符串 |        | 申请组名   |      |
| identityType  | 字符串 |        | 身份标识类型  |      |
| identityName  | 字符串 |        | 设备标识值   |      |
| reason  | 字符串 |        | 申请原因   |      |
| applyTime  | 字符串 |        | 申请时间   |      |
| properties  | 对象 | 否       | 其他属性   |      |


示例

```java
CreateGroupApplyRequest queryRequest = new CreateGroupApplyRequest();

queryRequest.setProduct("k91");
queryRequest.setDevice("k91.athena");
queryRequest.setGroupId(91L);
queryRequest.setGroupName("group66");
ResponseEntity<String> entity = WhippetBaseApi.applyGroup(queryRequest);
```

返回数据

```java
RequestEntity {
    header = ResponseHeader {
        code = '200',
        desc = '成功',
        msg = 'null',
        location = 'null'
    },
    body = ""
}
```

### 5、getApplyResult

获取申请结果

```
static ResponseEntity<Result<ApplyResultResponse>> getApplyResult(
                             OtaQueryApplyResultRequest resultRequest)
```

请求参数OtaQueryApplyResultRequest对象中的属性：

| 字段名称 | 类型   | 是否必传 | 字段描述 | 备注 |
| -------- | ------ | -------- | -------- | ---- |
| product   | 字符串 | 是       | 产品名称     |      |
| device  | 字符串 | 是       | 设备名称   |      |
| groupId  | 长整型Long |        | 申请组ID   |      |
| groupName  | 字符串 |        | 申请组名   |      |
| identityType  | 字符串 |        | 身份标识类型  |      |
| identityName  | 字符串 |        | 设备标识值   |      |


示例

```java
OtaQueryApplyResultRequest resultRequest = new OtaQueryApplyResultRequest();
resultRequest.setProduct("k91");
resultRequest.setDevice("k91.athena");
resultRequest.setGroupId(91L);
resultRequest.setGroupName("group66");
ResponseEntity<Result<ApplyResultResponse>> entity = WhippetBaseApi.getApplyResult(resultRequest);
```

返回数据

```java
RequestEntity {
    header = ResponseHeader {
        code = '200',
        desc = '成功',
        msg = 'null',
        location = 'null'
    },
    body = {
        code = 200,
        message = ok,
        data = ApplyResultResponse{...}
    }
}
```

### 6、updateResultReport

更新结果上报

```java
static ResponseEntity<String> updateResultReport(
                              StatReportRequest reportRequest) 
```

请求参数StatReportRequest对象中的属性：

| 字段名称 | 类型   | 是否必传 | 字段描述 | 备注 |
| -------- | ------ | -------- | -------- | ---- |
| date   | String |        | 上报日期     |      |
| time  | Long |        | 上报时间   |      |
| report_id  | String |        | 上报通道标识   |      |
| service_code  | int |        | 服务标识   |  ota:0  K91：1    |
| product   | String |        | 产品     |      |
| device  | String |        | 设备   |      |
| module_name  | String |        | 模块  | Ota默认为Rom     |
| identity  | 对象 |        | 唯一标识   |      |
| group_name   | String |        | 0-15, 若是ota， 则是分组名称     |      |
| region  | String |        | 区域   |      |
| language  | String |        | 语言  |     |
| cur_version  | String |        | 当前版本   |      |
| to_version  | String |        | 要升级的版本  |    |
| event  | int |        | 事件类型  | 开始升级：10，升级失败：100， 升级成功：0   |
| event_value  | int |        | 错误类型  |  0:升级成功; 1:下载失败; 2:包的MD5不对; 11:空间不足（任务暂停原因为下载管理空间不足）; 12:网络错误; 13:用户在下载管理删除任务; 14:Mirror列表错误; 23:包校验过程其他错误; 99:其他原因   |
| package_type  | int |        | 使用的升级包类型  |   |
| event_date  | String |        | 事件具体日期  |   |
| event_time  | Long |        | 事件具体时间  |   |


示例

```java
StatReportRequest request = new StatReportRequest();
ResponseEntity<String> entity = WhippetBaseApi.updateResultReport(request);
```

返回数据

```java
RequestEntity {
    header = ResponseHeader {
        code = '200',
        desc = '成功',
        msg = 'null',
        location = 'null'
    },
    body = ""
}
```

## 常见异常定义

状态码是header中的状态码，状态吗如下：

| 状态码值 | 解释                     |
| -------- | ------------------------ |
| 200      | success                  |
| 212      | 签名校验失败             |
| 213      | 接口数据为空             |
| 214      | 服务未注册               |
| 215      | Key获取失败              |
| 216      | body解密失败             |
| 222      | 未初始化或初始化参数为空 |
| 999      | 系统异常                 |

