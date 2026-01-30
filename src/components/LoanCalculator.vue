<template>
  <div>
    <div class="calculator-form">
      <div class="form-group">
        <label for="loanAmount">房贷金额（元）</label>
        <input type="number" id="loanAmount" v-model="loanAmount" placeholder="请输入房贷金额" min="0" step="0.01">
      </div>

      <div class="form-group">
        <label for="interestRate">年利率（%）</label>
        <input type="number" id="interestRate" v-model="interestRate" placeholder="请输入年利率，如4.5" min="0" step="0.01">
      </div>

      <div class="form-group">
        <label for="loanTerm">贷款年限（年）</label>
        <input type="number" id="loanTerm" v-model="loanTerm" placeholder="请输入贷款年限" min="1" max="50" step="1">
      </div>

      <div class="form-group">
        <label for="repaymentType">还款方式</label>
        <select id="repaymentType" v-model="repaymentType">
          <option value="equal-payment">等额本息</option>
          <option value="equal-principal">等额本金</option>
        </select>
      </div>

      <div class="form-group">
        <label for="firstPaymentDate">首次还款时间</label>
        <input type="date" id="firstPaymentDate" v-model="firstPaymentDate">
      </div>

      <button class="calculate-btn" @click="calculateLoan">计算</button>
      <div class="action-buttons">
        <button class="export-btn" @click="exportConfig">导出配置</button>
        <label class="import-btn">
          导入配置
          <input type="file" @change="importConfig" accept=".json" style="display: none;">
        </label>
      </div>
    </div>

    <div class="results" v-if="showResults">
      <div class="summary">
        <h2>还款概览</h2>
        <div class="summary-item">
          <span>贷款总额：</span>
          <span class="amount">{{ formatCurrency(loanAmount) }}</span>
        </div>
        <div class="summary-item">
          <span>还款总额：</span>
          <span class="amount">{{ formatCurrency(totalPayment) }}</span>
        </div>
        <div class="summary-item">
          <span>支付利息：</span>
          <span class="amount">{{ formatCurrency(totalInterest) }}</span>
        </div>
        <div class="summary-item">
          <span>还款期数：</span>
          <span>{{ payments.length }} 期</span>
        </div>
        <div class="summary-item">
          <span>当前剩余本金：</span>
          <span class="amount">{{ formatCurrency(currentRemainingPrincipal) }}</span>
        </div>
      </div>

      <div class="chart-section">
        <h2>还款趋势图</h2>
        <div class="chart-container">
          <canvas ref="chartRef"></canvas>
        </div>
      </div>

      <div class="rate-change-section">
        <h2>利率变更</h2>
        <div class="rate-change-form">
          <div class="form-group">
            <label for="newInterestRate">新利率（%）</label>
            <input type="number" v-model="newInterestRate" placeholder="请输入新利率" min="0" step="0.01">
          </div>
          <div class="form-group">
            <label for="rateChangeDate">生效日期</label>
            <input type="date" v-model="rateChangeDate">
          </div>
          <button class="rate-change-btn" @click="applyRateChange">应用利率变更</button>
        </div>
        <div class="history-section">
          <h3>历史操作</h3>
          <div class="history-list">
            <div v-if="rateChangesList.length === 0" class="history-empty">暂无操作记录</div>
            <div v-for="rc in rateChangesList" :key="rc.id" class="history-item">
              <div class="history-content">
                <span class="history-label">生效日期：</span>
                <span>{{ formatDate(rc.date) }}</span>
                <span class="history-label">新利率：</span>
                <span class="history-value">{{ rc.newRate }}%</span>
                <span v-if="rc.savedAmount" class="history-saved">节省: {{ formatCurrency(rc.savedAmount) }}</span>
              </div>
              <button class="history-delete-btn" @click="removeRateChange(rc.id)">删除</button>
            </div>
          </div>
        </div>
      </div>

      <div class="early-payment-section">
        <h2>提前还款</h2>
        <div class="early-payment-form">
          <div class="form-group">
            <label for="earlyPaymentAmount">提前还款金额（元）</label>
            <input type="number" v-model="earlyPaymentAmount" placeholder="请输入提前还款金额" min="0" step="0.01">
          </div>
          <div class="form-group">
            <label for="earlyPaymentDate">还款日期</label>
            <input type="date" v-model="earlyPaymentDate">
          </div>
          <div class="form-group">
            <label for="earlyPaymentType">提前还款方式</label>
            <select v-model="earlyPaymentType">
              <option value="reduce-payment">减少月供（期数不变）</option>
              <option value="reduce-term">减少期数（月供不变）</option>
            </select>
          </div>
          <button class="early-payment-btn" @click="applyEarlyPayment">提交提前还款</button>
        </div>
        <div class="history-section">
          <h3>历史操作</h3>
          <div class="history-list">
            <div v-if="earlyPaymentsList.length === 0" class="history-empty">暂无操作记录</div>
            <div v-for="ep in earlyPaymentsList" :key="ep.id" class="history-item">
              <div class="history-content">
                <span class="history-label">还款日期：</span>
                <span>{{ formatDate(ep.date) }}</span>
                <span class="history-label">金额：</span>
                <span class="history-value">{{ formatCurrency(ep.amount) }}</span>
                <span class="history-label">方式：</span>
                <span>{{ ep.type === 'reduce-payment' ? '减少月供' : '减少期数' }}</span>
                <span v-if="ep.savedAmount" class="history-saved">节省: {{ formatCurrency(ep.savedAmount) }}</span>
              </div>
              <button class="history-delete-btn" @click="removeEarlyPayment(ep.id)">删除</button>
            </div>
          </div>
        </div>
      </div>

      <div class="payment-details">
        <h2>还款明细</h2>
        <div class="table-container">
          <table>
            <thead>
              <tr>
                <th>期数</th>
                <th>还款日期</th>
                <th>月供金额</th>
                <th>本金</th>
                <th>利息</th>
                <th>剩余本金</th>
                <th>状态</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="payment in payments" :key="payment.period" :class="{ 'early-payment-row': payment.earlyPayment }">
                <td>{{ payment.period }}</td>
                <td>{{ formatDate(payment.date) }}<span v-if="payment.rateChanged" class="rate-change-mark">(利率已变更)</span><span v-if="payment.recalculated" class="recalc-mark">(已重新计算)</span></td>
                <td>{{ formatCurrency(payment.monthlyPayment) }}<div v-if="payment.earlyPayment" class="early-payment-info">提前还款：{{ formatCurrency(payment.earlyPayment) }}{{ payment.earlyPaymentType === 'reduce-payment' ? '（减少月供）' : '（减少期数）' }}</div></td>
                <td>{{ formatCurrency(payment.principal) }}</td>
                <td>{{ formatCurrency(payment.interest) }}</td>
                <td>{{ formatCurrency(payment.remainingPrincipal) }}</td>
                <td>
                  <span v-if="payment.earlyPayment" class="status-badge early-payment">已提前还款</span>
                  <span v-else class="status-badge unpaid">正常还款</span>
                </td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted, watch, nextTick } from 'vue'
