<template>
	<!-- 日期显示 -->
	<view class="date_box">
		<view
			v-for="(dateInfo, dateIndex) in dates"
			:key="dateIndex"
			class="calendar_date__box"
		>
			<view
				class="calendar_date"
				:class="{ isSelected: dateActiveIndex === dateIndex && dateInfo.type === 'cur' }"
				:style="{
					height: cellHeight + 'rpx',
					width: cellHeight + 'rpx',
					color: swiperMode === 'open' ? dateInfo.type === 'cur' ? '#2C2C2C' : '#959595' : '#2C2C2C',
					backgroundColor: dateActiveIndex === dateIndex && dateInfo.type === 'cur' ? dateActiveColor : ''
				}"
				@tap="chooseDate(dateInfo)"
			>
				<view class="calendar_date__number">{{ dateInfo.date }}</view>
				<view class="calendar_date__isToday" v-if="dateInfo.isToday" :style="{ backgroundColor: dateActiveColor }"></view>
				<view class="calendar_date__cricle"></view>
			</view>
		</view>
	</view>
</template>

<script>
	export default {
		props: {
			dates: {
				type: Array,
				default: () => []
			},
			cellHeight: { // 一列的高度
				type: Number,
				default: 75
			},
			dateActiveColor: { // 日期选中颜色
				type: String,
				default: '#FE6601'
			},
			selectedDate: {
				type: String,
				default: ''
			},
			swiperMode: { // 日历显示模式
				type: String,
				default: 'open'
			},
			showActive: { // 是否显示选中高亮日期
				type: Boolean,
				default: false
			}
		},
		computed: {
			dateActiveIndex() {
				return this.showActive ? this.dates.map(item => item.dateFormat).indexOf(this.selectedDate) : -1
			}
		},
		methods: {
			chooseDate(dateInfo) {
				this.$emit('chooseDate', dateInfo)
			}
		}
	}
</script>

<style>
	/* 日历轮播 */
	.date_box {
		display: flex;
		flex-wrap: wrap;
		justify-content: space-between;
	}
	.date_box .calendar_date__box {
		width: calc(100% / 7);
		margin-top: 20rpx;
	}
	.calendar_date__box .calendar_date {
		text-align: center;
		margin: 0 auto;
		font-weight: bold;
		font-size: 28rpx;
		border-radius: 50%;
		display: flex;
		flex-direction: column;
		align-items: center;
		justify-content: center;
		position: relative;
	}
	.calendar_date__box .calendar_date.isSelected {
		color: #FFFFFF !important;
	}
	.calendar_date .calendar_date__isToday {
		width: 100%;
		height: 100%;
		position: absolute;
		top: 0;
		left: 0;
		border-radius: 50%;
		z-index: -1;
		opacity: 0.4;
	}
	.calendar_date .calendar_date__cricle {
		width: 9rpx;
		height: 9rpx;
		border-radius: 50%;
		margin-top: 5rpx;
		background-color: #FFFFFF;
	}
	/* 日历轮播 */
</style>
