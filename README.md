# 轻盈计划 · 减脂健身追踪台

单文件离线工作台。一个 `index.html` 里装下了全部 CSS / JS / 图表绘制，**不引用任何外部 CDN、字体或图标库**，双击就能打开，断网照常用。

打开方式：直接双击 `index.html`，或把整个目录丢到任意静态服务器上。

---

## 一、8 张数据表

所有数据以「表 → 记录数组」的形式存在一个 JSON 文档里（本地落在 `localStorage` 的 `lq_db_v1`）。
每条记录统一带三个同步字段：

| 字段 | 说明 |
| --- | --- |
| `id` | 全局唯一，跨设备识别同一条记录 |
| `updatedAt` | 毫秒时间戳，合并时用于「最后写入优先」 |
| `deleted` | 墓碑标记。删除不物理移除，只置 `true`，这样别的设备才知道「这条被删了」 |

| # | 表名 | 中文 | 关键字段 |
| --- | --- | --- | --- |
| 1 | `profile` | 个人档案 | `name` `title` `sex` `age` `height` `activity` `deficit` `startWeight` `goalWeight` `habits[]` `theme{primary,accent,button,paper}` |
| 2 | `body_records` | 身体记录 | `date` `weight` `bodyFat` `note` |
| 3 | `diet_records` | 饮食记录 | `date` `meal` `name` `kcal` `p` `c` `f` |
| 4 | `training_plans` | 训练计划 | `weekday(1–7)` `name` |
| 5 | `training_logs` | 训练日志 | `date` `planId` `name` `done` `note` |
| 6 | `habit_logs` | 习惯打卡 | `date` `habit` `done` |
| 7 | `weekly_reviews` | 每周复盘 | `week` `delta` `trainCount` `avgKcal` `good` `bad` `next` |
| 8 | `diary` | 日记 | `date` `text` |

**表间关联**（逻辑关联，靠业务键串起来，不依赖数据库外键）：

```
profile.habits[]  ──1:N──►  habit_logs.habit
training_plans.id ──1:N──►  training_logs.planId
training_plans.weekday ──►  当前星期几 → 今日该练什么
body_records.date ─┐
diet_records.date ─┼─ 同一天 → 日视图（体重 + 摄入 + 打卡）
habit_logs.date   ─┤
training_logs.date ─┘
weekly_reviews.week ──►  body_records[week..week+6] 聚合
body_records / diet_records ──► profile 的性别/身高/体重 → BMR / TDEE / 宏量目标
```

---

## 二、双向同步怎么工作的

核心是一个快照 + 合并函数，两种触发方式：

**A. 手动 / 跨设备（零配置）**
- `导出快照 JSON` → 得到一个完整快照文件
- `导入 / 合并快照` → 读入对方快照，逐表逐条按 `id` 匹配：
  - 只有远端有 → 新增
  - 两边都有 → 谁的 `updatedAt` 新用谁
  - 远端是墓碑且更新 → 本地这条被删掉

**B. 在线端点（真正的云端存储）**
在「设置 → 数据存储与双向同步」里填一个地址即可：

```
GET  <endpoint>    → 返回快照 JSON
PUT  <endpoint>    → 写入快照 JSON（可换成 POST）
Header: Authorization: Bearer <token>   （可选）
```

`立即双向同步` 的执行顺序是 **先拉后推**：拉远端合并进本地 → 再把合并后的结果推回去。这样两端折腾到最后一定是同一个状态。勾上「打开页面时自动同步」后，每次开页面都会自动跑一次。

> 提示：浏览器出于安全考虑，会限制 `file://` 页面发跨域请求。想要自动云端同步，请通过 `http(s)://` 访问本页（部署后就是在线的那个地址）。

---

## 三、功能对照

| 需求 | 实现位置 |
| --- | --- |
| ① 每日体重 / 体脂记录，含示例数据、可清空 | 「身体记录」页；同一天重复保存会覆盖而非新增；`清空示例数据` / `清空全部数据` |
| ② 趋势折线图 + 7 日移动平均 | 「总览」页原生 Canvas 自绘，可切体重 / 体脂，鼠标或手指悬停看详情 |
| ③ 目标进度 + BMI | 「总览」页进度条 + 中国标准 BMI 分级（18.5 / 24 / 28） |
| ④ 周训练计划打卡 + 习惯打卡 | 「训练 · 习惯」页；点卡片打卡，习惯是 7 日网格 + 连击天数 |
| ⑤ BMR / TDEE + 500–750 kcal 缺口 | Mifflin-St Jeor，缺口 500 / 650 / 750 三档；自动给出蛋白 1.8 g/kg、脂肪 27%、碳水补足 |
| ⑥ 顶部问候 + 下一步建议 | 顶部卡片，按「没称重 → 该训练 → 蛋白不够 → 习惯没做完」的优先级排 |
| ⑦ 云端同步 8 表关联 | 见上文第二节 |
| ⑧ 自定义主题色（不改字号） | 「设置」页 4 个取色器 + 4 套预设；只动颜色变量，字号写死在版式里 |

---

## 四、自动校验

```bash
NODE_PATH=<jsdom 所在目录> node _smoke.js
```

无头跑一遍渲染、增删改、打卡、墓碑复活、快照合并、代谢公式，共 22 项断言。