import Chart from 'chart.js/auto'

// 响应式数据
const loanAmount = ref('')
const interestRate = ref('')
const loanTerm = ref('')
const repaymentType = ref('equal-payment')
const firstPaymentDate = ref('')

const newInterestRate = ref('')
const rateChangeDate = ref('')

const earlyPaymentAmount = ref('')
const earlyPaymentDate = ref('')
const earlyPaymentType = ref('reduce-payment')

const showResults = ref(false)
const payments = ref([])
const totalPayment = ref(0)
const totalInterest = ref(0)
const earlyPaymentsList = ref([])
const rateChangesList = ref([])
const nextId = ref(1)

const chartRef = ref(null)
let paymentChart = null

// 计算当前剩余本金
const currentRemainingPrincipal = computed(() => {
  const today = new Date()
  today.setHours(0, 0, 0, 0)

  if (payments.value.length === 0) {
    return parseFloat(loanAmount.value) || 0
  }

  for (let i = payments.value.length - 1; i >= 0; i--) {
    const paymentDate = new Date(payments.value[i].date)
    paymentDate.setHours(0, 0, 0, 0)

    if (paymentDate <= today) {
      return payments.value[i].remainingPrincipal
    }
  }

  return parseFloat(loanAmount.value) || 0
})

// 格式化货币
const formatCurrency = (amount) => {
  return '¥' + Number(amount).toFixed(2).replace(/\B(?=(\d{3})+(?!\d))/g, ',')
}

