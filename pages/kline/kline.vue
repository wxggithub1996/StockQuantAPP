<template>
	<view class="app-container" :class="{'is-landscape': isLandscape}">
		<view class="sidebar" :class="{'collapsed': isSidebarCollapsed}">
			<view class="sidebar-title">同组标的</view>
			<scroll-view scroll-y class="stock-list">
				<view class="stock-item" 
					v-for="item in stockList" :key="item.code"
					:class="{'active': item.code === currentCode}"
					@tap="switchStock(item)">
					<text class="s-name">{{ item.name }}</text>
					<text class="s-code">{{ item.code }}</text>
				</view>
			</scroll-view>
		</view>

		<view class="main">
			<view class="toggle-sidebar-btn" @tap="toggleSidebar">
				{{ isSidebarCollapsed ? '▶' : '◀' }}
			</view>

			<view class="loading-tips" v-if="loadingText">{{ loadingText }}</view>

			<view class="header">
				<text class="stock-title">{{ currentName }} ({{ currentCode }})</text>
				<view class="btn-rotate" @tap="toggleLandscape">
					⇋ {{ isLandscape ? '竖屏恢复' : '横屏模式' }}
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

			<view id="echarts-dom" class="chart-container" 
				:prop="chartRawData" :change:prop="echart.updateData"
				:landscape="isLandscape" :change:landscape="echart.updateLandscape">
			</view>
		</view>
	</view>
</template>

<script>
// ==========================================
// 🧠 逻辑层 
// ==========================================
export default {
	data() {
		return {
			savedUrl: '',
			currentCode: '',
			currentName: '行情详情',
			currentStatus: 1,
			isLandscape: false,
			isSidebarCollapsed: true,
			loadingText: '',
			stockList: [],
			chartRawData: null,
			
			lcdData: {
				open: '--', close: '--', high: '--', low: '--',
				pctChg: '--', sign: '', turn: '--', volHand: '--', amt: '--',
				highColor: 'val-normal', lowColor: 'val-normal'
			}
		}
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
		if (!this.savedUrl) return uni.showToast({ title: '服务器地址丢失', icon: 'none' });

		this.currentCode = options.code || '';
		try { this.currentName = decodeURIComponent(options.name || '行情详情'); } catch (e) {}
		this.currentStatus = options.status || 1;

		this.loadStockList();
		this.fetchKLineData();
	},
	// 🚨 修复 1：页面调整大小 (监听手机重力感应物理翻转)
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
		// 🚨 修复 2：退出 K 线页时，彻底解锁方向，防止影响 App 其他页面
		// #ifdef APP-PLUS
		plus.navigator.setFullscreen(false);
		plus.screen.lockOrientation('portrait-primary');
		setTimeout(() => plus.screen.unlockOrientation(), 200);
		// #endif
	},
	methods: {
		formatVol(v) {
			if (!v) return '--';
			if (v >= 100000000) return (v / 100000000).toFixed(2) + '亿';
			if (v >= 10000) return (v / 10000).toFixed(0) + '万';
			return v.toFixed(0);
		},
		formatHand(v) {
			if (!v) return '--';
			let hands = v / 100;
			if (hands >= 10000) return (hands / 10000).toFixed(2) + '万手';
			return hands.toFixed(0) + '手';
		},
		toggleSidebar() {
			this.isSidebarCollapsed = !this.isSidebarCollapsed;
		},
		// 🚨 修复 3：调用 App 原生底层锁定横竖屏，彻底告别 CSS 裁剪问题！
		toggleLandscape() {
			this.isLandscape = !this.isLandscape;
			// #ifdef APP-PLUS
			if (this.isLandscape) {
				plus.navigator.setFullscreen(true);
				plus.screen.lockOrientation('landscape-primary'); // 硬件级横屏
				this.isSidebarCollapsed = false;
			} else {
				plus.navigator.setFullscreen(false);
				plus.screen.lockOrientation('portrait-primary'); // 硬件级竖屏
				this.isSidebarCollapsed = true;
			}
			// #endif
		},
		switchStock(stock) {
			this.currentCode = stock.code;
			this.currentName = stock.name;
			if (!this.isLandscape) this.isSidebarCollapsed = true;
			this.fetchKLineData();
		},
		loadStockList() {
			const cacheKey = `pool_data_${this.currentStatus}`;
			const cachedList = uni.getStorageSync(cacheKey);
			if (cachedList && Array.isArray(cachedList)) {
				this.stockList = cachedList;
			} else {
				uni.request({
					url: `${this.savedUrl}/api/pool/app/${this.currentStatus}`,
					success: (res) => { if (res.statusCode === 200) this.stockList = res.data; }
				});
			}
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

			this.loadingText = '📡 数据通讯中...';
			this.chartRawData = null; 

			uni.request({
				url: `${this.savedUrl}/api/kline/${this.currentCode}`,
				success: (res) => {
					if (res.statusCode === 200 && Array.isArray(res.data) && res.data.length > 0) {
						this.loadingText = '';
						uni.setStorageSync(cacheKey, res.data); 
						this.chartRawData = res.data; 
						this.updateLcdBanner(res.data.length - 1);
					} else {
						this.loadingText = '❌ 该标的暂无数据';
					}
				},
				fail: () => { this.loadingText = '❌ 网络请求失败'; }
			});
		},
		onCrosshairMove(index) {
			this.updateLcdBanner(index);
		},
		updateLcdBanner(index) {
			if (!this.chartRawData || !this.chartRawData[index]) return;
			const d = this.chartRawData[index];
			const preClose = index > 0 ? this.chartRawData[index-1].close : d.open;
			const pctChg = ((d.close - preClose) / preClose * 100).toFixed(2);
			
			this.lcdData = {
				open: d.open.toFixed(2), close: d.close.toFixed(2),
				high: d.high.toFixed(2), low: d.low.toFixed(2),
				pctChg: pctChg, sign: pctChg > 0 ? '+' : '',
				turn: (d.turn || 0).toFixed(2),
				volHand: this.formatHand(d.volume),
				amt: this.formatVol(d.amount || (d.volume * d.close * 100)),
				highColor: d.high > preClose ? 'val-red' : (d.high < preClose ? 'val-green' : 'val-normal'),
				lowColor: d.low > preClose ? 'val-red' : (d.low < preClose ? 'val-green' : 'val-normal')
			};
		}
	}
}
</script>

