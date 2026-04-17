<template>
	<view class="page-container" style="width: 100vw; height: 100vh;">
		<web-view v-if="targetUrl" :src="targetUrl" @message="handleMessage"></web-view>
	</view>
</template>

<script>
export default {
	data() { 
		return { targetUrl: '' } 
	},
	onLoad(options) {
		const baseUrl = uni.getStorageSync('quant_server_url');
		if (!baseUrl) {
			uni.showToast({ title: '服务器地址丢失', icon: 'none' });
			return;
		}
		
		let safeName = '行情详情';
		try {
			if (options.name) safeName = decodeURIComponent(options.name);
		} catch (e) { console.error("转码异常", e); }
		
		uni.setNavigationBarTitle({ title: safeName });
		const status = options.status || 1;
		const urlSafeName = encodeURIComponent(safeName);
		
		this.targetUrl = `${baseUrl}/static/app_kline.html?code=${options.code}&name=${urlSafeName}&status=${status}`;
	},
	// 🚨 核心防御：当用户点击返回键离开 K 线页时，必须把状态栏恢复出来！
	onUnload() {
		// #ifdef APP-PLUS
		plus.navigator.setFullscreen(false);
		// #endif
	},
	methods: {
		// 🚨 接收 HTML 发来的全屏指令
		handleMessage(event) {
			const data = event.detail.data[0];
			if (data && data.action === 'enter_fullscreen') {
				// 进入横屏 -> 隐藏手机状态栏
				// #ifdef APP-PLUS
				plus.navigator.setFullscreen(true);
				// #endif
			} else if (data && data.action === 'exit_fullscreen') {
				// 恢复竖屏 -> 显示手机状态栏
				// #ifdef APP-PLUS
				plus.navigator.setFullscreen(false);
				// #endif
			}
		}
	}
}
</script>