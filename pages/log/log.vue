<template>
	<view class="log-page">
		<view class="global-controls">
			<view class="control-btn" @tap="expandAll">🔓 展开全部</view>
			<view class="control-btn" @tap="collapseAll">🔒 收起全部</view>
		</view>

		<scroll-view scroll-y class="log-container">
			<view v-for="(logs, date) in logData" :key="date" class="log-date-group">
				<view class="log-date-title" @tap="toggleDate(date)">
					<text>🗓️ {{ date }}</text>
					<text class="fold-icon">{{ collapsedMap[date] ? '➕' : '➖' }}</text>
				</view>
				
				<view v-if="!collapsedMap[date]">
					<view v-for="(log, index) in logs" :key="index" 
						class="log-card" 
						:class="'source-' + (log.source || 'default')">
						
						<view class="log-card-header">
							<text class="source-tag">{{ getSourceIcon(log.source) }} {{ log.source }}</text>
							<text class="log-time">{{ log.time }}</text>
						</view>

						<view class="log-card-body">
							<view class="stock-info">
								<text class="stock-name">{{ log.name || '未知股票' }}</text>
								<text class="stock-code">({{ log.code || '----' }})</text>
								<text v-if="log.code !== 'ALL'" class="btn-quick-look" @tap.stop="openKline(log)">📊 看图</text>
							</view>
							<text class="log-detail">{{ log.detail }}</text>
						</view>

						<view class="log-card-footer" v-if="log.code !== 'ALL'">
							<picker @change="onStatusChange($event, log)" :range="statusRange" range-key="label">
								<view class="status-picker">
									⚡ 上帝模式：强制变更为 <text class="picker-arrow">▼</text>
								</view>
							</picker>
						</view>
					</view>
				</view>
			</view>
			
			<view v-if="Object.keys(logData).length === 0" class="empty">暂无操作记录</view>
			<view style="height: 50px;"></view>
		</scroll-view>
	</view>
</template>

<script>
export default {
	data() {
		return {
			logData: {},
			collapsedMap: {}, // 记录折叠状态
			statusRange: [
				{ label: '👀 放入试盘池 (1)', value: 1 },
				{ label: '📉 放入回踩池 (2)', value: 2 },
				{ label: '🔥 放入突破池 (3)', value: 3 },
				{ label: '🗑️ 移入废弃池 (99)', value: 99 }
			]
		}
	},
	onLoad() {
		this.fetchLogs();
	},
	methods: {
		fetchLogs() {
			const baseUrl = uni.getStorageSync('quant_server_url');
			if (!baseUrl) return;
			
			uni.showLoading({ title: '同步审计日志...' });
			uni.request({
				url: baseUrl + '/api/logs',
				success: (res) => {
					// 确保数据结构正确
					this.logData = res.data || {};
					// 初始化折叠状态：默认展开
					Object.keys(this.logData).forEach(date => {
						if (this.collapsedMap[date] === undefined) {
							this.collapsedMap[date] = false;
						}
					});
				},
				fail: () => { uni.showToast({ title: '网络请求失败', icon: 'none' }); },
				complete: () => { uni.hideLoading(); }
			});
		},
		getSourceIcon(source) {
			const icons = { '自动处理': '🤖', '按钮处理': '🖱️', '手工干预': '🖐️' };
			return icons[source] || '📝';
		},
		toggleDate(date) {
			this.collapsedMap[date] = !this.collapsedMap[date];
		},
		expandAll() {
			Object.keys(this.collapsedMap).forEach(date => { this.collapsedMap[date] = false; });
		},
		collapseAll() {
			Object.keys(this.collapsedMap).forEach(date => { this.collapsedMap[date] = true; });
		},
		openKline(log) {
			const safeName = encodeURIComponent(log.name);
			uni.navigateTo({ url: `/pages/kline/kline?code=${log.code}&name=${safeName}` });
		},
		onStatusChange(e, log) {
			const selected = this.statusRange[e.detail.value];
			uni.showModal({
				title: '上帝视角确认',
				content: `确定要将 ${log.name} 强制转移到 [${selected.label}] 吗？`,
				success: (res) => {
					if (res.confirm) { this.executeStatusUpdate(log.code, selected.value); }
				}
			});
		},
		executeStatusUpdate(code, newStatus) {
			const baseUrl = uni.getStorageSync('quant_server_url');
			uni.request({
				url: baseUrl + '/api/update_status',
				method: 'POST',
				data: { code: code, new_status: newStatus, source: 'App上帝模式' },
				success: () => {
					uni.showToast({ title: '强制覆盖成功' });
					this.fetchLogs();
				}
			});
		}
	}
}
</script>

<style scoped>
.log-page { background: #f4f5f7; min-height: 100vh; display: flex; flex-direction: column; }
.global-controls { 
	display: flex; 
	justify-content: center; /* 按钮居中对齐 */
	padding: 12px; 
	background: #fff; 
	border-bottom: 1px solid #eee; 
	gap: 20px;
}
.control-btn { 
	font-size: 13px; 
	color: #1890ff; 
	padding: 5px 15px; 
	border: 1px solid #1890ff; 
	border-radius: 20px; 
	background: #f0f7ff;
}

.log-container { flex: 1; padding: 0 15px; box-sizing: border-box; }
.log-date-group { margin-top: 10px; }
.log-date-title { 
	display: flex; 
	justify-content: space-between; 
	align-items: center;
	font-size: 15px; 
	font-weight: bold; 
	color: #666; 
	padding: 12px 5px; 
	border-bottom: 1px solid #ddd;
}
.fold-icon { font-size: 12px; color: #bbb; }

.log-card { 
	background: #fff; border-radius: 8px; padding: 12px; margin: 10px 0;
	box-shadow: 0 2px 8px rgba(0,0,0,0.06); border-left: 5px solid #ccc;
}
.source-自动处理 { border-left-color: #1890ff; }
.source-按钮处理 { border-left-color: #faad14; }
.source-手工干预 { border-left-color: #ff4d4f; }

.log-card-header { display: flex; justify-content: space-between; margin-bottom: 8px; }
.source-tag { font-size: 11px; color: #888; }
.log-time { font-size: 11px; color: #bbb; }

.stock-info { display: flex; align-items: center; margin-bottom: 5px; }
.stock-name { font-size: 15px; font-weight: bold; color: #333; margin-right: 5px; }
.stock-code { font-size: 11px; color: #999; }
.btn-quick-look { font-size: 11px; color: #1890ff; background: #e6f7ff; padding: 2px 8px; border-radius: 4px; margin-left: auto; }

.log-detail { font-size: 13px; color: #d46b08; font-weight: bold; line-height: 1.4; display: block; margin: 5px 0; }

.log-card-footer { border-top: 1px dashed #eee; padding-top: 10px; margin-top: 5px; }
.status-picker { font-size: 12px; color: #777; background: #fafafa; padding: 8px; border-radius: 4px; text-align: center; }
.empty { text-align: center; padding: 100px 0; color: #999; font-size: 14px; }
</style>