<template>
  <div class="bug-broadcast-page">
    <a-card :bordered="false" class="filter-card">
      <a-row :gutter="16" type="flex" align="bottom" class="filter-row">
        <a-col :xs="24" class="project-filter-col">
          <label class="field-label">项目名称</label>
          <a-select
            class="project-select"
            show-search
            allow-clear
            placeholder="请选择项目"
            option-filter-prop="children"
            :value="selectedProjectId"
            :loading="projectLoading"
            :dropdown-style="{ maxHeight: '260px', overflow: 'auto' }"
            @focus="fetchProjects"
            @change="handleProjectChange"
          >
            <a-select-option
              v-for="project in projects"
              :key="project.id"
              :value="project.id"
            >
              {{ project.name }}
            </a-select-option>
          </a-select>
        </a-col>
        <a-col :xs="24" class="date-filter-col">
          <label class="field-label">统计日期</label>
          <a-range-picker
            class="date-picker"
            :value="dateRange"
            format="YYYY.M.D"
            :allow-clear="false"
            :ranges="dateShortcutRanges"
            :get-calendar-container="getDatePickerContainer"
            @change="handleDateChange"
          />
        </a-col>
        <a-col :xs="24" class="button-filter-col">
          <a-button
            block
            type="primary"
            size="large"
            class="generate-btn"
            :loading="generating"
            :disabled="!selectedProjectId"
            @click="generateBroadcast"
          >
            <a-icon v-if="!generating" type="thunderbolt" />
            {{ generating ? '生成中...' : '生成播报' }}
          </a-button>
        </a-col>
        <a-col :xs="24" class="export-filter-col">
          <a-dropdown :disabled="!reportText">
            <a-button block size="large" class="export-btn" :disabled="!reportText">
              <a-icon type="download" />
              导出图片 <a-icon type="down" />
            </a-button>
            <a-menu slot="overlay" @click="handleExportMenuClick">
              <a-menu-item key="trend">Bug趋势图</a-menu-item>
              <a-menu-item key="oxiOverview">线上oxi环境Bug统计</a-menu-item>
              <a-menu-item key="nonOxiOverview">非线上oxi环境Bug统计</a-menu-item>
            </a-menu>
          </a-dropdown>
        </a-col>
      </a-row>
    </a-card>

    <a-row :gutter="24" class="content-row">
      <a-col :span="24">
        <a-card :bordered="false" class="report-card">
          <div class="report-scroll">
            <div v-if="reportText" ref="reportPanel" class="report-panel">
              <div ref="overviewPanel" class="overview-head">
                <div class="overview-list">
                  <div v-for="group in overviewGroups" :key="group.key" class="overview-group" :data-group-key="group.key">
                    <div class="overview-group-title">{{ group.title }}</div>
                    <div v-for="item in group.items" :key="item.key" class="overview-item" :class="{ 'overview-sub-item': item.sub }">
                      <div class="overview-metric">
                        <div class="overview-line" :class="{ 'overview-sub-line': item.sub }">
                          <span class="overview-name">{{ item.label }}</span>
                          <strong class="overview-count">{{ item.count }} 个</strong>
                          <a-button
                            v-if="item.detailKey"
                            size="small"
                            type="primary"
                            ghost
                            class="overview-detail-btn"
                            @click="toggleOverviewDetail(item.detailKey)"
                          >
                            {{ isOverviewDetailExpanded(item.detailKey) ? '收起' : '明细' }}
                          </a-button>
                        </div>
                        <span v-if="item.highPriorityCount !== undefined" class="overview-priority">紧急/高优先级：{{ item.highPriorityCount }} 个</span>
                      </div>
                      <div v-if="(item.groupStats && item.groupStats.length) || (item.moduleStats && item.moduleStats.length) || (item.childStats && item.childStats.length)" class="overview-detail">
                        <div v-if="item.childStats && item.childStats.length" class="overview-child-list">
                          <div v-for="child in item.childStats" :key="`${item.key}-${child.key}`" class="overview-child-card">
                            <div class="overview-child-line">
                              <span>{{ child.label }}</span>
                              <strong>{{ child.count }} 个</strong>
                            </div>
                            <div v-if="child.moduleStats && child.moduleStats.length" class="overview-tags module-tags child-tags">
                              <span class="owner-label">模块名称</span>
                              <div class="owner-list">
                                <i v-for="module in child.moduleStats" :key="`${item.key}-${child.key}-module-${module.name}`">{{ module.name }} <b>{{ module.count }}</b></i>
                              </div>
                            </div>
                            <div v-if="child.groupStats && child.groupStats.length" class="overview-tags owner-tags child-tags">
                              <span class="owner-label">{{ child.groupLabel }}</span>
                              <div class="owner-list">
                                <i v-for="groupItem in child.groupStats" :key="`${item.key}-${child.key}-${groupItem.name}`">{{ groupItem.name }} <b>{{ groupItem.count }}</b></i>
                              </div>
                            </div>
                          </div>
                        </div>
                        <div v-if="item.moduleStats && item.moduleStats.length" class="overview-tags module-tags">
                          <span class="owner-label">模块名称</span>
                          <div class="owner-list">
                            <i v-for="module in item.moduleStats" :key="`${item.key}-module-${module.name}`">{{ module.name }} <b>{{ module.count }}</b></i>
                          </div>
                        </div>
                        <div v-if="item.groupStats && item.groupStats.length" class="overview-tags owner-tags">
                          <span class="owner-label">{{ item.groupLabel }}</span>
                          <div class="owner-list">
                            <i v-for="groupItem in item.groupStats" :key="`${item.key}-${groupItem.name}`">{{ groupItem.name }} <b>{{ groupItem.count }}</b></i>
                          </div>
                        </div>
                      </div>
                      <div v-if="item.detailKey && isOverviewDetailExpanded(item.detailKey)" class="overview-inline-detail">
                        <table class="broadcast-table detail-table">
                          <thead>
                            <tr>
                              <th>模块名称</th>
                              <th>Bug ID</th>
                              <th>Bug标题</th>
                              <th>停留天数</th>
                              <th>负责人</th>
                              <th>验证人</th>
                            </tr>
                          </thead>
                          <tbody>
                            <tr v-for="row in getPagedRows(getOverdueSection(item.detailKey))" :key="`${item.detailKey}-${row.id}`">
                              <td>{{ row.labels }}</td>
                              <td class="id-cell">{{ row.bugId }}</td>
                              <td class="subject-cell">{{ row.subject }}</td>
                              <td class="count-cell">{{ row.stayDays }}</td>
                              <td>{{ row.assignee }}</td>
                              <td>{{ row.verifier }}</td>
                            </tr>
                            <tr v-if="!getOverdueSection(item.detailKey).rows.length">
                              <td colspan="6" class="empty-cell">暂无数据</td>
                            </tr>
                          </tbody>
                        </table>
                        <div v-if="getOverdueSection(item.detailKey).total > pageSize" class="detail-pagination">
                          <span class="pagination-total">共 {{ getOverdueSection(item.detailKey).total }} 条，每页 {{ pageSize }} 条</span>
                          <div class="pagination-actions">
                            <a-button
                              size="small"
                              :disabled="getSectionPage(item.detailKey) <= 1"
                              @click="handleSectionPageChange(item.detailKey, getSectionPage(item.detailKey) - 1)"
                            >
                              上一页
                            </a-button>
                            <span class="pagination-page">{{ getSectionPage(item.detailKey) }} / {{ getSectionTotalPages(getOverdueSection(item.detailKey)) }}</span>
                            <a-button
                              size="small"
                              :disabled="getSectionPage(item.detailKey) >= getSectionTotalPages(getOverdueSection(item.detailKey))"
                              @click="handleSectionPageChange(item.detailKey, getSectionPage(item.detailKey) + 1)"
                            >
                              下一页
                            </a-button>
                          </div>
                        </div>
                      </div>
                    </div>
                  </div>
                </div>
              </div>

              <div ref="trendPanel" class="trend-panel">
                <div class="report-section-header">
                  <h3>Bug趋势</h3>
                </div>

                <div class="trend-content">
                  <div class="trend-toolbar">
                    <div class="trend-summary">
                      <div class="trend-stat created">
                        <span class="trend-stat-label">新增</span>
                        <strong>{{ trendTotals.created }}</strong>
                        <span>个</span>
                      </div>
                      <div class="trend-stat fixed">
                        <span class="trend-stat-label">已修复</span>
                        <strong>{{ trendTotals.fixed }}</strong>
                        <span>个</span>
                      </div>
                      <div class="trend-stat legacy">
                        <span class="trend-stat-label">待修复总Bug</span>
                        <strong>{{ trendTotals.legacy }}</strong>
                        <span>个</span>
                      </div>
                    </div>
                    <div class="chart-legend">
                      <span><i class="legend-dot red" />新增</span>
                      <span><i class="legend-dot green" />已修复</span>
                      <span><i class="legend-line blue" />待修复总Bug</span>
                    </div>
                  </div>
                  <div class="chart-wrap" @mouseleave="hideTrendTooltip">
                    <svg :viewBox="`0 0 ${chartWidth} ${chartHeight}`" preserveAspectRatio="none">
                      <g class="grid-lines">
                        <line
                          v-for="tick in yAxisTicks"
                          :key="tick.value"
                          x1="54"
                          :y1="tick.y"
                          :x2="chartWidth - 24"
                          :y2="tick.y"
                        />
                      </g>
                      <g class="axis-texts">
                        <text
                          v-for="tick in yAxisTicks"
                          :key="`label-${tick.value}`"
                          x="44"
                          :y="tick.y + 4"
                        >{{ tick.value }}</text>
                      </g>
                      <line class="base-axis" x1="54" :y1="chartBaseY" :x2="chartWidth - 24" :y2="chartBaseY" />
                      <path class="area-path" :d="legacyAreaPath" />
                      <polyline class="legacy-line" :points="legacyLinePoints" />
                      <g v-for="point in chartPoints" :key="point.date">
                        <rect
                          class="created-bar"
                          :x="point.x - point.barWidth - 1"
                          :y="point.createdY"
                          :width="point.barWidth"
                          :height="point.createdHeight"
                          rx="1.5"
                        />
                        <rect
                          class="fixed-bar"
                          :x="point.x + 1"
                          :y="point.fixedY"
                          :width="point.barWidth"
                          :height="point.fixedHeight"
                          rx="1.5"
                        />
                        <circle class="legacy-dot" :cx="point.x" :cy="point.legacyY" r="3" />
                        <text v-if="point.showLabel" class="axis-label" :x="point.x" :y="chartHeight - 12">{{ point.label }}</text>
                        <rect
                          class="hover-zone"
                          :x="point.hoverX"
                          y="20"
                          :width="point.hoverWidth"
                          :height="chartHeight - 40"
                          @mouseenter="showTrendTooltip(point)"
                          @mousemove="showTrendTooltip(point)"
                        />
                      </g>
                    </svg>
                    <div v-if="activeTrendPoint" class="trend-tooltip" :style="trendTooltipStyle">
                      <div class="tooltip-date">{{ activeTrendPoint.label }}</div>
                      <div><i class="legend-dot red" />新增：{{ activeTrendPoint.created }}</div>
                      <div><i class="legend-dot green" />修复：{{ activeTrendPoint.fixed }}</div>
                      <div><i class="legend-dot blue" />待修复总Bug：{{ activeTrendPoint.legacy }}</div>
                    </div>
                  </div>
                </div>
              </div>

            </div>
            <a-empty v-else description="选择项目并点击生成播报后展示内容" />
          </div>
        </a-card>
      </a-col>
    </a-row>
  </div>
