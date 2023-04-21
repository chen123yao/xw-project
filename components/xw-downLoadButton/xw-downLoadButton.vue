<template>
	<view class="xw-downLoadButton">
		<view class="" v-if="item">
			<view class="buttons"  v-if="downtype==3">
				<view class="button" @click="handleOpenApp">启 动</view>
			</view>
			<view class="buttons"  v-else-if="downtype==4">
				<view class="button" @click="installApp(item.filename)">未安装</view>
			</view>
			<view class="buttons" v-else-if="isTrue&&!downtype" @click="handleStop"	>
				<nvue-progress :loadLoading='progress' :mheight='8' :wNumber='2.5' :type='1'></nvue-progress>
			</view>
			<view class="buttons" v-else>
				<view class="button" @click="handleResume">继 续</view>
			</view>
		</view>
	</view>
</template>

<script>
	export default{
		data(){
			return{
				downtype:0,
				isTrue: true,
				progress:0,
			}
		},
		props:{
			item:{
				type:Object,
				default:()=>{}
			},
			isOpen:{
				type:Number,
				default:0
			}
		},
		methods:{
			//打开
			handleOpenApp(){
				plus.runtime.launchApplication( {pname:this.item.package_name}, err => {uni.showToast({ title:'打开失败',icon:'none'	})} );
			},
			//安装
			installApp(name){
				console.log('安装游戏', name);
				plus.runtime.openFile(name, null, (err) => {
					uni.showToast({ title:err.message + ' 请重新下载！' ,icon:'none'	})
					
					let that = this
					let downTaskList = this.$store.state.downTasksList
					plus.downloader.enumerate(function(tasks){
						if (tasks.length) {
							console.log('tasksLength', tasks.length)
							console.log('downTaskList', downTaskList)
							downTaskList.forEach(item => {
								console.log('downTaskList_item', item)
								
								downTaskList.splice(downTaskList.findIndex(taskitem => taskitem.game_id == item.game_id), 1)
								console.log('下载任务删除后的list:', downTaskList)
								tasks.forEach(task => {
									let downTaskItem = JSON.parse(task.options.data)
									console.log('taskAndtaskItem', downTaskItem)
									if (downTaskItem.game_id == item.game_id) {
										let fileName = task.filename
										console.log('删除下载任务:', fileName, item.gamename)
										task.abort()
									}
								})
								
								that.$store.commit('setDownTasksList', downTaskList) // 更新列表数据
								console.log('1111111')
								that.$emit('delItem', downTaskList)
								
								if (downTaskList.length <= 0) {
									console.log('下载管理已清空')
									plus.downloader.clear(-1);
								}
								
								uni.navigateTo({
									url: `/pages/view/gamedetail/gamedetail?gameid=${item.game_id}`
								})
							})
						}
					},-1)
					
					
					
				});
			},
			//暂停App
			handleStop(){
				console.log('暂停下载', this.downTask);
				this.isTrue = false;
				this.downTask.pause();
				
				let taskList = this.$store.state.downTasksList // 获取下载管理列表
				taskList.forEach((task)=>{ //遍历列表到找到当前游戏的下载任务
					if (this.item.game_id == task.game_id) { //通过游戏id寻找到当前任务
						task.status = 5 // 将当前任务的状态修改为暂停  5 
						console.log('暂停下载并设置status为5', task.status)
					}
				})
				console.log('taskList_pause', taskList)
				this.$store.commit('setDownTasksList', taskList) // 更新列表数据
			},
			//继续
			handleResume(){
				console.log('继续下载', this.downTask);
				this.isTrue = true;
				this.downTask.resume();
				
				let taskList = this.$store.state.downTasksList // 获取下载管理列表
				taskList.forEach((task)=>{ //遍历列表到找到当前游戏的下载任务
					if (this.item.game_id == task.game_id) { //通过游戏id寻找到当前任务
						task.status = 3 // 将当前任务的状态修改为暂停  5 
						console.log('继续下载并设置status为3', task.status)
					}
				})
				console.log('taskList_resume', taskList)
				this.$store.commit('setDownTasksList', taskList) // 更新列表数据
			},
			//开始下载
			startApp(){
				console.log('开始下载', this.downTask);
				this.downTask.start();
				
				let taskList = this.$store.state.downTasksList // 获取下载管理列表
				taskList.forEach((task)=>{ //遍历列表到找到当前游戏的下载任务
					if (this.item.game_id == task.game_id) { //通过游戏id寻找到当前任务
						task.status = 3 // 将当前任务的状态修改为暂停  5 
						console.log('开始下载并设置status为3', task.status)
					}
				})
				console.log('开始下载taskList', taskList)
				this.$store.commit('setDownTasksList', taskList) // 更新列表数据
				// let taskList = this.$store.state.downTasksList // 获取下载管理列表
				// taskList.unshift(this.item); // 将当前游戏任务添加至下载管理列表
				// this.$store.commit('setDownTasksList', taskList) // 更新列表数据
			},
			// 创建下载任务函数
			createDownload() {
				let  isinstallGame = uni.getStorageSync('isinstallGame').status;
				this.downTask = plus.downloader.createDownload(this.item.down_url, {data:JSON.stringify(this.item)},  (downloadFile, status) => {
					// 下载完成后的回调函数，成功失败都会进来
					if (status == 200) {
				        // this.$emit('getData',null)	 
						this.item.filename = downloadFile.filename;
						this.downtype = 4;
						if (isinstallGame) {
							this.installApp(downloadFile.filename)
						} else {
							uni.showToast({
								title:"下载完成",
								icon:'none'
							})
					   }
					}else{
						// uni.showToast({
						// 	title:'下载失败，请重新下载',
						// 	icon:'none',
						// 	success: () => {
								
						// 	}
						// })
					}
				})
				
				console.log('创建下载任务：', this.downTask)
				this.startApp()
				this.addEventList(this.downTask)
			},
			//监听
			addEventList(item){
				// console.log('监听下载任务：', item)
				item.addEventListener("statechanged", (res, status)=>{
					// console.log('监听任务：', res, status)
					switch(res.state) {
						case 1: // 开始
							console.log('开始下载...');
							break;
						case 2: // 已连接到服务器
							console.log('链接到服务器...');
							break;	
						case 3: // 已接收到数据	
							let _taskList = this.$store.state.downTasksList // 获取下载管理列表
							_taskList.forEach((task)=>{ //遍历列表到找到当前游戏的下载任务
								if (this.item.game_id == task.game_id) { //通过游戏id寻找到当前任务
									// 将当前任务的状态修改为暂停  5 
									if (task.status != 3) {
										this.handleStop()
									} else {
										this.item.myloading = this.changeSize(res.downloadedSize)
										this.item.myTotalData = this.changeSize(res.totalSize)
										if (res.downloadedSize > res.totalSize) {
											this.item.myloading = this.item.myTotalData
										}
										this.progress = Math.round(res.downloadedSize / res.totalSize * 100);
									}
								}
							})
							break;
						case 4: // 下载完成
							console.log('下载完成！');
							this.item.filename = res.filename;
							this.downtype = 4;
							
							let taskList = this.$store.state.downTasksList // 获取下载管理列表
							taskList.forEach((task)=>{ //遍历列表到找到当前游戏的下载任务
								if (this.item.game_id == task.game_id) { //通过游戏id寻找到当前任务
									task.status = 4 // 将当前任务的状态修改为下载完成  4 
									task.filename = res.filename
									console.log('下载完成并设置status为4', task.status)
								}
							})
							console.log('下载完成taskList', taskList)
							this.$store.commit('setDownTasksList', taskList) // 更新列表数据
							break;
					}
				})
			},
			changeSize(size) {
				let num = 0;
				if((size / 1024) <= 1024) {
					num = size / 1024;
					return num.toFixed(2) + 'KB';
				}
				if((size / 1024) > 1024 && (size / 1024) <= (1024 * 1024)) {
					num = (size / 1024) / 1024;
					return num.toFixed(2) + 'M';
				}
				if((size / 1024) > (1024 * 1024)) {
					num = ((size / 1024) / 1024) / 1024
					return num.toFixed(2) + 'G';
				}
			},
			initBtn() {
				let that = this;
				// enumerate方法获取所有的下载列表 并没有办法通过唯一id快速找到当前游戏的download 只能遍历所有去查找
				plus.downloader.enumerate(function(tasks) {
					console.log('mounted_获取下载列表数量: ' + tasks.length , tasks)
					let createTask = true; // 默认判断为新任务 通过后续的判断改变该状态
					if (tasks.length) {
						tasks.forEach(item => {
							let downTaskItem = JSON.parse(item.options.data);
							// console.log('mounted_获取每个下载任务：' , downTaskItem)
							// console.log('mounted_当前游戏id：' , that.item.game_id)
							if (downTaskItem.game_id == that.item.game_id) { // 通过游戏id判断 当前游戏是否已经存在于下载列表中
								console.log('mounted_获取当前游戏task', downTaskItem.game_id, downTaskItem.gamename)
								createTask = false; // 当前游戏存在于下载列表 则不再创建下载任务
								if (plus.runtime.isApplicationExist({pname:downTaskItem.package_name})) { // 通过下载完成的文件名判断当前游戏下载任务是否已经完成
									console.log('游戏已安装')
									that.downtype = 3; // 如果已经下载完成， 则将这个type 设置为显示打开的对应值
								} else {
									// 游戏未安装则处于多种可能的状态下  下载中/暂停/下载完成未安装 通过后续判断
									that.item.myloading = that.changeSize(item.downloadedSize); // 这里通过枚举下载的回调拿到当前游戏已经下载的数据大小 用来返还给父组件显示已经下载的大小
									that.item.myTotalData = that.changeSize(item.totalSize); // 这里通过枚举下载的回调拿到当前游戏总下载数据大小 用来返还给父组件显示
									if (item.downloadedSize > item.totalSize) {
										that.item.myloading = that.item.myTotalData
									}
									that.progress = Math.round(item.downloadedSize / item.totalSize * 100); // 这里通过枚举下载的回调拿到当前游戏已下载和总下载数据大小计算出进度条的值 用来显示下载进度
									that.downTask = item;
									// console.log('给downTask赋值', that.downTask)
									console.log('当前下载任务状态：' ,that.item.status)
									switch(that.item.status) {
										case 3:
											console.log('游戏正在下载安装包')
											that.isTrue = true;
											that.downtype = 0;
											// that.downTask.resume();
											that.addEventList(that.downTask);
											break;
										case 4:
											console.log('游戏已下载完还未安装')
											that.item.filename = item.filename;
											that.downtype = 4;
											if (that.item.isOpen == 9) {
												that.downtype = 3;
											}
											break;
										case 5:
											console.log('游戏已暂停下载安装包')
											that.isTrue = false;
											that.downtype = 0;
											that.addEventList(that.downTask);
											break;
									}
								}
							}
						})
						if (createTask) {
							console.log('当前下载列表找不到该游戏下载任务, 可以创建下载任务')
						}
					} else {
						console.log('下载列表为空，找不到下载任务，可以创建下载任务')
					}
					if (createTask) {
						that.createDownload();
					}
					console.log('当前downType（0下载中/暂停，3安装完成，4已下载未安装）：', that.downtype)
				}, -1)
			}
		},
		beforeDestroy() {
			console.log('beforeDestroy')
		},
		mounted() {
			this.initBtn();
		},
		watch: {
			isOpen: {
				handler(val){
					console.log(val)
					if (val == 9) {
						console.log('watch_open111', val)
						this.downtype = 3
					}
				},
				immediate: true
			}
		}
	}
</script>

<style scoped lang="scss">
	.buttons {
           @include flex;
		   justify-content: center;
			width: 140rpx;
			height: 60rpx;
			line-height: 60rpx;
			text-align: center;
			// padding:0 32rpx; 
			border: 2rpx solid #efefef;
			border-radius: 48rpx;
			// @include button(32rpx, 10rpx, 32rpx);
		.button {
			/* #ifndef APP-NVUE */
			white-space: nowrap;
			/* #endif */
			font-size: $xw-font;
			color: #FF5927;
		}
	}
</style>
