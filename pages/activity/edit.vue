<template>
	<view class="container">
		<view class="form-container">
			<view class="form-item">
				<text class="form-label">活动标题</text>
				<input class="form-input" type="text" v-model="title" placeholder="请输入活动标题" />
			</view>
			
			<view class="form-item">
				<text class="form-label">开始时间</text>
				<picker class="form-picker" mode="time" :value="startTime" @change="onStartTimeChange">
					<view class="picker-value">
						<text>{{ startTime || '请选择开始时间' }}</text>
					</view>
				</picker>
			</view>
			
			<view class="form-item">
				<text class="form-label">结束时间</text>
				<picker class="form-picker" mode="time" :value="endTime" @change="onEndTimeChange">
					<view class="picker-value">
						<text>{{ endTime || '请选择结束时间' }}</text>
					</view>
				</picker>
			</view>
			
			<view class="form-item">
				<text class="form-label">持续时间</text>
				<view class="duration-display">
					<text>{{ duration }} 分钟</text>
				</view>
			</view>
			
			<view class="form-item">
				<text class="form-label">活动类别</text>
				<picker class="form-picker" :range="categories" range-key="name" :value="categoryIndex" @change="onCategoryChange">
					<view class="picker-value">
						<text>{{ selectedCategory ? selectedCategory.name : '请选择活动类别' }}</text>
					</view>
				</picker>
			</view>
			
			<view class="form-item">
				<text class="form-label">备注</text>
				<textarea class="form-textarea" v-model="notes" placeholder="请输入备注信息" />
			</view>
			
			<view class="form-actions">
				<button class="save-button" @click="save">保存</button>
				<button class="cancel-button" @click="cancel">取消</button>
			</view>
		</view>
	</view>
</template>

<script>
	export default {
		data() {
			// 获取当前时间并格式化为HH:mm格式
			const now = new Date();
			const currentTime = `${String(now.getHours()).padStart(2, '0')}:${String(now.getMinutes()).padStart(2, '0')}`;
			return {
				activityId: '', // 添加活动ID
				title: '',
				startTime: currentTime,
				endTime: currentTime,
				duration: 0,
				categories: [],
				categoryIndex: 0,
				selectedCategory: null,
				notes: ''
			}
		},
		onLoad(options) {
			this.loadCategories();
			// 如果传入了活动ID，加载活动数据
			if (options && options.id) {
				this.activityId = options.id; // 保存活动ID
				this.loadActivity(options.id);
			}
		},
		methods: {
			// 加载活动数据
			async loadActivity(id) {
				try {
					const db = uniCloud.database();
					const activitiesCollection = db.collection('chronos-activities');
					const res = await activitiesCollection.doc(id).get();
					const activity = res.result.data;
					
					if (activity) {
						this.title = activity.title;
						// 使用本地时区处理时间
						// 确保时间戳是有效的数字
						const startTimestamp = typeof activity.start_time === 'object' && activity.start_time.$date ? 
							activity.start_time.$date : Number(activity.start_time);
						const endTimestamp = typeof activity.end_time === 'object' && activity.end_time.$date ? 
							activity.end_time.$date : Number(activity.end_time);
							
						const startDate = new Date(startTimestamp);
						const endDate = new Date(endTimestamp);
						
						// 检查日期是否有效
						if (!isNaN(startDate.getTime()) && !isNaN(endDate.getTime())) {
							// 格式化时间为HH:mm格式
							this.startTime = `${String(startDate.getHours()).padStart(2, '0')}:${String(startDate.getMinutes()).padStart(2, '0')}`;
							this.endTime = `${String(endDate.getHours()).padStart(2, '0')}:${String(endDate.getMinutes()).padStart(2, '0')}`;
						} else {
							// 如果日期无效，使用当前时间
							const now = new Date();
							this.startTime = `${String(now.getHours()).padStart(2, '0')}:${String(now.getMinutes()).padStart(2, '0')}`;
							this.endTime = `${String(now.getHours()).padStart(2, '0')}:${String(now.getMinutes()).padStart(2, '0')}`;
							console.error('无效的日期格式', activity.start_time, activity.end_time);
						}
						
						this.duration = activity.duration;
						this.notes = activity.notes;
						
						// 设置选中的类别
						const categoryIndex = this.categories.findIndex(c => c._id === activity.category);
						if (categoryIndex !== -1) {
							this.categoryIndex = categoryIndex;
							this.selectedCategory = this.categories[categoryIndex];
						}
					}
				}
				catch (e) {
					console.error('加载活动数据失败:', e);
					uni.showToast({
						title: '加载活动数据失败',
						icon: 'none'
					});
				}
			},
			
			// 加载活动类别
			async loadCategories() {
				try {
					const db = uniCloud.database();
					const categoriesCollection = db.collection('chronos-categories');
					const res = await categoriesCollection.get();
					this.categories = res.result.data || [];
					// 如果有类别数据，自动选择第一个类别
					if (this.categories.length > 0) {
						this.categoryIndex = 0;
						this.selectedCategory = this.categories[0];
					}
				} catch (e) {
					console.error('加载类别失败:', e);
					uni.showToast({
						title: '加载类别失败',
						icon: 'none'
					});
				}
			},
			
			// 开始时间变化
			onStartTimeChange(e) {
				this.startTime = e.detail.value;
				this.calculateDuration();
			},
			
			// 结束时间变化
			onEndTimeChange(e) {
				this.endTime = e.detail.value;
				this.calculateDuration();
			},
			
			// 计算持续时间
			calculateDuration() {
				if (this.startTime && this.endTime) {
					const start = new Date(`2000/01/01 ${this.startTime}`);
					const end = new Date(`2000/01/01 ${this.endTime}`);
					this.duration = Math.round((end - start) / 1000 / 60);
					
					// 如果结束时间小于开始时间，认为是跨天，加上24小时
					if (this.duration < 0) {
						this.duration += 24 * 60;
					}
				}
			},
			
			// 类别选择变化
			onCategoryChange(e) {
				this.categoryIndex = e.detail.value;
				this.selectedCategory = this.categories[this.categoryIndex];
			},
			
			// 保存活动
			async save() {
				if (!this.title) {
					uni.showToast({
						title: '请输入活动标题',
						icon: 'none'
					});
					return;
				}
				
				if (!this.startTime || !this.endTime) {
					uni.showToast({
						title: '请选择开始和结束时间',
						icon: 'none'
					});
					return;
				}
				
				if (!this.selectedCategory) {
					uni.showToast({
						title: '请选择活动类别',
						icon: 'none'
					});
					return;
				}
				
				try {
					const db = uniCloud.database();
					const activitiesCollection = db.collection('chronos-activities');
					
					// 构建当天的完整时间
					const today = new Date();
					const [startHours, startMinutes] = this.startTime.split(':').map(Number);
					const [endHours, endMinutes] = this.endTime.split(':').map(Number);

					// 使用本地时区创建时间
					const startTime = new Date(today.getFullYear(), today.getMonth(), today.getDate(), startHours, startMinutes, 0);
					const endTime = new Date(today.getFullYear(), today.getMonth(), today.getDate(), endHours, endMinutes, 0);
					// 确保时间戳使用本地时区
					
					// 如果结束时间小于开始时间，认为是跨天，结束时间加一天
					if (endTime < startTime) {
						endTime.setDate(endTime.getDate() + 1);
					}
					
					const activity = {
						title: this.title,
						start_time: startTime.getTime(),
						end_time: endTime.getTime(),
						duration: this.duration,
						category: this.selectedCategory._id,
						notes: this.notes
					};
					
					if (this.activityId) {
						await activitiesCollection.doc(this.activityId).update(activity);
					} else {
						await activitiesCollection.add(activity);
					}
					
					uni.showToast({
						title: '保存成功',
						icon: 'success',
						duration: 1500,
						success: () => {
							setTimeout(() => {
								uni.navigateBack();
							}, 1500);
						}
					});
					
				} catch (e) {
					console.error('保存活动失败:', e);
					uni.showToast({
						title: '保存失败',
						icon: 'none'
					});
				}
			},
			
			// 取消编辑，返回首页
			cancel() {
				uni.navigateBack();
			}
		}
	}
