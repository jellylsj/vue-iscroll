<template>
<div class="vue-iscroll-wrapper" :style="{bottom: bottomHeight}">
	<div class="vue-iscroll-scroller">
		<slot></slot>
		<slot name="pulldown"></slot>
  		<slot name="pullup"></slot>
	</div>
</div>
</template>
<script>
var iScroll = require('iscroll/build/iscroll-probe');

import {
	Pulldown,
	Pullup,
	addClass,
	removeClass,
	containClass
} from './pull.js'

var pullThreshold = 5;

const pulldownDefaultConfig = {
	content: 'Pull Down To Refresh',
	height: 60,
	autoRefresh: false,
	upContent: 'Pull Down To Refresh',
	downContent: 'Release To Refresh',
	loadingContent: 'Loading...',
	clsPrefix: 'vue-iscroll-pulldown-'
}
const pullupDefaultConfig = {
	content: 'Pull Up To Refresh',
	pullUpHeight: 60,
	height: 40,
	autoRefresh: false,
	upContent: 'Release To Refresh',
	downContent: 'Pull Up To Refresh',
	loadingContent: 'Loading...',
	clsPrefix: 'vue-iscroll-pullup-'
}
const scrollDefaultConfig = {
	probeType: 2,
	bounceTime: 250,
	bounceEasing: 'quadratic',
	mouseWheel: false,
	scrollbars: true,
	fadeScrollbars: true,
	interactiveScrollbars: false,
	scrollX: false,
	scrollY: true
}

