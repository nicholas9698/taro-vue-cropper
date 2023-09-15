<template>
    <view class="image-cropper-wrapper">
        <view class="bg_container">
            <view class="bg_top"></view>
            <view class="bg_middle">
                <view class="bg_middle_left"></view>
                <view class="cut_wrapper" :style="{ width: _cut_width + 'px', height: _cut_height + 'px'}">
                    <view class="border border-top-left"></view>
					<view class="border border-top-right"></view>
					<view class="border border-right-top"></view>
					<view class="border border-bottom-right"></view>
					<view class="border border-right-bottom"></view>
					<view class="border border-bottom-left"></view>
					<view class="border border-left-bottom"></view>
					<view class="border border-left-top"></view>
                </view>
                <view class="bg_middle_right"></view>
            </view>
            <view class="bg_bottom"></view>
        </view>
        <image class="img" :src="img" :style="{
            width: _img_width * scale + 'px',
            height: _img_height * scale + 'px',
            top: _img_top - (_img_height * (scale - 1)) / 2  + 'px',
            left: _img_left - (_img_width * (scale - 1)) /2 + 'px',
        }" @touchstart="_img_touch_start" @touchmove="_img_touch_move" @touchend="_img_touch_end"/>
		<IconFont class="btn1" name="success" size="2em" color="#70f3ff" @click="detect"></IconFont>
		<IconFont class="btn2" name="circle-close" size="2em" color="#fa2c19" @click="cancel"></IconFont>
		<div class="loading" :style="{zIndex: loading}">
			<IconFont name="loading1" size="25vw" color="#ffffff"></IconFont>
		</div>

        <canvas type="2d" canvas-id="my-canvas" class="my-canvas-class" :style="{
            width: _canvas_width + 'px',
            height: _canvas_height + 'px',
            left: _canvas_left + 'px',
            top: _canvas_top + 'px'
        }" ></canvas>
    </view>
