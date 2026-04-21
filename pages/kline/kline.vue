<template>
	<view class="app-container" :class="{ 'is-landscape': isLandscape }">
		<view class="sidebar" :class="{ collapsed: isSidebarCollapsed }">
			<view class="header-group">
				<picker @change="onStatusChange" :value="statusIdx" :range="statusOptions">
					<view class="group-selector">
						<text>{{ statusOptions[statusIdx] }}</text>
						<text class="arrow">▼</text>
					</view>
				</picker>
			</view>

			<scroll-view scroll-y class="stock-list">
				<view
					v-for="(item, index) in stockList"
					:key="item.code"
					class="stock-item"
					:class="{ active: item.code === currentCode, 'is-dragging': draggingIndex === index }"
					@tap="switchStock(item)"
				>
					<view class="stock-item-main">
						<view class="stock-texts">
							<text class="s-name">{{ item.name }}</text>
							<view class="code-line">
								<text class="s-code">{{ item.code }}</text>
								<text v-if="item.is_new" class="new-badge">新</text>
							</view>
						</view>
						<view class="stock-actions">
							<text class="pin-btn" :class="{ active: item.pinned }" @tap.stop="pinTop(item.code)">置顶</text>
							<text
								class="drag-handle"
								@touchstart.stop.prevent="onDragStart($event, index)"
								@touchmove.stop.prevent="onDragMove"
								@touchend.stop.prevent="onDragEnd"
							>⋮⋮</text>
						</view>
					</view>
				</view>
			</scroll-view>
		</view>

		<view class="main" :class="{ 'sidebar-collapsed': isSidebarCollapsed }">
			<view class="toggle-sidebar-btn" @tap="toggleSidebar">
				{{ isSidebarCollapsed ? '▶' : '◀' }}
			</view>

			<view class="loading-tips" v-if="loadingText">{{ loadingText }}</view>

			<view class="header">
				<text class="stock-title">{{ currentName }} ({{ currentCode }})</text>
				<view class="btn-rotate" @tap="toggleLandscape">
					{{ isLandscape ? '竖屏恢复' : '横屏模式' }}
				</view>
			</view>

			<view class="landscape-banner" v-if="isLandscape">
				<view class="lcd-block">
					<text class="lcd-price" :class="lcdColorClass">{{ lcdData.close }}</text>
					<text class="lcd-change" :class="lcdColorClass">{{ lcdData.sign }}{{ lcdData.pctChg }}%</text>
				</view>
				<view class="lcd-block">
					<view class="lcd-row">高 <text :class="lcdData.highColor">{{ lcdData.high }}</text></view>
					<view class="lcd-row">低 <text :class="lcdData.lowColor">{{ lcdData.low }}</text></view>
				</view>
				<view class="lcd-block">
					<view class="lcd-row">开 <text class="val-normal">{{ lcdData.open }}</text></view>
					<view class="lcd-row">换 <text class="val-normal">{{ lcdData.turn }}%</text></view>
				</view>
				<view class="lcd-block">
					<view class="lcd-row">量 <text class="val-normal">{{ lcdData.volHand }}</text></view>
					<view class="lcd-row">额 <text class="val-normal">{{ lcdData.amt }}</text></view>
				</view>
			</view>

			<view
				id="echarts-dom"
				class="chart-container"
				:prop="chartRawData"
				:change:prop="echart.updateData"
				:landscape="isLandscape"
				:change:landscape="echart.updateLandscape"
				:collapsed="isSidebarCollapsed"
				:change:collapsed="echart.updateCollapsed"
			></view>
		</view>
	</view>
</template>

