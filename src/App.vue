<template>
  <div class="app">
    <header>
      <h1>CrowdFunding DApp</h1>
      <div class="wallet-info" v-if="account">
        <span>{{ account.slice(0,6) }}...{{ account.slice(-4) }}</span>
        <span class="balance">{{ balance }} ETH</span>
        <span class="pending" v-if="pending > 0">待提取: {{ pending }} ETH</span>
        <button @click="doWithdrawPending" v-if="pending > 0" class="btn-sm">提取</button>
      </div>
      <button v-else @click="connectWallet" class="btn">连接钱包</button>
    </header>

    <div v-if="!account" class="connect-hint">
      <p>请先连接 MetaMask 钱包</p>
      <p>确保已切换到 Hardhat 本地网络 (Chain ID: 31337)</p>
      <p v-if="connectError" class="error-text">{{ connectError }}</p>
    </div>

    <div v-else class="main">
      <nav class="tabs">
        <button :class="{active: tab==='list'}" @click="tab='list'">项目列表</button>
        <button :class="{active: tab==='create'}" @click="tab='create'">创建项目</button>
        <button :class="{active: tab==='detail'}" @click="tab='detail'" v-if="selectedProject">项目详情</button>
        <button :class="{active: tab==='records'}" @click="tab='records'; loadRecords()">交易记录</button>
      </nav>

      <div v-if="tab==='create'" class="panel">
        <h2>创建众筹项目</h2>
        <form @submit.prevent="doCreateProject">
          <label>项目名称</label>
          <input v-model="form.name" placeholder="输入项目名称" required />

          <label>项目描述</label>
          <textarea v-model="form.description" placeholder="输入项目描述" required></textarea>

          <label>可捐赠上限 (ETH)</label>
          <input v-model="form.goal" type="number" step="0.01" min="0.01" required />

          <label>截止时间</label>
          <div style="display:flex;gap:8px;align-items:center">
            <input v-model="form.deadline" type="datetime-local" required style="flex:1" />
            <button type="button" @click="setDeadlineSoon" class="btn-sm" style="margin-top:0;white-space:nowrap">1分钟后</button>
            <button type="button" @click="setDeadlineHour" class="btn-sm" style="margin-top:0;white-space:nowrap">1小时后</button>
          </div>

          <label>早期捐赠者名额 (0表示不启用)</label>
          <input v-model="form.earlyLimit" type="number" min="0" />

          <label>每人奖励 (ETH)</label>
          <input v-model="form.earlyReward" type="number" step="0.001" min="0" />



          <button type="submit" class="btn" :disabled="loading">
            {{ loading ? '处理中...' : '创建项目' }}
          </button>
        </form>
      </div>

      <div v-if="tab==='list'" class="panel">
        <h2>众筹项目</h2>
        <div class="filter">
          <button :class="{active: filter==='all'}" @click="filter='all'">全部</button>
          <button :class="{active: filter==='ongoing'}" @click="filter='ongoing'">进行中</button>
          <button :class="{active: filter==='ended'}" @click="filter='ended'">已结束</button>
          <button @click="loadProjects" class="btn-sm">刷新</button>
        </div>

        <div v-if="projects.length === 0" class="empty">暂无项目</div>

        <div v-for="p in filteredProjects" :key="p.id" class="project-card" @click="selectProject(p)">
          <div class="project-header">
            <h3>{{ p.name }}</h3>
            <span :class="['status', p.ongoing ? 'ongoing' : 'ended']">
              {{ p.ongoing ? '进行中' : (p.succeeded ? '成功' : '未达标') }}
            </span>
          </div>
          <p class="desc">{{ p.description }}</p>
          <div class="progress-bar">
            <div class="progress-fill" :style="{width: Math.min(100, p.progress) + '%'}"></div>
          </div>
          <div class="stats">
            <span>{{ p.raised }} / {{ p.goal }} ETH</span>
            <span>{{ p.progress.toFixed(1) }}%</span>
          </div>
          <div class="meta">
            <span>截止: {{ p.deadlineStr }}</span>
            <span>捐赠者: {{ p.donorCount }}</span>
            <span v-if="p.earlyLimit > 0">早鸟: {{ p.earlyCount }}/{{ p.earlyLimit }}</span>
          </div>
        </div>
      </div>

      <div v-if="tab==='detail' && selectedProject" class="panel">
        <button @click="tab='list'" class="btn-back">返回列表</button>
        <h2>{{ selectedProject.name }}</h2>
        <p class="desc">{{ selectedProject.description }}</p>

        <div class="detail-grid">
          <div class="detail-item">
            <label>可捐赠上限</label>
            <span>{{ selectedProject.goal }} ETH</span>
          </div>
          <div class="detail-item">
            <label>已筹金额</label>
            <span>{{ selectedProject.raised }} ETH</span>
          </div>
          <div class="detail-item">
            <label>截止时间</label>
            <span>{{ selectedProject.deadlineStr }}</span>
          </div>
          <div class="detail-item">
            <label>状态</label>
            <span :class="['status', selectedProject.ongoing ? 'ongoing' : 'ended']">
              {{ selectedProject.ongoing ? '进行中' : (selectedProject.succeeded ? '成功' : '未达标') }}
            </span>
          </div>
          <div class="detail-item">
            <label>创建者</label>
            <span class="addr">{{ selectedProject.creator }}</span>
          </div>
          <div class="detail-item">
            <label>早期捐赠者</label>
            <span>{{ selectedProject.earlyCount }}/{{ selectedProject.earlyLimit }} (剩余 {{ selectedProject.remainingSlots }})</span>
          </div>

        </div>

        <div class="progress-bar large">
          <div class="progress-fill" :style="{width: Math.min(100, selectedProject.progress) + '%'}"></div>
        </div>
        <div class="stats large">
          <span>{{ selectedProject.progress.toFixed(1) }}%</span>
        </div>

        <div class="actions">
          <div v-if="selectedProject.ongoing" class="action-group">
            <h3>捐赠</h3>
            <p style="color:#666;font-size:13px;margin-bottom:8px">
              可捐赠上限: {{ selectedProject.goal }} ETH，已筹: {{ selectedProject.raised }} ETH，
              剩余额度: {{ Math.max(0, selectedProject.goal - selectedProject.raised).toFixed(4) }} ETH
            </p>
            <p v-if="donateAmount > 0 && donateAmount > selectedProject.goal - selectedProject.raised" style="color:#e94560;font-size:13px;margin-bottom:8px">
              超出上限，实际只捐赠 {{ Math.min(donateAmount, selectedProject.goal - selectedProject.raised).toFixed(4) }} ETH，多余部分退还
            </p>
            <form @submit.prevent="doDonate" class="donate-form">
              <input v-model="donateAmount" type="number" step="0.01" min="0.01" :max="selectedProject.goal - selectedProject.raised" placeholder="ETH 数量" required />
              <button type="submit" class="btn" :disabled="loading">捐赠</button>
            </form>
          </div>

          <div v-if="!selectedProject.closed && (selectedProject.deadlinePassed || selectedProject.succeeded)" class="action-group">
            <h3>关闭项目</h3>
            <p v-if="selectedProject.succeeded">上限已达成，可以关闭项目并提取资金</p>
            <p v-else>已到期，可以关闭项目</p>
            <button @click="doCloseProject" class="btn-warn" :disabled="loading">关闭项目</button>
          </div>
          <div v-if="!selectedProject.closed && !selectedProject.deadlinePassed && !selectedProject.succeeded" class="action-group">
            <h3>等待截止</h3>
            <p>项目截止时间: {{ selectedProject.deadlineStr }}</p>
            <p>到期后才能关闭项目</p>
          </div>

          <div v-if="selectedProject.closed && selectedProject.succeeded && isCreator" class="action-group success-box">
            <h3>项目成功！资金已进入待提取状态</h3>
            <p>您的项目已达标，筹集的 {{ selectedProject.raised }} ETH 已准备好提取。</p>
            <p v-if="pending > 0" style="font-size:18px;font-weight:600;color:#155724;margin:12px 0">待提取余额: {{ pending }} ETH</p>
            <button @click="doWithdrawPending" class="btn" :disabled="loading" v-if="pending > 0">
              {{ loading ? '处理中...' : '提取资金到我的账户' }}
            </button>
            <p v-else style="color:#666">资金已提取完毕</p>
          </div>

          <div v-if="selectedProject.closed && !selectedProject.succeeded" class="action-group">
            <h3>退款</h3>
            <p>项目未达标，捐赠者可取回资金</p>
            <button @click="doRefund" class="btn" :disabled="loading">申请退款</button>
          </div>

          <div v-if="selectedProject.closed && selectedProject.succeeded && selectedProject.earlyLimit > 0" class="action-group">
            <h3>早期捐赠者奖励</h3>
            <p v-if="selectedProject.isEarlyDonor && !selectedProject.hasClaimedReward">
              您是早期捐赠者，可以领取 {{ selectedProject.earlyReward }} ETH 奖励
            </p>
            <p v-else-if="selectedProject.isEarlyDonor && selectedProject.hasClaimedReward">
              您已领取过奖励
            </p>
            <p v-else>您不是早期捐赠者</p>
            <button v-if="selectedProject.isEarlyDonor && !selectedProject.hasClaimedReward"
                    @click="doClaimReward" class="btn" :disabled="loading">领取奖励</button>
          </div>




        </div>

        <div class="donors-section">
          <h3>捐赠者列表</h3>
          <div v-if="donors.length === 0" class="empty">暂无捐赠者</div>
          <table v-else>
            <thead>
              <tr><th>地址</th><th>捐赠金额 (ETH)</th><th>是否早期</th></tr>
            </thead>
            <tbody>
              <tr v-for="d in donors" :key="d.address">
                <td class="addr">{{ d.address.slice(0,6) }}...{{ d.address.slice(-4) }}</td>
                <td>{{ d.amount }}</td>
                <td>{{ d.isEarly ? '是' : '否' }}</td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>
    </div>

    <div v-if="tab==='records'" class="panel">
        <h2>合约交易记录</h2>
        <button @click="loadRecords" class="btn-sm" style="margin-bottom:16px">刷新</button>
        <div class="summary">合约地址: {{ contractAddress }} | 项目总数: {{ records.length }}</div>

        <div v-for="r in records" :key="r.id" class="record-card">
          <div class="project-header">
            <h3>项目 #{{ r.id }}: {{ r.name }}</h3>
            <span :class="['status', r.ongoing ? 'ongoing' : 'ended']">
              {{ r.closed ? '已关闭' : (r.ongoing ? '进行中' : '已结束') }}
            </span>
          </div>
          <p class="desc">{{ r.description }}</p>
          <div class="detail-grid">
            <div class="detail-item"><label>上限</label><span>{{ r.goal }} ETH</span></div>
            <div class="detail-item"><label>已筹</label><span>{{ r.raised }} ETH</span></div>
            <div class="detail-item"><label>截止</label><span>{{ r.deadlineStr }}</span></div>
            <div class="detail-item"><label>创建者</label><span class="addr">{{ r.creator }}</span></div>
            <div class="detail-item"><label>早鸟</label><span>{{ r.earlyCount }}/{{ r.earlyLimit }}</span></div>
            <div class="detail-item"><label>创建者已收款</label><span>{{ r.creatorPaid ? '是' : '否' }}</span></div>
          </div>
          <div class="progress-bar">
            <div class="progress-fill" :style="{width: Math.min(100, r.progress) + '%'}"></div>
          </div>
          <div class="stats"><span>{{ r.raised }} / {{ r.goal }} ETH</span><span>{{ r.progress.toFixed(1) }}%</span></div>

          <h4 style="margin-top:16px">捐赠记录 ({{ r.donations.length }} 笔)</h4>
          <div v-if="r.donations.length === 0" class="empty">暂无捐赠</div>
          <table v-else>
            <thead><tr><th>#</th><th>捐赠者</th><th>金额 (ETH)</th><th>时间</th><th>早鸟</th></tr></thead>
            <tbody>
              <tr v-for="(d, idx) in r.donations" :key="idx">
                <td>{{ idx + 1 }}</td>
                <td class="addr">{{ d.donor.slice(0,6) }}...{{ d.donor.slice(-4) }}</td>
                <td>{{ d.amount }}</td>
                <td>{{ d.time }}</td>
                <td>{{ d.isEarly ? '是' : '否' }}</td>
              </tr>
            </tbody>
          </table>

          <h4 style="margin-top:16px">捐赠者汇总 ({{ r.donorSummary.length }} 人)</h4>
          <table v-if="r.donorSummary.length > 0">
            <thead><tr><th>地址</th><th>总捐赠 (ETH)</th></tr></thead>
            <tbody>
              <tr v-for="ds in r.donorSummary" :key="ds.address">
                <td class="addr">{{ ds.address.slice(0,6) }}...{{ ds.address.slice(-4) }}</td>
                <td>{{ ds.amount }}</td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>

    <div v-if="message" :class="['toast', message.type]">{{ message.text }}</div>
  </div>
