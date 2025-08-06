# ESP32 + AS7341 环境光传感器 for Home Assistant "无感"节律照明

本项目旨在通过ESP32和AS7341光谱传感器，构建一个高精度的室外环境光监测站，并与Home Assistant的[Adaptive Lighting](https://github.com/basnijholt/adaptive-lighting)插件深度集成，实现室内照明色温与室外自然光的动态同步，创造一种真正“无感”且符合人体生理节律的智能照明体验。

## ✨ 项目亮点

- **高精度光谱传感**：使用AS7341 11通道光谱传感器，相比传统RGB传感器，能更精确地测量光线的色温（CCT）和照度（Lux）。
- **无感体验**：结合ESPHome的数据滤波和Adaptive Lighting的平滑过渡功能，消除因环境光突变（如云层遮挡）导致的灯光闪烁或突兀变化。
- **人性化校准**：提供了先进的、分时段的色温映射策略，让室内光线在模仿自然的同时，更符合人类在早晨（需要清爽）和傍晚（需要放松）的不同生理需求。
- **低成本 & 易于部署**：基于广泛可用且成本低廉的ESP32平台和ESPHome框架，无需编写复杂代码，通过YAML配置即可完成。

## 硬件需求

1.  **主控**: ESP32系列开发板 (本项目使用 **ESP32-S3**，其他型号如ESP32, ESP32-C3也兼容)。
2.  **传感器**: AS7341 11通道光谱传感器模块。
3.  **连接线**: 若干杜邦线。

## 连线图

将AS7341传感器模块连接到ESP32开发板。这是一个标准的I2C连接。

| AS7341 模块引脚 | 连接到 ESP32-S3 开发板引脚 | 说明                 |
| :-------------- | :------------------------- | :------------------- |
| **VIN** | **3.3V** | 供电                 |
| **GND** | **GND** | 地线                 |
| **SCL** | **GPIO9** | I2C 时钟线           |
| **SDA** | **GPIO8** | I2C 数据线           |
| INT (可选)      | GPIO10                     | 中断引脚（可选连接） |
| GPIO (可选)     | *不连接* |                      |

*注意：ESP32的IO MUX功能非常强大，您几乎可以将SCL/SDA/INT连接到任何其他GPIO引脚，只需在后续的YAML配置中正确指定即可。*

## 软件配置

### 第一步：配置ESPHome传感器固件

如果对esphome配置存在疑惑，可以参考B站相关视频

1.  在Home Assistant中安装并打开ESPHome插件。
2.  创建一个新的设备，并将下面提供的 `esphome.yaml` 内容粘贴进去。
3.  **重要**: 修改 `wifi:` 部分，填入您自己的Wi-Fi SSID和密码。
4.  保存配置，并首次通过USB将固件“安装”到您的ESP32开发板上。


### 第二步：配置Home Assistant集成

#### 1. 安装 Adaptive Lighting

如果您尚未安装，请通过HACS (Home Assistant Community Store) 安装名为 "Adaptive Lighting" 的自定义集成。

#### 2. 配置 Adaptive Lighting 使用传感器

1.  在Home Assistant中，导航到 **设置 > 设备与服务 > 添加集成**，然后选择 **Adaptive Lighting**。
2.  为这个实例命名（例如：“客厅节律照明”），并选择您希望控制的灯具或灯组。
3.  **核心步骤**:
    - 勾选 **`Use separate sensors for color and brightness`**。
    - 在 **`Color Temperature Sensor`** 字段，选择我们创建的 `sensor.outside_color_temperature` 实体。
    - 在 **`Brightness Sensor`** 字段，选择 `sensor.outside_illuminance` 实体。
4.  **设置平滑过渡**:
    - 找到 **`Transition time (seconds)`** 选项，并将其设置为一个较大的值，例如 `60`。这是实现“无感”变化的关键。
5.  保存配置。

## 高级调优技巧

### A. 自定义早晚色温曲线

为了让早晨的光线更清爽、傍晚的光线更温馨，我们可以创建一个模板传感器来对室外色温进行非线性映射。

1.  将以下代码添加到您的Home Assistant `configuration.yaml` 文件中：
```
    template:
      - sensor:
          - name: "Adjusted Outside Color Temperature"
            unique_id: adjusted_outside_color_temp_001
            unit_of_measurement: "K"
            state: >
              {# 定义基础参数 #}
              {% set outdoor_temp = states('sensor.outside_color_temperature') | float(default=4000) %}
              {% set outdoor_min = 2000 %}
              {% set outdoor_max = 8000 %}
              
              {# 定义两套不同的室内目标范围 #}
              {% set indoor_min_morning = 3200 %} 
              {% set indoor_max_morning = 6500 %}
              {% set indoor_min_evening = 2700 %}
              {% set indoor_max_evening = 6500 %}

              {# 判断当前是上午还是下午 #}
              {% if state_attr('sun.sun', 'azimuth') < 180 %}
                {% set indoor_min = indoor_min_morning %}
                {% set indoor_max = indoor_max_morning %}
              {% else %}
                {% set indoor_min = indoor_min_evening %}
                {% set indoor_max = indoor_max_evening %}
              {% endif %}

              {# 执行映射计算 #}
              {% set percentage = (outdoor_temp - outdoor_min) / (outdoor_max - outdoor_min) %}
              {% set indoor_temp = indoor_min + (percentage * (indoor_max - indoor_min)) %}
              
              {# 最终输出钳位和取整后的值 #}
              {% set final_temp = [indoor_min, indoor_temp, indoor_max] | sort | last %}
              {% set final_temp = [indoor_min, final_temp, indoor_max] | sort | first %}
              {{ final_temp | round(0) }}
```

3.  在开发者工具中重新加载模板实体。
4.  **返回Adaptive Lighting的配置**，将 `Color Temperature Sensor` 更改为新建的 `sensor.adjusted_outside_color_temperature`。