// 格式化日期
const formatDate = (date) => {
  const d = new Date(date)
  const year = d.getFullYear()
  const month = String(d.getMonth() + 1).padStart(2, '0')
  const day = String(d.getDate()).padStart(2, '0')
  return `${year}-${month}-${day}`
}

// 等额本息计算
const calculateEqualPayment = (loanAmount, annualRate, years, firstPaymentDate) => {
  const monthlyRate = annualRate / 100 / 12
  const totalMonths = years * 12

  // 计算月供
  const monthlyPayment = loanAmount * monthlyRate * Math.pow(1 + monthlyRate, totalMonths) /
    (Math.pow(1 + monthlyRate, totalMonths) - 1)

  let totalPayment = 0
  let totalInterest = 0
  let remainingPrincipal = loanAmount

  const paymentsArr = []
  const paymentDate = new Date(firstPaymentDate)

  for (let i = 1; i <= totalMonths; i++) {
    paymentDate.setMonth(paymentDate.getMonth() + 1)

    const interest = remainingPrincipal * monthlyRate
    const principal = monthlyPayment - interest

    remainingPrincipal -= principal
    totalPayment += monthlyPayment
    totalInterest += interest

    const finalRemainingPrincipal = Math.max(0, remainingPrincipal)

    paymentsArr.push({
      period: i,
      date: new Date(paymentDate),
      monthlyPayment,
      principal,
      interest,
      remainingPrincipal: finalRemainingPrincipal
    })
  }

  return { payments: paymentsArr, totalPayment, totalInterest }
}

// 等额本金计算
const calculateEqualPrincipal = (loanAmount, annualRate, years, firstPaymentDate) => {
  const monthlyRate = annualRate / 100 / 12
  const totalMonths = years * 12

  let totalPayment = 0
  let totalInterest = 0
  let remainingPrincipal = loanAmount
  const monthlyPrincipal = loanAmount / totalMonths

  const paymentsArr = []
  const paymentDate = new Date(firstPaymentDate)

  for (let i = 1; i <= totalMonths; i++) {
    paymentDate.setMonth(paymentDate.getMonth() + 1)

    const interest = remainingPrincipal * monthlyRate
    const monthlyPayment = monthlyPrincipal + interest

    remainingPrincipal -= monthlyPrincipal
    totalPayment += monthlyPayment
    totalInterest += interest

    const finalRemainingPrincipal = Math.max(0, remainingPrincipal)

    paymentsArr.push({
      period: i,
      date: new Date(paymentDate),
      monthlyPayment,
      principal: monthlyPrincipal,
      interest,
      remainingPrincipal: finalRemainingPrincipal
    })
  }

  return { payments: paymentsArr, totalPayment, totalInterest }
}