export default {
	name: "vue-iscroller",
	props: {
		scrollConfig:  {
			type: Object,
			default: () => {}
		},
		bottomHeight: {
			type: String,
			default: '0'
		},
		usePulldown: {
			type: Boolean,
			default: false
		},
		usePullup: {
			type: Boolean,
			default: false
		},
		pulldownConfig: {
			type: Object,
			default: () => {}
		},
		pullupConfig: {
			type: Object,
			default: () => {}
		}
	},
	compiled() {
		this.uuid = Math.random().toString(36).substring(3, 8);
	},
	ready() {
		this.$el.setAttribute('id', `vue-iscroll-scroller-${this.uuid}`);
		let content = null;
		const slotChildren = this.$el.querySelector('.vue-iscroll-scroller').childNodes;
		for (let i = 0; i < slotChildren.length; i++) {
			if (slotChildren[i].nodeType === 1) {
				content = slotChildren[i];
				break;
			}
		}

		if (!content) {
			throw new Error('no content is found');
		}

		var options = Object.assign(scrollDefaultConfig, this.scrollConfig);

		if(options.scrollX){
			//增加x轴宽度计算自适应宽度
			const contentChild = content.childNodes;
			let offsetWidth = 0;
			for (let i = 0; i < contentChild.length; i++) {
				if (contentChild[i].nodeType === 1) {
					offsetWidth += contentChild[i].offsetWidth
				}
			}

			this.$el.querySelector('.vue-iscroll-scroller').style.width = offsetWidth + 'px';
		}

		this._scroller = new iScroll('.vue-iscroll-wrapper', options);
		if (this.usePulldown) {
			// if use slot=pulldown
			let config = Object.assign(pulldownDefaultConfig, this.pulldownConfig);
			config.container = this.$el.querySelector('.vue-iscroll-scroller');

			if (!config.container) {
				throw new Error('pulldown has no container')
			}
			//构建pulldown的HTML
			var pulldown = this.pulldown = new Pulldown(config);
			pulldown.on('loading', () => {
				this.$dispatch('pulldown:loading', this.uuid)
			})
			var pulldownOffset = pulldown.element.offsetHeight;

		}

		if (this.usePullup) {
			let config = Object.assign(pullupDefaultConfig, this.pullupConfig);
			config.container = this.$el.querySelector('.vue-iscroll-scroller');

			if (!config.container) {
				throw new Error('pullup has no container')
			}
			//构建pullup的HTML
			var pullup = this.pullup = new Pullup(config);
			pullup.on('loading', () => {
				this.$dispatch('pullup:loading', this.uuid)
			})
			pullup.on('complete', () => {
				this.$dispatch('pullup:complete', this.uuid)
			})
			var pullupOffset = pullup.element.offsetHeight;
		}

		let startPos = null;
		this._scroller.on('scrollStart', function() {
			startPos = this.y;
		})

		var that = this; //保存this

		that._scroller.on('scroll', function() {
			if (that.usePulldown || that.usePullup) {
				/* 
	    			'scroll' called, but scroller is not moving!
						Probably because the content inside wrapper is small and fits the screen, so drag/scroll is disabled by iScroll.
						Fix this by a hack: Setting "myScroll.hasVerticalScroll=true" tricks iScroll to believe
						that there is a vertical scrollbar, and iScroll will enable dragging/scrolling again...
					*/
				this.hasVerticalScroll = true;
				startPos = -1000;
			} else if (startPos === -1000 && ((!that.usePullup && (this.y < 0)) || ((!that.usePulldown) && (this.y > 0)))) {
				/* 
	    			Scroller was not moving at first (and the trick above was applied), but now it's moving in the wrong direction.
						I.e. the user is either scrolling up while having no "pull-up-bar",
						or scrolling down while having no "pull-down-bar" => Disable the trick again and reset values...
					*/
				this.hasVerticalScroll = false;
				startPos = 0;
				this.scrollBy(0, -this.y, 0); //Adjust scrolling position to undo this "invalid" movement
			}

			if (that.usePulldown) {
				let config = Object.assign(pulldownDefaultConfig, this.pulldownConfig);
				if (this.y > pulldownOffset + pullThreshold && !containClass(pulldown.element, config.clsPrefix + 'down')) {
					pulldown.release();
					//console.log('call release')
					this.scrollBy(0, -pulldownOffset, 0); // Adjust scrolling position to match the change in pullDownEl's margin-top
				} else if (this.y < 0 && containClass(pulldown.element,  config.clsPrefix + 'down')) { // User changes his mind...
					pulldown.pull();
					this.scrollBy(0, pulldownOffset, 0); // Adjust scrolling position to match the change in pullDownEl's margin-top
				}

			}

			if (that.usePullup) {
				let config = Object.assign(pullupDefaultConfig, this.pullupConfig);
				if (this.y < (this.maxScrollY - pullupOffset + pullThreshold) && !containClass(pullup.element, config.clsPrefix + 'up')) {
					pullup.release();

				} else if (this.y > (this.maxScrollY - pullupOffset + pullThreshold) && containClass(pullup.element, config.clsPrefix + 'up')) {
					pullup.push();
				}
			}


		})

		that._scroller.on('scrollEnd', function() {
			if(pulldown){
				let config = Object.assign(pulldownDefaultConfig, this.pulldownConfig);
				if(containClass(pulldown.element,  config.clsPrefix + 'down')){
					//console.log('scroll end')
					pulldown.loading();
				}
			}

			if(pullup){
				let config = Object.assign(pullupDefaultConfig, this.pullupConfig);
				if(containClass(pullup.element, config.clsPrefix + 'up')){
					//console.log('scroll end');
					//console.log(this)
					//this.scrollBy(0,-pullupOffset, 0);
					pullup.loading();
				}
			}

			if (startPos === -1000) {
				// If scrollStartPos=-1000: Recalculate the true value of "hasVerticalScroll" as it may have been
				// altered in 'scroll' to enable pull-to-refresh/load when the content fits the screen...
				this.hasVerticalScroll = this.options.scrollY && this.maxScrollY < 0;
			}
		});

	},

	methods: {
		reset(timeout = 0) {
			//console.log('reset');
			//console.log(this._scroller);
			this._scroller && setTimeout(() => {
				this._scroller.refresh();
			}, timeout);
		},
		complete() {
			this.pullup = null;
			delete this.pullup;
			this.usePullup = false;
		}
	},
	events: {
		'init-callback': function(uuid) {
			//console.log('reset event')
			this.$el.setAttribute('id', `vue-iscroll-scroller-${this.uuid}`);
			let content = null;
			const slotChildren = this.$el.querySelector('.vue-iscroll-scroller').childNodes;
			for (let i = 0; i < slotChildren.length; i++) {
				if (slotChildren[i].nodeType === 1) {
					content = slotChildren[i];
					break;
				}
			}

			if (!content) {
				throw new Error('no content is found');
			}

			var options = Object.assign(scrollDefaultConfig, this.scrollConfig);

			if(options.scrollX){
				//增加x轴宽度计算自适应宽度
				const contentChild = content.childNodes;
				let offsetWidth = 0;
				for (let i = 0; i < contentChild.length; i++) {
					if (contentChild[i].nodeType === 1) {
						offsetWidth += contentChild[i].offsetWidth
					}
				}

				this.$el.querySelector('.vue-iscroll-scroller').style.width = offsetWidth + 'px';
			}
			this.reset();
		},
		'scroll-reset': function(uuid) {
			//console.log('reset event')
			this.reset();
		},
		'scroll-to': function(x, y, time, easing) {
			//滚动到任意的位置
			this._scroller.scrollTo(x, y, time, easing);
		},
		'scroll-to-element': function(el, time, offsetX, offsetY, easing){
			//滚动到这个元素的左上角位置。
			this._scroller.scrollToElement(el, time, offsetX, offsetY, easing);
		},
		//下拉刷新，重置iscroll
		'pulldown:reset': function(uuid) {
			if (uuid === this.uuid) {
				this.pulldown.reset(() => {
					this.reset();
				})
			}
		},
		//上拉加载，重置iscroll
		'pullup:reset': function(uuid) {
			if (uuid === this.uuid) {
				this.pullup.reset(() => {
					this.reset();
				})
			}
		},
		//上拉加载，数据加载完毕
		'pullup:done': function() {
			if (true) {
				this.pullup.complete(() => {
					this.reset();
				});

				this.complete();

			}
		}
	},
	beforeDestroy() {
		this._scroller.destroy();
	}
}
</script>
<style>
.vue-iscroll-wrapper{
	position: absolute;
	z-index: 1;
	top: 0;
	left: 0;
	width: 100%;
	overflow: hidden;
}
.vue-iscroll-scroller{
	position: absolute;
	width: 100%;
}
.vue-iscroll-pulldown-container,.vue-iscroll-pullup-container{
	margin-top: 0;
	background: #fff;
	transition: all linear 250ms;
}
.vue-iscroll-pulldown-up{
	margin-top: -60px;
}
.vue-iscroll-pulldown-down{
	transition: none;
}
.vue-iscroll-pulldown-loading{
	transition: none;
}

/* pullup */
.vue-iscroll-pullup-loading{
	
}
.vue-iscroll-pullup-down{
}
.vue-iscroll-pullup-up{
}
</style>