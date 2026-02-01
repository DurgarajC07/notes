# Dashboard Design & Data Visualization

## Table of Contents

1. [Dashboard Architecture](#dashboard-architecture)
2. [Real-Time Data Updates](#real-time-data-updates)
3. [Chart Components](#chart-components)
4. [Filter System](#filter-system)
5. [Data Aggregation](#data-aggregation)
6. [Performance Optimization](#performance-optimization)
7. [State Management](#state-management)
8. [Responsive Design](#responsive-design)
9. [Export Functionality](#export-functionality)
10. [Accessibility](#accessibility)
11. [Testing Strategy](#testing-strategy)
12. [Key Takeaways](#key-takeaways)

## Dashboard Architecture

### High-Level System Design

```
┌────────────────────────────────────────────────────────────┐
│                    Frontend Layer                          │
│                                                            │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐      │
│  │  Dashboard  │  │   Widgets   │  │   Charts    │      │
│  │   Layout    │  │  Component  │  │  Library    │      │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘      │
│         │                 │                 │             │
│         └─────────────────┴─────────────────┘             │
│                           │                               │
└───────────────────────────┼───────────────────────────────┘
                            │
┌───────────────────────────▼───────────────────────────────┐
│                  State Management Layer                    │
│   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│   │   Dashboard  │  │    Filter    │  │   WebSocket  │  │
│   │     Store    │  │    Store     │  │    Manager   │  │
│   └──────────────┘  └──────────────┘  └──────────────┘  │
└───────────────────────────┬───────────────────────────────┘
                            │
┌───────────────────────────▼───────────────────────────────┐
│                     API Layer                              │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐      │
│  │   REST API  │  │  WebSocket  │  │   GraphQL   │      │
│  │   Client    │  │   Client    │  │   Client    │      │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘      │
└─────────┼─────────────────┼─────────────────┼─────────────┘
          │                 │                 │
┌─────────▼─────────────────▼─────────────────▼─────────────┐
│                    Backend Services                        │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐      │
│  │  Analytics  │  │   Metrics   │  │  Time Series│      │
│  │   Service   │  │   Service   │  │   Database  │      │
│  └─────────────┘  └─────────────┘  └─────────────┘      │
└────────────────────────────────────────────────────────────┘
```

### Core Types

```typescript
// Dashboard configuration types
interface DashboardConfig {
  id: string;
  name: string;
  description?: string;
  layout: DashboardLayout;
  widgets: WidgetConfig[];
  filters: FilterConfig[];
  refreshInterval?: number;
  timeRange: TimeRange;
  permissions: DashboardPermissions;
}

interface DashboardLayout {
  type: "grid" | "flex" | "masonry";
  columns: number;
  gap: number;
  breakpoints: Record<string, LayoutBreakpoint>;
}

interface LayoutBreakpoint {
  columns: number;
  gap: number;
}

interface WidgetConfig {
  id: string;
  type: WidgetType;
  title: string;
  position: WidgetPosition;
  size: WidgetSize;
  dataSource: DataSource;
  visualization: VisualizationConfig;
  filters?: string[];
  refreshInterval?: number;
}

type WidgetType =
  | "chart"
  | "metric"
  | "table"
  | "map"
  | "text"
  | "iframe"
  | "custom";

interface WidgetPosition {
  x: number;
  y: number;
  row?: number;
  column?: number;
}

interface WidgetSize {
  width: number;
  height: number;
  minWidth?: number;
  minHeight?: number;
  maxWidth?: number;
  maxHeight?: number;
}

interface DataSource {
  type: "rest" | "graphql" | "websocket" | "static";
  endpoint: string;
  method?: string;
  params?: Record<string, any>;
  query?: string;
  transform?: (data: any) => any;
}

interface VisualizationConfig {
  chartType: ChartType;
  options: ChartOptions;
  colorScheme?: string[];
  responsive?: boolean;
}

type ChartType =
  | "line"
  | "bar"
  | "pie"
  | "doughnut"
  | "area"
  | "scatter"
  | "heatmap"
  | "gauge"
  | "funnel";

interface TimeRange {
  start: Date | string;
  end: Date | string;
  preset?: TimeRangePreset;
}

type TimeRangePreset =
  | "last_hour"
  | "last_24h"
  | "last_7d"
  | "last_30d"
  | "last_90d"
  | "custom";

interface DashboardPermissions {
  canEdit: boolean;
  canDelete: boolean;
  canShare: boolean;
  canExport: boolean;
  viewers: string[];
  editors: string[];
}
```

### Dashboard Store

```typescript
import create from "zustand";
import { devtools, persist } from "zustand/middleware";
import { immer } from "zustand/middleware/immer";

interface DashboardState {
  dashboards: Map<string, DashboardConfig>;
  activeDashboard: string | null;
  widgetData: Map<string, any>;
  filters: Map<string, FilterValue>;
  isLoading: Map<string, boolean>;
  errors: Map<string, Error>;

  // Actions
  loadDashboard: (id: string) => Promise<void>;
  updateWidget: (widgetId: string, data: any) => void;
  addWidget: (widget: WidgetConfig) => void;
  removeWidget: (widgetId: string) => void;
  updateFilter: (filterId: string, value: FilterValue) => void;
  clearFilters: () => void;
  refreshWidget: (widgetId: string) => Promise<void>;
  refreshAll: () => Promise<void>;
  setTimeRange: (range: TimeRange) => void;
}

export const useDashboardStore = create<DashboardState>()(
  devtools(
    persist(
      immer((set, get) => ({
        dashboards: new Map(),
        activeDashboard: null,
        widgetData: new Map(),
        filters: new Map(),
        isLoading: new Map(),
        errors: new Map(),

        loadDashboard: async (id) => {
          set((state) => {
            state.activeDashboard = id;
            state.isLoading.set(id, true);
          });

          try {
            const dashboard = await dashboardApi.get(id);
            set((state) => {
              state.dashboards.set(id, dashboard);
              state.isLoading.set(id, false);
            });

            // Load widget data
            await get().refreshAll();
          } catch (error) {
            set((state) => {
              state.errors.set(id, error as Error);
              state.isLoading.set(id, false);
            });
          }
        },

        updateWidget: (widgetId, data) => {
          set((state) => {
            state.widgetData.set(widgetId, data);
          });
        },

        addWidget: (widget) => {
          set((state) => {
            const dashboard = state.dashboards.get(state.activeDashboard!);
            if (dashboard) {
              dashboard.widgets.push(widget);
            }
          });
        },

        removeWidget: (widgetId) => {
          set((state) => {
            const dashboard = state.dashboards.get(state.activeDashboard!);
            if (dashboard) {
              dashboard.widgets = dashboard.widgets.filter(
                (w) => w.id !== widgetId,
              );
            }
            state.widgetData.delete(widgetId);
          });
        },

        updateFilter: (filterId, value) => {
          set((state) => {
            state.filters.set(filterId, value);
          });
          get().refreshAll();
        },

        clearFilters: () => {
          set((state) => {
            state.filters.clear();
          });
          get().refreshAll();
        },

        refreshWidget: async (widgetId) => {
          const dashboard = get().dashboards.get(get().activeDashboard!);
          const widget = dashboard?.widgets.find((w) => w.id === widgetId);
          if (!widget) return;

          set((state) => {
            state.isLoading.set(widgetId, true);
          });

          try {
            const data = await fetchWidgetData(widget, get().filters);
            set((state) => {
              state.widgetData.set(widgetId, data);
              state.isLoading.set(widgetId, false);
            });
          } catch (error) {
            set((state) => {
              state.errors.set(widgetId, error as Error);
              state.isLoading.set(widgetId, false);
            });
          }
        },

        refreshAll: async () => {
          const dashboard = get().dashboards.get(get().activeDashboard!);
          if (!dashboard) return;

          await Promise.all(
            dashboard.widgets.map((widget) => get().refreshWidget(widget.id)),
          );
        },

        setTimeRange: (range) => {
          set((state) => {
            const dashboard = state.dashboards.get(state.activeDashboard!);
            if (dashboard) {
              dashboard.timeRange = range;
            }
          });
          get().refreshAll();
        },
      })),
      {
        name: "dashboard-storage",
        partialize: (state) => ({
          activeDashboard: state.activeDashboard,
          filters: state.filters,
        }),
      },
    ),
  ),
);
```

## Real-Time Data Updates

### WebSocket Manager

```typescript
interface WebSocketMessage {
  type: "update" | "subscribe" | "unsubscribe";
  channel: string;
  data?: any;
}

class DashboardWebSocketManager {
  private ws: WebSocket | null = null;
  private subscriptions: Map<string, Set<(data: any) => void>> = new Map();
  private reconnectAttempts = 0;
  private maxReconnectAttempts = 5;
  private reconnectDelay = 1000;

  constructor(private url: string) {}

  connect(): Promise<void> {
    return new Promise((resolve, reject) => {
      this.ws = new WebSocket(this.url);

      this.ws.onopen = () => {
        console.log("WebSocket connected");
        this.reconnectAttempts = 0;
        resolve();
      };

      this.ws.onmessage = (event) => {
        const message: WebSocketMessage = JSON.parse(event.data);
        this.handleMessage(message);
      };

      this.ws.onerror = (error) => {
        console.error("WebSocket error:", error);
        reject(error);
      };

      this.ws.onclose = () => {
        console.log("WebSocket closed");
        this.attemptReconnect();
      };
    });
  }

  subscribe(channel: string, callback: (data: any) => void): () => void {
    if (!this.subscriptions.has(channel)) {
      this.subscriptions.set(channel, new Set());
      this.send({
        type: "subscribe",
        channel,
      });
    }

    this.subscriptions.get(channel)!.add(callback);

    // Return unsubscribe function
    return () => {
      const callbacks = this.subscriptions.get(channel);
      if (callbacks) {
        callbacks.delete(callback);
        if (callbacks.size === 0) {
          this.unsubscribe(channel);
        }
      }
    };
  }

  private unsubscribe(channel: string): void {
    this.subscriptions.delete(channel);
    this.send({
      type: "unsubscribe",
      channel,
    });
  }

  private handleMessage(message: WebSocketMessage): void {
    const callbacks = this.subscriptions.get(message.channel);
    if (callbacks) {
      callbacks.forEach((callback) => callback(message.data));
    }
  }

  private send(message: WebSocketMessage): void {
    if (this.ws?.readyState === WebSocket.OPEN) {
      this.ws.send(JSON.stringify(message));
    }
  }

  private attemptReconnect(): void {
    if (this.reconnectAttempts < this.maxReconnectAttempts) {
      this.reconnectAttempts++;
      const delay =
        this.reconnectDelay * Math.pow(2, this.reconnectAttempts - 1);

      setTimeout(() => {
        console.log(`Reconnecting... (attempt ${this.reconnectAttempts})`);
        this.connect();
      }, delay);
    }
  }

  disconnect(): void {
    this.subscriptions.clear();
    if (this.ws) {
      this.ws.close();
      this.ws = null;
    }
  }
}

// React hook for WebSocket subscriptions
export function useWebSocket(channel: string) {
  const [data, setData] = useState<any>(null);
  const wsManager = useRef<DashboardWebSocketManager | null>(null);

  useEffect(() => {
    if (!wsManager.current) {
      wsManager.current = new DashboardWebSocketManager(
        process.env.REACT_APP_WS_URL!,
      );
      wsManager.current.connect();
    }

    const unsubscribe = wsManager.current.subscribe(channel, (newData) => {
      setData(newData);
    });

    return () => {
      unsubscribe();
    };
  }, [channel]);

  return data;
}
```

### Real-Time Widget Component

```typescript
interface RealTimeWidgetProps {
  widget: WidgetConfig;
  channel: string;
}

export const RealTimeWidget: React.FC<RealTimeWidgetProps> = ({
  widget,
  channel,
}) => {
  const [data, setData] = useState<any[]>([]);
  const realtimeData = useWebSocket(channel);
  const maxDataPoints = 50;

  useEffect(() => {
    if (realtimeData) {
      setData((prev) => {
        const newData = [...prev, realtimeData];
        // Keep only last N data points
        return newData.slice(-maxDataPoints);
      });
    }
  }, [realtimeData]);

  return (
    <div className="realtime-widget">
      <div className="widget-header">
        <h3>{widget.title}</h3>
        <div className="status-indicator active">Live</div>
      </div>

      <div className="widget-content">
        <LineChart
          data={data}
          options={{
            animation: {
              duration: 750,
              easing: 'linear',
            },
            scales: {
              x: {
                type: 'time',
                time: {
                  unit: 'second',
                },
              },
            },
          }}
        />
      </div>
    </div>
  );
};
```

## Chart Components

### Unified Chart Component

```typescript
import {
  Chart as ChartJS,
  CategoryScale,
  LinearScale,
  PointElement,
  LineElement,
  BarElement,
  ArcElement,
  Title,
  Tooltip,
  Legend,
  Filler,
} from 'chart.js';
import { Line, Bar, Pie, Doughnut } from 'react-chartjs-2';

ChartJS.register(
  CategoryScale,
  LinearScale,
  PointElement,
  LineElement,
  BarElement,
  ArcElement,
  Title,
  Tooltip,
  Legend,
  Filler
);

interface ChartComponentProps {
  type: ChartType;
  data: ChartData;
  options?: ChartOptions;
  height?: number;
}

interface ChartData {
  labels: string[];
  datasets: ChartDataset[];
}

interface ChartDataset {
  label: string;
  data: number[];
  backgroundColor?: string | string[];
  borderColor?: string | string[];
  borderWidth?: number;
  fill?: boolean;
}

export const ChartComponent: React.FC<ChartComponentProps> = ({
  type,
  data,
  options = {},
  height = 300,
}) => {
  const defaultOptions: ChartOptions = {
    responsive: true,
    maintainAspectRatio: false,
    plugins: {
      legend: {
        position: 'bottom',
      },
      tooltip: {
        mode: 'index',
        intersect: false,
      },
    },
    ...options,
  };

  const chartProps = {
    data,
    options: defaultOptions,
    height,
  };

  switch (type) {
    case 'line':
      return <Line {...chartProps} />;
    case 'bar':
      return <Bar {...chartProps} />;
    case 'pie':
      return <Pie {...chartProps} />;
    case 'doughnut':
      return <Doughnut {...chartProps} />;
    default:
      return null;
  }
};
```

### Advanced Chart Examples

```typescript
// Multi-axis chart
export const MultiAxisChart: React.FC = () => {
  const data = {
    labels: ['Jan', 'Feb', 'Mar', 'Apr', 'May', 'Jun'],
    datasets: [
      {
        label: 'Revenue',
        data: [65000, 59000, 80000, 81000, 56000, 95000],
        borderColor: 'rgb(75, 192, 192)',
        backgroundColor: 'rgba(75, 192, 192, 0.2)',
        yAxisID: 'y',
      },
      {
        label: 'Orders',
        data: [280, 245, 320, 310, 260, 380],
        borderColor: 'rgb(255, 99, 132)',
        backgroundColor: 'rgba(255, 99, 132, 0.2)',
        yAxisID: 'y1',
      },
    ],
  };

  const options = {
    scales: {
      y: {
        type: 'linear',
        display: true,
        position: 'left',
        title: {
          display: true,
          text: 'Revenue ($)',
        },
      },
      y1: {
        type: 'linear',
        display: true,
        position: 'right',
        title: {
          display: true,
          text: 'Orders',
        },
        grid: {
          drawOnChartArea: false,
        },
      },
    },
  };

  return <Line data={data} options={options} />;
};

// Stacked area chart
export const StackedAreaChart: React.FC<{ data: TimeSeriesData[] }> = ({
  data,
}) => {
  const chartData = {
    labels: data.map((d) => d.timestamp),
    datasets: [
      {
        label: 'Mobile',
        data: data.map((d) => d.mobile),
        fill: true,
        backgroundColor: 'rgba(255, 99, 132, 0.5)',
        borderColor: 'rgb(255, 99, 132)',
      },
      {
        label: 'Desktop',
        data: data.map((d) => d.desktop),
        fill: true,
        backgroundColor: 'rgba(54, 162, 235, 0.5)',
        borderColor: 'rgb(54, 162, 235)',
      },
      {
        label: 'Tablet',
        data: data.map((d) => d.tablet),
        fill: true,
        backgroundColor: 'rgba(75, 192, 192, 0.5)',
        borderColor: 'rgb(75, 192, 192)',
      },
    ],
  };

  const options = {
    scales: {
      y: {
        stacked: true,
      },
    },
  };

  return <Line data={chartData} options={options} />;
};
```

### Custom Chart Components

```typescript
// Gauge chart
interface GaugeChartProps {
  value: number;
  min: number;
  max: number;
  label: string;
  thresholds?: Threshold[];
}

interface Threshold {
  value: number;
  color: string;
}

export const GaugeChart: React.FC<GaugeChartProps> = ({
  value,
  min,
  max,
  label,
  thresholds = [],
}) => {
  const percentage = ((value - min) / (max - min)) * 100;
  const rotation = (percentage / 100) * 180 - 90;

  const getColor = () => {
    for (const threshold of thresholds) {
      if (value <= threshold.value) {
        return threshold.color;
      }
    }
    return thresholds[thresholds.length - 1]?.color || '#3b82f6';
  };

  return (
    <div className="gauge-chart">
      <svg viewBox="0 0 200 120" className="gauge-svg">
        <path
          d="M 20 100 A 80 80 0 0 1 180 100"
          fill="none"
          stroke="#e5e7eb"
          strokeWidth="20"
          strokeLinecap="round"
        />
        <path
          d="M 20 100 A 80 80 0 0 1 180 100"
          fill="none"
          stroke={getColor()}
          strokeWidth="20"
          strokeLinecap="round"
          strokeDasharray={`${(percentage / 100) * 251.2} 251.2`}
        />
        <line
          x1="100"
          y1="100"
          x2="100"
          y2="30"
          stroke="#1f2937"
          strokeWidth="3"
          strokeLinecap="round"
          transform={`rotate(${rotation} 100 100)`}
        />
        <circle cx="100" cy="100" r="5" fill="#1f2937" />
      </svg>
      <div className="gauge-value">{value}</div>
      <div className="gauge-label">{label}</div>
    </div>
  );
};

// Heatmap component
interface HeatmapProps {
  data: HeatmapData[][];
  xLabels: string[];
  yLabels: string[];
  colorScale?: (value: number) => string;
}

interface HeatmapData {
  value: number;
  label?: string;
}

export const Heatmap: React.FC<HeatmapProps> = ({
  data,
  xLabels,
  yLabels,
  colorScale = defaultColorScale,
}) => {
  const cellWidth = 100 / xLabels.length;
  const cellHeight = 100 / yLabels.length;

  return (
    <div className="heatmap">
      <div className="heatmap-y-labels">
        {yLabels.map((label, i) => (
          <div key={i} className="y-label">
            {label}
          </div>
        ))}
      </div>

      <div className="heatmap-grid">
        {data.map((row, y) =>
          row.map((cell, x) => (
            <div
              key={`${x}-${y}`}
              className="heatmap-cell"
              style={{
                left: `${x * cellWidth}%`,
                top: `${y * cellHeight}%`,
                width: `${cellWidth}%`,
                height: `${cellHeight}%`,
                backgroundColor: colorScale(cell.value),
              }}
              title={cell.label || cell.value.toString()}
            >
              {cell.value}
            </div>
          ))
        )}
      </div>

      <div className="heatmap-x-labels">
        {xLabels.map((label, i) => (
          <div key={i} className="x-label">
            {label}
          </div>
        ))}
      </div>
    </div>
  );
};

function defaultColorScale(value: number): string {
  const scales = [
    { threshold: 0.2, color: '#dcfce7' },
    { threshold: 0.4, color: '#86efac' },
    { threshold: 0.6, color: '#4ade80' },
    { threshold: 0.8, color: '#22c55e' },
    { threshold: 1.0, color: '#16a34a' },
  ];

  for (const scale of scales) {
    if (value <= scale.threshold) {
      return scale.color;
    }
  }
  return scales[scales.length - 1].color;
}
```

## Filter System

### Filter Types and Configuration

```typescript
interface FilterConfig {
  id: string;
  type: FilterType;
  label: string;
  options?: FilterOption[];
  defaultValue?: any;
  multi?: boolean;
  searchable?: boolean;
  dependencies?: string[];
}

type FilterType =
  | 'select'
  | 'multiselect'
  | 'daterange'
  | 'slider'
  | 'search'
  | 'checkbox';

interface FilterOption {
  value: string | number;
  label: string;
  disabled?: boolean;
}

interface FilterValue {
  filterId: string;
  value: any;
}

// Filter components
export const FilterBar: React.FC<{ filters: FilterConfig[] }> = ({
  filters,
}) => {
  const { filters: activeFilters, updateFilter, clearFilters } =
    useDashboardStore();

  return (
    <div className="filter-bar">
      {filters.map((filter) => (
        <FilterInput
          key={filter.id}
          config={filter}
          value={activeFilters.get(filter.id)}
          onChange={(value) => updateFilter(filter.id, value)}
        />
      ))}

      {activeFilters.size > 0 && (
        <button onClick={clearFilters} className="clear-filters-btn">
          Clear All
        </button>
      )}
    </div>
  );
};

const FilterInput: React.FC<{
  config: FilterConfig;
  value: any;
  onChange: (value: any) => void;
}> = ({ config, value, onChange }) => {
  switch (config.type) {
    case 'select':
      return (
        <Select
          label={config.label}
          options={config.options!}
          value={value}
          onChange={onChange}
          searchable={config.searchable}
        />
      );

    case 'multiselect':
      return (
        <MultiSelect
          label={config.label}
          options={config.options!}
          value={value || []}
          onChange={onChange}
        />
      );

    case 'daterange':
      return (
        <DateRangePicker
          label={config.label}
          value={value}
          onChange={onChange}
        />
      );

    case 'slider':
      return (
        <Slider
          label={config.label}
          value={value}
          onChange={onChange}
          min={config.options![0].value as number}
          max={config.options![config.options!.length - 1].value as number}
        />
      );

    case 'search':
      return (
        <SearchInput
          label={config.label}
          value={value}
          onChange={onChange}
        />
      );

    default:
      return null;
  }
};
```

### Date Range Picker

```typescript
import { DatePicker } from '@mui/x-date-pickers';
import { addDays, subDays, startOfMonth, endOfMonth } from 'date-fns';

interface DateRangePickerProps {
  label: string;
  value: { start: Date; end: Date } | null;
  onChange: (value: { start: Date; end: Date }) => void;
}

const presets: Record<string, () => { start: Date; end: Date }> = {
  today: () => ({
    start: new Date(),
    end: new Date(),
  }),
  yesterday: () => ({
    start: subDays(new Date(), 1),
    end: subDays(new Date(), 1),
  }),
  last7days: () => ({
    start: subDays(new Date(), 7),
    end: new Date(),
  }),
  last30days: () => ({
    start: subDays(new Date(), 30),
    end: new Date(),
  }),
  thisMonth: () => ({
    start: startOfMonth(new Date()),
    end: endOfMonth(new Date()),
  }),
};

export const DateRangePicker: React.FC<DateRangePickerProps> = ({
  label,
  value,
  onChange,
}) => {
  const [tempValue, setTempValue] = useState(value);
  const [showPresets, setShowPresets] = useState(false);

  const handlePresetClick = (presetKey: string) => {
    const range = presets[presetKey]();
    setTempValue(range);
    onChange(range);
    setShowPresets(false);
  };

  return (
    <div className="date-range-picker">
      <label>{label}</label>

      <div className="date-inputs">
        <DatePicker
          label="Start Date"
          value={tempValue?.start}
          onChange={(date) => {
            if (date && tempValue) {
              const newValue = { ...tempValue, start: date };
              setTempValue(newValue);
              onChange(newValue);
            }
          }}
        />
        <span>to</span>
        <DatePicker
          label="End Date"
          value={tempValue?.end}
          onChange={(date) => {
            if (date && tempValue) {
              const newValue = { ...tempValue, end: date };
              setTempValue(newValue);
              onChange(newValue);
            }
          }}
        />
      </div>

      <button
        className="presets-toggle"
        onClick={() => setShowPresets(!showPresets)}
      >
        Quick Select
      </button>

      {showPresets && (
        <div className="presets-menu">
          {Object.keys(presets).map((key) => (
            <button
              key={key}
              onClick={() => handlePresetClick(key)}
              className="preset-option"
            >
              {key.replace(/([A-Z])/g, ' $1').trim()}
            </button>
          ))}
        </div>
      )}
    </div>
  );
};
```

## Data Aggregation

### Aggregation Engine

```typescript
interface AggregationConfig {
  field: string;
  operation: AggregationOperation;
  groupBy?: string;
  filter?: (item: any) => boolean;
}

type AggregationOperation = "sum" | "avg" | "min" | "max" | "count" | "median";

class DataAggregator {
  static aggregate<T>(
    data: T[],
    configs: AggregationConfig[],
  ): Record<string, any> {
    const results: Record<string, any> = {};

    for (const config of configs) {
      const filteredData = config.filter ? data.filter(config.filter) : data;

      if (config.groupBy) {
        results[config.field] = this.groupAndAggregate(
          filteredData,
          config.groupBy,
          config.field,
          config.operation,
        );
      } else {
        results[config.field] = this.performOperation(
          filteredData.map((item: any) => item[config.field]),
          config.operation,
        );
      }
    }

    return results;
  }

  private static groupAndAggregate<T>(
    data: T[],
    groupBy: string,
    field: string,
    operation: AggregationOperation,
  ): Record<string, number> {
    const groups: Record<string, number[]> = {};

    for (const item of data) {
      const key = (item as any)[groupBy];
      if (!groups[key]) {
        groups[key] = [];
      }
      groups[key].push((item as any)[field]);
    }

    const result: Record<string, number> = {};
    for (const [key, values] of Object.entries(groups)) {
      result[key] = this.performOperation(values, operation);
    }

    return result;
  }

  private static performOperation(
    values: number[],
    operation: AggregationOperation,
  ): number {
    switch (operation) {
      case "sum":
        return values.reduce((a, b) => a + b, 0);

      case "avg":
        return values.reduce((a, b) => a + b, 0) / values.length;

      case "min":
        return Math.min(...values);

      case "max":
        return Math.max(...values);

      case "count":
        return values.length;

      case "median":
        const sorted = [...values].sort((a, b) => a - b);
        const mid = Math.floor(sorted.length / 2);
        return sorted.length % 2 === 0
          ? (sorted[mid - 1] + sorted[mid]) / 2
          : sorted[mid];

      default:
        return 0;
    }
  }
}

// Usage example
const salesData = [
  { date: "2024-01-01", region: "North", revenue: 1000 },
  { date: "2024-01-01", region: "South", revenue: 1500 },
  { date: "2024-01-02", region: "North", revenue: 1200 },
  { date: "2024-01-02", region: "South", revenue: 1800 },
];

const aggregated = DataAggregator.aggregate(salesData, [
  { field: "revenue", operation: "sum" },
  { field: "revenue", operation: "avg", groupBy: "region" },
  { field: "revenue", operation: "max", groupBy: "date" },
]);
```

## Performance Optimization

### Virtual Scrolling for Large Tables

```typescript
import { useVirtual } from 'react-virtual';

interface VirtualTableProps {
  data: any[];
  columns: ColumnDef[];
  rowHeight: number;
}

export const VirtualTable: React.FC<VirtualTableProps> = ({
  data,
  columns,
  rowHeight,
}) => {
  const parentRef = useRef<HTMLDivElement>(null);

  const rowVirtualizer = useVirtual({
    size: data.length,
    parentRef,
    estimateSize: useCallback(() => rowHeight, [rowHeight]),
    overscan: 5,
  });

  return (
    <div ref={parentRef} className="virtual-table-container">
      <table className="virtual-table">
        <thead>
          <tr>
            {columns.map((col) => (
              <th key={col.id}>{col.header}</th>
            ))}
          </tr>
        </thead>
        <tbody
          style={{
            height: `${rowVirtualizer.totalSize}px`,
            position: 'relative',
          }}
        >
          {rowVirtualizer.virtualItems.map((virtualRow) => {
            const row = data[virtualRow.index];
            return (
              <tr
                key={virtualRow.index}
                style={{
                  position: 'absolute',
                  top: 0,
                  left: 0,
                  width: '100%',
                  height: `${virtualRow.size}px`,
                  transform: `translateY(${virtualRow.start}px)`,
                }}
              >
                {columns.map((col) => (
                  <td key={col.id}>{col.accessor(row)}</td>
                ))}
              </tr>
            );
          })}
        </tbody>
      </table>
    </div>
  );
};
```

### Memoization and Data Processing

```typescript
// Memoized data processing
export function useProcessedData(
  rawData: any[],
  filters: Map<string, FilterValue>,
  aggregations: AggregationConfig[],
) {
  return useMemo(() => {
    // Apply filters
    let filteredData = rawData;
    filters.forEach((filter) => {
      filteredData = applyFilter(filteredData, filter);
    });

    // Apply aggregations
    const aggregatedData = DataAggregator.aggregate(filteredData, aggregations);

    return { filteredData, aggregatedData };
  }, [rawData, filters, aggregations]);
}

// Debounced filter updates
export function useDebouncedFilter(delay: number = 300) {
  const { updateFilter } = useDashboardStore();

  return useMemo(
    () =>
      debounce((filterId: string, value: any) => {
        updateFilter(filterId, value);
      }, delay),
    [updateFilter, delay],
  );
}
```

## Export Functionality

### Export Service

```typescript
class DashboardExportService {
  async exportToPDF(dashboardId: string): Promise<Blob> {
    const dashboard = await dashboardApi.get(dashboardId);
    const html = await this.generateHTML(dashboard);

    // Use html2pdf or similar library
    const pdf = await html2pdf()
      .set({
        margin: 10,
        filename: `dashboard-${dashboardId}.pdf`,
        image: { type: "jpeg", quality: 0.98 },
        html2canvas: { scale: 2 },
        jsPDF: { unit: "mm", format: "a4", orientation: "landscape" },
      })
      .from(html)
      .output("blob");

    return pdf;
  }

  async exportToExcel(widgetId: string): Promise<Blob> {
    const data = useDashboardStore.getState().widgetData.get(widgetId);

    // Use exceljs or similar library
    const workbook = new ExcelJS.Workbook();
    const worksheet = workbook.addWorksheet("Data");

    // Add headers
    const headers = Object.keys(data[0]);
    worksheet.addRow(headers);

    // Add data
    data.forEach((row: any) => {
      worksheet.addRow(Object.values(row));
    });

    const buffer = await workbook.xlsx.writeBuffer();
    return new Blob([buffer], {
      type: "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet",
    });
  }

  async exportToCSV(widgetId: string): Promise<Blob> {
    const data = useDashboardStore.getState().widgetData.get(widgetId);
    const csv = this.convertToCSV(data);

    return new Blob([csv], { type: "text/csv;charset=utf-8;" });
  }

  private convertToCSV(data: any[]): string {
    if (data.length === 0) return "";

    const headers = Object.keys(data[0]);
    const rows = data.map((row) =>
      headers
        .map((header) => {
          const value = row[header];
          // Escape quotes and wrap in quotes if contains comma
          return typeof value === "string" && value.includes(",")
            ? `"${value.replace(/"/g, '""')}"`
            : value;
        })
        .join(","),
    );

    return [headers.join(","), ...rows].join("\n");
  }

  private async generateHTML(dashboard: DashboardConfig): Promise<string> {
    // Generate HTML representation of dashboard
    return `
      <html>
        <head>
          <title>${dashboard.name}</title>
          <style>${this.getExportStyles()}</style>
        </head>
        <body>
          <h1>${dashboard.name}</h1>
          ${dashboard.widgets.map((w) => this.renderWidget(w)).join("\n")}
        </body>
      </html>
    `;
  }

  private getExportStyles(): string {
    return `
      body { font-family: Arial, sans-serif; }
      .widget { margin: 20px; page-break-inside: avoid; }
      h2 { color: #333; }
      table { width: 100%; border-collapse: collapse; }
      th, td { padding: 8px; border: 1px solid #ddd; }
    `;
  }

  private renderWidget(widget: WidgetConfig): string {
    // Render widget to HTML
    return `
      <div class="widget">
        <h2>${widget.title}</h2>
        <!-- Widget content -->
      </div>
    `;
  }
}
```

## Key Takeaways

### 1. **Component-Based Architecture**

Build reusable widget components with standardized interfaces. Use composition over inheritance for maximum flexibility.

### 2. **Real-Time Updates**

Implement WebSocket connections for live data streams. Use efficient update strategies to prevent performance degradation.

### 3. **Efficient Data Processing**

Process data on the frontend only when necessary. Use aggregation servers for heavy computations. Cache processed results.

### 4. **Advanced Filtering**

Provide comprehensive filtering with date ranges, multi-select, and search. Make filters composable and url-serializable.

### 5. **Chart Optimization**

Use canvas-based charts for large datasets. Implement progressive rendering and data decimation for performance.

### 6. **State Management**

Centralize dashboard state with Zustand or Redux. Persist user preferences and filter states locally.

### 7. **Responsive Design**

Implement fluid grid layouts that adapt to screen sizes. Make charts and tables mobile-friendly with horizontal scrolling.

### 8. **Export Capabilities**

Support multiple export formats (PDF, Excel, CSV). Generate print-friendly versions of dashboards.

### 9. **Accessibility**

Ensure keyboard navigation works for all interactions. Provide alternative data representations for screen readers.

### 10. **Performance Monitoring**

Track render times and data fetch latencies. Implement lazy loading for off-screen widgets. Use virtual scrolling for large lists.

---

**Further Reading:**

- Chart.js Documentation
- D3.js for Advanced Visualizations
- React Query for Data Fetching
- WebSocket Best Practices