// 减少月供重新计算
const recalculateReducePayment = (paymentsArr, startIndex, remainingPrincipal, firstPaymentDate, repaymentType) => {
  const monthlyRate = interestRate.value / 100 / 12
  const remainingMonths = paymentsArr.length - startIndex

  let monthlyPayment
  if (repaymentType === 'equal-payment') {
    monthlyPayment = remainingPrincipal * monthlyRate * Math.pow(1 + monthlyRate, remainingMonths) /
      (Math.pow(1 + monthlyRate, remainingMonths) - 1)
  } else {
    monthlyPayment = paymentsArr[startIndex].monthlyPayment
  }

  for (let i = startIndex; i < paymentsArr.length; i++) {
    const interest = remainingPrincipal * monthlyRate
    const principal = monthlyPayment - interest
    remainingPrincipal -= principal
    paymentsArr[i].monthlyPayment = monthlyPayment
    paymentsArr[i].principal = principal
    paymentsArr[i].interest = interest
    paymentsArr[i].remainingPrincipal = Math.max(0, remainingPrincipal)
    paymentsArr[i].recalculated = true
  }

  return paymentsArr
}

// 减少期数重新计算
const recalculateReduceTerm = (paymentsArr, startIndex, remainingPrincipal, firstPaymentDate, repaymentType) => {
  const monthlyRate = interestRate.value / 100 / 12
  const monthlyPayment = paymentsArr[startIndex - 1]?.monthlyPayment || paymentsArr[0].monthlyPayment

  if (repaymentType === 'equal-payment') {
    // 等额本息：保持月供不变，计算剩余期数
    const totalInterest = monthlyPayment * paymentsArr.length - interestRate.value / 100 / 12 * remainingPrincipal
    const remainingMonths = Math.ceil(remainingPrincipal / (monthlyPayment - monthlyPayment / paymentsArr.length))

    for (let i = startIndex; i < paymentsArr.length; i++) {
      if (i - startIndex >= remainingMonths) {
        paymentsArr[i].removed = true
        continue
      }
      const interest = remainingPrincipal * monthlyRate
      const principal = monthlyPayment - interest
      remainingPrincipal -= principal
      paymentsArr[i].monthlyPayment = monthlyPayment
      paymentsArr[i].principal = principal
      paymentsArr[i].interest = interest
      paymentsArr[i].remainingPrincipal = Math.max(0, remainingPrincipal)
      paymentsArr[i].recalculated = true
    }
  } else {
    // 等额本金：保持月供不变，计算剩余期数
    const monthlyPrincipal = paymentsArr[0].principal
    const remainingMonths = Math.ceil(remainingPrincipal / monthlyPrincipal)

    for (let i = startIndex; i < paymentsArr.length; i++) {
      if (i - startIndex >= remainingMonths) {
        paymentsArr[i].removed = true
        continue
      }
      const interest = remainingPrincipal * monthlyRate
      const monthlyPayment = monthlyPrincipal + interest
      remainingPrincipal -= monthlyPrincipal
      paymentsArr[i].monthlyPayment = monthlyPayment
      paymentsArr[i].principal = monthlyPrincipal
      paymentsArr[i].interest = interest
      paymentsArr[i].remainingPrincipal = Math.max(0, remainingPrincipal)
      paymentsArr[i].recalculated = true
    }
  }

  return paymentsArr
}

// 应用单次提前还款
const applySingleEarlyPayment = (results, earlyPayment, firstPaymentDate) => {
  let paymentsArr = [...results.payments]

  const earlyPaymentDate = new Date(earlyPayment.date)
  const targetPeriodIndex = paymentsArr.findIndex(p => {
    const paymentDate = new Date(p.date)
    return paymentDate >= earlyPaymentDate
  })
  if (targetPeriodIndex === -1) return results

  const remainingPrincipalBefore = paymentsArr[targetPeriodIndex].remainingPrincipal
  const remainingPrincipalAfter = Math.max(0, remainingPrincipalBefore - earlyPayment.amount)

  paymentsArr[targetPeriodIndex].earlyPayment = earlyPayment.amount
  paymentsArr[targetPeriodIndex].earlyPaymentType = earlyPayment.type

  if (earlyPayment.type === 'reduce-payment') {
    paymentsArr = recalculateReducePayment(paymentsArr, targetPeriodIndex, remainingPrincipalAfter, firstPaymentDate, repaymentType.value)
  } else if (earlyPayment.type === 'reduce-term') {
    paymentsArr = recalculateReduceTerm(paymentsArr, targetPeriodIndex, remainingPrincipalAfter, firstPaymentDate, repaymentType.value)
  }

  let totalPayment2 = 0
  let totalInterest2 = 0
  paymentsArr.forEach(p => {
    if (!p.removed) {
      totalPayment2 += p.monthlyPayment
      if (p.earlyPayment) {
        totalPayment2 += p.earlyPayment
      }
      totalInterest2 += p.interest
    }
  })

  return {
    payments: paymentsArr.filter(p => !p.removed),
    totalPayment: totalPayment2,
    totalInterest: totalInterest2
  }
}

