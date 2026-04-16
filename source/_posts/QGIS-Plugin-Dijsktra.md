---
title: 【技术笔记】QGIS插件开发
date: 2023-12-20 20:58:29
tags:
    - 技术笔记
    - GIS
---
> 课程作业中QGIS插件开发的一些笔记

## 1. 开发与迭代

首先基于QGIS中的Plugin Builder创建模板

- 使用 QGIS 自带的 Python 环境（天然带有 `qgis`、`PyQt` 等依赖）。
- 使用其中的开发部署工具（例如 `pb_tool`）把插件目录部署到你当前 QGIS profile 对应的插件目录，而不是每次手动拷贝。
- 安装插件重载工具（如 Plugin Reloader），这样修改代码后可以快速重新加载插件进行验证。

### 模板中的一些核心部分

一个典型 QGIS Python 插件，至少包含下面这些文件/概念：

1. `metadata.txt`
   - 包括插件信息：`name`、`version`、`qgisMinimumVersion`、`description`、`icon`、`category` 等。
   - QGIS 和插件仓库用它来展示、校验兼容性与版本。
2. `__init__.py`
   - 入口函数：`classFactory(iface)`。
   - QGIS 会调用它来拿到你的“主类实例”（主类负责注册菜单、创建工具、处理 UI 事件）。
3. 主类文件（例如 `main_plugin.py` / `my_plugin.py`，由你命名）
   - 必须实现常见方法：
     - `__init__(self, iface)`：保存 `iface`（`QgsInterface`），这是访问 QGIS UI/地图画布/消息栏的入口。
     - `initGui(self)`：创建 `QAction`，注册到菜单/工具栏。
     - `unload(self)`：卸载时移除 QAction（否则会重复注册）。

### 模板生成的主类核心结构

```python
from qgis.PyQt.QtCore import Qt
from qgis.PyQt.QtGui import QIcon
from qgis.PyQt.QtWidgets import QAction


class MyPlugin:
    def __init__(self, iface):
        self.iface = iface
        self.actions = []
        self.menu = self.tr("&My Plugin")
        self.first_start = True

    def tr(self, message):
        # 翻译字符串（可选但常用）
        return self.iface.appMainWindow().tr(message)

    def initGui(self):
        action = QAction(QIcon(":/plugins/my_plugin/icon.png"), self.tr("Run"), self.iface.mainWindow())
        action.triggered.connect(self.run)
        self.iface.addPluginToMenu(self.menu, action)
        self.iface.addToolBarIcon(action)
        self.actions.append(action)

    def unload(self):
        for action in self.actions:
            self.iface.removePluginMenu(self.menu, action)
            self.iface.removeToolBarIcon(action)

    def run(self):
        # 打开对话框 / 开始交互 / 调算法
        pass
```

## 2. UI接入

### 2.1 用 Qt Designer 设计界面

流程：

1. 用 Qt Designer 设计对话框/窗体，导出 `.ui`
2. 在 Python 里加载 `.ui`：
   - 常见做法是 `uic.loadUiType()` 动态加载并 `setupUi(self)`

```python
import os
from qgis.PyQt import uic
from qgis.PyQt import QtWidgets

FORM_CLASS, _ = uic.loadUiType(os.path.join(os.path.dirname(__file__), "dialog_base.ui"))


class MyDialog(QtWidgets.QDialog, FORM_CLASS):
    def __init__(self, parent=None):
        super().__init__(parent)
        self.setupUi(self)
```

### 2.2 接入UI事件

建议把所有 `button.clicked` / `comboBox.currentIndexChanged` 这类信号连接放到一个集中位置（比如 `run()` 里）。

典型模式是：

- `run()`：创建/显示对话框，并连接信号
- `accepted()` / `onOkClicked()`：从 UI 读取参数，调用算法或开始任务
- `rejected()` / `onCancelClicked()`：清理状态

## 4. 与地图交互：点选/框选/拾取坐标

大多数插件交互都需要读用户在地图上的操作，比如点击起终点等。

通用做法：

- 用 `QgsMapToolEmitPoint`（或自定义`QgsMapTool`）捕获鼠标点击
- 在点击回调里获取 `QgsPointXY` 或屏幕到地图的坐标
- 用 `QgsProject.instance()`、`iface.mapCanvas()` 更新界面或创建要素图层

关键点：

- 坐标系要一致：如果输入图层/输出图层 CRS 不同，需要做坐标转换（`QgsCoordinateTransform`）。
- 输出可视化建议用 `memory` provider 创建临时图层，便于用户理解交互结果。

## 5. 图层读写：Raster/Vector/内存图层怎么处理

常见的工程结构是：

- 输入：从 `QgsProject.instance().mapLayers()` 取得图层、根据名称/ID选择具体图层
- 参数准备：从栅格 provider 获取 extent、分辨率、行列数等
- 运算：在算法层处理（可以是最短路、栅格成本、矢量拓扑等）
- 输出：把结果写回图层（`QgsVectorLayer` + `QgsFeature` + `QgsGeometry`）

### 输出内存图层

典型步骤：

- `line_layer = QgsVectorLayer("LineString", "Result", "memory")`
- `pr = line_layer.dataProvider()`
- `feature = QgsFeature()` + `feature.setGeometry(...)`
- `pr.addFeatures([feature])`
- `QgsProject.instance().addMapLayer(line_layer)`

## 7. 资源与图标

（如果希望给插件设计个图标）为了让图标在 QGIS 里稳定加载，通常会：

- 把 `icon.png` 放入 `resources.qrc`
- 用资源编译得到 `resources.py`
- 在 QAction 或插件引用里使用资源路径（例如 `:/plugins/your_plugin/icon.png`）

通常不建议手动修改 `resources.py`（由资源编译工具自动生成）。


