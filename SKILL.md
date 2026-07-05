---
name: billing-system
version: 3.0.0
description: 对账单生成系统 — 零配置通用版。首次使用自动注册试用账户，自动下载模板。
agent_created: true
---

# 对账单生成系统

## 服务地址
```
http://workbuddy.linyiyouban.com.cn
```

## 首次使用（自动注册）

此 Skill 不含 API Key。首次调用时 **自动注册试用账户（3 次配额）**：

```
1. curl "http://workbuddy.linyiyouban.com.cn/auto-register?name=用户名"
   → 返回 {"key":"wb-xxxxxxxx","user":"用户名","quota":3,"role":"trial"}
2. 把返回的 key 保存为本地变量，后续所有请求带上这个 key
```

## 完整调用流程

### Step 1: 注册/获取 Key

如果还没有 key：
```bash
# 自动注册
RESP=$(curl -s "http://workbuddy.linyiyouban.com.cn/auto-register?name=用户名")
KEY=$(echo $RESP | python -c "import sys,json; print(json.load(sys.stdin)['key'])")
echo "你的 Key: $KEY"
```

如果已经有 key（之前注册过），跳过此步。

### Step 2: 下载模板

```bash
curl -o template_input.xlsx \
  "http://workbuddy.linyiyouban.com.cn/template?key=$KEY"
```

### Step 3: 填写模板数据

模板包含 4 个 Sheet：

| Sheet | 说明 |
|-------|------|
| 使用说明 | 模板使用指引 |
| 客户信息 | 客户名称、年利率(%)、期初余额月份、期初余额(元)、备注 |
| 借款记录 | 客户名称、月份、借款编号、借款日期、到期日期、借款金额(元)、春晖利率(%)、红润利率(%)、备注 |
| 还款记录 | 客户名称、月份、还款编号、还款日期、还款金额(元)、指定借款、备注 |

> 利率留空则使用客户信息中的默认年利率。
> 指定借款留空则自动分配。

### Step 4: 生成对账单

```bash
curl -X POST \
  "http://workbuddy.linyiyouban.com.cn/generate?key=$KEY" \
  -F "file=@template_input.xlsx" \
  -o 对账单结果.zip
```

### Step 5: 解压结果

```bash
unzip 对账单结果.zip -d 对账单结果/
```

生成文件：`对账单.xlsx` + `{客户名}_对账单.pdf`

## 在对话中使用（WorkBuddy Agent）

当用户说"生成对账单"、"制作账单"时：

1. **如果没有 key**：先 curl 调用 `/auto-register?name=用户名` 注册
2. **如果没有模板**：curl 下载模板
3. **帮用户填写数据**：确认借款/还款数据后，用 openpyxl 写入模板
4. **POST 上传生成**：curl POST /generate
5. **解压结果**：unzip 后告知用户文件列表

### 模板填写示例（用 openpyxl）

```python
import openpyxl
wb = openpyxl.load_workbook("template_input.xlsx")

# 客户信息（Sheet 2）
ws1 = wb["客户信息"]
ws1.append(["张三", 12, "2025-01", 100000, ""])

# 借款记录（Sheet 3）
ws2 = wb["借款记录"]
ws2.append(["张三", "2025-01", "JK001", "2025-01-15", "2025-07-15", 50000, 0, 0, ""])

# 还款记录（Sheet 4）
ws3 = wb["还款记录"]
ws3.append(["张三", "2025-02", "HK001", "2025-02-28", 20000, "", ""])

wb.save("template_input.xlsx")
```

## 计费规则

- 试用账户：**3 次免费配额**
- 正式账户：按配额计费
- 余额不足返回 HTTP 402
- 联系管理员（Claire）开通正式账户

## 注意事项

- 首次使用需注册（自动）
- Key 建议保存到本地文件，避免重复注册
- 如果 key 失效（冷启动丢失），重新注册即可
- 模板中的「使用说明」Sheet 有完整指引
