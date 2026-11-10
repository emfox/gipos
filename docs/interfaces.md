# 接口契约

五份契约，全部可以独立替换实现。**伪代码里的类型只是说明，不是要求用 Python。**

## 1. Position（唯一的数据结构）

线上格式，`schema` 固定，新增字段只能**追加**且必须有默认值。

```jsonc
{
  "schema": "gipos/position/1",
  "frame": "genshin-map",     // 坐标系标识，强校验
  "x": -1234.5,               // frame 平面坐标，单位由 frame 定义（genshin-map = 米）
  "y": 678.0,
  "confidence": 0.86,         // 0..1，缺失视为 1.0
  "ts": 1794320971.5,         // 产生时刻，unix 秒
  "stale": false,             // true = 未更新，沿用上次观测
  "source": "genshin-x11",    // 生产者标识，便于多源共存
  "seq": 42                   // 生产者内自增，用于丢帧判定
}
```

约定：

- 消费者**必须忽略**不认识的字段（前向兼容）
- `frame` 不认识 → 拒收，记 `adapter.unknown_frame`
- `stale=true` 时只允许更新"陈旧"标记，不允许把它当作新位置去居中

## 2. 总线绑定：HTTP

参考绑定。同一语义也可实现为 in-process 调用、Unix socket、SSE。

| 方法 | 路径 | 说明 |
|---|---|---|
| GET | `/position` | 返回最新 Position，附加 `age`（秒）；无数据返回 `{"x":null,...}` 200 |
| POST | `/position` | 发布；`202` 接受，字段校验失败 `400` |
| GET | `/stream` | SSE，每条 `data: <Position>`（可选实现） |
| GET | `/health` | `{"ok":true,"uptime":12.3,"last_publish_age":0.4}` |
| GET | `/pos` | 兼容别名（M0 已有），等价 `/position` |

- CORS：`Access-Control-Allow-Origin: *`，允许 `Content-Type`
- 节流由生产者负责，总线**不做**去重与合并
- 总线保留最近一次值，进程重启即丢失（无持久化）

## 3. 位置源 PositionSource

```python
class PositionSource(Protocol):
    name: str                      # 如 "genshin-x11"
    frame: str                     # 它产出的坐标系
    def start(self, publish: Callable[[Position], None]) -> None: ...
    def stop(self) -> None: ...
```

约束：

- 只调用 `publish`，**不知道**总线和适配器存在
- 自己决定置信度与 `stale`，总线不再判断
- 允许完全不产出（游戏没开），不算错误

## 4. 采集 RoiProvider

```python
Frame = namedtuple("Frame", "ts image meta")   # image: HxWx3 uint8

class RoiProvider(Protocol):
    def frames(self) -> Iterator[Frame]: ...   # 阻塞产出，至少 0.5 fps
    def describe(self) -> dict: ...            # 窗口尺寸、ROI、DPI 等，用于日志与标定复用
```

实现：`x11`（XShm）、`wayland`（PipeWire portal）、`dir`（按顺序回放 PNG 目录，用于离线测试）。

约束：**只给像素**，不解释内容；ROI 由配置提供，不由 provider 猜测。

## 5. 底图 MapProvider

```python
class MapProvider(Protocol):
    frame: str                                     # 它服务的坐标系
    def levels(self) -> tuple[int, int]: ...        # (min_z, max_z)
    def tile(self, z: int, x: int, y: int) -> Image # 256×256，带本地缓存
    def pixel_to_frame(self, z, px, py) -> tuple[float, float]: ...
    def frame_to_pixel(self, x, y, z) -> tuple[float, float]: ...
```

第一个实现 `yuanshen-tiles`：

- 瓦片 URL 与地理参数取自网页地图的 `settings`：`center`、`size`、`tilesOffset`、`extension`
- `pixel_to_frame` = `(tile_x * 256 + px) / 2**z * scale + offset` 形式的仿射（具体系数在实现里从 settings 推导）
- 缓存：`~/.cache/gipos/tiles/z{z}/{x}_{y}.{ext}`，可整目录删除

## 6. 显示适配器 DisplayAdapter

```python
class DisplayAdapter(Protocol):
    frames: set[str]                      # 能处理的坐标系
    def attach(self, bus: Bus) -> None: ...
    def on_position(self, p: Position) -> None: ...   # 必须幂等、可丢帧
    def detach(self) -> None: ...
```

约束：

- **换算在适配器内**：`leaflet-web` 用 `map.project()` 三点探测解仿射再求逆；`overlay` 用窗口坐标 + 缩放；
  位置源对这些一无所知
- 收不到数据、frame 不匹配、被用户暂停 → 由适配器自己表现（面板提示 / 停止居中），不上报给总线
- 不得修改 Position，不得回写总线

已有的 `leaflet-web` 适配器具体做法（供新适配器参考）：

1. `document-start` 抢 `window.L`，patch `Map.prototype` 捕获实例
2. 三点探测 `project(latlng, 0)` 求 2×3 仿射，求逆得到「frame 坐标 → LatLng」
3. `setView(latlng, 保持当前 zoom)` + 一个 circleMarker 打点
4. 用户拖动即暂停跟随

## 7. 配置文件

`~/.config/gipos/config.toml`（示例）

```toml
[source.genshin-x11]
frame = "genshin-map"
fps = 2.0
confidence_min = 0.55

[capture.x11]
roi = [24, 24, 232, 232]      # x, y, w, h，按 2560x1440 标定
window_hint = "YuanShen"

[map.yuanshen-tiles]
frame = "genshin-map"
cache_dir = "~/.cache/gipos/tiles"
cache_limit_mb = 1024

[bus.http]
bind = "127.0.0.1:8765"
```

## 8. 版本策略

- `schema` 形如 `gipos/position/1`，破坏性变更才升主版本
- 端口契约以本文档为准；实现可先支持子集，但**不得改变语义**
- 未实现的端点返回 `501`，不返回假数据
