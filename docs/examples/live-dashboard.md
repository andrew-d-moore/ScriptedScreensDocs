# Live Dashboard Example

A real-time station monitoring dashboard that updates gauges, sparklines, and tables every tick.

## Full Script

```lua
local ui = ss.ui.surface("main")
ss.ui.activate("main")
local size = ui:size()
local W, H = size.w, size.h

ui:clear()

-- Background
ui:element({ id = "bg", type = "panel",
    rect = { unit = "px", x = 0, y = 0, w = W, h = H },
    style = { bg = "#0B1120" }
})

-- Title
ui:element({ id = "title", type = "label",
    rect = { unit = "px", x = 10, y = 4, w = W - 20, h = 20 },
    props = { text = "STATION MONITOR" },
    style = { font_size = 14, color = "#22C55E" }
})

-- Temperature gauge
local tempGauge = ui:element({ id = "temp", type = "gauge",
    rect = { unit = "px", x = 10, y = 30, w = 110, h = 70 },
    props = { value = 22, min = -20, max = 60, label = "TEMP", unit = "°C" },
})

-- Pressure gauge
local pressGauge = ui:element({ id = "press", type = "gauge",
    rect = { unit = "px", x = 130, y = 30, w = 110, h = 70 },
    props = { value = 101.3, min = 0, max = 200, label = "PRESS", unit = " kPa" },
})

-- Temperature sparkline
local tempHistory = {}
for i = 1, 30 do tempHistory[i] = 22 end

local tempSpark = ui:element({ id = "temp_spark", type = "sparkline",
    rect = { unit = "px", x = 250, y = 30, w = W - 260, h = 70 },
    props = { data = tempHistory, min = 15, max = 35 },
    style = { bg = "#111827", line_color = "#22C55E", fill_color = "#22C55E20" },
})

-- Zone status table
local zoneTable = ui:element({ id = "zones", type = "table",
    rect = { unit = "px", x = 10, y = 110, w = W - 20, h = H - 120 },
    props = {
        columns = { "Zone", "Temp °C", "Press kPa", "O2 %", "Status" },
        rows = {},
        col_widths = { 2, 1, 1, 1, 1 },
    },
    style = {
        header_bg = "#1E293B", header_color = "#94A3B8",
        row_bg = "#111827", alt_row_bg = "#0F172A",
        row_color = "#E2E8F0", font_size = 10, row_height = 20,
    },
})

ui:commit()

-- Simulate live data
local accum = 0
local LT = ic.enums.LogicType

function tick(dt)
    accum = accum + dt
    if accum < 0.5 then return end
    accum = 0

    -- Read real sensor data if available, otherwise simulate
    local temp = ic.read(0, LT.Temperature)
    if temp then temp = temp - 273.15 else temp = 22 + math.sin(os.clock()) * 3 end

    local press = ic.read(0, LT.Pressure) or (101.3 + math.cos(os.clock() * 0.7) * 5)
    local o2 = ic.read(0, LT.RatioOxygen) or (0.21 + math.sin(os.clock() * 0.3) * 0.02)

    -- Update gauges
    tempGauge:set_props({ value = string.format("%.1f", temp) })
    pressGauge:set_props({ value = string.format("%.1f", press) })

    -- Update sparkline
    table.remove(tempHistory, 1)
    tempHistory[#tempHistory + 1] = temp
    tempSpark:set_props({ data = tempHistory })

    -- Update table
    local status = "OK"
    local statusStyle = { color = "#22C55E" }
    if temp > 35 or temp < 10 then
        status = "ALERT"
        statusStyle = { color = "#EF4444" }
    elseif temp > 30 or temp < 15 then
        status = "WARN"
        statusStyle = { color = "#EAB308" }
    end

    zoneTable:set_props({
        rows = {
            { "HAB-1", string.format("%.1f", temp), string.format("%.1f", press),
              string.format("%.1f", o2 * 100), { text = status, style = statusStyle } },
            { "HAB-2", string.format("%.1f", temp - 1.2), string.format("%.1f", press + 2),
              string.format("%.1f", (o2 - 0.01) * 100), "OK" },
            { "AIRLOCK", string.format("%.1f", temp * 0.3), "0.1",
              "0.0", { text = "VACUUM", style = { color = "#64748B" } } },
        },
    })

    ui:commit()
end
```

## Key Patterns Demonstrated

- **Live gauge updates** via `handle:set_props()`
- **Rolling sparkline** using a shifting array buffer
- **Table with per-cell styling** for status badges
- **Throttled updates** at 2 Hz to avoid excessive rebuilds
- **Graceful fallback** — simulates data if no sensor is connected
