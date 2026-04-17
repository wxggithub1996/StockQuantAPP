<template>
	<view class="container">
		<view class="header">
			<view class="icon-btn" @tap="toggleDrawer"><text class="iconfont">≡</text></view>
			<text class="title">私有策略</text>
			<view class="icon-btn" @tap="refreshAll"><text style="font-size: 14px;">⟳</text></view>
		</view>

		<view class="tabs">
			<view v-for="tab in tabList" :key="tab.status" class="tab" :class="{ active: currentStatus === tab.status }"
				@tap="switchTab(tab.status)">
				{{ tab.label }}({{ counts[tab.status] || 0 }})
			</view>
		</view>

		<view class="table-container">
			<view class="table-wrapper" v-if="displayStocks.length > 0">
				<view class="tr th">
					<view class="td sticky-left th-left">
						<view class="th-title">
							<text class="th-name">股票名称</text>
							<text class="th-code">代码</text>
						</view>
					</view>
					<view class="td scroll-right">
						<view class="cell-sparkline"></view>
						<view class="cell-data sortable" @tap="toggleSort">涨幅 <text
								class="sort-icon">{{ sortIcon }}</text></view>
						<view class="cell-data">换手率</view>
						<view class="cell-data">成交量</view>
						<view class="cell-data">成交额</view>
					</view>
				</view>

				<view class="tr" v-for="(item, index) in displayStocks" :key="item.code"
					:class="{'is-dragging': draggingIndex === index}" @tap="openKline(item)">

					<view class="td sticky-left">
						<view class="left-icons">
							<text class="icon-drag" @touchstart.stop.prevent="onDragStart($event, index)"
								@touchmove.stop.prevent="onDragMove" @touchend.stop.prevent="onDragEnd">⋮⋮</text>
							<text class="icon-pin" :style="{opacity: item.pinned ? 1 : 0.2}"
								@tap.stop="pinTop(item.code)">📌</text>
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
								<polyline :points="item.points" fill="none"
									:stroke="item.change > 0 ? '#e83828' : '#00a854'" stroke-width="1.5" />
							</svg>
						</view>
						<view class="cell-data compound-cell">
							<view class="change-block" :class="item.change > 0 ? 'bg-red' : 'bg-green'">
								{{ item.change > 0 ? '+' : '' }}{{ item.change !== undefined && !isNaN(item.change) ? parseFloat(item.change).toFixed(2) : '0.00' }}%
							</view>
							<text class="price-text">{{ item.price || '--' }}</text>
						</view>
						<view class="cell-data">{{ item.turnover }}{{ item.turnover !== '--' ? '%' : '' }}</view>
						<view class="cell-data">{{ item.volumeHand }}</view>
						<view class="cell-data">{{ item.amountFmt }}</view>
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
					<text>显示ST/科创/创业</text>
					<switch color="#1890ff" :checked="showSpecial" @change="onChangeSpecial" style="transform:scale(0.8)"/>
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

	</view>
</template>

