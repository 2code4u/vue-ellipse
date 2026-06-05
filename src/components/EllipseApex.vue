<template>
  <div class="canvas-lines" ref="mainBlock">
    <canvas @mousedown="mouseDown" @mouseup="mouseUp" @mousemove="mouseMove" class="main-frame" ref="canvasRef">
    </canvas>

    <div class="telemetry">
      <div class="telemetry-group">
        <span class="telemetry-label">Мышь</span>
        <span>
          X:
          <span id="valMouseX" class="telemetry-value"> {{ currentMouseX }}</span>px |
          Y:
          <span id="valMouseY" class="telemetry-value">{{ currentMouseY }}</span>px
        </span>
      </div>

      <div class="telemetry-group">
        <span class="telemetry-label">Границы рамки X</span>
        <span>
          Слева:
          <span id="valLimitLeft" class="telemetry-value warn">
            {{ cx - limitX }}
          </span>px |
          Справа:
          <span id="valLimitRight" class="telemetry-value warn">
            {{ cx + limitX }}
          </span>px
        </span>
      </div>

      <div class="telemetry-group">
        <span class="telemetry-label">Границы рамки Y</span>
        <span>
          Сверху:
          <span id="valLimitTop" class="telemetry-value warn">
            {{ cy - limitY }}
          </span>px |
          Снизу:
          <span id="valLimitBottom" class="telemetry-value warn">
            {{ cy + limitY }}
          </span>px
        </span>
      </div>
      <div class="telemetry-group">
        <span class="telemetry-label">Радиусы эллипса</span>
        <span>RX: <span id="valRx" class="telemetry-value accent">
            {{ Math.round(rx) }}
          </span>px |
          RY:
          <span id="valRy" class="telemetry-value accent">
            {{ Math.round(ry) }}
          </span>px
        </span>
      </div>
    </div>
  </div>
</template>

<script lang="ts">
import { defineComponent } from 'vue'