</template>

<script>
import { createPublicClient, createWalletClient, custom, http, parseEther, formatEther } from 'viem'
import { CONTRACT_ADDRESS, CONTRACT_ABI } from './config.js'

export default {
  name: 'App',
  data() {
    return {
      account: null,
      balance: '0',
      pending: 0,
      tab: 'list',
      filter: 'all',
      loading: false,
      message: null,
      connectError: '',
      records: [],
      contractAddress: '0xe6e340d132b5f46d1e472debcd681b2abc16e57e',
      projects: [],
      selectedProject: null,
      donors: [],
      donateAmount: '',
      form: {
        name: '',
        description: '',
        goal: '',
        deadline: '',
        earlyLimit: 0,
        earlyReward: 0
      }
    }
  },
  computed: {
    filteredProjects() {
      if (this.filter === 'ongoing') return this.projects.filter(p => p.ongoing)
      if (this.filter === 'ended') return this.projects.filter(p => !p.ongoing)
      return this.projects
    },
    isCreator() {
      return this.selectedProject && this.account &&
        this.selectedProject.creator.toLowerCase() === this.account.toLowerCase()
    }
  },
  methods: {
    getPublicClient() {
      return createPublicClient({ chain: { id: 11155111, name: 'Sepolia', nativeCurrency: { name: 'ETH', symbol: 'ETH', decimals: 18 }, rpcUrls: { default: { http: ['https://sepolia.infura.io/v3/8743927dbd854bcd9adb2c351d6fa476'] } } }, transport: http('https://sepolia.infura.io/v3/8743927dbd854bcd9adb2c351d6fa476') })
    },
    getWalletClient() {
      return createWalletClient({ chain: { id: 11155111, name: 'Sepolia', nativeCurrency: { name: 'ETH', symbol: 'ETH', decimals: 18 }, rpcUrls: { default: { http: ['https://sepolia.infura.io/v3/8743927dbd854bcd9adb2c351d6fa476'] } } }, transport: custom(window.ethereum) })
    },
    async connectWallet() {
      this.connectError = ''
      console.log('connectWallet called')
      if (typeof window.ethereum === 'undefined') {
        this.connectError = '未检测到 MetaMask，请先安装 MetaMask 浏览器插件'
        return
      }
      try {
        console.log('Requesting accounts...')
        const accounts = await window.ethereum.request({ method: 'eth_requestAccounts' })
        console.log('Accounts:', accounts)
        if (!accounts || accounts.length === 0) {
          this.connectError = '未获取到账户，请在 MetaMask 中解锁钱包'
          return
        }
        const chainId = await window.ethereum.request({ method: 'eth_chainId' })
        console.log('Chain ID:', chainId)
        if (chainId !== '0x7a69') {
          this.connectError = '请在 MetaMask 中切换到 Hardhat 网络 (当前 Chain ID: ' + parseInt(chainId, 16) + ', 需要: 31337)'
          this.account = accounts[0]
          return
        }
        this.account = accounts[0]
        await this.updateBalance()
        await this.loadProjects()
        this.showToast('钱包已连接', 'success')
      } catch (e) {
        console.error('connectWallet error:', e)
        this.connectError = '连接失败: ' + (e.message || JSON.stringify(e))
      }
    },
    async updateBalance() {
      try {
        const client = this.getPublicClient()
        const bal = await client.getBalance({ address: this.account })
        this.balance = parseFloat(formatEther(bal)).toFixed(4)
        const pend = await client.readContract({
          address: CONTRACT_ADDRESS,
          abi: CONTRACT_ABI,
          functionName: 'pendingWithdrawal',
          args: [this.account]
        })
        this.pending = parseFloat(formatEther(pend)).toFixed(4)
      } catch (e) {
        console.error('updateBalance error:', e)
      }
    },
    async loadProjects() {
      try {
        const client = this.getPublicClient()
        const count = await client.readContract({
          address: CONTRACT_ADDRESS,
          abi: CONTRACT_ABI,
          functionName: 'projectCount'
        })
        const projects = []
        const now = Math.floor(Date.now() / 1000)
        for (let i = 1; i <= Number(count); i++) {
          const p = await client.readContract({
            address: CONTRACT_ADDRESS,
            abi: CONTRACT_ABI,
            functionName: 'getProject',
            args: [BigInt(i)]
          })
          const deadlinePassed = now >= Number(p.deadline)
          const ongoing = !p.closed && !deadlinePassed && p.raisedAmount < p.goalAmount
          const succeeded = p.raisedAmount >= p.goalAmount
          const goal = parseFloat(formatEther(p.goalAmount))
          const raised = parseFloat(formatEther(p.raisedAmount))
          const remaining = await client.readContract({
            address: CONTRACT_ADDRESS,
            abi: CONTRACT_ABI,
            functionName: 'getRemainingEarlyDonorSlots',
            args: [BigInt(i)]
          })
          const [, , donorCount] = await client.readContract({
            address: CONTRACT_ADDRESS,
            abi: CONTRACT_ABI,
            functionName: 'getProjectDonors',
            args: [BigInt(i), 0n, 1000n]
          })
          projects.push({
            id: i,
            name: p.name,
            description: p.description,
            goal,
            raised,
            deadline: Number(p.deadline),
            deadlineStr: new Date(Number(p.deadline) * 1000).toLocaleString(),
            creator: p.creator,
            closed: p.closed,
            ongoing,
            deadlinePassed,
            succeeded,
            progress: goal > 0 ? (raised / goal) * 100 : 0,
            earlyLimit: Number(p.earlyDonorLimit),
            earlyReward: parseFloat(formatEther(p.earlyDonorReward)),
            earlyCount: Number(p.earlyDonorCount),
            remainingSlots: Number(remaining),
            donorCount: Number(donorCount)
          })
        }
        this.projects = projects.reverse()
      } catch (e) {
        console.error('loadProjects error:', e)
      }
    },
    selectProject(p) {
      this.selectedProject = { ...p }
      this.tab = 'detail'
      this.loadDonors()
      this.checkEarlyDonorStatus()
    },
    async loadDonors() {
      if (!this.selectedProject) return
      try {
        const client = this.getPublicClient()
        const [addrs, amounts] = await client.readContract({
          address: CONTRACT_ADDRESS,
          abi: CONTRACT_ABI,
          functionName: 'getProjectDonors',
          args: [BigInt(this.selectedProject.id), 0n, 1000n]
        })
        const donors = []
        for (let i = 0; i < addrs.length; i++) {
          const isEarly = await client.readContract({
            address: CONTRACT_ADDRESS,
            abi: CONTRACT_ABI,
            functionName: 'isEarlyDonor',
            args: [BigInt(this.selectedProject.id), addrs[i]]
          })
          donors.push({
            address: addrs[i],
            amount: parseFloat(formatEther(amounts[i])).toFixed(4),
            isEarly
          })
        }
        this.donors = donors
      } catch (e) {
        console.error('loadDonors error:', e)
      }
    },
    async checkEarlyDonorStatus() {
      if (!this.selectedProject || !this.account) return
      try {
        const client = this.getPublicClient()
        const isEarly = await client.readContract({
          address: CONTRACT_ADDRESS,
          abi: CONTRACT_ABI,
          functionName: 'isEarlyDonor',
          args: [BigInt(this.selectedProject.id), this.account]
        })
        let hasClaimed = false
        if (isEarly) {
          hasClaimed = await client.readContract({
            address: CONTRACT_ADDRESS,
            abi: CONTRACT_ABI,
            functionName: 'hasClaimedEarlyReward',
            args: [BigInt(this.selectedProject.id), this.account]
          })
        }
        this.selectedProject.isEarlyDonor = isEarly
        this.selectedProject.hasClaimedReward = hasClaimed
      } catch (e) {
        console.error('checkEarlyDonorStatus error:', e)
      }
    },
    setDeadlineSoon() {
      const d = new Date(Date.now() + 60000)
      const y = d.getFullYear()
      const m = String(d.getMonth() + 1).padStart(2, '0')
      const day = String(d.getDate()).padStart(2, '0')
      const h = String(d.getHours()).padStart(2, '0')
      const min = String(d.getMinutes()).padStart(2, '0')
      this.form.deadline = y + '-' + m + '-' + day + 'T' + h + ':' + min
    },
    setDeadlineHour() {
      const d = new Date(Date.now() + 3600000)
      const y = d.getFullYear()
      const m = String(d.getMonth() + 1).padStart(2, '0')
      const day = String(d.getDate()).padStart(2, '0')
      const h = String(d.getHours()).padStart(2, '0')
      const min = String(d.getMinutes()).padStart(2, '0')
      this.form.deadline = y + '-' + m + '-' + day + 'T' + h + ':' + min
    },
    async doCreateProject() {
      this.loading = true
      try {
        const walletClient = this.getWalletClient()
        const deadline = Math.floor(new Date(this.form.deadline).getTime() / 1000)
        const goalWei = parseEther(String(this.form.goal))
        const earlyLimit = BigInt(this.form.earlyLimit || 0)
        const earlyReward = parseEther(String(this.form.earlyReward || 0))
        const hash = await walletClient.writeContract({
          address: CONTRACT_ADDRESS,
          abi: CONTRACT_ABI,
          functionName: 'createProject',
          args: [this.form.name, this.form.description, goalWei, BigInt(deadline), earlyLimit, earlyReward],
          account: this.account
        })
        this.showToast('交易已提交，等待确认...', 'info')
        const publicClient = this.getPublicClient()
        await publicClient.waitForTransactionReceipt({ hash })
        this.showToast('项目创建成功!', 'success')
        await this.loadProjects()
        this.tab = 'list'
      } catch (e) {
        this.showToast('创建失败: ' + (e.message?.slice(0, 100) || e), 'error')
      }
      this.loading = false
    },
    async doDonate() {
      this.loading = true
      try {
        const walletClient = this.getWalletClient()
        const hash = await walletClient.writeContract({
          address: CONTRACT_ADDRESS,
          abi: CONTRACT_ABI,
          functionName: 'donate',
          args: [BigInt(this.selectedProject.id)],
          value: parseEther(String(this.donateAmount)),
          account: this.account
        })
        this.showToast('交易已提交，等待确认...', 'info')
        const publicClient = this.getPublicClient()
        await publicClient.waitForTransactionReceipt({ hash })
        this.showToast('捐赠成功!', 'success')
        this.donateAmount = ''
        await this.loadProjects()
        this.selectProject(this.projects.find(p => p.id === this.selectedProject.id))
        await this.updateBalance()
      } catch (e) {
        this.showToast('捐赠失败: ' + (e.message?.slice(0, 100) || e), 'error')
      }
      this.loading = false
    },
    async doCloseProject() {
      this.loading = true
      try {
        const walletClient = this.getWalletClient()
        const hash = await walletClient.writeContract({
          address: CONTRACT_ADDRESS,
          abi: CONTRACT_ABI,
          functionName: 'closeProject',
          args: [BigInt(this.selectedProject.id)],
          account: this.account
        })
        this.showToast('交易已提交，等待确认...', 'info')
        const publicClient = this.getPublicClient()
        await publicClient.waitForTransactionReceipt({ hash })
        this.showToast('项目已关闭!', 'success')
        await this.loadProjects()
        this.selectProject(this.projects.find(p => p.id === this.selectedProject.id))
        await this.updateBalance()
      } catch (e) {
        this.showToast('关闭失败: ' + (e.message?.slice(0, 100) || e), 'error')
      }
      this.loading = false
    },
    async doRefund() {
      this.loading = true
      try {
        const walletClient = this.getWalletClient()
        const hash = await walletClient.writeContract({
          address: CONTRACT_ADDRESS,
          abi: CONTRACT_ABI,
          functionName: 'refund',
          args: [BigInt(this.selectedProject.id)],
          account: this.account
        })
        this.showToast('交易已提交，等待确认...', 'info')
        const publicClient = this.getPublicClient()
        await publicClient.waitForTransactionReceipt({ hash })
        this.showToast('退款成功!', 'success')
        await this.loadProjects()
        this.selectProject(this.projects.find(p => p.id === this.selectedProject.id))
        await this.updateBalance()
      } catch (e) {
        this.showToast('退款失败: ' + (e.message?.slice(0, 100) || e), 'error')
      }
      this.loading = false
    },
    async doClaimReward() {
      this.loading = true
      try {
        const walletClient = this.getWalletClient()
        const hash = await walletClient.writeContract({
          address: CONTRACT_ADDRESS,
          abi: CONTRACT_ABI,
          functionName: 'claimEarlyDonorReward',
          args: [BigInt(this.selectedProject.id)],
          account: this.account
        })
        this.showToast('交易已提交，等待确认...', 'info')
        const publicClient = this.getPublicClient()
        await publicClient.waitForTransactionReceipt({ hash })
        this.showToast('奖励领取成功!', 'success')
        await this.loadProjects()
        this.selectProject(this.projects.find(p => p.id === this.selectedProject.id))
        await this.updateBalance()
      } catch (e) {
        this.showToast('领取失败: ' + (e.message?.slice(0, 100) || e), 'error')
      }
      this.loading = false
    },

    async doWithdrawPending() {
      this.loading = true
      try {
        const walletClient = this.getWalletClient()
        const hash = await walletClient.writeContract({
          address: CONTRACT_ADDRESS,
          abi: CONTRACT_ABI,
          functionName: 'withdrawPending',
          account: this.account
        })
        this.showToast('交易已提交，等待确认...', 'info')
        const publicClient = this.getPublicClient()
        await publicClient.waitForTransactionReceipt({ hash })
        this.showToast('提取成功!', 'success')
        await this.updateBalance()
      } catch (e) {
        this.showToast('提取失败: ' + (e.message?.slice(0, 100) || e), 'error')
      }
      this.loading = false
    },
    async loadRecords() {
      try {
        const client = this.getPublicClient()
        const count = await client.readContract({ address: CONTRACT_ADDRESS, abi: CONTRACT_ABI, functionName: 'projectCount' })
        const records = []
        for (let i = 1; i <= Number(count); i++) {
          const p = await client.readContract({ address: CONTRACT_ADDRESS, abi: CONTRACT_ABI, functionName: 'getProject', args: [BigInt(i)] })
          const ongoing = await client.readContract({ address: CONTRACT_ADDRESS, abi: CONTRACT_ABI, functionName: 'isProjectOngoing', args: [BigInt(i)] })
          const goal = parseFloat(formatEther(p.goalAmount))
          const raised = parseFloat(formatEther(p.raisedAmount))
          const [donations] = await client.readContract({ address: CONTRACT_ADDRESS, abi: CONTRACT_ABI, functionName: 'getDonations', args: [BigInt(i), 0n, 1000n] })
          const donationList = []
          for (const d of donations) {
            const isEarly = await client.readContract({ address: CONTRACT_ADDRESS, abi: CONTRACT_ABI, functionName: 'isEarlyDonor', args: [BigInt(i), d.donor] })
            donationList.push({ donor: d.donor, amount: parseFloat(formatEther(d.amount)).toFixed(4), time: new Date(Number(d.timestamp) * 1000).toLocaleString(), isEarly })
          }
          const [donors, amounts, donorCount] = await client.readContract({ address: CONTRACT_ADDRESS, abi: CONTRACT_ABI, functionName: 'getProjectDonors', args: [BigInt(i), 0n, 1000n] })
          const donorSummary = []
          for (let j = 0; j < donors.length; j++) { donorSummary.push({ address: donors[j], amount: parseFloat(formatEther(amounts[j])).toFixed(4) }) }
          records.push({ id: i, name: p.name, description: p.description, goal, raised, deadline: Number(p.deadline), deadlineStr: new Date(Number(p.deadline) * 1000).toLocaleString(), creator: p.creator, closed: p.closed, ongoing, creatorPaid: p.creatorPaid, earlyLimit: Number(p.earlyDonorLimit), earlyCount: Number(p.earlyDonorCount), progress: goal > 0 ? (raised / goal) * 100 : 0, donations: donationList, donorSummary })
        }
        this.records = records.reverse()
      } catch (e) { console.error('loadRecords error:', e) }
    },
    showToast(text, type = 'info') {
      this.message = { text, type }
      setTimeout(() => { this.message = null }, 3000)
    }
  },
  async mounted() {
    if (window.ethereum) {
      try {
        const accounts = await window.ethereum.request({ method: 'eth_accounts' })
        if (accounts && accounts.length > 0) {
          this.account = accounts[0]
          await this.updateBalance()
          await this.loadProjects()
        }
      } catch (e) {
        console.error('mounted error:', e)
      }
    }
  }
}
</script>

