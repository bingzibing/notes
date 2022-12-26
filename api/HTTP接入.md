# 客户端接入文档

## 一 对接技术标准

同步请求采用原小米网自定义的x5协议调用(http)

### 域名

测试staging环境：whippet-updater-staging-cl77800.staging.ingress.mice.cc.d.xiaomi.net

国内环境：whippet.bsp.xiaomi.com

新加坡集群：whippet-intl.bsp.xiaomi.com

俄罗斯集群：whippet-russia.bsp.xiaomi.com

印度集群：whippet-india.bsp.xiaomi.com

欧洲集群：whippet-ams.bsp.xiaomi.com

### x5协议

参考文档：https://wiki.n.miui.com/pages/viewpage.action?pageId=262079415，

https://wiki.n.miui.com/pages/viewpage.action?pageId=62387920

该协议通过提供接口方提供的appId和key进行接口校验。appId和key需要找接口提供方分配。

**接口文档中的请求参数和返回参数均为x5协议的body内容。整体的http请求报文是x5协议的请求头和报文body组成。**

# 二 接口定义

## 1 检查更新

地址：/mi/ota/updater/v1/checkUpdate

#### 请求参数

字段名称类型是否必传字段描述备注product字符串是产品名称device字符串是设备名称lang字符串是语言，传入语言定义列表中的code字段[语言定义列表](https://xiaomi.f.mioffice.cn/docs/dock4KgiCiodj6FLSUsVOgA2z8e) modules列表数组是模块以及版本信息identity对象是身份对象标识properties对象否其他属性

| 字段名称   | 类型     | 是否必传 | 字段描述         | 备注 |
| ---------- | -------- | -------- | ---------------- | ---- |
| product    | 字符串   | 是       | 产品名称         |      |
| device     | 字符串   | 是       | 设备名称         |      |
| modules    | 列表数组 | 是       | 模块以及版本信息 |      |
| identity   | 对象     | 是       | 身份对象标识     |      |
| properties | 对象     | 否       | 其他属性         |      |

modules中对象的属性：

| 字段名称 | 类型   | 是否必传 | 字段描述 | 备注 |
| -------- | ------ | -------- | -------- | ---- |
| module   | 字符串 | 是       | 模块     |      |
| version  | 字符串 | 是       | 版本号   |      |

字段名称类型是否必传字段描述备注module字符串是模块version字符串是版本号

identity属性：

| 字段名称 | 类型   | 是否必传 | 字段描述 | 备注 |
| -------- | ------ | -------- | -------- | ---- |
| imei     | 字符串 | 否       |          |      |
| sn       | 字符串 | 否       |          |      |
| ...      | ...    | ...      | ...      | ...  |

字段名称类型是否必传字段描述备注imei字符串否sn字符串否...............

properties，根据实际需要传入

示例

```json
{
  "header": {
    "appid": "mibpm_1008",
    "method": "api/fin/importData",
    "url": "http://xxx",
    "sign": "31A307FB0243DDCDE9219D4E3E1D8403"
  },
  "body": {
    "product":"xiaomishouhuan",
    "device":"5NFC",
    "modules": [
      {
        "module": "ROS",
        "version": "21.01.03"
      }
    ],
    "identity": {
      "sn": "sn:ab.12.34.55",
      "miid": "hfdkasjflasjdfas"
    },
    "properties": {
      "os.version": "XXX",
      "os.name": "IOS"
    }
  }
}
```

#### 返回数据

状态码是body中的状态码，标识业务状态，状态码定义：

| 状态码值 | 解释                                                         |
| -------- | ------------------------------------------------------------ |
| 200      | 有更新，payload中会有更新的详细信息                          |
| 210      | 当前已经是最新                                               |
| 220      | 暂停更新，payload中会指定一个时间，在这个时间之内不要再检查更新，单位为分 |

状态为有更新的时候，返回数据示例

```json
{
  "header": {
    "code": "200",
    "msg": "请求成功",
    "desc": "http://xxx",
    "location": "CH"
  },
  "body": {
    "code": 200,
    "message": "处理成功",
    "data": {
      "modules": [
        {
          "product": "xiaomishouhuan",
          "device": "5NFC",
          "module": "ROS",
          "current": {
            "version": "21.01.04",
            "versionSerial": 1611044045370
          },
          "latest": {
            "version": "21.01.04",
            "version_serial": 1611044045370,
            "packages": [
              {
                "version": "21.01.04",
                "type": "",
                "md5": "XXX",
                "fileSize": "FULL_PACKAGE",
                "fileName": "xxx.zip",
                "description": {
                  "headImage": "",
                  "changeLog": {
                    "type": "text",
                    "content": ""
                  }
                },
                "downloadOption": {
                  "mirror": [
                    "https://data1.cdn.com/xxx.zip",
                    "https://data2.cdn.com/xxx.zip"
                  ]
                }
              }
            ]
          },
          "logInfo": {
            "content": "fadsfafsadf",
            "language": "zh_CN",
            "logItems": [
              {
                "logItemId": 2,
                "logId": 2,
                "category": "NEW",
                "categoryDesc": "新增",
                "content": "---...."
              }
            ]
          }
        }
      ]
    }
  }
}
```

是

```json
{
  "header": {
    "code": "200",
    "msg": "请求成功",
    "desc": "http://xxx",
    "location": "CH"
  },
  "body": {
    "code": 220,
    "message": "处理成功",
    "data": {
      "delayTime":"220400"
    }
  }
}
```