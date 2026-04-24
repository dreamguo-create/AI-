<template>
  <div class="report-dashboard">

    <t-card title="✨ AI 智能分析调度台 (大屏触控版)" bordered class="control-panel no-print">

      <t-alert
        theme="warning"
        style="margin-bottom: 24px; font-size: 18px;"
        message="数据隐私告知：本系统将采集居民健康档案数据，经加密传输后由内部 AI 引擎进行分析。数据不对外共享，仅用于护理参考。生成报告请在私密环境下操作。"
      />

      <div class="verify-section" v-if="!verifiedElder">
        <div class="verify-title">
          <span class="step-badge">第一步</span>
          输入居民姓名与身份证号进行身份核验
        </div>
        <div class="verify-inputs">
          <div class="verify-field">
            <span class="field-label">居民姓名</span>
            <t-input
              v-model="inputName"
              placeholder="请输入居民真实姓名"
              size="large"
              class="verify-input"
              clearable
              @keyup.enter="verifyElder"
            />
          </div>
          <div class="verify-field">
            <span class="field-label">身份证号</span>
            <t-input
              v-model="inputIdCard"
              placeholder="请输入18位身份证号码"
              size="large"
              class="verify-input"
              clearable
              type="password"
              :show-password-on="{ trigger: 'click' }"
              @keyup.enter="verifyElder"
            />
          </div>
          <t-button
            theme="primary"
            size="large"
            class="verify-btn touch-btn"
            :loading="isVerifying"
            @click="verifyElder"
          >
            <template #icon><span style="margin-right:10px; font-size: 24px;">🔍</span></template>
            {{ isVerifying ? '核验中...' : '查询核验' }}
          </t-button>
        </div>
        <p class="verify-tip">🔒 身份证号码将加密传输，不会被明文记录或展示</p>
      </div>

      <div v-else class="verified-section">
        <div class="verify-title">
          <span class="step-badge success">✅ 身份核验通过</span>
          已定位评估对象，可生成报告
        </div>
        <div class="action-bar">
          <div class="elder-card">
            <div class="elder-avatar">{{ verifiedElder.name.slice(-1) }}</div>
            <div class="elder-info">
              <div class="elder-name">{{ verifiedElder.name }}</div>
              <div class="elder-meta">
                <span>{{ verifiedElder.age }} 岁</span>
                <t-divider layout="vertical" />
                <span>{{ verifiedElder.bed }}</span>
                <t-divider layout="vertical" />
                <span>证件：{{ maskedIdCard }}</span>
              </div>
            </div>
            <t-button
              theme="default"
              size="small"
              style="margin-left: auto;"
              @click="resetVerify"
            >重新核验</t-button>
          </div>

          <t-button
            theme="primary"
            size="large"
            class="generate-btn touch-btn"
            :loading="isGenerating"
            @click="generateReport"
          >
            <template #icon><span style="margin-right:12px; font-size: 28px;">🧠</span></template>
            {{ isGenerating ? 'AI 深度分析建模中...' : '一键生成分析报告' }}
          </t-button>
        </div>
      </div>

    </t-card>

    <t-card class="report-display-panel" bordered>

      <div v-if="isGenerating" class="status-container no-print">
        <div class="scanning-effect"></div>
        <div style="font-size: 24px; color: #0052D9; margin-top: 30px; font-weight: bold;">
          正在读取核心体征数据并进行 AI 语义建模，请稍候...
        </div>
      </div>

      <div v-else-if="!reportContent" class="status-container empty-state no-print">
        <div style="font-size: 100px; margin-bottom: 40px;">📄</div>
        <h3 style="color: #666; font-size: 32px; font-weight: bold;">暂无报告数据</h3>
        <p style="color: #999; font-size: 22px; margin-top: 15px;">请在上方选择一位长辈，触摸点击生成报告</p>
      </div>

      <div v-else class="paper-report-container custom-scrollbar">
        <div class="a4-paper" id="print-area">

          <div class="report-actions no-print">
            <div class="action-icon-btn" @click="toggleFullscreen" title="全屏阅读">
              <span class="icon-gap">{{ isFullscreen ? '⊠' : '⛶' }}</span>
              <span class="btn-label">{{ isFullscreen ? '退出全屏' : '全屏阅读' }}</span>
            </div>
            <div :class="['action-icon-btn', 'speech-btn', isSpeaking ? 'is-speaking' : '']" @click="handleSpeechToggle">
              <span class="icon-gap">{{ isSpeaking ? '⏹️' : '🔊' }}</span>
              <span class="btn-label">{{ isSpeaking ? '停止播报' : '播报报告' }}</span>
            </div>
          </div>

          <div class="paper-header">
            <h2>颐云智慧养老 · AI 健康分析评估报告</h2>
            <div class="meta-info">
              <span>生成时间：{{ currentTime }}</span>
              <span>分析引擎：内部健康评估模型 v3</span>
            </div>
          </div>

          <t-divider dashed />

          <div class="paper-content" v-html="sanitizedReport"></div>

          <div class="paper-footer">
            <p>【免责声明】本报告由系统客观数据与大语言模型自动生成，仅供护理参考，不可替代专业医疗诊断。</p>
            <p class="print-only confidential-mark">
              【机密文件】仅限授权护理人员查阅 · 打印人：{{ operatorName }} · 打印时间：{{ currentTime }}
            </p>
          </div>
        </div>

        <div class="bottom-actions no-print">
          <t-button theme="default" size="large" class="action-btn touch-btn" @click="stopSpeechAndReset">
            <template #icon><span style="margin-right:10px; font-size: 26px;">🔄</span></template>
            重新评估
          </t-button>

          <t-button theme="success" size="large" class="action-btn touch-btn" @click="printReport">
            <template #icon><span style="margin-right:10px; font-size: 26px;">🖨️</span></template>
            打印报告
          </t-button>
        </div>
      </div>

    </t-card>

    <transition name="fullscreen-fade">
      <div v-if="isFullscreen" class="fullscreen-overlay" @click.self="toggleFullscreen">
        <div class="fullscreen-paper custom-scrollbar">
          <div class="fullscreen-close" @click="toggleFullscreen">✕ 退出全屏</div>
          <div class="fullscreen-tts no-print">
            <div :class="['action-icon-btn', 'speech-btn', isSpeaking ? 'is-speaking' : '']" @click="handleSpeechToggle">
              <span class="icon-gap">{{ isSpeaking ? '⏹️' : '🔊' }}</span>
              <span class="btn-label">{{ isSpeaking ? '停止播报' : '播报报告' }}</span>
            </div>
          </div>

          <div class="paper-header">
            <h2>颐云智慧养老 · AI 健康分析评估报告</h2>
            <div class="meta-info">
              <span>生成时间：{{ currentTime }}</span>
              <span>分析引擎：内部健康评估模型 v3</span>
            </div>
          </div>
          <t-divider dashed />
          <div class="paper-content" v-html="sanitizedReport"></div>
          <div class="paper-footer">
            <p>【免责声明】本报告由系统客观数据与大语言模型自动生成，仅供护理参考，不可替代专业医疗诊断。</p>
          </div>
        </div>
      </div>
    </transition>

    <t-dialog
      v-model:visible="showSpeechConfirm"
      header="🔊 播报前请确认"
      :on-confirm="startSpeech"
      confirm-btn="确认播报"
      cancel-btn="取消"
      :z-index="10005"
    >
      <p style="font-size: 20px; line-height: 2;">
        报告内容涉及居民 <strong>健康隐私</strong>，语音播报将在当前环境中大声朗读。<br/>
        请确认您当前处于 <strong>私密环境</strong>，周围无无关人员。
      </p>
    </t-dialog>

  </div>
