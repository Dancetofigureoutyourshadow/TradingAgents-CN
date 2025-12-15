<template>
  <div class="paper-trading">
    <div class="header">
      <div class="title">
        <el-icon style="margin-right:8px"><CreditCard /></el-icon>
        <span>模拟交易</span>
      </div>
      <div class="actions">
        <el-button :icon="Refresh" text size="small" @click="refreshAll">刷新</el-button>
        <el-button type="primary" :icon="Plus" @click="openOrderDialog">下市场单</el-button>
        <el-button type="danger" plain :icon="Delete" @click="confirmReset">重置账户</el-button>
      </div>
    </div>

    <!-- 风险提示横幅 -->
    <el-alert
      type="warning"
      :closable="false"
      show-icon
      style="margin-bottom: 16px;"
    >
      <template #title>
        <div style="font-weight: 600; font-size: 14px;">⚠️ 模拟交易风险提示</div>
      </template>
      <div style="font-size: 13px; line-height: 1.8;">
        <p style="margin: 0 0 8px 0;">
          <strong>1. 模拟性质：</strong>本功能为模拟交易工具，使用虚拟资金，不涉及真实资金交易，仅供学习和练习使用。
        </p>
        <p style="margin: 0 0 8px 0;">
          <strong>2. 数据延迟：</strong>模拟交易使用的行情数据可能存在延迟，与实际市场行情存在差异，成交价格和时机仅供参考。
        </p>
        <p style="margin: 0 0 8px 0;">
          <strong>3. 实盘差异：</strong>模拟交易环境与真实交易存在显著差异，包括但不限于：滑点、流动性、交易成本、心理压力等因素，模拟盈利不代表实盘能够盈利。
        </p>
        <p style="margin: 0;">
          <strong>4. 投资风险：</strong>股票投资存在市场风险，可能导致本金损失。请勿将模拟交易结果作为实盘投资决策依据，实盘交易前请充分评估自身风险承受能力并咨询专业投资顾问。
        </p>
      </div>
    </el-alert>

    <el-row :gutter="16" class="body">
      <el-col :span="8">
        <el-card shadow="hover" class="account-card">
          <template #header><div class="card-hd">账户信息</div></template>
          <div v-if="account">
            <el-tabs v-model="activeMarketTab" type="border-card">
              <!-- A股账户 -->
              <el-tab-pane label="🇨🇳 A股" name="CN">
                <el-descriptions :column="1" border>
                  <el-descriptions-item label="可用资金">¥{{ fmtAmount(account.cash?.CNY || account.cash) }}</el-descriptions-item>
                  <el-descriptions-item label="持仓市值">¥{{ fmtAmount(account.positions_value?.CNY || account.positions_value) }}</el-descriptions-item>
                  <el-descriptions-item label="总资产">¥{{ fmtAmount(account.equity?.CNY || account.equity) }}</el-descriptions-item>
                  <el-descriptions-item label="已实现盈亏">
                    <span :style="{ color: (account.realized_pnl?.CNY !== undefined ? account.realized_pnl.CNY : (typeof account.realized_pnl === 'number' ? account.realized_pnl : 0)) >= 0 ? '#F56C6C' : '#67C23A' }">
                      ¥{{ fmtAmount(account.realized_pnl?.CNY !== undefined ? account.realized_pnl.CNY : (typeof account.realized_pnl === 'number' ? account.realized_pnl : 0)) }}
                    </span>
                  </el-descriptions-item>
                </el-descriptions>
              </el-tab-pane>

              <!-- 港股账户 -->
              <el-tab-pane label="🇭🇰 港股" name="HK">
                <el-descriptions :column="1" border>
                  <el-descriptions-item label="可用资金">HK${{ fmtAmount(account.cash?.HKD || 0) }}</el-descriptions-item>
                  <el-descriptions-item label="持仓市值">HK${{ fmtAmount(account.positions_value?.HKD || 0) }}</el-descriptions-item>
                  <el-descriptions-item label="总资产">HK${{ fmtAmount(account.equity?.HKD || 0) }}</el-descriptions-item>
                  <el-descriptions-item label="已实现盈亏">
                    <span :style="{ color: (account.realized_pnl?.HKD || 0) >= 0 ? '#F56C6C' : '#67C23A' }">
                      HK${{ fmtAmount(account.realized_pnl?.HKD || 0) }}
                    </span>
                  </el-descriptions-item>
                </el-descriptions>
              </el-tab-pane>

              <!-- 美股账户 -->
              <el-tab-pane label="🇺🇸 美股" name="US">
                <el-descriptions :column="1" border>
                  <el-descriptions-item label="可用资金">${{ fmtAmount(account.cash?.USD || 0) }}</el-descriptions-item>
                  <el-descriptions-item label="持仓市值">${{ fmtAmount(account.positions_value?.USD || 0) }}</el-descriptions-item>
                  <el-descriptions-item label="总资产">${{ fmtAmount(account.equity?.USD || 0) }}</el-descriptions-item>
                  <el-descriptions-item label="已实现盈亏">
                    <span :style="{ color: (account.realized_pnl?.USD || 0) >= 0 ? '#F56C6C' : '#67C23A' }">
                      ${{ fmtAmount(account.realized_pnl?.USD || 0) }}
                    </span>
                  </el-descriptions-item>
                </el-descriptions>
              </el-tab-pane>
            </el-tabs>

            <div style="margin-top: 12px; text-align: center; color: #909399; font-size: 12px">
              更新时间: {{ formatDateTime(account.updated_at) }}
            </div>
          </div>
          <el-empty v-else description="暂无账户数据" />
        </el-card>
      </el-col>

      <el-col :span="16">
        <el-card shadow="hover" class="positions-card">
          <template #header>
            <div class="card-hd">
              持仓
              <span style="margin-left: 8px; font-size: 12px; color: #909399; font-weight: normal">
                ({{ filteredPositions.length }} 个)
              </span>
            </div>
          </template>
          <el-table :data="filteredPositions" size="small" v-loading="loading.positions">
            <el-table-column label="代码" width="100">
              <template #default="{ row }">
                <el-link type="primary" @click="viewStockDetail(row.code)">{{ row.code }}</el-link>
              </template>
            </el-table-column>
            <el-table-column label="名称" width="100">
              <template #default="{ row }">{{ row.name || '-' }}</template>
            </el-table-column>
            <el-table-column label="市场" width="70">
              <template #default="{ row }">
                <el-tag v-if="row.market === 'CN'" type="danger" size="small">🇨🇳 A股</el-tag>
                <el-tag v-else-if="row.market === 'HK'" type="warning" size="small">🇭🇰 港股</el-tag>
                <el-tag v-else-if="row.market === 'US'" type="info" size="small">🇺🇸 美股</el-tag>
                <el-tag v-else size="small">{{ row.market || 'CN' }}</el-tag>
              </template>
            </el-table-column>
            <el-table-column label="数量" width="80">
              <template #default="{ row }">
                {{ row.quantity }}
                <span v-if="row.available_qty !== undefined && row.available_qty < row.quantity" style="color: #909399; font-size: 11px">
                  (可用{{ row.available_qty }})
                </span>
              </template>
            </el-table-column>
            <el-table-column label="均价" width="100">
              <template #default="{ row }">{{ getCurrencySymbol(row.currency) }}{{ fmtPrice(row.avg_cost) }}</template>
            </el-table-column>
            <el-table-column label="最新价" width="100">
              <template #default="{ row }">{{ getCurrencySymbol(row.currency) }}{{ fmtPrice(row.last_price) }}</template>
            </el-table-column>
            <el-table-column label="浮盈" width="120">
              <template #default="{ row }">
                <span :style="{ color: (Number(row.last_price || 0) - Number(row.avg_cost || 0)) >= 0 ? '#F56C6C' : '#67C23A' }">
                  {{ getCurrencySymbol(row.currency) }}{{ fmtAmount((Number(row.last_price || 0) - Number(row.avg_cost || 0)) * Number(row.quantity || 0)) }}
                </span>
              </template>
            </el-table-column>
            <el-table-column label="操作" width="200">
              <template #default="{ row }">
                <el-button size="small" type="primary" link @click="viewStockDetail(row.code)">详情</el-button>
                <el-button size="small" type="success" link @click="goAnalysisWithCode(row.code)">分析</el-button>
                <el-button size="small" type="danger" link @click="sellPosition(row)">卖出</el-button>
              </template>
            </el-table-column>
          </el-table>
        </el-card>

        <el-card shadow="hover" class="orders-card" style="margin-top:16px">
          <template #header>
            <div class="card-hd">
              订单记录
              <span style="margin-left: 8px; font-size: 12px; color: #909399; font-weight: normal">
                ({{ filteredOrders.length }} 条)
              </span>
            </div>
          </template>
          <el-table :data="filteredOrders" size="small" v-loading="loading.orders">
            <el-table-column label="时间" width="160">
              <template #default="{ row }">{{ formatDateTime(row.created_at) }}</template>
            </el-table-column>
            <el-table-column label="方向" width="80">
              <template #default="{ row }">
                <el-tag :type="row.side === 'buy' ? 'danger' : 'success'" size="small">
                  {{ row.side === 'buy' ? '买入' : '卖出' }}
                </el-tag>
              </template>
            </el-table-column>
            <el-table-column label="代码" width="100">
              <template #default="{ row }">
                <el-link type="primary" @click="viewStockDetail(row.code)">{{ row.code }}</el-link>
              </template>
            </el-table-column>
            <el-table-column label="名称" width="100">
              <template #default="{ row }">{{ row.name || '-' }}</template>
            </el-table-column>
            <el-table-column prop="price" label="成交价" width="100">
              <template #default="{ row }">{{ fmtPrice(row.price) }}</template>
            </el-table-column>
            <el-table-column prop="quantity" label="数量" width="100" />
            <el-table-column label="状态" width="100">
              <template #default="{ row }">
                <el-tag :type="row.status === 'filled' ? 'info' : 'success'" size="small">
                  {{ row.status === 'filled' ? '已成交' : row.status }}
                </el-tag>
              </template>
            </el-table-column>
            <!-- 关联分析报告 -->
            <el-table-column label="关联分析" width="120">
              <template #default="{ row }">
                <el-button v-if="row.analysis_id" size="small" type="primary" link @click="viewReport(row.analysis_id)">
                  查看报告
                </el-button>
                <span v-else style="color: #909399;">-</span>
              </template>
            </el-table-column>
          </el-table>
        </el-card>
      </el-col>
    </el-row>

    <el-dialog v-model="orderDialog" title="下市场单" width="480px">
      <!-- 分析上下文提示 -->
      <div v-if="(order as any).analysis_id" class="analysis-context" style="margin-bottom:12px">
        <el-alert :closable="false" type="info" show-icon>
          <template #title>
            来自分析报告：<span style="font-family:monospace">{{ (order as any).analysis_id }}</span>
            <el-button link size="small" type="primary" style="margin-left:8px" @click="viewReport((order as any).analysis_id)">查看报告</el-button>
          </template>
          <div v-if="analysisLoading" style="color:#666">正在加载分析摘要…</div>
          <div v-else-if="analysisContext">
            <div style="font-size:12px;color:#666">
              <span>标的：{{ analysisContext.stock_symbol || '-' }}</span>
              <span style="margin-left:8px">模型建议：{{ analysisContext.recommendation || '-' }}</span>
            </div>
          </div>
        </el-alert>
      </div>

      <el-form label-width="90px">
        <el-form-item label="方向">
          <el-radio-group v-model="order.side">
            <el-radio-button label="buy">买入</el-radio-button>
            <el-radio-button label="sell">卖出</el-radio-button>
          </el-radio-group>
        </el-form-item>
        <el-form-item label="代码">
          <el-autocomplete
            v-model="order.code"
            :fetch-suggestions="querySearch"
            placeholder="输入股票代码或名称搜索（如：600519 或 茅台）"
            :trigger-on-focus="false"
            clearable
            @select="handleSelect"
            @change="onCodeChange"
            style="width: 100%"
          >
            <template #default="{ item }">
              <div class="stock-suggest-item">
                <span class="stock-code">{{ item.code }}</span>
                <span class="stock-name">{{ item.name }}</span>
                <el-tag v-if="item.market" size="small" style="margin-left: 8px">
                  {{ item.market}}
                </el-tag>
              </div>
            </template>
          </el-autocomplete>
        </el-form-item>
        <el-form-item label="市场" v-if="detectedMarket">
          <el-tag v-if="detectedMarket === 'CN'" type="danger">🇨🇳 A股市场 (CNY)</el-tag>
          <el-tag v-else-if="detectedMarket === 'HK'" type="warning">🇭🇰 港股市场 (HKD)</el-tag>
          <el-tag v-else-if="detectedMarket === 'US'" type="info">🇺🇸 美股市场 (USD)</el-tag>
          <div style="margin-top: 8px; font-size: 12px; color: #909399">
            <span v-if="detectedMarket === 'CN'">💡 A股T+1，今天买入明天可卖</span>
            <span v-else-if="detectedMarket === 'HK'">💡 港股T+0，买入后立即可卖</span>
            <span v-else-if="detectedMarket === 'US'">💡 美股T+0，买入后立即可卖 | 零佣金</span>
          </div>
        </el-form-item>
        <el-form-item label="数量">
          <el-input-number v-model="order.qty" :min="100" :step="100" />
        </el-form-item>
        <el-form-item label="价格">
          <el-input-number 
            v-model="order.price" 
            :min="0.01" 
            :precision="2" 
            :step="0.01"
            placeholder="只有同步过实时价格的股票才能获取到"
            style="width: 100%"
          >
            <template #append>
              <el-button 
                :icon="Refresh" 
                :loading="fetchingPrice"
                @click="fetchLatestPrice(order.code)"
                :disabled="!order.code"
              >
                刷新
              </el-button>
            </template>
          </el-input-number>
          <div v-if="fetchingPrice" style="margin-top: 4px; font-size: 12px; color: #909399">
            正在获取最新价格...
          </div>
          <div v-else-if="order.price" style="margin-top: 4px; font-size: 12px; color: #67C23A">
            💡 当前价格可手动修改
          </div>
          <div style="margin-top: 8px;">
            <el-button 
              size="small" 
              type="primary" 
              plain
              :disabled="!order.code"
              @click="showSyncPriceDialog"
            >
              🔄 同步价格
            </el-button>
            <span style="margin-left: 8px; font-size: 12px; color: #909399">
              (同步最新实时行情数据)
            </span>
          </div>
        </el-form-item>
      </el-form>
      <template #footer>
        <el-button @click="orderDialog=false">取消</el-button>
        <el-button type="primary" @click="submitOrder">提交</el-button>
      </template>
    </el-dialog>

    <!-- 同步价格对话框 -->
    <el-dialog
      v-model="syncPriceDialogVisible"
      title="同步股票数据"
      width="500px"
    >
      <el-form :model="syncPriceForm" label-width="120px">
        <el-form-item label="股票代码">
          <el-input v-model="order.code" disabled />
        </el-form-item>
        <el-form-item label="同步内容">
          <el-checkbox-group v-model="syncPriceForm.syncTypes">
            <el-checkbox label="realtime">实时行情</el-checkbox>
            <el-checkbox label="historical">历史行情数据</el-checkbox>
            <el-checkbox label="financial">财务数据</el-checkbox>
            <el-checkbox label="basic">基础数据</el-checkbox>
          </el-checkbox-group>
        </el-form-item>
        <el-form-item label="数据源">
          <el-radio-group v-model="syncPriceForm.dataSource">
            <el-radio label="tushare">Tushare</el-radio>
            <el-radio label="akshare">AKShare</el-radio>
          </el-radio-group>
        </el-form-item>
        <el-form-item label="历史数据天数" v-if="syncPriceForm.syncTypes.includes('historical')">
          <el-input-number v-model="syncPriceForm.days" :min="1" :max="3650" />
          <span style="margin-left: 10px; color: #909399; font-size: 12px;">
            (最多3650天，约10年)
          </span>
        </el-form-item>
      </el-form>

      <template #footer>
        <el-button @click="syncPriceDialogVisible = false">取消</el-button>
        <el-button type="primary" @click="handleSyncPrice" :loading="syncPriceLoading">
          开始同步
        </el-button>
      </template>
    </el-dialog>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, computed } from 'vue'
