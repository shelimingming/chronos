<template>
	<view class="content">
		<image class="logo" src="/static/logo.png"></image>
		<view class="text-area">
			<text class="title">{{title}}</text>
		</view>
		
		<view class="button-group">
			<button type="primary" class="login-btn" @click="toLoginPage">登录</button>
			<button type="default" class="userinfo-btn" @click="toUserinfoPage">个人中心</button>
		</view>
	</view>
</template>

<script>
	import { store } from '@/uni_modules/uni-id-pages/common/store.js'
	export default {
		data() {
			return {
				title: 'Chronos 时间管理大师',
				userInfo: {}
			}
		},
		computed: {
			hasLogin() {
				return store.hasLogin
			}
		},
		onLoad() {
			this.userInfo = store.userInfo
		},
		methods: {
			toLoginPage() {
				uni.navigateTo({
					url: '/uni_modules/uni-id-pages/pages/login/login-withoutpwd'
				})
			},
			toUserinfoPage() {
				if (!this.hasLogin) {
					uni.showToast({
						title: '请先登录',
						icon: 'none'
					})
					setTimeout(() => {
						this.toLoginPage()
					}, 1500)
					return
				}
				uni.navigateTo({
					url: '/uni_modules/uni-id-pages/pages/userinfo/userinfo'
				})
			}
		}
	}
</script>

<style>
	.content {
		display: flex;
		flex-direction: column;
		align-items: center;
		justify-content: center;
	}

	.logo {
		height: 200rpx;
		width: 200rpx;
		margin-top: 200rpx;
		margin-left: auto;
		margin-right: auto;
		margin-bottom: 50rpx;
	}

	.text-area {
		display: flex;
		justify-content: center;
		margin-bottom: 50rpx;
	}

	.title {
		font-size: 36rpx;
		color: #8f8f94;
	}
	
	.button-group {
		width: 80%;
		margin-top: 50rpx;
	}
	
	.login-btn, .userinfo-btn {
		margin-bottom: 30rpx;
	}
</style>