<script>
	export default {
		data() {
			return {
				currentStatus: 1,
				tabList: [{
					label: '试盘',
					status: 1
				}, {
					label: '回踩',
					status: 2
				}, {
					label: '突破',
					status: 3
				}, {
					label: '废弃',
					status: 99
				}],
				counts: {
					1: 0,
					2: 0,
					3: 0,
					99: 0
				},
				sortState: 'default',
				isDrawerOpen: false,
				isModalOpen: false,
				showSpecial: false,
				tempUrl: '',
				savedUrl: '',
				isLoading: false,
				allStocks: [],
				isAuthUnlocked: false, // 权限解锁状态
				// 拖动排序相关状态
				draggingIndex: -1,
				isTaskRunning: false,
				runningTaskName: '',
				dragStartY: 0
			}
		},
		onShow() {
			// 每次回到首页，都去后台查一下状态
			this.checkTaskStatus();
		},
		computed: {
			sortIcon() {
				return this.sortState === 'desc' ? '↓' : (this.sortState === 'asc' ? '↑' : '');
			},
			displayStocks() {
				if (!Array.isArray(this.allStocks)) return [];
				let list = this.allStocks.filter(s => s.status === this.currentStatus);
				if (!this.showSpecial) list = list.filter(s => !s.isST && !(s.code && s.code.startsWith('688')));
				if (this.sortState === 'desc') list.sort((a, b) => parseFloat(b.change || 0) - parseFloat(a.change || 0));
				else if (this.sortState === 'asc') list.sort((a, b) => parseFloat(a.change || 0) - parseFloat(b.change ||
					0));
				return list;
			}
		},
		onLoad() {
			this.savedUrl = uni.getStorageSync('quant_server_url') || '';
			if (!this.savedUrl) {
				setTimeout(() => {
					this.isDrawerOpen = true;
				}, 500);
			} else {
				this.refreshAll();
			}
			this.fetchCounts(); // 启动时加载一次数字
			this.fetchDataFromPython();
		},
		methods: {
			// 🚀 新增：探测后台任务状态
			checkTaskStatus() {
				if (!this.savedUrl) return;
				uni.request({
					url: this.savedUrl + '/api/task_status',
					method: 'GET',
					success: (res) => {
						if (res.statusCode === 200) {
							this.isTaskRunning = res.data.is_running;
							this.runningTaskName = res.data.task_name;
						}
					}
				});
			},
			// 🚀 1. 新增：开关切换事件
			onChangeSpecial(e) {
				this.showSpecial = e.detail.value;
				this.fetchCounts(); // 🚨 开关变动，立刻重新获取数字
				this.fetchDataFromPython(true); // 🚨 同时强制刷新当前列表数据
			},

			// 核心方法：要求授权验证
			requireAuth(actionCallback) {
				if (this.isAuthUnlocked) {
					actionCallback();
					return;
				}
				uni.showModal({
					title: '系统风控拦截',
					content: '请输入架构师授权密码：',
					editable: true, // 开启输入框
					placeholderText: '请输入密码',
					success: (res) => {
						if (res.confirm) {
							if (res.content === '888') {
								this.isAuthUnlocked = true;
								uni.showToast({
									title: '解锁成功',
									icon: 'success'
								});
								actionCallback();
							} else {
								uni.showToast({
									title: '密码错误',
									icon: 'none'
								});
							}
						}
					}
				});
			},
			// 🚀 新增：静默预加载 K 线数据 (带 Ngrok 防封锁保护)
			async silentPreloadKLines(stockList) {
				if (!this.savedUrl || !Array.isArray(stockList)) return;

				// 过滤出本地还没有缓存的股票
				const needLoadStocks = stockList.filter(stock => !uni.getStorageSync(`kline_data_${stock.code}`));

				if (needLoadStocks.length === 0) return; // 全都有缓存了，收工！
				console.log(`[静默预加载] 发现 ${needLoadStocks.length} 只股票需要缓存...`);

				// 采用 for 循环 + await，一只一只地请求，绝不并发，保护 Ngrok
				for (let i = 0; i < needLoadStocks.length; i++) {
					const stock = needLoadStocks[i];
					const cacheKey = `kline_data_${stock.code}`;

					try {
						// 包装成 Promise 以支持 await 阻塞
						const res = await new Promise((resolve) => {
							uni.request({
								url: `${this.savedUrl}/api/kline/${stock.code}`,
								method: 'GET',
								success: (res) => resolve(res),
								fail: () => resolve(null)
							});
						});

						if (res && res.statusCode === 200 && Array.isArray(res.data) && res.data.length > 0) {
							// 拿到数据，悄悄塞进本地硬盘
							uni.setStorageSync(cacheKey, res.data);
						}

						// 🛡️ 核心保护：每拉完一只股票，强行休息 300 毫秒，防止 Ngrok 崩溃
						await new Promise(r => setTimeout(r, 300));

					} catch (e) {
						console.error(`[静默预加载] ${stock.name} 失败:`, e);
					}
				}
				console.log('[静默预加载] 队列执行完毕！');
			},

			refreshAll(force = false) {
				if (force) {
					// 1. 清空所有池子的列表缓存
					[1, 2, 3, 99].forEach(status => uni.removeStorageSync(`pool_data_${status}`));

					// 2. 🚀 清空所有 K 线的详细缓存
					try {
						const res = uni.getStorageInfoSync();
						res.keys.forEach(key => {
							if (key.startsWith('kline_data_')) {
								uni.removeStorageSync(key);
							}
						});
					} catch (e) {
						console.error('清理 K 线缓存失败', e);
					}

					uni.showToast({
						title: '开始全面更新数据',
						icon: 'loading'
					});
				}

				this.fetchCounts();
				this.fetchDataFromPython(force);
			},
			fetchCounts() {
				if (!this.savedUrl) return;
				// 🚨 显式传递字符串格式的布尔值
				uni.request({
					url: `${this.savedUrl}/api/counts?show_special=${this.showSpecial}`,
					method: 'GET',
					success: (res) => {
						if (res.statusCode === 200) {
							this.counts = Object.assign({}, this.counts, res.data);
						}
					}
				});
			},
			fetchDataFromPython(force = false) {
				if (!this.savedUrl) return;

				const cacheKey = `pool_data_${this.currentStatus}`;
				const cachedRawData = uni.getStorageSync(cacheKey);

				// 🚨 核心逻辑：如果不强制刷新，且本地有缓存，直接秒出！
				if (!force && cachedRawData) {
					this.processAndRender(cachedRawData);
					return;
				}

				// 没有缓存或强制刷新时，走网络请求
				this.isLoading = true;
				uni.request({
					url: `${this.savedUrl}/api/pool/app/${this.currentStatus}?show_special=${this.showSpecial}`,
					method: 'GET',
					success: (res) => {
						if (res.statusCode === 200 && Array.isArray(res.data)) {
							// 拉到数据后，立刻存入手机本地
							uni.setStorageSync(cacheKey, res.data);
							this.processAndRender(res.data);
						} else {
							this.allStocks = [];
						}
					},
					complete: () => {
						this.isLoading = false;
					}
				});
			},
			// 3. 新增：将原本繁杂的数据格式化提炼为一个独立函数
			processAndRender(rawData) {
				this.allStocks = rawData.map(item => {
					const changeVal = parseFloat(item.pct_change || item.change || 0);

					let amt = item.amount || '--';
					if (amt !== '--' && String(amt).indexOf('亿') === -1 && String(amt).indexOf('万') === -1) {
						let v = parseFloat(amt);
						if (!isNaN(v)) {
							if (v >= 100000000) amt = (v / 100000000).toFixed(2) + '亿';
							else if (v >= 10000) amt = (v / 10000).toFixed(0) + '万';
							else amt = v.toFixed(0);
						}
					}

					let volHand = '--';
					if (item.raw_volume && item.raw_volume !== '') {
						let shares = parseFloat(item.raw_volume);
						if (!isNaN(shares)) {
							let hands = shares / 100;
							if (hands >= 10000) volHand = (hands / 10000).toFixed(2) + '万手';
							else volHand = hands.toFixed(0) + '手';
						}
					}

					let turn = item.turnover || '--';
					if (turn !== '--' && !isNaN(parseFloat(turn))) turn = parseFloat(turn).toFixed(2);

					return {
						name: item.name,
						code: item.code,
						price: item.price || '--',
						change: changeVal,
						turnover: turn,
						volumeHand: volHand,
						amountFmt: amt,
						status: this.currentStatus,
						isST: item.name ? item.name.includes('ST') : false,
						pinned: false,
						points: item.sparkline || (changeVal > 0 ? "0,15 15,10 25,12 35,0" : "0,0 15,5 25,2 35,15")
					}
				});
				// 🚀 核心新增：屏幕渲染完成后，立马安排后台小弟去拉取这批股票的 K 线
				// 不用 await，让它自己挂在后台慢慢跑，不阻塞用户操作
				this.silentPreloadKLines(this.allStocks);
			},
			switchTab(status) {
				this.currentStatus = status;
				this.sortState = 'default';
				this.fetchDataFromPython();
			},
			toggleSort() {
				const states = ['default', 'desc', 'asc'];
				this.sortState = states[(states.indexOf(this.sortState) + 1) % 3];
			},
			toggleDrawer() {
				this.isDrawerOpen = !this.isDrawerOpen;
			},
			openConfigModal() {
				this.tempUrl = uni.getStorageSync('quant_server_url') || '';
				this.isDrawerOpen = false;
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
				// ⚠️ 增加传递 status 参数，告诉 K 线页加载哪个侧边栏列表
				const safeName = encodeURIComponent(stock.name);

				uni.navigateTo({
					url: `/pages/kline/kline?code=${stock.code}&name=${safeName}&status=${stock.status}`,
					fail: (err) => {
						console.error("【跳转失败】", err);
						uni.showToast({
							title: '跳转失败，请重试',
							icon: 'none'
						});
					}
				});
			},
			// 优化：使用 code 作为唯一标识进行置顶，避免因为排序导致 index 错乱
			pinTop(code) {
				const targetIndex = this.allStocks.findIndex(s => s.code === code);
				if (targetIndex > -1) {
					const stock = this.allStocks.splice(targetIndex, 1)[0];
					stock.pinned = !stock.pinned;
					this.allStocks.unshift(stock);
				}
			},

			// --- 核心：触摸拖动排序逻辑 ---
			onDragStart(e, index) {
				// 如果处于涨幅排序状态，禁止手动拖动，防止逻辑冲突
				if (this.sortState !== 'default') {
					uni.showToast({
						title: '请先点击涨幅恢复默认排序',
						icon: 'none'
					});
					return;
				}
				this.draggingIndex = index;
				this.dragStartY = e.touches[0].clientY;
				// 触发手机轻微震动反馈
				// #ifdef APP-PLUS
				uni.vibrateShort();
				// #endif
			},
			onDragMove(e) {
				if (this.draggingIndex === -1) return;
				const currentY = e.touches[0].clientY;
				const diff = currentY - this.dragStartY;
				const rowHeight = 45; // 近似的一行高度，用于触发交换阈值

				// 当滑动距离超过一行的高度时，触发位置交换
				if (Math.abs(diff) > rowHeight) {
					const offset = diff > 0 ? 1 : -1;
					let targetIndex = this.draggingIndex + offset;

					if (targetIndex >= 0 && targetIndex < this.displayStocks.length) {
						// 找到当前项和目标项在总数据源中的真实索引
						const currentItem = this.displayStocks[this.draggingIndex];
						const targetItem = this.displayStocks[targetIndex];
						const allCurrentIdx = this.allStocks.indexOf(currentItem);
						const allTargetIdx = this.allStocks.indexOf(targetItem);

						if (allCurrentIdx > -1 && allTargetIdx > -1) {
							// 响应式交换数据
							const item = this.allStocks.splice(allCurrentIdx, 1)[0];
							this.allStocks.splice(allTargetIdx, 0, item);

							this.draggingIndex = targetIndex;
							this.dragStartY = currentY; // 重置起始点
						}
					}
				}
			},
			onDragEnd() {
				this.draggingIndex = -1;
			},
			// ---------------------------------
			// 🚨 拦截层：只负责弹窗和确认，绝对不在这里写 uni.request
			handleAction(actionType) {
				this.isDrawerOpen = false;
				if (actionType === 'log') {
					uni.navigateTo({
						url: '/pages/log/log'
					});
					return;
				}

				// 🚨 客户端强拦截：跨端互斥
				if (this.isTaskRunning) {
					uni.showToast({
						title: `系统正在执行【${this.runningTaskName}】\n请稍后再试`,
						icon: 'none',
						duration: 2500
					});
					return;
				}

				this.requireAuth(() => {
					const title = actionType === 'sync' ? '确认执行增量同步？' : '确认执行深度重置？';
					const content = actionType === 'sync' ? '此操作将拉取全市场数据并重新运行策略。' : '警告：此操作将清空现有管道并进行历史雷达回溯！';

					uni.showModal({
						title: title,
						content: content,
						success: (res) => {
							if (res.confirm) {
								this.executeOperation(actionType);
							}
						}
					});
				});
			},

			// 🚨 执行层：这个函数只允许被 res.confirm 内部调用！
			executeOperation(actionType) {
				let apiEndpoint = actionType === 'sync' ? '/api/run_strategy' : '/api/reset_bootstrap';

				// 点击后立刻锁定前端状态
				this.isTaskRunning = true;
				this.runningTaskName = actionType === 'sync' ? '增量同步' : '深度重置';
				uni.showLoading({
					title: '系统执行中...',
					mask: true
				});

				uni.request({
					url: this.savedUrl + apiEndpoint,
					method: 'POST',
					data: actionType === 'sync' ? {
						source: '按钮处理'
					} : {},
					success: (res) => {
						// 处理由于极端并发被后端拦截的情况
						if (res.data.status === 'running') {
							uni.showToast({
								title: res.data.msg,
								icon: 'none',
								duration: 3000
							});
							return;
						}
						uni.showToast({
							title: '执行成功',
							icon: 'success'
						});
						this.refreshAll(true);
					},
					complete: () => {
						uni.hideLoading();
						this.checkTaskStatus(); // 结束时重置状态
					}
				});
			},
			// 列表排序/置顶也建议加上密码校验
			pinTop(index) {
				this.requireAuth(() => {
					const stock = this.allStocks.splice(index, 1)[0];
					stock.pinned = !stock.pinned;
					this.allStocks.unshift(stock);
					// 可以在此处同步到后端或本地持久化
				});
			}
		}
	}