import { useRoute, useRouter } from 'vue-router'
import { ElMessage, ElMessageBox } from 'element-plus'
import { CreditCard, Refresh, Plus, Delete } from '@element-plus/icons-vue'
import { paperApi } from '@/api/paper'
import { analysisApi } from '@/api/analysis'
import { stocksApi } from '@/api/stocks'
import { stockSyncApi } from '@/api/stockSync'
import { formatDateTime } from '@/utils/datetime'

// 路由与初始化
const route = useRoute()
const router = useRouter()

// 数据
const account = ref<any | null>(null)
const positions = ref<any[]>([])
const orders = ref<any[]>([])
const loading = ref({ account: false, positions: false, orders: false })

const orderDialog = ref(false)
const order = ref({ side: 'buy', code: '', qty: 100, price: null as number | null })
const detectedMarket = ref<string>('')
const activeMarketTab = ref<string>('CN')
const fetchingPrice = ref(false)

// 同步价格对话框
const syncPriceDialogVisible = ref(false)
const syncPriceLoading = ref(false)
const syncPriceForm = ref({
  syncTypes: ['realtime'],  // 默认只选中实时行情
  dataSource: 'tushare' as 'tushare' | 'akshare',
  days: 365
})

// 计算属性：根据当前市场标签页过滤持仓
const filteredPositions = computed(() => {
  if (!positions.value || positions.value.length === 0) return []
  return positions.value.filter(pos => {
    const market = pos.market || 'CN'
    return market === activeMarketTab.value
  })
})