</script>

<style>
.container {
	padding: 20px;
	background-color: #f8f8f8;
	height: 100vh;
}

.form-container {
	background-color: #ffffff;
	border-radius: 10px;
	padding: 20px;
	box-shadow: 0 2px 10px rgba(0, 0, 0, 0.05);
}

.form-item {
	margin-bottom: 20px;
}

.form-label {
	display: block;
	font-size: 16px;
	color: #333;
	margin-bottom: 8px;
	font-weight: bold;
}

.form-input {
	width: 100%;
	height: 45px;
	border: 1px solid #e0e0e0;
	border-radius: 5px;
	padding: 0 15px;
	font-size: 16px;
	background-color: #f9f9f9;
}

.form-picker {
	width: 100%;
}

.picker-value {
	height: 45px;
	border: 1px solid #e0e0e0;
	border-radius: 5px;
	padding: 0 15px;
	font-size: 16px;
	background-color: #f9f9f9;
	display: flex;
	align-items: center;
}

.placeholder {
	color: #999;
}

.duration-display {
	height: 45px;
	border: 1px solid #e0e0e0;
	border-radius: 5px;
	padding: 0 15px;
	font-size: 16px;
	background-color: #f9f9f9;
	display: flex;
	align-items: center;
	color: #333;
}

.form-textarea {
	width: 100%;
	height: 100px;
	border: 1px solid #e0e0e0;
	border-radius: 5px;
	padding: 10px 15px;
	font-size: 16px;
	background-color: #f9f9f9;
}

.form-actions {
	display: flex;
	justify-content: space-between;
	margin-top: 30px;
}

.save-button {
	width: 45%;
	height: 45px;
	background-color: #4CAF50;
	color: white;
	border-radius: 5px;
	font-size: 16px;
}

.cancel-button {
	width: 45%;
	height: 45px;
	background-color: #f5f5f5;
	color: #333;
	border-radius: 5px;
	font-size: 16px;
}
</style>