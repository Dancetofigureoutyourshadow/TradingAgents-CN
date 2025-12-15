<template>
  <div class="stock-detail">
    <!-- 顶部：代码 / 名称 / 操作 -->
    <div class="header">
      <div class="title">
        <div class="code">{{ code }}</div>
        <div class="name">{{ stockName || '-' }}</div>
        <el-tag size="small">{{ market || '-' }}</el-tag>
      </div>
      <div class="actions">
        <el-button @click="onToggleFavorite">
          <el-icon><Star /></el-icon> {{ isFav ? '已自选' : '加自选' }}
        </el-button>
        <!-- 🔥 港股和美股不显示"同步数据"按钮 -->
        <el-button
          v-if="market !== 'HK' && market !== 'US'"
          type="primary"
          @click="showSyncDialog"
          :loading="syncLoading"
        >
          <el-icon><Refresh /></el-icon> 同步数据
        </el-button>
        <el-button type="warning" @click="clearCache" :loading="clearCacheLoading">
          <el-icon><Delete /></el-icon> 清除缓存
        </el-button>
        <el-button type="success" @click="openOrderDialogFromTrades">
          <el-icon><CreditCard /></el-icon> 模拟交易
        </el-button>
      </div>
    </div>

    <!-- 报价条 -->
    <el-card class="quote-card" shadow="hover">
      <div class="quote">
        <div class="price-row">
          <div class="price" :class="changeClass">{{ fmtPrice(quote.price) }}</div>
          <div class="change" :class="changeClass">
            <span>{{ fmtPercent(quote.changePercent) }}</span>
          </div>
          <el-tag type="info" size="small">{{ refreshText }}</el-tag>
          <el-button text size="small" @click="refreshMockQuote" :icon="Refresh">刷新</el-button>
        </div>
        <div class="stats">
          <div class="item"><span>今开</span><b>{{ fmtPrice(quote.open) }}</b></div>
          <div class="item"><span>最高</span><b>{{ fmtPrice(quote.high) }}</b></div>
          <div class="item"><span>最低</span><b>{{ fmtPrice(quote.low) }}</b></div>
          <div class="item"><span>昨收</span><b>{{ fmtPrice(quote.prevClose) }}</b></div>
          <div class="item">
            <span>成交量</span>
            <b>
              {{ fmtVolume(quote.volume) }}
              <el-tooltip v-if="quote.tradeDate && !isToday(quote.tradeDate)" :content="`数据日期: ${quote.tradeDate}`" placement="top">
                <el-tag size="small" type="warning" style="margin-left: 4px;">{{ formatDateTag(quote.tradeDate) }}</el-tag>
              </el-tooltip>
            </b>
          </div>
          <div class="item">
            <span>成交额</span>
            <b>
              {{ fmtAmount(quote.amount) }}
              <el-tooltip v-if="quote.tradeDate && !isToday(quote.tradeDate)" :content="`数据日期: ${quote.tradeDate}`" placement="top">
                <el-tag size="small" type="warning" style="margin-left: 4px;">{{ formatDateTag(quote.tradeDate) }}</el-tag>
              </el-tooltip>
            </b>
          </div>
          <div class="item">
            <span>换手率</span>
            <b>
              {{ fmtPercent(quote.turnover) }}
              <el-tooltip v-if="quote.turnoverDate && !isToday(quote.turnoverDate)" :content="`数据日期: ${quote.turnoverDate}`" placement="top">
                <el-tag size="small" type="warning" style="margin-left: 4px;">{{ formatDateTag(quote.turnoverDate) }}</el-tag>
              </el-tooltip>
            </b>
          </div>
          <div class="item">
            <span>振幅</span>
            <b>
              {{ Number.isFinite(quote.amplitude) ? quote.amplitude.toFixed(2) + '%' : '-' }}
              <el-tooltip v-if="quote.amplitudeDate && !isToday(quote.amplitudeDate)" :content="`数据日期: ${quote.amplitudeDate}`" placement="top">
                <el-tag size="small" type="warning" style="margin-left: 4px;">{{ formatDateTag(quote.amplitudeDate) }}</el-tag>
              </el-tooltip>
            </b>
          </div>
        </div>
        <!-- 同步状态提示 -->
        <div class="sync-status" v-if="quote.updatedAt || syncStatus">
          <el-icon><Clock /></el-icon>
          <span class="sync-info">
            <!-- 🔥 优先显示股票自己的更新时间 -->
            <template v-if="quote.updatedAt">
              数据更新: {{ formatQuoteUpdateTime(quote.updatedAt) }}
            </template>
            <template v-else-if="syncStatus">
              后端同步: {{ formatSyncTime(syncStatus.last_sync_time) }}
              <span v-if="syncStatus.interval_seconds">{{ formatSyncInterval(syncStatus.interval_seconds) }}</span>
            </template>
            <el-tag
              v-if="syncStatus?.data_source"
              size="small"
              type="success"
              style="margin-left: 4px"
            >
              {{ syncStatus.data_source }}
            </el-tag>
          </span>
        </div>
      </div>
    </el-card>

    <el-row :gutter="16" class="body">
      <el-col :span="20">
        <!-- K线蜡烛图 -->
        <el-card shadow="hover">
          <template #header>
            <div class="card-hd">
              <div>价格K线</div>
              <div class="kline-controls">
                <!-- 周期选择 -->
                <el-segmented v-model="period" :options="periodOptions" size="small" />
                
                <!-- 主图指标选择 -->
                <el-segmented 
                  v-model="mainIndicator" 
                  :options="mainIndicatorOptions" 
                  size="small" 
                  style="margin-left: 12px"
                />
                
                <!-- 均线管理 -->
                <el-dropdown trigger="click" style="margin-left: 12px" v-if="mainIndicator === 'MA' || mainIndicator === 'BOTH'">
                  <el-button size="small" :icon="TrendCharts">
                    均线 <el-icon class="el-icon--right"><arrow-down /></el-icon>
                  </el-button>
                  <template #dropdown>
                    <el-dropdown-menu>
                      <el-dropdown-item v-for="(ma, idx) in maConfigs" :key="idx">
                        <div class="ma-item">
                          <el-checkbox v-model="ma.enabled" @change="toggleMA(idx)">
                            <span :style="{ color: ma.color, fontWeight: 'bold' }">MA{{ ma.period }}</span>
                          </el-checkbox>
                          <div class="ma-actions">
                            <el-button link size="small" @click="editMA(ma)">编辑</el-button>
                            <el-button link size="small" type="danger" @click="deleteMA(idx)">删除</el-button>
                          </div>
                        </div>
                      </el-dropdown-item>
                      <el-dropdown-item divided>
                        <el-button link size="small" @click="addMA">+ 添加均线</el-button>
                      </el-dropdown-item>
                    </el-dropdown-menu>
                  </template>
                </el-dropdown>

                <!-- 技术指标选择 -->
                <el-select 
                  v-model="indicators" 
                  multiple 
                  collapse-tags
                  collapse-tags-tooltip
                  :max-collapse-tags="2"
                  placeholder="选择指标" 
                  size="small" 
                  style="width: 180px; margin-left: 12px"
                  @change="onIndicatorChange"
                >
                  <el-option
                    v-for="item in availableIndicators"
                    :key="item.value"
                    :label="item.label"
                    :value="item.value"
                    :disabled="!indicators.includes(item.value) && indicators.length >= maxIndicators"
                  />
                </el-select>
                
                <!-- 交易记录按钮-->
                <el-button 
                  size="small"
                  @click="showTradesDialog = true"
                  style="margin-left: 12px"
                >
                  <el-icon><Document /></el-icon>
                  交易记录 ({{ stockTrades.length }})
                </el-button>
              </div>
            </div>
          </template>
          <div class="kline-container">
            <!-- 加载中提示 -->
            <div v-if="isLoadingMore" class="loading-overlay">
              <el-icon class="is-loading" :size="20"><Refresh /></el-icon>
              <span>正在加载更多历史数据...</span>
            </div>
            <v-chart 
              ref="klineChart" 
              class="k-chart" 
              :option="kOption" 
              autoresize 
              @datazoom="onDataZoom"
            />
            <div class="kline-footer">
              <div class="legend">当前周期：{{ period }} · 数据源：{{ klineSource || '-' }} · 最近：{{ lastKTime || '-' }} · 收：{{ fmtPrice(lastKClose) }}</div>
              <el-button 
                v-if="hasMoreData" 
                size="small" 
                :loading="isLoadingMore" 
                @click="fetchKline(true)"
                :icon="Refresh"
              >
                加载更多历史数据
              </el-button>
              <el-tag v-else size="small" type="info">全部数据已加载</el-tag>
            </div>
          </div>
        </el-card>

        <!-- 详细分析结果（方案B）：仅在进行中或有结果时显示 -->
        <el-card v-if="analysisStatus==='running' || lastAnalysis" shadow="hover" class="analysis-detail-card" id="analysis-detail">
          <template #header><div class="card-hd">详细分析结果</div></template>
          <div v-if="analysisStatus==='running'" class="running">
            <el-progress :percentage="analysisProgress" :text-inside="true" style="width:100%" />
            <div class="hint">{{ analysisMessage || '正在生成分析报告…' }}</div>
          </div>
          <div v-else class="detail">
            <!-- 分析时间和信心度 -->
            <div class="analysis-meta">
              <span class="analysis-time">
                <el-icon><Clock /></el-icon>
                分析时间：{{ formatAnalysisTime(lastTaskInfo?.end_time) }}
              </span>
              <span class="confidence">
                <el-icon><TrendCharts /></el-icon>
                信心度：{{ fmtConf(lastAnalysis?.confidence_score ?? lastAnalysis?.overall_score) }}
              </span>
            </div>

            <!-- 投资建议 - 重点突出 -->
            <div class="recommendation-box">
              <div class="recommendation-header">
                <el-icon class="icon"><TrendCharts /></el-icon>
                <span class="title">投资建议</span>
              </div>
              <div class="recommendation-content">
                <div class="recommendation-text">
                  {{ lastAnalysis?.recommendation || '-' }}
                </div>
              </div>
            </div>

            <!-- 分析摘要 -->
            <div class="summary-section">
              <div class="summary-title">
                <el-icon><Reading /></el-icon>
                分析摘要
              </div>
              <div class="summary-text markdown-body" v-html="renderMarkdown(lastAnalysis?.summary || '-')"></div>
            </div>

            <!-- 详细报告展示 -->
            <div v-if="lastAnalysis?.reports && Object.keys(lastAnalysis.reports).length > 0" class="reports-section">
              <el-divider />
              <div class="reports-header">
                <span class="reports-title">📊 详细分析报告 ({{ Object.keys(lastAnalysis.reports).length }})</span>
                <el-button
                  type="primary"
                  plain
                  @click="showReportsDialog = true"
                  :icon="Document"
                >
                  查看完整报告
                </el-button>
              </div>

              <!-- 报告列表预览 -->
              <div class="reports-preview">
                <el-tag
                  v-for="(content, key) in lastAnalysis.reports"
                  :key="key"
                  size="small"
                  effect="plain"
                  class="report-tag"
                  @click="openReport(key)"
                >
                  {{ formatReportName(key) }}
                </el-tag>
              </div>
            </div>
          </div>
        </el-card>

        <!-- 新闻与公告：位于详细分析结果下方 -->
        <el-card shadow="hover" class="news-card">
          <template #header>
            <div class="card-hd">
              <div>近期新闻与公告</div>
              <el-select v-model="newsFilter" size="small" style="width: 160px">
                <el-option label="全部" value="all" />
                <el-option label="新闻" value="news" />
                <el-option label="公告" value="announcement" />
              </el-select>
            </div>
          </template>
          <el-empty v-if="newsItems.length === 0" description="暂无新闻" />
          <div v-else class="news-list">
            <div v-for="(n, i) in filteredNews" :key="i" class="news-item">
              <div class="row">
                <div class="left">
                  <el-tag size="small" effect="plain" :type="n.type==='announcement' ? 'warning' : 'info'" class="tag">{{ n.type==='announcement' ? '公告' : '新闻' }}</el-tag>
                  <div class="title">
                    <template v-if="n.url && n.url !== '#'">
                      <a :href="n.url" target="_blank" rel="noopener">{{ n.title || '查看详情' }}</a>
                      <el-icon class="ext"><Link /></el-icon>
                    </template>
                    <template v-else>
                      <span>{{ n.title || '（无标题）' }}</span>
                    </template>
                  </div>
                </div>
                <div class="right">{{ formatNewsTime(n.time) }}</div>
              </div>
              <div class="meta">{{ n.source || '-' }} · {{ newsSource || '-' }}</div>
            </div>
          </div>
        </el-card>




      </el-col>

      <el-col :span="4">
        <!-- 基本面快照 -->
        <el-card shadow="hover">
          <template #header><div class="card-hd">基本面快照</div></template>
          <div class="facts">
            <div class="fact"><span>行业</span><b>{{ basics.industry }}</b></div>
            <div class="fact"><span>板块</span><b>{{ basics.sector }}</b></div>
            <div class="fact"><span>总市值</span><b>{{ fmtAmount(basics.marketCap) }}</b></div>
            <div class="fact">
              <span>PE(TTM)</span>
              <b>
                {{ Number.isFinite(basics.pe) ? basics.pe.toFixed(2) : '-' }}
                <el-tag v-if="basics.peIsRealtime" type="success" size="small" style="margin-left: 4px">实时</el-tag>
              </b>
            </div>
            <div class="fact">
              <span>PB(市净率)</span>
              <b>
                {{ Number.isFinite(basics.pb) ? basics.pb.toFixed(2) : '-' }}
                <el-tag v-if="basics.peIsRealtime" type="success" size="small" style="margin-left: 4px">实时</el-tag>
              </b>
            </div>
            <div class="fact"><span>PS(TTM)</span><b>{{ Number.isFinite(basics.ps) ? basics.ps.toFixed(2) : '-' }}</b></div>
            <div class="fact"><span>ROE</span><b>{{ fmtPercent(basics.roe) }}</b></div>
            <div class="fact"><span>负债率</span><b>{{ fmtPercent(basics.debtRatio) }}</b></div>
          </div>
        </el-card>



        <!-- 快捷操作 -->
        <el-card shadow="hover" class="actions-card">
          <template #header><div class="card-hd">快捷操作</div></template>
          <div class="quick-actions">
            <el-button type="primary" @click="onAnalyze" :icon="TrendCharts" plain>发起分析</el-button>
            <el-button @click="onToggleFavorite" :icon="Star">{{ isFav ? '移出自选' : '加入自选' }}</el-button>
            <el-button type="success" :icon="CreditCard" @click="openOrderDialogFromTrades">模拟交易</el-button>
          </div>
        </el-card>
      </el-col>
    </el-row>

    <!-- 详细报告对话框 -->
    <el-dialog
      v-model="showReportsDialog"
      title="📊 详细分析报告"
      width="80%"
      :close-on-click-modal="false"
      class="reports-dialog"
    >
      <el-tabs v-model="activeReportTab" type="border-card">
        <el-tab-pane
          v-for="(content, key) in lastAnalysis?.reports"
          :key="key"
          :label="formatReportName(key)"
          :name="key"
        >
          <div class="report-content">
            <el-scrollbar height="500px">
              <div class="markdown-body" v-html="renderMarkdown(content)"></div>
            </el-scrollbar>
          </div>
        </el-tab-pane>
      </el-tabs>

      <template #footer>
        <el-button @click="showReportsDialog = false">关闭</el-button>
        <el-button type="primary" @click="exportReport">导出报告</el-button>
      </template>
    </el-dialog>

    <!-- 数据同步对话框 -->
    <el-dialog
      v-model="syncDialogVisible"
      title="同步股票数据"
      width="500px"
    >
      <el-form :model="syncForm" label-width="120px">
        <el-form-item label="股票代码">
          <el-input v-model="code" disabled />
        </el-form-item>
        <el-form-item label="股票名称">
          <el-input v-model="stockName" disabled />
        </el-form-item>
        <el-form-item label="同步内容">
          <el-checkbox-group v-model="syncForm.syncTypes">
            <el-checkbox label="realtime">实时行情</el-checkbox>
            <el-checkbox label="historical">历史行情数据</el-checkbox>
            <el-checkbox label="financial">财务数据</el-checkbox>
            <el-checkbox label="basic">基础数据</el-checkbox>
