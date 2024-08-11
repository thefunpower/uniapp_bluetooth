<template>
	<view style="margin: 20px;">
		<button @click="openBluetooth">打开蓝牙</button>
		<button @click="searchBluetooth">开始搜索蓝牙设备</button>

		<view v-if="connectedDeviceId" style="card">
			<view style="margin-bottom: 20px;border-bottom: 1px solid #eee;">当前设备 ：{{connectedDeviceName}}</view>
			<view>
				<view>原始值：</view>
				<input type="text" style="border: 1px solid #eee;padding: 10px;" v-model="dataToSend" placeholder="输入要发送的数据" />
				<button @click="sendData">发送数据</button>
			</view>
		</view>

		<view v-if="receivedData" style="border: 1px solid #eee;padding: 10px;">
			<text>接收到的数据: </text>
			<view>
				{{ receivedData }}
			</view>
		</view>

		<view v-for="(device, index) in devices" class="device space-between" :key="index">
			<text>{{ device.name || '未知设备' }} </text>
			<view @click="connectToDevice(device)">连接</view>
		</view>




	</view>
</template>

<script>
	export default {
		data() {
			return {
				dataToSend: "",
				checkLette: 'AC-', // 蓝牙ID前缀
				devices: [], // 存储搜索到的设备
				receivedData: null, // 保存接收到的数据
				connectedDeviceId: null, // 当前连接的设备ID
				serviceId: '', // 服务UUID
				characteristicId: '', // 特征UUID
				connectedDeviceName: '', //当前连接的设备
				characteristicIdWrite: '',
			};
		},
		onLoad() {
			this.openBluetooth();
			this.connectedDeviceName = uni.getStorageSync('uni_bluetooth_name');
			this.connectedDeviceId = uni.getStorageSync('uni_bluetooth_deviceId');
			this.serviceId = uni.getStorageSync('uni_bluetooth_serviceId');
			this.characteristicId = uni.getStorageSync('uni_bluetooth_characteristicId');
			this.characteristicIdWrite = uni.getStorageSync('uni_bluetooth_characteristicIdWrite'); 
		},
		methods: {
			sendData() {
				if (!this.dataToSend) {
					console.warn('没有要发送的数据');
					return;
				} 
				// 将字符串转换为 HEX
				const hexData = this.strToHex(this.dataToSend); // 转换为 HEX 
				console.log('hex',hexData);
				// 将 HEX 转换为 ArrayBuffer
				const buffer = this.hexToArrayBuffer(hexData); // 转换为 ArrayBuffer 
				console.log('ArrayBuffer',buffer);
				uni.writeBLECharacteristicValue({
					deviceId: this.connectedDeviceId,
					serviceId: this.serviceId,
					characteristicId: this.characteristicIdWrite,
					value: buffer,
					success: (res) => {
						uni.showToast({
							title: "发送数据成功",
							icon: "none"
						})
						console.log('发送数据成功', res);
					},
					fail: (err) => {
						uni.showToast({
							title: "发送数据失败",
							icon: "none"
						})
						console.error('发送数据失败', err);
					}
				});
			},
			openBluetooth() {
				uni.openBluetoothAdapter({
					success: (res) => {
						console.log('蓝牙适配器已打开', res);
						this.searchBluetooth();
					},
					fail: (err) => {
						console.error('打开蓝牙适配器失败', err);
					}
				});
			},

			// 启动搜索设备
			searchBluetooth() {
				// 清空之前的设备列表
				this.devices = [];

				uni.startBluetoothDevicesDiscovery({
					success: (res) => {
						//console.log('开始搜索设备', res);
					},
					fail: (err) => {
						//console.error('开始搜索设备失败', err);
					}
				});

				// 监听设备发现
				uni.onBluetoothDeviceFound((devices) => { 
					devices.devices.forEach(device => {
						// 根据前缀过滤设备
						if (device.name && (this.checkLette === '' || device.name.startsWith(this
								.checkLette))) {
							this.devices.push(device);
							if (this.connectedDeviceId && this.connectedDeviceId == device.deviceId) { 
								this.doConnectToDeviceId(this.connectedDeviceId); 
							}
						}
					});

					// 去重处理
					this.devices = [...new Map(this.devices.map(item => [item.deviceId, item])).values()];
				});

				// 可选：设置一个定时器停止搜索
				setTimeout(() => {
					uni.stopBluetoothDevicesDiscovery({
						success: () => {
							console.log('停止搜索设备');
						},
						fail: (err) => {
							console.error('停止搜索设备失败', err);
						}
					});
				}, 10 * 1000); // 停止搜索
			},

			// 连接到蓝牙设备
			connectToDevice(device) {
				uni.setStorageSync('uni_bluetooth_name', device.name);
				this.connectedDeviceName = device.name;
				let deviceId = device.deviceId;
				this.connectedDeviceId = deviceId; // 纪录当前连接的设备ID
				this.doConnectToDeviceId(deviceId);
			},
			// 
			doConnectToDeviceId(deviceId) {
				console.log("connect DeviceId",deviceId);
				uni.createBLEConnection({
					deviceId: deviceId,
					success: (res) => {
						uni.showToast({
							title: "连接成功",
							icon: 'none'
						})
						console.log('成功连接到设备', deviceId);
						this.getDeviceServices(); // 连接成功后获取服务
					},
					fail: (err) => {
						uni.showToast({
							title: "连接设备失败",
							icon: 'none'
						})
						console.error('连接设备失败', err);
					}
				});
			},

			// 获取设备的服务
			getDeviceServices() {
				uni.getBLEDeviceServices({
					deviceId: this.connectedDeviceId,
					success: (res) => {
						console.log('获取到设备服务:', res.services);
						if (res.services.length > 0) {
							// 假设我们取第一个服务
							this.serviceId = res.services[0].uuid;
							this.getDeviceCharacteristics(); // 获取特征值
						}
					},
					fail: (err) => {
						console.error('获取设备服务失败', err);
					}
				});
			},

			// 获取服务中的特征值
			getDeviceCharacteristics() {
				uni.getBLEDeviceCharacteristics({
					deviceId: this.connectedDeviceId,
					serviceId: this.serviceId,
					success: (res) => {
						console.log(res);
						if (res.characteristics.length > 0) {
							res.characteristics.forEach(characteristic => {
								if (characteristic.properties.write) {
									this.characteristicIdWrite = characteristic.uuid;
								} else {
									this.characteristicId = characteristic.uuid;
								}
							});
							this.startListenData(); // 开始监听数据
						}
					},
					fail: (err) => {
						uni.showToast({
							title: '获取设备特征值失败',
							icon: 'none'
						})
						console.error('获取设备特征值失败', err);
					}
				});
			},
			// 开始监听蓝牙设备发来的数据
			startListenData() {
				console.log("deviceId:" + this.connectedDeviceId);
				console.log("serviceId:" + this.serviceId);
				console.log("characteristicId:" + this.characteristicId);
				uni.setStorageSync('uni_bluetooth_deviceId', this.connectedDeviceId);
				uni.setStorageSync('uni_bluetooth_serviceId', this.serviceId);
				uni.setStorageSync('uni_bluetooth_characteristicId', this.characteristicId);
				uni.setStorageSync('uni_bluetooth_characteristicIdWrite', this.characteristicIdWrite);

				uni.notifyBLECharacteristicValueChange({
					state: true,
					deviceId: this.connectedDeviceId,
					serviceId: this.serviceId,
					characteristicId: this.characteristicId,
					success: (res) => {
						console.log('已设置通知', res);
						// 监听特征值变化
						uni.onBLECharacteristicValueChange((characteristic) => {
							const {
								value
							} = characteristic;
							this.receivedData = this.ab2str(value); // 将 ArrayBuffer 转换为字符串
							console.log('接收到数据:', this.receivedData);
						});
					},
					fail: (err) => {
						uni.showToast({
							title: '监听蓝牙数据失败',
							icon: 'none'
						})
						console.error('设置通知失败', err);
					}
				});
			},

			// ArrayBuffer 转字符串的工具方法
			ab2str(buf) {
				return String.fromCharCode.apply(null, new Uint8Array(buf));
			},
			str2ab(str) {
				const buf = new ArrayBuffer(str.length);
				const bufView = new Uint8Array(buf);
				for (let i = 0; i < str.length; i++) {
					bufView[i] = str.charCodeAt(i);
				}
				return buf;
			},
			strToHex(str) {
				return str.split('').map(char => {
					const hex = char.charCodeAt(0).toString(16); // 获取字符的 Unicode 编码并转为 HEX
					return hex.padStart(2, '0'); // 确保每个字节为两位
				}).join('');
			},
			// 将 HEX 字符串转换为 ArrayBuffer
			hexToArrayBuffer(hex) {
				const len = hex.length;
				const buffer = new Uint8Array(len / 2);
				for (let i = 0; i < len; i += 2) {
					buffer[i / 2] = parseInt(hex.substr(i, 2), 16);
				}
				return buffer.buffer;
			},
		}
	};
</script>

<style scoped>
	button {
		margin: 5px;
	}

	.device {
		margin-bottom: 20px;
	}

	.space-between {
		display: flex;
		justify-content: space-between;
		align-items: center;
	}
</style>