// 应用单次利率变更
const applySingleRateChange = (results, rateChange, firstPaymentDate) => {
  let paymentsArr = [...results.payments]

  const changeDate = new Date(rateChange.date)
  const changePeriodIndex = paymentsArr.findIndex(p => {
    const paymentDate = new Date(p.date)
    return paymentDate >= changeDate
  })

  if (changePeriodIndex === -1) return results

  let remainingPrincipal
  if (changePeriodIndex > 0) {
    remainingPrincipal = paymentsArr[changePeriodIndex - 1].remainingPrincipal
  } else {
    remainingPrincipal = loanAmount.value
  }

  if (paymentsArr[changePeriodIndex] && paymentsArr[changePeriodIndex].earlyPayment) {
    remainingPrincipal = Math.max(0, remainingPrincipal - paymentsArr[changePeriodIndex].earlyPayment)
  }

  const newRate = rateChange.newRate
  const monthlyRate = newRate / 100 / 12
  const remainingMonths = paymentsArr.length - changePeriodIndex

  let monthlyPayment
  if (repaymentType.value === 'equal-payment') {
    monthlyPayment = remainingPrincipal * monthlyRate * Math.pow(1 + monthlyRate, remainingMonths) /
      (Math.pow(1 + monthlyRate, remainingMonths) - 1)
  } else {
    const monthlyPrincipal = paymentsArr[0].principal
    for (let i = changePeriodIndex; i < paymentsArr.length; i++) {
      const interest = remainingPrincipal * monthlyRate
      paymentsArr[i].monthlyPayment = monthlyPrincipal + interest
      paymentsArr[i].principal = monthlyPrincipal
      paymentsArr[i].interest = interest
      remainingPrincipal -= monthlyPrincipal
      paymentsArr[i].remainingPrincipal = Math.max(0, remainingPrincipal)
      paymentsArr[i].rateChanged = true
    }

    let totalPayment2 = 0
    let totalInterest2 = 0
    paymentsArr.forEach(p => {
      if (!p.removed) {
        totalPayment2 += p.monthlyPayment
        if (p.earlyPayment) {
          totalPayment2 += p.earlyPayment
        }
        totalInterest2 += p.interest
      }
    })

    return {
      payments: paymentsArr.filter(p => !p.removed),
      totalPayment: totalPayment2,
      totalInterest: totalInterest2
    }
  }

  for (let i = changePeriodIndex; i < paymentsArr.length; i++) {
    const interest = remainingPrincipal * monthlyRate
    const principal = monthlyPayment - interest
    remainingPrincipal -= principal
    paymentsArr[i].monthlyPayment = monthlyPayment
    paymentsArr[i].principal = principal
    paymentsArr[i].interest = interest
    paymentsArr[i].remainingPrincipal = Math.max(0, remainingPrincipal)
    paymentsArr[i].rateChanged = true
  }

  let totalPayment2 = 0
  let totalInterest2 = 0
  paymentsArr.forEach(p => {
    if (!p.removed) {
      totalPayment2 += p.monthlyPayment
      if (p.earlyPayment) {
        totalPayment2 += p.earlyPayment
      }
      totalInterest2 += p.interest
    }
  })

  return {
    payments: paymentsArr.filter(p => !p.removed),
    totalPayment: totalPayment2,
    totalInterest: totalInterest2
  }
}