<!--            <el-checkbox label="minute">分钟级别数据</el-checkbox>-->
          </el-checkbox-group>
        </el-form-item>
        <el-form-item label="数据源">
          <el-radio-group v-model="syncForm.dataSource">
            <el-radio label="tushare">Tushare</el-radio>
            <el-radio label="akshare">AKShare</el-radio>
          </el-radio-group>
        </el-form-item>
        <el-form-item label="历史数据天数" v-if="syncForm.syncTypes.includes('historical')">
          <el-input-number v-model="syncForm.days" :min="1" :max="3650" />
          <span style="margin-left: 10px; color: #909399; font-size: 12px;">
            (最多3650天，约10年)
          </span>
        </el-form-item>
        <el-form-item label="分钟周期" v-if="syncForm.syncTypes.includes('minute')">
          <el-radio-group v-model="syncForm.minutePeriod">
            <el-radio label="5min">5分钟</el-radio>
            <el-radio label="15min">15分钟</el-radio>
            <el-radio label="30min">30分钟</el-radio>
            <el-radio label="60min">60分钟</el-radio>
          </el-radio-group>
          <div style="margin-top: 8px; color: #909399; font-size: 12px;">
            💡 分钟数据量较大，默认只获取最近7天
          </div>
        </el-form-item>
      </el-form>

      <template #footer>
        <el-button @click="syncDialogVisible = false">取消</el-button>
        <el-button type="primary" @click="handleSync" :loading="syncLoading">
          开始同步
        </el-button>
      </template>
    </el-dialog>

    <!-- 均线编辑对话框 -->
    <el-dialog
      v-model="showMADialog"
      :title="editingMA && maConfigs.find(m => m.period === editingMA.period) ? '编辑均线' : '添加均线'"
      width="400px"
    >
      <el-form v-if="editingMA" label-width="80px">
        <el-form-item label="周期">
          <el-input-number v-model="editingMA.period" :min="1" :max="250" />
        </el-form-item>
        <el-form-item label="颜色">
          <el-color-picker v-model="editingMA.color" />
        </el-form-item>
      </el-form>
      <template #footer>
        <el-button @click="showMADialog = false">取消</el-button>
        <el-button type="primary" @click="saveMA">保存</el-button>
      </template>
    </el-dialog>

    <!-- 交易记录对话框 -->
    <el-dialog
      v-model="showTradesDialog"
      title="📊 模拟交易记录"
      width="800px"
    >
      <div class="trades-dialog">
        <div class="trades-summary">
          <el-descriptions :column="3" border>
            <el-descriptions-item label="股票代码">
              <el-tag>{{ code }}</el-tag>
            </el-descriptions-item>
            <el-descriptions-item label="交易次数">
              <el-tag type="info">{{ stockTrades.length }} 笔</el-tag>
            </el-descriptions-item>
            <el-descriptions-item label="持仓成本">
              <el-tag v-if="avgPrice" type="warning">￥{{ avgPrice.toFixed(2) }}</el-tag>
              <span v-else>-</span>
            </el-descriptions-item>
            <el-descriptions-item label="当前持仓">
              <el-tag type="info">{{ currentQuantity }} 股</el-tag>
            </el-descriptions-item>
            <el-descriptions-item label="总持仓盈亏">
              <el-tag v-if="avgPrice && quote.price && currentQuantity > 0" :type="getProfitTagType()">
                {{ formatProfit() }}
              </el-tag>
              <span v-else>-</span>
            </el-descriptions-item>
          </el-descriptions>
          <div v-if="avgPrice" class="trades-note">
            <el-icon><InfoFilled /></el-icon>
            持仓成本为按数量加权计算的平均成本，已考虑所有买入和卖出交易
          </div>
        </div>

        <el-divider />

        <el-table :data="stockTrades" stripe border style="width: 100%">
          <el-table-column label="时间" width="180">
            <template #default="{ row }">
              {{ formatTradeTime(row.timestamp) }}
            </template>
          </el-table-column>
          <el-table-column label="类型" width="80" align="center">
            <template #default="{ row }">
              <el-tag :type="row.side === 'buy' ? 'danger' : 'success'" size="small">
                {{ row.side === 'buy' ? '买入' : '卖出' }}
              </el-tag>
            </template>
          </el-table-column>
          <el-table-column label="数量" prop="quantity" width="120" align="right" />
          <el-table-column label="价格" width="140" align="right">
            <template #default="{ row }">
              ￥{{ row.price.toFixed(2) }}
            </template>
          </el-table-column>
          <el-table-column label="成交金额" align="right">
            <template #default="{ row }">
              ￥{{ row.amount.toFixed(2) }}
            </template>
          </el-table-column>
        </el-table>
      </div>

      <template #footer>
        <el-button @click="showTradesDialog = false">关闭</el-button>
      </template>
    </el-dialog>
  </div>
</template>

<script setup lang="ts">
import { ref, reactive, computed, onMounted, onUnmounted, watch } from 'vue'
import { useRoute, useRouter } from 'vue-router'
import { ElMessage, ElMessageBox } from 'element-plus'
import { TrendCharts, Star, Refresh, Link, Document, Clock, Reading, CreditCard, Delete, ArrowDown, InfoFilled } from '@element-plus/icons-vue'
import { marked } from 'marked'
import { stocksApi } from '@/api/stocks'
import { analysisApi } from '@/api/analysis'
import { ApiClient } from '@/api/request'
import { stockSyncApi } from '@/api/stockSync'
import { clearAllCache } from '@/api/cache'
import { paperApi } from '@/api/paper'
import { use as echartsUse } from 'echarts/core'
import { CandlestickChart, LineChart, BarChart } from 'echarts/charts'