<style>
* { margin: 0; padding: 0; box-sizing: border-box; }
body { font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif; background: #f5f5f5; color: #333; }
.app { max-width: 960px; margin: 0 auto; padding: 20px; }
header { display: flex; align-items: center; justify-content: space-between; background: #1a1a2e; color: #fff; padding: 16px 24px; border-radius: 12px; margin-bottom: 24px; }
header h1 { font-size: 20px; }
.wallet-info { display: flex; align-items: center; gap: 12px; font-size: 14px; }
.balance { background: #16213e; padding: 4px 12px; border-radius: 8px; }
.pending { background: #e94560; padding: 4px 12px; border-radius: 8px; }
.connect-hint { text-align: center; padding: 80px 20px; background: #fff; border-radius: 12px; }
.connect-hint p { margin: 8px 0; color: #666; }
.error-text { color: #dc3545; font-weight: 500; margin-top: 12px; }
.tabs { display: flex; gap: 8px; margin-bottom: 20px; }
.tabs button { padding: 10px 20px; border: none; background: #fff; border-radius: 8px; cursor: pointer; font-size: 14px; }
.tabs button.active { background: #1a1a2e; color: #fff; }
.panel { background: #fff; border-radius: 12px; padding: 24px; }
.panel h2 { margin-bottom: 20px; }
label { display: block; margin: 12px 0 4px; font-size: 14px; color: #666; }
input, textarea { width: 100%; padding: 10px 12px; border: 1px solid #ddd; border-radius: 8px; font-size: 14px; }
textarea { min-height: 80px; resize: vertical; }
.btn { padding: 10px 24px; background: #1a1a2e; color: #fff; border: none; border-radius: 8px; cursor: pointer; font-size: 14px; margin-top: 12px; }
.btn:hover { background: #16213e; }
.btn:disabled { opacity: 0.5; cursor: not-allowed; }
.btn-sm { padding: 6px 14px; background: #1a1a2e; color: #fff; border: none; border-radius: 6px; cursor: pointer; font-size: 12px; }
.btn-warn { padding: 10px 24px; background: #e94560; color: #fff; border: none; border-radius: 8px; cursor: pointer; font-size: 14px; margin-top: 12px; }
.btn-back { padding: 6px 14px; background: #eee; border: none; border-radius: 6px; cursor: pointer; margin-bottom: 16px; }
.summary { background: #f0f0f0; padding: 8px 12px; border-radius: 8px; margin-top: 8px; font-size: 13px; }
.filter { display: flex; gap: 8px; margin-bottom: 16px; }
.filter button { padding: 6px 14px; border: 1px solid #ddd; background: #fff; border-radius: 6px; cursor: pointer; font-size: 13px; }
.filter button.active { background: #1a1a2e; color: #fff; border-color: #1a1a2e; }
.project-card { background: #fafafa; border: 1px solid #eee; border-radius: 10px; padding: 16px; margin-bottom: 12px; cursor: pointer; transition: box-shadow 0.2s; }
.project-card:hover { box-shadow: 0 2px 12px rgba(0,0,0,0.08); }
.project-header { display: flex; justify-content: space-between; align-items: center; margin-bottom: 8px; }
.project-header h3 { font-size: 16px; }
.desc { color: #666; font-size: 13px; margin-bottom: 12px; }
.status { padding: 3px 10px; border-radius: 12px; font-size: 12px; }
.status.ongoing { background: #d4edda; color: #155724; }
.status.ended { background: #f8d7da; color: #721c24; }
.progress-bar { height: 8px; background: #eee; border-radius: 4px; overflow: hidden; margin-bottom: 8px; }
.progress-bar.large { height: 12px; border-radius: 6px; margin: 16px 0; }
.progress-fill { height: 100%; background: linear-gradient(90deg, #1a1a2e, #e94560); border-radius: 4px; transition: width 0.3s; }
.stats { display: flex; justify-content: space-between; font-size: 13px; color: #666; margin-bottom: 8px; }
.stats.large { font-size: 16px; font-weight: 600; }
.meta { display: flex; gap: 16px; font-size: 12px; color: #999; }
.detail-grid { display: grid; grid-template-columns: 1fr 1fr; gap: 16px; margin: 16px 0; }
.detail-item label { font-size: 12px; color: #999; margin-bottom: 2px; }
.detail-item span { font-size: 14px; font-weight: 500; }
.addr { font-family: monospace; font-size: 12px; }
.actions { margin: 24px 0; }
.action-group { background: #fafafa; border: 1px solid #eee; border-radius: 10px; padding: 16px; margin-bottom: 12px; }
.action-group h3 { font-size: 14px; margin-bottom: 8px; }
.action-group p { font-size: 13px; color: #666; margin-bottom: 8px; }
.donate-form { display: flex; gap: 8px; }
.donate-form input { flex: 1; }
.donors-section { margin-top: 24px; }
.donors-section h3 { margin-bottom: 12px; }
table { width: 100%; border-collapse: collapse; }
th, td { padding: 10px 12px; text-align: left; border-bottom: 1px solid #eee; font-size: 13px; }
th { font-weight: 600; color: #666; }
.empty { text-align: center; padding: 40px; color: #999; }
.success-box { background: #d4edda; border-color: #c3e6cb; }
.record-card { background: #fafafa; border: 1px solid #eee; border-radius: 10px; padding: 20px; margin-bottom: 20px; }
.record-card h4 { color: #666; font-size: 14px; }
.toast { position: fixed; top: 20px; right: 20px; padding: 12px 20px; border-radius: 8px; color: #fff; font-size: 14px; z-index: 1000; animation: slideIn 0.3s; }
.toast.success { background: #28a745; }
.toast.error { background: #dc3545; }
.toast.info { background: #17a2b8; }
@keyframes slideIn { from { transform: translateX(100%); opacity: 0; } to { transform: translateX(0); opacity: 1; } }
</style>
