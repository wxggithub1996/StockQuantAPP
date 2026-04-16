<template>
	<view class="container">
		<view class="header">
			<view class="icon-btn" @tap="toggleDrawer"><text class="iconfont">≡</text></view>
			<text class="title">量化终端</text>
			<view class="icon-btn" @tap="refreshAll"><text style="font-size: 14px;">🔄</text></view>
		</view>

		<view class="tabs">
			<view v-for="tab in tabList" :key="tab.status" class="tab" :class="{ active: currentStatus === tab.status }" @tap="switchTab(tab.status)">
				{{ tab.label }}({{ counts[tab.status] || 0 }})
			</view>
		</view>

		<view class="table-container">
			<view class="table-wrapper" v-if="displayStocks.length > 0">
				<view class="tr th">
					<view class="td sticky-left">名称/代码</view>
					<view class="td scroll-right">
						<view class="cell-sparkline"></view>
						<view class="cell-data sortable" @tap="toggleSort">涨幅 <text class="sort-icon">{{ sortIcon }}</text></view>
						<view class="cell-data">换手率</view>
						<view class="cell-data">成交额</view>
					</view>
				</view>

				<view class="tr" v-for="(item, index) in displayStocks" :key="index" @tap="openKline(item)">
					<view class="td sticky-left">
						<view class="left-icons">
							<text class="icon-drag">⋮⋮</text>
							<text class="icon-pin" :style="{opacity: item.pinned ? 1 : 0.2}" @tap.stop="pinTop(index)">📌</text>
						</view>
						<view class="stock-info">
							<text class="stock-name">{{ item.name || '--' }}</text>
							<view class="code-row">
								<text class="stock-code">{{ item.code || '--' }}</text>
								<text class="badge st" v-if="item.isST">ST</text>
								<text class="badge ke" v-if="item.code && item.code.startsWith('688')">科</text>
							</view>
						</view>
					</view>

					<view class="td scroll-right">
						<view class="cell-sparkline">
							<svg v-if="item.points" width="35" height="15" viewBox="0 0 35 15">
								<polyline :points="item.points" fill="none" :stroke="item.change > 0 ? '#e83828' : '#00a854'" stroke-width="1.5" />
							</svg>
						</view>
						<view class="cell-data compound-cell">
							<view class="change-block" :class="item.change > 0 ? 'bg-red' : 'bg-green'">
								{{ item.change > 0 ? '+' : '' }}{{ item.change !== undefined && !isNaN(item.change) ? parseFloat(item.change).toFixed(2) : '0.00' }}%
							</view>
							<text class="price-text">{{ item.price || '--' }}</text>
						</view>
						<view class="cell-data">{{ item.turnover || '--' }}%</view>
						<view class="cell-data">{{ item.volume || '--' }}</view>
					</view>
				</view>
			</view>
			<view class="empty-state" v-else><text>{{ isLoading ? '正在加载...' : '暂无数据' }}</text></view>
		</view>

		<view class="drawer-overlay" v-if="isDrawerOpen" @tap="toggleDrawer"></view>
		<view class="drawer-panel" :class="{'drawer-open': isDrawerOpen}">
			<view class="drawer-header">控制台</view>
			<view class="drawer-menu">
				<button class="menu-btn" @tap="handleAction('sync')">🔄 增量同步</button>
				<button class="menu-btn" @tap="handleAction('log')">📜 操作日志</button>
				<button class="menu-btn danger" @tap="handleAction('reset')">⚠️ 深度重置系统</button>
				<view class="menu-item-row">
					<text>显示 ST / 科创</text>
					<switch color="#1890ff" :checked="showSpecial" @change="showSpecial = !showSpecial" style="transform:scale(0.8)"/>
				</view>
				<button class="menu-btn outline" @tap="openConfigModal">⚙️ 配置服务器域名</button>
			</view>
		</view>

		<view class="modal-overlay" v-if="isModalOpen">
			<view class="modal-box">
				<view class="modal-title">服务器配置</view>
				<input class="modal-input" v-model="tempUrl" placeholder="http://xxxx.ngrok.io" />
				<view class="modal-actions">
					<button class="modal-btn cancel" @tap="isModalOpen = false">取消</button>
					<button class="modal-btn confirm" @tap="saveConfig">保存</button>
				</view>
			</view>
		</view>

		<view class="modal-overlay" v-if="isLogOpen">
			<view class="modal-box" style="height: 60vh; display: flex; flex-direction: column;">
				<view class="modal-title">📜 操作日志</view>
				<scroll-view scroll-y style="flex: 1; margin-bottom: 15px; font-size: 12px; color: #666;">
					<view style="padding: 10px; border-bottom: 1px dashed #eee;">正在向后端请求日志列表...</view>
				</scroll-view>
				<button class="modal-btn cancel" @tap="isLogOpen = false">关闭</button>
			</view>
		</view>

		<view class="kline-modal" :class="{'kline-open': isKlineOpen}">
			<view class="kline-header">
				<text class="kline-title">{{ currentStock.name }} ({{ currentStock.code }})</text>
				<text class="kline-close" @tap="isKlineOpen = false">关闭</text>
			</view>
			<view class="kline-body">
				<view style="text-align: center; margin-top: 50%; color: #999;">
					K线图表区域<br>
					请求接口: /api/kline/{{ currentStock.code }}
				</view>
			</view>
		</view>

	</view>