// 重新计算贷款
const recalculateLoan = () => {
  const baseResults = repaymentType.value === 'equal-payment'
    ? calculateEqualPayment(loanAmount.value, interestRate.value, loanTerm.value, new Date(firstPaymentDate.value))
    : calculateEqualPrincipal(loanAmount.value, interestRate.value, loanTerm.value, new Date(firstPaymentDate.value))

  let results = baseResults

  const allChanges = []
  earlyPaymentsList.value.forEach(ep => {
    allChanges.push({
      type: 'early-payment',
      date: new Date(ep.date),
      data: ep,
      priority: 1
    })
  })
  rateChangesList.value.forEach(rc => {
    allChanges.push({
      type: 'rate-change',
      date: new Date(rc.date),
      data: rc,
      priority: 2
    })
  })

  allChanges.sort((a, b) => {
    const dateDiff = a.date - b.date
    if (dateDiff !== 0) return dateDiff
    return a.priority - b.priority
  })

  allChanges.forEach(change => {
    if (change.type === 'early-payment') {
      results = applySingleEarlyPayment(results, change.data, new Date(firstPaymentDate.value))
    } else if (change.type === 'rate-change') {
      results = applySingleRateChange(results, change.data, new Date(firstPaymentDate.value))
    }
  })

  return results
}

// 计算贷款
const calculateLoan = () => {
  if (!loanAmount.value || loanAmount.value <= 0) {
    alert('请输入有效的房贷金额')
    return
  }
  if (!interestRate.value || interestRate.value <= 0) {
    alert('请输入有效的年利率')
    return
  }
  if (!loanTerm.value || loanTerm.value <= 0) {
    alert('请输入有效的贷款年限')
    return
  }
  if (!firstPaymentDate.value) {
    alert('请选择首次还款时间')
    return
  }

  earlyPaymentsList.value = []
  rateChangesList.value = []
  nextId.value = 1

  const results = recalculateLoan()
  payments.value = results.payments
  totalPayment.value = results.totalPayment
  totalInterest.value = results.totalInterest
  showResults.value = true

  nextTick(() => {
    updatePaymentChart(results)
  })
}

// 应用利率变更
const applyRateChange = () => {
  if (!newInterestRate.value || newInterestRate.value <= 0) {
    alert('请输入有效的利率')
    return
  }
  if (!rateChangeDate.value) {
    alert('请选择生效日期')
    return
  }

  const rateChange = {
    id: `rc_${nextId.value++}`,
    newRate: parseFloat(newInterestRate.value),
    date: new Date(rateChangeDate.value)
  }

  rateChangesList.value.push(rateChange)

  const results = recalculateLoan()

  // 计算节省的利息
  const previousTotalInterest = totalInterest.value
  rateChange.savedAmount = previousTotalInterest - results.totalInterest

  payments.value = results.payments
  totalPayment.value = results.totalPayment
  totalInterest.value = results.totalInterest

  nextTick(() => {
    updatePaymentChart(results)
  })

  newInterestRate.value = ''
  rateChangeDate.value = ''
}

// 删除利率变更
const removeRateChange = (id) => {
  const index = rateChangesList.value.findIndex(rc => rc.id === id)
  if (index !== -1) {
    rateChangesList.value.splice(index, 1)

    const results = recalculateLoan()
    payments.value = results.payments
    totalPayment.value = results.totalPayment
    totalInterest.value = results.totalInterest

    nextTick(() => {
      updatePaymentChart(results)
    })
  }
}