<script>
export default {
	data() {
		return {
			savedUrl: '',
			currentCode: '',
			currentName: '行情详情',
			currentStatus: 1,
			statusOptions: ['试盘池', '回踩池', '突破池', '废弃池'],
			statusValues: [1, 2, 3, 99],
			statusIdx: 0,
			isLandscape: false,
			isSidebarCollapsed: true,
			loadingText: '',
			stockList: [],
			chartRawData: null,
			draggingIndex: -1,
			dragStartY: 0,
			lcdData: {
				open: '--',
				close: '--',
				high: '--',
				low: '--',
				pctChg: '--',
				sign: '',
				turn: '--',
				volHand: '--',
				amt: '--',
				highColor: 'val-normal',
				lowColor: 'val-normal'
			}
		};
	},
	computed: {
		lcdColorClass() {
			if (this.lcdData.pctChg === '--') return 'val-normal';
			const chg = parseFloat(this.lcdData.pctChg);
			return chg > 0 ? 'val-red' : (chg < 0 ? 'val-green' : 'val-normal');
		}
	},
	onLoad(options) {
		this.savedUrl = uni.getStorageSync('quant_server_url') || '';
		if (!this.savedUrl) {
			uni.showToast({ title: '服务器地址丢失', icon: 'none' });
			return;
		}

		this.currentCode = options.code || '';
		try {
			this.currentName = decodeURIComponent(options.name || '行情详情');
		} catch (e) {}
		this.currentStatus = parseInt(options.status || 1, 10);
		this.syncStatusIdx();

		this.loadStockList({ preferCode: this.currentCode });
		this.fetchKLineData();
	},
	onResize(res) {
		if (res.size.windowWidth > res.size.windowHeight) {
			this.isLandscape = true;
			this.isSidebarCollapsed = false;
			// #ifdef APP-PLUS
			plus.navigator.setFullscreen(true);
			// #endif
		} else {
			this.isLandscape = false;
			this.isSidebarCollapsed = true;
			// #ifdef APP-PLUS
			plus.navigator.setFullscreen(false);
			// #endif
		}
	},
	onUnload() {
		// #ifdef APP-PLUS
		plus.navigator.setFullscreen(false);
		plus.screen.lockOrientation('portrait-primary');
		setTimeout(() => plus.screen.unlockOrientation(), 200);
		// #endif
	},
	methods: {
		syncStatusIdx() {
			const idx = this.statusValues.indexOf(Number(this.currentStatus));
			this.statusIdx = idx > -1 ? idx : 0;
		},
		normalizeStocks(rawData) {
			if (!Array.isArray(rawData)) return [];
			return rawData.map(item => ({
				name: item.name,
				code: item.code,
				price: item.price || '--',
				change: parseFloat(item.change || item.pct_change || 0),
				turnover: item.turnover || '--',
				volumeHand: item.volumeHand || '--',
				amountFmt: item.amountFmt || item.amount || '--',
				status: item.status || this.currentStatus,
				isST: item.isST === true || (item.name ? item.name.includes('ST') : false),
				is_new: item.is_new === true,
				pinned: item.pinned === true
			}));
		},
		persistCurrentListCache() {
			uni.setStorageSync(`pool_data_${this.currentStatus}`, this.stockList);
		},
		formatVol(v) {
			if (!v) return '--';
			if (v >= 100000000) return (v / 100000000).toFixed(2) + '亿';
			if (v >= 10000) return (v / 10000).toFixed(0) + '万';
			return v.toFixed(0);
		},
		formatHand(v) {
			if (!v) return '--';
			const hands = v / 100;
			if (hands >= 10000) return (hands / 10000).toFixed(2) + '万手';
			return hands.toFixed(0) + '手';
		},
		toggleSidebar() {
			this.isSidebarCollapsed = !this.isSidebarCollapsed;
		},
		toggleLandscape() {
			this.isLandscape = !this.isLandscape;
			// #ifdef APP-PLUS
			if (this.isLandscape) {
				plus.navigator.setFullscreen(true);
				plus.screen.lockOrientation('landscape-primary');
				this.isSidebarCollapsed = false;
			} else {
				plus.navigator.setFullscreen(false);
				plus.screen.lockOrientation('portrait-primary');
				this.isSidebarCollapsed = true;
			}
			// #endif
		},
		onStatusChange(e) {
			this.statusIdx = Number(e.detail.value);
			this.currentStatus = this.statusValues[this.statusIdx];
			this.loadStockList();
		},
		loadStockList({ preferCode } = {}) {
			const cacheKey = `pool_data_${this.currentStatus}`;
			const cachedList = uni.getStorageSync(cacheKey);
			if (cachedList && Array.isArray(cachedList) && cachedList.length > 0) {
				this.stockList = this.normalizeStocks(cachedList);
				this.afterStockListLoaded(preferCode);
				return;
			}

			uni.request({
				url: `${this.savedUrl}/api/pool/app/${this.currentStatus}`,
				header: {
					"ngrok-skip-browser-warning": "69420"
				},
				success: (res) => {
					this.stockList = res.statusCode === 200 ? this.normalizeStocks(res.data) : [];
					this.persistCurrentListCache();
					this.afterStockListLoaded(preferCode);
				},
				fail: () => {
					this.stockList = [];
				}
			});
		},
		afterStockListLoaded(preferCode) {
			if (!this.stockList.length) return;
			const targetCode = preferCode || this.currentCode;
			const target = this.stockList.find(item => item.code === targetCode) || this.stockList[0];
			if (this.currentCode !== target.code) {
				this.currentCode = target.code;
				this.currentName = target.name;
				this.fetchKLineData();
			}
		},
		switchStock(stock) {
			this.currentCode = stock.code;
			this.currentName = stock.name;
			if (!this.isLandscape) this.isSidebarCollapsed = true;
			this.fetchKLineData();
		},
		pinTop(code) {
			const targetIndex = this.stockList.findIndex(item => item.code === code);
			if (targetIndex < 0) return;
			const stock = this.stockList.splice(targetIndex, 1)[0];
			stock.pinned = !stock.pinned;
			this.stockList.unshift(stock);
			this.persistCurrentListCache();
		},
		onDragStart(e, index) {
			this.draggingIndex = index;
			this.dragStartY = e.touches[0].clientY;
			// #ifdef APP-PLUS
			uni.vibrateShort();
			// #endif
		},
		onDragMove(e) {
			if (this.draggingIndex === -1) return;
			const currentY = e.touches[0].clientY;
			const diff = currentY - this.dragStartY;
			const rowHeight = 72;
			if (Math.abs(diff) <= rowHeight) return;

			const offset = diff > 0 ? 1 : -1;
			const targetIndex = this.draggingIndex + offset;
			if (targetIndex < 0 || targetIndex >= this.stockList.length) return;

			const item = this.stockList.splice(this.draggingIndex, 1)[0];
			this.stockList.splice(targetIndex, 0, item);
			this.draggingIndex = targetIndex;
			this.dragStartY = currentY;
		},
		onDragEnd() {
			this.draggingIndex = -1;
			this.persistCurrentListCache();
		},
		fetchKLineData() {
			if (!this.currentCode) return;
			const cacheKey = `kline_data_${this.currentCode}`;
			const cachedData = uni.getStorageSync(cacheKey);

			if (cachedData && Array.isArray(cachedData) && cachedData.length > 0) {
				this.loadingText = '';
				this.chartRawData = cachedData;
				this.updateLcdBanner(cachedData.length - 1);
				return;
			}

			this.loadingText = '数据通讯中...';
			this.chartRawData = null;

			uni.request({
				url: `${this.savedUrl}/api/kline/${this.currentCode}`,
				header: {
					"ngrok-skip-browser-warning": "69420"
				},
				success: (res) => {
					if (res.statusCode === 200 && Array.isArray(res.data) && res.data.length > 0) {
						this.loadingText = '';
						uni.setStorageSync(cacheKey, res.data);
						this.chartRawData = res.data;
						this.updateLcdBanner(res.data.length - 1);
					} else {
						this.loadingText = '该标的暂无数据';
					}
				},
				fail: () => {
					this.loadingText = '网络请求失败';
				}
			});
		},
		onCrosshairMove(index) {
			this.updateLcdBanner(index);
		},
		updateLcdBanner(index) {
			if (!this.chartRawData || !this.chartRawData[index]) return;
			const d = this.chartRawData[index];
			const preClose = index > 0 ? this.chartRawData[index - 1].close : d.open;
			const pctChg = ((d.close - preClose) / preClose * 100).toFixed(2);

			this.lcdData = {
				open: d.open.toFixed(2),
				close: d.close.toFixed(2),
				high: d.high.toFixed(2),
				low: d.low.toFixed(2),
				pctChg,
				sign: pctChg > 0 ? '+' : '',
				turn: (d.turn || 0).toFixed(2),
				volHand: this.formatHand(d.volume),
				amt: this.formatVol(d.amount || (d.volume * d.close * 100)),
				highColor: d.high > preClose ? 'val-red' : (d.high < preClose ? 'val-green' : 'val-normal'),
				lowColor: d.low > preClose ? 'val-red' : (d.low < preClose ? 'val-green' : 'val-normal')
			};
		}
	}
};
</script>

