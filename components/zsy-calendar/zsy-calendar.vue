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
			<swiper
				key="normalSwiper"
				circular
				:style="{height: swiperHeight}"
				:current="current"
				:duration="duration"
				:skip-hidden-item-layout="true"
				@change="e => current = e.detail.current"
			>
				<swiper-item v-for="(swiper, swiperIndex) in 3" :key="swiperIndex" class="swiper-item">
					<DateBox
						:dates="getcurCalendarDates[swiperIndex]"
						:cellHeight="cellHeight"
						:selectedDate="selectedDate"
						:dateActiveColor="dateActiveColor"
						:swiperMode="swiperMode"
						:showActive="emitTimer === null"
						@chooseDate="chooseDate"
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
				calendarSwiperDates: [], // 日历轮播日期信息
				swiperChangeByClick: false, // 是否通过点击上月份或下月份的日期进行轮播切换
				swiperMode: this.mode, // 日历轮播显示模式 open：展开 close：收缩
				monthDateCache: {}, // 月份日期缓存数据
				emitTimer: null, // 日期改变向父级传递当前选中日期计时器
				dateClick: false // 是否进行了日期的点击选择
			}
		},
		computed: {
			// 返回当前日期信息（展开状态下为每月，收缩状态下为每周）
			getcurCalendarDates() {
				if (this.swiperMode === 'open') { // 展开
					return this.calendarSwiperDates
				} else {
					return this.getCalendarShrinkSwiperDates()
				}
			},
			// 计算选中日期的上月、本月、下月的年月信息
			getAdjacentYMD() {
				const year = this.getAssignDateInfo(false, 0)
				const month = this.getAssignDateInfo(false, 1)
				const prev = `${month === 1 ? year - 1 : year}-${month === 1 ? 12 : month - 1}`
				const cur = `${year}-${month}`
				const next = `${month === 12 ? year + 1 : year}-${month === 12 ? 1 : month + 1}`
				return [prev, cur, next]
			},
			/* 获取指定日期信息
			isToday: 是否获取当天的信息还是选中日期的信息
			index: 0 表示年份 1 表示月份 2 表示日期 */
			getAssignDateInfo() {
				return (isToday, index) => {
					return (isToday ? this.today : this.selectedDate).split('-')[index] * 1
				}
			},
			// 是否显示回到今天按钮
			showBackToTodayBtn() {
				return this.getAssignDateInfo(false, 0) !== this.getAssignDateInfo(true, 0) || this.getAssignDateInfo(false, 1) !== this.getAssignDateInfo(true, 1)
			},
			// 返回轮播图高度
			swiperHeight() {
				const normalHeight = (this.calendarSwiperDates[this.current] || []).length / 7 * (this.cellHeight + 20) + 'rpx'
				const shrinkHeight = this.cellHeight + 20 + 'rpx'
				return this.swiperMode === 'open' ? normalHeight : shrinkHeight
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
			selectedDate: {
				deep: true,
				handler(newV, oldV) {
					// 判断月历日期数据需不需要改变
					if (this.swiperMode === 'close') {
						setTimeout(() => {
							this.generateAdjacentMonthDate() // 生成临近月份日期缓存数据
						}, this.duration);
					}
					if (newV && (oldV === null || this.dateClick)) { // 初始化/日历点击选择时直接返回
						this.emitDate()
						this.dateClick = false
					} else { // 其它情况做防抖处理
						if (this.emitTimer !== null) {
							clearTimeout(this.emitTimer)
							this.emitTimer = null
						}
						this.emitTimer = setTimeout(() => {
							this.emitDate()
							this.emitTimer = null
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
				this.generateAdjacentMonthDate() // 生成临近月份日期缓存数据
			},
			// 初始化周数
			initWeek() {
				const normalWeek = ['日', '一', '二', '三', '四', '五', '六'] // 正常周数
				const sIndex = this.sundayIndex < 0 ? 0 : this.sundayIndex >= normalWeek.length ? normalWeek.length - 1 : this.sundayIndex
				normalWeek.unshift(...normalWeek.slice(-sIndex))
				normalWeek.length = 7
				this.week = normalWeek
			},
			// 根据current自动对轮播数据进行衔接排序
			adjacentSortByCurrent(prev, cur, next) {
				let arr
				if (this.current === 0) {
					arr = [cur, next, prev]
				} else if (this.current === 1) {
					arr = [prev, cur, next]
				} else if (this.current === 2) {
					arr = [next, prev, cur]
				}
				return arr
			},
			// 生成本月、上个月、下个月日期信息
			generateAdjacentMonthDate() {
				const arr = []
				this.getAdjacentYMD.map(YM => {
					const [year, month] = YM.split('-')
					arr.push(this.generateMonthDateCache(year, month))
				})
				const [prev, cur, next] = arr
				this.calendarSwiperDates = this.adjacentSortByCurrent(prev, cur, next)
				if (this.swiperChangeByClick) {
					this.swiperChangeByClick = false
				}
			},
			// 生成月份日期缓存数据并返回
			generateMonthDateCache(year, month) {
				year = Number(year)
				month = Number(month)
				// 缓存中已存在
				if (this.monthDateCache[`${year}-${month}`]) return this.monthDateCache[`${year}-${month}`]
				let calendarDate = []
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
							dateFormat: `${month === 1 ? year - 1 : year}-${String(month === 1 ? 12 : month - 1).padStart(2, '0')}-${String(prevMonthDates - i).padStart(2, '0')}`,
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
						dateFormat: `${year}-${String(month).padStart(2, '0')}-${String(i).padStart(2, '0')}`,
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
							dateFormat: `${month === 12 ? year + 1 : year}-${String(month === 12 ? 1 : month + 1).padStart(2, '0')}-${String(i).padStart(2, '0')}`,
							type: 'next'
						}
						// 下个月的前几天包含今天
						this.theDateIsToday(item)
						calendarDate.push(item)
					}
				}
				this.monthDateCache[`${year}-${month}`] = deepClone(calendarDate)
				return this.monthDateCache[`${year}-${month}`]
			},
			// 轮播图切换结束
			swiperChange(e) {
				// 切换上个月/下个月，默认选中一号 / 切换上一周/下一周，默认选中第一天
				if (!this.swiperChangeByClick) {
					this.getPrevOrNextDate(e)
				}
				if (this.swiperMode === 'open') { // 展开
					// 通过点击上个月/下个月日期进行切换，不需要默认选中下个月的一号，直接选中点击的那个日期
					setTimeout(() => {
						this.generateAdjacentMonthDate() // // 重新生成临近月份日期缓存数据
					}, this.duration)
				}
			},
			// 判断日期是否为今天
			theDateIsToday(item) {
				if (`${item.year}${item.month}${item.date}` === `${this.getAssignDateInfo(true, 0)}${this.getAssignDateInfo(true, 1)}${this.getAssignDateInfo(true, 2)}`) {
					item.isToday = true
				}
			},
			// 计算收缩时的日历轮播日期信息
			getCalendarShrinkSwiperDates() {
				const [prevYM, curYM, nextYM] = this.getAdjacentYMD
				// 本月日期数据
				const curDates = this.monthDateCache[curYM]
				// 计算当前日期所在行
				const line = Math.floor(curDates.map(item => item.dateFormat).indexOf(this.selectedDate) / 7)
				// 当前周日期信息
				const cur = curDates.slice(line * 7, (line + 1) * 7)
				let prev, next
				/**
				 * 获取上一周日期信息
				 * 注意：当选中日期为第一周要额外判断，如果刚好为日历的第一天，则上一周数据应为上一个月的最后一周，否则为上一个月的倒数第二周
				 */ 
				if (line === 0) {
					// 获取上个月日历数据
					const prevDates = this.monthDateCache[prevYM]
					// 获取上个月的日历行数
					const prevDatesLine = prevDates.length / 7
					if (curDates[0].dateFormat === this.selectedDate) { // 选中日期刚好为日历第一天
						prev = prevDates.slice((prevDatesLine - 1) * 7) // 上个月倒数第一周数据
					} else {
						prev = prevDates.slice((prevDatesLine - 2) * 7, (prevDatesLine - 1) * 7) // 上个月倒数第二周数据
					}
				} else {
					prev = curDates.slice((line - 1) * 7, line * 7)
				}
				/**
				 * 获取下一周日期信息
				 * 注意：当选中日期为最后一周要额外判断，如果刚好为日历的最后一天，则下一周数据应为下一个月的第一周，否则为下一个月的第二周
				 */ 
				if (line + 1 === curDates.length / 7) {
					// 获取下个月的日期数据
					const nextDates = this.monthDateCache[nextYM]
					if (curDates[curDates.length - 1].dateFormat === this.selectedDate) { // 选中日期刚好为日历最后一天
						next = nextDates.slice(0, 7) // 下个月第一周数据
					} else {
						next = nextDates.slice(7, 14) // 下个月第二周数据
					}
				} else {
					next = curDates.slice((line + 1) * 7, (line + 2) * 7)
				}
				return this.adjacentSortByCurrent(prev, cur, next)
			},
			// 手动切换日历
			switchCalendar(type) {
				const currentKey = this.swiperMode === 'close' ? 'shrinkCurrent' : 'current'
				const v = this[currentKey] + (type === 'prev' ? -1 : 1)
				this[currentKey] = v === -1 ? 2 : v === 3 ? 0 : v
			},
			// 获取月的一号日期/周的第一天
			getPrevOrNextDate(type) {
				if (this.swiperMode === 'open') {
					const year = this.getAssignDateInfo(false, 0)
					let month = this.getAssignDateInfo(false, 1)
					month = month + type
					// 判断切换月份时选中当前日期高亮还是一号，若选中当前日期高亮需进行大小判断
					const curActiveDate = this.getAssignDateInfo(false, 2)
					const maxDate = new Date(year, month, 0).getDate()
					const date = this.changeSetDefault ? 1 : curActiveDate > maxDate ? maxDate : curActiveDate
					this.selectedDate = parseTime(new Date(year, month - 1, date), '{y}-{m}-{d}')
				} else {
					let current = this.current + type < 0 ? 2 : this.current + type > 2 ? 0 : this.current + type
					this.selectedDate = this.getcurCalendarDates[current][0].dateFormat
				}
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
				this.generateAdjacentMonthDate()
			},
			// 日历轮播展开的情况下选择日期
			chooseDate(dateInfo) {
				// 重复点击后续不做处理
				if (dateInfo.dateFormat === this.selectedDate) return false
				if (this.swiperMode === 'open') { // 展开
					// 是否点击了上个月份的后几天或者点击了下个月份的前几天
					if (dateInfo.type !== 'cur') {
						if (dateInfo.type === 'prev') { // 点击了上个月份的后几天，滑到上个月
							this.current = this.current === 0 ? 2 : this.current - 1
						} else { // 点击了下个月份的前几天，滑到下个月
							this.current = this.current === 2 ? 0 : this.current + 1
						}
						// 将选中日期赋值为当前点击的那个日期
						this.swiperChangeByClick = true
					} else {
						this.dateClick = true
					}
				} else { // 收缩
					// 是否点击了上个月份的后几天或者点击了下个月份的前几天
					if (dateInfo.type !== 'cur') {
						// 将选中日期赋值为当前点击的那个日期
						this.swiperChangeByClick = true
					}
					this.dateClick = true
				}
				// 将当前选中的日期清空并选中最新的日期
				this.selectedDate = dateInfo.dateFormat
			},
			// 向父组件传递当前选中数据
			emitDate() {
				const e = {
					selectedDate: this.selectedDate
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
		transform: rotate(0deg);
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
