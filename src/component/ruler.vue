<template>
  <div :class="`ruler-container ruler-${type}`">
    <canvas :id="id" class="ruler" width="100%" height="100%" />
    <div class="lines" v-if="isHorizontal">
      <div
        v-for="(o, k) in lines"
        :key="k"
        class="line"
        :id="k"
        :style="{ left: o.position + 'px', top: 0 }"
        @mousedown="(e) => handleMousedown(k, e)"
      >
        <div class="value">{{ o.value }}</div>
      </div>
    </div>
    <div class="lines" v-if="!isHorizontal">
      <div
        v-for="(o, k) in lines"
        :key="k"
        class="line"
        :id="k"
        :style="{ left: 0, top: o.position + 'px' }"
        @mousedown="(e) => handleMousedown(k, e)"
      >
        <div class="value">{{ o.value }}</div>
      </div>
    </div>
    <div class="auxiliary-line" :style="linePosition">
      <div class="value">{{ currentValue }}</div>
    </div>
  </div>
</template>

<script>
import * as _ from "lodash";
import * as defaultParams from "./config.ts";

export default {
  name: 'ruler',
  props: {
    id: {
      type: String,
      default: "h",
    },
    params: {
      type: Object,
      default: () => {},
    },
    position: {
      // 起始点位置
      type: Number,
      default: 0,
    },
    type: {
      // 水平还是垂直
      type: String,
      default: "horizontal",
    },
    size: {
      // 画布有多大
      type: Number,
      default: 5000,
    },
    scale: {
      // 缩放比例
      type: Number,
      default: 1,
    },
    ratio: {
      type: Number,
      default: window.devicePixelRatio,
    },
    offset: {
      type: Number,
      default: 0,
    },
  },
  data() {
    return {
      lines: {},
      moveKey: "",
      current: null,
      width: 0,
      height: 0,
      settings: {
        step: 10,
        stepValue: 10,
      },
    };
  },
  computed: {
    isHorizontal() {
      return this.type === "horizontal";
    },
    length() {
      return this.isHorizontal ? this.width : this.height;
    },
    breadth() {
      return this.isHorizontal ? this.height : this.width;
    },
    total() {
      // 总的值
      return this.size / this.scale;
    },
    zero() {
      // 0的位置
      return this.size / 2 + this.offset;
    },
    min() {
      return _.round(-this.zero / this.scale, 2);
    },
    startValue() {
      const origin = (this.position + 20 - this.zero) / this.scale; // 开始位置的数值
      return _.round(origin, 2);
    },
    defaultConfig() {
      const config = this.isHorizontal
        ? { ...this.params, ...defaultParams.X }
        : { ...this.params, ...defaultParams.Y };
      return config;
    },
    defaultStep() {
      return this.defaultConfig.step;
    },
    defaultStepValue() {
      return this.defaultConfig.stepValue;
    },
    step() {
      // 画图时 一小格对应的px
      let step = this.defaultStep * this.scale;
      return this.getStep(step);
    },
    stepValue() {
      const rate = this.defaultStepValue / this.defaultStep;
      return Math.ceil((this.step * rate) / this.scale);
    },
    linePosition() {
      return {
        [this.isHorizontal ? "left" : "top"]: (this.current ?? 0) + "px",
      };
    },
    currentValue() {
      return this.getValue(this.current);
    },
    range() {
      return this.defaultConfig.range; // 需要显示的数值的间隔
    },
    list() {
      // 按数值间隔显示的方式
      const list = [];
      const range = this.range; // 需要显示的数值的间隔
      const origin = this.startValue; // 开始位置的数值
      const max = this.length / this.scale + origin;
      const start = Math.ceil(origin / range) * range;
      for (let i = start; i <= max; i += range) {
        list.push(i);
      }
      return list;
    },
  },
  watch: {
    position: function(c) {
      this.drawRuler(Math.floor(c));
    },
    offset: function() {
      this.init();
    },
  },
  mounted() {
    this.$nextTick(() => {
      this.init();
    });
  },
  methods: {
    init() {
      const canvas = document.getElementById(this.id);
      this.width = canvas.clientWidth;
      this.height = canvas.clientHeight;
      canvas.width = this.width;
      canvas.height = this.height;
      this.drawRuler(Math.floor(this.position));
      canvas.addEventListener(
        "mousemove",
        (e) => {
          this.current = this.isHorizontal ? e.layerX : e.layerY;
        },
        false
      );
      canvas.addEventListener("click", this.handleClick, false);
    },
    drawRuler(position) {
      // 起始的物理位置
      const canvas = document.getElementById(this.id);
      const cxt = canvas.getContext("2d");
      const step = this.step;
      const stepValue = this.stepValue;
      const startValue = this.startValue; // 开始的值
      const length = this.length; // 尺子的长度
      const breadth = this.breadth; // 尺子的宽度
      const list = [...this.list];
      const params = this.defaultConfig;
      const isHorizontal = this.isHorizontal;
      const min = this.min;
      const scale = this.scale;
      const { rangeStep, valueType, lineHeight, background } = params;
      const origin = Math.floor(startValue / stepValue);
      const startValueInt = Math.floor(startValue / stepValue) * stepValue;
      const restPosition = (startValue - startValueInt) * scale; // 补偿
      const lines = { ...this.lines };
      for (let k of Object.keys(lines)) {
        const line = lines[k];
        line.position = (line.value - startValue) * scale;
      }
      this.lines = lines;
      // 清空画布
      cxt.clearRect(0, 0, this.width, this.height);
      // 背景
      if (background) {
        cxt.fillStyle = background;
        cxt.fillRect(0, 0, this.width, this.height);
      }

      const t_0 = Math.floor(breadth * lineHeight[0]);
      const t_1 = Math.floor(breadth * lineHeight[1]);
      const t_2 = Math.floor(breadth * lineHeight[2]);
      const t_3 = Math.floor(breadth * lineHeight[3]);

      if (isHorizontal) {
        // 画刻度线
        for (let i = 1; i < length / step + 1; i++) {
          cxt.beginPath();
          cxt.save();
          cxt.strokeStyle = params.lineColor[0];
          cxt.lineWidth = params.lineWidth[0];
          cxt.lineCap = "round";
          const x = step * i - restPosition;
          cxt.moveTo(x, 0);
          cxt.lineTo(x, t_0);

          if ((i + origin) % 5 === 0) {
            cxt.strokeStyle = params.lineColor[1];
            cxt.lineWidth = params.lineWidth[1];
            cxt.lineTo(x, t_1);
          }
          if ((i + origin) % 10 === 0) {
            cxt.strokeStyle = params.lineColor[2];
            cxt.lineWidth = params.lineWidth[2];
            cxt.lineTo(x, t_2);
          }

          if (valueType !== "range" && (i + origin) % rangeStep === 0) {
            cxt.font = params.font[0];
            cxt.fillStyle = params.textColor[0];
            cxt.textAlign = "left";
            cxt.textBaseline = "middle";

            const num = startValueInt + i * stepValue;
            if (list.includes(num)) {
              list.splice(list.indexOf(num), 1);
            }
            cxt.fillText(num, x, t_3);
          }
          cxt.stroke();
          cxt.restore();
          cxt.closePath();
        }

        // 添加数字
        if (valueType !== "step") {
          cxt.beginPath();
          cxt.font = params.font[0];
          cxt.fillStyle = params.textColor[0];
          cxt.textAlign = "left";
          cxt.textBaseline = "middle";

          list.forEach(function(num) {
            cxt.fillText(num.toString(), (num - min) * scale - position, t_3);
          });
          cxt.restore();
          cxt.closePath();
        }
      } else {
        // 画刻度线
        for (let i = 1; i < length / step + 1; i++) {
          cxt.beginPath();
          cxt.save();
          cxt.strokeStyle = params.lineColor[0];
          cxt.lineWidth = params.lineWidth[0];
          cxt.lineCap = "round";
          const point_s = step * i - restPosition;
          cxt.moveTo(0, point_s);
          cxt.lineTo(t_0, point_s);

          if ((i + origin) % 5 === 0) {
            cxt.strokeStyle = params.lineColor[1];
            cxt.lineWidth = params.lineWidth[1];
            cxt.lineTo(t_1, point_s);
          }
          if ((i + origin) % 10 === 0) {
            cxt.strokeStyle = params.lineColor[2];
            cxt.lineWidth = params.lineWidth[2];
            cxt.lineTo(t_2, point_s);
          }

          cxt.stroke();
          cxt.restore();
          cxt.closePath();

          if (valueType !== "range" && (i + origin) % rangeStep === 0) {
            cxt.font = params.font[0];
            cxt.fillStyle = params.textColor[0];
            cxt.textAlign = "right";
            cxt.textBaseline = "middle";

            const num = startValueInt + i * stepValue;
            cxt.beginPath();
            cxt.translate(0, 0);
            cxt.rotate(-Math.PI / 2);
            cxt.fillText(num.toString(), -point_s, t_3);
            cxt.rotate(Math.PI / 2);
            cxt.translate(0, 0);

            if (list.includes(num)) {
              list.splice(list.indexOf(num), 1);
            }
          }
        }

        // 添加数字
        if (valueType !== "step") {
          cxt.beginPath();
          cxt.font = params.font[0];
          cxt.fillStyle = params.textColor[0];
          cxt.textAlign = "right";
          cxt.textBaseline = "middle";
          cxt.translate(0, 0);
          cxt.rotate(-Math.PI / 2);
          list.reverse().forEach(function(num) {
            cxt.fillText(num.toString(), -(num - min) * scale + position, t_3);
          });
          cxt.rotate(Math.PI / 2);
          cxt.translate(0, 0);
        }
      }
    },
    handleClick(e) {
      const current = this.isHorizontal ? e.layerX : e.layerY;
      const lines = { ...this.lines };
      const value = this.getValue(current);
      const keys = Object.keys(lines);
      if (!keys.includes(value)) {
        lines[value] = {
          value: value,
          position: current,
        };
        this.lines = lines;
      }
    },
    handleMousedown(key, e) {
      e.stopPropagation();
      this.moveKey = key;
      document.addEventListener("mousemove", this.handleMouseMove);
      document.addEventListener("mouseup", this.handleMouseUp);
    },
    handleMouseUp() {
      document.removeEventListener("mousemove", this.handleMouseMove);
      const lines = this.lines;
      const line = lines[this.moveKey];
      if (line && (line.position > this.length || line.position < 0)) {
        Reflect.deleteProperty(lines, this.moveKey);
        this.lines = lines;
      }
    },
    handleMouseMove(e) {
      const line = this.lines[this.moveKey];
      const current = this.isHorizontal ? e.x - 246 : e.y - 70;
      line.position = current;
      line.value = this.getValue(current);
      this.lines = { ...this.lines, [this.moveKey]: line };
      window.getSelection
        ? window.getSelection().removeAllRanges()
        : document.selection.empty();
    },
    getValue(c) {
      const value = c
        ? (c + this.position + 20) / this.scale + this.min
        : this.startValue;
      return Math.ceil(value);
    },
    getStep(step) {
      if (this.scale === 1) {
        return step;
      }
      if (this.scale < 1) {
        if (step < 6) {
          return Math.round(this.getStep(step * 2));
        } else {
          return Math.round(step);
        }
      } else if (this.scale <= 2) {
        return Math.round(5 * this.scale);
      } else {
        return Math.round(2 * this.scale);
      }
    },
  },
};
</script>