</template>

<script setup lang="ts">
import { ref, computed, onMounted, onBeforeUnmount } from 'vue'
import { MessagePlugin } from 'tdesign-vue-next'
import axios from 'axios'
import DOMPurify from 'dompurify'

// ==================== 响应式状态 ====================

const inputName     = ref('')
const inputIdCard   = ref('')
const isVerifying   = ref(false)
const elderList     = ref<{ id: number; name: string; age: string; bed: string; idCardNo: string }[]>([])
const verifiedElder = ref<{ id: number; name: string; age: string; bed: string } | null>(null)

const maskedIdCard = computed(() => {
  const raw = inputIdCard.value.replace(/\s/g, '')
  if (raw.length < 10) return raw
  return raw.slice(0, 6) + '********' + raw.slice(-4)
})

const isGenerating = ref(false)
const reportContent = ref('')
const isSpeaking = ref(false)
const showSpeechConfirm = ref(false)
const isFullscreen = ref(false)

const operatorName = computed(() => {
  try {
    const userStr = localStorage.getItem('userInfo')
    if (userStr) return JSON.parse(userStr)?.name || '未知操作员'
  } catch { /* ignore */ }
  return '未知操作员'
})

const currentTime = computed(() => {
  const now = new Date()
  return `${now.getFullYear()}-${String(now.getMonth() + 1).padStart(2, '0')}-${String(now.getDate()).padStart(2, '0')} ${String(now.getHours()).padStart(2, '0')}:${String(now.getMinutes()).padStart(2, '0')}`
})

