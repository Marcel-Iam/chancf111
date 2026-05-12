# 凤姐转运 - 订单管理系统

## 项目概述

前端纯静态网站，用 GitHub 仓库作为数据库。没有后端服务器，所有读写通过 GitHub REST API 完成。部署在 GitHub Pages 或任何静态托管上都可以。

仓库：`Marcel-Iam/chancf111`


## 文件结构

```
/
├── index.html              客户下单页面
├── admin.html              管理后台
├── orders.json             订单数据（初始值为 []，不可为空白文件）
├── products.json           产品列表
├── history_pickup/         取货单 PDF 存档
│   ├── index.json          索引（文件名、日期、提货人、订单数等 meta）
│   ├── pickup_20250508_103000.pdf
│   └── ...
└── history_shipping/       寄件单 PDF 存档
    ├── index.json          索引（文件名、日期、订单数、收件人数等 meta）
    ├── shipping_20250508_143000.pdf
    └── ...
```


## 数据结构

### orders.json

数组，每个元素是一个订单。初始状态必须是 `[]`，空白文件会导致 `JSON.parse` 报错。

```json
{
  "id": "ORD_1715200000001_a1b2",
  "created_at": "2025-05-08T10:30:00.000Z",
  "created_by": "小陈",
  "paid_status": false,
  "picked_up": false,
  "shipped": false,
  "incoming": [
    {
      "express_code": "DD20250508001",
      "pickup_code": "8832",
      "products": [
        { "product_id": "p001", "name": "产品A", "quantity": 20 }
      ]
    },
    {
      "express_code": "DD20250508002",
      "pickup_code": "5541",
      "products": [
        { "product_id": "p002", "name": "产品B", "quantity": 5 }
      ]
    }
  ],
  "outgoing": [
    {
      "name": "张伟",
      "phone": "13800001111",
      "address": "北京市朝阳区建国路88号",
      "products": [
        { "product_id": "p001", "name": "产品A", "quantity": 10 },
        { "product_id": "p002", "name": "产品B", "quantity": 5 }
      ],
      "notes": "工作日白天送"
    }
  ]
}
```

关键字段说明：

- `id`：格式 `ORD_{timestamp}_{random4}`，由 index.html 在提交时生成
- `created_by`：填表人称呼，用于追踪谁提交的订单
- `picked_up`：是否已从快递处取回货物
- `shipped`：是否已寄出给收件人
- `paid_status`：运费是否已收，boolean。默认 `false`
- `picked_up`：货物是否已从快递处取回
- `shipped`：货物是否已寄出给收件人
- `incoming`：来件信息数组。一个大订单可以包含多张来件单，每张有独立的 `express_code`（内部单号）、`pickup_code`（取货时的确认码）、`products`
- `outgoing`：收件人列表（一批货可能分寄给多人）
- 数量校验跨所有来件单合并计算，与所有收件人合计对比

**注意**：`incoming` 是数组，不是对象。这是本次改动的核心破坏性变更，旧格式（`incoming` 为对象）不兼容，读取时不做自动转换。

### products.json

```json
[
  { "id": "p001", "name": "产品A" },
  { "id": "p002", "name": "产品B" }
]
```

大约 20 个产品。`id` 是短码（用于程序内部匹配），`name` 是中文全称（用于显示）。

### history_pickup/index.json 和 history_shipping/index.json

数组，每个元素记录一次 PDF 生成：

```json
{
  "filename": "pickup_20250508_103000.pdf",
  "type": "pickup",
  "pickup_date": "2025-05-08",
  "person": "小陈",
  "order_count": 3,
  "created_at": "2025-05-08T10:30:00.000Z"
}
```

寄件单的 meta 字段略有不同：`type: "shipping"`，`date`，`order_count`，`recipient_count`。


## GitHub API 认证

GitHub token 不能明文存放在源码里。GitHub 的 secret scanning 会自动检测并吊销以 `ghp_` 开头的 token，包括 Base64 编码过的。

当前方案：XOR 加密。用密钥 `orderSysKey2025` 对 token 的每个字符做 XOR，输出一串用点号分隔的数字（例如 `23.45.12.67.89...`），存入 `CONFIG._k`。运行时由 `_d()` 函数解密。

生成加密 token 的方法（浏览器控制台执行）：