// 计算属性：根据当前市场标签页过滤订单
const filteredOrders = computed(() => {
  if (!orders.value || orders.value.length === 0) return []
  return orders.value.filter(ord => {
    const market = ord.market || 'CN'
    return market === activeMarketTab.value
  })
})

// 分析上下文
const analysisContext = ref<any | null>(null)
const analysisLoading = ref(false)

// 方法
function fmtPrice(n: number | null | undefined) {
  if (n == null || Number.isNaN(n as any)) return '-'
  return Number(n).toFixed(2)
}
function fmtAmount(n: number | null | undefined) {
  if (n == null || Number.isNaN(n as any)) return '-'
  return Number(n).toFixed(2)
}

// 获取货币符号
function getCurrencySymbol(currency: string | undefined) {
  if (!currency) return '¥'
  if (currency === 'CNY') return '¥'
  if (currency === 'HKD') return 'HK$'
  if (currency === 'USD') return '$'
  return ''
}

// 检测市场类型并获取最新价格
async function detectMarket() {
  const code = order.value.code.trim().toUpperCase()
  if (!code) {
    detectedMarket.value = ''
    order.value.price = null
    return
  }

  // 美股：纯字母
  if (/^[A-Z]+$/.test(code)) {
    detectedMarket.value = 'US'
  }
  // 港股：4-5位数字或.HK后缀
  else if (/^\d{4,5}$/.test(code) || code.endsWith('.HK')) {
    detectedMarket.value = 'HK'
  }
  // A股：6位数字
  else if (/^\d{6}$/.test(code)) {
    detectedMarket.value = 'CN'
  }
  // 默认A股
  else {
    detectedMarket.value = 'CN'
  }

  // 🔥 自动获取最新价格
  await fetchLatestPrice(code)
}