// 应用提前还款
const applyEarlyPayment = () => {
  if (!earlyPaymentAmount.value || earlyPaymentAmount.value <= 0) {
    alert('请输入有效的提前还款金额')
    return
  }
  if (!earlyPaymentDate.value) {
    alert('请选择还款日期')
    return
  }

  const earlyPayment = {
    id: `ep_${nextId.value++}`,
    amount: parseFloat(earlyPaymentAmount.value),
    type: earlyPaymentType.value,
    date: new Date(earlyPaymentDate.value)
  }

  earlyPaymentsList.value.push(earlyPayment)

  const results = recalculateLoan()

  // 计算节省的利息
  const previousTotalInterest = totalInterest.value
  earlyPayment.savedAmount = previousTotalInterest - results.totalInterest

  payments.value = results.payments
  totalPayment.value = results.totalPayment
  totalInterest.value = results.totalInterest

  nextTick(() => {
    updatePaymentChart(results)
  })

  earlyPaymentAmount.value = ''
  earlyPaymentDate.value = ''
}

// 删除提前还款
const removeEarlyPayment = (id) => {
  const index = earlyPaymentsList.value.findIndex(ep => ep.id === id)
  if (index !== -1) {
    earlyPaymentsList.value.splice(index, 1)

    const results = recalculateLoan()
    payments.value = results.payments
    totalPayment.value = results.totalPayment
    totalInterest.value = results.totalInterest

    nextTick(() => {
      updatePaymentChart(results)
    })
  }
}

// 更新图表
const updatePaymentChart = (results) => {
  if (!chartRef.value) return

  const ctx = chartRef.value.getContext('2d')

  const labels = results.payments.map(p => `第${p.period}期`)
  const monthlyPayments = results.payments.map(p => p.monthlyPayment.toFixed(2))

  const gradient = ctx.createLinearGradient(0, 0, 0, 300)
  gradient.addColorStop(0, 'rgba(102, 126, 234, 0.2)')
  gradient.addColorStop(1, 'rgba(102, 126, 234, 0)')

  const pointBackgroundColors = []
  const pointRadius = []
  const pointBorderWidth = []

  results.payments.forEach(payment => {
    if (payment.earlyPayment || payment.rateChanged) {
      pointBackgroundColors.push('rgba(255, 87, 87, 1)')
      pointRadius.push(0)
      pointBorderWidth.push(0)
    } else {
      pointBackgroundColors.push('rgba(102, 126, 234, 1)')
      pointRadius.push(0)
      pointBorderWidth.push(0)
    }
  })

  if (paymentChart) {
    paymentChart.destroy()
  }

  paymentChart = new Chart(ctx, {
    type: 'line',
    data: {
      labels,
      datasets: [{
        label: '月供金额',
        data: monthlyPayments,
        borderColor: 'rgba(102, 126, 234, 1)',
        backgroundColor: gradient,
        pointBackgroundColor: pointBackgroundColors,
        pointBorderColor: 'rgba(255, 255, 255, 1)',
        pointRadius,
        pointHoverRadius: 5,
        pointBorderWidth: 2,
        borderWidth: 2,
        tension: 0.4,
        fill: true
      }]
    },
    options: {
      responsive: true,
      maintainAspectRatio: false,
      animation: {
        duration: 800,
        easing: 'easeOutQuart'
      },
      plugins: {
        legend: {
          display: false
        },
        tooltip: {
          mode: 'index',
          intersect: false,
          backgroundColor: 'rgba(255, 255, 255, 0.95)',
          titleColor: '#333',
          bodyColor: '#666',
          borderColor: 'rgba(102, 126, 234, 0.2)',
          borderWidth: 1,
          titleFont: {
            size: 13,
            weight: '600'
          },
          bodyFont: {
            size: 12
          },
          padding: 10,
          callbacks: {
            label: (context) => {
              const payment = results.payments[context.dataIndex]
              let label = `月供: ¥${parseFloat(context.raw).toLocaleString('zh-CN', { minimumFractionDigits: 2, maximumFractionDigits: 2 })}`
              if (payment.earlyPayment) {
                const typeText = payment.earlyPaymentType === 'reduce-payment' ? '（减少月供）' : '（减少期数）'
                label += `\n💰 提前还款: ¥${payment.earlyPayment.toFixed(2)}${typeText}`
              }
              if (payment.rateChanged) {
                label += `\n📈 利率已变更`
              }
              return label
            }
          }
        }
      },
      scales: {
        x: {
          grid: {
            display: false
          },
          ticks: {
            font: {
              size: 11
            },
            maxTicksLimit: 15
          }
        },
        y: {
          grid: {
            color: 'rgba(0, 0, 0, 0.04)'
          },
          ticks: {
            font: {
              size: 11
            },
            callback: (value) => {
              if (value >= 10000) {
                return '¥' + (value / 10000).toFixed(1) + '万'
              }
              return '¥' + value.toLocaleString()
            }
          }
        }
      }
    }
  })
}

