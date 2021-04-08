<template>
  <transition name="viewer-fade">
    <div class="dmj-image__wrapper" :style="{ 'z-index': zIndex }">
      <div class="dmj-image__mask" @click.self="handleMaskClick"></div>
      <!-- 关闭 -->
      <span class="dmj-image__btn dmj-image__close" @click="hide">
         <a-icon type="close" />
      </span>
      <!-- 操作 -->
      <div class="dmj-image__btn dmj-image__handel">
        <div class="dmj-image__handel__inner">
					<a-icon type="zoom-out" @click="handleActions('zoomOut')"/>
					<a-icon type="zoom-in" @click="handleActions('zoomIn')"/>
					<a-icon type="undo" @click="handleActions('anticlocelise')"/>
					<a-icon type="redo"  @click="handleActions('clocelise')"/>
        </div>
      </div>
      <!-- CANVAS -->
      <div class="dmj-image__box">
          <!-- v-for="(url, i) in urlList"
          v-if="i === index" -->
        <img
          ref="img"
          class="dmj-image__img"
          :src="currentImg"
          :style="imgStyle"
          @load="handleImgLoad"
          @error="handleImgError"
          @mousedown="handleMouseDown">
      </div>
    </div>
  </transition>
</template> 

<script>
const on = (function() {
  if (document.addEventListener) {
    return function(element, event, handler) {
      if (element && event && handler) {
        element.addEventListener(event, handler, false);
      }
    };
  } else {
    return function(element, event, handler) {
      if (element && event && handler) {
        element.attachEvent('on' + event, handler);
      }
    };
  }
})();
const off = (function() {
  if (document.removeEventListener) {
    return function(element, event, handler) {
      if (element && event) {
        element.removeEventListener(event, handler, false);
      }
    };
  } else {
    return function(element, event, handler) {
      if (element && event) {
        element.detachEvent('on' + event, handler);
      }
    };
  }
})();
const isFirefox = function() {
  return !!window.navigator.userAgent.match(/firefox/i);
};
function rafThrottle(fn) {
  let locked = false;
  return function(...args) {
    if (locked) return;
    locked = true;
    window.requestAnimationFrame(_ => {
      fn.apply(this, args);
      locked = false;
    });
  };
}
let prevOverflow = '';


const mousewheelEventName = isFirefox() ? 'DOMMouseScroll' : 'mousewheel';