</template>

<script>
import moment from 'moment'
import { postAction } from '@/api/manage'

const ORGANIZATION_ID = '64bde4dca0c93ee744686327'
const API_BASE_URL = `${window.location.protocol}//${window.location.hostname}:8080/qaPlatform/aliyun`
const YUNXIAO_TOKEN = 'pt-xKIboN2cltp2ICeM34Ct9bZZ_677a328a-167e-46fd-b90f-c590a30b3d28'

export default {
  name: 'BugBroadcast',
  data () {
    const today = moment()
    return {
      projectLoading: false,
      generating: false,
      projects: [],
      selectedProjectId: undefined,
      selectedProjectName: '',
      selectedProjectSpaceId: undefined,
      dateRange: [today, today],
      dateShortcutRanges: {
        今天: [moment(), moment()],
        昨天: [moment().subtract(1, 'days'), moment().subtract(1, 'days')],
        最近7天: [moment().subtract(6, 'days'), moment()],
        最近14天: [moment().subtract(13, 'days'), moment()]
      },
      reportText: '',
      chartWidth: 720,
      chartHeight: 230,
      bugGroups: {
        created: [],
        repaired: [],
        fixed: [],
        closed: [],
        pending: [],
        unresolvedOverdue: [],
        unverifiedOverdue: [],
        oxiPending: [],
        oxiFixed: [],
        oxiUnresolvedOverdue: [],
        oxiUnverifiedOverdue: []
      },
      trendData: [],
      detailPageMap: {},
      expandedDetailMap: {},
      pageSize: 10,
      activeTrendPoint: null
    }
  },

  computed: {
    formattedDateRange () {
      const start = this.dateRange[0].format('YYYY.M.D')
      const end = this.dateRange[1].format('YYYY.M.D')
      return this.dateRange[0].isSame(this.dateRange[1], 'day') ? start : `${start} - ${end}`
    },

    overviewSections () {
      return [
        this.buildOverviewSection('created', '新增Bug', this.bugGroups.created),
        this.buildOverviewSection('repaired', '修复Bug', this.bugGroups.repaired),
        this.buildOverviewSection('closed', '关闭Bug', this.bugGroups.closed),
        this.buildOverviewSection('fixed', '待验证Bug', this.bugGroups.fixed),
        this.buildOverviewSection('pending', '待修复Bug', this.bugGroups.pending)
      ]
    },

    overviewGroups () {
      return [
        {
          key: 'nonOxi',
          title: '非线上oxi环境Bug统计',
          items: [
            { key: 'created', label: `${this.periodTitlePrefix()}新增Bug`, count: this.bugGroups.created.length, highPriorityCount: this.countHighPriority(this.bugGroups.created), groupLabel: '负责人', groupStats: this.groupByUser(this.bugGroups.created, 'assignedTo'), moduleStats: this.groupByLabelCount(this.bugGroups.created) },
            {
              key: 'repaired',
              label: `${this.periodTitlePrefix()}已修复Bug`,
              count: this.bugGroups.repaired.length,
              highPriorityCount: this.countHighPriority(this.bugGroups.repaired),
              childStats: [
                { key: 'fixed', label: '待验证', count: this.bugGroups.fixed.length, groupLabel: '验证人', groupStats: this.groupByUser(this.bugGroups.fixed, 'verifier'), moduleStats: this.groupByLabelCount(this.bugGroups.fixed) },
                { key: 'closed', label: '已关闭', count: this.bugGroups.closed.length, groupLabel: '验证人', groupStats: this.groupByUser(this.bugGroups.closed, 'verifier'), moduleStats: this.groupByLabelCount(this.bugGroups.closed) }
              ]
            },
            { key: 'pending', label: '待修复Bug', count: this.bugGroups.pending.length, highPriorityCount: this.countHighPriority(this.bugGroups.pending), groupLabel: '负责人', groupStats: this.groupByUser(this.bugGroups.pending, 'assignedTo'), moduleStats: this.groupByLabelCount(this.bugGroups.pending) },
            { key: 'unresolvedOverdue', label: '超过3天未解决的Bug', count: this.bugGroups.unresolvedOverdue.length, highPriorityCount: this.countHighPriority(this.bugGroups.unresolvedOverdue), detailKey: 'unresolved', groupLabel: '负责人', groupStats: this.groupByUser(this.bugGroups.unresolvedOverdue, 'assignedTo'), moduleStats: this.groupByLabelCount(this.bugGroups.unresolvedOverdue) },
            { key: 'unverifiedOverdue', label: '超过3天未验证的Bug', count: this.bugGroups.unverifiedOverdue.length, highPriorityCount: this.countHighPriority(this.bugGroups.unverifiedOverdue), detailKey: 'unverified', groupLabel: '验证人', groupStats: this.groupByUser(this.bugGroups.unverifiedOverdue, 'verifier'), moduleStats: this.groupByLabelCount(this.bugGroups.unverifiedOverdue) }
          ]
        },
        {
          key: 'oxi',
          title: '线上oxi环境Bug统计',
          items: [
            { key: 'oxiPending', label: '待修复Bug', count: this.bugGroups.oxiPending.length, highPriorityCount: this.countHighPriority(this.bugGroups.oxiPending), groupLabel: '负责人', groupStats: this.groupByUser(this.bugGroups.oxiPending, 'assignedTo'), moduleStats: this.groupByLabelCount(this.bugGroups.oxiPending) },
            { key: 'oxiFixed', label: '待验证Bug', count: this.bugGroups.oxiFixed.length, highPriorityCount: this.countHighPriority(this.bugGroups.oxiFixed), groupLabel: '负责人', groupStats: this.groupByUser(this.bugGroups.oxiFixed, 'assignedTo'), moduleStats: this.groupByLabelCount(this.bugGroups.oxiFixed) },
            { key: 'oxiUnresolvedOverdue', label: '超过3天未解决的Bug', count: this.bugGroups.oxiUnresolvedOverdue.length, highPriorityCount: this.countHighPriority(this.bugGroups.oxiUnresolvedOverdue), detailKey: 'oxiUnresolved', groupLabel: '负责人', groupStats: this.groupByUser(this.bugGroups.oxiUnresolvedOverdue, 'assignedTo'), moduleStats: this.groupByLabelCount(this.bugGroups.oxiUnresolvedOverdue) },
            { key: 'oxiUnverifiedOverdue', label: '超过3天未验证的Bug', count: this.bugGroups.oxiUnverifiedOverdue.length, highPriorityCount: this.countHighPriority(this.bugGroups.oxiUnverifiedOverdue), detailKey: 'oxiUnverified', groupLabel: '验证人', groupStats: this.groupByUser(this.bugGroups.oxiUnverifiedOverdue, 'verifier'), moduleStats: this.groupByLabelCount(this.bugGroups.oxiUnverifiedOverdue) }
          ]
        }
      ]
    },

    overdueSections () {
      return [
        {
          key: 'unresolved',
          title: '超过3天未解决Bug',
          total: this.bugGroups.unresolvedOverdue.length,
          groupLabel: '负责人',
          groupStats: this.groupByUser(this.bugGroups.unresolvedOverdue, 'assignedTo'),
          rows: this.buildDetailRows(this.bugGroups.unresolvedOverdue, 'gmtCreate')
        },
        {
          key: 'unverified',
          title: '超过3天未验证Bug',
          total: this.bugGroups.unverifiedOverdue.length,
          groupLabel: '验证人',
          groupStats: this.groupByUser(this.bugGroups.unverifiedOverdue, 'verifier'),
          rows: this.buildDetailRows(this.bugGroups.unverifiedOverdue, 'gmtModified')
        },
        {
          key: 'oxiUnresolved',
          title: '超过3天未解决Bug',
          total: this.bugGroups.oxiUnresolvedOverdue.length,
          groupLabel: '负责人',
          groupStats: this.groupByUser(this.bugGroups.oxiUnresolvedOverdue, 'assignedTo'),
          rows: this.buildDetailRows(this.bugGroups.oxiUnresolvedOverdue, 'gmtCreate')
        },
        {
          key: 'oxiUnverified',
          title: '超过3天未验证Bug',
          total: this.bugGroups.oxiUnverifiedOverdue.length,
          groupLabel: '验证人',
          groupStats: this.groupByUser(this.bugGroups.oxiUnverifiedOverdue, 'verifier'),
          rows: this.buildDetailRows(this.bugGroups.oxiUnverifiedOverdue, 'gmtModified')
        }
      ]
    },

    overdueTotal () {
      return this.bugGroups.unresolvedOverdue.length + this.bugGroups.unverifiedOverdue.length
    },

    trendTotals () {
      const totals = this.trendData.reduce((total, item) => {
        total.created += item.created
        total.fixed += item.fixed
        total.legacy = item.legacy
        return total
      }, { created: 0, fixed: 0, legacy: 0 })
      return totals
    },

    chartTopY () {
      return 28
    },

    chartBaseY () {
      return 152
    },

    chartBottomY () {
      return 186
    },

    maxChartValue () {
      const values = this.trendData.reduce((list, item) => list.concat([item.created, item.fixed, item.legacy]), [])
      const maxValue = Math.max(10, ...values)
      return Math.ceil(maxValue / 10) * 10
    },

    yAxisTicks () {
      const step = Math.max(1, Math.ceil(this.maxChartValue / 5))
      return [5, 4, 3, 2, 1, 0].map(index => {
        const value = index === 0 ? 0 : step * index
        return {
          value,
          y: this.scaleLine(value)
        }
      })
    },

    xLabelStep () {
      const count = this.trendData.length
      if (count <= 10) {
        return 1
      }
      return Math.ceil(count / 6)
    },

    trendTooltipStyle () {
      if (!this.activeTrendPoint) {
        return {}
      }
      const leftPercent = (this.activeTrendPoint.x / this.chartWidth) * 100
      return {
        left: `calc(${leftPercent}% - 66px)`
      }
    },

    chartPoints () {
      if (!this.trendData.length) {
        return []
      }
      const left = 68
      const right = this.chartWidth - 30
      const step = this.trendData.length === 1 ? 0 : (right - left) / (this.trendData.length - 1)
      const barWidth = this.trendData.length > 20 ? 5 : 8
      return this.trendData.map((item, index) => {
        const x = left + step * index
        const createdHeight = this.scaleBar(item.created, 'up')
        const fixedHeight = this.scaleBar(item.fixed, 'down')
        return {
          ...item,
          x,
          barWidth,
          label: moment(item.date).format('M-D'),
          showLabel: index === 0 || index === this.trendData.length - 1 || index % this.xLabelStep === 0,
          legacyY: this.scaleLine(item.legacy),
          createdY: this.chartBaseY - createdHeight,
          createdHeight,
          fixedY: this.chartBaseY + 1,
          fixedHeight,
          hoverX: Math.max(54, x - step / 2),
          hoverWidth: this.trendData.length === 1 ? right - left : Math.max(10, step)
        }
      })
    },

    legacyLinePoints () {
      return this.chartPoints.map(point => `${point.x},${point.legacyY}`).join(' ')
    },

    legacyAreaPath () {
      if (!this.chartPoints.length) {
        return ''
      }
      const baseY = this.chartBaseY
      const first = this.chartPoints[0]
      const last = this.chartPoints[this.chartPoints.length - 1]
      return `M ${first.x} ${baseY} L ${this.legacyLinePoints} L ${last.x} ${baseY} Z`
    }
  },

  mounted () {
    document.body.classList.add('bug-broadcast-page-active')
    this.fetchProjects()
  },

  beforeDestroy () {
    document.body.classList.remove('bug-broadcast-page-active')
  },

  methods: {
    getYunxiaoToken () {
      return localStorage.getItem('yunxiaoToken') || localStorage.getItem('YUNXIAO_TOKEN') || YUNXIAO_TOKEN
    },

    getDatePickerContainer () {
      return document.body
    },

    async fetchProjects () {
      if (this.projectLoading || this.projects.length) {
        return
      }
      this.projectLoading = true
      try {
        const response = await postAction(`${API_BASE_URL}/searchProjectByAliyun`, {
          organizationId: ORGANIZATION_ID,
          token: this.getYunxiaoToken(),
          page: 1,
          perPage: 100
        })
        if (response.status === 'SUCCESS') {
          this.projects = (response.data || []).map(item => ({
            id: item.id,
            name: item.name,
            spaceId: item.id
          }))
          this.setDefaultProject()
        } else {
          throw new Error(response.data || response.msg || '项目列表获取失败')
        }
      } catch (error) {
        this.$message.error(error.message || '项目列表获取失败')
      } finally {
        this.projectLoading = false
      }
    },

    handleExportMenuClick ({ key }) {
      this.exportReportImage(key)
    },

    async exportReportImage (type) {
      if (!this.reportText || !this.$refs.reportPanel) {
        this.$message.warning('暂无可导出的播报内容')
        return
      }
      try {
        const dataUrl = await this.createReportImage(type)
        const link = document.createElement('a')
        link.href = dataUrl
        const suffixMap = {
          oxiOverview: '线上oxi环境Bug统计',
          nonOxiOverview: '非线上oxi环境Bug统计',
          trend: 'Bug趋势图',
          overview: '概览',
          detail: '明细'
        }
        const suffix = suffixMap[type] || '播报'
        link.download = `云效Bug情况播报_${suffix}_${this.dateRange[0].format('YYYYMMDD')}_${this.dateRange[1].format('YYYYMMDD')}.png`
        link.click()
        this.$message.success('图片导出成功')
      } catch (error) {
        this.$message.error('图片导出失败')
      }
    },

    createReportImage (type) {
      return new Promise((resolve, reject) => {
        const source = this.createExportSource(type)
        const sandbox = document.createElement('div')
        sandbox.style.position = 'fixed'
        sandbox.style.left = '-10000px'
        sandbox.style.top = '0'
        sandbox.style.zIndex = '-1'
        sandbox.style.background = '#fbfdff'
        sandbox.appendChild(source)
        document.body.appendChild(sandbox)

        this.$nextTick(() => {
          window.requestAnimationFrame(() => {
            const targetWidthMap = {
              oxiOverview: 1440,
              nonOxiOverview: 1440,
              overview: 1440,
              trend: 1440,
              detail: 1440
            }
            const targetWidth = targetWidthMap[type] || 1180
            source.style.width = `${targetWidth}px`
            source.classList.add(`export-${type}`)
            const tableWidths = Array.from(source.querySelectorAll('table')).map(table => Math.min(table.scrollWidth, targetWidth))
            const rect = source.getBoundingClientRect()
            const exportPadding = type === 'trend' ? 96 : 120
            const minHeight = type === 'trend' ? 620 : 360
            const width = Math.ceil(Math.max(rect.width, ...tableWidths, targetWidth) + exportPadding)
            const height = Math.ceil(Math.max(source.scrollHeight, rect.height, minHeight) + exportPadding)
            source.style.minHeight = `${height - exportPadding}px`
            source.style.background = '#fbfdff'
            const html = `
              <style>${this.getExportStyles()}</style>
              <div xmlns="http://www.w3.org/1999/xhtml" class="export-root">${source.outerHTML}</div>`
            const svg = `<svg xmlns="http://www.w3.org/2000/svg" width="${width}" height="${height}"><foreignObject width="100%" height="100%">${html}</foreignObject></svg>`
            const image = new Image()
            image.onload = () => {
              const canvas = document.createElement('canvas')
              canvas.width = width * 2
              canvas.height = height * 2
              const ctx = canvas.getContext('2d')
              ctx.fillStyle = '#fbfdff'
              ctx.fillRect(0, 0, canvas.width, canvas.height)
              ctx.scale(2, 2)
              ctx.drawImage(image, 0, 0)
              document.body.removeChild(sandbox)
              resolve(canvas.toDataURL('image/png'))
            }
            image.onerror = error => {
              document.body.removeChild(sandbox)
              reject(error)
            }
            image.src = `data:image/svg+xml;charset=utf-8,${encodeURIComponent(svg)}`
          })
        })
      })
    },

    createExportSource (type) {
      const wrapper = document.createElement('div')
      if (type === 'detail') {
        Array.from(this.$refs.reportPanel.querySelectorAll('.report-section')).forEach(node => wrapper.appendChild(node.cloneNode(true)))
      } else if (type === 'trend') {
        const trendPanel = this.$refs.trendPanel.cloneNode(true)
        this.prepareExportTrendPanel(trendPanel)
        wrapper.appendChild(trendPanel)
      } else if (type === 'oxiOverview' || type === 'nonOxiOverview') {
        const groupKey = type === 'oxiOverview' ? 'oxi' : 'nonOxi'
        const overviewGroup = this.$refs.overviewPanel.querySelector(`[data-group-key="${groupKey}"]`)
        if (overviewGroup) {
          wrapper.appendChild(overviewGroup.cloneNode(true))
        }
      } else if (type === 'overview') {
        wrapper.appendChild(this.$refs.overviewPanel.cloneNode(true))
      }
      return wrapper
    },

    prepareExportTrendPanel (trendPanel) {
      const svg = trendPanel.querySelector('svg')
      if (!svg) {
        return
      }
      svg.setAttribute('xmlns', 'http://www.w3.org/2000/svg')
      svg.setAttribute('width', '1280')
      svg.setAttribute('height', '320')
      svg.setAttribute('preserveAspectRatio', 'none')
      svg.style.width = '100%'
      svg.style.height = '320px'
      svg.querySelectorAll('.grid-lines line').forEach(item => item.setAttribute('style', 'stroke:#eef2f7;stroke-width:1'))
      svg.querySelectorAll('.axis-texts text').forEach(item => item.setAttribute('style', 'fill:#7b8797;font-size:12px;text-anchor:end'))
      svg.querySelectorAll('.base-axis').forEach(item => item.setAttribute('style', 'stroke:#d8dee8;stroke-width:1.4'))
      svg.querySelectorAll('.area-path').forEach(item => item.setAttribute('style', 'fill:rgba(107,184,255,.14)'))
      svg.querySelectorAll('.legacy-line').forEach(item => item.setAttribute('style', 'fill:none;stroke:#6bb8ff;stroke-linecap:round;stroke-linejoin:round;stroke-width:3'))
      svg.querySelectorAll('.legacy-dot').forEach(item => item.setAttribute('style', 'fill:#fff;stroke:#6bb8ff;stroke-width:2'))
      svg.querySelectorAll('.created-bar').forEach(item => item.setAttribute('style', 'fill:#f05a4f'))
      svg.querySelectorAll('.fixed-bar').forEach(item => item.setAttribute('style', 'fill:#23b26d'))
      svg.querySelectorAll('.hover-zone').forEach(item => item.setAttribute('style', 'fill:transparent'))
      svg.querySelectorAll('.axis-label').forEach(item => item.setAttribute('style', 'fill:#475569;font-size:12px;font-weight:700;text-anchor:middle'))
    },

    getExportStyles () {
      return `
        *{box-sizing:border-box}body{margin:0}.export-root{padding:24px;background:#fbfdff;color:#223047;font-family:Arial,'Microsoft YaHei',sans-serif}.export-title{margin:0 0 18px;color:#182235;font-size:22px;font-weight:800}.trend-panel{margin-bottom:18px;overflow:hidden;border:1px solid #e4ebf5;border-radius:16px;background:#fff;box-shadow:0 8px 20px rgba(39,65,96,.04)}.trend-content{padding:16px 20px 28px}.section-title{display:flex;align-items:flex-start;justify-content:space-between;margin-bottom:12px}.section-title h2{margin:0;color:#182235;font-size:20px;font-weight:700}.trend-summary{display:flex;gap:18px;margin-bottom:10px;color:#667085;font-size:13px}.chart-legend{display:flex;flex-wrap:nowrap;justify-content:flex-end;gap:14px;margin-bottom:8px;color:#445166;font-size:12px;white-space:nowrap}.legend-dot,.legend-line{display:inline-block;margin-right:5px;vertical-align:middle}.legend-dot{width:8px;height:8px;border-radius:2px}.legend-line{width:16px;height:3px;border-radius:999px}.red{background:#f05a4f}.green{background:#23b26d}.blue{background:#6bb8ff}.chart-wrap{position:relative;height:380px;padding:12px 16px 28px;border:1px solid #edf2f8;border-radius:16px;background:#fff}.chart-wrap svg{width:100%;height:320px;display:block;overflow:visible}.grid-lines line{stroke:#eef2f7;stroke-width:1}.axis-texts text{fill:#7b8797;font-size:12px;text-anchor:end}.base-axis{stroke:#d8dee8;stroke-width:1.4}.area-path{fill:rgba(107,184,255,.14)}.legacy-line{fill:none;stroke:#6bb8ff;stroke-linecap:round;stroke-linejoin:round;stroke-width:3}.legacy-dot{fill:#fff;stroke:#6bb8ff;stroke-width:2}.created-bar{fill:#f05a4f}.fixed-bar{fill:#23b26d}.hover-zone{fill:transparent}.axis-label{fill:#475569;font-size:12px;font-weight:700;text-anchor:middle}.trend-tooltip{display:none}.report-head{display:block;margin-bottom:18px;overflow:hidden;border:1px solid #dce8f7;border-radius:16px;background:linear-gradient(135deg,#f5faff 0%,#edf7ff 100%)}.report-section-header{padding:12px 18px;border-bottom:1px solid #e2edf9;background:linear-gradient(135deg,#eaf4ff 0%,#f5faff 100%)}.report-section-header h3{margin:0;color:#17345f;font-size:18px;font-weight:800}.report-head h3{margin:0;color:#17345f;font-size:19px;font-weight:800}.overview-list{padding:0;color:#20304a;font-size:14px}.overview-group{overflow:hidden;margin-bottom:18px;border:1px solid #e4ebf5;border-radius:16px;background:#fff;box-shadow:0 8px 20px rgba(39,65,96,.04)}.overview-group+.overview-group{margin-top:0}.overview-group-title{padding:12px 18px;color:#17345f;font-size:18px;font-weight:800;border-bottom:1px solid #e2edf9;background:linear-gradient(135deg,#eaf4ff 0%,#f5faff 100%)}.overview-item{display:grid;grid-template-columns:minmax(220px,280px) minmax(0,1fr);gap:12px 18px;align-items:flex-start;margin:12px 16px;padding:14px 16px;border:1px solid rgba(213,225,241,.88);border-radius:14px;background:linear-gradient(135deg,#fff 0%,#f9fcff 100%);box-shadow:0 8px 20px rgba(39,65,96,.04)}.overview-metric{min-width:0}.overview-line{display:flex;gap:6px;align-items:baseline;color:#20304a;font-weight:700}.overview-name{flex:0 0 auto;white-space:nowrap;text-align:left}.overview-name::after{content:'：'}.overview-count{color:#0f68b1;font-size:20px;font-weight:800;letter-spacing:.2px}.overview-priority{display:inline-flex;align-items:center;width:fit-content;margin-top:8px;padding:3px 10px;color:#fa8c16;font-size:12px;font-weight:700;line-height:1.6;border:1px solid rgba(250,140,22,.22);border-radius:999px;background:rgba(255,241,230,.92)}.report-head .overview-priority{color:#fa8c16}.overview-sub-item{margin-left:42px;background:linear-gradient(135deg,#fbfdff 0%,#f7faff 100%);box-shadow:none}.overview-sub-line{color:#7a8798;font-size:13px;font-weight:600;line-height:1.8}.overview-sub-line .overview-name{flex-basis:auto;color:#7a8798;font-weight:600;text-align:left}.overview-sub-line .overview-count{color:#6b7a90;font-size:15px}.overview-sub-line .overview-priority{color:#a65f2b;font-size:12px;font-weight:600;background:rgba(255,247,230,.86)}.overview-detail{display:flex;flex-direction:column;gap:8px;min-width:0}.overview-inline-detail{grid-column:1/-1;overflow:visible;margin-top:4px;border:1px solid #dce8f7;border-radius:14px;background:#fff;box-shadow:inset 0 1px 0 rgba(255,255,255,.8)}.overview-inline-detail .broadcast-table{width:100%;min-width:100%}.overview-inline-detail .detail-pagination{border-radius:0 0 14px 14px}.overview-child-list{display:grid;grid-template-columns:minmax(0,1fr);gap:10px}.overview-child-card{min-width:0;padding:10px 12px;border:1px solid #edf2f8;border-radius:12px;background:linear-gradient(135deg,#fff 0%,#f8fbff 100%)}.overview-child-line{display:flex;align-items:baseline;justify-content:flex-start;gap:8px;color:#53657d;font-size:13px;font-weight:700}.overview-child-line strong{flex:0 0 auto;color:#0f68b1;font-size:16px;font-weight:800}.child-tags{margin-top:8px;padding:8px 10px}.overview-tags{display:grid;grid-template-columns:82px minmax(0,1fr);gap:10px;align-items:flex-start;margin:0;padding:10px 12px;border:1px solid #edf2f8;border-radius:12px;background:#f8fbff}.overview-tags.owner-tags{background:#fff}.overview-tags.module-tags{background:#f8fbff}.overview-tags .owner-label{display:inline-flex;align-items:center;justify-content:center;min-height:24px;padding:2px 8px;color:#607089;font-size:12px;font-weight:700;line-height:1.6;text-align:center;white-space:nowrap;border-radius:999px;background:#edf4ff}.overview-tags .owner-label::after{content:''}.owner-list{display:flex;flex:1;flex-wrap:wrap;gap:6px 8px;min-width:0;color:#607089;font-size:12px;font-weight:600;line-height:1.8}.owner-list i{display:inline-flex;align-items:center;gap:4px;min-height:24px;padding:2px 8px;font-style:normal;border:1px solid #e4ebf5;border-radius:999px;background:#fff}.owner-list b{color:#0f68b1;font-size:13px;font-weight:800}.owner-tags .owner-label{color:#5f6f86;background:#f1f5f9}.owner-tags .owner-list i{color:#53657d;border-color:#dce8f7;background:#f7fbff}.module-tags .owner-label{color:#607089;background:#edf4ff}.module-tags .owner-list i{color:#607089;border-color:#e4ebf5;background:#fff}.report-section{margin-top:18px}.report-section h4{margin:0 0 12px;color:#172033;font-size:16px;font-weight:800}.report-table-block{margin-bottom:16px;border:1px solid #e4ebf5;border-radius:14px;background:#fff;overflow:hidden}.table-title{display:flex;align-items:center;justify-content:space-between;gap:12px;padding:12px 14px;border-bottom:1px solid #edf2f8;background:#f8fbff}.table-title strong{color:#20304a}.table-title span{color:#f05a4f;font-size:12px;font-weight:800}.broadcast-table{width:100%;border-collapse:collapse;table-layout:auto;font-size:13px}.broadcast-table th{padding:11px 12px;color:#324860;font-weight:800;text-align:center;white-space:nowrap;background:#f3f7fc}.broadcast-table td{padding:11px 12px;color:#344054;border-top:1px solid #edf2f8;text-align:center;vertical-align:middle;white-space:nowrap}.count-cell,.id-cell{color:#0f68b1;font-weight:800}.subject-cell{min-width:420px;line-height:1.6;text-align:left!important}.empty-cell{color:#8b98aa!important;text-align:center}.trend-toolbar{display:flex;align-items:center;justify-content:space-between;gap:16px;margin-bottom:12px}.trend-summary{display:flex;flex-wrap:wrap;gap:10px;color:#667085;font-size:13px}.trend-stat{display:inline-flex;align-items:baseline;gap:4px;min-width:126px;padding:10px 14px;border:1px solid #e4ebf5;border-radius:14px;background:#fff;box-shadow:0 8px 18px rgba(39,65,96,.05)}.trend-stat-label{margin-right:4px;color:#607089;font-size:12px;font-weight:700}.trend-stat strong{color:#0f68b1;font-size:22px;font-weight:800}.trend-stat.created strong{color:#f05a4f}.trend-stat.fixed strong{color:#23b26d}.trend-stat.legacy strong{color:#6bb8ff}.report-section{margin-top:18px;overflow:hidden;border:1px solid #e4ebf5;border-radius:16px;background:linear-gradient(180deg,#fff 0%,#f8fbff 100%);box-shadow:0 8px 20px rgba(39,65,96,.04)}.report-section>.report-section-header{overflow:hidden;border-bottom:1px solid #e2edf9}.report-table-block{margin:16px;overflow-x:auto;overflow-y:hidden;border:1px solid #e4ebf5;border-radius:14px;background:#fff;box-shadow:0 8px 20px rgba(39,65,96,.05)}.table-title{display:flex;align-items:center;justify-content:space-between;gap:12px;padding:12px 16px;border-bottom:1px solid #edf2f8;background:linear-gradient(135deg,#fbfdff 0%,#f2f7ff 100%)}.table-title strong{display:inline-flex;align-items:center;gap:8px;color:#20304a;font-size:14px}.table-title strong::before{display:inline-block;width:6px;height:16px;content:'';border-radius:999px;background:#1890ff}.broadcast-table tbody tr:nth-child(even) td{background:#fbfdff}.subject-cell{min-width:420px;max-width:720px;white-space:normal!important}.detail-table th,.detail-table td{text-align:left}.detail-table .count-cell,.detail-table .id-cell{text-align:left}`
    },

    setDefaultProject () {
      if (this.selectedProjectId) {
        return
      }
      const defaultProject = this.projects.find(item => item.name === 'D&B')
      if (defaultProject) {
        this.handleProjectChange(defaultProject.id)
      }
    },

    handleProjectChange (value) {
      this.selectedProjectId = value
      const project = this.projects.find(item => item.id === value)
      this.selectedProjectName = project ? project.name : ''
      this.selectedProjectSpaceId = project ? project.spaceId : undefined
    },

    handleDateChange (dates) {
      this.dateRange = dates
    },

    async generateBroadcast () {
      if (!this.selectedProjectId) {
        this.$message.warning('请先选择项目')
        return
      }
      this.generating = true
      try {
        this.activeTrendPoint = null
        this.detailPageMap = {}
        this.expandedDetailMap = {}
        const groups = await this.fetchBugGroups()
        this.bugGroups = groups
        this.trendData = this.buildTrendData(groups)
        this.reportText = this.buildReportText(groups)
        this.$message.success('生成播报成功')
      } catch (error) {
        this.$message.error(error.message || '生成播报失败')
      } finally {
        this.generating = false
      }
    },

    async fetchBugGroups () {
      const trendDates = this.getDateList()
      const [created, repaired, fixed, closed, pending, unresolvedList, unverifiedList, oxiPending, oxiFixed, oxiUnresolvedList, oxiUnverifiedList, trendPendingList] = await Promise.all([
        this.searchBugWorkitems(this.buildCreatedCondition()),
        this.searchBugWorkitems(this.buildRepairedCondition()),
        this.searchBugWorkitems(this.buildFixedCondition()),
        this.searchBugWorkitems(this.buildClosedCondition()),
        this.searchBugWorkitems(this.buildNonOxiPendingCondition()),
        this.searchBugWorkitems(this.buildNonOxiPendingCondition()),
        this.searchBugWorkitems(this.buildNonOxiUnverifiedCondition()),
        this.searchBugWorkitems(this.buildOxiPendingCondition()),
        this.searchBugWorkitems(this.buildOxiFixedCondition()),
        this.searchBugWorkitems(this.buildOxiPendingCondition()),
        this.searchBugWorkitems(this.buildOxiFixedCondition()),
        Promise.all(trendDates.map(date => this.searchBugWorkitems(this.buildPendingTrendCondition(date))))
      ])
      const createdList = this.getBugResultList(created)
      const repairedList = this.getBugResultList(repaired)
      const fixedList = this.getBugResultList(fixed)
      const closedList = this.getBugResultList(closed)
      const pendingList = this.getBugResultList(pending)
      const unresolvedSourceList = this.getBugResultList(unresolvedList)
      const unverifiedSourceList = this.getBugResultList(unverifiedList)
      const oxiPendingList = this.getBugResultList(oxiPending)
      const oxiFixedList = this.getBugResultList(oxiFixed)
      const oxiUnresolvedSourceList = this.getBugResultList(oxiUnresolvedList)
      const oxiUnverifiedSourceList = this.getBugResultList(oxiUnverifiedList)
      const unresolvedOverdue = unresolvedSourceList.filter(item => this.getStayDaysByField(item, 'gmtCreate') > 3)
      const unverifiedOverdue = unverifiedSourceList.filter(item => this.getStayDaysByField(item, 'gmtModified') > 3)
      const oxiUnresolvedOverdue = oxiUnresolvedSourceList.filter(item => this.getStayDaysByField(item, 'gmtCreate') > 3)
      const oxiUnverifiedOverdue = oxiUnverifiedSourceList.filter(item => this.getStayDaysByField(item, 'gmtModified') > 3)
      const trendPendingCounts = trendDates.reduce((map, date, index) => {
        map[date] = this.getBugResultTotal(trendPendingList[index])
        return map
      }, {})
      return {
        created: createdList,
        repaired: repairedList,
        fixed: fixedList,
        closed: closedList,
        pending: pendingList,
        unresolvedOverdue,
        unverifiedOverdue,
        oxiPending: oxiPendingList,
        oxiFixed: oxiFixedList,
        oxiUnresolvedOverdue,
        oxiUnverifiedOverdue,
        trendPendingCounts
      }
    },

    async searchBugWorkitems (conditions) {
      const response = await postAction(`${API_BASE_URL}/searchBugWorkitemsByAliyun`, {
        organizationId: ORGANIZATION_ID,
        token: this.getYunxiaoToken(),
        spaceId: this.selectedProjectSpaceId,
        conditions,
        page: 1,
        perPage: 200,
        fetchAll: true,
        sort: 'desc'
      })
      if (response.status !== 'SUCCESS') {
        throw new Error(response.data || response.msg || 'Bug数据获取失败')
      }
      return this.normalizeBugResult(response.data)
    },

    normalizeBugResult (data) {
      if (Array.isArray(data)) {
        return {
          list: data,
          totalCount: data.length
        }
      }
      const list = data && Array.isArray(data.list) ? data.list : []
      return {
        list,
        totalCount: data && typeof data.totalCount === 'number' ? data.totalCount : list.length
      }
    },

    getBugResultList (result) {
      return Array.isArray(result) ? result : (result && result.list) || []
    },

    getBugResultTotal (result) {
      return Array.isArray(result) ? result.length : (result && result.totalCount) || 0
    },

    getSectionPage (key) {
      return this.detailPageMap[key] || 1
    },

    getSectionTotalPages (section) {
      return Math.max(1, Math.ceil(section.total / this.pageSize))
    },

    getOverdueSection (key) {
      return this.overdueSections.find(section => section.key === key) || {
        key,
        title: '',
        total: 0,
        rows: []
      }
    },

    isOverviewDetailExpanded (key) {
      return !!this.expandedDetailMap[key]
    },

    toggleOverviewDetail (key) {
      this.$set(this.expandedDetailMap, key, !this.expandedDetailMap[key])
      if (!this.detailPageMap[key]) {
        this.$set(this.detailPageMap, key, 1)
      }
    },

    getPagedRows (section) {
      const page = this.getSectionPage(section.key)
      const start = (page - 1) * this.pageSize
      return section.rows.slice(start, start + this.pageSize)
    },

    handleSectionPageChange (key, page) {
      this.$set(this.detailPageMap, key, page)
    },

    buildCreatedCondition () {
      return this.stringifyConditions([
        this.createDateFilter('gmtCreate')
      ])
    },

    buildRepairedCondition () {
      return this.stringifyConditions([
        this.createStatusFilter(['29', '31', '626216', 'a3b8a981c6846aaba83c05fb', '100085']),
        this.createDateFilter('gmtModified')
      ])
    },

    buildFixedCondition () {
      return this.stringifyConditions([
        this.createStatusFilter(['29', '31', '626216', 'a3b8a981c6846aaba83c05fb']),
        this.createDateFilter('gmtModified')
      ])
    },

    buildClosedCondition () {
      return this.stringifyConditions([
        this.createStatusFilter(['100085']),
        this.createDateFilter('gmtModified')
      ])
    },

    buildPendingCondition () {
      return this.stringifyConditions([
        this.createStatusFilter(['28', '30', '100010'])
      ])
    },

    buildPendingBeforeEndCondition () {
      return this.stringifyConditions([
        this.createStatusFilter(['28', '30', '100010']),
        this.createDateBeforeFilter('gmtCreate', this.dateRange[1].format('YYYY-MM-DD'))
      ])
    },

    buildPendingTrendCondition (date) {
      return this.stringifyConditions([
        this.createStatusFilter(['28', '30', '100010']),
        this.createDateBeforeFilter('gmtCreate', date)
      ])
    },

    buildUnverifiedCondition () {
      return this.stringifyConditions([
        this.createStatusFilter(['29', '31', '626216', 'a3b8a981c6846aaba83c05fb'])
      ])
    },

    buildUnverifiedBeforeEndCondition () {
      return this.stringifyConditions([
        this.createStatusFilter(['29', '31', '626216', 'a3b8a981c6846aaba83c05fb']),
        this.createDateBeforeFilter('gmtModified', this.dateRange[1].format('YYYY-MM-DD'))
      ])
    },

    buildOxiPendingCondition () {
      return this.stringifyConditions([
        this.createStatusFilter(['28', '30', '100010']),
        this.createEnvironmentFilter(['oxi'])
      ])
    },

    buildOxiFixedCondition () {
      return this.stringifyConditions([
        this.createStatusFilter(['29', '31', '626216', 'a3b8a981c6846aaba83c05fb']),
        this.createEnvironmentFilter(['oxi'])
      ])
    },

    buildNonOxiPendingCondition () {
      return this.stringifyConditions([
        this.createStatusFilter(['28', '30', '100010']),
        this.createEnvironmentFilter(['sun', 'test', 'dev', 'EMPTY_VALUE'])
      ])
    },

    buildNonOxiUnverifiedCondition () {
      return this.stringifyConditions([
        this.createStatusFilter(['29', '31', '626216', 'a3b8a981c6846aaba83c05fb']),
        this.createEnvironmentFilter(['sun', 'test', 'dev', 'EMPTY_VALUE'])
      ])
    },

    stringifyConditions (filters) {
      return JSON.stringify({
        conditionGroups: [filters]
      })
    },

    createStatusFilter (value) {
      return {
        fieldIdentifier: 'status',
        operator: 'CONTAINS',
        value,
        toValue: null,
        className: 'status',
        format: 'list'
      }
    },

    createEnvironmentFilter (value) {
      return {
        fieldIdentifier: '343800234986603657da5a1816',
        operator: 'CONTAINS',
        value,
        toValue: null,
        className: 'string',
        format: 'list'
      }
    },

    createDateFilter (fieldIdentifier) {
      return {
        fieldIdentifier,
        operator: 'BETWEEN',
        value: [this.dateRange[0].format('YYYY-MM-DD 00:00:00')],
        toValue: this.dateRange[1].format('YYYY-MM-DD 23:59:59'),
        className: 'dateTime',
        format: 'input'
      }
    },

    createDateBeforeFilter (fieldIdentifier, date) {
      return {
        fieldIdentifier,
        operator: 'BETWEEN',
        value: ['1970-01-01 00:00:00'],
        toValue: `${date} 23:59:59`,
        className: 'dateTime',
        format: 'input'
      }
    },

    buildOverviewSection (key, title, list) {
      return {
        key,
        title,
        total: list.length,
        highPriorityTotal: this.countHighPriority(list),
        rows: this.groupByLabel(list)
      }
    },

    periodTitlePrefix () {
      const start = this.dateRange[0]
      const end = this.dateRange[1]
      if (start.isSame(end, 'day') && start.isSame(moment(), 'day')) {
        return '今日'
      }
      if (start.isSame(end, 'day')) {
        return start.format('M.D')
      }
      return `${start.format('M.DD')}-${end.format('M.DD')}`
    },

    groupByUser (list, field) {
      const map = {}
      list.forEach(item => {
        const name = this.getUserName(item[field])
        map[name] = (map[name] || 0) + 1
      })
      return Object.keys(map).map(name => ({ name, count: map[name] })).sort((a, b) => b.count - a.count)
    },

    groupByLabelCount (list) {
      return this.groupByLabel(list).map(item => ({
        name: item.label,
        count: item.count
      }))
    },

    countHighPriority (list) {
      return list.filter(item => this.isHighPriority(item)).length
    },

    buildReportText (groups) {
      return [
        `【云效Bug情况播报】 ${this.formattedDateRange}`,
        '',
        '一、Bug明细',
        this.renderSimpleSection('新增Bug', groups.created),
        this.renderSimpleSection('修复Bug', groups.repaired),
        this.renderSimpleSection('关闭Bug', groups.closed),
        this.renderSimpleSection('待验证Bug', groups.fixed),
        this.renderSimpleSection('待修复Bug', groups.pending),
        '',
        this.renderDetailSection('超过3天未解决Bug', groups.unresolvedOverdue),
        this.renderDetailSection('超过3天未验证Bug', groups.unverifiedOverdue)
      ].join('\n')
    },

    renderSimpleSection (title, list, countLabel) {
      return [
        `${title}：${list.length}个`,
        `| 标签名称 | ${countLabel} | 优先级 | 负责人 | 验证人 |`,
        '| --- | --- | --- | --- | --- |',
        ...this.groupByLabel(list).map(item => `| ${item.label} | ${item.count} | ${item.priority} | ${item.assignee} | ${item.verifier} |`),
        ''
      ].join('\n')
    },

    buildDetailRows (list, stayField) {
      return list.slice().sort((a, b) => this.getStayDaysByField(b, stayField) - this.getStayDaysByField(a, stayField)).map(item => ({
        id: item.id,
        labels: this.getLabels(item),
        bugId: this.getBugId(item),
        subject: this.getBugSubject(item),
        stayDays: this.getStayDaysByField(item, stayField),
        assignee: this.getUserName(item.assignedTo),
        verifier: this.getUserName(item.verifier)
      }))
    },

    renderDetailSection (title, list) {
      const stayField = title.indexOf('未验证') > -1 ? 'gmtModified' : 'gmtCreate'
      return [
        `${title}：${list.length}个`,
        '| 模块名称 | Bug ID | bug标题 | 停留天数 | 负责人 | 验证人 |',
        '| --- | --- | --- | --- | --- | --- |',
        ...list.slice().sort((a, b) => this.getStayDaysByField(b, stayField) - this.getStayDaysByField(a, stayField)).slice(0, 20).map(item => `| ${this.getLabels(item)} | ${this.getBugId(item)} | ${this.getBugSubject(item)} | ${this.getStayDaysByField(item, stayField)} | ${this.getUserName(item.assignedTo)} | ${this.getUserName(item.verifier)} |`),
        ''
      ].join('\n')
    },

    groupByLabel (list) {
      const map = {}
      list.forEach(item => {
        // 一个 Bug 可能有多个标签，播报按每个 labels[].name 分别聚合数量。
        this.getLabelNames(item).forEach(label => {
          if (!map[label]) {
            map[label] = {
              label,
              count: 0,
              highPriorityCount: 0,
              assignee: this.getUserName(item.assignedTo),
              verifier: this.getUserName(item.verifier)
            }
          }
          map[label].count += 1
          if (this.isHighPriority(item)) {
            map[label].highPriorityCount += 1
          }
        })
      })
      return Object.values(map).sort((a, b) => b.count - a.count)
    },

    buildTrendData (groups) {
      const days = this.getDateList()
      return days.map(date => ({
        date,
        created: this.countByDate(groups.created, date, 'gmtCreate'),
        fixed: this.countByDate(groups.repaired, date, 'gmtModified'),
        legacy: groups.trendPendingCounts && groups.trendPendingCounts[date] !== undefined ? groups.trendPendingCounts[date] : groups.pending.length
      }))
    },

    createEmptyTrend () {
      return this.getDateList().map(date => ({ date, created: 0, fixed: 0, legacy: 0 }))
    },

    getDateList () {
      const dates = []
      const current = this.dateRange[0].clone()
      const end = this.dateRange[1].clone()
      while (current.isSameOrBefore(end, 'day')) {
        dates.push(current.format('YYYY-MM-DD'))
        current.add(1, 'day')
      }
      return dates
    },

    countByDate (list, date, field) {
      return list.filter(item => moment(item[field]).format('YYYY-MM-DD') === date).length
    },

    getLabelNames (item) {
      return item.labels && item.labels.length ? item.labels.map(label => label.name || '-').filter(Boolean) : ['-']
    },

    getLabels (item) {
      return this.getLabelNames(item).join('、')
    },

    getPriority (item) {
      const priority = (item.customFieldValues || []).find(field => field.fieldId === 'priority' || field.fieldName === '优先级')
      return priority && priority.values && priority.values.length ? priority.values[0].displayValue : '-'
    },

    isHighPriority (item) {
      const priority = this.getPriority(item)
      return priority.indexOf('紧急') > -1 || priority.indexOf('高') > -1
    },

    getBugId (item) {
      return item.serialNumber || item.id || '-'
    },

    getBugSubject (item) {
      return item.subject || '-'
    },

    getStatus (item) {
      return item.status ? item.status.displayName || item.status.name || '-' : '-'
    },

    getUserName (user) {
      return user ? user.name || '-' : '-'
    },

    getStayDaysByField (item, field) {
      const time = item[field]
      return time ? Math.max(0, moment().diff(moment(time), 'days')) : 0
    },

    showTrendTooltip (point) {
      this.activeTrendPoint = point
    },

    hideTrendTooltip () {
      this.activeTrendPoint = null
    },

    scaleLine (value) {
      return this.chartBaseY - (value / this.maxChartValue) * (this.chartBaseY - this.chartTopY)
    },

    scaleBar (value, direction) {
      const maxHeight = direction === 'down' ? this.chartBottomY - this.chartBaseY : this.chartBaseY - this.chartTopY
      return Math.max(0, (value / this.maxChartValue) * maxHeight)
    }
  }
}
</script>

