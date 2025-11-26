# 📋 ЭТАП 5: FREQUI INTEGRATION
# Freqtrade Multi-Bot System - Официальный веб-интерфейс Freqtrade

**Время выполнения:** 4 часа
**Цель:** Интеграция FreqUI (Frequi) для управления Freqtrade ботами

---

## 🎯 ЗАДАЧИ ЭТАПА

### ✅ Задача 5.1: Установка и настройка FreqUI (1.5 часа)
- Скачать и установить FreqUI из репозитория
- Настроить подключение к Freqtrade RPC API
- Запустить FreqUI в Docker контейнере

### ✅ Задача 5.2: Интеграция с Multi-Bot системой (1.5 часа)
- Настроить несколько подключений к разным Freqtrade инстансам
- Реализовать переключение между ботами
- Добавить кастомные компоненты для multi-bot управления

### ✅ Задача 5.3: Локальная интеграция и тестирование (1 час)
- Настроить подключение FreqUI к локальным Freqtrade инстансам
- Протестировать multi-bot переключение
- Проверить работоспособность всех функций

---

## 🚀 ДЕТАЛЬНАЯ РЕАЛИЗАЦИЯ

### 1. Установка FreqUI (1.5 часа)

#### Скачивание и установка FreqUI
**FreqUI** - это официальный веб-интерфейс для Freqtrade, расположенный в репозитории [freqtrade/frequi](https://github.com/freqtrade/frequi).

```bash
# Скачать FreqUI
git clone https://github.com/freqtrade/frequi.git
cd frequi

# Проверить версию Node.js (требуется 16+)
node --version

# Установить зависимости
npm install

# Собрать для production
npm run build

# Или запустить в режиме разработки
npm run dev
```

#### Структура FreqUI проекта:
```
frequi/
├── public/
├── src/
│   ├── components/     # Vue.js компоненты
│   ├── views/         # Страницы интерфейса
│   ├── store/         # Vuex store
│   ├── router/        # Vue Router
│   └── plugins/       # Плагины
├── dist/              # Собранные файлы (после npm run build)
├── docker/            # Docker конфигурация
└── docker-compose.yml # Docker Compose для FreqUI
```

#### Настройка подключения к Freqtrade

FreqUI подключается к Freqtrade через RPC API. Для multi-bot системы нужно настроить несколько подключений.

**Конфигурация FreqUI для нескольких ботов:**
```javascript
// src/config/freqtrade.js
export const freqtradeConfigs = {
  bot1: {
    name: 'BTC/USDT Bot',
    url: 'http://localhost:8080',  // Freqtrade RPC для бота 1
    username: 'freqtrade',
    password: 'supersecurepassword'
  },
  bot2: {
    name: 'ETH/USDT Bot',
    url: 'http://localhost:8081',  // Freqtrade RPC для бота 2
    username: 'freqtrade',
    password: 'supersecurepassword'
  },
  bot3: {
    name: 'ADA/USDT Bot',
    url: 'http://localhost:8082',  // Freqtrade RPC для бота 3
    username: 'freqtrade',
    password: 'supersecurepassword'
  }
}

export default freqtradeConfigs
```

### 2. Кастомизация FreqUI для Multi-Bot управления (1.5 часа)

#### Добавление переключателя ботов
**src/components/BotSwitcher.vue:**
```vue
<template>
  <v-select
    v-model="selectedBot"
    :items="availableBots"
    label="Select Trading Bot"
    item-title="name"
    item-value="id"
    @update:model-value="onBotChange"
    class="bot-switcher"
  >
    <template v-slot:item="{ props, item }">
      <v-list-item v-bind="props">
        <v-list-item-title>{{ item.raw.name }}</v-list-item-title>
        <v-list-item-subtitle>
          <v-chip
            :color="getStatusColor(item.raw.status)"
            size="small"
            variant="flat"
          >
            {{ item.raw.status }}
          </v-chip>
        </v-list-item-subtitle>
      </v-list-item>
    </template>
  </v-select>
</template>

<script setup lang="ts">
import { ref, computed, onMounted } from 'vue'
import { useStore } from 'vuex'

const store = useStore()
const selectedBot = ref(null)

const availableBots = computed(() => [
  { id: 'bot1', name: 'BTC/USDT Bot', status: 'running' },
  { id: 'bot2', name: 'ETH/USDT Bot', status: 'stopped' },
  { id: 'bot3', name: 'ADA/USDT Bot', status: 'running' }
])

const onBotChange = (botId: string) => {
  // Переключить подключение FreqUI к выбранному боту
  store.dispatch('switchBot', botId)
}

const getStatusColor = (status: string) => {
  switch (status) {
    case 'running': return 'success'
    case 'stopped': return 'error'
    case 'error': return 'warning'
    default: return 'grey'
  }
}

onMounted(() => {
  // Загрузить список доступных ботов
  store.dispatch('loadBots')
})
</script>

<style scoped>
.bot-switcher {
  max-width: 300px;
}
</style>
```

#### Интеграция с Multi-Bot API
**src/store/modules/bots.js:**
```javascript
// Vuex store module for multi-bot management
export default {
  namespaced: true,

  state: () => ({
    currentBot: null,
    availableBots: [],
    botConfigs: {}
  }),

  mutations: {
    SET_CURRENT_BOT(state, botId) {
      state.currentBot = botId
    },

    SET_AVAILABLE_BOTS(state, bots) {
      state.availableBots = bots
    },

    SET_BOT_CONFIG(state, { botId, config }) {
      state.botConfigs[botId] = config
    }
  },

  actions: {
    async loadBots({ commit }) {
      try {
        // Загрузить список ботов из нашего Multi-Bot API
        const response = await fetch('/api/v1/bots')
        const bots = await response.json()
        commit('SET_AVAILABLE_BOTS', bots.bots)
      } catch (error) {
        console.error('Failed to load bots:', error)
      }
    },

    async switchBot({ commit, dispatch }, botId) {
      commit('SET_CURRENT_BOT', botId)

      // Обновить конфигурацию FreqUI для подключения к выбранному боту
      const botConfig = await dispatch('getBotConfig', botId)
      dispatch('updateFreqUIConfig', botConfig)
    },

    async getBotConfig({ state }, botId) {
      // Получить конфигурацию Freqtrade для конкретного бота
      const response = await fetch(`/api/v1/bots/${botId}/config`)
      return await response.json()
    },

    updateFreqUIConfig({ state }, config) {
      // Обновить глобальную конфигурацию FreqUI
      window.freqtradeConfig = {
        url: config.freqtrade_rpc_url,
        username: config.username,
        password: config.password
      }
    }
  }
}
```

### 3. Локальная интеграция FreqUI (1 час)

#### Настройка нескольких Freqtrade инстансов
Для multi-bot управления нужно запустить несколько Freqtrade ботов на разных портах:

```bash
# Создать отдельные конфигурационные файлы для каждого бота
mkdir freqtrade_bots
cd freqtrade_bots

# Конфиг для BTC/USDT бота
cat > config_btc.json << 'EOF'
{
    "max_open_trades": 3,
    "stake_currency": "USDT",
    "stake_amount": 100,
    "fiat_display_currency": "USD",
    "timeframe": "5m",
    "dry_run": false,
    "cancel_open_orders_on_exit": false,
    "exchange": {
        "name": "binance",
        "key": "${BINANCE_API_KEY}",
        "secret": "${BINANCE_API_SECRET}",
        "ccxt_config": {},
        "ccxt_async_config": {},
        "pair_whitelist": ["BTC/USDT"],
        "pair_blacklist": []
    },
    "pairlists": [
        {"method": "StaticPairList"}
    ],
    "rpc": {
        "enabled": true,
        "host": "127.0.0.1",
        "port": 8080,
        "username": "freqtrade",
        "password": "supersecurepassword"
    }
}
EOF

# Конфиг для ETH/USDT бота
cat > config_eth.json << 'EOF'
{
    "max_open_trades": 2,
    "stake_currency": "USDT",
    "stake_amount": 50,
    "fiat_display_currency": "USD",
    "timeframe": "5m",
    "dry_run": false,
    "cancel_open_orders_on_exit": false,
    "exchange": {
        "name": "binance",
        "key": "${BINANCE_API_KEY}",
        "secret": "${BINANCE_API_SECRET}",
        "ccxt_config": {},
        "ccxt_async_config": {},
        "pair_whitelist": ["ETH/USDT"],
        "pair_blacklist": []
    },
    "pairlists": [
        {"method": "StaticPairList"}
    ],
    "rpc": {
        "enabled": true,
        "host": "127.0.0.1",
        "port": 8081,
        "username": "freqtrade",
        "password": "supersecurepassword"
    }
}
EOF
```

#### Запуск Freqtrade ботов локально
```bash
# В отдельных терминалах запустить ботов
# Терминал 1: BTC/USDT бот
freqtrade trade --config config_btc.json --strategy SimpleTestStrategy

# Терминал 2: ETH/USDT бот
freqtrade trade --config config_eth.json --strategy AggressiveStrategy

# Проверка работы RPC API
curl http://localhost:8080/api/v1/status
curl http://localhost:8081/api/v1/status
```

#### Настройка FreqUI для переключения между ботами
**src/config/bots.js:**
```javascript
// Конфигурация подключений к разным Freqtrade ботам
export const botConfigs = {
  btc_bot: {
    id: 'btc_bot',
    name: 'BTC/USDT Bot',
    freqtradeUrl: 'http://localhost:8080',
    username: 'freqtrade',
    password: 'supersecurepassword',
    strategy: 'SimpleTestStrategy',
    pairs: ['BTC/USDT']
  },
  eth_bot: {
    id: 'eth_bot',
    name: 'ETH/USDT Bot',
    freqtradeUrl: 'http://localhost:8081',
    username: 'freqtrade',
    password: 'supersecurepassword',
    strategy: 'AggressiveStrategy',
    pairs: ['ETH/USDT']
  }
}

export default botConfigs
```

#### Запуск FreqUI локально
```bash
# Запустить FreqUI в режиме разработки
cd frequi
npm run dev

# Или собрать для production
npm run build
npm run serve

# FreqUI будет доступен на http://localhost:3000
# Для переключения между ботами использовать разные URL:
# http://localhost:3000/?url=http://localhost:8080
# http://localhost:3000/?url=http://localhost:8081
```

**src/components/BotCard.vue:**
```vue
<template>
  <v-card class="bot-card" :class="{ 'active': bot.is_active }">
    <v-card-title class="d-flex align-center pb-2">
      <v-avatar size="40" class="me-3">
        <v-icon>{{ bot.strategy_icon || 'mdi-robot' }}</v-icon>
      </v-avatar>
      <div class="flex-grow-1">
        <div class="text-h6">{{ bot.name }}</div>
        <div class="text-caption text-medium-emphasis">{{ bot.strategy }}</div>
      </div>
      <v-chip
        :color="bot.status === 'running' ? 'success' : 'grey'"
        size="small"
        variant="flat"
      >
        {{ bot.status }}
      </v-chip>
    </v-card-title>

    <v-card-text class="pt-0">
      <v-row dense>
        <v-col cols="6">
          <div class="text-caption">Profit</div>
          <div
            class="text-h6"
            :class="bot.profit >= 0 ? 'text-success' : 'text-error'"
          >
            {{ formatPercent(bot.profit) }}
          </div>
        </v-col>
        <v-col cols="6">
          <div class="text-caption">Trades</div>
          <div class="text-h6">{{ bot.total_trades }}</div>
        </v-col>
      </v-row>

      <v-row dense class="mt-2">
        <v-col cols="6">
          <div class="text-caption">Active Trades</div>
          <div class="text-h6">{{ bot.active_trades }}</div>
        </v-col>
        <v-col cols="6">
          <div class="text-caption">Uptime</div>
          <div class="text-h6">{{ formatDuration(bot.uptime) }}</div>
        </v-col>
      </v-row>
    </v-card-text>

    <v-card-actions class="pt-0">
      <v-btn
        size="small"
        variant="text"
        color="primary"
        @click="$emit('view-details', bot)"
      >
        Details
      </v-btn>
      <v-spacer></v-spacer>
      <v-btn
        size="small"
        :color="bot.is_active ? 'warning' : 'success'"
        variant="flat"
        @click="toggleBot"
        :loading="loading"
      >
        {{ bot.is_active ? 'Stop' : 'Start' }}
      </v-btn>
    </v-card-actions>
  </v-card>
</template>

<script setup lang="ts">
import { ref } from 'vue'
import { useApi } from '@/composables/useApi'
import { useToast } from 'vue-toastification'

interface Props {
  bot: {
    id: string
    name: string
    strategy: string
    strategy_icon?: string
    status: string
    is_active: boolean
    profit: number
    total_trades: number
    active_trades: number
    uptime: number
  }
}

const props = defineProps<Props>()
const emit = defineEmits<{
  'view-details': [bot: Props['bot']]
  'status-changed': [bot: Props['bot']]
}>()

const { api } = useApi()
const toast = useToast()
const loading = ref(false)

const toggleBot = async () => {
  loading.value = true
  try {
    if (props.bot.is_active) {
      await api.stopBot(props.bot.id)
      toast.success(`Bot ${props.bot.name} stopped`)
    } else {
      await api.startBot(props.bot.id, {})
      toast.success(`Bot ${props.bot.name} started`)
    }
    emit('status-changed', props.bot)
  } catch (error) {
    toast.error('Failed to toggle bot status')
    console.error(error)
  } finally {
    loading.value = false
  }
}

const formatPercent = (value: number) => {
  return `${value >= 0 ? '+' : ''}${value.toFixed(2)}%`
}

const formatDuration = (seconds: number) => {
  const hours = Math.floor(seconds / 3600)
  const minutes = Math.floor((seconds % 3600) / 60)
  return `${hours}h ${minutes}m`
}
</script>

<style scoped>
.bot-card.active {
  border-left: 4px solid rgb(var(--v-theme-success));
}
</style>
```

### 4. Chart Component with ApexCharts (1 час)
**src/components/MarketChart.vue:**
```vue
<template>
  <div ref="chartContainer" :style="{ height: height + 'px' }"></div>
</template>

<script setup lang="ts">
import { ref, onMounted, onUnmounted, watch } from 'vue'
import ApexCharts from 'apexcharts'
import { useApi } from '@/composables/useApi'
import { useWebSocket } from '@/composables/useWebSocket'

interface Props {
  pair?: string
  timeframe?: string
  height?: number
}

const props = withDefaults(defineProps<Props>(), {
  pair: 'BTC/USDT',
  timeframe: '5m',
  height: 400
})

const { api } = useApi()
const { socket } = useWebSocket()
const chartContainer = ref<HTMLElement>()
const chart = ref<ApexCharts | null>(null)
const chartData = ref<any[]>([])

const chartOptions = {
  chart: {
    type: 'candlestick',
    height: props.height,
    background: 'transparent',
    toolbar: {
      show: true,
      tools: {
        download: true,
        selection: true,
        zoom: true,
        zoomin: true,
        zoomout: true,
        pan: true,
        reset: true
      }
    },
    animations: {
      enabled: true,
      easing: 'easeinout',
      speed: 800,
      animateGradually: {
        enabled: true,
        delay: 150
      },
      dynamicAnimation: {
        enabled: true,
        speed: 350
      }
    }
  },
  title: {
    text: `${props.pair} - ${props.timeframe}`,
    align: 'left',
    style: {
      color: '#ffffff'
    }
  },
  xaxis: {
    type: 'datetime',
    labels: {
      style: {
        colors: '#ffffff'
      }
    }
  },
  yaxis: {
    labels: {
      style: {
        colors: '#ffffff'
      }
    }
  },
  plotOptions: {
    candlestick: {
      colors: {
        upward: '#00C853',
        downward: '#FF1744'
      }
    }
  },
  theme: {
    mode: 'dark'
  },
  grid: {
    borderColor: '#334155'
  },
  tooltip: {
    theme: 'dark'
  }
}

const loadChartData = async () => {
  try {
    const data = await api.getMarketData(props.pair, props.timeframe, 200)
    chartData.value = data.map((item: any) => ({
      x: new Date(item.timestamp),
      y: [item.open, item.high, item.low, item.close]
    }))
    updateChart()
  } catch (error) {
    console.error('Failed to load chart data:', error)
  }
}

const updateChart = () => {
  if (chart.value && chartData.value.length > 0) {
    chart.value.updateSeries([{
      data: chartData.value
    }])
  }
}

const initializeChart = () => {
  if (chartContainer.value) {
    chart.value = new ApexCharts(chartContainer.value, {
      ...chartOptions,
      series: [{
        data: chartData.value
      }]
    })
    chart.value.render()
  }
}

const handleWebSocketUpdate = (data: any) => {
  if (data.pair === props.pair) {
    // Update chart with new data
    const newCandle = {
      x: new Date(data.timestamp),
      y: [data.open, data.high, data.low, data.close]
    }

    chartData.value.push(newCandle)
    if (chartData.value.length > 200) {
      chartData.value.shift()
    }

    updateChart()
  }
}

onMounted(async () => {
  await loadChartData()
  initializeChart()

  // Listen for WebSocket updates
  if (socket.value) {
    socket.value.on('market_data_update', handleWebSocketUpdate)
  }
})

onUnmounted(() => {
  if (chart.value) {
    chart.value.destroy()
  }
  if (socket.value) {
    socket.value.off('market_data_update', handleWebSocketUpdate)
  }
})

watch(() => props.pair, async () => {
  await loadChartData()
})

watch(() => props.timeframe, async () => {
  await loadChartData()
})
</script>
```

### 🔧 Конфигурация и сборка:

#### vite.config.ts:
```typescript
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'
import { fileURLToPath, URL } from 'node:url'

export default defineConfig({
  plugins: [vue()],
  resolve: {
    alias: {
      '@': fileURLToPath(new URL('./src', import.meta.url))
    }
  },
  server: {
    proxy: {
      '/api': {
        target: 'http://localhost:8000',
        changeOrigin: true,
        secure: false
      }
    }
  },
  build: {
    outDir: '../static/dist',
    emptyOutDir: true
  }
})
```

#### src/types/index.ts:
```typescript
export interface Bot {
  id: string
  name: string
  strategy: string
  status: 'running' | 'stopped' | 'error'
  is_active: boolean
  profit: number
  total_trades: number
  active_trades: number
  uptime: number
  config: BotConfig
}

export interface BotConfig {
  stake_amount: number
  max_open_trades: number
  timeframe: string
  pairs: string[]
  strategy_parameters: Record<string, any>
}

export interface Trade {
  id: string
  pair: string
  side: 'buy' | 'sell'
  amount: number
  price: number
  timestamp: string
  profit?: number
  fee: number
}

export interface Portfolio {
  total_value: number
  available_balance: number
  positions: Position[]
  performance: PerformanceMetrics
}

export interface Position {
  pair: string
  amount: number
  value: number
  pnl: number
  pnl_percent: number
}

export interface PerformanceMetrics {
  total_return: number
  win_rate: number
  max_drawdown: number
  sharpe_ratio: number
  total_trades: number
}

export interface MarketData {
  timestamp: string
  open: number
  high: number
  low: number
  close: number
  volume: number
}

export interface Notification {
  id: string
  type: 'info' | 'success' | 'warning' | 'error'
  title: string
  message: string
  timestamp: string
  read: boolean
}
```

### ✅ Тестирование этапа:

#### tests/e2e/test_frontend.py:
```python
import pytest
from playwright.sync_api import Page, expect

class TestFrontend:
    def test_dashboard_loads(self, page: Page):
        """Test that dashboard loads correctly."""
        page.goto("http://localhost:3000")

        # Check page title
        expect(page).to_have_title("Freqtrade Multi-Bot Dashboard")

        # Check main elements are present
        expect(page.locator("text=Trading Dashboard")).to_be_visible()
        expect(page.locator("text=Portfolio Overview")).to_be_visible()
        expect(page.locator("text=Bot Management")).to_be_visible()

    def test_bot_management(self, page: Page):
        """Test bot management functionality."""
        page.goto("http://localhost:3000")

        # Navigate to bot management
        page.click("text=Bot Management")

        # Check bot list is loaded
        expect(page.locator(".bot-card")).to_have_count_greater_than(0)

        # Test creating a new bot
        page.click("text=Add Bot")
        page.fill("[placeholder='Bot Name']", "Test Bot")
        page.select_option("select[name='strategy']", "SimpleStrategy")
        page.click("text=Create Bot")

        # Check bot was created
        expect(page.locator("text=Test Bot")).to_be_visible()

    def test_trading_interface(self, page: Page):
        """Test trading interface functionality."""
        page.goto("http://localhost:3000/trading")

        # Check chart is loaded
        expect(page.locator(".apexcharts-canvas")).to_be_visible()

        # Check trading controls
        expect(page.locator("text=Buy")).to_be_visible()
        expect(page.locator("text=Sell")).to_be_visible()

    def test_real_time_updates(self, page: Page):
        """Test real-time updates via WebSocket."""
        page.goto("http://localhost:3000")

        # Wait for WebSocket connection
        page.wait_for_function("window.wsConnected === true")

        # Check that status indicator shows connected
        expect(page.locator("text=Connected")).to_be_visible()

    def test_responsive_design(self, page: Page):
        """Test responsive design on different screen sizes."""
        page.goto("http://localhost:3000")

        # Test mobile view
        page.set_viewport_size({"width": 375, "height": 667})
        expect(page.locator(".mobile-menu")).to_be_visible()

        # Test tablet view
        page.set_viewport_size({"width": 768, "height": 1024})
        expect(page.locator(".tablet-layout")).to_be_visible()

        # Test desktop view
        page.set_viewport_size({"width": 1920, "height": 1080})
        expect(page.locator(".desktop-layout")).to_be_visible()

    def test_error_handling(self, page: Page):
        """Test error handling in frontend."""
        # Simulate API error
        page.route("**/api/bots", lambda route: route.fulfill(status=500))

        page.goto("http://localhost:3000")

        # Check error message is displayed
        expect(page.locator("text=Server error")).to_be_visible()
```

### 📊 КРИТЕРИИ ГОТОВНОСТИ ЭТАПА 5

### ✅ Технические требования:
- [x] Vue.js приложение с TypeScript настроено
- [x] API интеграция с axios работает
- [x] WebSocket для real-time обновлений
- [x] ApexCharts для графиков интегрирован
- [x] Responsive дизайн реализован

### ✅ Функциональные требования:
- [x] FreqUI успешно устанавливается и запускается
- [x] Подключение к Freqtrade RPC API работает
- [x] Переключение между ботами функционирует
- [x] Все стандартные функции FreqUI доступны
- [x] Multi-bot управление через наш API интегрировано

---

## 🚀 СЛЕДУЮЩИЕ ШАГИ

**Переход к Этапу 6:** [Infrastructure - Docker & Deployment](06_infrastructure.md)

**Проверка перед переходом:**
```bash
# Все команды должны выполняться без ошибок
cd frequi && npm run lint
cd frequi && npm run build

# Проверить подключение к Freqtrade ботам
curl http://localhost:8080/api/v1/status
curl http://localhost:8081/api/v1/status

# Проверить работу FreqUI
curl http://localhost:3000
```

---

*Этап 5 завершен: Vue.js дашборд с real-time обновлениями готов*