import { GridComponent, TooltipComponent, DataZoomComponent, LegendComponent, TitleComponent } from 'echarts/components'
import { CanvasRenderer } from 'echarts/renderers'
import VChart from 'vue-echarts'
import type { EChartsOption } from 'echarts'
import { favoritesApi } from '@/api/favorites'
import { useNotificationStore } from '@/stores/notifications'


echartsUse([CandlestickChart, LineChart, BarChart, GridComponent, TooltipComponent, DataZoomComponent, LegendComponent, TitleComponent, CanvasRenderer])

const route = useRoute()
const router = useRouter()


// 分析状态
const analysisStatus = ref<'idle' | 'running' | 'completed' | 'failed'>('idle')
const analysisProgress = ref(0)
const analysisMessage = ref('')
const currentTaskId = ref<string | null>(null)
const lastAnalysis = ref<any | null>(null)
const lastTaskInfo = ref<any | null>(null) // 保存任务信息（包含 end_time 等）

// 报告对话框
const showReportsDialog = ref(false)
const activeReportTab = ref('')

const notifStore = useNotificationStore()

const lastAnalysisTagType = computed(() => {
  const reco = String(lastAnalysis.value?.recommendation || '').toLowerCase()
  if (reco.includes('买') || reco.includes('buy') || reco.includes('增持') || reco.includes('强')) return 'success'
  if (reco.includes('卖') || reco.includes('sell')) return 'danger'
  if (reco.includes('减持') || reco.includes('谨慎')) return 'warning'
  return 'info'
})

// 股票代码（从路由参数获取）
const code = computed(() => {
  const routeCode = String(route.params.code || '').toUpperCase()
  if (!routeCode) {
    ElMessage.error('股票代码不能为空')
    router.push({ name: 'Dashboard' })
    return ''
  }
  return routeCode
})
const symbol = computed(() => code.value.split('.')[0])  // 提取6位代码
const stockName = ref('')
const market = ref('')
const isFav = ref(false)

// ECharts K线配置
const kOption = ref<EChartsOption>({
  grid: { left: 40, right: 20, top: 20, bottom: 40 },
  tooltip: {
    trigger: 'axis',
    axisPointer: { type: 'cross' }
  },
  xAxis: {
    type: 'category',
    data: [],
    boundaryGap: true,
    axisLine: { onZero: false }
  },
  yAxis: {
    scale: true,
    type: 'value'
  },
  dataZoom: [
    { type: 'inside', start: 70, end: 100 },
    { start: 70, end: 100 }
  ],
  series: [
    {
      type: 'candlestick',
      name: 'K线',
      data: [],
      itemStyle: {
        color: '#ef4444',
        color0: '#16a34a',
        borderColor: '#ef4444',
        borderColor0: '#16a34a'
      }
    }
  ]
})
const lastKTime = ref<string | null>(null)
const lastKClose = ref<number | null>(null)

// 报价（初始化）
const quote = reactive({
  price: NaN,
  changePercent: NaN,
  open: NaN,
  high: NaN,
  low: NaN,
  prevClose: NaN,
  volume: NaN,
  amount: NaN,
  turnover: NaN,
  amplitude: NaN,  // 振幅（替代量比）
  tradeDate: null as string | null,  // 交易日期（用于成交量、成交额）
  turnoverDate: null as string | null,  // 换手率数据日期
  amplitudeDate: null as string | null,  // 振幅数据日期
  updatedAt: null as string | null  // 🔥 数据更新时间
})

const lastRefreshAt = ref<Date | null>(null)
const refreshText = computed(() => lastRefreshAt.value ? `已刷新 ${lastRefreshAt.value.toLocaleTimeString()}` : '未刷新')
const changeClass = computed(() => quote.changePercent > 0 ? 'up' : quote.changePercent < 0 ? 'down' : '')

// 🔥 日期判断和格式化函数
function isToday(dateStr: string | null): boolean {
  if (!dateStr) return false
  const today = new Date().toISOString().split('T')[0].replace(/-/g, '')
  const targetDate = dateStr.replace(/-/g, '')
  return today === targetDate
}

function formatDateTag(dateStr: string | null): string {
  if (!dateStr) return ''
  // 将 YYYYMMDD 或 YYYY-MM-DD 格式转换为 MM-DD
  const cleaned = dateStr.replace(/-/g, '')
  if (cleaned.length === 8) {
    return `${cleaned.substring(4, 6)}-${cleaned.substring(6, 8)}`
  }
  return dateStr
}

// 同步状态
const syncStatus = ref<any>(null)

// 数据同步对话框
const syncDialogVisible = ref(false)
const syncLoading = ref(false)
const syncForm = reactive({
  syncTypes: ['realtime'],  // 默认选中实时行情
  dataSource: 'tushare' as 'tushare' | 'akshare',
  days: 365,
  minutePeriod: '5min' as '5min' | '15min' | '30min' | '60min'
})

// 清除缓存
const clearCacheLoading = ref(false)

// 显示同步对话框
function showSyncDialog() {
  syncDialogVisible.value = true
}