</script>

<style scoped>
	/* 修复了原生导航栏重叠和刘海屏问题 */
	.header {
		display: flex;
		justify-content: space-between;
		align-items: center;
		height: 44px;
		padding-top: var(--status-bar-height);
		padding-left: 15px;
		padding-right: 15px;
		background: #f8f9fa;
		border-bottom: 1px solid #eee;
		flex-shrink: 0;
		box-sizing: content-box;
	}

	.container {
		width: 100vw;
		height: 100vh;
		display: flex;
		flex-direction: column;
		background: #fff;
		overflow: hidden;
	}

	.tabs {
		display: flex;
		border-bottom: 1px solid #eee;
		background: #fff;
		flex-shrink: 0;
	}

	.tab {
		flex: 1;
		text-align: center;
		padding: 12px 0;
		font-size: 13px;
		color: #666;
		transition: 0.2s;
	}

	.tab.active {
		color: #1890ff;
		font-weight: bold;
		position: relative;
	}

	.tab.active::after {
		content: '';
		position: absolute;
		bottom: 0;
		left: 30%;
		width: 40%;
		height: 3px;
		background: #1890ff;
		border-radius: 2px;
	}

	.table-container {
		flex: 1;
		overflow: hidden;
		position: relative;
	}

	.table-wrapper {
		height: 100%;
		overflow: auto;
		display: flex;
		flex-direction: column;
	}

	.tr {
		display: flex;
		width: max-content;
		min-width: 100%;
		border-bottom: 1px solid #f9f9f9;
		background: #fff;
		transition: background 0.2s;
	}

	/* 拖拽状态的高亮提示 */
	.is-dragging {
		background: #f0f7ff;
		box-shadow: 0 2px 10px rgba(0, 0, 0, 0.1);
		z-index: 9;
		position: relative;
	}

	.th {
		background: #fafafa;
		position: sticky;
		top: 0;
		z-index: 20;
		height: 38px;
	}

	/* 优化2：减小固定列宽度，收缩留白 */
	.sticky-left {
		position: sticky;
		left: 0;
		background: #fff;
		z-index: 10;
		width: 106px;
		/* 宽度从 130px 缩减到 106px，消除大量空白 */
		box-sizing: border-box;
		flex-shrink: 0;
		display: flex;
		align-items: center;
		border-right: 1px solid #f0f0f0;
		padding-left: 8px;
	}

	.th-left {
		background: #fafafa;
		z-index: 11;
	}

	/* 优化3：表头样式重构 */
	.th-title {
		display: flex;
		align-items: baseline;
		gap: 4px;
		padding-left: 4px;
	}

	.th-name {
		font-size: 13px;
		font-weight: bold;
		color: #333;
	}

	.th-code {
		font-size: 10px;
		color: #999;
		font-weight: normal;
	}

	.scroll-right {
		display: flex;
	}

	.cell-data {
		width: 80px;
		text-align: center;
		display: flex;
		justify-content: center;
		align-items: center;
		flex-shrink: 0;
		font-size: 13px;
	}

	.cell-sparkline {
		width: 45px;
		flex-shrink: 0;
		display: flex;
		align-items: center;
		justify-content: center;
	}

	.compound-cell {
		flex-direction: column;
		padding: 6px 0;
		width: 80px;
	}

	.change-block {
		width: 62px;
		border-radius: 3px;
		color: #fff;
		font-size: 12px;
		font-weight: bold;
		padding: 2px 0;
		text-align: center;
	}

	.bg-red {
		background: #e83828;
	}

	.bg-green {
		background: #00a854;
	}

	.price-text {
		font-size: 11px;
		color: #999;
		margin-top: 2px;
	}

	.stock-info {
		display: flex;
		flex-direction: column;
	}

	.stock-name {
		font-size: 15px;
		font-weight: bold;
		color: #333;
	}

	.code-row {
		display: flex;
		align-items: center;
		margin-top: 1px;
	}

	.stock-code {
		font-size: 11px;
		color: #999;
		margin-right: 4px;
	}

	.badge {
		font-size: 9px;
		border: 1px solid;
		padding: 0 2px;
		border-radius: 2px;
		margin-left: 2px;
		line-height: 1;
	}

	.badge.st {
		border-color: #faad14;
		color: #faad14;
	}

	.badge.ke {
		border-color: #1890ff;
		color: #1890ff;
	}

	.left-icons {
		display: flex;
		flex-direction: column;
		margin-right: 8px;
		font-size: 12px;
		color: #bbb;
		align-items: center;
	}

	.icon-drag {
		padding: 4px 2px;
		font-size: 10px;
		cursor: grab;
	}

	.empty-state {
		padding: 50px;
		text-align: center;
		color: #999;
		font-size: 14px;
	}

	/* 侧边栏和弹窗样式... (与原代码一致) */
	.drawer-overlay {
		position: fixed;
		top: 0;
		left: 0;
		width: 100vw;
		height: 100vh;
		background: rgba(0, 0, 0, 0.5);
		z-index: 100;
	}

	.drawer-panel {
		position: fixed;
		top: 0;
		left: -260px;
		width: 240px;
		height: 100vh;
		background: #fff;
		z-index: 101;
		transition: 0.3s;
		padding: 40px 15px;
	}

	.drawer-open {
		left: 0;
	}

	.menu-btn {
		width: 100%;
		margin-bottom: 12px;
		font-size: 14px;
		background: #f5f5f5;
		border: none;
	}

	.menu-item-row {
		display: flex;
		justify-content: space-between;
		align-items: center;
		padding: 10px;
		font-size: 14px;
		color: #333;
		background: #fafafa;
		border-radius: 6px;
		margin-bottom: 12px;
	}

	.modal-overlay {
		position: fixed;
		top: 0;
		left: 0;
		width: 100vw;
		height: 100vh;
		background: rgba(0, 0, 0, 0.6);
		z-index: 9999;
		display: flex;
		justify-content: center;
		align-items: center;
	}

	.modal-box {
		width: 80%;
		background: #fff;
		border-radius: 12px;
		padding: 20px;
		box-shadow: 0 4px 20px rgba(0, 0, 0, 0.2);
	}

	.modal-input {
		border: 1px solid #ddd;
		padding: 10px;
		border-radius: 6px;
		margin: 15px 0;
		width: 90%;
	}

	.modal-actions {
		display: flex;
		gap: 10px;
	}

	.modal-btn {
		flex: 1;
		font-size: 14px;
	}
</style>