</template>
<script>
import { toRefs, reactive, getCurrentInstance } from 'vue';
import Taro from '@tarojs/taro';
import { IconFont } from '@nutui/icons-vue-taro';
export default {
    name: 'Cropper',
    components: {
		IconFont
},
    setup() {
        const state = reactive({
            img: '',
			loading: -2,
            _cut_width: 200,
            _cut_height: 200,
            _cut_left: 0,
            _cut_top: 0,
			_cut_ratio: 4/3,
			_img_ratio: 1,
            _img_height: 0,
            _img_width: 0,
            _img_left: 0,
            _img_top: 0,
            _canvas_height: 0,
            _canvas_width: 0,
            _canvas_left: 0,
            _canvas_top: 0,
			_window_height: 0,
			_window_width: 0,
            scale: 1,
			max_scale: 2,
			min_scale: 0.5,
            _touch_end_flag: true,
            _touch_potinter_one: true,
			_hypotenuse_length: 0,
			ctx: null,
			_img_touch_relative: [{
				x: 0,
				y: 0
			},
			{
				x: 0,
				y: 0
			}]
        });
		Taro.getSystemInfo({
			success: function (res) {
				state._window_height = res.windowHeight;
				state._window_width = res.windowWidth;
				let initial_cut_width = Math.floor((state._window_width * 2) / 3);
				let initial_cut_height = initial_cut_width / state._cut_ratio;
				if (initial_cut_height >= state._window_height) {
					initial_cut_height = Math.floor(state._window_height / 2);
					initial_cut_width = initial_cut_height * state._cut_ratio;
				}
				state._cut_width = initial_cut_width;
				state._cut_height = initial_cut_height;
				state._cut_top = (state._window_height - state._cut_height) / 2;
				state._cut_left = (state._window_width - state._cut_width) / 2;
			}
		});
		Taro.eventCenter.on('acceptDataFromOpenedPage', (data) => {
			state.img = data.img;
			Taro.getImageInfo({
				src: state.img,
				success: function (res) {
					state._img_ratio = res.width / res.height;
					state._img_width = state._window_width * 0.8;
					state._img_left = state._window_width * 0.1;
					state._img_height = 0.8 * (res.height / (res.width / state._window_width));
					state._img_top = 0.1 * (res.height / (res.width / state._window_width));
				}
			});
        });

		// const initCanvas = () => {
		// 	state.ctx = Taro.createCanvasContext('my-canvas');
		// }

        const _img_touch_start = (e) => {
            state._touch_end_flag = false;
            if (e.touches.length === 1) {
                state._touch_potinter_one = true;
				state._img_touch_relative[0] = {
					x: e.touches[0].clientX - state._img_left,
					y: e.touches[0].clientY - state._img_top
				}
            } else {
				state._touch_potinter_one = false;
				let width = Math.abs(e.touches[0].clientX - e.touches[1].clientX);
				let height = Math.abs(e.touches[0].clientY - e.touches[1].clientY);
				state._hypotenuse_length = Math.sqrt(
					Math.pow(width, 2) + Math.pow(height, 2)
				);
			}
        }
		const _img_touch_end = () => {
			state._touch_end_flag = true;
		}

		const _img_touch_move = (e) => {
			if (state._touch_end_flag) {
				return;
			}
			if (e.touches.length === 1 && state._touch_potinter_one) {
				let left = e.touches[0].clientX - state._img_touch_relative[0].x;
				let top = e.touches[0].clientY - state._img_touch_relative[0].y;
				setTimeout(() => {
					state._img_left = left;
					state._img_top = top;
				}, 0);
			} else if (e.touches.length >=2 && !state._touch_potinter_one) {
				// 双指放大
				let width = Math.abs(e.touches[0].clientX - e.touches[1].clientX);
				let height = Math.abs(e.touches[0].clientY - e.touches[1].clientY);

				let new_hypotenuse_length = Math.sqrt(
					Math.pow(width, 2) + Math.pow(height, 2)
				);
				let newScale = state.scale * (new_hypotenuse_length / state._hypotenuse_length);
				newScale = newScale > state.max_scale || newScale < state.min_scale ? state.scale : newScale;
				state._hypotenuse_length = new_hypotenuse_length;
				setTimeout(() => {
					state.scale = newScale;
				}, 0);
			}
		}
		const detect = () => {
			state.loading = 0;
		}
		const cancel = () => {
			state.loading = -2;
			state.img = '';
			Taro.eventCenter.trigger('cancelOcr', null);
		}
        return {
            ...toRefs(state),
            _img_touch_start,
			_img_touch_end,
			_img_touch_move,
			detect,
			cancel
        }
    }
}
</script>
<style lang="scss">
.image-cropper-wrapper {
	width: 100vw;
	height: 100vh;
	position: fixed;
	top: 0;
	left: 0;
	z-index: 1;
	.bg_container {
		height: 100%;
		width: 100%;
		display: flex;
		flex-direction: column;
		justify-content: center;
		pointer-events: none;
		.bg_top {
			flex: 1;
			background-color: rgba(0, 0, 0, 0.3);
		}
		.bg_middle {
			display: flex;
			.bg_middle_left,
			.bg_middle_right {
				flex: 1;
				background-color: rgba(0, 0, 0, 0.3);
			}
			.cut_wrapper {
				position: relative;
				.border {
					background-color: rgba(255, 255, 255, 0.4);
					position: absolute;
				}
				.border-top-left {
					height: 4px;
					top: -4px;
					left: -4px;
					width: 30rpx;
				}
				.border-left-top {
					height: 30rpx;
					top: -4px;
					left: -4px;
					width: 4px;
				}
				.border-top-right {
					top: -4px;
					right: -4px;
					width: 30rpx;
					height: 4px;
				}
				.border-right-top {
					top: -4px;
					right: -4px;
					height: 30rpx;
					width: 4px;
				}
				.border-right-bottom {
					bottom: -4px;
					right: -4px;
					height: 30rpx;
					width: 4px;
				}
				.border-bottom-right {
					width: 30rpx;
					height: 4px;
					bottom: -4px;
					right: -4px;
				}
				.border-left-bottom {
					height: 30rpx;
					width: 4px;
					bottom: -4px;
					left: -4px;
				}
				.border-bottom-left {
					width: 30rpx;
					height: 4px;
					bottom: -4px;
					left: -4px;
				}
			}
		}
		.bg_bottom {
			flex: 1;
			background-color: rgba(0, 0, 0, 0.3);
		}
	}
	.img {
		position: absolute;
		z-index: -1;
		backface-visibility: hidden;
		transform-origin: center center;
	}
	.btn1 {
		bottom: 10rpx;
		right: calc(50vw - 250rpx);
		position: absolute;
		z-index: 0;
	}
	.btn2 {
		bottom: 10rpx;
		left: calc(50vw - 250rpx);
		position: absolute;
		z-index: 0;
	}
	.loading {
		height: 25vw;
		width: 25vw;
		position: absolute;
		left: 37.5vw;
		top: calc(50vh - 12.5vw - 150rpx);
	}
	.my-canvas-class {
		position: absolute;
		z-index: -2;
	}
}
</style>
