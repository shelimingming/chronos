<template>
	<view class="container">
		<!-- 日期选择器 -->
		<view class="date-selector">
			<view class="date-arrow" @click="changeDate(-1)"><text class="arrow-text">◀</text></view>
			<view class="current-date">{{ formatDate(currentDate) }}</view>
			<view class="date-arrow" @click="changeDate(1)"><text class="arrow-text">▶</text></view>
		</view>
		
		<!-- 时间轴视图 -->
		<scroll-view scroll-y class="timeline-container">
			<view v-for="hour in hours" :key="hour" class="timeline-hour">
				<view class="hour-label">{{ hour }}:00</view>
				<view class="hour-content">
					<!-- 显示该小时内的活动 -->
					<view 
						v-for="activity in getActivitiesByHour(hour)" 
						:key="activity._id"
						class="activity-item"
						@click.stop="editActivity(activity._id)"
						:style="{
							backgroundColor: activity.categoryColor || '#a0c4ff',
							height: calculateActivityHeight(activity) + 'px',
							top: calculateActivityTop(activity, hour) + 'px',
							left: '0',
							width: '100%'
						}"
					>
						<view class="activity-marker"></view>
						<text class="activity-title">{{ activity.title }}</text>
					</view>
				</view>
			</view>
		</scroll-view>
		
		<!-- 添加活动按钮 -->
		<view class="add-button" @click="addActivity">
			<text class="add-icon">+</text>
		</view>
	</view>
</template>

<script>
	export default {
		data() {
			return {
				currentDate: new Date(),
				hours: Array.from({length: 24}, (_, i) => i),
				activities: [],
				categories: []
			};
		},
		onLoad() {
			this.loadCategories().then(() => {
				this.loadActivities();
			});
		},
		onShow() {
			// 每次页面显示时重新加载活动数据
			this.loadCategories().then(() => {
				this.loadActivities();
			});
		},
		methods: {
			// 加载活动类别
			async loadCategories() {
				try {
					const db = uniCloud.database();
					const categoriesCollection = db.collection('chronos-categories');
					const res = await categoriesCollection.get();
					this.categories = res.result.data || [];
				} catch (e) {
					console.error('加载类别失败:', e);
					uni.showToast({
						title: '加载类别失败',
						icon: 'none'
					});
				}
			},
			
			// 加载活动数据
			async loadActivities() {
				try {
					const db = uniCloud.database();
					const activitiesCollection = db.collection('chronos-activities');
					
					// 获取当天开始和结束时间
					const startOfDay = new Date(this.currentDate);
					startOfDay.setHours(0, 0, 0, 0);
					
					const endOfDay = new Date(this.currentDate);
					endOfDay.setHours(23, 59, 59, 999);
					
					// 将Date对象转换为时间戳（毫秒数）
					const startTimestamp = startOfDay.getTime();
					const endTimestamp = endOfDay.getTime();
					
					// 查询当天的活动
					const res = await activitiesCollection
						.where({
							start_time: db.command.gte(startTimestamp).and(db.command.lte(endTimestamp))
						})
						.orderBy('start_time', 'asc')
						.get();
					
					let activities = res.result.data || [];
					
					// 为每个活动添加类别颜色
					activities = activities.map(activity => {
						const category = this.categories.find(c => c._id === activity.category);
						return {
							...activity,
							categoryColor: category ? category.color : '#a0c4ff'
						};
					});
					
					this.activities = activities;
				} catch (e) {
					console.error('加载活动失败:', e);
					uni.showToast({
						title: '加载活动失败',
						icon: 'none'
					});
				}
			},
			
			// 根据小时获取活动
			getActivitiesByHour(hour) {
				return this.activities.filter(activity => {
					const startHour = new Date(activity.start_time).getHours();
					const endHour = new Date(activity.end_time).getHours();
					
					// 如果活动跨越多个小时，只要有部分在当前小时内就显示
					return (startHour <= hour && endHour >= hour) || 
						   (startHour === hour);
				});
			},
			
			// 计算活动在时间轴上的高度（基于持续时间）
			calculateActivityHeight(activity) {
				// 每分钟1px高度，最小高度为30px
				return Math.max(activity.duration, 30);
			},
			
			// 计算活动在时间轴上的位置
			calculateActivityTop(activity, hour) {
				const startTime = new Date(activity.start_time);
				const startHour = startTime.getHours();
				
				if (startHour === hour) {
					// 如果活动开始于当前小时，根据分钟计算位置
					return startTime.getMinutes();
				} else if (startHour < hour) {
					// 如果活动开始于之前的小时，从当前小时的开始位置显示
					return 0;
				}
				return 0;
			},
			
			// 格式化日期为 YYYY年MM月DD日 星期X
			formatDate(date) {
				const year = date.getFullYear();
				const month = date.getMonth() + 1;
				const day = date.getDate();
				const weekdays = ['日', '一', '二', '三', '四', '五', '六'];
				const weekday = weekdays[date.getDay()];
				
				return `${year}年${month}月${day}日 星期${weekday}`;
			},
			
			// 切换日期
			changeDate(offset) {
				const newDate = new Date(this.currentDate);
				newDate.setDate(newDate.getDate() + offset);
				this.currentDate = newDate;
				this.loadActivities();
			},
			
			// 添加新活动
			addActivity() {
				// 跳转到活动编辑页面
				uni.navigateTo({
					url: '/pages/activity/edit'
				});
			},
			// 编辑活动
			editActivity(id) {
				console.log('编辑活动被触发，活动ID:', id);
				uni.navigateTo({
					url: `/pages/activity/edit?id=${id}`
				}).then(() => {
					console.log('页面跳转成功');
				}).catch(err => {
					console.error('页面跳转失败:', err);
				});
			}
		}
	}