<script module="echart" lang="renderjs">
// ==========================================
// 🎨 RenderJS 渲染层 
// ==========================================
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
		// 清理 observer 防止内存泄漏
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
			
			// 🚨 修复 4：引入浏览器 ResizeObserver，全天候侦听容器物理尺寸变化
			// 不管横屏竖屏、隐藏状态栏，只要 DOM 尺寸变了，立马通知 ECharts 自适应，绝不裁剪！
			if (window.ResizeObserver) {
				this.resizeObserver = new ResizeObserver(() => {
					if (myChart) myChart.resize();
				});
				this.resizeObserver.observe(dom);
			} else {
				window.addEventListener('resize', () => { if(myChart) myChart.resize(); });
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

		drawChart(data) {
					if (!myChart) return;
					
					const dates = data.map(item => item.date);
					const kData = data.map(item => [item.open, item.close, item.low, item.high]);
					const volData = data.map(item => ({
						value: item.volume,
						itemStyle: { color: item.close >= item.open ? '#e83828' : '#00a854' }
					}));
		
					// 🚨 优化1：动态计算最近 60 个交易日的精确索引
					const totalPoints = data.length;
					const startIdx = Math.max(0, totalPoints - 60);
		
					const option = {
						animation: false,
						tooltip: { 
							show: true, showContent: true, 
							trigger: 'axis', axisPointer: { type: 'cross' }, 
							backgroundColor: 'rgba(255, 255, 255, 0.95)', padding: 8, textStyle: { fontSize: 11 },
							formatter: function(params) {
								let i = params[0].dataIndex; let d = data[i]; 
								let preClose = i > 0 ? data[i-1].close : d.open;
								let pctChg = ((d.close - preClose) / preClose * 100).toFixed(2); 
								let pctColor = pctChg > 0 ? '#e83828' : (pctChg < 0 ? '#00a854' : '#333'); 
								let sign = pctChg > 0 ? '+' : '';
								
								let hands = d.volume / 100;
								let vStr = hands >= 10000 ? (hands/10000).toFixed(2)+'万手' : hands.toFixed(0)+'手';
								
								return `<div style="font-family: sans-serif;">
									<div style="margin-bottom: 3px; border-bottom: 1px solid #eee; padding-bottom: 3px;"><b>${d.date}</b></div>
									开: ${d.open.toFixed(2)} &nbsp; 收: <span style="color:${d.close>d.open?'#e83828':'#00a854'}">${d.close.toFixed(2)}</span><br/>
									高: ${d.high.toFixed(2)} &nbsp; 低: ${d.low.toFixed(2)}<br/>
									涨幅: <b style="color:${pctColor}">${sign}${pctChg}%</b><br/>
									量: ${vStr} &nbsp; 换手: <b style="color:#d46b08">${(d.turn||0).toFixed(2)}%</b>
								</div>`;
							}
						},
						axisPointer: { link: { xAxisIndex: 'all' } },
						// 🚨 优化2：稍微压缩 K 线和成交量的高度，给底部增高的滑块腾出空间
						grid: [
							{ left: '10%', right: '4%', top: '6%', height: '58%' },  // 主图高度 58%
							{ left: '10%', right: '4%', top: '70%', height: '14%' }  // 副图高度 14%
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
								startValue: startIdx,   // 内部滑动也绑定到精确索引
								endValue: totalPoints - 1 
							},
							{ 
								type: 'slider', 
								xAxisIndex: [0, 1], 
								bottom: '2%',           // 贴近底部
								height: 32,             // 🚨 优化3：高度从默认增加到 32px，手指更容易点中
								startValue: startIdx,   // 默认只加载 startIdx(近60天) 到结尾
								endValue: totalPoints - 1,
								handleSize: '150%',     // 🚨 优化4：将左右两侧用于拖拽的把手放大 1.5倍，解决边缘拖动不灵敏的问题
								// 使用带把手的专业图标
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
					myChart.on('updateAxisPointer', function (event) {
						if (event.axesInfo && event.axesInfo.length > 0 && globalOwner) {
							globalOwner.callMethod('onCrosshairMove', event.axesInfo[0].value);
						}
					});
				},
		
		updateLandscape(newVal) {
			if (!myChart) return;
			// 只隐藏弹窗黑框，光标依然保留
			myChart.setOption({ tooltip: { showContent: !newVal } });
		}
	}
}
</script>

<style scoped>
/* ==========================================
   💅 样式布局 (已移除容易出 Bug 的 CSS 旋转动画)
   ========================================== */
* { box-sizing: border-box; }

.app-container { 
	width: 100vw; height: 100vh; 
	display: flex; overflow: hidden; background: #fff; 
}

/* 侧边栏 */
.sidebar { 
	width: 120px; background: #f8f9fa; border-right: 1px solid #eee; display: flex; flex-direction: column; 
	transition: margin-left 0.3s ease; flex-shrink: 0; z-index: 10; height: 100%;
	padding-left: env(safe-area-inset-left);
	padding-top: max(10px, env(safe-area-inset-top));
}
.sidebar.collapsed { margin-left: calc(-120px - env(safe-area-inset-left)); }
.sidebar-title { padding: 10px; font-size: 12px; font-weight: bold; text-align: center; background: #fff; border-bottom: 1px solid #eee; color: #1890ff; flex-shrink: 0;}
.stock-list { flex: 1; height: 0; }
.stock-item { padding: 12px 10px; font-size: 13px; border-bottom: 1px solid #f0f0f0; display: flex; flex-direction: column; }
.stock-item.active { background: #e6f7ff; border-left: 3px solid #1890ff; }
.s-name { font-weight: bold; color: #333; }
.s-code { font-size: 10px; color: #999; margin-top: 2px; }

/* 主体区域 (加入了溢出保护) */
.main { flex: 1; display: flex; flex-direction: column; width: 100%; height: 100%; overflow: hidden; position: relative;}

.toggle-sidebar-btn {
	position: absolute; left: 0; top: 50%; transform: translateY(-50%);
	width: 15px; height: 40px; background: #fff; border: 1px solid #ddd; border-left: none;
	display: flex; justify-content: center; align-items: center; font-size: 12px; color: #999;
	border-radius: 0 4px 4px 0; z-index: 20; box-shadow: 2px 0 5px rgba(0,0,0,0.05);
}

/* 顶部控制栏 */
.header { 
	display: flex; justify-content: space-between; align-items: center; padding: 10px 15px; 
	border-bottom: 1px solid #eee; flex-shrink: 0; background: #fff; 
	padding-top: max(10px, env(safe-area-inset-top));
}
.stock-title { font-size: 16px; font-weight: bold; color: #333; }
.btn-rotate { background: #f0f7ff; color: #1890ff; border: 1px solid #1890ff; padding: 4px 10px; border-radius: 15px; font-size: 12px; font-weight: bold;}

/* 横屏数据横幅 (LCD) */
.landscape-banner { 
	display: flex; padding: 5px 15px; background: #fff; border-bottom: 1px solid #eee; 
	font-size: 11px; flex-shrink: 0; justify-content: space-between; align-items: center;
}
.lcd-block { display: flex; flex-direction: column; align-items: flex-start; justify-content: center; }
.lcd-row { display: flex; gap: 4px; color: #666; margin: 1px 0;}
.lcd-price { font-size: 16px; line-height: 1; font-weight: bold;}
.lcd-change { font-size: 11px; font-weight: bold;}

.val-red { color: #e83828; font-weight: bold; }
.val-green { color: #00a854; font-weight: bold; }
.val-normal { color: #333; font-weight: bold; }

/* 横屏微调 (物理横屏下，安全距离等自动发生改变) */
.is-landscape .header { padding: 4px 15px; padding-top: 4px; } 
.is-landscape .stock-title { font-size: 14px; }
.is-landscape .sidebar { width: 140px; padding-top: 0; }
.is-landscape .sidebar.collapsed { margin-left: calc(-140px - env(safe-area-inset-left)); }

/* 图表容器 (强制宽高100%，防止挤压变形) */
.chart-container { flex: 1; width: 100%; height: 100%; overflow: hidden; }
.loading-tips { position: absolute; top: 40%; left: 50%; transform: translate(-50%, -50%); color: #1890ff; font-size: 13px; z-index: 99; padding: 8px 15px; background: #e6f7ff; border-radius: 6px; }
</style>