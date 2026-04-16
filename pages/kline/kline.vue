<template>
	<view class="page-container">
		<web-view v-if="targetUrl" :src="targetUrl"></web-view>
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
		
		// 动态拼接代码
		this.targetUrl = `${baseUrl}/static/app_kline.html?code=${options.code}`;
		uni.setNavigationBarTitle({ title: decodeURIComponent(options.name) || '行情详情' });
	}
}
</script>