<script module="echart" lang="renderjs">
let myChart = null;
let pendingData = null;
let globalOwner = null;

export default {
	mounted() {
		if (typeof window.echarts === 'object') {
			this.initChart();
		} else {
			const script = document.createElement('script');
			script.src = 'https://cdn.jsdelivr.net/npm/echarts@5.5.0/dist/echarts.min.js';
			script.onload = this.initChart;
			document.head.appendChild(script);
		}
	},
	destroyed() {
		if (myChart) {
			myChart.dispose();
			myChart = null;
		}
		if (this.resizeObserver) {
			this.resizeObserver.disconnect();
		}
		pendingData = null;
		globalOwner = null;
	},
	methods: {
		initChart() {
			const dom = document.getElementById('echarts-dom');
			if (!dom) return;

			if (myChart) myChart.dispose();
			myChart = window.echarts.init(dom);

			if (window.ResizeObserver) {
				this.resizeObserver = new ResizeObserver(() => {
					if (myChart) myChart.resize();
				});
				this.resizeObserver.observe(dom);
			} else {
				window.addEventListener('resize', () => {
					if (myChart) myChart.resize();
				});
			}

			if (pendingData) {
				this.drawChart(pendingData);
				pendingData = null;
			}
		},
		updateData(newVal, oldVal, ownerInstance) {
			if (!newVal || newVal.length === 0) return;
			if (ownerInstance) globalOwner = ownerInstance;

			if (!myChart) {
				pendingData = newVal;
				return;
			}
			this.drawChart(newVal);
		},
		updateLandscape(newVal) {
			if (!myChart) return;
			myChart.setOption({ tooltip: { showContent: !newVal } });
			setTimeout(() => {
				if (myChart) myChart.resize();
			}, 100);
		},
		updateCollapsed() {
			if (!myChart) return;
			setTimeout(() => {
				if (myChart) myChart.resize();
			}, 320);
		},
		drawChart(data) {
			if (!myChart) return;

			const dates = data.map(item => item.date);
			const kData = data.map(item => [item.open, item.close, item.low, item.high]);
			const volData = data.map(item => ({
				value: item.volume,
				itemStyle: { color: item.close >= item.open ? '#e83828' : '#00a854' }
			}));

			const totalPoints = data.length;
			const startIdx = Math.max(0, totalPoints - 60);

			const option = {
				animation: false,
				tooltip: {
					show: true,
					showContent: true,
					trigger: 'axis',
					axisPointer: { type: 'cross' },
					backgroundColor: 'rgba(255, 255, 255, 0.95)',
					padding: 8,
					textStyle: { fontSize: 11 },
					formatter: function(params) {
						const i = params[0].dataIndex;
						const d = data[i];
						const preClose = i > 0 ? data[i - 1].close : d.open;
						const pctChg = ((d.close - preClose) / preClose * 100).toFixed(2);
						const pctColor = pctChg > 0 ? '#e83828' : (pctChg < 0 ? '#00a854' : '#333');
						const sign = pctChg > 0 ? '+' : '';
						const hands = d.volume / 100;
						const vStr = hands >= 10000 ? (hands / 10000).toFixed(2) + '万手' : hands.toFixed(0) + '手';

						return `<div style="font-family: sans-serif;">
							<div style="margin-bottom: 3px; border-bottom: 1px solid #eee; padding-bottom: 3px;"><b>${d.date}</b></div>
							开: ${d.open.toFixed(2)} &nbsp; 收: <span style="color:${d.close > d.open ? '#e83828' : '#00a854'}">${d.close.toFixed(2)}</span><br/>
							高: ${d.high.toFixed(2)} &nbsp; 低: ${d.low.toFixed(2)}<br/>
							涨幅: <b style="color:${pctColor}">${sign}${pctChg}%</b><br/>
							量: ${vStr} &nbsp; 换手: <b style="color:#d46b08">${(d.turn || 0).toFixed(2)}%</b>
						</div>`;
					}
				},
				axisPointer: { link: { xAxisIndex: 'all' } },
				grid: [
					{ left: '10%', right: '4%', top: '6%', height: '58%' },
					{ left: '10%', right: '4%', top: '70%', height: '14%' }
				],
				xAxis: [
					{ type: 'category', data: dates, boundaryGap: true, axisLine: { onZero: false }, splitLine: { show: false }, axisLabel: { fontSize: 10 } },
					{ type: 'category', data: dates, gridIndex: 1, axisLabel: { show: false }, axisTick: { show: false } }
				],
				yAxis: [
					{ scale: true, splitLine: { lineStyle: { type: 'dashed', color: '#f0f0f0' } }, axisLabel: { fontSize: 10 } },
					{ scale: true, gridIndex: 1, axisLabel: { show: false }, splitLine: { show: false } }
				],
				dataZoom: [
					{
						type: 'inside',
						xAxisIndex: [0, 1],
						startValue: startIdx,
						endValue: totalPoints - 1
					},
					{
						type: 'slider',
						xAxisIndex: [0, 1],
						bottom: '2%',
						height: 32,
						startValue: startIdx,
						endValue: totalPoints - 1,
						handleSize: '150%',
						handleIcon: 'path://M10.7,11.9v-1.3H9.3v1.3c-4.9,0.3-8.8,4.4-8.8,9.4c0,5,3.9,9.1,8.8,9.4v1.3h1.3v-1.3c4.9-0.3,8.8-4.4,8.8-9.4C19.5,16.3,15.6,12.2,10.7,11.9z M13.3,24.4H6.7V23h6.6V24.4z M13.3,19.6H6.7v-1.4h6.6V19.6z',
						dataBackground: {
							lineStyle: { color: '#1890ff' },
							areaStyle: { color: '#1890ff', opacity: 0.1 }
						},
						selectedDataBackground: {
							lineStyle: { color: '#1890ff' },
							areaStyle: { color: '#1890ff', opacity: 0.3 }
						}
					}
				],
				series: [
					{ name: '日K', type: 'candlestick', data: kData, itemStyle: { color: '#e83828', color0: '#00a854', borderColor: '#e83828', borderColor0: '#00a854' } },
					{ name: '成交量', type: 'bar', xAxisIndex: 1, yAxisIndex: 1, data: volData }
				]
			};

			myChart.setOption(option);

			myChart.off('updateAxisPointer');
			myChart.on('updateAxisPointer', function(event) {
				if (event.axesInfo && event.axesInfo.length > 0 && globalOwner) {
					globalOwner.callMethod('onCrosshairMove', event.axesInfo[0].value);
				}
			});
		}
	}
};
</script>