// 导出配置
const exportConfig = () => {
  const config = {
    loanAmount: loanAmount.value,
    annualRate: interestRate.value,
    loanYears: loanTerm.value,
    repaymentType: repaymentType.value,
    firstPaymentDate: firstPaymentDate.value,
    earlyPayments: earlyPaymentsList.value,
    rateChanges: rateChangesList.value,
    exportDate: new Date().toISOString()
  }

  const dataStr = JSON.stringify(config, null, 2)
  const dataBlob = new Blob([dataStr], { type: 'application/json' })

  const url = URL.createObjectURL(dataBlob)
  const link = document.createElement('a')
  link.href = url
  const fileName = `房贷配置_${new Date().toISOString().split('T')[0]}.json`
  link.download = fileName
  link.click()

  URL.revokeObjectURL(url)
}

// 导入配置
const importConfig = (event) => {
  const file = event.target.files[0]
  if (!file) return

  const reader = new FileReader()
  reader.onload = (e) => {
    try {
      const config = JSON.parse(e.target.result)

      if (!config.loanAmount || !config.annualRate || !config.loanYears || !config.repaymentType || !config.firstPaymentDate) {
        alert('配置文件格式不正确，缺少必需字段')
        return
      }

      loanAmount.value = config.loanAmount
      interestRate.value = config.annualRate
      loanTerm.value = config.loanYears
      repaymentType.value = config.repaymentType
      firstPaymentDate.value = config.firstPaymentDate

      if (config.earlyPayments && Array.isArray(config.earlyPayments)) {
        earlyPaymentsList.value = config.earlyPayments.map(ep => ({
          ...ep,
          date: new Date(ep.date)
        }))
        const maxId = earlyPaymentsList.value.reduce((max, ep) => Math.max(max, parseInt(ep.id?.replace(/\D/g, '') || 0)), 0)
        nextId.value = maxId + 1
      } else {
        earlyPaymentsList.value = []
      }

      if (config.rateChanges && Array.isArray(config.rateChanges)) {
        rateChangesList.value = config.rateChanges.map(rc => ({
          ...rc,
          date: new Date(rc.date)
        }))
      } else {
        rateChangesList.value = []
      }

      const results = recalculateLoan()
      payments.value = results.payments
      totalPayment.value = results.totalPayment
      totalInterest.value = results.totalInterest
      showResults.value = true

      nextTick(() => {
        updatePaymentChart(results)
      })

      alert('配置导入成功！')
    } catch (error) {
      console.error('导入配置失败:', error)
      alert('导入配置失败，请确保文件格式正确')
    }
  }

  reader.readAsText(file)
  event.target.value = ''
}

// 初始化
onMounted(() => {
  const today = new Date()
  const nextMonth = new Date(today.getFullYear(), today.getMonth() + 1, 1)

  if (!firstPaymentDate.value) {
    firstPaymentDate.value = nextMonth.toISOString().split('T')[0]
  }

  rateChangeDate.value = nextMonth.toISOString().split('T')[0]
  earlyPaymentDate.value = nextMonth.toISOString().split('T')[0]
})
</script>