<style scoped>
.bug-broadcast-page {
  box-sizing: border-box;
  display: flex;
  flex-direction: column;
  height: calc(100vh - 64px);
  min-height: 0;
  overflow: hidden;
  padding: 24px;
  background:
    radial-gradient(circle at top left, rgba(24, 144, 255, 0.13), transparent 32%),
    linear-gradient(135deg, #f7fbff 0%, #f5f7fb 45%, #fff 100%);
}

.hero-panel {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 20px;
  padding: 28px 32px;
  color: #1f3657;
  border: 1px solid #dbeafe;
  border-radius: 24px;
  background:
    radial-gradient(circle at top right, rgba(35, 166, 161, 0.18), transparent 30%),
    linear-gradient(135deg, #f8fbff 0%, #eef7ff 58%, #f7fffd 100%);
  box-shadow: 0 14px 36px rgba(32, 87, 129, 0.1);
}

.hero-panel h1 {
  margin: 0;
  color: #17345f;
  font-size: 30px;
  font-weight: 700;
}

.hero-desc {
  max-width: 620px;
  margin: 10px 0 0;
  color: #5f6f86;
}

.filter-card,
.report-card,
.trend-card {
  border-radius: 20px;
  box-shadow: 0 16px 40px rgba(28, 48, 78, 0.08);
}

.filter-card {
  flex: 0 0 auto;
  z-index: 20;
}

.filter-row {
  max-width: 980px;
}

.project-filter-col,
.date-filter-col {
  flex: 0 0 300px;
  width: 300px;
  max-width: 300px;
}

.button-filter-col {
  flex: 0 0 140px;
  width: 140px;
  max-width: 140px;
  margin-left: 16px;
}

.export-filter-col {
  flex: 0 0 150px;
  width: 150px;
  max-width: 150px;
  margin-left: 8px;
}

.field-label {
  display: block;
  margin-bottom: 8px;
  color: #334155;
  font-weight: 600;
}

.project-select,
.date-picker {
  width: 100%;
}

.date-picker {
  min-width: 300px;
}

.generate-btn,
.export-btn {
  height: 32px;
  line-height: 30px;
}

.generate-btn {
  position: relative;
  overflow: hidden;
  font-weight: 700;
}

.export-filter-col /deep/ .ant-dropdown-trigger {
  width: 100%;
}

.generate-btn[disabled],
.generate-btn[disabled]:hover,
.generate-btn[disabled]:focus {
  color: #fff;
  border-color: #1890ff;
  background: #1890ff;
  opacity: 1;
}

.generate-btn[disabled]::after,
.generate-btn[disabled]:hover::after,
.generate-btn[disabled]:focus::after {
  position: absolute;
  inset: 0;
  content: '';
  background: rgba(255, 255, 255, 0.48);
}

.content-row {
  flex: 1;
  min-height: 0;
  margin-top: 24px;
  overflow: hidden;
}

.content-row > .ant-col {
  height: 100%;
}

.report-card {
  height: 100%;
  min-height: 0;
  background: transparent;
  box-shadow: none;
}

.report-card /deep/ .ant-card-body {
  display: flex;
  flex-direction: column;
  height: 100%;
  min-height: 0;
  padding: 0;
}

.trend-card {
  min-height: 360px;
}

.section-title {
  display: flex;
  align-items: flex-start;
  justify-content: space-between;
  margin-bottom: 18px;
}

.section-title h2 {
  margin: 0;
  color: #182235;
  font-size: 20px;
  font-weight: 700;
}

.title-date {
  margin-left: 10px;
  color: #5f6f86;
  font-size: 15px;
  font-weight: 700;
}

.report-scroll {
  flex: 1;
  min-height: 0;
  overflow: auto;
  padding: 18px;
  border: 1px solid #e8edf5;
  border-radius: 16px;
  background: #fbfdff;
}

.report-panel {
  color: #223047;
}

.trend-panel {
  margin-bottom: 18px;
  overflow: hidden;
  border: 1px solid #e4ebf5;
  border-radius: 16px;
  background: #fff;
  box-shadow: 0 8px 20px rgba(39, 65, 96, 0.04);
}

.trend-content {
  padding: 18px 20px 16px;
  background:
    radial-gradient(circle at 90% 8%, rgba(107, 184, 255, 0.14), transparent 28%),
    linear-gradient(180deg, #fff 0%, #f8fbff 100%);
}

.report-head {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 12px;
  margin-bottom: 18px;
  overflow: hidden;
  border: 1px solid #dce8f7;
  border-radius: 16px;
  background: linear-gradient(135deg, #f5faff 0%, #edf7ff 100%);
}

.report-section-header {
  padding: 12px 18px;
  border-bottom: 1px solid #e2edf9;
  background: linear-gradient(135deg, #eaf4ff 0%, #f5faff 100%);
}

.report-section-header h3 {
  margin: 0;
  color: #17345f;
  font-size: 18px;
  font-weight: 800;
}

.report-head h3 {
  margin: 0;
  color: #17345f;
  font-size: 18px;
  font-weight: 800;
}

.report-head span {
  color: #1e3a5f;
  font-weight: 700;
}

.overview-head {
  display: block;
}

.overview-list {
  padding: 0;
  color: #20304a;
  font-size: 14px;
}

.overview-group {
  overflow: hidden;
  margin-bottom: 18px;
  border: 1px solid #e4ebf5;
  border-radius: 16px;
  background: #fff;
  box-shadow: 0 8px 20px rgba(39, 65, 96, 0.04);
}

.overview-group + .overview-group {
  margin-top: 0;
}

.overview-group-title {
  padding: 12px 18px;
  color: #17345f;
  font-size: 18px;
  font-weight: 800;
  border-bottom: 1px solid #e2edf9;
  background: linear-gradient(135deg, #eaf4ff 0%, #f5faff 100%);
}

.overview-item {
  display: grid;
  grid-template-columns: minmax(220px, 280px) minmax(0, 1fr);
  gap: 12px 18px;
  align-items: flex-start;
  margin: 12px 16px;
  padding: 14px 16px;
  border: 1px solid rgba(213, 225, 241, 0.88);
  border-radius: 14px;
  background: linear-gradient(135deg, #ffffff 0%, #f9fcff 100%);
  box-shadow: 0 8px 20px rgba(39, 65, 96, 0.04);
}

.overview-metric {
  min-width: 0;
}

.overview-line {
  display: flex;
  flex-wrap: nowrap;
  align-items: baseline;
  justify-content: flex-start;
  gap: 6px;
  color: #20304a;
  font-weight: 700;
}

.overview-name {
  flex: 0 0 auto;
  white-space: nowrap;
  text-align: left;
}

.overview-name::after {
  content: '：';
}

.overview-count {
  flex: 0 0 auto;
  color: #0f68b1;
  font-size: 20px;
  font-weight: 800;
  letter-spacing: 0.2px;
  white-space: nowrap;
}

.overview-priority {
  display: inline-flex;
  align-items: center;
  width: fit-content;
  margin-top: 8px;
  padding: 3px 10px;
  color: #fa8c16;
  font-size: 12px;
  font-weight: 700;
  line-height: 1.6;
  border: 1px solid rgba(250, 140, 22, 0.22);
  border-radius: 999px;
  background: rgba(255, 241, 230, 0.92);
}

.overview-detail-btn {
  flex: 0 0 auto;
  width: fit-content;
  margin-left: 8px;
  font-size: 12px;
  font-weight: 700;
  border-radius: 999px;
}

.overview-inline-detail {
  grid-column: 1 / -1;
  overflow: hidden;
  margin-top: 4px;
  border: 1px solid #dce8f7;
  border-radius: 14px;
  background: #fff;
  box-shadow: inset 0 1px 0 rgba(255, 255, 255, 0.8);
}

.inline-detail-title {
  margin: 0;
}

.overview-inline-detail .broadcast-table {
  width: 100%;
}

.overview-inline-detail .detail-pagination {
  border-radius: 0 0 14px 14px;
}

.report-head .overview-priority {
  color: #fa8c16;
}

.overview-sub-item {
  margin-left: 42px;
  background: linear-gradient(135deg, #fbfdff 0%, #f7faff 100%);
  box-shadow: none;
}

.overview-sub-line {
  color: #7a8798;
  font-size: 13px;
  font-weight: 600;
  line-height: 1.8;
}

.overview-sub-line .overview-name {
  flex-basis: auto;
  color: #7a8798;
  font-weight: 600;
  text-align: left;
}

.overview-sub-line .overview-count {
  color: #6b7a90;
  font-size: 15px;
}

.overview-sub-line .overview-priority {
  color: #a65f2b;
  font-size: 12px;
  font-weight: 600;
  background: rgba(255, 247, 230, 0.86);
}

.overview-detail {
  display: flex;
  flex-direction: column;
  gap: 8px;
  min-width: 0;
}

.overview-child-list {
  display: grid;
  grid-template-columns: minmax(0, 1fr);
  gap: 10px;
}

.overview-child-card {
  min-width: 0;
  padding: 10px 12px;
  border: 1px solid #edf2f8;
  border-radius: 12px;
  background: linear-gradient(135deg, #ffffff 0%, #f8fbff 100%);
}

.overview-child-line {
  display: flex;
  align-items: baseline;
  justify-content: flex-start;
  gap: 8px;
  color: #53657d;
  font-size: 13px;
  font-weight: 700;
}

.overview-child-line strong {
  flex: 0 0 auto;
  color: #0f68b1;
  font-size: 16px;
  font-weight: 800;
}

.child-tags {
  margin-top: 8px;
  padding: 8px 10px;
}

.overview-tags {
  display: grid;
  grid-template-columns: 82px minmax(0, 1fr);
  gap: 10px;
  align-items: flex-start;
  margin: 0;
  padding: 10px 12px;
  border: 1px solid #edf2f8;
  border-radius: 12px;
  background: #f8fbff;
}

.overview-tags.owner-tags {
  background: #fff;
}

.overview-tags.module-tags {
  background: #f8fbff;
}

.overview-tags .owner-label {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  min-height: 24px;
  padding: 2px 8px;
  color: #607089;
  font-size: 12px;
  font-weight: 700;
  line-height: 1.6;
  text-align: center;
  white-space: nowrap;
  border-radius: 999px;
  background: #edf4ff;
}

.overview-tags .owner-label::after {
  content: '';
}

.owner-list {
  display: flex;
  flex: 1;
  flex-wrap: wrap;
  gap: 6px 8px;
  min-width: 0;
  color: #607089;
  font-size: 12px;
  font-weight: 600;
  line-height: 1.8;
}

.owner-list i {
  display: inline-flex;
  align-items: center;
  gap: 4px;
  min-height: 24px;
  padding: 2px 8px;
  font-style: normal;
  border: 1px solid #e4ebf5;
  border-radius: 999px;
  background: #fff;
}

.owner-list b {
  color: #0f68b1;
  font-size: 13px;
  font-weight: 800;
}

.owner-tags .owner-label {
  color: #5f6f86;
  background: #f1f5f9;
}

.owner-tags .owner-list i {
  color: #53657d;
  border-color: #dce8f7;
  background: #f7fbff;
}

.module-tags .owner-label {
  color: #607089;
  background: #edf4ff;
}

.module-tags .owner-list i {
  color: #607089;
  border-color: #e4ebf5;
  background: #fff;
}

.report-section {
  margin-top: 18px;
  overflow: hidden;
  border: 1px solid #e4ebf5;
  border-radius: 16px;
  background: linear-gradient(180deg, #ffffff 0%, #f8fbff 100%);
  box-shadow: 0 8px 20px rgba(39, 65, 96, 0.04);
}

.report-section > .report-section-header {
  overflow: hidden;
  border-bottom: 1px solid #e2edf9;
}

.report-table-block {
  margin: 16px;
  overflow-x: auto;
  overflow-y: hidden;
  border: 1px solid #e4ebf5;
  border-radius: 14px;
  background: #fff;
  box-shadow: 0 8px 20px rgba(39, 65, 96, 0.05);
}

.table-title {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 12px;
  padding: 12px 16px;
  border-bottom: 1px solid #edf2f8;
  background: linear-gradient(135deg, #fbfdff 0%, #f2f7ff 100%);
}

.table-title strong {
  display: inline-flex;
  align-items: center;
  gap: 8px;
  color: #20304a;
  font-size: 14px;
}

.table-title strong::before {
  display: inline-block;
  width: 6px;
  height: 16px;
  content: '';
  border-radius: 999px;
  background: #1890ff;
}

.table-title span {
  color: #f05a4f;
  font-size: 12px;
  font-weight: 800;
}

.broadcast-table {
  width: max-content;
  min-width: 100%;
  border-collapse: collapse;
  table-layout: auto;
  font-size: 13px;
}

.broadcast-table th {
  padding: 12px 14px;
  color: #324860;
  font-weight: 800;
  text-align: center;
  white-space: nowrap;
  background: #f3f7fc;
}

.broadcast-table td {
  padding: 12px 14px;
  color: #344054;
  border-top: 1px solid #edf2f8;
  text-align: center;
  vertical-align: middle;
  white-space: nowrap;
  background: #fff;
}

.broadcast-table tbody tr:nth-child(even) td {
  background: #fbfdff;
}

.broadcast-table tbody tr:hover td {
  background: #f2f8ff;
}

.count-cell,
.id-cell {
  color: #0f68b1;
  font-weight: 800;
}

.subject-cell {
  min-width: 420px;
  max-width: 720px;
  line-height: 1.6;
  text-align: left !important;
  white-space: normal !important;
}

.detail-table th:nth-child(3),
.detail-table td:nth-child(3) {
  width: auto;
}

.detail-table th,
.detail-table td {
  text-align: left;
}

.detail-table .count-cell,
.detail-table .id-cell {
  text-align: left;
}

.empty-cell {
  color: #8b98aa !important;
  text-align: center;
}

.detail-pagination {
  display: flex;
  align-items: center;
  justify-content: flex-end;
  gap: 14px;
  padding: 12px 14px 14px;
  border-top: 1px solid #edf2f8;
  background: #fbfdff;
}

.pagination-total {
  color: #607089;
  font-size: 12px;
  font-weight: 600;
}

.pagination-actions {
  display: inline-flex;
  align-items: center;
  gap: 8px;
}

.pagination-page {
  min-width: 48px;
  color: #20304a;
  font-size: 12px;
  font-weight: 700;
  text-align: center;
}

.trend-toolbar {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 16px;
  margin-bottom: 12px;
}

.trend-summary {
  display: flex;
  flex-wrap: wrap;
  gap: 10px;
  color: #667085;
  font-size: 13px;
}

.trend-stat {
  display: inline-flex;
  align-items: baseline;
  gap: 4px;
  min-width: 126px;
  padding: 10px 14px;
  border: 1px solid #e4ebf5;
  border-radius: 14px;
  background: #fff;
  box-shadow: 0 8px 18px rgba(39, 65, 96, 0.05);
}

.trend-stat-label {
  margin-right: 4px;
  color: #607089;
  font-size: 12px;
  font-weight: 700;
}

.trend-stat strong {
  color: #0f68b1;
  font-size: 22px;
  font-weight: 800;
}

.trend-stat.created strong {
  color: #f05a4f;
}

.trend-stat.fixed strong {
  color: #23b26d;
}

.trend-stat.legacy strong {
  color: #6bb8ff;
}

.chart-legend {
  display: flex;
  justify-content: flex-end;
  gap: 14px;
  color: #445166;
  font-size: 12px;
}

.legend-dot,
.legend-line {
  display: inline-block;
  margin-right: 5px;
  vertical-align: middle;
}

.legend-dot {
  width: 8px;
  height: 8px;
  border-radius: 2px;
}

.legend-line {
  width: 16px;
  height: 3px;
  border-radius: 999px;
}

.red {
  background: #f05a4f;
}

.green {
  background: #23b26d;
}

.yellow {
  background: #f5b622;
}

.blue {
  background: #6bb8ff;
}

.chart-wrap {
  position: relative;
  height: 260px;
  padding: 12px 8px 0;
  border: 1px solid #edf2f8;
  border-radius: 16px;
  background: #fff;
}

.chart-wrap svg {
  width: 100%;
  height: 100%;
}

.grid-lines line {
  stroke: #eef2f7;
  stroke-width: 1;
}

.axis-texts text {
  fill: #7b8797;
  font-size: 12px;
  text-anchor: end;
}

.base-axis {
  stroke: #d8dee8;
  stroke-width: 1.4;
}

.area-path {
  fill: rgba(107, 184, 255, 0.14);
}

.legacy-line {
  fill: none;
  stroke: #6bb8ff;
  stroke-linecap: round;
  stroke-linejoin: round;
  stroke-width: 3;
}

.legacy-dot {
  fill: #fff;
  stroke: #6bb8ff;
  stroke-width: 2;
}

.created-bar {
  fill: #f05a4f;
}

.fixed-bar {
  fill: #23b26d;
}

.hover-zone {
  fill: transparent;
  cursor: pointer;
}

.axis-label {
  fill: #475569;
  font-size: 12px;
  font-weight: 700;
  text-anchor: middle;
}

.trend-tooltip {
  position: absolute;
  top: 38px;
  z-index: 2;
  min-width: 132px;
  padding: 12px 14px;
  color: #5d6878;
  font-size: 12px;
  line-height: 2;
  pointer-events: none;
  border: 1px solid #eef2f7;
  border-radius: 10px;
  background: rgba(255, 255, 255, 0.96);
  box-shadow: 0 14px 34px rgba(31, 45, 61, 0.12);
}

.tooltip-date {
  margin-bottom: 4px;
  color: #344054;
  font-weight: 800;
}

@media (max-width: 768px) {
  .bug-broadcast-page {
    height: calc(100vh - 64px);
    padding: 16px;
  }

  .hero-panel {
    display: block;
    padding: 22px;
  }

  .filter-row {
    max-width: none;
  }

  .project-filter-col,
  .date-filter-col,
  .button-filter-col,
  .export-filter-col {
    flex: 0 0 100%;
    width: 100%;
    max-width: 100%;
  }

  .button-filter-col,
  .export-filter-col {
    margin-left: 0;
  }

  .date-picker {
    min-width: 0;
  }

  .report-head,
  .table-title {
    display: block;
  }

  .overview-tags {
    grid-template-columns: max-content 1fr;
  }

  .overview-item {
    grid-template-columns: 1fr;
  }

  .overview-sub-item {
    margin-left: 18px;
  }

  .overview-tags {
    grid-template-columns: 1fr;
  }

  .trend-toolbar {
    align-items: flex-start;
    flex-direction: column;
  }

  .broadcast-table {
    min-width: 680px;
  }
}

/deep/ .ant-calendar-picker-container {
  z-index: 30;
}
</style>

<style>
body.bug-broadcast-page-active .footer.custom-render,
body.bug-broadcast-page-active .ant-pro-global-footer {
  display: none !important;
}
</style>
