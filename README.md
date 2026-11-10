# gipos — Genshin Impact Positioning

把「我在游戏里的位置」从游戏画面里算出来，通过一个**与显示端无关**的通道发布出去，
由各个显示适配器决定画在哪、怎么画。

```
游戏画面 ──▶ 位置源 ──▶ 位置总线 ──▶ 显示适配器 ──▶ 网页地图 / 悬浮层 / 终端 / …
```

本项目**只负责定位和发布**，不做地图客户端、不做自动打点、不读写游戏内存。

## 设计原则

1. **端口与适配器**：位置源、底图、采集、发布、显示五者互相只认接口，不认实现。
2. **坐标中立**：总线上只有带 `frame` 标识的平面坐标；换算成 Leaflet latlng、屏幕像素、UE 世界坐标都由适配器自己做。
3. **一切可回放**：每个端口都有回放/假实现，不启动游戏也能开发与回归。
4. **配置与代码分离**：ROI、掩膜、门限都在配置里，游戏改版时只改配置。
5. **不碰游戏**：只截屏，不注入、不读内存、不 hook 渲染管线。

## 状态

| 阶段 | 内容 | 状态 |
|---|---|---|
| Spec | 需求、架构、接口契约 | v0.2 草案，待定稿 |
| M1 | 采集与 ROI 标定 | 未开始 |
| M2 | 定位（瓦片 NCC） | 未开始 |
| M3 | 总线与适配器通用化 | 未开始 |

本仓库当前**只放规范**，不含实现。

## 文档

| 文件 | 内容 |
|---|---|
| [docs/requirements.md](docs/requirements.md) | 目标、非目标、功能/非功能需求、里程碑、风险 |
| [docs/architecture.md](docs/architecture.md) | 分层、模块、解耦规则、扩展点、决策记录 |
| [docs/interfaces.md](docs/interfaces.md) | 五份端口契约与线上格式（冻结目标） |

## 规划中的目录

```
docs/            规范（当前只有这些）
src/gipos/
  core/          Position 事件、Bus 协议、配置加载
  capture/       x11 / wayland-portal / dir(回放)
  sources/       genshin-x11 / manual / replay
  maps/          yuanshen-tiles / offline-pack
  adapters/      leaflet-web / cli / overlay
  bus/           inproc / http
tests/           回放驱动的回归
```

## 许可

待定。