// 执行同步
async function handleSync() {
  if (syncForm.syncTypes.length === 0) {
    ElMessage.warning('请至少选择一种同步内容')
    return
  }

  syncLoading.value = true
  try {
    const res = await stockSyncApi.syncSingle({
      symbol: code.value,
      sync_realtime: syncForm.syncTypes.includes('realtime'),
      sync_historical: syncForm.syncTypes.includes('historical'),
      sync_financial: syncForm.syncTypes.includes('financial'),
      sync_basic: syncForm.syncTypes.includes('basic'),
      sync_minute: syncForm.syncTypes.includes('minute'),
      minute_period: syncForm.minutePeriod,
      data_source: syncForm.dataSource,
      days: syncForm.days
    })

    if (res.success) {
      const data = res.data
      let message = `股票 ${code.value} 数据同步完成\n`

      if (data.realtime_sync) {
        if (data.realtime_sync.success) {
          // 🔥 如果切换了数据源，显示提示信息
          if (data.realtime_sync.data_source_used && data.realtime_sync.data_source_used !== syncForm.dataSource) {
            message += `✅ 实时行情同步成功（已自动切换到 ${data.realtime_sync.data_source_used.toUpperCase()} 数据源）\n`
          } else {
            message += `✅ 实时行情同步成功\n`
          }
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

      if (data.minute_sync) {
        if (data.minute_sync.success) {
          message += `✅ 分钟数据: ${data.minute_sync.records || 0} 条记录，保存 ${data.minute_sync.saved || 0} 条\n`
        } else {
          message += `❌ 分钟数据同步失败: ${data.minute_sync.error || '未知错误'}\n`
        }
      }

      ElMessage.success(message)
      syncDialogVisible.value = false

      // 刷新页面数据
      await fetchQuote()
      await fetchFundamentals()
    } else {
      ElMessage.error(res.message || '同步失败')
    }
  } catch (error: any) {
    console.error('同步失败:', error)
    ElMessage.error(error.message || '同步失败，请稍后重试')
  } finally {
    syncLoading.value = false
  }
}

async function refreshMockQuote() {
  // 改为调用后端接口获取真实数据
  await fetchQuote()
}

// 清除缓存
async function clearCache() {
  try {
    await ElMessageBox.confirm(
      '确定要清除所有缓存吗？清除后需要重新从数据源获取数据。',
      '清除缓存',
      {
        confirmButtonText: '确定',
        cancelButtonText: '取消',
        type: 'warning'
      }
    )

    clearCacheLoading.value = true
    await clearAllCache()
    ElMessage.success('缓存已清除，正在刷新数据...')

    // 刷新当前页面数据
    await Promise.all([
      fetchQuote(),
      fetchFundamentals(),
      fetchKline(),
      fetchNews()
    ])

    ElMessage.success('数据已刷新')
  } catch (error: any) {
    if (error !== 'cancel') {
      console.error('清除缓存失败:', error)
      ElMessage.error(error.message || '清除缓存失败')
    }
  } finally {
    clearCacheLoading.value = false
  }
}

async function fetchQuote() {
  // 🔥 参数验证：确保股票代码不为空
  if (!code.value) {
    console.warn('股票代码为空，跳过获取报价')
    return
  }

  try {
    const res = await stocksApi.getQuote(code.value)
    const d: any = (res as any)?.data || {}
    // 后端为 snake_case，前端状态为 camelCase，这里进行映射
    quote.price = Number(d.price ?? d.close ?? quote.price)
    quote.changePercent = Number(d.change_percent ?? quote.changePercent)
    quote.open = Number(d.open ?? quote.open)
    quote.high = Number(d.high ?? quote.high)
    quote.low = Number(d.low ?? quote.low)
    quote.prevClose = Number(d.prev_close ?? quote.prevClose)
    quote.volume = Number.isFinite(d.volume) ? Number(d.volume) : quote.volume
    quote.amount = Number.isFinite(d.amount) ? Number(d.amount) : quote.amount
    quote.turnover = Number.isFinite(d.turnover_rate) ? Number(d.turnover_rate) : quote.turnover
    quote.amplitude = Number.isFinite(d.amplitude) ? Number(d.amplitude) : quote.amplitude

    // 🔥 获取数据日期（用于标注非当天数据）
    quote.tradeDate = d.trade_date || null  // 交易日期（用于成交量、成交额）
    quote.turnoverDate = d.turnover_rate_date || d.trade_date || null
    quote.amplitudeDate = d.amplitude_date || d.trade_date || null
    quote.updatedAt = d.updated_at || null  // 🔥 数据更新时间

    if (d.name) stockName.value = d.name
    if (d.market) market.value = d.market
    lastRefreshAt.value = new Date()
  } catch (e) {
    console.error('获取报价失败', e)
  }
}

async function fetchFundamentals() {
  try {
    const res = await stocksApi.getFundamentals(code.value)
    const f: any = (res as any)?.data || {}
    // 基本面快照映射（以后台为准）
    if (f.name) stockName.value = f.name
    if (f.market) market.value = f.market
    basics.industry = f.industry || basics.industry
    basics.sector = f.sector || basics.sector || '—'
    // 后端 total_mv 单位：亿元，这里转为元以便与金额格式化函数配合
    basics.marketCap = Number.isFinite(f.total_mv) ? Number(f.total_mv) * 1e8 : basics.marketCap
    // 优先使用 pe_ttm，其次 pe
    basics.pe = Number.isFinite(f.pe_ttm) ? Number(f.pe_ttm) : (Number.isFinite(f.pe) ? Number(f.pe) : basics.pe)
    // 🔥 新增：PB（市净率）
    basics.pb = Number.isFinite(f.pb) ? Number(f.pb) : basics.pb
    // 🔥 新增：PS（市销率）- 优先使用 ps_ttm，其次 ps
    basics.ps = Number.isFinite(f.ps_ttm) ? Number(f.ps_ttm) : (Number.isFinite(f.ps) ? Number(f.ps) : basics.ps)
    // ROE 和负债率
    basics.roe = Number.isFinite(f.roe) ? Number(f.roe) : basics.roe
    const ff: any = f
    basics.debtRatio = Number.isFinite(ff.debt_ratio) ? Number(ff.debt_ratio) : basics.debtRatio

    // 获取PE/PB的实时标识
    basics.peIsRealtime = ff.pe_is_realtime || false
    basics.peSource = ff.pe_source || ''
    basics.peUpdatedAt = ff.pe_updated_at || null
  } catch (e) {
    console.error('获取基本面失败', e)
  }
}

async function fetchSyncStatus() {
  try {
    const res = await ApiClient.get('/api/stock-data/sync-status/quotes')
    const d: any = (res as any)?.data || {}
    syncStatus.value = d
  } catch (e) {
    console.warn('获取同步状态失败', e)
  }
}

// 获取模拟交易数据
async function fetchStockTrades() {
  try {
    const res = await paperApi.getStockTrades(code.value)
    if (res.success && res.data) {
      stockTrades.value = res.data.trades || []
      avgPrice.value = res.data.avg_price
      currentQuantity.value = res.data.current_quantity || 0  // 当前持有数量
      console.log(`📊 获取到 ${code.value} 的交易记录: ${stockTrades.value.length} 条, 均价: ${avgPrice.value}, 持仓: ${currentQuantity.value}`)
      // 重新渲染图表以显示均价线
      updateKlineChart()
    }
  } catch (e) {
    console.warn('获取模拟交易数据失败', e)
    stockTrades.value = []
    avgPrice.value = null
    currentQuantity.value = 0
  }
}

let timer: any = null
async function checkFavorite() {
  try {
    const res: any = await favoritesApi.check(code.value)
    const d: any = (res as any)?.data || {}
    isFav.value = !!d.is_favorite
  } catch (e) {
    console.warn('检查自选失败', e)
  }
}
onMounted(async () => {
  // 首次加载：打通后端（并行）
  await Promise.all([
    fetchQuote(),
    fetchFundamentals(),
    fetchKline(),
    fetchNews(),
    checkFavorite(),
    fetchLatestAnalysis(),  // 获取最新的历史分析报告
    fetchSyncStatus(),  // 获取同步状态
    fetchStockTrades()  // 获取模拟交易数据
  ])
  // 每30秒刷新一次报价
  timer = setInterval(fetchQuote, 30000)
})
onUnmounted(() => { if (timer) clearInterval(timer) })



// K线占位相关
const periodOptions = ['日K','周K','月K']
const period = ref('日K')

const klineSource = ref<string | undefined>(undefined)

// 均线配置
interface MAConfig {
  period: number
  color: string
  enabled: boolean
}

const maConfigs = ref<MAConfig[]>([
  { period: 5, color: '#FF6B6B', enabled: true },
  { period: 10, color: '#4ECDC4', enabled: true },
  { period: 20, color: '#FFD93D', enabled: true },
  { period: 60, color: '#95E1D3', enabled: true }
])

const showMADialog = ref(false)
const editingMA = ref<MAConfig | null>(null)

// 主图指标配置（均线/BOLL等）
const mainIndicator = ref<'MA' | 'BOLL' | 'BOTH'>('MA') // 默认显示均线
const mainIndicatorOptions = [
  { value: 'MA', label: '仅均线' },
  { value: 'BOLL', label: '仅BOLL' },
  { value: 'BOTH', label: '均线+BOLL' }
]

// 技术指标配置
const indicators = ref<string[]>(['MACD', 'VOL']) // 默认显示MACD和成交量
const availableIndicators = [
  { value: 'MACD', label: 'MACD' },
  { value: 'RSI', label: 'RSI' },
  { value: 'KDJ', label: 'KDJ' },
  { value: 'VOL', label: '成交量' }
]
const maxIndicators = 2 // 最多显示2个副图

// K线原始数据
const klineData = ref<any[]>([])
const isLoadingMore = ref(false) // 是否正在加载更多数据
const hasMoreData = ref(true) // 是否还有更多数据
const currentLimit = ref(200) // 当前已加载的数据量
const klineChart = ref<any>(null) // ECharts 实例引用
const lastDataZoomStart = ref(0) // 上次 dataZoom 的 start 值
const savedDataZoomState = ref<{ start: number; end: number } | null>(null) // 保存的 dataZoom 状态

// 模拟交易数据
const stockTrades = ref<any[]>([])
const avgPrice = ref<number | null>(null)
const currentQuantity = ref<number>(0)  // 当前持有数量
const showAvgPrice = ref(true) // 是否显示均价线
const showTradesDialog = ref(false) // 是否显示交易记录对话框

function periodLabelToParam(p: string): string {
  if (p.includes('5')) return '5m'
  if (p.includes('15')) return '15m'
  if (p.includes('60')) return '60m'
  if (p.includes('日')) return 'day'
  if (p.includes('周')) return 'week'
  if (p.includes('月')) return 'month'
  return '5m'
}

// 当周期切换时刷新K线
watch(period, () => { 
  // 重置加载状态
  currentLimit.value = 200
  hasMoreData.value = true
  klineData.value = []
  fetchKline() 
})

// 当均线配置或指标配置改变时，重新渲染图表
watch([maConfigs, indicators, mainIndicator], () => { updateKlineChart() }, { deep: true })

// 处理指标选择变更
function onIndicatorChange(value: string[]) {
  // 限制最多只能选择 maxIndicators 个指标
  if (value.length > maxIndicators) {
    ElMessage.warning(`最多只能同时显示 ${maxIndicators} 个副图指标`)
    // 保留最后选择的 maxIndicators 个
    indicators.value = value.slice(-maxIndicators)
  }
}

// 监听 dataZoom 事件，当拖动到最左侧时自动加载更多数据
function onDataZoom(params: any) {
  // params.batch 包含所有 dataZoom 组件的状态
  if (!params || !params.batch || params.batch.length === 0) return
  
  const dataZoom = params.batch[0]
  const currentStart = dataZoom.start || 0
  const currentEnd = dataZoom.end || 100
  
  // 保存当前的 dataZoom 状态
  savedDataZoomState.value = { start: currentStart, end: currentEnd }
  
  // 当拖动到最左侧（start <= 5）且还有更多数据时，自动加载
  if (currentStart <= 5 && hasMoreData.value && !isLoadingMore.value) {
    // 防止频繁触发：只有当 start 从大于 5 变为小于等于 5 时才触发
    if (lastDataZoomStart.value > 5) {
      console.log('🔄 检测到滚动到最左侧，自动加载更多数据...')
      fetchKline(true)
    }
  }
  
  lastDataZoomStart.value = currentStart
}

// 计算均线数据
function calculateMA(data: number[], period: number): (number | null)[] {
  const result: (number | null)[] = []
  for (let i = 0; i < data.length; i++) {
    if (i < period - 1) {
      result.push(null)
    } else {
      let sum = 0
      for (let j = 0; j < period; j++) {
        sum += data[i - j]
      }
      result.push(+(sum / period).toFixed(2))
    }
  }
  return result
}

// 计算MACD
function calculateMACD(data: number[]) {
  const ema12: number[] = []
  const ema26: number[] = []
  const dif: number[] = []
  const dea: number[] = []
  const macd: number[] = []

  let ema12Val = data[0]
  let ema26Val = data[0]
  let deaVal = 0

  for (let i = 0; i < data.length; i++) {
    ema12Val = (data[i] * 2 + ema12Val * 11) / 13
    ema26Val = (data[i] * 2 + ema26Val * 25) / 27
    ema12.push(ema12Val)
    ema26.push(ema26Val)

    const difVal = ema12Val - ema26Val
    dif.push(difVal)

    deaVal = (difVal * 2 + deaVal * 8) / 10
    dea.push(deaVal)

    macd.push((difVal - deaVal) * 2)
  }

  return { dif, dea, macd }
}

// 计算RSI
function calculateRSI(data: number[], period: number = 14) {
  const rsi: (number | null)[] = []
  let avgGain = 0
  let avgLoss = 0

  for (let i = 0; i < data.length; i++) {
    if (i === 0) {
      rsi.push(null)
      continue
    }

    const change = data[i] - data[i - 1]
    const gain = change > 0 ? change : 0
    const loss = change < 0 ? -change : 0

    if (i < period) {
      avgGain += gain
      avgLoss += loss
      if (i === period - 1) {
        avgGain /= period
        avgLoss /= period
      }
      rsi.push(null)
    } else {
      avgGain = (avgGain * (period - 1) + gain) / period
      avgLoss = (avgLoss * (period - 1) + loss) / period
      const rs = avgLoss === 0 ? 100 : avgGain / avgLoss
      rsi.push(+(100 - (100 / (1 + rs))).toFixed(2))
    }
  }

  return rsi
}

// 计算KDJ
function calculateKDJ(high: number[], low: number[], close: number[], period: number = 9) {
  const rsv: number[] = []
  const k: number[] = []
  const d: number[] = []
  const j: number[] = []

  let kVal = 50
  let dVal = 50

  for (let i = 0; i < close.length; i++) {
    if (i < period - 1) {
      rsv.push(50)
      k.push(50)
      d.push(50)
      j.push(50)
    } else {
      const periodHigh = Math.max(...high.slice(i - period + 1, i + 1))
      const periodLow = Math.min(...low.slice(i - period + 1, i + 1))
      const rsvVal = periodHigh === periodLow ? 50 : ((close[i] - periodLow) / (periodHigh - periodLow)) * 100
      rsv.push(rsvVal)

      kVal = (kVal * 2 + rsvVal) / 3
      dVal = (dVal * 2 + kVal) / 3
      const jVal = 3 * kVal - 2 * dVal

      k.push(+kVal.toFixed(2))
      d.push(+dVal.toFixed(2))
      j.push(+jVal.toFixed(2))
    }
  }

  return { k, d, j }
}

// 计算BOLL（布林带）
function calculateBOLL(data: number[], period: number = 20, multiplier: number = 2) {
  const middle: (number | null)[] = []  // 中轨（MA）
  const upper: (number | null)[] = []   // 上轨
  const lower: (number | null)[] = []   // 下轨
  
  for (let i = 0; i < data.length; i++) {
    if (i < period - 1) {
      middle.push(null)
      upper.push(null)
      lower.push(null)
    } else {
      // 计算中轨（简单移动平均）
      let sum = 0
      for (let j = 0; j < period; j++) {
        sum += data[i - j]
      }
      const ma = sum / period
      
      // 计算标准差
      let variance = 0
      for (let j = 0; j < period; j++) {
        variance += Math.pow(data[i - j] - ma, 2)
      }
      const stdDev = Math.sqrt(variance / period)
      
      middle.push(+ma.toFixed(2))
      upper.push(+(ma + multiplier * stdDev).toFixed(2))
      lower.push(+(ma - multiplier * stdDev).toFixed(2))
    }
  }
  
  return { middle, upper, lower }
}

// 更新K线图表
function updateKlineChart() {
  if (klineData.value.length === 0) return

  const category = klineData.value.map((item: any) => 
    String(item.time || item.trade_time || item.trade_date || '')
  )
  const values = klineData.value.map((item: any) => [
    Number(item.open ?? NaN),
    Number(item.close ?? NaN),
    Number(item.low ?? NaN),
    Number(item.high ?? NaN)
  ])
  const closeData = klineData.value.map((item: any) => Number(item.close ?? NaN))
  
  // 🔥 处理成交量数据：如果最新数据日期与当前日期一致，去除最后三位
  const today = new Date().toISOString().split('T')[0].replace(/-/g, '')
  const volumeData = klineData.value.map((item: any, index: any) => {
    const volume = Number(item.volume ?? 0)
    // 检查是否是最新数据且日期与今天一致
    if (index === klineData.value.length - 1) {
      const itemDate = String(item.time || item.trade_time || item.trade_date || '').replace(/-/g, '').substring(0, 8)
      if (itemDate === today) {
        // 去除最后三位（除以1000）
        return Math.floor(volume / 100)
      }
    }
    return volume
  })
  
  const highData = klineData.value.map((item: any) => Number(item.high ?? NaN))
  const lowData = klineData.value.map((item: any) => Number(item.low ?? NaN))

  // 基础系列：K线
  const series: any[] = [
    {
      type: 'candlestick',
      name: 'K线',
      data: values,
      itemStyle: {
        color: '#ef4444',
        color0: '#16a34a',
        borderColor: '#ef4444',
        borderColor0: '#16a34a'
      }
    }
  ]

  // 添加主图指标（均线或BOLL）
  const legend: string[] = ['K线']
  
  // 根据选择添加均线
  if (mainIndicator.value === 'MA' || mainIndicator.value === 'BOTH') {
    const enabledMAs = maConfigs.value.filter(ma => ma.enabled)
    enabledMAs.forEach(ma => {
      const maData = calculateMA(closeData, ma.period)
      series.push({
        type: 'line',
        name: `MA${ma.period}`,
        data: maData,
        smooth: true,
        showSymbol: false,
        lineStyle: {
          width: 1.5,
          color: ma.color
        },
        z: 1  // 确保在K线上方
      })
      legend.push(`MA${ma.period}`)
    })
  }
  
  // 根据选择添加BOLL线
  if (mainIndicator.value === 'BOLL' || mainIndicator.value === 'BOTH') {
    const bollData = calculateBOLL(closeData, 20, 2)
    series.push(
      {
        type: 'line',
        name: 'BOLL-上轨',
        data: bollData.upper,
        showSymbol: false,
        lineStyle: { width: 1, color: '#FF6B6B', type: 'dashed' },
        z: 1
      },
      {
        type: 'line',
        name: 'BOLL-中轨',
        data: bollData.middle,
        showSymbol: false,
        lineStyle: { width: 1.5, color: '#FFD700' },
        z: 1
      },
      {
        type: 'line',
        name: 'BOLL-下轨',
        data: bollData.lower,
        showSymbol: false,
        lineStyle: { width: 1, color: '#16a34a', type: 'dashed' },
        z: 1,
        areaStyle: {
          color: {
            type: 'linear',
            x: 0,
            y: 0,
            x2: 0,
            y2: 1,
            colorStops: [
              { offset: 0, color: 'rgba(255, 107, 107, 0.05)' },
              { offset: 1, color: 'rgba(22, 163, 74, 0.05)' }
            ]
          },
          origin: 'start'
        }
      }
    )
    legend.push('BOLL-上轨', 'BOLL-中轨', 'BOLL-下轨')
  }
  
  // 添加模拟交易均价线
  if (showAvgPrice.value && avgPrice.value !== null) {
    const avgPriceData = new Array(category.length).fill(avgPrice.value)
    series.push({
      type: 'line',
      name: '成本价',
      data: avgPriceData,
      showSymbol: false,
      lineStyle: { 
        width: 2, 
        color: '#FF8C00',  // 橙色
        type: 'dashed'  // 虚线
      },
      z: 2,  // 确保在最上层
      markLine: {
        symbol: 'none',
        label: {
          show: true,
          position: 'start',  // 在最左侧显示
          formatter: `成本: ${avgPrice.value.toFixed(2)}`,
          color: '#FF8C00',
          fontSize: 12,
          fontWeight: 'bold'
        },
        lineStyle: {
          color: '#FF8C00',
          width: 2,
          type: 'dashed'
        },
        data: [{
          yAxis: avgPrice.value
        }]
      }
    })
    legend.push('成本价')
  }

  // 计算网格布局
  // 根据副图数量动态调整主图高度和副图位置
  const subplotCount = Math.min(indicators.value.length, maxIndicators)
  const mainChartHeight = subplotCount === 0 ? 360 : (subplotCount === 1 ? 280 : 240)
  const subplotHeight = 80
  const gridGap = 20 // 图表间距
  
  const grids: any[] = [{ left: 60, right: 60, top: 60, height: mainChartHeight }]
  const xAxes: any[] = [
    {
      type: 'category',
      data: category,
      gridIndex: 0,
      boundaryGap: true,
      axisLine: { onZero: false },
      splitLine: { show: false },
      axisLabel: { show: false }
    }
  ]
  const yAxes: any[] = [
    {
      scale: true,
      gridIndex: 0,
      splitLine: { show: true },
      axisLabel: { inside: false }
    }
  ]
  
  // 计算 dataZoom 的位置，确保不与副图重叠
  const totalChartHeight = mainChartHeight + 60 + (subplotCount * (subplotHeight + gridGap))
  const dataZoomTop = totalChartHeight + 10 // 留出10px间距
  
  const dataZooms: any[] = [
    { type: 'inside', xAxisIndex: [0], start: 70, end: 100 },
    { 
      show: true, 
      xAxisIndex: [0], 
      type: 'slider', 
      bottom: 10, // 使用 bottom 而不是 top，确保始终在底部
      height: 20,
      start: 70, 
      end: 100 
    }
  ]

  let currentTop = mainChartHeight + 60 + gridGap
  let gridIndex = 1

  // 添加指标副图
  indicators.value.forEach(indicator => {
    if (indicator === 'VOL') {
      grids.push({ left: 60, right: 60, top: currentTop, height: subplotHeight })
      xAxes.push({
        type: 'category',
        data: category,
        gridIndex,
        boundaryGap: true,
        axisLine: { onZero: false },
        splitLine: { show: false },
        axisLabel: { show: false }
      })
      yAxes.push({
        scale: false,  // 不使用scale，使用固定范围
        gridIndex,
        splitLine: { show: true },
        min: 0,  // 从0开始
        max: (value: any) => {
          // 动态计算最大值，留出20%空间
          return value.max * 1.2
        },
        axisLabel: { 
          inside: false,
          formatter: (value: number) => {
            // 成交量格式化：亿/万
            if (value >= 100000000) return (value / 100000000).toFixed(1) + '亿'
            if (value >= 10000) return (value / 10000).toFixed(1) + '万'
            return value.toFixed(0)
          }
        }
      })
      series.push({
        type: 'bar',
        name: '成交量',
        data: volumeData,
        xAxisIndex: gridIndex,
        yAxisIndex: gridIndex,
        itemStyle: {
          color: (params: any) => {
            const idx = params.dataIndex
            if (idx === 0) return '#ef4444'
            return values[idx][1] >= values[idx][0] ? '#ef4444' : '#16a34a'
          }
        }
      })
      legend.push('成交量')
      dataZooms[0].xAxisIndex.push(gridIndex)
      dataZooms[1].xAxisIndex.push(gridIndex)
      currentTop += subplotHeight + gridGap
      gridIndex++
    } else if (indicator === 'MACD') {
      const macdData = calculateMACD(closeData)
      grids.push({ left: 60, right: 60, top: currentTop, height: subplotHeight })
      xAxes.push({
        type: 'category',
        data: category,
        gridIndex,
        boundaryGap: true,
        axisLine: { onZero: false },
        splitLine: { show: false },
        axisLabel: { show: false }
      })
      yAxes.push({
        scale: true,
        gridIndex,
        splitLine: { show: true },
        axisLabel: { inside: false }
      })
      series.push(
        {
          type: 'line',
          name: 'DIF',
          data: macdData.dif,
          xAxisIndex: gridIndex,
          yAxisIndex: gridIndex,
          showSymbol: false,
          lineStyle: { width: 1.5, color: '#FFD700' }
        },
        {
          type: 'line',
          name: 'DEA',
          data: macdData.dea,
          xAxisIndex: gridIndex,
          yAxisIndex: gridIndex,
          showSymbol: false,
          lineStyle: { width: 1.5, color: '#00CED1' }
        },
        {
          type: 'bar',
          name: 'MACD',
          data: macdData.macd,
          xAxisIndex: gridIndex,
          yAxisIndex: gridIndex,
          itemStyle: {
            color: (params: any) => params.data >= 0 ? '#ef4444' : '#16a34a'
          }
        }
      )
      legend.push('DIF', 'DEA', 'MACD')
      dataZooms[0].xAxisIndex.push(gridIndex)
      dataZooms[1].xAxisIndex.push(gridIndex)
      currentTop += subplotHeight + gridGap
      gridIndex++
    } else if (indicator === 'RSI') {
      const rsiData = calculateRSI(closeData, 14)
      grids.push({ left: 60, right: 60, top: currentTop, height: subplotHeight })
      xAxes.push({
        type: 'category',
        data: category,
        gridIndex,
        boundaryGap: true,
        axisLine: { onZero: false },
        splitLine: { show: false },
        axisLabel: { show: false }
      })
      yAxes.push({
        scale: false,
        gridIndex,
        min: 0,
        max: 100,
        splitLine: { show: true },
        axisLabel: { inside: false }
      })
      series.push(
        {
          type: 'line',
          name: 'RSI',
          data: rsiData,
          xAxisIndex: gridIndex,
          yAxisIndex: gridIndex,
          showSymbol: false,
          lineStyle: { width: 1.5, color: '#9C27B0' },
          markLine: {
            silent: true,
            symbol: 'none',
            data: [
              { yAxis: 70, lineStyle: { color: '#ef4444', type: 'dashed' } },
              { yAxis: 30, lineStyle: { color: '#16a34a', type: 'dashed' } }
            ]
          }
        }
      )
      legend.push('RSI')
      dataZooms[0].xAxisIndex.push(gridIndex)
      dataZooms[1].xAxisIndex.push(gridIndex)
      currentTop += subplotHeight + gridGap
      gridIndex++
    } else if (indicator === 'KDJ') {
      const kdjData = calculateKDJ(highData, lowData, closeData, 9)
      grids.push({ left: 60, right: 60, top: currentTop, height: subplotHeight })
      xAxes.push({
        type: 'category',
        data: category,
        gridIndex,
        boundaryGap: true,
        axisLine: { onZero: false },
        splitLine: { show: false },
        axisLabel: { show: gridIndex === grids.length - 1 }
      })
      yAxes.push({
        scale: false,
        gridIndex,
        min: 0,
        max: 100,
        splitLine: { show: true },
        axisLabel: { inside: false }
      })
      series.push(
        {
          type: 'line',
          name: 'K',
          data: kdjData.k,
          xAxisIndex: gridIndex,
          yAxisIndex: gridIndex,
          showSymbol: false,
          lineStyle: { width: 1.5, color: '#FF6B6B' }
        },
        {
          type: 'line',
          name: 'D',
          data: kdjData.d,
          xAxisIndex: gridIndex,
          yAxisIndex: gridIndex,
          showSymbol: false,
          lineStyle: { width: 1.5, color: '#4ECDC4' }
        },
        {
          type: 'line',
          name: 'J',
          data: kdjData.j,
          xAxisIndex: gridIndex,
          yAxisIndex: gridIndex,
          showSymbol: false,
          lineStyle: { width: 1.5, color: '#FFD93D' }
        }
      )
      legend.push('K', 'D', 'J')
      dataZooms[0].xAxisIndex.push(gridIndex)
      dataZooms[1].xAxisIndex.push(gridIndex)
      currentTop += subplotHeight + gridGap
      gridIndex++
    }
  })

  // 显示最后一个grid的xAxis标签，并留出空间给 dataZoom
  if (xAxes.length > 1) {
    xAxes[xAxes.length - 1].axisLabel.show = true
  } else {
    // 如果没有副图，主图显示x轴标签
    xAxes[0].axisLabel.show = true
  }

  // 如果有保存的 dataZoom 状态（加载更多数据时），恢复位置
  if (savedDataZoomState.value && isLoadingMore.value) {
    // 计算新的位置：保持可视区域的数据条数不变
    const oldTotalData = klineData.value.length - 200 // 加载前的数据总量
    const newTotalData = klineData.value.length // 加载后的数据总量
    const addedDataCount = newTotalData - oldTotalData // 新增的数据量
    
    if (addedDataCount > 0 && oldTotalData > 0) {
      // 计算原来显示的数据条数
      const oldVisibleRange = (savedDataZoomState.value.end - savedDataZoomState.value.start) / 100 * oldTotalData
      
      // 新的 start 位置需要向后偏移，以保持相同的可视数据
      const newStart = (savedDataZoomState.value.start / 100 * oldTotalData + addedDataCount) / newTotalData * 100
      const newEnd = newStart + (oldVisibleRange / newTotalData * 100)
      
      // 更新所有 dataZoom 的位置
      dataZooms.forEach(dz => {
        dz.start = Math.max(0, newStart)
        dz.end = Math.min(100, newEnd)
      })
      
      console.log(`📍 恢复图表位置: start ${savedDataZoomState.value.start.toFixed(2)}% -> ${newStart.toFixed(2)}%, end ${savedDataZoomState.value.end.toFixed(2)}% -> ${newEnd.toFixed(2)}%`)
    }
  }

  kOption.value = {
    legend: {
      data: legend,
      top: 10,
      left: 'center'
    },
    tooltip: {
      trigger: 'axis',
      axisPointer: { type: 'cross' }
    },
    grid: grids,
    xAxis: xAxes,
    yAxis: yAxes,
    dataZoom: dataZooms,
    series
  }
}

async function fetchKline(isLoadMore = false) {
  try {
    if (isLoadMore && (isLoadingMore.value || !hasMoreData.value)) {
      return // 正在加载或没有更多数据时不重复加载
    }

    // 保存当前数据量，用于计算加载后的位置
    const oldDataLength = klineData.value.length

    if (isLoadMore) {
      isLoadingMore.value = true
      currentLimit.value += 200 // 每次加载200条
    } else {
      // 初始加载，重置状态
      currentLimit.value = 200
      hasMoreData.value = true
      klineData.value = []
    }

    const param = periodLabelToParam(period.value)
    const res = await stocksApi.getKline(code.value, param as any, currentLimit.value, 'none')
    const d: any = (res as any)?.data || {}
    klineSource.value = d.source
    const items: any[] = Array.isArray(d.items) ? d.items : []

    // 检查是否还有更多数据
    if (items.length < currentLimit.value) {
      hasMoreData.value = false // 没有更多数据了
    }

    // 计算新增数据量
    const newDataCount = items.length - oldDataLength

    // 保存原始数据
    klineData.value = items

    // 更新最后价格
    if (items.length > 0) {
      const lastItem = items[items.length - 1]
      lastKTime.value = String(lastItem.time || lastItem.trade_time || lastItem.trade_date || '')
      lastKClose.value = Number(lastItem.close ?? null)
    }

    // 渲染图表
    updateKlineChart()

    // 显示加载提示
    if (isLoadMore) {
      if (newDataCount > 0) {
        ElMessage.success({
          message: `自动加载了 ${newDataCount} 条历史数据${!hasMoreData.value ? '（全部数据已加载）' : ''}`,
          duration: 2000
        })
      } else if (!hasMoreData.value) {
        ElMessage.info({
          message: '已加载全部数据',
          duration: 2000
        })
      }
    }
  } catch (e) {
    console.error('获取K线失败', e)
    if (isLoadMore) {
      ElMessage.error('加载更多数据失败')
    }
  } finally {
    if (isLoadMore) {
      isLoadingMore.value = false
      // 重置 lastDataZoomStart，防止立即再次触发
      lastDataZoomStart.value = 10
      // 清除保存的状态
      setTimeout(() => {
        savedDataZoomState.value = null
      }, 100)
    }
  }
}

// 均线管理函数
function addMA() {
  editingMA.value = { period: 30, color: '#FF6B6B', enabled: true }
  showMADialog.value = true
}

function editMA(ma: MAConfig) {
  editingMA.value = { ...ma }
  showMADialog.value = true
}

function deleteMA(index: number) {
  maConfigs.value.splice(index, 1)
}

function saveMA() {
  if (!editingMA.value) return
  
  const existingIndex = maConfigs.value.findIndex(ma => ma.period === editingMA.value!.period)
  if (existingIndex >= 0) {
    maConfigs.value[existingIndex] = editingMA.value
  } else {
    maConfigs.value.push(editingMA.value)
  }
  
  showMADialog.value = false
  editingMA.value = null
}

function toggleMA(index: number) {
  maConfigs.value[index].enabled = !maConfigs.value[index].enabled
}

// 格式化交易时间
function formatTradeTime(timestamp: string): string {
  if (!timestamp) return '-'
  try {
    const date = new Date(timestamp)
    return date.toLocaleString('zh-CN', {
      year: 'numeric',
      month: '2-digit',
      day: '2-digit',
      hour: '2-digit',
      minute: '2-digit',
      second: '2-digit'
    })
  } catch (e) {
    return timestamp
  }
}

// 获取盈亏标签类型（赚了红色，亏了绿色，0也是红色）
function getProfitTagType(): string {
  if (!avgPrice.value || !quote.price || currentQuantity.value <= 0) return 'info'
  const totalProfit = (quote.price - avgPrice.value) * currentQuantity.value
  if (totalProfit >= 0) return 'danger'  // 红色（赚了或0）
  return 'success'  // 绿色（亏了）
}

// 格式化总持仓盈亏金额
function formatProfit(): string {
  if (!avgPrice.value || !quote.price || currentQuantity.value <= 0) return '-'
  const totalProfit = (quote.price - avgPrice.value) * currentQuantity.value
  const profitStr = totalProfit >= 0 ? `+￥${totalProfit.toFixed(2)}` : `￥${totalProfit.toFixed(2)}`
  return profitStr
}


// 新闻
const newsFilter = ref('all')
const newsItems = ref<any[]>([])
const newsSource = ref<string | undefined>(undefined)

function cleanTitle(s: any): string {
  const t = String(s || '')
  return t.replace(/<[^>]+>/g, '').replace(/&nbsp;/g, ' ').trim()
}

async function fetchNews() {
  try {
    const res = await stocksApi.getNews(code.value, 30, 50, true)
    const d: any = (res as any)?.data || {}
    const itemsRaw: any[] = Array.isArray(d.items) ? d.items : []
    newsItems.value = itemsRaw.map((it: any) => {
      const title = cleanTitle(it.title || it.summary || it.name || '')
      const url = it.url || it.link || '#'
      const source = it.source || d.source || ''
      const time = it.time || it.pub_time || it.publish_time || it.pub_date || ''
      const type = it.type || 'news'
      return { title, url, source, time, type }
    })
    newsSource.value = d.source
  } catch (e) {
    console.error('获取新闻失败', e)
  }
}

const filteredNews = computed(() => {
  if (newsFilter.value === 'news') return newsItems.value.filter(x => x.type === 'news')
  if (newsFilter.value === 'announcement') return newsItems.value.filter(x => x.type === 'announcement')
  return newsItems.value
})

// 基本面（mock）
const basics = reactive({
  industry: '-',
  sector: '-',
  marketCap: NaN,
  pe: NaN,
  pb: NaN,              // 🔥 新增：市净率
  ps: NaN,              // 🔥 新增：市销率
  roe: NaN,
  debtRatio: NaN,
  peIsRealtime: false,  // PE是否为实时数据
  peSource: '',         // PE数据来源
  peUpdatedAt: null     // PE更新时间
})

// 操作
function onAnalyze() {
  router.push({ name: 'SingleAnalysis', query: { stock: code.value } })
}
async function onToggleFavorite() {
  try {
    if (!isFav.value) {
      const payload = {
        symbol: symbol.value,
        stock_code: symbol.value,  // 兼容字段
        stock_name: stockName.value,
        market: market.value
      }
      await favoritesApi.add(payload)
      isFav.value = true
      ElMessage.success('已加入自选')
    } else {
      await favoritesApi.remove(code.value)
      isFav.value = false
      ElMessage.success('已移出自选')
    }
  } catch (e: any) {
    console.error('自选操作失败', e)
    ElMessage.error(e?.message || '自选操作失败')
  }
}

// 从交易记录打开下单对话框
function openOrderDialogFromTrades() {
  order.value.code = code.value
  order.value.side = 'buy'
  order.value.qty = 100
  order.value.price = quote.price && Number.isFinite(quote.price) ? quote.price : null
  orderDialog.value = true
}

// 获取最新价格
async function fetchLatestPrice() {
  if (!code.value) return

  try {
    fetchingPrice.value = true
    await fetchQuote()
    if (quote.price && Number.isFinite(quote.price)) {
      order.value.price = quote.price
    } else {
      order.value.price = null
      ElMessage.warning('未能获取到有效价格，请手动输入')
    }
  } catch (error) {
    console.warn('获取最新价格失败:', error)
    order.value.price = null
    ElMessage.error('获取价格失败')
  } finally {
    fetchingPrice.value = false
  }
}

// 提交订单
async function submitOrder() {
  try {
    // 验证价格
    if (order.value.price === null || order.value.price <= 0) {
      ElMessage.warning('请输入有效的价格')
      return
    }

    const payload: any = {
      side: order.value.side,
      code: order.value.code,
      quantity: Number(order.value.qty),
      price: Number(order.value.price)
    }

    const res = await paperApi.placeOrder(payload)
    if (res.success) {
      ElMessage.success('下单成功')
      orderDialog.value = false
      // 刷新交易记录
      await fetchStockTrades()
    } else {
      ElMessage.error(res.message || '下单失败')
    }
  } catch (e: any) {
    ElMessage.error(e?.message || '下单失败')
  }
}

// 获取最新的历史分析报告
async function fetchLatestAnalysis() {
  try {
    console.log('🔍 [fetchLatestAnalysis] 开始获取历史分析报告, symbol:', symbol.value)

    const resp: any = await analysisApi.getHistory({
      symbol: symbol.value,
      stock_code: symbol.value,  // 兼容字段
      page: 1,
      page_size: 1,
      status: 'completed'
    })

    console.log('🔍 [fetchLatestAnalysis] API响应:', resp)
    console.log('🔍 [fetchLatestAnalysis] resp.data:', resp?.data)
    console.log('🔍 [fetchLatestAnalysis] resp.data.data:', resp?.data?.data)

    // 修复：API返回格式是 { success: true, data: { tasks: [...] } }
    // 所以需要先取 resp.data，再取 data.tasks
    const responseData = resp?.data || resp
    console.log('🔍 [fetchLatestAnalysis] responseData:', responseData)

    // 如果responseData有success字段，说明是标准响应格式，需要再取一层data
    const actualData = responseData?.success ? responseData.data : responseData
    console.log('🔍 [fetchLatestAnalysis] actualData:', actualData)

    const tasks = actualData?.tasks || actualData?.analyses || []
    console.log('🔍 [fetchLatestAnalysis] tasks:', tasks)
    console.log('🔍 [fetchLatestAnalysis] tasks.length:', tasks?.length)
    console.log('🔍 [fetchLatestAnalysis] tasks && tasks.length > 0:', tasks && tasks.length > 0)

    if (tasks && tasks.length > 0) {
      const latestTask = tasks[0]
      console.log('✅ [fetchLatestAnalysis] 找到任务:', latestTask)
      console.log('🔍 [fetchLatestAnalysis] latestTask.result_data:', latestTask.result_data)
      console.log('🔍 [fetchLatestAnalysis] latestTask.result:', latestTask.result)
      console.log('🔍 [fetchLatestAnalysis] latestTask.task_id:', latestTask.task_id)
      console.log('🔍 [fetchLatestAnalysis] latestTask.end_time:', latestTask.end_time)

      // 保存任务信息（包含 end_time 等）
      lastTaskInfo.value = latestTask

      // 优先使用 result_data 字段（后端实际返回的字段名）
      if (latestTask.result_data) {
        lastAnalysis.value = latestTask.result_data
        analysisStatus.value = 'completed'
        console.log('✅ 加载历史分析报告成功 (result_data):', latestTask.result_data)
        console.log('🔍 [fetchLatestAnalysis] lastAnalysis.value.reports:', lastAnalysis.value?.reports)
      }
      // 兼容旧的 result 字段
      else if (latestTask.result) {
        lastAnalysis.value = latestTask.result
        analysisStatus.value = 'completed'
        console.log('✅ 加载历史分析报告成功 (result):', latestTask.result)
        console.log('🔍 [fetchLatestAnalysis] lastAnalysis.value.reports:', lastAnalysis.value?.reports)
      }
      // 否则尝试通过 task_id 获取结果
      else if (latestTask.task_id) {
        console.log('🔍 [fetchLatestAnalysis] 通过task_id获取结果:', latestTask.task_id)
        try {
          const resultResp: any = await analysisApi.getTaskResult(latestTask.task_id)
          console.log('🔍 [fetchLatestAnalysis] getTaskResult响应:', resultResp)
          lastAnalysis.value = resultResp?.data || resultResp
          analysisStatus.value = 'completed'
          console.log('✅ 通过 task_id 加载分析报告成功:', lastAnalysis.value)
          console.log('🔍 [fetchLatestAnalysis] lastAnalysis.value.reports:', lastAnalysis.value?.reports)
        } catch (e) {
          console.warn('⚠️ 获取任务结果失败:', e)
        }
      }
    } else {
      console.log('ℹ️ 该股票暂无历史分析报告')
      console.log('🔍 [fetchLatestAnalysis] 判断条件: tasks=', tasks, ', tasks.length=', tasks?.length)
    }
  } catch (e) {
    console.warn('⚠️ 获取历史分析报告失败:', e)
  }
}

// 格式化
function fmtPrice(v: any) { const n = Number(v); return Number.isFinite(n) ? n.toFixed(2) : '-' }
function fmtPercent(v: any) { const n = Number(v); return Number.isFinite(n) ? `${n>0?'+':''}${n.toFixed(2)}%` : '-' }
function fmtVolume(v: any) {
  const n = Number(v)
  if (!Number.isFinite(n)) return '-'

  // 🔥 数据库存储的是"股"，直接显示为"万股"或"亿股"
  if (n >= 1e8) return (n/1e8).toFixed(2) + '亿股'
  if (n >= 1e4) return (n/1e4).toFixed(2) + '万股'
  return n.toFixed(0) + '股'
}
function fmtAmount(v: any) {
  const n = Number(v)
  if (!Number.isFinite(n)) return '-'
  if (n >= 1e12) return (n/1e12).toFixed(2) + '万亿'
  if (n >= 1e8) return (n/1e8).toFixed(2) + '亿'
  if (n >= 1e4) return (n/1e4).toFixed(2) + '万'
  return n.toFixed(0)
}
// 🔥 新增：格式化同步时间（添加时区标识）
function formatSyncTime(timeStr: string | null | undefined): string {
  if (!timeStr) return '未同步'
  // 后端返回的时间已经是 UTC+8 时区，添加时区标识
  return `${timeStr} (UTC+8)`
}

// 🔥 新增：格式化股票更新时间
function formatQuoteUpdateTime(timeStr: string | null | undefined): string {
  if (!timeStr) return '未更新'
  try {
    // 后端返回的时间已经是 UTC+8 时区，但没有时区标识
    // 需要手动添加 +08:00 时区标识，然后转换为本地时间显示
    let isoString = timeStr
    if (!timeStr.includes('+') && !timeStr.includes('Z')) {
      // 如果没有时区标识，添加 +08:00
      isoString = timeStr.replace(/(\.\d+)?$/, '+08:00')
    }
    const date = new Date(isoString)
    const year = date.getFullYear()
    const month = String(date.getMonth() + 1).padStart(2, '0')
    const day = String(date.getDate()).padStart(2, '0')
    const hours = String(date.getHours()).padStart(2, '0')
    const minutes = String(date.getMinutes()).padStart(2, '0')
    const seconds = String(date.getSeconds()).padStart(2, '0')
    return `${year}-${month}-${day} ${hours}:${minutes}:${seconds}`
  } catch (e) {
    return timeStr
  }
}

// 🔥 新增：格式化同步间隔
function formatSyncInterval(seconds: number): string {
  if (!seconds || seconds <= 0) return ''

  if (seconds < 60) {
    // 小于60秒，显示秒数
    return `(每${seconds}秒)`
  } else if (seconds < 3600) {
    // 小于1小时，显示分钟数
    const minutes = Math.round(seconds / 60)
    return `(每${minutes}分钟)`
  } else {
    // 大于等于1小时，显示小时数
    const hours = Math.round(seconds / 3600)
    return `(每${hours}小时)`
  }
}
function fmtConf(v: any) {
  const n = Number(v)
  if (!Number.isFinite(n)) return '-'
  const pct = n <= 1 ? n * 100 : n
  return `${Math.round(pct)}%`
}

import { formatDateTimeWithRelative, formatDateTime } from '@/utils/datetime'

// 格式化分析时间（处理UTC时间转换为中国本地时间）
function formatAnalysisTime(dateStr: any): string {
  return formatDateTimeWithRelative(dateStr)
}

// 格式化新闻时间（简洁格式：MM-DD HH:mm）
function formatNewsTime(dateStr: string | null | undefined): string {
  if (!dateStr) return '-'

  try {
    // 使用 formatDateTime 工具函数，自定义格式
    return formatDateTime(dateStr, {
      timeZone: 'Asia/Shanghai',
      month: '2-digit',
      day: '2-digit',
      hour: '2-digit',
      minute: '2-digit',
      hour12: false
    }).replace(/\//g, '-').replace(/,/g, '')  // 移除逗号和斜杠
  } catch (e) {
    console.error('新闻时间格式化错误:', e, dateStr)
    return String(dateStr)
  }
}

// 格式化报告名称
function formatReportName(key: string): string {
  // 完整的13个报告映射
  const nameMap: Record<string, string> = {
    // 分析师团队 (4个)
    'market_report': '📈 市场技术分析',
    'sentiment_report': '💭 市场情绪分析',
    'news_report': '📰 新闻事件分析',
    'fundamentals_report': '💰 基本面分析',

    // 研究团队 (3个)
    'bull_researcher': '🐂 多头研究员',
    'bear_researcher': '🐻 空头研究员',
    'research_team_decision': '🔬 研究经理决策',

    // 交易团队 (1个)
    'trader_investment_plan': '💼 交易员计划',

    // 风险管理团队 (4个)
    'risky_analyst': '⚡ 激进分析师',
    'safe_analyst': '🛡️ 保守分析师',
    'neutral_analyst': '⚖️ 中性分析师',
    'risk_management_decision': '👔 投资组合经理',

    // 最终决策 (1个)
    'final_trade_decision': '🎯 最终交易决策',

    // 兼容旧字段
    'investment_plan': '📋 投资建议',
    'investment_debate_state': '🔬 研究团队决策（旧）',
    'risk_debate_state': '⚖️ 风险管理团队（旧）'
  }
  return nameMap[key] || key.replace(/_/g, ' ').replace(/\b\w/g, l => l.toUpperCase())
}

// 渲染Markdown
function renderMarkdown(content: string): string {
  if (!content) return '<p>暂无内容</p>'
  try {
    return marked(content)
  } catch (e) {
    console.error('Markdown渲染失败:', e)
    return `<pre>${content}</pre>`
  }
}

// 打开指定报告
function openReport(reportKey: string) {
  showReportsDialog.value = true
  activeReportTab.value = reportKey
}

// 导出报告
function exportReport() {
  if (!lastAnalysis.value?.reports) {
    ElMessage.warning('暂无报告可导出')
    return
  }

  // 生成Markdown格式的完整报告
  let fullReport = `# ${code.value} 股票分析报告\n\n`

  // 格式化分析时间用于报告
  const reportTime = lastTaskInfo.value?.end_time
    ? new Date(lastTaskInfo.value.end_time).toLocaleString('zh-CN', {
        timeZone: 'Asia/Shanghai',
        year: 'numeric',
        month: '2-digit',
        day: '2-digit',
        hour: '2-digit',
        minute: '2-digit',
        hour12: false
      })
    : lastAnalysis.value?.analysis_date

  fullReport += `**分析时间**: ${reportTime}\n`
  fullReport += `**投资建议**: ${lastAnalysis.value.recommendation}\n`
  fullReport += `**信心度**: ${fmtConf(lastAnalysis.value.confidence_score)}\n\n`
  fullReport += `---\n\n`

  for (const [key, content] of Object.entries(lastAnalysis.value.reports)) {
    fullReport += `## ${formatReportName(key)}\n\n`
    fullReport += `${content}\n\n`
    fullReport += `---\n\n`
  }

  // 创建下载链接
  const blob = new Blob([fullReport], { type: 'text/markdown;charset=utf-8' })
  const url = URL.createObjectURL(blob)
  const link = document.createElement('a')
  link.href = url

  // 使用分析日期作为文件名（简化格式）
  const fileDate = lastAnalysis.value.analysis_date || new Date().toISOString().slice(0, 10)
  link.download = `${code.value}_分析报告_${fileDate}.md`
  document.body.appendChild(link)
  link.click()
  document.body.removeChild(link)
  URL.revokeObjectURL(url)

  ElMessage.success('报告已导出')
}

</script>

<style scoped lang="scss">
.stock-detail {
  display: flex; flex-direction: column; gap: 16px;
}

.header { display: flex; justify-content: space-between; align-items: center; }
.title { display: flex; align-items: center; gap: 12px; }
.code { font-size: 22px; font-weight: 700; }
.name { font-size: 18px; color: var(--el-text-color-regular); }
.actions { display: flex; gap: 8px; }

.quote-card { border-radius: 12px; }
.quote { display: flex; flex-direction: column; gap: 8px; }
.price-row { display: flex; align-items: center; gap: 12px; }
.price { font-size: 32px; font-weight: 800; }
.change { font-size: 16px; font-weight: 700; }
.up { color: #e53935; }
.down { color: #16a34a; }
.stats { display: grid; grid-template-columns: repeat(8, 1fr); gap: 10px; margin-top: 6px; }
.stats .item { display: flex; flex-direction: column; font-size: 12px; color: var(--el-text-color-secondary); }
.stats .item b { color: var(--el-text-color-primary); font-size: 14px; }

.body { margin-top: 4px; }
.card-hd { display: flex; align-items: center; justify-content: space-between; }
.kline-controls { display: flex; align-items: center; gap: 8px; }
.kline-container { position: relative; }
.loading-overlay {
  position: absolute;
  top: 10px;
  left: 50%;
  transform: translateX(-50%);
  z-index: 100;
  background: rgba(64, 158, 255, 0.95);
  color: white;
  padding: 8px 16px;
  border-radius: 20px;
  display: flex;
  align-items: center;
  gap: 8px;
  box-shadow: 0 2px 12px rgba(0, 0, 0, 0.15);
  font-size: 13px;
  font-weight: 500;
}
.k-chart { 
  height: 580px; /* 增加高度以适应副图 */ 
  min-height: 450px;
}
.kline-footer { 
  display: flex; 
  align-items: center; 
  justify-content: space-between; 
  margin-top: 8px; 
  gap: 12px;
}
.legend { 
  font-size: 12px; 
  color: var(--el-text-color-secondary); 
  flex: 1;
}

.news-card .news-list { display: flex; flex-direction: column; }
.news-item { padding: 10px 12px; border-bottom: 1px solid var(--el-border-color-lighter); transition: background-color .2s ease; }
.news-item:last-child { border-bottom: none; }
.news-item:hover { background: var(--el-fill-color-light); border-radius: 8px; }
.news-item .row { display: flex; align-items: flex-start; justify-content: space-between; gap: 8px; }
.news-item .left { display: flex; align-items: flex-start; gap: 8px; flex: 1 1 auto; min-width: 0; }
.news-item .tag { flex: 0 0 auto; }
.news-item .title { font-weight: 600; display: flex; align-items: center; gap: 6px; flex: 1 1 auto; min-width: 0; }
.news-item .title a, .news-item .title span { color: var(--el-text-color-primary); text-decoration: none; display: -webkit-box; -webkit-box-orient: vertical; -webkit-line-clamp: 2; overflow: hidden; }
.news-item .title a:hover { text-decoration: underline; }
.news-item .ext { color: var(--el-text-color-placeholder); font-size: 14px; }
.news-item .title:hover .ext { color: var(--el-color-primary); }
.news-item .right { color: var(--el-text-color-secondary); font-size: 12px; white-space: nowrap; margin-left: 8px; }
.news-item .meta { font-size: 12px; color: var(--el-text-color-secondary); margin-top: 4px; }

.sentiment { font-size: 12px; }
.sentiment.pos { color: #ef4444; }
.sentiment.neu { color: #64748b; }
.sentiment.neg { color: #10b981; }

.facts { display: flex; flex-direction: column; gap: 10px; }
.fact { display: flex; flex-direction: column; font-size: 12px; }
.fact span { color: var(--el-text-color-secondary); margin-bottom: 4px; }
.fact b { font-size: 14px; color: var(--el-text-color-primary); }

.quick-actions { display: flex; flex-direction: column; gap: 8px; }

@media (max-width: 1024px) {
  .stats { grid-template-columns: repeat(4, 1fr); }
}

/* 报告相关样式 */
.reports-section {
  margin-top: 8px;
}

.reports-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 16px;
  margin-top: 8px;
}

.reports-title {
  font-size: 15px;
  font-weight: 600;
  color: var(--el-text-color-primary);
  display: flex;
  align-items: center;
  gap: 6px;
}

.reports-preview {
  display: flex;
  flex-wrap: wrap;
  gap: 10px;
  padding: 12px;
  background: var(--el-fill-color-lighter);
  border-radius: 8px;
}

.report-tag {
  cursor: pointer;
  transition: all 0.2s ease;
  font-size: 13px;
  padding: 6px 12px;
}

.report-tag:hover {
  transform: translateY(-2px);
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
}

/* 报告对话框样式 */
.reports-dialog :deep(.el-dialog__body) {
  padding: 0;
}

.report-content {
  padding: 20px;
}

.markdown-body {
  font-size: 14px;
  line-height: 1.8;
  color: var(--el-text-color-primary);
}

.markdown-body h1 {
  font-size: 24px;
  font-weight: 700;
  margin: 20px 0 16px;
  padding-bottom: 8px;
  border-bottom: 2px solid var(--el-border-color);
}

.markdown-body h2 {
  font-size: 20px;
  font-weight: 600;
  margin: 16px 0 12px;
}

.markdown-body h3 {
  font-size: 16px;
  font-weight: 600;
  margin: 12px 0 8px;
}

.markdown-body p {
  margin: 8px 0;
}

.markdown-body ul, .markdown-body ol {
  margin: 8px 0;
  padding-left: 24px;
}

.markdown-body li {
  margin: 4px 0;
}

.markdown-body code {
  background: var(--el-fill-color-light);
  padding: 2px 6px;
  border-radius: 4px;
  font-family: 'Courier New', monospace;
}

.markdown-body pre {
  background: var(--el-fill-color-light);
  padding: 12px;
  border-radius: 8px;
  overflow-x: auto;
  margin: 12px 0;
}

.markdown-body blockquote {
  border-left: 4px solid var(--el-color-primary);
  padding-left: 12px;
  margin: 12px 0;
  color: var(--el-text-color-secondary);
}

.markdown-body table {
  width: 100%;
  border-collapse: collapse;
  margin: 12px 0;
}

.markdown-body th, .markdown-body td {
  border: 1px solid var(--el-border-color);
  padding: 8px 12px;
  text-align: left;
}

.markdown-body th {
  background: var(--el-fill-color-light);
  font-weight: 600;
}

.analysis-detail-card .detail {
  display: flex;
  flex-direction: column;
  gap: 16px;
}

/* 分析时间元信息 */
.analysis-meta {
  display: flex;
  align-items: center;
  gap: 24px;
  padding: 8px 12px;
  background: var(--el-fill-color-lighter);
  border-radius: 6px;
  font-size: 13px;
  color: var(--el-text-color-secondary);
}

.analysis-meta .analysis-time,
.analysis-meta .confidence {
  display: flex;
  align-items: center;
  gap: 6px;
}

.analysis-meta .el-icon {
  font-size: 14px;
}

/* 投资建议盒子 - 重点突出 */
.recommendation-box {
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  border-radius: 12px;
  padding: 20px 24px;
  box-shadow: 0 4px 16px rgba(102, 126, 234, 0.25);
  transition: all 0.3s ease;
  margin: 16px 0;
}

.recommendation-box:hover {
  box-shadow: 0 6px 20px rgba(102, 126, 234, 0.35);
  transform: translateY(-2px);
}

.recommendation-header {
  display: flex;
  align-items: center;
  gap: 8px;
  margin-bottom: 16px;
  color: rgba(255, 255, 255, 0.95);
  font-size: 15px;
  font-weight: 600;
}

.recommendation-header .icon {
  font-size: 20px;
}

.recommendation-content {
  background: rgba(255, 255, 255, 0.98);
  border-radius: 8px;
  padding: 16px 20px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
}

.recommendation-text {
  color: #1f2937;
  font-size: 15px;
  line-height: 1.8;
  font-weight: 500;
  word-wrap: break-word;
  word-break: break-word;
  white-space: pre-wrap;
}

/* 分析摘要 */
.summary-section {
  padding: 18px 20px;
  background: #f8fafc;
  border-radius: 8px;
  border-left: 4px solid #3b82f6;
  margin-top: 16px;
}

.summary-title {
  display: flex;
  align-items: center;
  gap: 8px;
  font-size: 15px;
  font-weight: 600;
  color: #1e40af;
  margin-bottom: 12px;
}

.summary-title .el-icon {
  font-size: 18px;
  color: #3b82f6;
}

.summary-text {
  color: #334155;
  line-height: 1.8;
  font-size: 14px;
  word-wrap: break-word;
  word-break: break-word;
  white-space: pre-wrap;
}

/* 同步状态提示 */
.sync-status {
  display: flex;
  align-items: center;
  gap: 6px;
  margin-top: 12px;
  padding: 8px 12px;
  background: #f0f9ff;
  border-radius: 6px;
  border: 1px solid #bae6fd;
  font-size: 13px;
  color: #0369a1;
}

.sync-status .el-icon {
  font-size: 14px;
  color: #0284c7;
}

.sync-info {
  display: flex;
  align-items: center;
  gap: 4px;
  flex-wrap: wrap;
}

/* 均线管理样式 */
.ma-item {
  display: flex;
  align-items: center;
  justify-content: space-between;
  width: 100%;
  padding: 4px 0;
}

.ma-actions {
  display: flex;
  gap: 4px;
  margin-left: 12px;
}

/* 交易记录对话框样式 */
.trades-dialog {
  max-height: 600px;
  overflow-y: auto;
}

.trades-summary {
  margin-bottom: 16px;
}

.trades-note {
  display: flex;
  align-items: center;
  gap: 6px;
  margin-top: 12px;
  padding: 8px 12px;
  background: #f0f9ff;
  border-radius: 6px;
  border: 1px solid #bae6fd;
  font-size: 12px;
  color: #0369a1;
}

.trades-note .el-icon {
  font-size: 14px;
  color: #0284c7;
}
</style>