</template>

<script>
export default {
	data() {
		return {
			currentStatus: 1, 
			tabList: [ { label: '试盘', status: 1 }, { label: '回踩', status: 2 }, { label: '突破', status: 3 }, { label: '废弃', status: 99 } ],
			counts: { 1: 0, 2: 0, 3: 0, 99: 0 }, // 存储所有池子的数量
			sortState: 'default',
			isDrawerOpen: false,
			isModalOpen: false,
			isLogOpen: false,
			isKlineOpen: false,
			showSpecial: false, // 默认关闭ST/科创显示
			tempUrl: '',
			savedUrl: '',
			isLoading: false,
			allStocks: [],
			currentStock: {} // 当前点击查看的股票
		}
	},
	computed: {
		sortIcon() { return this.sortState === 'desc' ? '↓' : (this.sortState === 'asc' ? '↑' : ''); },
		displayStocks() {
			if (!Array.isArray(this.allStocks)) return [];
			let list = this.allStocks.filter(s => s.status === this.currentStatus);
			if (!this.showSpecial) list = list.filter(s => !s.isST && !(s.code && s.code.startsWith('688')));
			if (this.sortState === 'desc') list.sort((a, b) => parseFloat(b.change || 0) - parseFloat(a.change || 0));
			else if (this.sortState === 'asc') list.sort((a, b) => parseFloat(a.change || 0) - parseFloat(b.change || 0));
			return list;
		}
	},
	onLoad() {
		this.savedUrl = uni.getStorageSync('quant_server_url') || '';
		if(!this.savedUrl) {
			setTimeout(() => { this.isDrawerOpen = true; }, 500);
		} else {
			this.refreshAll();
		}
	},
	methods: {
		refreshAll() {
			this.fetchCounts();
			this.fetchDataFromPython();
		},
		// 独立获取各个池子的数量
		fetchCounts() {
			if (!this.savedUrl) return;
			uni.request({
				url: `${this.savedUrl}/api/counts`,
				method: 'GET',
				success: (res) => {
					if (res.statusCode === 200 && res.data) {
						this.counts = res.data; // 假设后端返回 {'1': 62, '2': 95, ...}
					}
				}
			});
		},
		fetchDataFromPython() {
			if (!this.savedUrl) return;
			this.isLoading = true;
			uni.request({
				url: `${this.savedUrl}/api/pool/app/${this.currentStatus}`,
				method: 'GET',
				success: (res) => {
					if (res.statusCode === 200 && Array.isArray(res.data)) {
						this.allStocks = res.data.map(item => {
							const changeVal = parseFloat(item.pct_change || item.change || 0);
							return {
								name: item.name, code: item.code,
								price: item.price, change: changeVal, 
								turnover: item.turnover, volume: item.volume,
								status: this.currentStatus,
								isST: item.name ? item.name.includes('ST') : false,
								pinned: false,
								points: item.sparkline || (changeVal > 0 ? "0,15 15,10 25,12 35,0" : "0,0 15,5 25,2 35,15")
							}
						});
					} else { this.allStocks = []; }
				},
				complete: () => { this.isLoading = false; }
			});
		},
		switchTab(status) {
			this.currentStatus = status;
			this.sortState = 'default';
			this.fetchDataFromPython(); // 切换时只刷新列表数据，不刷 count 以免闪烁
		},
		toggleSort() {
			const states = ['default', 'desc', 'asc'];
			this.sortState = states[(states.indexOf(this.sortState) + 1) % 3];
		},
		toggleDrawer() { this.isDrawerOpen = !this.isDrawerOpen; },
		openConfigModal() {
			this.tempUrl = uni.getStorageSync('quant_server_url') || '';
			this.isDrawerOpen = false; // 修复1: 打开弹窗时关闭侧边栏
			this.isModalOpen = true;
		},
		saveConfig() {
			if (this.tempUrl.endsWith('/')) this.tempUrl = this.tempUrl.slice(0, -1);
			uni.setStorageSync('quant_server_url', this.tempUrl);
			this.savedUrl = this.tempUrl;
			this.isModalOpen = false;
			this.refreshAll();
		},
		// 点击股票行，打开K线
		openKline(stock) {
		    // ⚠️ 极其关键：必须用 encodeURIComponent 将中文和 * 号等特殊字符转码！
		    const safeName = encodeURIComponent(stock.name);
		    
		    uni.navigateTo({
		        url: `/pages/kline/kline?code=${stock.code}&name=${safeName}`,
		        fail: (err) => {
		            console.error("【跳转失败】", err);
		            uni.showToast({ title: '跳转失败，请重试', icon: 'none' });
		        }
		    });
		},
		// 点击图钉置顶
		pinTop(index) {
			// 在前端数组里把这只票移到最前面
			const stock = this.allStocks.splice(index, 1)[0];
			stock.pinned = !stock.pinned;
			this.allStocks.unshift(stock);
		},
		handleAction(actionType) {
		   this.isDrawerOpen = false;
		       if (actionType === 'log') {
		           uni.navigateTo({
		               url: '/pages/log/log',
		               fail: (err) => { console.error("日志页跳转失败:", err); }
		           });
			} else {
				let apiEndpoint = actionType === 'sync' ? '/api/run_strategy' : '/api/reset_bootstrap';
				uni.showLoading({ title: '执行中...' });
				uni.request({
					url: this.savedUrl + apiEndpoint,
					method: 'POST',
					success: () => { uni.showToast({ title: '成功' }); this.refreshAll(); },
					complete: () => { uni.hideLoading(); }
				});
			}
		}
	}
}
</script>

