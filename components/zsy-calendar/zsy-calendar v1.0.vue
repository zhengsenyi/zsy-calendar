<template>
	<!-- 日历滚动插件 -->
	<view class="zsy_calendar">
		<!-- 日历顶部信息 -->
		<view class="calendar_info">
			<text class="title">每日记录</text>
			<text class="desc">
				({{ getAssignDateInfo(false, 0) === getAssignDateInfo(true, 0) ? '' : getAssignDateInfo(false, 0) + '年' }}{{ getAssignDateInfo(false, 1) }}月)
			</text>
			<text v-show="showBackToTodayBtn" class="backToToday" :style="{color: dateActiveColor}" @tap="goToDate()">回到今天</text>
		</view>
		
		<!-- 日历周数 -->
		<view class="calendar_week">
			<view v-for="(item, index) in week" :key="index" class="calendar_week__item">{{ item }}</view>
		</view>
		
		<!-- 日历轮播 -->
		<view class="calendar_swiper">
			<!-- 展开情况下的日历轮播 -->
			<swiper
				v-if="swiperMode === 'open'"
				key="normalSwiper"
				circular
				:style="{height: swiperHeight('open')}"
				:current="current"
				:duration="duration"
				:skip-hidden-item-layout="true"
				@change="e => current = e.detail.current"
			>
				<swiper-item v-for="(swiper, swiperIndex) in 3" :key="swiperIndex" class="swiper-item">
					<DateBox
						:dates="calendarSwiperDates[swiperIndex]"
						:cellHeight="cellHeight"
						:dateActiveColor="dateActiveColor"
						:swiperMode="swiperMode"
						@chooseDate="chooseDate"
					/>
				</swiper-item>
			</swiper>
			
			<!-- 收缩情况下的日历轮播 -->
			<swiper
				v-else
				key="shrinkSwiper"
				circular
				:style="{height: swiperHeight('close')}"
				:current="shrinkCurrent"
				:duration="duration"
				:skip-hidden-item-layout="true"
				@change="e => shrinkCurrent = e.detail.current"
			>
				<swiper-item v-for="(swiper, swiperIndex) in 3" :key="swiperIndex" class="swiper-item">
					<DateBox
						:dates="calendarSwiperShrinkDates[swiperIndex]"
						:cellHeight="cellHeight"
						:dateActiveColor="dateActiveColor"
						:swiperMode="swiperMode"
						@chooseDate="chooseShrinkDate"
					/>
				</swiper-item>
			</swiper>
		</view>
	
		<!-- 日历切换模式 -->
		<view class="calendar_toggle" @tap="swiperMode = swiperMode === 'open' ? 'close' : 'open'">
			<view class="icon" :class="{down: swiperMode === 'close'}"></view>
		</view>
	</view>
</template>