</script>

<style>
.container {
	padding: 0;
	position: relative;
	height: 100vh;
	background-color: #f8f8f8;
	display: flex;
	flex-direction: column;
	box-sizing: border-box;
	padding-bottom: var(--window-bottom); /* 适应底部安全区域 */
}

.date-selector {
	display: flex;
	justify-content: space-between;
	align-items: center;
	padding: 15px;
	background-color: #ffffff;
	border-bottom: 1px solid #eaeaea;
}

.date-arrow {
	width: 30px;
	height: 30px;
	display: flex;
	justify-content: center;
	align-items: center;
	border-radius: 15px;
	background-color: #f0f0f0;
}

.arrow-text {
	font-size: 16px;
	color: #333;
}

.current-date {
	font-size: 16px;
	font-weight: bold;
	color: #333;
}

.timeline-container {
	height: calc(100vh - 130px - var(--window-bottom)); /* 减去顶部区域和底部安全区域 */
	background-color: #ffffff;
	flex: 1;
	overflow: hidden;
}

.timeline-hour {
	display: flex;
	position: relative;
	height: calc((100vh - 130px - var(--window-bottom)) / 24); /* 动态计算每小时高度 */
	min-height: 40px; /* 减小最小高度以适应小屏幕 */
	border-bottom: 1px solid #f0f0f0;
	flex-shrink: 0; /* 防止压缩 */
}

.hour-label {
	width: 50px;
	padding: 5px;
	text-align: center;
	color: #666;
	font-size: 12px;
	border-right: 1px solid #f0f0f0;
	display: flex;
	align-items: center;
	justify-content: center;
}

.hour-content {
	flex: 1;
	position: relative;
}

.activity-item {
	position: absolute;
	border-radius: 4px;
	padding: 5px 10px;
	overflow: hidden;
	display: flex;
	align-items: center;
}

.activity-marker {
	width: 4px;
	height: 100%;
	background-color: rgba(255, 255, 255, 0.7);
	margin-right: 8px;
}

.activity-title {
	color: #fff;
	font-size: 12px;
	font-weight: bold;
	text-shadow: 0 1px 1px rgba(0, 0, 0, 0.1);
	overflow: hidden;
	text-overflow: ellipsis;
	white-space: nowrap;
	max-width: 100%;
}

.add-button {
	position: fixed;
	right: 20px;
	bottom: calc(var(--window-bottom) + 20px); /* 使用安全区域变量动态调整 */
	width: 50px;
	height: 50px;
	border-radius: 25px;
	background-color: #007aff;
	display: flex;
	justify-content: center;
	align-items: center;
	box-shadow: 0 2px 10px rgba(0, 0, 0, 0.2);
	z-index: 999; /* 确保按钮在最上层 */
}

.add-icon {
	color: #ffffff;
	font-size: 24px;
	font-weight: bold;
}
</style>