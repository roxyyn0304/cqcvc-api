# 超星综合教学管理系统 API 文档

> 学校：重庆城市职业学院 `jw.cqcvc.edu.cn`  
> 系统：超星综合教学管理系统 v3  
> 逆向时间：2026-09-20  
> 说明：本文档基于逆向分析整理，仅供学习与开发参考。请遵守学校规定与相关法律法规，切勿滥用。

---

## 适用范围

本文档的接口基于超星综合教学管理系统的**通用平台代码**逆向，大部分接口**适用于所有使用超星教务的学校**（域名形如 `jw.xxx.edu.cn/admin` 或 `xxx.jw.chaoxing.com/admin`）。

具体情况：

| 项目 | 通用性 | 说明 |
|---|---|---|
| 登录流程 | ✅ 通用 | RSA 公钥、`/admin/login` 路径、参数格式全平台一致 |
| 课表 `getXsdSykb` | ✅ 通用 | 无参数，服务端自动识别，响应结构一致 |
| 成绩 `xsdQueryXscjList` | ✅ 通用 | jqGrid 标准接口，字段一致 |
| 考试 `ajaxXsksList` | ✅ 通用 | jqGrid 标准接口，字段一致 |
| 选课 `listV2` | ✅ 通用 | 但具体可选课程取决于各校排课 |
| 节次时间表 | ⚠️ 各校不同 | 本文档的 10 节次时间仅适用于本校 |
| 学期参数 `startXnxq` | ⚠️ 各校不同 | 编号规则可能因校而异 |
| 数据库标识 `dbname` | ⚠️ 各校不同 | 如 `cqcszyxy` 为本校专属 |
| 学期总数 | ⚠️ 各校不同 | 如 `20` 周可能为 16-22 不等 |
| 菜单结构 / 角色权限 | ⚠️ 各校不同 | 菜单 ID 和权限配置因校定制 |
| 通知 / 工作台 | ✅ 通用 | 路径和格式一致 |
| RSA 公钥 | ✅ 通用 | 所有超星教务共用同一公钥 |

**使用其他学校的同学**：接口路径和参数格式可直接复用，但需注意节次时间、学期编号、菜单 ID 等需要根据自己的学校调整。建议先用浏览器抓包确认关键接口的参数差异。

---

## 适用范围

本文档的接口基于超星综合教学管理系统的**通用平台代码**逆向，大部分接口**适用于所有使用超星教务的学校**（域名形如 `jw.xxx.edu.cn/admin` 或 `xxx.jw.chaoxing.com/admin`）。

具体情况：

| 项目 | 通用性 | 说明 |
|---|---|---|
| 登录流程 | ✅ 通用 | RSA 公钥、`/admin/login` 路径、参数格式全平台一致 |
| 课表 `getXsdSykb` | ✅ 通用 | 无参数，服务端自动识别，响应结构一致 |
| 成绩 `xsdQueryXscjList` | ✅ 通用 | jqGrid 标准接口，字段一致 |
| 考试 `ajaxXsksList` | ✅ 通用 | jqGrid 标准接口，字段一致 |
| 选课 `listV2` | ✅ 通用 | 但具体可选课程取决于各校排课 |
| 节次时间表 | ⚠️ 各校不同 | 本文档的 10 节次时间仅适用于本校 |
| 学期参数 `startXnxq` | ⚠️ 各校不同 | 编号规则可能因校而异 |
| 数据库标识 `dbname` | ⚠️ 各校不同 | 如 `cqcszyxy` 为本校专属 |
| 学期总数 | ⚠️ 各校不同 | 如 `20` 周可能为 16-22 不等 |
| 菜单结构 / 角色权限 | ⚠️ 各校不同 | 菜单 ID 和权限配置因校定制 |
| 通知 / 工作台 | ✅ 通用 | 路径和格式一致 |
| RSA 公钥 | ✅ 通用 | 所有超星教务共用同一公钥 |

**使用其他学校的同学**：接口路径和参数格式可直接复用，但需注意节次时间、学期编号、菜单 ID 等需要根据自己的学校调整。建议先用浏览器抓包确认关键接口的参数差异。

---

## 目录