```js
function enc(t) {
  var k = 'orderSysKey2025';
  var r = [];
  for (var i = 0; i < t.length; i++) {
    r.push(t.charCodeAt(i) ^ k.charCodeAt(i % k.length));
  }
  return r.join('.');
}
enc('ghp_actualTokenHere')
```

输出的数字字符串放入 `CONFIG._k`。两个文件（index.html 和 admin.html）的 `CONFIG._k` 必须一致。

当前 `CONFIG._k` 设为 `'xxx'` 占位符。部署前需替换成真实加密后的 token。


## UTF-8 编码处理

GitHub API 返回的 content 是 Base64 编码的。直接用 `atob()` 解码中文会乱码，因为 `atob()` 按 Latin-1 处理字节。

读取时用 `decodeB64()`：
```js
function decodeB64(str) {
  const bytes = Uint8Array.from(atob(str), c => c.charCodeAt(0));
  return new TextDecoder().decode(bytes);
}
```

写入时用 `encodeB64()`：
```js
function encodeB64(str) {
  return btoa(unescape(encodeURIComponent(str)));
}
```

两个文件都必须用这对函数。不要用 `atob`/`btoa` 直接处理 JSON 字符串。


## 并发写入处理

多人同时写入 `orders.json` 会导致 SHA 冲突（GitHub API 要求每次写入带上最新 SHA）。

当前方案：retry 机制。写入前先读最新 SHA，写入失败后等待 500ms * (重试次数) 再重新读取、重新写入，最多重试 3 次。

index.html 里是 `writeOrderWithRetry` 和 `updateOrderWithRetry`。
admin.html 里是 `updateWithRetry`（通用版，接受 mutator 函数）。


## index.html - 客户下单页面

### 页面结构

页面分为上下两个明显独立的区域：

**上方（可折叠）：查找 / 修改已提交订单**
- 标题改为"查找 / 修改已提交订单"，默认折叠
- 输入任意一个来件单号即可搜寻整个大订单（`incoming.some` 遍历）
- 载入后表单填入订单资料，底部按钮切换为"发送修改"和"取消修改"

**下方（金色边框卡片）：提交新订单**
- 填表人称呼
- 来件信息：每张来件单是一个独立卡片（可增删），每张卡有订单号、取货码、来件产品列表
- 收件人信息：每个收件人是一个独立卡片（可增删）

### 功能

- 一个订单可以包含多张来件单，每张独立填写
- 空的收件人卡片自动跳过
- 空的产品行自动跳过
- 产品数量校验：所有来件单产品合计 vs 所有收件人产品合计，不匹配时弹出确认框，用户可选择仍然提交

### 安全

- `esc()` 函数对所有用户输入做 HTML 实体转义，防止 XSS


## admin.html - 管理后台

### 密码

硬编码在 `CONFIG.password`，当前是 `456456`。输入正确后隐藏密码框，显示管理界面。

### 四个主分页

1. **来件管理**
2. **寄件管理**
3. **历史档案**
4. **产品管理**

### 来件管理

子视图切换：未取货 / 已取货（按钮在工具栏右侧，刷新旁边）

**未取货视图：**
- 表格列：checkbox | 填表人 | 订单号 | 取货码 | [动态产品列] | 日期 | 已付运费
- 每张来件单占一行，同一大订单的填表人和日期用 rowspan 合并
- 产品数量按来件单分行显示（不合并），底部合计行统计所有来件单的产品总数
- 工具栏按钮：生成取件单、已取货、编辑、刷新
- 生成取件单：弹窗填写提货日期和提货人，生成 PDF 后自动在新标签页打开，同时上传到 `history_pickup/`。PDF 里每张来件单各占一行，填表人用 rowspan 合并
- 已取货：勾选的订单标记 `picked_up: true`
- 编辑模式：隐藏 checkbox，每个大订单右侧出现"修改"和"删除"按钮（rowspan 合并）
- 已付运费列：checkbox，勾选和取消勾选都会弹出确认框，确认后才写入。取消弹窗时 checkbox 恢复原状，数据不变。修改 overlay 内不含此字段

**已取货视图：**
- 同样的表格，不含合计行
- 编辑模式下每行显示"修改"和"未取货"按钮
- 未取货：将 `picked_up` 改回 `false`

### 寄件管理

子视图切换：未寄出 / 已寄出