// 获取最新价格
async function fetchLatestPrice(code: string) {
  if (!code) return
  
  try {
    fetchingPrice.value = true
    const res = await stocksApi.getQuote(code)
    if (res.success && res.data && res.data.price !== undefined) {
      order.value.price = res.data.price
    } else {
      // 如果获取失败，清空价格
      order.value.price = null
    }
  } catch (error) {
    console.warn('获取最新价格失败:', error)
    order.value.price = null
  } finally {
    fetchingPrice.value = false
  }
}

// 🔥 股票搜索（自动完成）
const querySearch = (queryString: string, cb: (suggestions: any[]) => void) => {
  if (!queryString || queryString.trim().length === 0) {
    cb([])
    return
  }

  // 异步搜索
  stocksApi.searchStocks(queryString.trim(), 10)
    .then(res => {
      if (res.success && res.data && Array.isArray(res.data)) {
        const suggestions = res.data.map((item: any) => ({
          value: item.symbol || item.code || '',
          code: item.symbol || item.code || '',
          name: item.name || '',
          market: item.market || 'CN'
        }))
        cb(suggestions)
      } else {
        cb([])
      }
    })
    .catch(error => {
      console.warn('搜索股票失败:', error)
      cb([])
    })
}

// 选择股票后的处理
const handleSelect = async (item: any) => {
  if (item && item.code) {
    order.value.code = item.code
    // 自动检测市场并获取价格
    await detectMarket()
  }
}

