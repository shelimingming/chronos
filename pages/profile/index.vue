<template>
	<view class="container">
		<!-- 用户信息卡片 -->
		<view class="user-card">
			<view class="avatar-container">
				<uni-id-pages-avatar width="150rpx" height="150rpx"></uni-id-pages-avatar>
			</view>
			<view class="user-info">
				<text class="nickname">{{ userInfo.nickname || '未设置昵称' }}</text>
				<text class="user-id">ID: {{ userInfo._id || '未登录' }}</text>
			</view>
		</view>
		
		<!-- 数据统计卡片 -->
		<view class="stats-card">
			<view class="stats-item">
				<text class="stats-value">{{ activityCount }}</text>
				<text class="stats-label">活动记录</text>
			</view>
			<view class="stats-item">
				<text class="stats-value">{{ totalHours }}</text>
				<text class="stats-label">总时长(小时)</text>
			</view>
			<view class="stats-item">
				<text class="stats-value">{{ categoryCount }}</text>
				<text class="stats-label">活动类别</text>
			</view>
		</view>
		
		<!-- 功能列表 -->
		<uni-list class="menu-list">
			<uni-list-item title="个人资料" @click="goToUserInfo" showArrow link>
				<template v-slot:header>
					<view class="menu-icon">
						<uni-icons type="person" size="24" color="#007aff"></uni-icons>
					</view>
				</template>
			</uni-list-item>
			
			<uni-list-item title="我的活动记录" @click="goToMyActivities" showArrow link>
				<template v-slot:header>
					<view class="menu-icon">
						<uni-icons type="calendar" size="24" color="#007aff"></uni-icons>
					</view>
				</template>
			</uni-list-item>
			
			<uni-list-item title="数据分析" @click="goToAnalysis" showArrow link>
				<template v-slot:header>
					<view class="menu-icon">
						<uni-icons type="chart" size="24" color="#007aff"></uni-icons>
					</view>
				</template>
			</uni-list-item>
			
			<uni-list-item title="设置" @click="goToSettings" showArrow link>
				<template v-slot:header>
					<view class="menu-icon">
						<uni-icons type="gear" size="24" color="#007aff"></uni-icons>
					</view>
				</template>
			</uni-list-item>
		</uni-list>
		
		<!-- 登录/退出按钮 -->
		<view class="login-btn-container">
			<button v-if="!userInfo._id" class="login-btn" @click="login">登录/注册</button>
			<button v-else class="logout-btn" @click="logout">退出登录</button>
		</view>
	</view>
</template>