**未寄出视图：**
- 表格列：checkbox | 填表人 | 订单号 | 收件人 | 地址 | 电话 | [动态产品列] | 备注 | 已付运费
- 订单号列显示该订单所有来件单号，换行排列。若未取货，每个单号后显示红色 `(未取货)` 标签
- 多个收件人用 rowspan 合并填表人/订单号列
- 排序：已取货的订单排前面，未取货的排后面
- 底部合计行
- 工具栏按钮：生成寄件单、已寄出、编辑、刷新
- 生成寄件单 PDF 不含填表人和订单号列（只有收件人、地址、电话、产品、备注）
- 已寄出：勾选的订单标记 `shipped: true`
- 编辑模式同来件管理

**已寄出视图：**
- 编辑模式下显示"修改"和"未寄出"按钮

### 修改订单 Overlay

点击表格中的"修改"按钮弹出全屏 overlay 表单，结构与 index.html 一致：

- 填表人称呼
- 来件信息：每张来件单一张卡片（可增删）
- 收件人列表（可增删，每个收件人有产品列表）
- 保存时做数量校验（跨所有来件单合并），不匹配弹确认框
- 校验弹窗的 z-index (960) 高于修改 overlay (950)，不会被遮挡

### 删除订单

编辑模式下点"删除"弹出确认框，确认框显示该订单所有来件单号，确认后从 `orders.json` 中移除该订单。

### 历史档案

子视图切换：历史取货单 / 历史寄件单

- 列表显示每次生成的 PDF 的 meta 信息（日期、提货人、订单数等）
- 点击"检视"通过 GitHub API 读取 PDF 内容，转成 blob URL 在新标签页打开
- 编辑模式下"检视"变为"删除"，删除时弹确认框提示无法还原
- 删除操作同时删除 PDF 文件和 index.json 中的对应记录

### 产品管理

- 表格显示所有产品，列标题为"简称 (ID)"和"全称"
- 可增删产品行
- 编辑产品时不会丢失未保存的输入（增删行前先同步 DOM 到内存）
- 保存时写入 `products.json`

### PDF 生成

使用 html2pdf.js（CDN 引入）。流程：
1. 拼接 HTML 字符串（内联样式的表格）
2. 放入隐藏的 `#pdfRender` div
3. html2pdf 渲染为 blob
4. blob URL 在新标签页打开
5. blob 转 Base64 上传到 GitHub 对应的 history 文件夹
6. 更新 index.json 索引

取件单用 portrait，寄件单用 landscape。


## z-index 层级

| 层级 | 元素 |
|------|------|
| 960 | `.modal-overlay`（确认弹窗、导出弹窗、数量校验弹窗、删除确认） |
| 950 | `.edit-overlay`（修改订单表单） |
| 998 | `.loading-overlay`（加载遮罩） |
| 999 | `.toast`（提示消息） |


## 设计规范

- 字体：Noto Sans SC（Google Fonts CDN）
- 主色：`#b8860b`（暗金色）
- 背景：`#f5f0eb`（暖灰）
- 卡片：白色，1px `#d4cdc4` 边框，8px 圆角
- 危险操作：`#c0392b`
- 成功操作：`#27ae60`
- 按钮风格：`.btn-accent`（实心金色）、`.btn-success`（实心绿色）、`.btn-outline`（描边灰色）、`.btn-danger`（实心红色）
- 移动端响应式：700px 以下 form-row 堆叠，按钮缩小
- index.html 的新订单区域用金色边框 (`.new-order-section`) 与查找区域视觉区分


## 注意事项

1. `CONFIG._k` 在两个 HTML 文件中都设为 `'xxx'` 占位符，部署前必须替换
2. 两个文件的 `CONFIG.owner`、`CONFIG.repo`、`CONFIG._k` 必须一致
3. GitHub token 需要 repo 的读写权限
4. `orders.json` 必须存在且内容为 `[]`，空白文件或不存在都会导致 `JSON.parse` 报错崩溃
5. `products.json` 同上，初始内容为 `[]` 或已有产品列表
6. `history_pickup/` 和 `history_shipping/` 文件夹不需要预先创建，首次生成 PDF 时会自动创建
7. 所有用户输入输出都必须经过 `esc()` 函数转义
8. 所有 JSON 读写都必须用 `decodeB64()`/`encodeB64()` 处理编码
9. PDF 文件是二进制，上传时用 `b2b64(blob)` 转 Base64，不经过 `encodeB64()`
10. `incoming` 是数组，旧格式（对象）不兼容，代码内不做自动转换