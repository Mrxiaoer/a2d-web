<template>
	<section>
		<el-row :gutter="24">
			<el-col :xs="8" :sm="8" :md="8" :lg="8" :xl="8">
				<el-input v-model="url" readonly><template slot="prepend">采集网址</template></el-input>
			</el-col>
			<el-col :xs="3" :sm="3" :md="3" :lg="3" :xl="3">
				<el-button type="primary" @click="startSpider()" v-if="content">爬取表头信息</el-button>
			</el-col>
			<el-col :xs="4" :sm="4" :md="4" :lg="5" :xl="8">
				<el-button type="primary" @click="confirmHandle()" v-if="content">确定选择</el-button>
			</el-col>
			<el-col :xs="4" :sm="4" :md="4" :lg="4" :xl="8">
			  <el-select v-model="waite" clearable placeholder="请选择等待时间" v-if="content">
			    <el-option
			      v-for="item in options"
			      :key="item.value"
			      :label="item.label"
			      :value="item.value">
			    </el-option>
			  </el-select>
			</el-col>
			<el-col :xs="2" :sm="2" :md="2" :lg="2" :xl="2">
				<el-button type="primary" :disabled="isNext" v-if="content" @click="isNext = true">页面操作</el-button>
			</el-col>
			<el-col :xs="2" :sm="2" :md="2" :lg="2" :xl="2">
				<el-button type="primary" :disabled="!isNext" :loading="isLoading" v-if="content" @click="reRenderHandle()">{{isLoading ? "操作中" : "完成"}}</el-button>
			</el-col>
			<el-col :xs="24" style="marginTop: 20px;" v-loading="dialogLoading">
        <el-tooltip class="item" :content="tooltipText" placement="top">
          <div id="spiderContent" class="spiderContent">
						<iframe ref="iframe" id="iframe" frameborder="0" scrolling="auto" @load="loaded"></iframe>
					</div>
        </el-tooltip>
      </el-col>
		</el-row>
		<!-- 弹窗, 确认采集的项目 -->
    <spider-confirm v-if="spiderConfirmVisible" ref="spiderConfirm"></spider-confirm>
	</section>
</template>