<style scoped>
* { box-sizing: border-box; }

.app-container {
	width: 100vw;
	height: 100vh;
	display: flex;
	overflow: hidden;
	background: #fff;
}

.sidebar {
	width: 168px;
	background: #f8f9fa;
	border-right: 1px solid #eee;
	display: flex;
	flex-direction: column;
	transition: width 0.3s ease, border-color 0.3s ease;
	flex-shrink: 0;
	overflow: hidden;
}

.sidebar.collapsed {
	width: 0;
	border-right-color: transparent;
}

.header-group {
	padding: calc(10px + env(safe-area-inset-top)) 10px 10px;
	background: #fff;
	border-bottom: 1px solid #eee;
}

.group-selector {
	height: 36px;
	padding: 0 12px;
	border: 1px solid #d9d9d9;
	border-radius: 10px;
	background: #f7f9fc;
	display: flex;
	align-items: center;
	justify-content: space-between;
	font-size: 13px;
	font-weight: 600;
	color: #2f3b52;
}

.arrow {
	font-size: 11px;
	color: #7f8ea3;
}

.stock-list {
	flex: 1;
	height: 0;
}

.stock-item {
	padding: 10px 10px 10px 12px;
	border-bottom: 1px solid #eef1f5;
	background: #fff;
}

.stock-item.active {
	background: #e6f4ff;
	border-left: 3px solid #1890ff;
}