export default defineComponent({
  name: 'EllipseApex',
  components: {
  },

  data: () => ({
    rectCanvas: {} as DOMRect,
    contextCanvas: {} as CanvasRenderingContext2D,
    isDragging: false as boolean,
    apexOffsetX: 0 as number, // Величина смещения верхней вершины по оси X
    apexOffsetY: 0 as number, // Величина смещения верхней вершины по оси Y
    limitX: 340, // размер описанного прямоугольника x
    limitY: 240, // размер описанного прямоугольника y
    currentMouseX: 0,
    currentMouseY: 0,
    cx: 400,     // Центр X
    cy: 300,     // Центр Y
    rx: 240,     // горизонтальный радиус
    ry: 120,     // вертикальный радиус
  }),

  computed: {
    aspectRatio(): number { // Коэффициент пропорциональности (во сколько раз ry меньше rx)
      return this.ry / this.rx
    }
  },

  mounted() {
    const block: HTMLElement = this.$refs.mainBlock as HTMLElement
    const canvas: HTMLCanvasElement = this.$refs.canvasRef as HTMLCanvasElement
    canvas.width = block.clientWidth
    canvas.height = block.clientHeight

    this.loadElements()

    let rect = canvas.getBoundingClientRect();
    let ctx = canvas.getContext('2d')
    if (!ctx || !rect) return

    this.contextCanvas = ctx
    this.rectCanvas = rect
    this.renderDraw(ctx)
  },

  methods: {
    loadElements() {
      // Параметры деформируемого эллипса
      this.apexOffsetX = this.rx
    },

    getMousePos(event: MouseEvent) {
      const rect = this.rectCanvas
      this.currentMouseX = event.clientX - rect.left
      this.currentMouseY = event.clientY - rect.top
    },

    rescaleEllipse(angle: number) {
      const cosA = Math.cos(angle)
      const sinA = Math.sin(angle)

      const factorX = Math.sqrt(cosA * cosA + this.aspectRatio * this.aspectRatio * sinA * sinA)
      const rxMaxX = this.limitX / factorX
      const factorY = Math.sqrt(sinA * sinA + this.aspectRatio * this.aspectRatio * cosA * cosA)
      const rxMaxY = this.limitY / factorY

      // Итоговый рабочий радиус — это строгий минимум между двумя максимумами.
      // Так эллипс гарантированно упрется в одну из сторон, заполнив максимум пространства.
      this.rx = Math.min(rxMaxX, rxMaxY)
      this.ry = this.rx * this.aspectRatio

      // Пересчитываем точные координаты апекса на основе вычисленного максимального радиуса
      this.apexOffsetX = this.rx * cosA;
      this.apexOffsetY = this.rx * sinA;
    },

    renderDraw(canvasCtx: CanvasRenderingContext2D) {
      canvasCtx.clearRect(-1000, -1000, 3000, 3000)

      // Отрисовка ограничительного прямоугольника
      canvasCtx.lineWidth = 1
      canvasCtx.strokeStyle = '#e74c3c'
      canvasCtx.setLineDash([6, 4])
      canvasCtx.strokeRect(this.cx - this.limitX, this.cy - this.limitY, this.limitX * 2, this.limitY * 2)
      canvasCtx.setLineDash([])

      // Вычисляем текущий угол поворота эллипса в радианах
      const angle = Math.atan2(this.apexOffsetY, this.apexOffsetX)

      // Сжимаем эллипс при приближение апекса к углу
      this.rescaleEllipse(angle)

      // Координаты вершины для отрисовки маркера
      const topX = this.cx + this.apexOffsetX
      const topY = this.cy + this.apexOffsetY

      // Отрисовка геометрически правильного эллипса
      canvasCtx.beginPath();
      canvasCtx.ellipse(this.cx, this.cy, this.rx, this.ry, angle, 0, Math.PI * 2);

      canvasCtx.lineWidth = 4;
      canvasCtx.strokeStyle = '#3498db';
      canvasCtx.stroke();
      canvasCtx.fillStyle = 'rgba(52, 152, 219, 0.1)';
      canvasCtx.fill();

      // Отрисовка точки (новой правой вершины)
      canvasCtx.beginPath()
      canvasCtx.arc(topX, topY, 8, 0, Math.PI * 2)
      canvasCtx.fillStyle = this.isDragging ? '#e67e22' : '#e74c3c'
      canvasCtx.fill()

      // Опорные оси центра
      canvasCtx.lineWidth = 1;
      canvasCtx.strokeStyle = 'rgba(0,0,0,0.1)';
      canvasCtx.beginPath(); canvasCtx.moveTo(this.cx - 200, this.cy); canvasCtx.lineTo(this.cx + 200, this.cy); canvasCtx.stroke();
      canvasCtx.beginPath(); canvasCtx.moveTo(this.cx, this.cy - 200); canvasCtx.lineTo(this.cx, this.cy + 200); canvasCtx.stroke();

    },

    mouseIn(startX: number, startY: number) {
      const clickRadius = 15; // Радиус клика для захвата точки (в пикселях)
      const topX = this.cx + this.apexOffsetX;
      const topY = this.cy + this.apexOffsetY;
      const dist = Math.hypot(startX - topX, startY - topY)
      return dist <= clickRadius
    },

    mouseDown(event: MouseEvent) {
      this.getMousePos(event)

      if (this.mouseIn(this.currentMouseX, this.currentMouseY)) {
        this.isDragging = true
      }
    },

    mouseUp() {
      this.isDragging = false
    },

    mouseMove(event: MouseEvent) {
      event.preventDefault()
      this.getMousePos(event)
      if (!this.isDragging) {
        return
      }

      if (!this.contextCanvas) return
      // Записываем чистый вектор направления мыши от центра холста
      this.apexOffsetX = this.currentMouseX - this.cx
      this.apexOffsetY = this.currentMouseY - this.cy

      // Дополнительная защита: не даем радиусу стать нулевым, чтобы эллипс не исчез
      if (Math.hypot(this.apexOffsetX, this.apexOffsetY) < 30) {
        this.apexOffsetX = 30;
        this.apexOffsetY = 0;
      }
      this.renderDraw(this.contextCanvas)
    },
  }
})
</script>

<style>
.canvas-lines {
  margin: 12px;
  height: 75vh;
  padding: 20px 0 100px 0;
}

.canvas-lines__button {
  width: 60%;
  height: 45px;
  background-color: mediumaquamarine;
  font-size: 24px;
  margin-top: 40px;
}

.main-frame {
  display: block;
  border: 1px solid #ccc;
  cursor: default;
  box-shadow: 0 4px 10px rgba(0, 0, 0, 0.05);
}

.telemetry {
  position: fixed;
  bottom: 0;
  left: 0;
  right: 0;
  background: rgba(44, 62, 80, 0.95);
  color: #ecf0f1;
  padding: 15px 30px;
  font-family: monospace;
  font-size: 14px;
  display: flex;
  justify-content: space-around;
  align-items: center;
  box-shadow: 0 -4px 15px rgba(0, 0, 0, 0.2);
  z-index: 1000;
}

.telemetry-group {
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.telemetry-label {
  color: #95a5a6;
  font-size: 11px;
  text-transform: uppercase;
}

.telemetry-value {
  color: #2ecc71;
  font-weight: bold;
}

.telemetry-value.accent {
  color: #3498db;
}

.telemetry-value.warn {
  color: #e74c3c;
}
</style>