// 代码输入框变化时的处理
const onCodeChange = () => {
  // 如果用户手动清空或修改代码，清空价格
  if (!order.value.code) {
    order.value.price = null
    detectedMarket.value = ''
  }
}

// 🔥 显示同步价格对话框
const showSyncPriceDialog = () => {
  if (!order.value.code) {
    ElMessage.warning('请先输入股票代码')
    return
  }
  syncPriceDialogVisible.value = true
}

// 🔥 执行同步价格
const handleSyncPrice = async () => {
  if (syncPriceForm.value.syncTypes.length === 0) {
    ElMessage.warning('请至少选择一种同步内容')
    return
  }

  syncPriceLoading.value = true
  try {
    const res = await stockSyncApi.syncSingle({
      symbol: order.value.code,
      sync_realtime: syncPriceForm.value.syncTypes.includes('realtime'),
      sync_historical: syncPriceForm.value.syncTypes.includes('historical'),
      sync_financial: syncPriceForm.value.syncTypes.includes('financial'),
      sync_basic: syncPriceForm.value.syncTypes.includes('basic'),
      data_source: syncPriceForm.value.dataSource,
      days: syncPriceForm.value.days
    })

    if (res.success) {
      const data = res.data
      let message = `股票 ${order.value.code} 数据同步完成\n`

      if (data.realtime_sync) {
        if (data.realtime_sync.success) {
          message += `✅ 实时行情同步成功\n`
        } else {
          message += `❌ 实时行情同步失败: ${data.realtime_sync.error || '未知错误'}\n`
        }
      }

      if (data.historical_sync) {
        if (data.historical_sync.success) {
          message += `✅ 历史数据: ${data.historical_sync.records || 0} 条记录\n`
        } else {
          message += `❌ 历史数据同步失败: ${data.historical_sync.error || '未知错误'}\n`
        }
      }

      if (data.financial_sync) {
        if (data.financial_sync.success) {
          message += `✅ 财务数据同步成功\n`
        } else {
          message += `❌ 财务数据同步失败: ${data.financial_sync.error || '未知错误'}\n`
        }
      }

      if (data.basic_sync) {
        if (data.basic_sync.success) {
          message += `✅ 基础数据同步成功\n`
        } else {
          message += `❌ 基础数据同步失败: ${data.basic_sync.error || '未知错误'}\n`
        }
      }

      ElMessage.success(message)
      syncPriceDialogVisible.value = false

      // 🔥 同步完成后，自动获取最新价格
      await fetchLatestPrice(order.value.code)
    } else {
      ElMessage.error(res.message || '同步失败')
    }
  } catch (error: any) {
    console.error('同步失败:', error)
    ElMessage.error(error.message || '同步失败，请稍后重试')
  } finally {
    syncPriceLoading.value = false
  }
}