<style scoped>
@import '/lib/common.css';
.ruler-hide {
  display: none !important;
}
.ruler-container {
  position: absolute;
  user-select: none;
  background: #f2f2f2;
}
.ruler-container:hover .lines .line {
  pointer-events: auto;
  cursor: ew-resize;
}
.ruler-container .ruler {
  width: 100%;
  height: 100%;
  pointer-events: auto;
}
.ruler-container .auxiliary-line {
  color: #0086FF;
  display: none;
  position: absolute;
  pointer-events: none;
  z-index: 999;
}
.ruler-container .auxiliary-line .value {
  position: absolute;
}
.ruler-container .lines {
  pointer-events: none;
}
.ruler-container .lines .line {
  position: absolute;
  color: lightgreen;
}
.ruler-container .lines .line .value {
  position: absolute;
}
.ruler-container .lines:hover + .auxiliary-line {
  display: none;
}
.ruler-horizontal {
  width: calc(100% - 20px);
  height: 21px;
  left: 20px;
  top: 0;
}
.ruler-horizontal:hover .auxiliary-line {
  display: block;
  top: 0;
  height: 100vh;
  border-left: 1px dashed #0086FF;
}
.ruler-horizontal .lines .line {
  height: 100vh;
  border-left: 1px solid lightgreen;
  padding-left: 5px;
}
.ruler-horizontal .lines .line .value {
  top: 20px;
}
.ruler-vertical {
  width: 21px;
  left: 0;
  top: 20px;
  height: calc(100% - 20px);
}
.ruler-vertical:hover .auxiliary-line {
  display: block;
  left: 0;
  width: 100vw;
  height: 1px;
  border-top: 1px dashed #0086FF;
}
.ruler-vertical:hover .auxiliary-line .value {
  left: 0;
  bottom: 0;
}
.ruler-vertical:hover .lines .line {
  cursor: ns-resize;
}
.ruler-vertical .lines .line {
  position: absolute;
  width: 100vw;
  border-top: 1px solid lightgreen;
  top: 30px;
  left: 0;
  height: 0;
  padding-top: 5px;
}

</style>
