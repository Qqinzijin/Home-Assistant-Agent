# Home Assistant 仿真实验室

本项目用官方 Home Assistant Container 镜像运行一个本地、可重置的 120㎡ 三室一厅虚拟住宅。运行时数据库、认证信息和日志不会提交到 Git。

## 启动

前置条件：Docker Desktop 已运行。

```powershell
docker compose up -d
docker compose logs -f homeassistant
```

打开 `http://localhost:8123`，创建本地管理员账户并完成初始化。

启动后，侧边栏会出现 **家庭数字孪生**。它将实时读取 Home Assistant 实体状态：点击平面图上的设备图标可打开详情并控制，后续 Agent 调用同一实体时图标状态会同步变化。

## 户型与分区

| 区域 | 主要标准实体 | 典型 Agent 场景 |
| --- | --- | --- |
| 玄关 | 玄关灯、入户门锁、入户门已打开 | 回家、离家、安全确认 |
| 客餐厅 | 主灯、氛围灯、餐厅灯、空调、窗帘、空气净化器、温度、照度、PM2.5 | 观影、会客、空气质量调节 |
| 主卧 | 主灯、床头灯、空调、窗帘、温度 | 睡眠、起床、舒适度调节 |
| 次卧 | 灯、空调、窗帘、温度 | 客人模式、节能控制 |
| 书房 | 灯、风扇 | 专注/离开书房 |
| 厨房 | 灯、油烟机、洗碗机、燃气报警 | 烹饪安全、饭后清洁 |
| 卫生间 | 灯、排风扇、湿度、漏水报警 | 洗澡后除湿、漏水告警 |
| 阳台 | 灯、洗衣机、电动晾衣架 | 洗衣、晾晒 |

区域归属由 Home Assistant 的运行时注册表保存。在 UI 打开 **设置 → 区域、标签和分区 → 区域 → 创建区域**，建立上述 8 个区域；随后在 **设置 → 设备与服务 → 实体** 搜索实体名称，点开实体并将“区域”设为对应房间。Template 仿真实体没有物理设备条目，直接在实体详情中分配区域即可。

`input_boolean` 和 `input_number` 仅为仿真器内部状态；Agent 应调用标准的 `light`、`climate`、`fan`、`cover`、`lock`、`switch`、`sensor` 和 `binary_sensor` 实体。门锁解锁属于高风险动作，后续 Agent 必须要求二次确认。

## 创建 Agent API Token

在 Home Assistant 的个人资料页面创建 **Long-Lived Access Token**，并只保存在本地环境变量中：

```powershell
$env:HA_URL = "http://localhost:8123"
$env:HA_TOKEN = "替换为你的 token"
```

不要把 token 写入仓库、README 或聊天记录。

## REST API 验证

读取所有标准实体状态：

```powershell
$headers = @{ Authorization = "Bearer $env:HA_TOKEN" }
Invoke-RestMethod "$env:HA_URL/api/states" -Headers $headers |
  Where-Object entity_id -match '^(light|climate|fan|cover|lock|switch|sensor|binary_sensor|scene)\\.' |
  Select-Object entity_id, state
```

打开客厅主灯：

```powershell
$headers = @{ Authorization = "Bearer $env:HA_TOKEN"; "Content-Type" = "application/json" }
$body = @{ entity_id = "light.ke_ting_zhu_deng_2" } | ConvertTo-Json
Invoke-RestMethod "$env:HA_URL/api/services/light/turn_on" -Method Post -Headers $headers -Body $body
```

将客厅窗帘开合到 35%：

```powershell
$body = @{ entity_id = "cover.ke_ting_chuang_lian_2"; position = 35 } | ConvertTo-Json
Invoke-RestMethod "$env:HA_URL/api/services/cover/set_cover_position" -Method Post -Headers $headers -Body $body
```

Home Assistant REST API 使用 Bearer token；状态读取是 `GET /api/states`，服务调用为 `POST /api/services/<domain>/<service>`。详见 [官方 REST API 文档](https://developers.home-assistant.io/docs/api/rest/)。

## 配置校验与更新

```powershell
docker compose run --rm homeassistant python -m homeassistant --script check_config --config /config
docker compose restart homeassistant
```