async function fetchAccount() {
  try {
    loading.value.account = true
    const res = await paperApi.getAccount()
    if (res.success) {
      account.value = res.data.account
      // 可选：也可从account接口带回的positions中填充
      // positions.value = res.data.positions || positions.value
    }
  } catch (e: any) {
    ElMessage.error(e?.message || '获取账户失败')
  } finally {
    loading.value.account = false
  }
}

async function fetchPositions() {
  try {
    loading.value.positions = true
    const res = await paperApi.getPositions()
    if (res.success) {
      positions.value = res.data.items || []
      // 批量获取股票名称
      await fetchStockNames(positions.value)
    }
  } catch (e: any) {
    ElMessage.error(e?.message || '获取持仓失败')
  } finally {
    loading.value.positions = false
  }
}

async function fetchOrders() {
  try {
    loading.value.orders = true
    const res = await paperApi.getOrders(50)
    if (res.success) {
      orders.value = res.data.items || []
      // 批量获取股票名称
      await fetchStockNames(orders.value)
    }
  } catch (e: any) {
    ElMessage.error(e?.message || '获取订单失败')
  } finally {
    loading.value.orders = false
  }
}

// 批量获取股票名称
async function fetchStockNames(items: any[]) {
  if (!items || items.length === 0) return

  // 获取所有唯一的股票代码
  const codes = [...new Set(items.map(item => item.code).filter(Boolean))]

  // 并行获取所有股票的名称
  await Promise.all(
    codes.map(async (code) => {
      try {
        const res = await stocksApi.getQuote(code)
        if (res.success && res.data && res.data.name) {
          // 更新所有包含该代码的项目
          items.forEach(item => {
            if (item.code === code) {
              item.name = res.data.name
            }
          })
        }
      } catch (error) {
        console.warn(`获取股票 ${code} 名称失败:`, error)
      }
    })
  )
}