<style scoped>
.container { width: 100vw; height: 100vh; display: flex; flex-direction: column; background: #fff; overflow: hidden; }
.header { display: flex; justify-content: space-between; align-items: center; height: 44px; padding: 0 15px; background: #f8f9fa; border-bottom: 1px solid #eee; flex-shrink: 0; }
.tabs { display: flex; border-bottom: 1px solid #eee; background: #fff; flex-shrink: 0; }
.tab { flex: 1; text-align: center; padding: 12px 0; font-size: 12px; color: #666; transition: 0.2s; }
.tab.active { color: #1890ff; font-weight: bold; position: relative; }
.tab.active::after { content: ''; position: absolute; bottom: 0; left: 20%; width: 60%; height: 3px; background: #1890ff; border-radius: 2px; }

.table-container { flex: 1; overflow: hidden; position: relative; }
.table-wrapper { height: 100%; overflow: auto; display: flex; flex-direction: column; }
.tr { display: flex; width: max-content; min-width: 100%; border-bottom: 1px solid #f5f5f5; }
.th { background: #fafafa; position: sticky; top: 0; z-index: 20; height: 35px; }

.sticky-left { position: sticky; left: 0; background: #fff; z-index: 10; width: 130px; flex-shrink: 0; display: flex; align-items: center; border-right: 1px solid #f0f0f0; padding-left: 10px; }
.th .sticky-left { background: #fafafa; z-index: 11; }
.scroll-right { display: flex; }
.cell-data { width: 80px; text-align: center; display: flex; justify-content: center; align-items: center; flex-shrink: 0; font-size: 13px; }
.cell-sparkline { width: 45px; flex-shrink: 0; display: flex; align-items: center; justify-content: center; }

.compound-cell { flex-direction: column; padding: 6px 0; width: 80px; }
.change-block { width: 62px; border-radius: 3px; color: #fff; font-size: 12px; font-weight: bold; padding: 2px 0; text-align: center; }
.bg-red { background: #e83828; }
.bg-green { background: #00a854; }
.price-text { font-size: 11px; color: #999; margin-top: 2px; }

.stock-info { display: flex; flex-direction: column; }
.stock-name { font-size: 14px; font-weight: bold; color: #333; }
.code-row { display: flex; align-items: center; }
.stock-code { font-size: 10px; color: #999; margin-right: 4px; }
.badge { font-size: 9px; border: 1px solid; padding: 0 2px; border-radius: 2px; margin-left: 2px; line-height: 1; }
.badge.st { border-color: #faad14; color: #faad14; }
.badge.ke { border-color: #1890ff; color: #1890ff; }
.left-icons { display: flex; flex-direction: column; margin-right: 6px; font-size: 10px; color: #ccc; }

.empty-state { padding: 50px; text-align: center; color: #999; font-size: 14px; }

/* 侧边栏 */
.drawer-overlay { position: fixed; top: 0; left: 0; width: 100vw; height: 100vh; background: rgba(0,0,0,0.5); z-index: 100; }
.drawer-panel { position: fixed; top: 0; left: -260px; width: 240px; height: 100vh; background: #fff; z-index: 101; transition: 0.3s; padding: 40px 15px; }
.drawer-open { left: 0; }
.menu-btn { width: 100%; margin-bottom: 12px; font-size: 14px; background: #f5f5f5; border: none; }

/* 修复1: 提升弹窗层级，遮盖所有元素 */
.modal-overlay { position: fixed; top: 0; left: 0; width: 100vw; height: 100vh; background: rgba(0,0,0,0.6); z-index: 9999; display: flex; justify-content: center; align-items: center; }
.modal-box { width: 80%; background: #fff; border-radius: 12px; padding: 20px; box-shadow: 0 4px 20px rgba(0,0,0,0.2); }
.modal-input { border: 1px solid #ddd; padding: 10px; border-radius: 6px; margin: 15px 0; width: 90%; }
.modal-actions { display: flex; gap: 10px; }
.modal-btn { flex: 1; font-size: 14px; }

/* 详情 K 线弹窗 */
.kline-modal { position: fixed; bottom: -100%; left: 0; width: 100vw; height: 90vh; background: #fff; z-index: 8888; transition: bottom 0.3s; border-radius: 15px 15px 0 0; box-shadow: 0 -2px 20px rgba(0,0,0,0.2); display: flex; flex-direction: column; }
.kline-open { bottom: 0; }
.kline-header { padding: 15px 20px; border-bottom: 1px solid #eee; display: flex; justify-content: space-between; align-items: center; }
.kline-title { font-size: 16px; font-weight: bold; }
.kline-close { color: #1890ff; font-size: 14px; }
.kline-body { flex: 1; background: #f9f9f9; }
</style>