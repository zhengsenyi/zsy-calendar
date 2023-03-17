<template>
	<!-- 仿钉钉打卡日历组件 -->
	<view class="calendar_container">
		<zsyCalendar
			@change="change"
		/>
	</view>
</template>

<script>
	import zsyCalendar from '@/components/zsy-calendar/zsy-calendar'
	export default {
		name: 'Calendar',
		components: {
			zsyCalendar
		},
		methods: {
			// 日历选中日期改变事件回调
			change(e) {
				console.log(e)
			}
		}
	}
</script>

<style>
	.calendar_container {
		min-height: calc(100vh - var(--window-top));
		background-color: #f5f5f5;
		padding: 30rpx;
		box-sizing: border-box;
	}
</style>