function openOrderDialog() {
  // 重置价格
  order.value.price = null
  orderDialog.value = true
}

async function submitOrder() {
  try {
    // 🔥 验证价格
    if (order.value.price === null || order.value.price <= 0) {
      ElMessage.warning('请输入有效的价格')
      return
    }

    const payload: any = { 
      side: order.value.side as 'buy' | 'sell', 
      code: order.value.code, 
      quantity: Number(order.value.qty),
      price: Number(order.value.price)  // 🔥 传递用户输入的价格
    }
    if ((order.value as any).analysis_id) payload.analysis_id = (order.value as any).analysis_id
    const res = await paperApi.placeOrder(payload)
    if (res.success) {
      ElMessage.success('下单成功')
      orderDialog.value = false
      await refreshAll()
    } else {
      ElMessage.error(res.message || '下单失败')
    }
  } catch (e: any) {
    ElMessage.error(e?.message || '下单失败')
  }
}

async function confirmReset() {
  try {
    await ElMessageBox.confirm('将清空所有订单与持仓，并重置账户为初始现金，确认重置？', '重置账户', { type: 'warning' })
    const res = await paperApi.resetAccount()
    if (res.success) {
      ElMessage.success('账户已重置')
      await refreshAll()
    }
  } catch (e) {
    // 取消或失败
  }
}

async function refreshAll() {
  await Promise.all([fetchAccount(), fetchPositions(), fetchOrders()])
}

// 查看报告详情（跳转到报告详情页）
function viewReport(analysisId: string) {
  if (!analysisId) return
  // 跳转到报告详情页
  router.push({ name: 'ReportDetail', params: { id: analysisId } })
}