<script>
	import { store, mutations } from '@/uni_modules/uni-id-pages/common/store.js'
	
	export default {
		data() {
			return {
				activityCount: 0,
				totalHours: 0,
				categoryCount: 0
			};
		},
		computed: {
			userInfo() {
				return store.userInfo
			}
		},
		onLoad() {
			this.loadUserStats();
		},
		onShow() {
			// 每次页面显示时重新加载用户统计数据
			this.loadUserStats();
		},
		methods: {
			// 加载用户统计数据
			async loadUserStats() {
				if (!this.userInfo._id) return;
				
				try {
					const db = uniCloud.database();
					
					// 获取活动总数
					const activitiesRes = await db.collection('chronos-activities')
						.where({
							user_id: this.userInfo._id
						})
						.count();
					this.activityCount = activitiesRes.result.total;
					
					// 获取活动总时长
					const activitiesData = await db.collection('chronos-activities')
						.where({
							user_id: this.userInfo._id
						})
						.field({
							startTime: true,
							endTime: true
						})
						.get();
					
					// 计算总时长（小时）
					let totalMs = 0;
					activitiesData.result.data.forEach(activity => {
						if (activity.startTime && activity.endTime) {
							totalMs += activity.endTime - activity.startTime;
						}
					});
					this.totalHours = Math.round(totalMs / (1000 * 60 * 60) * 10) / 10; // 转换为小时并保留一位小数
					
					// 获取用户使用的类别数量
					const categoriesRes = await db.collection('chronos-activities')
						.where({
							user_id: this.userInfo._id
						})
						.field({
							categoryId: true
						})
						.get();
					
					// 计算不重复的类别数量
					const uniqueCategories = new Set();
					categoriesRes.result.data.forEach(activity => {
						if (activity.categoryId) {
							uniqueCategories.add(activity.categoryId);
						}
					});
					this.categoryCount = uniqueCategories.size;
					
				} catch (e) {
					console.error('加载用户统计数据失败:', e);
				}
			},
			
			// 跳转到用户信息页面
			goToUserInfo() {
				if (!this.userInfo._id) {
					this.showLoginTip();
					return;
				}
				uni.navigateTo({
					url: '/uni_modules/uni-id-pages/pages/userinfo/userinfo'
				});
			},
			
			// 跳转到我的活动记录页面
			goToMyActivities() {
				if (!this.userInfo._id) {
					this.showLoginTip();
					return;
				}
				// 目前跳转到首页，后续可以开发专门的我的活动页面
				uni.switchTab({
					url: '/pages/index/index'
				});
			},
			
			// 跳转到数据分析页面
			goToAnalysis() {
				if (!this.userInfo._id) {
					this.showLoginTip();
					return;
				}
				uni.switchTab({
					url: '/pages/statistics/index'
				});
			},
			
			// 跳转到设置页面
			goToSettings() {
				if (!this.userInfo._id) {
					this.showLoginTip();
					return;
				}
				// 目前没有专门的设置页面，暂时跳转到用户信息页
				uni.navigateTo({
					url: '/uni_modules/uni-id-pages/pages/userinfo/userinfo'
				});
			},
			
			// 显示登录提示
			showLoginTip() {
				uni.showToast({
					title: '请先登录',
					icon: 'none'
				});
			},
			
			// 登录
			login() {
				uni.navigateTo({
					url: '/uni_modules/uni-id-pages/pages/login/login-withoutpwd'
				});
			},
			
			// 退出登录
			logout() {
				mutations.logout();
				// 重置统计数据
				this.activityCount = 0;
				this.totalHours = 0;
				this.categoryCount = 0;
			}
		}
	}
</script>

<style>
.container {
	padding: 20px;
	display: flex;
	flex-direction: column;
	align-items: center;
	background-color: #f8f8f8;
	min-height: 100vh;
}

.user-card {
	width: 100%;
	background-color: #ffffff;
	border-radius: 15px;
	padding: 30rpx;
	display: flex;
	flex-direction: column;
	align-items: center;
	margin-bottom: 30rpx;
	box-shadow: 0 2px 12px rgba(0, 0, 0, 0.1);
}

.avatar-container {
	margin-bottom: 20rpx;
}

.user-info {
	display: flex;
	flex-direction: column;
	align-items: center;
	gap: 10rpx;
}

.nickname {
	font-size: 36rpx;
	font-weight: bold;
	color: #333;
}

.user-id {
	font-size: 24rpx;
	color: #999;
}

.stats-card {
	width: 100%;
	background-color: #ffffff;
	border-radius: 15px;
	padding: 30rpx;
	display: flex;
	justify-content: space-around;
	margin-bottom: 30rpx;
	box-shadow: 0 2px 12px rgba(0, 0, 0, 0.1);
}

.stats-item {
	display: flex;
	flex-direction: column;
	align-items: center;
	gap: 10rpx;
}

.stats-value {
	font-size: 40rpx;
	font-weight: bold;
	color: #007aff;
}

.stats-label {
	font-size: 24rpx;
	color: #666;
}

.menu-list {
	width: 100%;
	background-color: #ffffff;
	border-radius: 15px;
	overflow: hidden;
	margin-bottom: 30rpx;
	box-shadow: 0 2px 12px rgba(0, 0, 0, 0.1);
}

.menu-icon {
	margin-right: 20rpx;
}

.login-btn-container {
	width: 100%;
	padding: 30rpx;
}

.login-btn, .logout-btn {
	width: 100%;
	height: 80rpx;
	line-height: 80rpx;
	border-radius: 40rpx;
	font-size: 32rpx;
	font-weight: bold;
}

.login-btn {
	background-color: #007aff;
	color: #ffffff;
}

.logout-btn {
	background-color: #f5f5f5;
	color: #666;
	border: 2rpx solid #ddd;
}
</style>