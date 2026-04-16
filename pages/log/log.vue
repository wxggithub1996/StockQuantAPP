<template>
    <view class="log-page">
        <scroll-view scroll-y class="log-container">
            <view v-for="(line, index) in logLines" :key="index" class="log-item">
                <text class="log-text">{{ line }}</text>
            </view>
            <view v-if="logLines.length === 0" class="empty">暂无日志内容</view>
        </scroll-view>
    </view>
</template>

<script>
export default {
    data() { return { logLines: [] } },
    onLoad() { this.fetchLogs(); },
    methods: {
        fetchLogs() {
            const baseUrl = uni.getStorageSync('quant_server_url');
            uni.request({
                url: baseUrl + '/api/logs',
                success: (res) => {
                    // 确保 res.data 是一个数组
                    if (Array.isArray(res.data)) {
                        this.logLines = res.data;
                    } else {
                        // 如果后端传的是字符串，手动转成数组
                        this.logLines = [res.data];
                    }
                }
            });
        }
    }
}
</script>