<script>
import API from '@/api';
import SpiderConfirm from './spider-confirm';
import { mapMutations } from 'vuex';
import { getXPathForElement, getElementByXPath } from '@/utils';
export default {
	data() {
		return {
			id: '',
			url: '',
			content: '',
			waite: '',
			texts: [],
			isNext: false, //是否点击了下一步，获取点击事件
			button: '',
			elements: [],
			thead: '',
			tooltipText: '请点击表格头部进行采集',
			infoXPath: '', //详情的xpath
			spiderConfirmVisible: false,
			dialogLoading: false,
			isLoading: false,
			options: [
				{
					value: '2000',
					label: '2秒'
				},
				{
					value: '4000',
					label: '4秒'
				},
				{
					value: '6000',
					label: '6秒'
				},
				{
					value: '8000',
					label: '8秒'
				},
				{
					value: '10000',
					label: '10秒'
				}
			]
		};
	},
	components: {
		SpiderConfirm
	},
	watch: {
		waite(val) {
			// this.setSize();
			// this.renderContent();
			// this.startSpider();
			if (val) {
				this.getSpiderResult();
			}
		},
		isNext(val) {
			if (val) {
				this.tooltipText = '采集详细信息：请依次点击“详情”按钮和“确定”按钮';
				this.listenElement();
			}
		}
	},
	activated() {
		this.getSpiderResult();
	},
	methods: {
		loaded() {
			this.setSize();
			this.renderContent();
			this.getChooseElement();
			// this.startSpider();
		},
		// 根据id爬取到需要采集的页面
		getSpiderResult() {
			this.id = this.$store.state.spiderId;
			this.url = this.$store.state.spiderUrl;
			this.dialogLoading = true;
			// console.log(this.id, this.url);
			this.$nextTick(() => {
				const params = {
					linkId: this.id,
					waite: this.waite
				};
				if (this.id) {
					API.spiderconfig.spider(params).then(({ data }) => {
						console.log('data', data);
						if (data && data.code === 0) {
							this.content = data.contents;
							this.renderContent();
						} else {
							this.content = data.msg;
							this.renderContent();
							this.$message.error(data.msg);
						}
						this.dialogLoading = false;
					});
				}
			});
		},
		// 设置尺寸
		setSize() {
			const content = document.getElementById('spiderContent');
			const iframe = document.getElementById('iframe');
			const deviceWidth = document.documentElement.clientWidth;
			const deviceHeight = document.documentElement.clientHeight;
			// content.style.width = deviceWidth + 'px';
			content.style.width = '100%';
			content.style.height = deviceHeight + 'px';
			// iframe.style.width = deviceWidth + 'px';
			iframe.style.width = '100%';
			iframe.style.height = deviceHeight + 'px';
		},
		// 渲染iframe的字符串内容
		renderContent() {
			const iframe = this.$refs.iframe;
			if (iframe && this.content) {
				const doc = this.$refs.iframe.contentWindow.document;
				doc.open();
				doc.write(this.content);
				doc.close();
			}
		},
		// 点击表格开始采集
		startSpider() {
			const that = this;
			const thead = this.getTheadElement();
			this.thead = thead;
			console.log('thead', thead);

			if (thead) {
				// thead.onclick = function() {
				thead.style.backgroundColor = '#17b3a3';
				that.confirmHandle(thead);
				// };
			}
		},
		// 弹窗-询问是否开始采集
		confirmHandle(thead) {
			this.$confirm('您已选择了数据项，是否进行下一步操作?', '操作提示', {
				confirmButtonText: '开始采集',
				cancelButtonText: '取消',
				type: 'warning'
			})
				.then(() => {
					this.spiderHandle(thead);
				})
				.catch(() => {
					if (thead) thead.style.backgroundColor = '';
					this.$message({
						type: 'info',
						message: '已取消采集'
					});
				});
		},
		// 获取采集项目后弹窗采集项目列表
		async spiderHandle(thead) {
			const texts = await this.getSpiderItems(thead);
			console.log('texts', texts);
			if (texts && texts.length > 0) {
				this.spiderConfirmVisible = true;
				this.$nextTick(() => {
					this.$refs.spiderConfirm.init(texts, thead);
				});
			}
		},
		// 从后台获取采集表头项目
		async getSpiderItems(thead) {
			// const thead = this.getTheadElement();
			const xpaths = thead
				? this.getTheadXpath(thead)
				: this.getElementsXpath();

			if (xpaths) {
				const params = {
					linkId: this.$store.state.spiderId,
					xpath: xpaths
				};

				console.log('params', params);

				try {
					const res = await API.spiderconfig.getSpiderItems(params);
					const data = await this.render(res.data);
					return data;
				} catch (err) {
					console.log(err);
				}
			}
		},
		// 渲染采集项目
		async render(data) {
			if (data && data.code === 0) {
				return data.pageInfos;
			} else {
				this.$message.error(data.msg);
			}
		},
		// 监听点击事件，获取选择的项目
		getChooseElement() {
			const self = this;
			const iframe = this.$refs.iframe;

			if (iframe) {
				const doc = iframe.contentWindow.document;
				const tags = doc.getElementsByTagName('*');
				const elements = [];

				for (let i = 0; i < tags.length; i++) {
					const element = tags[i];
					const tagName = element.tagName;

					element.onclick = function(event) {
						// 阻止默认提交事件
						event.preventDefault();
						event.stopPropagation();

						var e =
							event || window.event || arguments.callee.caller.arguments[0];

						if (e) {
							// console.log('e', e);
							const element = e.target || e.srcElement;
							// 如果点击了图标元素，就获取它的父元素作为element
							if (element.tagName == 'I') element = element.parentNode;

							// 判断元素是否有isChoose类
							if (self.hasClass(element, 'isChoose')) {
								// 如果已经有这个类，就清空颜色，删除元素
								element.style.backgroundColor = '';
								self.removeClass(element, 'isChoose');
								const index = self.elements.indexOf(element);
								if (index > -1) {
									self.elements.splice(index, 1);
								}
							} else {
								// 否则添加颜色，添加元素
								element.style.backgroundColor = '#17b3a3';
								self.addClass(element, 'isChoose');
								self.elements.push(element);
							}

							console.log('self.elements', self.elements);
						}
					};
				}
			}
		},
		// 获取选择元素的xpath
		getElementsXpath() {
			if (this.elements.length > 0) {
				return this.elements.map(item => {
					return getXPathForElement(item).replace('html[1]', 'html');
				});
			}
		},
		// 获取表头的xpath
		getTheadXpath(thead) {
			if (thead) {
				return getXPathForElement(thead).replace('html[1]', 'html');
			}
		},
		// 获取thead元素
		getTheadElement() {
			const iframe = this.$refs.iframe;
			if (iframe) {
				const doc = iframe.contentWindow.document;
				const tables = doc.getElementsByTagName('table');
				// 遍历table元素，只获取显示的并且有thead子元素的table
				for (const i in tables) {
					if (tables.hasOwnProperty(i)) {
						const table = tables[i];
						if (
							table.clientHeight > 0 &&
							table.clientWidth > 0 &&
							table.children.length > 0
						) {
							for (const l in table.children) {
								if (table.children.hasOwnProperty(l)) {
									const children = table.children[l];
									if (children.tagName == 'THEAD') {
										return children;
									}
								}
							}
						}
					}
				}
			}
		},
		// 开始采集详情信息，监听鼠标点击事件
		listenElement() {
			const self = this;
			const iframe = this.$refs.iframe;

			if (iframe && this.isNext) {
				const doc = iframe.contentWindow.document;
				const tags = doc.getElementsByTagName('*');
				const buttons = [];

				for (let i = 0; i < tags.length; i++) {
					const element = tags[i];
					const tagName = element.tagName;

					// 判断类型，只有button或a链接才能触发事件
					// if (tagName == 'BUTTON' || tagName == 'A') {
					if (true) {
						element.onclick = function(event) {
							// 阻止默认提交事件
							event.preventDefault();
							event.stopPropagation();

							var e =
								event || window.event || arguments.callee.caller.arguments[0];

							if (e) {
								// console.log('e', e);
								const button = e.target || e.srcElement;
								// 如果点击了图标元素，就获取它的父元素作为button
								if (button.tagName == 'I') button = button.parentNode;

								self.button = button;
							}
						};
					}
				}
			}
		},
		// 获取详情的xpath，点击确定后，重新渲染页面
		reRenderHandle() {
			if (this.isNext && this.button) {
				const params = {
					linkId: this.$store.state.spiderId,
					waite: this.waite,
					xpath: getXPathForElement(this.button).replace('html[1]', 'html')
				};

				console.log('params', params);
				this.isLoading = true;
				this.dialogLoading = true;

				console.log('已安确定按钮~');
				API.spiderconfig.spider(params).then(({ data }) => {
					console.log('data', data);
					if (data && data.code === 0) {
						this.content = data.contents;
						this.renderContent();
						this.getChooseElement();
					} else {
						this.content = data.msg;
						this.renderContent();
						this.$message.error(data.msg);
					}
					this.elements = [];
					this.dialogLoading = false;
					this.isNext = false;
					this.isLoading = false;
				});
			}
		},
		// 判断是否有这个类名
		hasClass(ele, cls) {
			return ele.className.match(new RegExp('(\\s|^)' + cls + '(\\s|$)'));
		},
		//为指定的dom元素添加样式
		addClass(ele, cls) {
			if (!this.hasClass(ele, cls)) ele.className += ' ' + cls;
		},
		//删除指定dom元素的样式
		removeClass(ele, cls) {
			if (this.hasClass(ele, cls)) {
				var reg = new RegExp('(\\s|^)' + cls + '(\\s|$)'); //正则表达式，需要补血
				ele.className = ele.className.replace(reg, ' ');
			}
		},
		//如果存在(不存在)，就删除(添加)一个样式
		toggleClass(ele, cls) {
			if (this.hasClass(ele, cls)) {
				this.removeClass(ele, cls);
			} else {
				this.addClass(ele, cls);
			}
		},
		// 从前端获取采集表头项目,
		// Note: 此功能已弃用
		getSpiderItemsFront() {
			const texts = [];
			const content = document.getElementById('spiderContent');
			const tr = content
				.getElementsByTagName('table')[0]
				.getElementsByTagName('thead')[0]
				.getElementsByTagName('tr')[0].children;
			for (let i = 0; i < tr.length; i++) {
				const element = tr[i];
				texts.push(element.innerText);
			}
			this.texts = texts;
		},
		...mapMutations(['ADD_CONTENT_TAB', 'UPDATE_CONTENT_TABS'])
	}
};
</script>

<style lang="scss">
.spiderContent {
	padding: 20px;
	border: 2px solid #17b3a3;
	border-radius: 4px;
	cursor: pointer;
	overflow: auto;
}
</style>