- [1. 基础信息](#1-基础信息)
- [2. RSA 公钥](#2-rsa-公钥)
- [3. 登录认证](#3-登录认证)
- [4. 通用请求头](#4-通用请求头)
- [5. 接口速查表](#5-接口速查表)
- [6. 课表](#6-课表)
- [7. 成绩](#7-成绩)
- [8. 选课](#8-选课)
- [9. 通知与消息](#9-通知与消息)
- [10. 工作台](#10-工作台)
- [11. 系统](#11-系统)
- [12. 考试安排](#12-考试安排)
- [13. 数据字典](#13-数据字典)
- [14. WAF 与安全校验](#14-waf-与安全校验)
- [15. 注意事项](#15-注意事项)
- [16. 待进一步确认](#16-待进一步确认)
- [17. 免责声明](#17-免责声明)

---

## 1. 基础信息

| 项目 | 内容 |
|---|---|
| Base URL | `https://jw.cqcvc.edu.cn` |
| 认证方式 | 基于 Session Cookie，核心为 `JSESSIONID` |
| 密码加密 | RSA PKCS1v15，前端 JS 加密后 Base64 提交 |
| 请求编码 | `application/x-www-form-urlencoded` 或 `application/json` |
| 验证码 | 非 `*.chaoxing.com` 域名不触发验证码 |
| 错误码 | `ret=0` 成功；`ret=-1` 失败，`msg` 含具体原因 |

---

## 2. RSA 公钥

所有超星综合教务系统共用同一公钥：

`MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQCwC58ftEM2SJHu2H/IIF7DfAi74AtaQSXjGy9PWEb5qD2s0+uh+n1YZBEKDBwwLWZL6T2wVC26pGJuniTOPzGxe5ARTwMATsGKkDTKVNNxkZWxZJS8tuxlJoNP9RD5/u9H3wYJKxcv4VsnH1CwFqlEq7NihIjxvwk7F0omsIbphwIDAQAB`

| 项目 | 说明 |
|---|---|
| 算法 | RSA / PKCS1v15 padding |
| 加密对象 | 明文密码 |
| 输出 | Base64 编码密文 |
| Java/Android | 使用 `javax.crypto.Cipher` + `RSAPublicKeySpec` |

---

## 3. 登录认证

### 3.1 表单登录

```http
POST /admin/login
Content-Type: application/x-www-form-urlencoded

username={学号明文}&password={RSA加密密码}&jcaptchaCode=&rememberMe=1
```

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| username | String | 是 | 学号/工号明文 |
| password | String | 是 | RSA 加密后的密码，Base64 |
| jcaptchaCode | String | 否 | 验证码，非 chaoxing 域名为空 |
| rememberMe | String | 否 | `"1"` 表示自动登录 |

**成功响应**：302 重定向到 `/admin`，`Set-Cookie` 包含 `JSESSIONID`。

**失败响应**：302 重定向回登录页，可能带 `?jcaptchaError=1`，或提示用户名密码错误。

### 3.2 重要 Cookie

| Cookie | 说明 |
|---|---|
| JSESSIONID | **必须**，Session 标识，httpOnly |
| uid | 用户唯一标识 |
| route | 负载均衡路由标识 |
| rememberMe | 加密的持久登录凭证 |
| username | 用户名明文 |
| jw_uf | 用户特征标识 |
| jw_uf_d | 时间戳 |
| jw_uf_u | 数据库标识，如 `cqcszyxy` |
| yysz | 语言标识，`ch`=中文 |

### 3.3 CAS 统一认证登录

```http
GET /admin/caslogin
```

跳转到超星 CAS 认证中心，认证完成后回调 `/admin`，流程与标准 CAS 一致。

### 3.4 学习通扫码登录

```text
iframe src:
https://passport2.chaoxing.com/cloudscanlogin?pcrefer=https://jw.chaoxing.com/beta/admin/api/scanLoginLocal/{hostname}/&customurl=&mobiletip=教务管理系统
```

非 `chaoxing.com` 域名时走此路径，通过学习通 APP 扫码完成认证。

---

## 4. 通用请求头

所有 AJAX 请求建议携带：

```http
# 换成你本机浏览器的真实 UA
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/154.0.0.0 Safari/537.36 Edg/154.0.0.0
Accept: application/json, text/javascript, */*; q=0.01
X-Requested-With: XMLHttpRequest
Accept-Language: zh-CN,zh;q=0.9
```

部分接口需要：

- `Content-Type: application/json`：如 `listV2`
- `Content-Type: application/x-www-form-urlencoded`：如课程表、成绩查询

---

## 5. 接口速查表

| 模块 | 方法 | 路径 | 说明 |
|---|---|---|---|
| 登录 | POST | `/admin/login` | 表单登录 |
| 登录 | GET | `/admin/caslogin` | CAS 统一认证 |
| 登录 | iframe | `https://passport2.chaoxing.com/cloudscanlogin?...` | 学习通扫码登录 |
| 课表 | POST | `/admin/getXsdSykb` | 获取学生课表，无参数 |
| 课表 | GET | `/admin/getCurrentPkZc` | 当前排课周次列表 |
| 课表 | GET | `/admin/api/getZclistByXnxq` | 当前第几周 |
| 课表 | GET | `/admin/api/jcsj/xqsj/getXqList` | 学期列表 |
| 课表 | GET | `/admin/api/getXlzc` | 学历层次 |
| 成绩 | POST | `/admin/xsd/xsdcjcx/xsdQueryXscjList?fxbz=0&gridtype=jqgrid` | 成绩查询 jqGrid |
| 成绩 | GET | `/admin/xsd/xsdzgcjcx/getXspjxfjd` | 平均学分绩点 |
| 成绩 | GET | `/admin/xsd/xyjc/ajaxTable` | 学业成绩表格 |
| 成绩 | GET | `/admin/xsd/xsdcjcx/getCurrentXnxq` | 当前学年学期 |
| 成绩 | GET | `/admin/xsd/xsdcjcx/qbcjcx` | 成绩页面框架 |
| 选课 | POST | `/admin/xsd/xk/listV2` | 可选课程列表，JSON |
| 选课 | POST | `/admin/getMenuList` | 菜单列表 |
| 选课 | POST | `/admin/encLogin` | 加密跳转登录 |
| 通知 | GET | `/admin/getWdxxList` | 未读消息列表 |
| 通知 | GET | `/admin/getNoticeType` | 通知类型 |
| 通知 | POST | `/admin/system/tzsjx/ajaxList` | 通知列表 |
| 通知 | GET | `/admin/system/tzsjx/showdetail?id={通知ID}` | 通知详情 |
| 工作台 | GET | `/admin/workcenter/homepage/list?center=msg` | 消息 |
| 工作台 | GET | `/admin/workcenter/homepage/list?center=sp` | 审批 |
| 工作台 | GET | `/admin/workcenter/homepage/list?center=download` | 下载 |
| 工作台 | GET | `/admin/workcenter/homepage/list?center=kf` | 客服 |
| 工作台 | GET | `/admin/workcenter/download/getTotalRedCount` | 下载中心红点数 |
| 工作台 | GET | `/admin/workcenter/message/getTotalRedCount` | 消息中心红点数 |
| 系统 | GET | `/admin/system/usermanage/roleCache` | 角色缓存 |
| 系统 | GET | `/admin/system/usermanage/setLanguage` | 设置语言 |
| 系统 | GET | `/admin/system/usermanage/setCyjs` | 常用角色管理 |
| 系统 | GET | `/admin/caslogout` | CAS 登出 |
| 系统 | GET | `/admin/twiceAuth?type=1` | 二次认证 |
| 系统 | POST | `/admin/getDictByGroupCode` | 字典查询 |
| 系统 | GET | `/admin/system/gridfield/getTranslation?yysz=ch` | 字段翻译 |
| 系统 | POST | `/admin/system/export/urlExport` | 导出 |
| 考试 | POST | `/admin/xsd/kwglXsdKscx/ajaxXsksList?gridtype=jqgrid` | 考试安排查询 |

---

## 6. 课表

### 6.1 获取学生课表

```http
POST /admin/getXsdSykb
Content-Type: application/x-www-form-urlencoded
X-Requested-With: XMLHttpRequest
```

**无需参数**，服务端根据当前 Session 自动识别学生。

**响应格式**：

```json
{
  "ret": 0,
  "msg": "获取成功",
  "data": {
    "sjk": [
      {
        "jxbzc": "-",
        "tmc": "-",
        "kcmc": "-",
        "type": "-",
        "xkrs": "-",
        "zcstr": "-"
      }
    ],
    "jcKcxx": [
      {
        "jc": "1",
        "kssj": "8:20",
        "jssj": "9:05",
        "kbxx": [
          {
            "yzxq": "1",
            "kcxx": [
              {
                "kcmc": "大学英语",
                "teacher": "王五",
                "classroom": "文华楼119"
              }
            ]
          }
        ]
      }
    ],
    "kbxjxsmc": "1"
  }
}
```

**数据结构说明**：

- `jcKcxx`：节次课程信息数组，共 10 节课，早 8:20 到晚 21:10。
- `kbxx`：每节课下挂 7 天，`yzxq` 1=周一，7=周日。
- `kcxx`：该时段该天的课程，可能 1 门或多门。
- 空课时课程名、教师、教室均为 `"-"`。

### 6.2 节次时间表

| 节次 | 开始 | 结束 |
|---|---|---|
| 1 | 8:20 | 9:05 |
| 2 | 9:15 | 10:00 |
| 3 | 10:30 | 11:15 |
| 4 | 11:25 | 12:10 |
| 5 | 14:00 | 14:45 |
| 6 | 14:55 | 15:40 |
| 7 | 16:10 | 16:55 |
| 8 | 17:05 | 17:50 |
| 9 | 19:30 | 20:15 |
| 10 | 20:25 | 21:10 |

### 6.3 当前排课周次列表

```http
GET /admin/getCurrentPkZc
```

响应：

```json
{
  "ret": 0,
  "msg": "获取成功",
  "data": ["1","2","3","4","5","6","7","8","9","10",
           "11","12","13","14","15","16","17","18","19","20"]
}
```

### 6.4 当前是第几周

```http
GET /admin/api/getZclistByXnxq
```

响应：

```json
{
  "ret": 0,
  "msg": "操作成功",
  "data": {
    "dqzc": "3"
  }
}
```

### 6.5 学期列表

```http
GET /admin/api/jcsj/xqsj/getXqList
```

响应：

```json
{
  "ret": 0,
  "data": [
    {
      "id": "...",
      "xqh": "1",
      "xqmc": "2026-2027-1",
      "dataXnxq": "...",
      "currentRoleId": "...",
      "currentJsId": "...",
      "userRoleId": "...",
      "dataAuth": "...",
      "xqdz": "...",
      "xqym": "...",
      "xqdwfzrjgh": "..."
    }
  ]
}
```

### 6.6 学历层次

```http
GET /admin/api/getXlzc
```

响应：

```json
{
  "ret": 0,
  "data": {
    "xlzc": "..."
  }
}
```

---

## 7. 成绩

### 7.1 成绩查询（jqGrid 数据接口）

```http
POST /admin/xsd/xsdcjcx/xsdQueryXscjList?fxbz=0&gridtype=jqgrid
Content-Type: application/x-www-form-urlencoded
X-Requested-With: XMLHttpRequest

page.pn={页码}&page.size={每页条数}&startXnxq={起始学期}&endXnxq={结束学期}&sort=xnxq&order=desc
```

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| fxbz | Int | 是 | 辅修标志，0=主修，1=辅修，9=微专业 |
| gridtype | String | 是 | 固定 `jqgrid` |
| page.pn | Int | 是 | 页码，从 1 开始 |
| page.size | Int | 否 | 每页条数，默认 20 |
| startXnxq | String | 是 | 起始学年学期，如 `001` |
| endXnxq | String | 是 | 结束学年学期，如 `001` |
| sort | String | 否 | 排序字段，默认 `xnxq` |
| order | String | 否 | `desc` 或 `asc` |

**响应格式**：

```json
{
  "ret": 0,
  "page": 1,
  "rows": 20,
  "total": 10,
  "totalPages": 1,
  "results": [
    {
      "id": "xxx",
      "xnxq": "2026-2027-1",
      "kcmc": "高等数学",
      "xf": "4.0",
      "kcxz": "01",
      "kclx": "99",
      "ksxs": "1",
      "kcgs": "1",
      "kcsx": "1",
      "kclb": "01",
      "xdxz": "1",
      "zhcj": "85.0",
      "fxcj": "...",
      "hdxf": "4.0",
      "jd": "3.5",
      "sfbk": "0",
      "tscjzwmc": "...",
      "cjfxms": "...",
      "cjlrjsxm": "...",
      "kkyxmc": "...",
      "xh": "1145141919",
      "xqxh": "..."
    }
  ]
}
```

### 7.2 平均学分绩点

```http
GET /admin/xsd/xsdzgcjcx/getXspjxfjd
```

响应：

```json
{
  "ret": 0,
  "msg": "操作成功",
  "data": "暂无"
}
```

### 7.3 学业成绩表格

```http
GET /admin/xsd/xyjc/ajaxTable
```

响应：数组或空数组 `[]`，需要额外参数，具体参数需进一步抓包。

### 7.4 当前学年学期

```http
GET /admin/xsd/xsdcjcx/getCurrentXnxq
```

响应：

```json
{
  "ret": 0,
  "data": "2026-2027-1"
}
```

### 7.5 成绩页面框架 URL

```http
GET /admin/xsd/xsdcjcx/qbcjcx
```

返回 HTML 页面，内嵌 jqGrid 表格配置，通过 iframe 加载。

---

## 8. 选课

### 8.1 可选课程列表

```http
POST /admin/xsd/xk/listV2
Content-Type: application/json
X-Requested-With: XMLHttpRequest

{}
```

当前非选课期时：

```json
{
  "ret": -1,
  "msg": "没有可选的教学班",
  "data": {
    "xsxh": "1145141919",
    "xkxnxq": "2026-2027-1",
    "cxxkwkcts": "暂无课程",
    "dbname": "cqcszyxy",
    "xsxm": "张三",
    "xsdxksfycyxrl": "0",
    "xsxb": "1",
    "showGoHome": "1"
  }
}
```

选课期时预计返回课程列表，待进一步抓包确认完整字段。

### 8.2 菜单列表

```http
POST /admin/getMenuList
Content-Type: application/x-www-form-urlencoded

id={菜单模块ID}
```

返回左侧菜单结构，用于获取各功能模块的 URL 和权限。

### 8.3 加密跳转登录

```http
POST /admin/encLogin
Content-Type: application/x-www-form-urlencoded

id={页面ID}
```

生成加密的一次性访问 URL，用于打开子系统页面。

---

## 9. 通知与消息

### 9.1 未读消息列表

```http
GET /admin/getWdxxList
```

响应：JSON 数组，每项包含 `id`、`title`、`releasedate`、`sfyd`、`sfzd`。

### 9.2 通知类型

```http
GET /admin/getNoticeType
```

### 9.3 通知列表

```http
POST /admin/system/tzsjx/ajaxList
Content-Type: application/x-www-form-urlencoded

gridtype=jqgrid&queryFields=id,dqstatus,collectstatus,title,content,releaseDate,&_search=false&page.size=500&page.pn=1&sort=id&order=asc
```

### 9.4 通知详情

```http
GET /admin/system/tzsjx/showdetail?id={通知ID}
```

---

## 10. 工作台

```http
GET /admin/workcenter/homepage/list?center=msg      // 消息
GET /admin/workcenter/homepage/list?center=sp       // 审批
GET /admin/workcenter/homepage/list?center=download // 下载
GET /admin/workcenter/homepage/list?center=kf       // 客服
GET /admin/workcenter/download/getTotalRedCount     // 下载中心红点数
GET /admin/workcenter/message/getTotalRedCount      // 消息中心红点数
```

---

## 11. 系统

```http
GET  /admin/system/usermanage/roleCache
GET  /admin/system/usermanage/setLanguage
GET  /admin/system/usermanage/setCyjs
GET  /admin/caslogout
GET  /admin/twiceAuth?type=1
POST /admin/getDictByGroupCode
GET  /admin/system/gridfield/getTranslation?yysz=ch
POST /admin/system/export/urlExport
```

### 11.1 字典查询

```http
POST /admin/getDictByGroupCode
Content-Type: application/x-www-form-urlencoded

groupCode={字典组名}
```

### 11.2 字段翻译

```http
GET /admin/system/gridfield/getTranslation?yysz=ch
```

### 11.3 导出

```http
POST /admin/system/export/urlExport
```

---

## 12. 考试安排

### 12.1 考试安排查询（jqGrid 数据接口）

```http
POST /admin/xsd/kwglXsdKscx/ajaxXsksList?gridtype=jqgrid
Content-Type: application/x-www-form-urlencoded; charset=UTF-8
X-Requested-With: XMLHttpRequest
```

**请求参数**：

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| queryFields | String | 是 | 固定 `id,kspcmc,xh,xm,kcmc,kssj,jsmc,ksfs,ksxs,zwh,bkcs,bz,rwbz,` |
| gridtype | String | 是 | 固定 `jqgrid` |
| page.pn | Int | 是 | 页码，从 1 开始 |
| page.size | Int | 否 | 每页条数，默认 50 |
| _search | Boolean | 是 | 固定 `false` |
| sort | String | 否 | 默认 `bjdm asc,zwh asc,id` |
| order | String | 否 | `asc` 或 `desc` |
| nd | Long | 否 | 时间戳 |

**请求示例**：

```text
queryFields=id,kspcmc,xh,xm,kcmc,kssj,jsmc,ksfs,ksxs,zwh,bkcs,bz,rwbz,&_search=false&nd=1789879534017&page.size=50&page.pn=1&sort=bjdm+asc,zwh+asc,id&order=asc
```

**响应格式**：

```json
{
  "ret": 0,
  "msg": "ok",
  "page": 1,
  "rows": 50,
  "total": 0,
  "totalPages": 0,
  "results": [
    {
      "id": "xxx",
      "kspcmc": "期末考试",
      "xh": "1145141919",
      "xm": "张三",
      "kcmc": "高等数学",
      "kssj": "2027-01-10 09:00",
      "jsmc": "文华楼427",
      "ksfs": "1",
      "ksxs": "1",
      "zwh": "01",
      "bkcs": "0",
      "bz": "",
      "rwbz": ""
    }
  ]
}
```

**字段说明**：

| 字段 | 含义 |
|---|---|
| id | 主键 |
| kspcmc | 考试场次名称 |
| xh | 学号 |
| xm | 姓名 |
| kcmc | 课程名称 |
| kssj | 考试时间 |
| jsmc | 教室/考场名称 |
| ksfs | 考试方式 |
| ksxs | 考试形式，1=考试，2=考查，3=其他 |
| zwh | 座位号 |
| bkcs | 补考次数，0=正考 |
| bz | 备注 |
| rwbz | 任务标志 |

---

## 13. 数据字典

### 13.1 课程性质 `kcxz`

| 代码 | 含义 |
|---|---|
| 01 | 公共必修课 |
| 02 | 第二课堂 |
| 03 | 学科基础课 |
| 04 | 专业核心课 |
| 05 | 专业拓展课 |
| 06 | 顶岗实习 |
| 07 | 军训 |
| 08 | 实习实训 |
| 12 | 专业基础课 |
| 17 | 通识选修课 |
| 99 | 公共选修课 |

### 13.2 课程属性 `kcsx`

| 代码 | 含义 |
|---|---|
| 1 | 必修 |
| 2 | 限选 |
| 3 | 任选 |
| 4 | 辅修 |
| 5 | 实践 |
| 6 | 双必 |
| 7 | 双选 |
| 8 | 通选 |
| 9 | 其他 |
| 10 | 公选 |

### 13.3 是否补考 `sfbk`

| 代码 | 含义 |
|---|---|
| 0 | 正考 |
| 1 | 补考 |
| 2 | 缓考补考 |
| 3 | 清考 |

### 13.4 修读性质 `xdxz`

| 代码 | 含义 |
|---|---|
| 1 | 初修 |
| 2 | 重修 |

### 13.5 考试形式 `ksxs`

| 代码 | 含义 |
|---|---|
| 1 | 考试 |
| 2 | 考查 |
| 3 | 其他 |

### 13.6 错误码

| ret | 含义 |
|---|---|
| 0 | 成功 |
| -1 | 失败，`msg` 字段含具体原因 |

---

## 14. WAF 与安全校验

- 域名 `jw.cqcvc.edu.cn` 启用了 **Web 应用防火墙**，基于 IP 特征 + UA 特征。
- `/admin/xsd/xk` 等前端 SPA 页面直接 curl 访问可能返回 502，被 WAF 拦截。
- JSON API 接口，如 `/admin/xsd/xk/listV2`，通常不受影响。
- 建议使用完整浏览器 UA 和合理的请求间隔。

---

## 15. 注意事项

1. **Session 过期**：`JSESSIONID` 有效期不确定，长时间无操作后需重新登录。
2. **Cookie 绑定**：Session 绑定 IP + UID，换 IP 后 Session 可能失效。
3. **学年学期参数**：`startXnxq` / `endXnxq` 格式为 `001`（第一学期）、`002`（第二学期），而非完整年份。
4. **课程列表为空**：非选课时间段，`listV2` 返回“没有可选的教学班”，属正常现象。
5. **成绩为空**：新生第一学期可能暂无成绩数据。
6. **Content-Type 区分**：
   - `listV2` 使用 `application/json`
   - 课表、成绩查询等使用 `application/x-www-form-urlencoded`

---

## 16. 待进一步确认

1. `listV2` 在选课期返回的完整课程字段。
2. `/admin/xsd/xyjc/ajaxTable` 所需完整参数。
3. `/admin/system/export/urlExport` 导出参数。
4. 学习通扫码登录回调细节。
5. `twiceAuth` 二次认证触发条件。
6. CAS 登录完整回调链路与票据校验。

---

## 17. 免责声明

本文档仅用于技术学习、接口整理与开发参考。  
请勿将相关接口用于未经授权的访问、批量爬取、账号盗用或其他违反学校规定与法律法规的行为。  
使用本文档产生的任何后果由使用者自行承担。