// ==================== 🌟 修复 #2：通过身份证号精准计算实际年龄 ====================
const getAgeFromIdCard = (idCard: string): string => {
  if (!idCard || idCard.length !== 18) return '未知'
  // 提取身份证里的年份(6-9位)、月份(10-11位)、日期(12-13位)
  const year = parseInt(idCard.substring(6, 10))
  const month = parseInt(idCard.substring(10, 12))
  const day = parseInt(idCard.substring(12, 14))
  
  const now = new Date()
  let age = now.getFullYear() - year
  // 如果当前月份小于出生月份，或者同月但当前日期小于出生日期，周岁减一
  if (now.getMonth() + 1 < month || (now.getMonth() + 1 === month && now.getDate() < day)) {
    age--
  }
  return age > 0 ? String(age) : '未知'
}

// ==================== 报告格式化 + XSS 清洗 ====================
const formattedReport = computed(() => {
  if (!reportContent.value) return ''

  let text = reportContent.value
    .replace(/\\n/g, '\n')
    .replace(/\\"/g, '"')

  text = text.replace(/【(.*?)】/g,
    '<h3 style="color: #0052D9; margin-top: 60px; margin-bottom: 30px; border-left: 8px solid #0052D9; padding-left: 20px; font-size: 28px; font-weight: 900; letter-spacing: 1px;">$1</h3>'
  )

  text = text.replace(/\*\*(.*?)\*\*/g,
    '<strong style="color: #111; font-weight: 800; background-color: #f0f3f7; padding: 6px 12px; border-radius: 8px; margin: 0 6px; display: inline-block;">$1</strong>'
  )

  // ⭐ 核心新增：无情干掉所有残留的单个星号 (*) 及其后面的多余空格
  text = text.replace(/\*\s*/g, '')

  text = text.replace(/\n/g, '<br/>')
  return text
})

const sanitizedReport = computed(() => {
  if (!formattedReport.value) return ''
  return DOMPurify.sanitize(formattedReport.value, {
    ALLOWED_TAGS: ['br', 'strong', 'h3', 'p', 'span', 'em', 'ul', 'li', 'ol'],
    ALLOWED_ATTR: ['style']
  })
})

// ==================== TTS 语音播报 ====================
let speechInstance: SpeechSynthesisUtterance | null = null
let sentenceQueue: string[] = []
let sentenceIndex = 0

const handleSpeechToggle = () => {
  if (isSpeaking.value) {
    window.speechSynthesis.cancel()
    isSpeaking.value = false
    sentenceQueue = []
    sentenceIndex = 0
    return
  }
  // 点击播放时，弹出隐私确认（因为加了 z-index 10005，全屏下也能弹出来）
  showSpeechConfirm.value = true
}

const pickChineseVoice = (): SpeechSynthesisVoice | null => {
  const voices = window.speechSynthesis.getVoices()
  const priority = [
    'Microsoft Xiaoxiao Online',
    'Microsoft Xiaoxiao',
    'Microsoft XiaoXiao',
    'Google 普通话（中国大陆）',
    'Ting-Ting',
    'Meijia',
    'Microsoft Huihui',
  ]
  for (const name of priority) {
    const v = voices.find(v => v.name.includes(name))
    if (v) return v
  }
  return voices.find(v => v.lang.startsWith('zh') && /female|woman/i.test(v.name))
    || voices.find(v => v.lang.startsWith('zh'))
    || null
}

const splitToSentences = (text: string): string[] => {
  return text.split(/(?<=[。！？；…])\s*/).filter(s => s.trim().length > 0)
}

const speakNextSentence = () => {
  if (sentenceIndex >= sentenceQueue.length) {
    isSpeaking.value = false
    speechInstance = null
    return
  }

  const text = sentenceQueue[sentenceIndex++]
  speechInstance = new SpeechSynthesisUtterance(text)
  speechInstance.lang   = 'zh-CN'
  speechInstance.rate   = 0.85
  speechInstance.pitch  = 1.05
  speechInstance.volume = 1.0

  const voice = pickChineseVoice()
  if (voice) speechInstance.voice = voice

  speechInstance.onend   = () => { speechInstance = null; speakNextSentence() }
  speechInstance.onerror = (e) => {
    if ((e as any).error !== 'interrupted') {
      isSpeaking.value = false
    }
  }
  window.speechSynthesis.speak(speechInstance)
}

const startSpeech = () => {
  showSpeechConfirm.value = false

  // 1. 深度唤醒引擎（极其关键：不仅要清空，还要强制重置状态）
  window.speechSynthesis.cancel()
  window.speechSynthesis.resume()

  // 2. 极致文本清洗，增加“呼吸感”
  const pureText = reportContent.value
    .replace(/【/g, '')
    .replace(/】/g, '，')     // ⭐ 优化1：把标题括号换成逗号，让 AI 读完标题后有个换气的停顿，更自然
    .replace(/\*\*/g, '')
    .replace(/\\n/g, '。')
    .replace(/\n/g, '。')
    .replace(/<[^>]+>/g, '')
    .replace(/([。！？；])\1+/g, '$1') // ⭐ 优化2：去除连续重复的标点（比如。。。），防乱读
    .replace(/，，+/g, '，')

  // 3. 完美规避 15 秒 Bug 的切分法
  const sentences = pureText
    .split(/(?<=[。！？；])/)
    .map(s => s.trim())
    .filter(s => s.length > 2)

  if (sentences.length === 0) return

  isSpeaking.value = true
  const voice = pickChineseVoice()

  // 🔥 4. 原生队列魔法：不等待，一瞬间全推给 C++ 底层！
  sentences.forEach((text, index) => {
    const utterance = new SpeechSynthesisUtterance(text)
    utterance.lang = 'zh-CN'
    utterance.rate = 0.9    // 适老化流畅语速
    utterance.pitch = 1.05  // 略微提高音调，声音更明亮精神
    utterance.volume = 1

    if (voice) utterance.voice = voice

    // 只有最后一句读完，才关闭喇叭动画
    utterance.onend = () => {
      if (index === sentences.length - 1) {
        isSpeaking.value = false
      }
    }

    utterance.onerror = (e) => {
      if (e.error !== 'interrupted') isSpeaking.value = false
    }

    // ⭐ 核心：直接遍历执行 speak。底层系统会自动接管队列，实现 0 卡顿的无缝衔接
    window.speechSynthesis.speak(utterance)
  })
}
const stopSpeechAndReset = () => {
  window.speechSynthesis.cancel()
  isSpeaking.value = false
  speechInstance = null
  sentenceQueue = []
  sentenceIndex = 0
  reportContent.value = ''
}

onBeforeUnmount(() => {
  window.speechSynthesis.cancel()
  document.body.style.overflow = ''
  if (speechInstance) {
    speechInstance.onstart = null
    speechInstance.onend = null
    speechInstance.onerror = null
    speechInstance = null
  }
})

const printReport = () => { window.print() }

const toggleFullscreen = () => {
  isFullscreen.value = !isFullscreen.value
  document.body.style.overflow = isFullscreen.value ? 'hidden' : ''
}

// ==================== 居民列表预加载 ====================
const fetchElderList = async () => {
  try {
    const baseURL = (import.meta as any).env.VITE_APP_BASE_API || '/api'
    const token   = (localStorage.getItem('token') || '').replace(/"/g, '')
    const res = await axios.get(baseURL + '/elder/selectList', {
      headers: { Authorization: 'Bearer ' + token }
    })
    if (res.data.code === 200 || res.data.code === 1) {
      elderList.value = (res.data.data || []).map((item: any) => {
        const idCardNo = item.idCardNo || item.id_card_no || ''
        return {
          id:       item.id,
          name:     item.name        || '',
          // 使用身份证重新验证年龄，如果算不出再用后端的兜底
          age:      idCardNo && idCardNo.length === 18 ? getAgeFromIdCard(idCardNo) : String(item.age ?? item.elderAge ?? '未知'),
          bed:      item.bedNumber || item.bed_number || item.bedNo || '暂无',
          idCardNo: idCardNo
        }
      })
    }
  } catch (e) {
    console.error('[elderList] 加载失败：', e)
    elderList.value = []
  }
}

onMounted(() => { fetchElderList() })

// ==================== 身份核验 ====================

const isValidIdCard = (id: string) => /^\d{17}[\dXx]$/.test(id.trim())

const verifyElder = async () => {
  const name   = inputName.value.trim()
  const idCard = inputIdCard.value.trim().toUpperCase()

  if (!name)   { MessagePlugin.warning('请输入居民姓名'); return }
  if (!idCard) { MessagePlugin.warning('请输入身份证号码'); return }
  if (!isValidIdCard(idCard)) {
    MessagePlugin.error('身份证号码格式不正确（需18位，末位可为X），请重新输入')
    return
  }

  isVerifying.value = true
  await new Promise(r => setTimeout(r, 400))

  try {
    const matched = elderList.value.find(
      e => e.name === name && e.idCardNo.toUpperCase() === idCard
    )

    if (matched) {
      // 🌟 既然匹配成功了，用手敲进去的、完美的 18 位身份证来精确算岁数
      const exactAge = getAgeFromIdCard(idCard)
      
      verifiedElder.value = {
        id:   matched.id,
        name: matched.name,
        age:  exactAge !== '未知' ? exactAge : matched.age, // 精准覆盖
        bed:  matched.bed
      }
      MessagePlugin.success(`✅ 核验通过，已找到居民：${matched.name}`)
    } else {
      if (elderList.value.length === 0) {
        MessagePlugin.error('居民数据加载失败，请刷新页面后重试')
      } else {
        MessagePlugin.error('未找到匹配居民，请确认姓名与身份证号是否正确')
      }
    }
  } finally {
    isVerifying.value = false
  }
}

const resetVerify = () => {
  verifiedElder.value = null
  inputName.value     = ''
  inputIdCard.value   = ''
  reportContent.value = ''
  isFullscreen.value  = false
  document.body.style.overflow = ''
  window.speechSynthesis.cancel()
  isSpeaking.value    = false
  speechInstance      = null
  sentenceQueue       = []
  sentenceIndex       = 0
}

// ==================== 生成报告 ====================
const generateReport = async () => {
  if (!verifiedElder.value) {
    MessagePlugin.warning('请先完成居民身份核验！')
    return
  }
  isGenerating.value = true
  reportContent.value = ''

  try {
    const baseURL = (import.meta as any).env.VITE_APP_BASE_API || '/api'
    const token = (localStorage.getItem('token') || '').replace(/"/g, '')
    const res = await axios.post(
      baseURL + '/report/generate',
      { elderId: verifiedElder.value.id },
      { timeout: 60000, headers: { Authorization: 'Bearer ' + token } }
    )
    reportContent.value = res.data.data || res.data.msg || ''
    MessagePlugin.success('✨ 分析完成！可以点右上角播放，请注意保护居民隐私。')

  } catch (e: any) {
    if (e?.response?.status === 401) {
      MessagePlugin.error('登录已过期，请重新登录后再试')
    } else if (e?.response?.status === 404) {
      MessagePlugin.error('接口地址不存在，请联系管理员检查配置')
    } else if (e?.code === 'ECONNABORTED') {
      MessagePlugin.warning('AI 分析超时（超过 60 秒），请稍后重试')
    } else if (!navigator.onLine) {
      MessagePlugin.error('当前网络已断开，请检查网络连接')
    } else {
      MessagePlugin.error(`服务异常（${e?.response?.status ?? '未知错误'}），请联系管理员`)
    }
  } finally {
    isGenerating.value = false
  }
}
</script>

<style scoped>
/* ==================== 整体布局 ==================== */
.report-dashboard {
  padding: 40px;
  background-color: #f3f5f8;
  min-height: calc(100vh - 60px);
}

.control-panel {
  margin-bottom: 24px;
  border-radius: 16px;
  box-shadow: 0 6px 16px rgba(0, 0, 0, 0.06);
}

/* ==================== 身份核验区 ==================== */
.verify-section, .verified-section {
  padding: 10px 0 6px;
}

.verify-title {
  font-size: 22px;
  font-weight: bold;
  color: #333;
  margin-bottom: 24px;
  display: flex;
  align-items: center;
  gap: 12px;
}

.step-badge {
  display: inline-block;
  background: #0052D9;
  color: #fff;
  font-size: 18px;
  font-weight: bold;
  padding: 4px 16px;
  border-radius: 20px;
  flex-shrink: 0;
}

.step-badge.success {
  background: #00a870;
}

.verify-inputs {
  display: flex;
  align-items: center;
  flex-wrap: wrap;
  gap: 20px;
  margin-bottom: 16px;
}

.verify-field {
  display: flex;
  align-items: center;
  gap: 12px;
  flex: 1 1 280px;
  min-width: 0;
}

.field-label {
  font-size: 20px;
  font-weight: bold;
  color: #111;
  white-space: nowrap;
  flex-shrink: 0;
}

.verify-input :deep(.t-input) {
  height: 64px !important;
  font-size: 20px;
  border-radius: 10px;
}

.verify-btn {
  height: 68px !important;
  font-size: 22px;
  border-radius: 12px;
  padding: 0 36px;
  font-weight: bold;
  flex-shrink: 0;
}

.verify-tip {
  font-size: 17px;
  color: #888;
  margin: 0;
  padding-left: 4px;
}

/* 核验通过后的居民信息卡 */
.elder-card {
  display: flex;
  align-items: center;
  gap: 16px;
  background: linear-gradient(135deg, #e8f0ff 0%, #f0f5ff 100%);
  border: 2px solid #b3c9ff;
  border-radius: 14px;
  padding: 14px 22px;
  flex: 1 1 auto;
  min-width: 0;
}

.elder-avatar {
  width: 52px;
  height: 52px;
  border-radius: 50%;
  background: #0052D9;
  color: white;
  font-size: 22px;
  font-weight: 900;
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
}

.elder-info { flex: 1; min-width: 0; }

.elder-name {
  font-size: 22px;
  font-weight: 900;
  color: #111;
  margin-bottom: 4px;
}

.elder-meta {
  display: flex;
  align-items: center;
  font-size: 16px;
  color: #555;
  gap: 4px;
  flex-wrap: wrap;
}

/* action-bar（核验通过后显示）*/
.action-bar {
  display: flex;
  justify-content: space-between;
  align-items: center;
  flex-wrap: wrap;
  gap: 20px;
}

.generate-btn {
  height: 68px !important;
  font-size: 24px;
  border-radius: 12px;
  padding: 0 40px;
  font-weight: bold;
  letter-spacing: 2px;
  flex-shrink: 0;
}

/* ==================== 报告区 ==================== */
.report-display-panel {
  border-radius: 16px;
  box-shadow: 0 6px 16px rgba(0, 0, 0, 0.06);
}

.status-container {
  height: 700px;
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
}

.scanning-effect {
  width: 160px;
  height: 160px;
  background: url('https://img.icons8.com/color/144/000000/artificial-intelligence.png') center/cover;
  position: relative;
  margin-bottom: 50px;
}

.scanning-effect::after {
  content: '';
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  height: 8px;
  background: #0052D9;
  box-shadow: 0 0 20px #0052D9;
  animation: scan 2s linear infinite;
}

@keyframes scan {
  0%   { top: -10px; }
  100% { top: 170px; }
}

.custom-scrollbar {
  min-height: 500px;
  max-height: calc(100vh - 380px);
  overflow-y: auto;
  padding: 40px 0;
  background-color: #e5e8ec;
  border-radius: 16px;
  display: flex;
  flex-direction: column;
  align-items: center;
}

.custom-scrollbar::-webkit-scrollbar { width: 16px; }
.custom-scrollbar::-webkit-scrollbar-thumb { background: #b0b4bb; border-radius: 8px; }
.custom-scrollbar::-webkit-scrollbar-track { background: rgba(0, 0, 0, 0.01); }

/* ==================== A4 纸样式 ==================== */
.a4-paper {
  width: 100%;
  max-width: 960px;
  background-color: #ffffff;
  padding: 80px 100px;
  box-shadow: 0 20px 50px rgba(0, 0, 0, 0.12);
  position: relative;
  margin-bottom: 50px;
  flex-shrink: 0;
  border-radius: 12px;
}

@keyframes pulse {
  0%   { box-shadow: 0 0 0 0 rgba(227, 77, 89, 0.7); }
  70%  { box-shadow: 0 0 0 14px rgba(227, 77, 89, 0); }
  100% { box-shadow: 0 0 0 0 rgba(227, 77, 89, 0); }
}

.paper-header { 
  text-align: center; 
  margin-bottom: 40px; 
  padding-top: 30px; 
}

.paper-header h2 {
  font-size: 38px;
  color: #111;
  letter-spacing: 4px;
  margin-bottom: 25px;
  font-weight: 900;
}

.meta-info {
  display: flex;
  justify-content: space-between;
  color: #444;
  font-size: 20px;
  border-bottom: 4px solid #111;
  padding-bottom: 20px;
}

.paper-content {
  font-size: 22px;
  line-height: 2.8;
  color: #222;
  min-height: 500px;
  text-align: justify;
  letter-spacing: 0.5px;
}

.paper-footer {
  margin-top: 100px;
  padding-top: 30px;
  border-top: 3px dashed #ddd;
  font-size: 18px;
  color: #777;
  text-align: center;
}

.confidential-mark {
  display: none;
  color: #c00;
  font-weight: bold;
  margin-top: 12px;
}

.bottom-actions {
  display: flex;
  gap: 40px;
  margin-bottom: 40px;
  flex-shrink: 0;
}

.action-btn {
  width: 260px;
  height: 64px !important;
  font-size: 22px;
}

/* ==================== 全屏阅读模式 ==================== */
.fullscreen-overlay {
  position: fixed;
  inset: 0;
  z-index: 9999;
  background: rgba(20, 24, 35, 0.92);
  display: flex;
  justify-content: center;
  align-items: flex-start;
  overflow-y: auto;
  padding: 40px 20px 60px;
}

.fullscreen-paper {
  width: 100%;
  max-width: 1100px;
  background: #fff;
  border-radius: 20px;
  padding: 80px 100px;
  position: relative;
  box-shadow: 0 40px 80px rgba(0,0,0,0.5);
  max-height: none !important;
  min-height: auto !important;
  overflow-y: visible !important;
}

.fullscreen-paper .paper-content {
  font-size: 26px;
  line-height: 3;
}

.fullscreen-paper .paper-header h2 {
  font-size: 42px;
}

.fullscreen-close {
  position: absolute;
  top: 28px;
  right: 36px;
  cursor: pointer;
  background: #f3f5f8;
  color: #444;
  font-size: 18px;
  font-weight: bold;
  padding: 8px 20px;
  border-radius: 30px;
  transition: all 0.2s;
  user-select: none;
}
.fullscreen-close:hover { background: #e34d59; color: #fff; }

.fullscreen-tts {
  position: absolute;
  top: 28px;
  right: 150px;
}

/* ======== 右上角按钮组 ======== */
.report-actions {
  position: absolute;
  top: 30px;
  right: 40px;
  display: flex;
  gap: 12px;
  z-index: 100;
}

.action-icon-btn {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 8px 18px;
  border-radius: 24px;
  font-size: 16px;
  font-weight: bold;
  cursor: pointer;
  background: #0052D9;
  color: white;
  box-shadow: 0 4px 12px rgba(0,82,217,0.25);
  transition: all 0.25s;
  user-select: none;
  white-space: nowrap;
}
.action-icon-btn:hover { transform: scale(1.05); background: #003eb3; }
.action-icon-btn .btn-label { font-size: 16px; }
.icon-gap { font-size: 18px; }

.speech-btn.is-speaking {
  background: #e34d59;
  animation: pulse 1.5s infinite;
}

.fullscreen-fade-enter-active,
.fullscreen-fade-leave-active { transition: opacity 0.3s ease; }
.fullscreen-fade-enter-from,
.fullscreen-fade-leave-to { opacity: 0; }

/* ==================== 打印样式 ==================== */
@media print {
  .no-print { display: none !important; }

  .confidential-mark { display: block !important; }
  .print-only { display: block !important; }

  .report-dashboard,
  .custom-scrollbar {
    background: white !important;
    padding: 0 !important;
    height: auto !important;
    overflow: visible !important;
    max-height: none !important;
  }

  .a4-paper {
    box-shadow: none !important;
    padding: 0 !important;
    max-width: 100% !important;
    margin: 0 !important;
  }
}
</style>