<script>
	import { parseTime, deepClone } from './js/utils.js'
	import DateBox from './dateBox.vue'
	export default {
		name: 'ZsyCalendar',
		components: {
			DateBox
		},
		props: {
			duration: { // 轮播图动画时长
				type: Number,
				default: 300
			},
			cellHeight: { // 一列的高度
				type: Number,
				default: 75
			},
			dateActiveColor: { // 日期选中颜色
				type: String,
				default: '#FE6601'
			},
			sundayIndex: { // 星期天所在索引，0表示第一个、6表示最后一个
				type: Number,
				default: 6
			},
			mode: { // 日历模式
				type: String,
				default: 'open'
			},
			changeSetDefault: { // 月份切换时是否显示一号还是当前月份选中高亮
				type: Boolean,
				default: true
			},
			defaultSelectedDate: { // 默认选中日期
				type: String | null,
				default: null
			},
			showArrowBtn: { // 是否显示左右切换按钮
				type: Boolean,
				default: true
			}
		},
		data() {
			return {
				today: parseTime(new Date(), '{y}-{m}-{d}'), // 今天日期
				selectedDate: null, // 选中日期
				week: [], // 日历周数
				current: 1, // 当前日历轮播默认显示索引
				shrinkCurrent: 1, // 缩放日历轮播默认显示索引
				calendarSwiperDates: [], // 日历轮播日期信息
				calendarSwiperShrinkDates: [], // 日历轮播收缩时的日期信息
				dateActive: -1, // 日期选中索引
				swiperByClick: false, // 是否通过点击上月份或下月份的日期进行轮播切换
				shrinkSwiperByClick: false, // 是否通过点击上月份或下月份的日期进行收缩日历的轮播切换
				swiperMode: this.mode, // 日历轮播显示模式 open：展开 close：收缩
				dateCache: {}, // 日期缓存
				emitTimer: null, // 日期改变向父级传递当前选中日期计时器
				dateClick: false // 是否进行了日期的点击选择
			}
		},
		computed: {
			/* 获取指定日期信息
			isCurDate: 是否获取当天的信息还是选中日期的信息
			index: 0 表示年份 1 表示月份 2 表示日期 */
			getAssignDateInfo() {
				return (isCurDate, index) => {
					return (isCurDate ? this.today : this.selectedDate).split('-')[index] * 1
				}
			},
			// 是否显示回到今天按钮
			showBackToTodayBtn() {
				return this.getAssignDateInfo(false, 0) !== this.getAssignDateInfo(true, 0) || this.getAssignDateInfo(false, 1) !== this.getAssignDateInfo(true, 1)
			},
			// 返回轮播图高度
			swiperHeight() {
				return (swiperMode) => {
					const normalHeight = (this.calendarSwiperDates[this.current] || []).length / 7 * (this.cellHeight + 20) + 'rpx'
					const shrinkHeight = this.cellHeight + 20 + 'rpx'
					return swiperMode === 'open' ? normalHeight : shrinkHeight
				}
			}
		},
		watch: {
			// 展开日历轮播切换
			current(newV, oldV) {
				if (newV === 0 && oldV === 2) { // 右滑
					this.swiperChange(1)
					return
				}
				if (newV === 2 && oldV === 0) { // 左滑
					this.swiperChange(-1)
					return
				}
				if (newV > oldV) { // 右滑
					this.swiperChange(1)
				} else { // 左滑
					this.swiperChange(-1)
				}
			},
			// 收缩日历轮播切换
			shrinkCurrent(newV, oldV) {
				if (newV === 0 && oldV === 2) { // 右滑
					this.shrinkSwiperChange(1)
					return
				}
				if (newV === 2 && oldV === 0) { // 左滑
					this.shrinkSwiperChange(-1)
					return
				}
				if (newV > oldV) { // 右滑
					this.shrinkSwiperChange(1)
				} else { // 左滑
					this.shrinkSwiperChange(-1)
				}
			},
			// 日历显示方式切换
			swiperMode(newV) {
				// 当收缩时初始化收缩轮播图的日期数据
				if (newV === 'close') {
					this.initCalendarShrinkSwiperDates()
				}
			},
			selectedDate: {
				deep: true,
				handler(newV, oldV) {
					if (newV && (oldV === null || this.dateClick)) { // 初始化/日历点击选择时直接返回
						this.emitDate()
						this.dateClick = false
					} else { // 其它情况做防抖处理
						if (this.emitTimer !== null) {
							clearTimeout(this.emitTimer)
						}
						this.emitTimer = setTimeout(() => {
							this.emitDate()
						}, this.duration + 200)
					}
				}
			}
		},
		created() {
			this.init() // 初始化数据
		},
		methods: {
			// 初始化数据
			init() {
				if (this.selectedDate === null) { // 默认选中日期为当天
					this.selectedDate = this.defaultSelectedDate || this.today
				}
				this.initWeek() // 初始化要显示的周数
				this.initCalendarSwiperDates() // 初始化日历轮播日期信息
				// 解决swiperMode初始化为收缩时没有初始化日历收缩轮播日期信息
				if (this.swiperMode === 'close') {
					this.initCalendarShrinkSwiperDates()
				}
			},
			// 初始化周数
			initWeek() {
				const normalWeek = ['日', '一', '二', '三', '四', '五', '六'] // 正常周数
				const sIndex = this.sundayIndex < 0 ? 0 : this.sundayIndex >= normalWeek.length ? normalWeek.length - 1 : this.sundayIndex
				normalWeek.unshift(...normalWeek.slice(-sIndex))
				normalWeek.length = 7
				this.week = normalWeek
			},
			// 初始化展开时的日历轮播日期信息
			initCalendarSwiperDates(cb) {
				const year = this.getAssignDateInfo(false, 0)
				const month = this.getAssignDateInfo(false, 1)
				const cur = this.generateCalendar(year, month)
				const prev = this.generateCalendar(month === 1 ? year - 1 : year, month === 1 ? 12 : month - 1)
				const next = this.generateCalendar(month === 12 ? year + 1 : year, month === 12 ? 1 : month + 1)
				// 根据current来判断相邻的轮播存放哪些日历数据
				if (this.current === 0) {
					this.calendarSwiperDates = [cur, next, prev]
				} else if (this.current === 1) {
					this.calendarSwiperDates = [prev, cur, next]
				} else if (this.current === 2) {
					this.calendarSwiperDates = [next, prev, cur]
				}
				this.swiperByClick = false
				// 初始化日期信息完毕执行回调函数
				cb && cb()
			},
			// 生成展开的日历数据
			generateCalendar(year, month) {
				let calendarDate = []
				// 先获取缓存里面有没有该月的日期数据
				if (this.dateCache[`${year}-${month}`]) {
					calendarDate = deepClone(this.dateCache[`${year}-${month}`])
				} else { // 进行月份日期的计算
					const monthDates = new Date(year, month, 0).getDate() // 获取此月份总天数
					const normalWeek = ['一', '二', '三', '四', '五', '六', '日'] // 正常周数
					const monthFirstDay = normalWeek[new Date(year, month - 1, 0).getDay()] // 获取本月一号为星期几
					const monthFirstDayIndex = this.week.indexOf(monthFirstDay) // 计算本月一号在日历周数中的索引，索引之前的填充上个月的后几天
					// 本月一号在日历中不是第一个位置，需要进行填充
					if (monthFirstDayIndex !== 0) {
						const prevMonthDates = new Date(year, month - 1, 0).getDate() // 获取上一个月份的总天数
						// 填充本月一号之前的数据
						for (let i = 0; i < monthFirstDayIndex; i ++) {
							const item = {
								year: month === 1 ? year - 1 : year,
								month: month === 1 ? 12 : month - 1,
								date: prevMonthDates - i,
								type: 'prev'
							}
							// 判断填充的日期是否包含今天日期
							this.theDateIsToday(item)
							calendarDate.unshift(item)
						}
					}
					// 循环生成当月所有日期
					for (let i = 1; i <= monthDates; i ++) {
						const item = {
							year,
							month,
							date: i,
							isSelected: false,
							isToday: false,
							type: 'cur'
						}
						// 今天的日期在不在里面
						this.theDateIsToday(item)
						calendarDate.push(item)
					}
					const residue = calendarDate.length % 7
					// 判断是否需要填充下个月的前几天
					if (residue !== 0) {
						for (let i = 1; i <= 7 - residue; i ++) {
							const item = {
								year: month === 12 ? year + 1 : year,
								month: month === 12 ? 1 : month + 1,
								date: i,
								type: 'next'
							}
							// 下个月的前几天包含今天
							this.theDateIsToday(item)
							calendarDate.push(item)
						}
					}
					this.dateCache[`${year}-${month}`] = deepClone(calendarDate)
				}
				// 进行日期的默认选中
				if (year === this.getAssignDateInfo(false, 0) && month === this.getAssignDateInfo(false, 1)) {
					for (let i = 0, len = calendarDate.length; i < len; i++) {
						if (calendarDate[i].type === 'cur' && calendarDate[i].date === this.getAssignDateInfo(false, 2)) {
							calendarDate[i].isSelected = true
							this.dateActive = i
							break
						}
					}
				}
				return calendarDate
			},
			// 判断日期是否为今天
			theDateIsToday(item) {
				if (item.year + '-' + item.month + '-' + item.date === this.getAssignDateInfo(true, 0) + '-' + this.getAssignDateInfo(true, 1) + '-' + this.getAssignDateInfo(true, 2)) {
					item.isToday = true
				}
			},
			// 初始化收缩时的日历轮播日期信息
			initCalendarShrinkSwiperDates(swiperChangeType) {
				let line = null
				/**
				 * 日历收缩事件/当前滑动不涉及到到上个/下个月的日期数据
				 * 日历滑动到上一周并且本周不属于第一行并且上一周选中的日期必须是本月份里面的日期
				 * 日历滑动到下一周且本周不属于最后一行
				 */
				const curDateLine = Math.floor(this.dateActive / 7)
				if (!swiperChangeType ||
						(swiperChangeType === -1 && curDateLine !== 0 && this.calendarSwiperDates[this.current][(curDateLine - 1) * 7].type === 'cur') ||
						(swiperChangeType === 1 && curDateLine + 1 !== this.calendarSwiperDates[this.current].length / 7)
				) {
					// 计算当前周选中日期处于日历中的哪一行位置
					const curCalendarSwiperDates = this.calendarSwiperDates[this.current]
					line = Math.floor(curCalendarSwiperDates.map(item => item.type === 'cur' ? item.date : -1).indexOf(this.getAssignDateInfo(false, 2)) / 7)
					// 收缩日历滑动事件需要进行日期的选中处理
					if (swiperChangeType) {
						// 将当前选中日期清除选中状态
						this.calendarSwiperDates[this.current][this.dateActive].isSelected = false
						// 重新计算日期选中高亮并把下一个日期进行选中
						this.dateActive = line * 7
						this.calendarSwiperDates[this.current][this.dateActive].isSelected = true
					}
				} else { // 收缩日历滑动事件
					// 将当前选中日期清除选中状态
					this.calendarSwiperDates[this.current][this.dateActive].isSelected = false
					// 涉及了上个月/下个月的日期数据，需要重新计算展开日历轮播的日期数据
					let currentNum = this.current + swiperChangeType
					currentNum = currentNum > 2 ? 0 : currentNum < 0 ? 2 : currentNum
					this.current = currentNum
					// 计算上一周/下一周选中日期处于日历中的哪一行位置
					const curCalendarSwiperDates = this.calendarSwiperDates[this.current]
					line = Math.floor(curCalendarSwiperDates.map(item => item.type === 'cur' ? item.date : -1).indexOf(this.getAssignDateInfo(false, 2)) / 7)
					// 重新计算日期选中高亮并把下一个日期进行选中
					this.dateActive = line * 7
					this.calendarSwiperDates[this.current][this.dateActive].isSelected = true
				}
				const cur = this.generateShrinkCalendar(0, line)
				const prev = this.generateShrinkCalendar(-1, line)
				const next = this.generateShrinkCalendar(1, line)
				// 根据shrinkCurrent来判断相邻的轮播存放哪些日历数据
				if (this.shrinkCurrent === 0) {
					this.calendarSwiperShrinkDates = [cur, next, prev]
				} else if (this.shrinkCurrent === 1) {
					this.calendarSwiperShrinkDates = [prev, cur, next]
				} else if (this.shrinkCurrent === 2) {
					this.calendarSwiperShrinkDates = [next, prev, cur]
				}
			},
			// 生成收缩的日历数据
			generateShrinkCalendar(type, line) {
				// 返回当前这一周的日期数据
				if (type === 0) {
					return this.calendarSwiperDates[this.current].slice(line * 7, (line + 1) * 7)
				}
				// 返回上一周的日期数据
				if (type === -1) {
					// 当前选中的日期是否位于第一行
					if (line === 0) {
						/**
						 * 当前日历的第一行是否包含有上个月的日期
						 * 如果有包含，则返回上个月的倒数第二行日期
						 * 如果没有包含，则返回上个月的倒数第一行日期
						 */
						// 计算上个月的索引值
						const prevIndex = this.current === 0 ? 2 : this.current - 1
						// 获取上个月的日期数据
						const prevCalendarSwiperDates = this.calendarSwiperDates[prevIndex]
						// 获取上个月的日历行数
						const prevCalendarSwiperDatesLine = prevCalendarSwiperDates.length / 7
						if (this.calendarSwiperDates[this.current][0].type === 'prev') { // 倒数第二行
							return prevCalendarSwiperDates.slice((prevCalendarSwiperDatesLine - 2) * 7, (prevCalendarSwiperDatesLine - 1) * 7)
						} else { // 倒数第一行
							return prevCalendarSwiperDates.slice((prevCalendarSwiperDatesLine - 1) * 7)
						}
					} else {
						return this.calendarSwiperDates[this.current].slice((line - 1) * 7, line * 7)
					}
				}
				// 返回下一周的日期数据
				if (type === 1) {
					// 计算当前日历月份总共有多少行
					const curMonthMaxLine = this.calendarSwiperDates[this.current].length / 7
					// 当前选中的日期是否位于最后一行
					if (line === curMonthMaxLine - 1) {
						/**
						 * 当前日历的最后一行是否包含有下个月的日期
						 * 如果有包含，则返回下个月的第二行日期
						 * 如果没有包含，则返回上个月的第一行日期
						 */
						// 计算下个月的索引值
						const nextIndex = this.current === 2 ? 0 : this.current + 1
						// 获取下个月的日期数据
						const nextCalendarSwiperDates = this.calendarSwiperDates[nextIndex]
						// 获取下个月的日历行数
						const nextCalendarSwiperDatesLine = nextCalendarSwiperDates.length / 7
						if (this.calendarSwiperDates[this.current][this.calendarSwiperDates[this.current].length - 1].type === 'next') { // 第二行
							return nextCalendarSwiperDates.slice(7, 14)
						} else { // 第一行
							return nextCalendarSwiperDates.slice(0, 7)
						}
					} else {
						return this.calendarSwiperDates[this.current].slice((line + 1) * 7, (line + 2) * 7)
					}
				}
			},
			// 展开日历轮播切换事件
			swiperChange(type) {
				// 通过点击上个月/下个月日期进行切换，不需要默认选中下个月的一号，直接选中点击的那个日期
				if (!this.swiperByClick && this.swiperMode === 'open') {
					this.getPrevOrNextDate(type)
				}
				setTimeout(() => { // 设置定时器是为了防止轮播切换时生成数据造成页面卡顿
					this.initCalendarSwiperDates(() => {
						this.swiperMode === 'close' && this.initCalendarShrinkSwiperDates()
					}) // 初始化日历轮播日期信息
				}, this.swiperMode === 'open' ? this.duration : 0)
			},
			// 收缩日历轮播切换事件
			shrinkSwiperChange(type) {
				// 默认选中下个星期的开始日期
				this.getPrevOrNextStartDate(type)
				setTimeout(() => { // 设置定时器是为了防止轮播切换时生成数据造成页面卡顿
					this.initCalendarShrinkSwiperDates(type) // 初始化日历轮播日期信息
				}, this.duration)
			},
			// 手动切换日历
			switchCalendar(type) {
				const currentKey = this.swiperMode === 'close' ? 'shrinkCurrent' : 'current'
				const v = this[currentKey] + (type === 'prev' ? -1 : 1)
				this[currentKey] = v === -1 ? 2 : v === 3 ? 0 : v
			},
			// 获取上一个月/下一个月的一号日期
			getPrevOrNextDate(type) {
				const year = this.getAssignDateInfo(false, 0)
				let month = this.getAssignDateInfo(false, 1)
				month = month + type
				// 判断切换月份时选中当前日期高亮还是一号，若选中当前日期高亮需进行大小判断
				const curActiveDate = this.getAssignDateInfo(false, 2)
				const maxDate = new Date(year, month, 0).getDate()
				const date = this.changeSetDefault ? 1 : curActiveDate > maxDate ? maxDate : curActiveDate
				this.selectedDate = parseTime(new Date(year, month - 1, date), '{y}-{m}-{d}')
			},
			// 获取上个星期/下一星期的开始日期
			getPrevOrNextStartDate(type) {
				const date = this.calendarSwiperShrinkDates[this.shrinkCurrent][0]
				this.selectedDate = parseTime(new Date(date.year, date.month - 1, date.date), '{y}-{m}-{d}')
			},
			// 前往某一天 格式 YYYY-MM | YYYY-MM-DD
			goToDate(date = this.today) {
				try {
					if (date.split('-').length < 2 || date.split('-').length > 3) throw '参数有误'
					if (date.split('-').length === 2) {
						date += '-01'
					}
				} catch (err) {
					throw Error('请检查参数是否符合规范')
				}
				this.selectedDate = date
				this.initCalendarSwiperDates(() => {
					this.initCalendarShrinkSwiperDates()
				})
			},
			// 日历轮播展开的情况下选择日期
			chooseDate(dateInfo, dateIndex) {
				// 重复点击后续不做处理
				if (dateInfo.isSelected) return false
				// 是否点击了上个月份的后几天或者点击了下个月份的前几天
				if (dateInfo.type !== 'cur') {
					if (dateInfo.type === 'prev') { // 点击了上个月份的后几天，滑到上个月
						this.current = this.current === 0 ? 2 : this.current - 1
					} else { // 点击了下个月份的前几天，滑到下个月
						this.current = this.current === 2 ? 0 : this.current + 1
					}
					// 将选中日期赋值为当前点击的那个日期
					this.selectedDate = parseTime(new Date(dateInfo.year, dateInfo.month - 1, dateInfo.date), '{y}-{m}-{d}')
					this.swiperByClick = true
					return false
				}
				// 将当前选中的日期清空并选中最新的日期
				this.calendarSwiperDates[this.current][this.dateActive].isSelected = false
				this.dateActive = dateIndex
				const date = this.calendarSwiperDates[this.current][this.dateActive]
				date.isSelected = true
				this.selectedDate = parseTime(new Date(date.year, date.month - 1, date.date), '{y}-{m}-{d}')
				this.dateClick = true
			},
			// 日历轮播收缩的情况下选择日期
			chooseShrinkDate(dateInfo, dateIndex) {
				// 重复点击后续不做处理
				if (dateInfo.isSelected) return false
				this.dateClick = true
				// 是否点击了上个月份的后几天或者点击了下个月份的前几天
				if (dateInfo.type !== 'cur') {
					if (dateInfo.type === 'prev') { // 点击了上个月份的后几天，切换到上个月
						this.current = this.current === 0 ? 2 : this.current - 1
					} else { // 点击了下个月份的前几天，切换到下个月
						this.current = this.current === 2 ? 0 : this.current + 1
					}
					this.dateActive = dateIndex
					// 将选中日期赋值为当前点击的那个日期
					this.selectedDate = parseTime(new Date(dateInfo.year, dateInfo.month - 1, dateInfo.date), '{y}-{m}-{d}')
					return false
				}
				// 计算当前选中日期之前有多少个日期
				const dateActiveLine = Math.floor(this.dateActive / 7) * 7
				// 将当前选中的日期清空并选中最新的日期
				this.calendarSwiperDates[this.current][this.dateActive].isSelected = false
				this.dateActive = dateIndex + dateActiveLine
				const date = this.calendarSwiperDates[this.current][this.dateActive]
				date.isSelected = true
				this.selectedDate = parseTime(new Date(date.year, date.month - 1, date.date), '{y}-{m}-{d}')
			},
			// 向父组件传递当前选中数据
			emitDate() {
				const { year, month, date } = this.calendarSwiperDates[this.current][this.dateActive]
				const e = {
					selectedDate: this.selectedDate,
					year,
					month,
					date
				}
				this.$emit('change', e)
			}
		}
	}