export default {
  name: 'imageView',
  props: {
    currentImg:{
    	type: String,
      default: ''
    },
    zIndex: {
      type: Number,
      default: 2000
    },
    onClose: {
      type: Function,
      default: () => {}
    },
    appendToBody: {
      type: Boolean,
      default: true
    },
    maskClosable: {
      type: Boolean,
      default: true
    }
  },

  data() {
    return {
      loading: false,
      transform: {
        scale: 1,
        deg: 0,
        offsetX: 0,
        offsetY: 0,
        enableTransition: false
      }
    };
  },
  computed: {
    imgStyle() {
      const { scale, deg, offsetX, offsetY, enableTransition } = this.transform;
      const style = {
        transform: `scale(${scale}) rotate(${deg}deg)`,
        transition: enableTransition ? 'transform .3s' : '',
        'margin-left': `${offsetX}px`,
        'margin-top': `${offsetY}px`
      };
      return style;
    }
  },
  watch: {
    currentImg(val) {
      this.$nextTick(_ => {
        const $img = this.$refs.img[0];
        if (!$img.complete) {
          this.loading = true;
        }
      });
    }
  },
  methods: {
    hide() {
      this.deviceSupportUninstall();
      this.onClose();
    },
    deviceSupportInstall() {
      this._keyDownHandler = rafThrottle(e => {
        const keyCode = e.keyCode;
        switch (keyCode) {
          // ESC
          case 27:
            this.hide();
            break;
          // LEFT_ARROW
          case 38:
            this.handleActions('zoomIn');
            break;
          // DOWN_ARROW
          case 40:
            this.handleActions('zoomOut');
            break;
        }
      });
      this._mouseWheelHandler = rafThrottle(e => {
        const delta = e.wheelDelta ? e.wheelDelta : -e.detail;
        if (delta > 0) {
          this.handleActions('zoomIn', {
            zoomRate: 0.015,
            enableTransition: false
          });
        } else {
          this.handleActions('zoomOut', {
            zoomRate: 0.015,
            enableTransition: false
          });
        }
      });
      on(document, 'keydown', this._keyDownHandler);
      on(document, mousewheelEventName, this._mouseWheelHandler);
    },
    deviceSupportUninstall() {
      off(document, 'keydown', this._keyDownHandler);
      off(document, mousewheelEventName, this._mouseWheelHandler);
      this._keyDownHandler = null;
      this._mouseWheelHandler = null;
    },
    handleImgLoad(e) {
      this.loading = false;
    },
    handleImgError(e) {
      this.loading = false;
      e.target.alt = '加载失败';
    },
    handleMouseDown(e) {
      if (this.loading || e.button !== 0) return;

      const { offsetX, offsetY } = this.transform;
      const startX = e.pageX;
      const startY = e.pageY;
      this._dragHandler = rafThrottle(ev => {
        this.transform.offsetX = offsetX + ev.pageX - startX;
        this.transform.offsetY = offsetY + ev.pageY - startY;
      });
      on(document, 'mousemove', this._dragHandler);
      on(document, 'mouseup', ev => {
        off(document, 'mousemove', this._dragHandler);
      });

      e.preventDefault();
    },
    handleMaskClick() {
      if (this.maskClosable) {
        this.hide();
      }
    },
    reset() {
      this.transform = {
        scale: 1,
        deg: 0,
        offsetX: 0,
        offsetY: 0,
        enableTransition: false
      };
    },
    handleActions(action, options = {}) {
      if (this.loading) return;
      const { zoomRate, rotateDeg, enableTransition } = {
        zoomRate: 0.2,
        rotateDeg: 90,
        enableTransition: true,
        ...options
      };
      const { transform } = this;
      switch (action) {
        case 'zoomOut':
          if (transform.scale > 0.2) {
            transform.scale = parseFloat((transform.scale - zoomRate).toFixed(3));
          }
          break;
        case 'zoomIn':
          transform.scale = parseFloat((transform.scale + zoomRate).toFixed(3));
          break;
        case 'clocelise':
          transform.deg += rotateDeg;
          break;
        case 'anticlocelise':
          transform.deg -= rotateDeg;
          break;
      }
      transform.enableTransition = enableTransition;
    }
  },
  mounted() {
		this.deviceSupportInstall();
		prevOverflow = document.body.style.overflow;
		document.body.style.overflow = 'hidden';
    if (this.appendToBody) {
      document.body.appendChild(this.$el);
    }
  },
  destroyed() {
		document.body.style.overflow = prevOverflow;
    if (this.appendToBody && this.$el && this.$el.parentNode) {
      this.$el.parentNode.removeChild(this.$el);
    }
  }
};
</script>

<style scoped>
.dmj-image__wrapper {
	position: fixed;
	top:0;
	right:0;
	bottom:0;
	left:0;
}
.dmj-image__box{
	width:100%;
	height:100%;
	display:flex;
	justify-content:center;
	align-items:center
}
.dmj-image__mask{
	position:absolute;
	width:100%;
	height:100%;
	top:0;
	left:0;
	opacity:.5;
	background:#000
}
.dmj-image__btn{
	position:absolute;
	z-index:1;
	display:flex;
	align-items:center;
	justify-content:center;
	border-radius:50%;
	opacity:.8;
	cursor:pointer;
	box-sizing:border-box;
	-webkit-user-select:none;
	-moz-user-select:none;
	-ms-user-select:none;
	user-select:none
}
.dmj-image__close{
	top:40px;
	right:40px;
	width:40px;
	height:40px;
	font-size:24px;
	color:#fff;
	background-color:#606266
}
.dmj-image__handel{
	left:50%;
	bottom:30px;
	transform:translateX(-50%);
	width:282px;
	height:44px;
	padding:0 23px;
	background-color:#606266;
	border-color:#fff;
	border-radius:22px
}
.dmj-image__handel__inner{
	width:100%;
	height:100%;
	text-align:justify;
	cursor:default;
	font-size:23px;
	color:#fff;
	display:flex;
	align-items:center;
	justify-content:space-around
}
.viewer-fade-enter-active{
	animation:viewer-fade-in .3s;
}
.viewer-fade-leave-active{
	animation:viewer-fade-out .3s
}
@keyframes viewer-fade-in{
	0%{
		transform:translate3d(0,-20px,0);
		opacity:0
	}
	100%{
		transform:translate3d(0,0,0);
		opacity:1
	}
}
@keyframes viewer-fade-out{
	0%{
		transform:translate3d(0,0,0);
		opacity:1
	}100%{
		transform:translate3d(0,-20px,0);
		opacity:0
	}
}

</style>