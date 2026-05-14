# Cesium 地图与 GeoServer 排障

## Cesium 在项目中的作用

Cesium 用于智慧路灯数字孪生地图展示，主要承载路灯点位、道路发光线、告警点位、区域定位和 2D/3D 地图交互。

相关前端文件：

- `project/src/OverviewApp.vue`
- `project/src/utils/cesiumRuntime.ts`
- `project/src/utils/cesiumMapHelpers.ts`
- `project/src/utils/mapData.ts`
- `project/src/components/overview/CesiumSceneControls.vue`
- `project/src/components/overview/LampPointPanel.vue`
- `project/src/components/overview/AlarmPointPanel.vue`
- `project/src/components/overview/MapDeviceDetailPanel.vue`

## 地图加载流程

地图加载的大致流程：

1. 前端加载 Cesium runtime。
2. 读取 Cesium Ion Token。
3. 创建地形。
4. 创建底图。
5. 加载本地路灯点位 JSON。
6. 请求 GeoServer WFS 道路 GeoJSON。
7. 初始化 Cesium Viewer。
8. 渲染道路发光线。
9. 渲染路灯点位。
10. 绑定点位点击、相机飞行、2D/3D 切换和全屏控制。

## Cesium runtime

`cesiumRuntime.ts` 负责加载 Cesium 运行时。

项目使用 CDN 加载 Cesium，主要来源包括：

- jsDelivr
- unpkg

如果一个 CDN 加载失败，应尝试备用 CDN。

## Cesium Ion Token

Cesium Ion Token 从前端环境变量读取：

- `VITE_CESIUM_TOKEN`

## 底图策略

底图创建逻辑在 `createCesiumBaseLayer` 中。

底图模式由环境变量控制：

- `VITE_CESIUM_BASEMAP=auto`：自动模式，优先使用 Ion 影像，失败后回退。
- `VITE_CESIUM_BASEMAP=osm`：使用 OpenStreetMap。
- `VITE_CESIUM_BASEMAP=satellite`：使用备用卫星瓦片。

当前项目优先使用 Cesium Ion 无标签卫星影像，避免绿色道路和文字标签覆盖业务点位。

## 路灯点位数据来源

本地路灯点位文件：

- `project/src/mock/lampList_real_road_two_sides_200m.json`

前端通过 `lampList200mUrl` 加载该文件。点位字段主要包括：

- `lng`：经度。
- `lat`：纬度。
- `status`：状态。
- `alarmLevel`：告警等级。
- `power`：功率。
- `brightness`：亮度。

Cesium 使用以下方式定位点位：

- `Cartesian3.fromDegrees(lng, lat)`

## GeoServer WFS 数据

道路数据通过 GeoServer WFS 获取。

相关代码：

- `project/src/utils/mapData.ts`

WFS 请求会设置：

- `service=WFS`
- `request=GetFeature`
- `outputFormat=application/json`

前端 Vite 代理中配置：

- `/geoserver` 代理到 `VITE_GEOSERVER_PROXY_TARGET`，默认 `http://localhost:8080`

## GeoServer 请求失败策略

当 GeoServer WFS 请求失败时，系统不应中断 Cesium 初始化。

当前策略：

- 如果 WFS 返回非 2xx 状态，返回空 `FeatureCollection`。
- 如果请求异常，返回空 `FeatureCollection`。
- 道路发光线可以不显示，但路灯点位仍应继续加载。

这可以避免 GeoServer 不可用时整张地图空白。

## 地图无底图排查

地图无底图时，按以下顺序排查：

1. 检查 `VITE_CESIUM_TOKEN` 是否配置。
2. 检查 Cesium Ion Token 是否有效。
3. 检查浏览器是否能访问 Cesium Ion。
4. 查看浏览器控制台是否有 Ion imagery 加载错误。
5. 检查 `VITE_CESIUM_BASEMAP` 是否被设置为错误值。
6. 尝试切换到 `satellite` 或 `osm` 备用底图。
7. 确认 Cesium runtime 是否加载成功。

## 出现绿色道路或文字标签排查

如果地图中出现绿色道路、英文道路名或不需要的文字标签，通常是底图样式问题。

处理建议：

- 使用 Cesium Ion 的无标签卫星影像。
- 避免使用带道路文字的 OSM 样式作为主底图。
- 如果必须使用 OSM，降低业务点位和道路线条被遮挡的影响。
- 业务道路发光线应由前端实体渲染，而不是依赖底图道路样式。

## 路灯点位不显示排查

路灯点位不显示时，按以下顺序排查：

1. 检查本地点位 JSON 是否能加载。
2. 检查点位字段是否包含 `lng` 和 `lat`。
3. 检查经纬度是否为数字。
4. 检查 Cesium Viewer 是否初始化成功。
5. 检查点位图层是否被关闭。
6. 检查相机是否飞到正确区域。
7. 检查点位是否因距离显示条件过远而隐藏。
8. 检查图标资源是否加载成功。

## 地图 502 排查

如果 `/geoserver` 请求返回 502，说明前端代理无法访问 GeoServer。

排查步骤：

1. 检查 GeoServer 是否启动。
2. 检查 GeoServer 端口是否为 8080。
3. 检查 `VITE_GEOSERVER_PROXY_TARGET` 是否正确。
4. 检查 WFS 图层名称和工作区是否正确。
5. 检查 Nginx 或 Vite 代理配置。
6. 即使 GeoServer 失败，前端也应继续加载本地点位。

## 地图卡在加载中

可能原因：

- Cesium runtime CDN 加载失败。
- Ion 地形或影像请求超时。
- 浏览器网络无法访问外部瓦片服务。
- 代码等待某个地图资源但没有兜底。
- 容器尺寸为 0，导致 Canvas 未正确渲染。

建议：

- 查看浏览器控制台。
- 检查网络请求。
- 确认地图容器有宽高。
- 启用底图和地形回退。

## 本地 mock 数据与后端数据区别

本地 mock 点位用于地图演示和离线开发，主要保证前端地图可以独立展示。

后端设备接口用于真实设备列表、搜索和详情。

当前前端会把本地点位数据和后端设备详情结合使用：地图点位负责空间定位，设备详情接口负责补充设备运行数据。