</script>

<style>
	.zsy_calendar {
		width: 100%;
		padding: 20rpx 0;
		box-sizing: border-box;
		background-color: #fff;
		border-radius: 20rpx;
	}
	
	/* 日历顶部信息 */
	.calendar_info {
		display: flex;
		align-items: center;
		padding: 0 20rpx;
	}
	.calendar_info .title {
		font-size: 34rpx;
		font-weight: bold;
		color: #2C2C2C;
	}
	.calendar_info .desc {
		margin-left: 29rpx;
		font-size: 28rpx;
		color: #959595;
	}
	.calendar_info .backToToday {
		margin-left: auto;
		font-size: 24rpx;
	}
	/* 日历顶部信息 */
	
	/* 日历周数 */
	.calendar_week {
		display: flex;
		align-items: center;
		justify-content: space-between;
		font-size: 26rpx;
		color: #959595;
		margin: 20rpx 0rpx;
	}
	.calendar_week .calendar_week__item {
		width: calc(100% / 7);
		text-align: center;
	}
	/* 日历周数 */
	
	/* 日历切换模式 */
	.calendar_toggle {
		position: relative;
		padding: 10rpx 0;
		margin: 10rpx 20rpx 0;
		display: flex;
		justify-content: center;
	}
	.calendar_toggle .icon {
		width: 30rpx;
		height: 30rpx;
		background-image: url('../../static/zsy-calendar/arrow.png');
		background-size: contain;
		background-repeat: no-repeat;
		margin: 0 auto;
		transition: all .3s;
	}
	.icon.down {
		transform: rotate(180deg);
	}
	.calendar_toggle::before, .calendar_toggle::after {
		width: calc(50% - 30rpx);
		border-top: solid 2rpx #EAEAEA;
		content: '';
		display: block;
		position: absolute;
		top: 50%;
		transform: translateY(-50%);
	}
	.calendar_toggle::before {
		left: 0;
	}
	.calendar_toggle::after {
		right: 0;
	}
	/* 日历切换模式 */
</style>