.stock-item.is-dragging {
	background: #f0f7ff;
	box-shadow: 0 2px 10px rgba(24, 144, 255, 0.14);
}

.stock-item-main {
	display: flex;
	justify-content: space-between;
	align-items: flex-start;
	gap: 10px;
}

.stock-texts {
	flex: 1;
	min-width: 0;
}

.stock-actions {
	display: flex;
	flex-direction: column;
	align-items: flex-end;
	gap: 8px;
	flex-shrink: 0;
}

.s-name {
	font-weight: 700;
	color: #253142;
	font-size: 13px;
	line-height: 1.35;
	display: -webkit-box;
	-webkit-box-orient: vertical;
	-webkit-line-clamp: 2;
	overflow: hidden;
}

.code-line {
	display: flex;
	align-items: center;
	gap: 6px;
	margin-top: 4px;
	flex-wrap: wrap;
}

.s-code {
	font-size: 11px;
	color: #8c99ad;
}

.pin-btn {
	font-size: 11px;
	line-height: 1;
	padding: 5px 9px;
	border-radius: 999px;
	background: #eef3f9;
	color: #607089;
}

.pin-btn.active {
	background: #fff2e8;
	color: #d46b08;
}

.drag-handle {
	font-size: 14px;
	line-height: 1;
	color: #93a1b5;
	padding: 2px 4px;
	letter-spacing: -1px;
}

