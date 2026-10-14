# 超星综合教学管理系统 API 文档

> 学校：重庆城市职业学院 `jw.cqcvc.edu.cn`  
> 系统：超星综合教学管理系统 v3  
> 说明：本文档 = 逆向初稿 + 真实账号实测 + HttpWatch 抓包交叉验证，仅供学习与开发参考；请遵守学校规定与相关法律法规，切勿滥用。

**修订史**：2026-09-20 逆向 → 09-23 真实账号实测修订 → 10-13 两轮抓包交叉验证（全量 456 条目 + 选课专项 112 条目）。  
**全文接口的验证状态见 §1 的「验证状态总览」**；凡被后续抓包推翻的旧结论，正文中以 `📌 修正` 标注并给出实测依据。

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
| 选课 `/admin/xsd/xk/*` | ⚠️ 部分通用 | 流程与参数名通用（`listV2`→`listjxbDataV2`→`xsdXkV2`/`xsdTkV2`），但批次 `mapsList`、搜索配置 `xsdxkzdjson`、可选课程由各校排课决定 |
| 节次时间表 | ⚠️ 各校不同 | 本校 10 节次；可从 `getZclistByXnxq` 的 `data.jcsjszList` 动态取，**不必硬编码**（§6.2/§6.4） |
| 学期参数 `startXnxq` | ⚠️ 各校不同 | 本校 `001`（入学以来）与 `2026-2027-1` 均可；其余接口的 `xnxq` 用完整串（§7.1、§15 第 3 条） |
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
- [17. 学生课表真数据源 sdpkkbList（周次）](#17-学生课表真数据源-sdpkkblist周次)
- [18. 开学日期与当前周推算](#18-开学日期与当前周推算)
- [19. 未收录接口（抓包实测）](#19-未收录接口抓包实测)
- [20. 免责声明](#20-免责声明)

---

## 1. 基础信息

| 项目 | 内容 |
|---|---|
| Base URL | `https://jw.cqcvc.edu.cn` |
| 认证方式 | 基于 Session Cookie，核心为 `JSESSIONID` |
| 密码加密 | RSA PKCS1v15，前端 JS 加密后 Base64 提交 |
| 请求编码 | `application/x-www-form-urlencoded` 或 `application/json` |
| 验证码 | 非 `*.chaoxing.com` 域名不触发验证码 |
| 错误码 | `ret=0` 成功；`ret!=0` 需结合 `msg` 判断，见 §13.6 |

### 验证状态总览

| 模块 | 状态 | 依据 |
|---|---|---|
| §3 登录认证 | ⚠️ 仅逆向 | 两轮抓包均未录到登录流量 |
| §6 / §17 课表 | ✅ 已验证 | `queryKbForXsd`、`sdpkkbList`、`getbzxx`、`getZclistByXnxq`、`getXqList`、`getXlzc` 全部实测命中 |
| §7 成绩 | ✅ 已验证 | `xsdQueryXscjList`（`fxbz=0/1/9`）、`getCurrentXnxq`、`qbcjcx`、`getPjxtsz` |
| §8 选课 | ✅ 已验证 | 302 入口 → `listV2` → `listjxbDataV2` → `xsdXkV2`/`xsdTkV2` → 选课字典 |
| §12 考试 | ✅ 已验证 | `ajaxXsksList`、`checkWfwSfpj`、`getKspc` |
| §11 系统 | ✅ 大部分已验证 | `getTranslation`、`getDictByGroupCode`、`gridfield/*` 实测；`roleCache`/`setLanguage` 等仅逆向 |
| §9 通知 / §10 工作台 | ⚠️ 部分 | 路径已确认，响应体未落盘（红点接口仅验证到 200） |
| §13 数据字典 | ⚠️ 未复验 | 两轮抓包 `total=0` 无数据行可比对；且 `sdpkkbList` 返回中文而非代码（§15 第 7 条） |
| §14 WAF | ✅ 已验证 | 09-23 换 UA 实测 200 / 403 |
| 其余待办 | — | 见 §16 待进一步确认 |

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

**实测补充**：推荐先 `GET /admin/login`（获取 `uid`/`route` Cookie 与隐藏域 `jcaptchaCode`/`rememberMe`）再 POST；密码错误时也可能**200 重新渲染登录页**（响应体内含“密码错误”标记），并非一律 302；未登录访问业务接口返回 **303** 跳转登录页；成功判定建议用“最终 URL 不以 `login` 结尾”。

### 3.1.1 登录后的重定向链（2026-10-14 实测，重要）

登录**不是一次跳转，而是三跳链**，逐跳实测如下：

| 跳 | 请求 | 响应 | Location |
|---|---|---|---|
| 1 | `POST /admin/login` | 302 | `/admin` |
| 2 | `GET /admin` | 302 | `/frontend/views/student/xkPage/mobile/index.html?sfkqcas=1&dbname=cqcszyxy&sfzzpj=false&qfxssfyxxk=1&xknew=1&enc=<32 位>&username=<学号>&sfksxk=true&sfksxksq=false` |
| 3 | `GET /frontend/views/student/xkPage/mobile/index.html?...` | 200 | —（标题「学生选课」，33,802 字节 HTML） |

三条实践结论：

1. **判定登录成功不要只看第一跳**——第一跳的 302 只说明凭据被接受，真正的落地页在第 3 跳。
2. **任何自建 HTTP 客户端都要放行 `/frontend` 前缀**。落地页在 `/admin` 之外，若客户端只白名单 `/admin*`，会在第 3 跳被自己的范围检查拦下并抛错（表象为「登录失败/网络错误」，但账号与密码其实都对）。2026-10-14 排查的插件真机登录失败正是此因（修复见插件 v1.1.7）。
3. 第 2 跳的 query 携带 `enc`、`username`（学号明文）、`dbname`，属敏感参数，日志与报错截图中应脱敏。

**Cookie 实测**（2026-10-14，登录成功后）：`uid`、`route`、`initPass`、`defaultPass`、`inactivePass`、`rememberMe`、`puid`、`username`、`jw_uf`、`jw_uf_d`、`jw_uf_u` —— **未见 `JSESSIONID`**，会话疑似由 `puid` + `jw_uf*` 承载（见 §3.2 的修订标注）。

### 3.2 重要 Cookie

| Cookie | 说明 |
|---|---|
| JSESSIONID | 📌 修正（2026-10-14）：早期结论称「必须」，但实测登录成功后的 `Set-Cookie` 中**未出现该 Cookie**，会话疑似由 `puid` + `jw_uf*` 承载，待进一步复核 |
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

> 状态图例：✅ 已抓包实测 · ⚠️ 仅逆向、未复验 · 📎 仅页面 JS 引用、未产生流量

| 模块 | 方法 | 路径 | 说明 | 状态 |
|---|---|---|---|---|
| 登录 | POST | `/admin/login` | 表单登录（§3.1） | ⚠️ |
| 登录 | GET | `/admin/caslogin` | CAS 统一认证（§3.3） | ⚠️ |
| 登录 | iframe | `https://passport2.chaoxing.com/cloudscanlogin?...` | 学习通扫码登录（§3.4） | ⚠️ |
| 登录 | GET | `/admin/caslogout` | 登出 | ⚠️ |
| 课表 | POST | `/admin/getXsdSykb` | 学生课表网格，无参数（§6.1） | ⚠️ |
| 课表 | GET | `/admin/getCurrentPkZc` | 当前排课周次列表（§6.3） | ⚠️ |
| 课表 | GET | `/admin/api/getZclistByXnxq?xnxq=…&xqid=…` | 当前周 `dqzc` + 节次表 `jcsjszList`（§6.4） | ✅ |
| 课表 | GET | `/admin/api/jcsj/xqsj/getXqList` | 校区列表（§6.5） | ✅ |
| 课表 | GET | `/admin/pkgl/xskb/queryKbForXsd?xnxq=…` | 课表页，隐藏域 `xhid`/`xqdm`（§17.1） | ✅ |
| 课表 | GET | `/admin/pkgl/xskb/sdpkkbList?…` | **课表周次真数据源**（§17.2） | ✅ |
| 课表 | POST | `/admin/pkgl/xskb/getbzxx` | 课表备注，`xnxq`+`xhid`（§17.2） | ✅ |
| 课表 | GET | `/admin/api/getXlzc` | 学历层次（§6.6） | ✅ |
| 成绩 | POST | `/admin/xsd/xsdcjcx/xsdQueryXscjList?fxbz=0\|1\|9&gridtype=jqgrid` | 成绩查询 jqGrid（§7.1） | ✅ |
| 成绩 | GET | `/admin/xsd/xsdcjcx/qbcjcx[?fxtype=1\|9]` | 成绩页 / 辅修 / 微专业（§7.5） | ✅ |
| 成绩 | GET | `/admin/xsd/xsdcjcx/getCurrentXnxq` | 当前学年学期（§7.4） | ✅ |
| 成绩 | GET | `/admin/xsd/xsdzgcjcx/getXspjxfjd` | 平均学分绩点（§7.2） | ⚠️ |
| 成绩 | GET | `/admin/xsd/xyjc/ajaxTable` | 学业成绩表格（§7.3） | ⚠️ |
| 选课 | GET | `/admin/xsd/xk` | 选课页入口，**302 → `/frontend/views/student/xkPage/index.html`**（§8.1） | ✅ |
| 选课 | POST | `/admin/xsd/xk/listV2` | 选课初始化：批次 `mapsList` / `pcencs` / 字典，无请求体（§8.2） | ✅ |
| 选课 | POST | `/admin/xsd/xk/listjxbDataV2` | 教学班列表，`from`/`pcid`/`pcenc`/筛选项（§8.3） | ✅ |
| 选课 | POST | `/admin/xsd/xk/xsdXkV2` | 选课，`jxbid`+`pcid` → `选课成功`（§8.4） | ✅ |
| 选课 | POST | `/admin/xsd/xk/xsdTkV2` | 退课，`jxbid`+`pcid` → `退课成功`（§8.4） | ✅ |
| 选课 | POST | `/admin/xsd/xkprocess/student/getDictByGroupCode` | 选课模块字典，`groupCode=ZD_KCXZ/ZD_KCLB`（§8.6） | ✅ |
| 选课 | POST | `/admin/getMenuList` | 左侧菜单，`id={cmid}`（§8.7，响应体未落盘） | ⚠️ |
| 选课 | POST | `/admin/encLogin` | 加密跳转登录（§8.8） | ⚠️ |
| 选课 | GET | `/admin/xkgl/xkrwgl/ajaxListDatas` | 选课任务管理网格，**管理端**（见 §19.6） | 📎 |
| 评教 | GET | `/admin/xsd/pj/xspj/getPjxtsz` | 评教开关设置（§19.1） | ✅ |
| 检查 | POST | `/admin/xsd/kcapcx/checkSfkc` | 开课状态检查，`xnxq`+`type`（§19.2） | ✅ |
| 检查 | POST | `/admin/xsd/pkgl/xskb/checkWfwSfpj` | 评教前置检查·课表页（§19.2） | ✅ |
| 检查 | POST | `/admin/xsd/kwglXsdKscx/checkWfwSfpj` | 评教前置检查·考试页（§19.2） | ✅ |
| 考试 | POST | `/admin/xsd/kwglXsdKscx/ajaxXsksList?gridtype=jqgrid` | 考试安排 jqGrid（§12.1） | ✅ |
| 考试 | GET | `/admin/xsd/kwglXsdKscx/getKspc?xnxq={学期}` | 考试批次，对应 §12 的 `pcid`（§19.5） | ✅ |
| 通知 | GET | `/admin/getWdxxList` | 未读消息列表（§9.1） | ⚠️ |
| 通知 | GET | `/admin/getNoticeType` | 通知类型（§9.2） | ⚠️ |
| 通知 | POST | `/admin/system/tzsjx/ajaxList` | 通知列表（§9.3） | ⚠️ |
| 通知 | GET | `/admin/system/tzsjx/showdetail?id={通知ID}` | 通知详情（§9.4） | ⚠️ |
| 工作台 | GET | `/admin/workcenter/homepage/list?center=msg\|sp\|download\|kf` | 消息 / 审批 / 下载 / 客服（§10） | ⚠️ |
| 工作台 | GET | `/admin/workcenter/{download,message}/getTotalRedCount` | 下载 / 消息中心红点数（§10） | ⚠️ |
| 系统 | POST | `/admin/getDictByGroupCode` | 字典查询（§11.1） | ✅ |
| 系统 | GET/POST | `/admin/system/gridfield/getTranslation[?yysz=ch]` | 字段翻译，两种方法同响应（§11.2） | ✅ |
| 系统 | GET | `/admin/api/getSetting` | 系统开关，响应键是 `extend`（§19.3） | ✅ |
| 系统 | POST | `/admin/system/export/urlExport` | 导出（§11.3） | ⚠️ |
| 系统 | GET | `/admin/system/usermanage/roleCache` | 角色缓存 | ⚠️ |
| 系统 | GET | `/admin/system/usermanage/setLanguage` | 设置语言 | ⚠️ |
| 系统 | GET | `/admin/system/usermanage/setCyjs` | 常用角色管理 | ⚠️ |
| 系统 | GET | `/admin/twiceAuth?type=1` | 二次认证 | ⚠️ |
| 字段 | GET | `/admin/system/gridfield/toFieldBind?gridId=…&gridurl=…` | 「表格字段设置」页 HTML（§19.4） | ✅ |
| 字段 | POST | `/admin/system/gridfield/queryFieldByGridId?gridId=…&gridurl=…` | 查询网格字段配置，请求 `{}`（§19.4） | ✅ |
| 字段 | POST | `/admin/system/gridfield/insertOrUpdate` | 保存网格字段配置（§19.4） | ✅ |

> 工作台的 `homepage/list` 四个 `center` 与两个红点接口在两轮抓包里都出现过（红点接口实测 200），但响应体未落盘，故仍标 ⚠️（见 §16）。

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

> **实测**：`kcxx` 课程对象仅含 `kcmc / teacher / classroom`，**不含周次字段**；`sjk` 为孤儿行（其 `zcstr` 恒为 `-`，与网格课程无法关联）。课表真实周次来自另一接口，见 [§17 学生课表真数据源 sdpkkbList](#17-学生课表真数据源-sdpkkblist周次)。

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

> 📌 上表不必硬编码：`getZclistByXnxq`（§6.4）的 `data.jcsjszList` 就是它的动态来源，逐条给出 `jc / kssj / jssj / xnxq / xqid / djs / sjd`，按校区+学期返回 10 条，各校可直接取用。

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

### 6.4 当前是第几周（含节次时间表）

```http
GET /admin/api/getZclistByXnxq?xnxq={学期}&role=&userId=&xqid={校区ID}
```

| 参数 | 说明 |
|---|---|
| xnxq | 学年学期，如 `2026-2027-1` |
| role / userId | 页面原样带的空值，可留空 |
| xqid | 校区 ID，与 §6.5 的 `id`、`sdpkkbList` 的 `xqdm` 同源，如 `00001` |

> 📌 修正：本接口**带参数**（09-23 曾误记为"无参数"），页面实际发送上面 4 个。

响应（实测原文节选）：

```json
{
  "ret": 0,
  "msg": "操作成功",
  "data": {
    "jcsjszList": [
      {"kssj": "8:20", "jssj": "9:05", "jc": "1", "xnxq": "2026-2027-1", "xqid": "00001",
       "pkjcszid": "c0b9cff6c7d844edb370d048df868dc7", "djs": "1", "sjd": "sw", "sjdtj": 0, "sjdxh": 0},
      { "…": "共 10 条，对应 1–10 节次" }
    ],
    "zclist": [
      {"zc": "1", "minrq": "2026-09-05 00:00:00", "maxrq": "2026-09-06 00:00:00", "rqfw": "09.05-09.06"},
      { "…": "共 20 条，每周一条" }
    ],
    "jsxq": {"xnxq": "2026-2027-1", "xqid": "00001", "pkzxzc": "1", "pkzdzc": "19", "mzpkts": "7", "mtbhxjc": "10", "…": ""},
    "dqzc": "7",
    "kbxssj": "1"
  }
}
```

| 字段 | 含义 |
|---|---|
| `jcsjszList` | **节次时间表**（§6.2 那张硬编码表的动态来源），按校区+学期返回，10 条 |
| `zclist` | **权威周历**：每周 `zc`（第几周）+ `minrq`/`maxrq`（周一~周日日期）+ `rqfw`（`09.05-09.06` 形态），20 条——开学日、周次高亮直接取它，**比 §18 的公式推算准** |
| `jsxq` | 排课设置：`xnxq`/`xqid`/`pkjcszid`/`pkzxzc`（排课起始周）/`pkzdzc`（排课终止周）/`mzpkts`（每周调停次数）/`mtbhxjc`（每天不排课节次）/`kbxssj` |
| `dqzc` | 当前第几周（实测 `7`），用于 §18 公式兜底 |
| `kbxssj` | 上课显示时间开关，字符串 `"1"` / `"0"` |

> 📌 **2026-10-14 实测：`xqid` 是必填开关**——不带它只回 `{"ret":0,"data":{"dqzc":"7"}}`，带了才有 `jcsjszList` / `zclist` / `jsxq` / `kbxssj` 完整 5 键。

### 6.5 校区列表

```http
GET /admin/api/jcsj/xqsj/getXqList
```

**返回的是校区列表**（`xq` = 校区而非学期；`xqdz` 是**学校地址**、不是日期范围）：

```json
{
  "ret": 0,
  "data": [
    {
      "id": "00001",
      "xqmc": "永川",
      "xqdz": "重庆市永川区兴龙大道1999号"
    }
  ]
}
```

> - `id`（如 `00001`）可直接用作 `sdpkkbList` 的 `xqdm`、`getZclistByXnxq` 的 `xqid`。
> - **权威学期列表**在成绩页 `qbcjcx` 的 `select#startXnxq`：首项 `<option value="001">入学以来</option>`，其后是 `2026-2027-2`、`2026-2027-1`…；两种形态都是 §7.1 的合法取值，页面默认取 `001`。
> - `data[0]` 除三要素外还带会话上下文字段（`currentUserId`、`currentUserName`（学号）、`currentRoleId`、`dataXnxq`、`xqh`、`lssjjg`…），上面的示例只列了校区三要素。

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

queryFields=id%2Cxnxq%2Ckcmc%2Cxf%2Ckcxz%2Ckclx%2Cksxs%2Ckcgs%2Cxdxz%2Ckclb%2Ccjfxms%2Czhcj%2Chdxf%2Ctscjzwmc%2Csfbk%2Ccjlrjsxm%2Ckcsx%2Cfxcj%2Ckkyxmc%2C
&_search=false&nd={时间戳}&page.size=20&page.pn=1&sort=xnxq+desc%2Cid&order=desc
&startXnxq=001&endXnxq=001
&kcmc=&sfjg=&kcxz=&kclx=&kclb=&kcgs=
&query.startXnxq%7C%7C=001&query.endXnxq%7C%7C=001&query.kcmc%7C%7C=&query.sfjg%7C%7C=&query.kcxz%7C%7C=&query.kclx%7C%7C=&query.kclb%7C%7C=&query.kcgs%7C%7C=
```

> 上面是抓包原文，同一页签 5 次请求完全一致。

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| fxbz | Int | 是 | 辅修标志，0=主修，1=辅修，9=微专业（页面 `qbcjcx?fxtype=1/9` 即对应 1/9，见 §7.5） |
| gridtype | String | 是 | 固定 `jqgrid` |
| queryFields | String | 是 | 要返回的列，逗号 URL 编码，末尾保留一个 `%2C`。**不是固定值**：抓包中 `fxbz=0` 的请求以 `…cjlrjsxm%2Ckcsx%2Cfxcj%2Ckkyxmc%2C` 结尾，而 `fxbz=1`、`fxbz=9` 止于 `…cjlrjsxm%2C`；列集合会随「表格字段设置」（§19.4 `insertOrUpdate`）变化，别写死 |
| `_search` | Bool | 是 | 固定 `false` |
| nd | Long | 是 | 前端时间戳，任意当前毫秒值 |
| page.pn | Int | 是 | 页码，从 1 开始 |
| page.size | Int | 否 | 每页条数，默认 20（响应 `rows` 实测 20） |
| startXnxq | String | 是 | 起始学年学期，**页面默认写死 `001`（入学以来）**，也可填完整串 `2026-2027-1`；取自 `select#startXnxq` 选项 |
| endXnxq | String | 是 | 结束学年学期，同上，页面默认 `001` |
| sort | String | 否 | 实测为**多键** `xnxq desc,id`（URL 编码 `xnxq+desc%2Cid`），非单字段 `xnxq` |
| order | String | 否 | `desc` 或 `asc` |
| kcmc / sfjg / kcxz / kclx / kclb / kcgs | String | 否 | 课程名称/归属/性质/类型/类别/过程考核等筛选，留空=不过滤 |
| `query.{字段}\|\|=` | String | 否 | jqGrid 插件把上面筛选项再重复提交一份（`\|\|` URL 编码为 `%7C%7C`），可原样带上 |

> 📌 修正：`001` 完全可用——成绩页 JS `$(document).ready` 里就写着 `$("#startXnxq").val("001"); $("#endXnxq").val("001");`，5 次实测请求发的也全是 `001`（09-23 曾误判"`001` 形态不适用本校"）。

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

**实测要点**：

- 有数据时键顺序为 `msg, ret, page, rows, total, totalPages, results`（顺序不影响解析）。
- **无数据时响应不含 `results` 键**：`{"msg":"ok","ret":0,"page":1,"rows":20,"total":0,"totalPages":0}`，解析要容错。
- 三个页签 `fxbz=0/1/9` 各发一次请求，均返回 `rows:20`。
- **`ret` 的特殊值**（成绩页 jqGrid `loadComplete` 回调的实测逻辑）：
  - `ret == 0`：正常；
  - `ret == 1`：把 `msg` 当提示语展示（如"评教未完成"类）；
  - `ret == -1`：显示"未获取到评教数据，无法查看成绩"——**不是网络/鉴权失败**，见 §13.6。

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
GET /admin/xsd/xsdcjcx/qbcjcx              # 主修
GET /admin/xsd/xsdcjcx/qbcjcx?fxtype=1     # 辅修（页面标题"学生端学生成绩查询-辅修"）
GET /admin/xsd/xsdcjcx/qbcjcx?fxtype=9     # 微专业（页面标题"学生端学生成绩查询-微专业"）
```

返回 HTML 页面，内嵌 jqGrid 表格配置，通过 iframe 加载。

**该页还提供**：

- `select#startXnxq` / `select#endXnxq` 的完整学期选项列表（权威学期来源，见 §6.5）；
- 页面加载即执行 `$("#startXnxq").val("001"); $("#endXnxq").val("001"); search("xsdcjcxGridIdGrid");`，**默认按"入学以来"查询**；
- 评教依赖：jqGrid `loadComplete` 里按 `ret==1 / -1` 显示评教提示（见 §7.1）；
- 关联接口：`/admin/xsd/pj/xspj/getPjxtsz`（评教开关，§19.1）、`/admin/system/gridfield/toFieldBind?gridId=xsdcjcxGridIdGrid&gridurl=/admin/xsd/xsdcjcx/xsdQueryXscjList`（表格字段设置页）。

---

## 8. 选课

> 本章全部来自 2026-10-13 的**选课专项抓包**（112 条网络条目），链路为：**302 入口 → `listV2` 初始化 → `listjxbDataV2` 列教学班 → `xsdXkV2` 选课 / `xsdTkV2` 退课**，外加选课模块字典。
> 尚缺：`getMenuList` 响应体未落盘、`encLogin` 无流量（见 §16）。
> ⚠️ `/admin/xkgl/xkrwgl/ajaxListDatas`（选课任务管理）是**管理端**网格，只在 `curdtools_jqgrid.js` 注册表里出现；学生端选课走的是下面这套 `/frontend/**` Vue 页面，两者不是一回事。

### 8.1 页面入口与前端结构

```http
GET /admin/xsd/xk
→ 302 Location: https://jw.cqcvc.edu.cn/frontend/views/student/xkPage/index.html
```

学生选课是**独立的 Vue 单页**（资源前缀 `/frontend/`，与 `/admin/**` 的 JSP 体系不同）：

```text
/frontend/views/student/xkPage/index.html
/frontend/views/student/xkPage/js/{index,xsdxk,jhxk,fjjx,cxxk,ggxxk}.js
/frontend/views/student/xkPage/css/xkPage.css
/frontend/utils/{vue,axios,element,http-vue-loader,request,utils,jquery}.js
```

- 搜索区字段**不是写死的**，由 `listV2` 返回的 `xsdxkzdjson` 动态生成（每项含 `key/label/isSelect/control(type=input|select)/type`）。
- 请求前缀取隐藏域 `#adminPath`（= `/admin`）或全局 `$jwctx + '/admin'`。
- 页面顶部「教务首页」判断依据是 `listV2` 的 `showGoHome`；退出走 `GET /admin/caslogout`。

### 8.2 选课初始化 listV2（✅ 实测）

```http
POST /admin/xsd/xk/listV2
Content-Type: application/json
X-Requested-With: XMLHttpRequest

（无请求体）
```

页面 JS 原文：`request({ url: $jwctx + '/admin/xsd/xk/listV2', method: 'post' })`（注意 `Content-Type` 是 json，但不带 body）。

**选课期实测响应**（21,834 字节）：

```json
{"ret":0,"msg":"success","data":{
  "xkxnxq":"2026-2027-1", "showGoHome":"1",
  "xsxm":"…", "xsxh":"…", "xsxb":"1", "dbname":"…",
  "sfwjxb":"False", "hadJxbxmlx":"", "cxxkwkcts":"暂无课程", "xsdxksfycyxrl":"0",
  "currentxq":"", "loginImage5":"ce67c911…", "loginImage6":"",
  "pcencs":{"03e882ba19f6423d9d1d5e8be881cc12":"fD4d3Lfl…=="},
  "mapsList":[ { "…选课批次规则，见下…": "" } ],
  "kcgsList":[ "…7 项课程归属字典…" ],
  "kkxqList":[ "…开课校区（同 §6.5 结构）…" ],
  "kcxzList":[ ], "kclxList":[ ], "jxmsList":[ ]
}}
```

`data` 共 **20** 个键：`kcgsList, currentxq, loginImage6, mapsList, sfwjxb, xkxnxq, cxxkwkcts, hadJxbxmlx, loginImage5, xsxm, xsdxksfycyxrl, pcencs, kkxqList, xsxh, jxmsList, kclxList, dbname, xsxb, showGoHome, kcxzList`（实测个数：`mapsList` 1 个批次、`kcgsList` 7 项、`kkxqList` 1 个校区、`kcxzList` 11 项、`kclxList` 11 项、`jxmsList` 3 项）。

| 字段 | 说明 |
|---|---|
| `mapsList` | **选课批次列表**，每项即一个可选批次，`xkgzid` 就是后续所有接口要用的 `pcid` |
| `pcencs` | `{pcid → pcenc}`，该批次的加密串，`listjxbDataV2` / `queryChildClassesV2` 必须回传 |
| `kcgsList` | 课程归属字典（`ZD_KCGS`），实测 7 项：语言类2 / 实验（实训）类3 / 体育类4 / 实践类5 / 艺术类6 / 其他类9 / 无WU |
| `kkxqList` | 开课校区列表（`id`/`xqh`/`xqmc`/`xqdz`，与 §6.5 同源） |
| `kcxzList` / `kclxList` / `jxmsList` | 课程性质 / 课程类型 / 教学模式 下拉选项 |
| `xsxm` `xsxh` `xsxb` `dbname` | 学生姓名 / 学号 / 性别 / 数据库标识（与 §8.2 末尾"非选课期"示例完全一致） |
| `xsdxkzdjson`（在 `mapsList[i]` 内） | 搜索区配置 JSON，决定页面上出现哪些筛选框 |

`mapsList[0]` 实测样例：

```json
{"xkgzid":"03e882ba19f6423d9d1d5e8be881cc12",
 "xkgzMc":"2026年秋期在校生公选课选课第1批",
 "type":"2", "kklx":"…", "xkms":"…", "xtxksjpchc":"1",
 "sfkxk":"1", "sfkqhbxk":"0", "sfkqxbm":"0", "sfkzjxbrl":"0", "fyzssz":"0",
 "cqbmms":"", "bz":"选课截止，选课人数不到30人不开课。",
 "xtxksjkxpcs":"…", "yxkxjxbsfmtsq":"…", "xkPromptMessage":"…", "time":"…",
 "xsdxkzdjson":"[{\"key\":\"kcmc\",\"label\":\"课程名称\",\"isSelect\":true,\"control\":\"input\",…}…]"}
```

**非选课期**（2026-09-20 逆向记录，非本次抓包）仍是：

```json
{"ret":-1,"msg":"没有可选的教学班","data":{"xsxh":"…","xkxnxq":"2026-2027-1","cxxkwkcts":"暂无课程","dbname":"cqcszyxy","xsxm":"…","xsdxksfycyxrl":"0","xsxb":"1","showGoHome":"1"}}
```

> 两种形态的公共键就是上面 8 个，选课期额外多出批次/字典类的 11 个键。

### 8.3 教学班列表 listjxbDataV2（✅ 实测）

```http
POST /admin/xsd/xk/listjxbDataV2
Content-Type: application/x-www-form-urlencoded
X-Requested-With: XMLHttpRequest
```

| 参数 | 必填 | 说明（取自页面 js） |
|---|---|---|
| from | 是 | 页签来源，实测取值 `jhxk`（计划选课）/ `fjjx` / `ggxxk`（公共选修课） |
| pcid | 是 | 批次 id = `mapsList[i].xkgzid` |
| pcenc | 是 | `modelObject.pcencs[pcid]`，加密串 |
| kkxq | 否 | 开课校区，取 `kkxqList[i].id`（如 `00001`） |
| kcmc / teacher / jxbmc | 否 | 课程名称 / 教师 / 教学班名称，模糊搜索 |
| kcxz / kcgs / kclx / kclb | 否 | 课程性质 / 归属 / 类型 / 类别（下拉值见 `kcxzList` 等） |
| jxms | 否 | 教学模式 |
| glymjxb / glctjxb | 否 | 关联用民/体教学班筛选（`input[name^='init…']` 的 radio） |

**实测响应**（45,333 字节）：

```json
{"ret":0,"msg":"success","data":{
  "mysksjd":["10101","10102","10203", "…共 308 个已选时间节次码…"],
  "initData":[ { "…教学班对象，共 56 条、48 个字段…" : "" } ],
  "condition":{"jxms":["中文教学"],"kclx":["公共选修课"],"kcgs":["--"],"kcxz":["公共选修课"],"kkxq":["00001"],"kclb":["17"]},
  "kfrlbfb":"100"
}}
```

- `initData` 字段全集：`id, kkxxdm, jxbmc, kclb, kcjj, kclbname, kclx, kcid, sfyzjxb, fjxb, kcsx, kcsxcode, sfpk, childJxbIds, pendingReleaseCount, bmzt, jxbxmlx, pyfakc, sfym, bz, kcxz, kcxzcode, kcbh, kcmc, kcgs, xf, type, jxms, yxrl, teacher, sksjdd, source, status, sfkxk, bjrs, jsrl, sfct, kkxq, kkxqdm, kkxqmc, jsbq, jsjj, jxblx, kkyx, jxbbh, jxbzc, ksxs, label`
- 实测样例值：`kcmc="园林植物栽培与养护(公选)"`、`teacher="曹会娟"`、`xf="3.0"`、`kkxqmc="永川"`、`kclb="17"/kclbname="公选"`、`kcxz="公共选修课"`、`jxms="中文教学"`、`ksxs="考查"`、`bjrs="1500"`、`sfkxk="1"`（可选）、`bmzt="0"`（报名状态）、`status="0"`、`sksjdd="--/--"`（上课时间地点未排）、`sfyzjxb="0"`（无子教学班）
- `mysksjd` 是**我已选时间**的节次码集合（`校区+学期+节次` 编码），用于把冲突时段的课置灰。
- `condition` 是当前生效的筛选条件回显。

### 8.4 选课与退课（✅ 实测）

```http
POST /admin/xsd/xk/xsdXkV2        # 选课
POST /admin/xsd/xk/xsdTkV2        # 退课
Content-Type: application/x-www-form-urlencoded
X-Requested-With: XMLHttpRequest
```

抓包实测请求体（两接口一致）：

```text
jxbid=ad435dcfe7294a1e8af79a3793d4bde1&pcid=03e882ba19f6423d9d1d5e8be881cc12
```

| 参数 | 说明 |
|---|---|
| `jxbid` | 教学班 id，取 `initData[i].id` |
| `pcid` | 批次 id = `mapsList[i].xkgzid` |
| `sfqc` | 页面 js 里的可选参数，冲突补报确认后带 `1`（首次不传） |
| `zjxbid` | 子教学班 id，仅 `fjxb` 场景（先经 `queryChildClassesV2` 选子班）传 |

响应：

```json
{"ret": 0, "msg": "选课成功"}
{"ret": 0, "msg": "退课成功"}
```

- 失败时 `ret != 0`，`msg` 直接弹给用户；页面还会用 `handleCqbmConflictConfirm(res, pcid, …)` 处理**冲突报名**确认（`cqbmms` 配置项）。
- 选课/退课后页面会立刻重新 `POST listjxbDataV2` 刷新列表（抓包里 3 次调用夹着 `xsdXkV2`、`xsdTkV2`）。

### 8.5 其余选课接口（页面 JS 引用，本次未产生流量）

| 方法 | 路径 | 参数 | 返回（按 js 读取方式） |
|---|---|---|---|
| POST | `/admin/xsd/xk/checkSfxkBypcid` | `pcid` | `ret == 1` 表示该批次已选过课 |
| GET | `/admin/xsd/xk/getShowTab` | 无 | `extend.mapsList`（按批次数划分页签） |
| GET | `/admin/xsd/xk/xkpcxsmd/getbxpcid` | 无 | `data` = 必选本批次名单 |
| GET | `/admin/xsd/xk/getJsxx?jxbid=…` | jxbid | `data[]` 含 `jsxm`、`jsbh`（教师简介弹窗） |
| GET | `/admin/xsd/xk/getKcjj?jxbid=…` | jxbid | 课程简介 HTML 页 |
| POST | `/admin/xsd/xk/queryChildClassesV2` | `jxbid, pcid, pcenc, from` | `extend.zjxb` 子教学班数组（`>1` 个时进选子班页） |
| POST | `/admin/xsd/xk/queryChildClasses` | 同上 | 旧版，仍存在于 js 中 |
| GET | `/admin/xsd/xk/getEndTime/{kklx}?pcid=…` | kklx, pcid | 选课倒计时截止时间 |
| GET | `/admin/xsd/xk/getStartEndTime?pcid=…` | pcid | 起止时间 |
| POST | `/admin/xsd/xk/qxhbxk` | — | 取消合并班选课 |
| POST | `/admin/xsd/xk/applyMt` / `deleteMtApply` | — | 免修/替代申请相关 |
| — | `/admin/xsd/xk/listJxb/cxxk/jxb`、`/listJxb/cxxk/bind`、`/cxxk/th`、`/cxxk/thtk`、`/cxxk/childClass?jxbid=` | — | cxxk 页签（`cxxk.js`）专用 |

### 8.6 选课模块字典（✅ 实测，注意是另一个路径）

```http
POST /admin/xsd/xkprocess/student/getDictByGroupCode
Content-Type: application/x-www-form-urlencoded

groupCode={字典组}
```

与 §11.1 的 `/admin/getDictByGroupCode` 是**两个不同路径**（参数相同，都在选课页被调用）。实测三个字典组：

| groupCode | 响应 |
|---|---|
| `ZD_KCXZ`（课程性质） | `{"ret":0,"msg":"获取成功!","data":[ … 元素含 dictname/dictcode/sfbx/sfsjhj/sfgxk/sfqy/sort/groupcode … ]}` |
| `ZD_KCLB`（课程类别） | `{"ret":0,"msg":"获取成功!","data":[ {"dictname":"公共基础课","dictcode":"01",…} ]}` |
| `ZD_JXBXMLX`（教学班姓名类型？） | `{"ret":-1,"msg":"获取失败"}` ← **本校未配置该组，属"无数据"而非故障，正好印证 §13.6** |

字典元素实测样例（`ZD_KCXZ`）：`dictname="公共选修课"`、`dictcode="99"`、`sfbx="1"`、`sfsjhj="1"`、`sfgxk="1"`、`sfqy="1"`、`canUpdate="0"`、`updateDate="2024-01-24 16:10:21"`。

### 8.7 菜单列表（请求✅ / 响应❌ 未落盘）

```http
POST /admin/getMenuList
Content-Type: application/x-www-form-urlencoded

id={菜单模块ID}
```

返回左侧菜单结构，用于获取各功能模块的 URL 和权限。

**实测**：请求体 `id=79f069873ada42d1a5096ec0884f3966`（同一 `cmid` 也用于 `/admin/indexMain/{cmid}` 与页签标题「选课查询」），`Content-Type` 为 `application/json`（页面 js 用 `dataType:'json'`），状态 200；**响应体本次未落盘**，只能从 js 反推是**数组**（判空写法 `if (res == null || res.length == 0)`），每项用于渲染 `#leftMenu`。待补录，见 §16。

### 8.8 加密跳转登录（两次抓包均未产生流量）

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

响应元素结构（经选课页同款接口实测，见 §8.6）：

```json
{"ret":0,"msg":"获取成功!","data":[
  {"dictname":"公共选修课","dictcode":"99","groupcode":"ZD_KCXZ","sort":0,
   "sfbx":"1","sfsjhj":"1","sfgxk":"1","sfqy":"1","canUpdate":"0","canDel":"0",
   "updateDate":"2024-01-24 16:10:21","groupid":"…","id":"…","new":false, "…会话上下文字段…": ""}
]}
```

| 项 | 说明 |
|---|---|
| 已知 groupCode | `ZD_KCXZ`（课程性质）、`ZD_KCLB`（课程类别）、`ZD_KCGS`（课程归属，经 `listV2.kcgsList` 返回）、`ZD_JXBXMLX`（本校未配置） |
| 字典组未配置时 | `{"ret":-1,"msg":"获取失败"}` —— 属"无数据"，**不是故障**（§13.6） |
| ⚠️ 两个路径 | 选课页用的是 `/admin/xsd/xkprocess/student/getDictByGroupCode`（参数完全相同），普通页面用本节路径，别混 |

### 11.2 字段翻译

```http
GET  /admin/system/gridfield/getTranslation?yysz=ch
POST /admin/system/gridfield/getTranslation        # Content-Type: application/x-www-form-urlencoded
                                                   # body: yysz=ch
```

**实测**：两种方法都会被页面调用，响应相同：

```json
{"ret": -1, "msg": "操作成功"}
```

> 📌 `ret = -1` 却 `msg = 操作成功`——该接口把 `-1` 当"无待翻译项"用，**不能据此判定失败**（§13.6）。每个网格加载都会连打数次（单轮抓包 25 次），自行实现时建议缓存。

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
| xnxq | String | 是 | 学年学期，如 `2026-2027-1`（页面会随页签切到 `2026-2027-2`） |
| pcid | String | 否 | 考试批次 ID，来自 `getKspc?xnxq=`（§19.5），留空=全部批次 |
| `query.xnxq\|\|` / `query.pcid\|\|` | String | 否 | jqGrid 把上面两项再重复提交一份（`\|\|` 编码为 `%7C%7C`） |

**请求示例**：

```text
queryFields=id,kspcmc,xh,xm,kcmc,kssj,jsmc,ksfs,ksxs,zwh,bkcs,bz,rwbz,&_search=false&nd=1791899576083&page.size=50&page.pn=1&sort=bjdm+asc,zwh+asc,id&order=asc&xnxq=2026-2027-1&pcid=&query.xnxq%7C%7C=2026-2027-1&query.pcid%7C%7C=
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

> **空结果**：无考试记录时返回 `{"msg":"ok","ret":0,"page":1,"rows":50,"total":0,"totalPages":0}`，**不含 `results` 键**；`rows` 固定为 50（= `page.size` 默认值）。

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
| 1 | 部分接口用作"带提示的成功"，`msg` 为提示文案（如成绩页评教未完成） |
| -1 | 失败，`msg` 字段含具体原因 |

> 📌 修正：`-1` **不总是失败**。实测样本：
> - `getTranslation`（GET/POST）成功也返回 `{"ret":-1,"msg":"操作成功"}` —— `-1` 表示"无待翻译项"；
> - `xsdQueryXscjList` 在"未获取到评教数据"时返回 `-1`，成绩页据此显示提示而非报错；
> - 选课期 `listV2` 无课时同样 `-1`（见 §8.2）。
>
> **判定建议**：`ret==0` 必为成功；`ret!=0` 时结合 `msg` 与该接口语义判断，不要只看 ret。

---

## 14. WAF 与安全校验

- 域名 `jw.cqcvc.edu.cn` 启用了 **Web 应用防火墙**，基于 IP 特征 + UA 特征。
- **实测（2026-09-23，同会话同端点仅换 UA）**：浏览器 UA（Edge/Windows）访问 `getCurrentXnxq` → **200**；无 UA → **403**；`ZhengfangAcademicPlugin/1` → **403**。
- **结论修正**：JSON API **同样按 UA 拦截**——原文“JSON API 通常不受影响”仅在携带浏览器 UA 时成立；登录 POST 携带浏览器 UA 实测 `302 → /admin` 成功。
- SPA 页面 curl 直访仍可能502；建议始终携带完整浏览器 UA 并保持合理请求间隔。

---

## 15. 注意事项

1. **Session 过期**：`JSESSIONID` 有效期不确定，长时间无操作需重新登录。
2. **Cookie 绑定**：Session 绑定 IP + UID，换 IP 后可能失效。
3. **学期参数两套口径**：成绩接口的 `startXnxq` / `endXnxq` 用 `001`（入学以来）或完整串均可，见 §7.1；`sdpkkbList`、`getZclistByXnxq`、`ajaxXsksList` 的 `xnxq` 一律用完整串 `2026-2027-1`。
4. **空结果要容错**：非选课期 `listV2` 返回"没有可选的教学班"、新生可能没有成绩、jqGrid 空结果**不带 `results` 键**（§7.1 / §12）。
5. **`ret` 不能单独判断成败**：`-1` 可能表示"无数据 / 未配置"，见 §13.6。
6. **Content-Type**：只有 `listV2` 用 `application/json`（且**不带 body**）；课表、成绩、考试、`getbzxx`、`check*`、`insertOrUpdate`、`getTranslation` 等实测均为 `application/x-www-form-urlencoded`。
7. **字典编码不统一**：`sdpkkbList` 的 `kcxz` / `ksxs` 是中文，成绩、考试 jqGrid 的同名字段是 §13 的数字代码，两边不能混用。
8. **高频接口缓存**：`getTranslation` 每个网格加载都会连打（单轮 25 次），自行实现时建议缓存（§11.2）。

---

## 16. 待进一步确认

**选课（专项抓包后剩余）**

1. `getMenuList` 的响应体——请求已验证（`id={cmid}`），响应未落盘（§8.7）。
2. `encLogin` 的参数与响应——两轮抓包均无流量（§8.8）。
3. §8.5 那 12 个"JS 引用但无流量"接口的响应结构（`checkSfxkBypcid`、`getShowTab`、`getbxpcid`、`getJsxx`、`getKcjj`、`queryChildClasses(V2)`、`getEndTime`、`qxhbxk`、`applyMt`…）。
4. `fjjx` / `cxxk` 两个页签的确切含义（`from` 实测仅 `jhxk` / `fjjx` / `ggxxk` 三种）。
5. `xkprocess/getDictByGroupCode` 的 `ZD_JXBXMLX` 为何返回"获取失败"（未配置还是权限）。
6. `/admin/xkgl/xkrwgl/ajaxListDatas`（**管理端**选课任务网格）的 queryFields / 响应结构——两轮抓包都只见于 JS 注册表。

**其它**

7. 通知（§9）、工作台（§10）的响应体——目前只验到路径与状态码 200。
8. `/admin/xsd/xyjc/ajaxTable` 的完整参数、`/admin/system/export/urlExport` 的导出参数。
9. 登录链路：学习通扫码回调、`twiceAuth` 触发条件（抓包中出现 `/admin/system/usermanage/checkNeedTwiceAuth?roleId=`）、CAS 票据校验。
10. `getKspc?xnxq=` 仅 1 条样本待复验；`checkSfkc` / `checkWfwSfpj` 的确切语义（参数由调用位置推测）。
11. `getSetting` 的 `extend` 下还有哪些开关（现见 1 个）；`gridId` 全集（现见 3 个）。

---

## 17. 学生课表真数据源 sdpkkbList（周次）

课表网格接口（§6.1）**不含周次**；网页课表上每门课显示的周次来自本节接口。抓包复验：URL 与参数逐字命中、`xhid` 恰为 86 位、响应样例 `xf=2 / zongxs=32 / xkrs=44`、`kcmc` 与 `croommc` 确为 HTML 锚点。

实测请求原文（参数顺序即页面实际发送顺序）：

```http
GET /admin/pkgl/xskb/sdpkkbList?xnxq=2026-2027-1&xhid=WGEyQ0B766D960A4BE43DBAA8A1ADD2047FC67DC5FC36A2F2D217F62DEF7AAA2CBBF8FC200B73B46567C78&xqdm=00001&zdzc=&zxzc=&xskbxslx=0
```

### 17.1 打开课表页获取隐藏域

```http
GET /admin/pkgl/xskb/queryKbForXsd?xnxq={学期}&zxzc=&zdzc=&xskbxslx=0
```

- `zxzc` / `zdzc`：起始/终止周次筛选，留空 = 全部；`xskbxslx`：显示类型页签（默认 `0`）。
- 响应 HTML 内含隐藏域：`xhid`（学生哈希，长度 86）、`xqdm`（校区代码，如 `00001`，与 §6.5 的 `id` 一致）。

### 17.2 查询课表数据（含周次）

```http
GET /admin/pkgl/xskb/sdpkkbList?xnxq={学期}&xhid={xhid}&xqdm={xqdm}&zdzc=&zxzc=&xskbxslx=0
```

页面 JS 为 `$.ajax` 默认 **GET**；每行 = 一个课程时段（与网格课程一一对应，实测 38 行对应 38 门）。

响应 `{ret:0, msg:"操作成功", data:[...]}`，关键字段：

| 字段 | 含义 | 实测样例（脱敏） |
|---|---|---|
| `xingqi` | 星期 1-7 | `1` |
| `djc` | 起始节次 | `1` |
| `kcmc` | 课程名，**HTML 锚点，需剥标签** | `<a ...>大学英语Ⅰ</a>` |
| `zc` | 周次区间式 | `4-5,9-18` |
| `zcstr` | 周次展开式（逗号分隔，优先使用） | `4,5,9,10,...,18` |
| `croommc` | 教室，**HTML 锚点，需剥标签** | 文华楼119 |
| `jxlmc` | 楼栋 | 文华楼 |
| `jxbmc` | 教学班名称 | 大学英语Ⅰ(公共课)-理论114 |
| `xq` / `xqmc` | 校区 | `00001` / 永川 |
| `xf` `zongxs` `xkrs` | 学分 / 总学时 / 选课人数 | `2` / `32` / `44` |

注意：

- **数据形态事实**：同一课程名在不同星期**各自一行、周次可以不同**；**同一星期、同一 `djc` 也可能有多行**（`croommc` 教室、`zcstr` 周次不同——真实样本：周四第1-2节高等数学 = 文华楼125 第15-18周 + 文华楼521 第4-5周）。
- `kcmc`、`croommc` 字段值为 HTML 锚点（含 `<a>` 标签）。
- 网格与周次源的连接、去重、合并等属使用方的数据处理逻辑，不在本文档范围。
- 同页相关接口：备注 `POST /admin/pkgl/xskb/getbzxx`（参数 `xnxq`、`xhid`）、打印 `reportforxskb`；周次筛选与周次合并算法都在该页 JS 中。

## 18. 开学日期与当前周推算

- **首选真源（2026-10-14 实测）**：§6.4 `getZclistByXnxq`（**必须带 `xqid`**）返回的 **`data.zclist`** 就是权威周历，每周一条 `minrq`/`maxrq`——**开学日 = `zclist[0].minrq`**，本校实测 **2026-09-05**；周次高亮直接按日期落在哪周即可，无需自行累加。
- **兜底公式**（拿不到 `zclist` 时才用）：**开学日 = 服务器本周一 − (dqzc − 1) × 7 天**，其中 `dqzc` 来自同一接口、`Date` 响应头为中国标准时间 +8。
- ⚠️ **本校公式法会早 5 天**：实测第 1 周是**不完整周**（`zclist[0] = 09.05–09.06`，仅周六日），公式按整周回推得到 2026-08-31，比真实开学日 **早 5 天**。两次抓包（09-23 `dqzc=4`、10-13 `dqzc=7`）回推的都是 08-31，说明公式稳定但系统性偏差——**只有学校第一周恰好是完整周时才准**。
- 用途：客户端课表的 `startDate`、当前周高亮；**仅对当前学期有效**（跨学期需带对应 `xnxq` 重新查）。
- 实现提示：QuickJS 的 `Date.parse` 不识别 RFC1123，需按 `Wed, 23 Sep 2026 06:00:00 GMT` 手写解析；Android 宿主会把 `Date` 响应头透出给插件。

---

## 19. 未收录接口（抓包实测）

> 2026-10-13 全量抓包解析（456 条网络条目、61 次 `/admin` 调用、6 个页面）后**发现、此前未收录**的接口；每个接口的实测请求与响应都直接写在本节。

### 19.1 评教开关

```http
GET /admin/xsd/pj/xspj/getPjxtsz
```

```json
{"ret":0,"msg":"操作成功","data":{"pjmbxsbprzp":"0","wpjcsfkccj":"0","xspjwcpjc":"0","wpjsfbkccj":"0","wpksfbkccj":"0"}}
```

五个键均为 `"0"/"1"` 开关，命名可直读为：评教模式是否上报评教人、**未评教是否可见成绩**、学生未完成评教、未评教是否可看课程、未评教是否可看考试（语义按拼音首字母推测，**待确认**）。成绩页加载时会调它，与 §7.1 的 `ret==-1` 提示联动。

### 19.2 两个检查接口

```http
POST /admin/xsd/kcapcx/checkSfkc                      # body: xnxq=2026-2027-1&type=1
POST /admin/xsd/pkgl/xskb/checkWfwSfpj                # body: xnxq=2026-2027-1
POST /admin/xsd/kwglXsdKscx/checkWfwSfpj              # body: xnxq=2026-2027-1 / 2026-2027-2
```

三者响应均为 `{"ret":0,"msg":""}`（`application/x-www-form-urlencoded`）。课表页与考试页各有一个 `checkWfwSfpj`（路径前缀不同，别写混）；语义按参数与调用时机推测为评教前置检查，**待确认**。

### 19.3 系统开关

```http
GET /admin/api/getSetting
```

```json
{"ret":0,"msg":"操作成功","extend":{"SYSTEM_XLXSMZDYT":"1"}}
```

注意响应键是 **`extend`**，不是其它接口的 `data`。

### 19.4 表格字段设置（gridfield 三件套）

```http
# 1) 字段配置页 HTML（页面标题"表格字段设置"）
GET  /admin/system/gridfield/toFieldBind?gridId=xsdcjcxGridIdGrid&gridurl=/admin/xsd/xsdcjcx/xsdQueryXscjList

# 2) 查询字段配置（请求体为 {}，响应为 {}）
POST /admin/system/gridfield/queryFieldByGridId?gridId=xsdcjcxGridIdGrid&gridurl=/admin/xsd/xsdcjcx/xsdQueryXscjList

# 3) 保存字段配置
POST /admin/system/gridfield/insertOrUpdate
Content-Type: application/x-www-form-urlencoded

gridId=xsdkscxGridIdGrid&fields=kspcmc%2Cxh%2Cxm%2Ckcmc%2Ckssj%2Cjsmc%2Cksfs%2Cksxs%2Czwh%2Cbkcs%2Cbz%2Crwbz%2CserialIndex&fieldsQuery=xnxq%2Cpcid&fieldsQuerySort=xnxq%2Cpcid&gridurl=%2Fadmin%2Fxsd%2FkwglXsdKscx%2FajaxXsksList
```

- `fields` / `fieldsQuery` / `fieldsQuerySort` 即该网格的**列定义与查询列**，`gridurl` 是对应的数据接口（URL 编码）。
- 响应 `{"ret":0,"msg":"操作成功"}`；另有未抓到流量的 `insertOrUpdateAll`（批量保存）。
- 保存后还会带出一次 `xsdQueryXscjList` / `ajaxXsksList` 重查，可据此反推列配置是否生效。
- 已知 `gridId`：`xsdcjcxGridIdGrid`（成绩）、`xsdkscxGridIdGrid`（考试）、`xkrwglGridIdGrid`（选课任务）。

### 19.5 考试批次

```http
GET /admin/xsd/kwglXsdKscx/getKspc?xnxq=2026-2027-2
```

响应 `{"ret":0,"msg":""}`（本次仅 1 条样本，批次 ID 即 §12 的 `pcid`，**待复验**）。考试页还有 iframe 子页：

```text
/admin/kwgl/kspc   /admin/kwgl/kscc   /admin/kwgl/kssj   /admin/kwgl/kcgl
/admin/kwgl/jkyx   /admin/kwgl/jkjs   /admin/kwgl/jkrs   /admin/kwgl/cdqxkz
```

### 19.6 其它有引用、未产生流量的路径

```text
/admin/pkgl/xskb/reportforxskb                      课表打印
/admin/xsd/xsdzgcjcx/chooseTemplate?startXnxq=&endXnxq=&dylx=1   成绩打印模板选择
/admin/system/export/urlExport                      导出（§11.3）
/admin/system/attachment/{showImage,getFileViewUrl,getAttachmentById}
/admin/checkImage/index.html   /admin/customExport/index.html
/admin/pygcgl/pygccxtj/kkxxcxtj/kkxxjxb             开课信息-教学班
/admin/jsd/qxzkb/querylist                          全校总课表
/admin/xkgl/xkrwgl/ajaxListDatas                    选课任务管理（见 §8）
```

---

## 20. 免责声明

本文档仅用于技术学习、接口整理与开发参考。  
请勿将相关接口用于未经授权的访问、批量爬取、账号盗用或其他违反学校规定与法律法规的行为。  
使用本文档产生的任何后果由使用者自行承担。