// 跳转到分析页面（带股票代码和市场）
function goAnalysisWithCode(stockCode: string) {
  if (!stockCode) return
  // 根据股票代码判断市场
  const market = getMarketByCode(stockCode)
  router.push({ name: 'SingleAnalysis', query: { stock: stockCode, market } })
}

// 根据股票代码判断市场
function getMarketByCode(code: string): string {
  if (!code) return 'A股'

  // 6位数字 = A股
  if (/^\d{6}$/.test(code)) {
    return 'A股'
  }

  // 包含 .HK = 港股
  if (code.includes('.HK') || code.includes('.hk')) {
    return '港股'
  }

  // 其他 = 美股
  return '美股'
}

// 查看股票详情（跳转到股票详情页）
function viewStockDetail(stockCode: string) {
  if (!stockCode) return
  // 跳转到股票详情页
  router.push({ name: 'StockDetail', params: { code: stockCode } })
}

// 卖出持仓
async function sellPosition(position: any) {
  if (!position || !position.code) return

  try {
    // 确认卖出
    await ElMessageBox.confirm(
      `确认卖出 ${position.name || position.code}？

当前持仓：${position.quantity} 股
均价：${fmtPrice(position.avg_cost)}
最新价：${fmtPrice(position.last_price)}`,
      '卖出确认',
      {
        confirmButtonText: '确认卖出',
        cancelButtonText: '取消',
        type: 'warning'
      }
    )

    // 提交卖出订单
    const payload = {
      side: 'sell' as const,
      code: position.code,
      quantity: position.quantity
    }

    const res = await paperApi.placeOrder(payload)
    if (res.success) {
      ElMessage.success('卖出成功')
      await refreshAll()
    } else {
      ElMessage.error(res.message || '卖出失败')
    }
  } catch (error: any) {
    if (error !== 'cancel') {
      console.error('卖出失败:', error)
      ElMessage.error(error?.message || '卖出失败')
    }
  }
}

async function fetchAnalysisContext(analysisId: string) {
  try {
    analysisLoading.value = true
    analysisContext.value = null
    const res = await analysisApi.getResult(analysisId)
    analysisContext.value = res as any
  } catch (e) {
    // 忽略错误，仅用于展示
  } finally {
    analysisLoading.value = false
  }
}

onMounted(() => {
  let hasPrefill = false
  const qCode = String(route.query.code || '').trim()
  if (qCode) {
    order.value.code = qCode
    hasPrefill = true
  }
  const qSide = String(route.query.side || '').trim().toLowerCase()
  if (qSide === 'buy' || qSide === 'sell') {
    order.value.side = qSide as 'buy' | 'sell'
    hasPrefill = true
  }
  const qQty = Number(route.query.qty || route.query.quantity || 0)
  if (!Number.isNaN(qQty) && qQty > 0) {
    order.value.qty = Math.round(qQty)
    hasPrefill = true
  }
  // 可选：后续用于下单时带上分析ID
  const qAnalysisId = String(route.query.analysis_id || '').trim()
  if (qAnalysisId) {
    // 暂存于本地，等待提交订单时附带
    ;(order as any).analysis_id = qAnalysisId
    fetchAnalysisContext(qAnalysisId)
    hasPrefill = true
  }
  if (hasPrefill) {
    orderDialog.value = true
  }
  refreshAll()
})
</script>

<style scoped>
.paper-trading { padding: 16px; }
.header { display:flex; align-items:center; justify-content:space-between; margin-bottom: 12px; }
.title { display:flex; align-items:center; font-weight: 600; font-size: 16px; }
.card-hd { font-weight: 600; }

/* 股票搜索建议样式 */
.stock-suggest-item {
  display: flex;
  align-items: center;
  padding: 4px 0;
}

.stock-code {
  font-family: monospace;
  font-weight: 600;
  margin-right: 12px;
  min-width: 80px;
  color: #409EFF;
}

.stock-name {
  color: #606266;
}
</style>