.new-badge {
	background: #ff4d4f;
	color: #fff;
	font-size: 10px;
	padding: 1px 5px;
	border-radius: 999px;
	font-weight: 700;
	line-height: 1.4;
}

.main {
	flex: 1;
	min-width: 0;
	display: flex;
	flex-direction: column;
	width: 100%;
	height: 100%;
	overflow: hidden;
	position: relative;
}

.toggle-sidebar-btn {
	position: absolute;
	left: 0;
	top: calc(12px + env(safe-area-inset-top));
	width: 24px;
	height: 34px;
	background: #fff;
	border: 1px solid #d8dee8;
	border-left: none;
	display: flex;
	justify-content: center;
	align-items: center;
	font-size: 12px;
	color: #66758a;
	border-radius: 0 8px 8px 0;
	z-index: 20;
	box-shadow: 2px 4px 10px rgba(15, 23, 42, 0.08);
}

.header {
	display: flex;
	justify-content: space-between;
	align-items: center;
	padding: calc(12px + env(safe-area-inset-top)) 15px 12px 38px;
	border-bottom: 1px solid #eee;
	flex-shrink: 0;
	background: #fff;
	gap: 10px;
}

.stock-title {
	font-size: 16px;
	font-weight: 700;
	color: #253142;
	flex: 1;
	min-width: 0;
	overflow: hidden;
	text-overflow: ellipsis;
	white-space: nowrap;
}

.btn-rotate {
	background: #f0f7ff;
	color: #1890ff;
	border: 1px solid #1890ff;
	padding: 4px 10px;
	border-radius: 15px;
	font-size: 12px;
	font-weight: 700;
	flex-shrink: 0;
}

.landscape-banner {
	display: flex;
	padding: 6px 15px 6px 38px;
	background: #fff;
	border-bottom: 1px solid #eee;
	font-size: 11px;
	flex-shrink: 0;
	justify-content: space-between;
	align-items: center;
	gap: 8px;
}

.lcd-block {
	display: flex;
	flex-direction: column;
	align-items: flex-start;
	justify-content: center;
}

.lcd-row {
	display: flex;
	gap: 4px;
	color: #666;
	margin: 1px 0;
}

.lcd-price {
	font-size: 16px;
	line-height: 1;
	font-weight: bold;
}

.lcd-change {
	font-size: 11px;
	font-weight: bold;
}

.val-red {
	color: #e83828;
	font-weight: bold;
}

.val-green {
	color: #00a854;
	font-weight: bold;
}

.val-normal {
	color: #333;
	font-weight: bold;
}

.chart-container {
	flex: 1;
	width: 100%;
	height: 100%;
	overflow: hidden;
}

.loading-tips {
	position: absolute;
	top: 40%;
	left: 50%;
	transform: translate(-50%, -50%);
	color: #1890ff;
	font-size: 13px;
	z-index: 99;
	padding: 8px 15px;
	background: #e6f7ff;
	border-radius: 6px;
}

.is-landscape .sidebar {
	width: 188px;
}

.is-landscape .sidebar.collapsed {
	width: 0;
}

.is-landscape .header {
	padding-top: 12px;
}

.is-landscape .stock-item {
	padding-top: 9px;
	padding-bottom: 9px;
}

.is-landscape .stock-actions {
	gap: 6px;
}

.is-landscape .pin-btn {
	padding: 4px 8px;
}
</style>
