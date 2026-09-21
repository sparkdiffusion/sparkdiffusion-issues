<script setup lang="ts">
import * as echarts from 'echarts'
import { onBeforeUnmount, onMounted, ref } from 'vue'
import type { EChartsOption } from 'echarts'

const host = ref<HTMLDivElement | null>(null)
let chart: echarts.ECharts | null = null
let observer: ResizeObserver | null = null

type Graphic = Record<string, unknown>
type Palette = ReturnType<typeof readPalette>

const BASE_HEIGHT = 470
const GAP = 10

function readPalette() {
  const styles = getComputedStyle(document.documentElement)
  return {
    ink: styles.getPropertyValue('--ink').trim() || '#e8ecff',
    muted: styles.getPropertyValue('--muted').trim() || 'rgba(232,236,255,.64)',
    quiet: styles.getPropertyValue('--quiet').trim() || 'rgba(232,236,255,.38)',
    rule: styles.getPropertyValue('--rule').trim() || 'rgba(232,236,255,.16)',
    signal: styles.getPropertyValue('--signal').trim() || '#4d8bff',
    violet: styles.getPropertyValue('--violet').trim() || '#9d7bff',
    panel: 'rgba(8, 14, 31, .72)',
    panelSoft: 'rgba(232, 236, 255, .025)',
    signalSoft: 'rgba(77, 139, 255, .10)',
    violetSoft: 'rgba(157, 123, 255, .10)',
  }
}

function rect(x: number, y: number, width: number, height: number, fill: string, stroke: string, radius = 7, lineWidth = 1): Graphic {
  return { type: 'rect', silent: true, shape: { x, y, width, height, r: radius }, style: { fill, stroke, lineWidth } }
}

function text(x: number, y: number, value: string, fill: string, font = '500 12px "DM Sans"', align: 'left' | 'center' | 'right' = 'left', lineHeight = 16): Graphic {
  return { type: 'text', silent: true, style: { x, y, text: value, fill, font, textAlign: align, textVerticalAlign: 'middle', lineHeight } }
}

function line(x1: number, y1: number, x2: number, y2: number, stroke: string, lineWidth = 1, dashed = false): Graphic {
  return { type: 'line', silent: true, shape: { x1, y1, x2, y2 }, style: { stroke, lineWidth, lineDash: dashed ? [5, 5] : undefined } }
}

function polyline(points: number[][], stroke: string, lineWidth = 1, dashed = false): Graphic {
  return { type: 'polyline', silent: true, shape: { points }, style: { fill: 'none', stroke, lineWidth, lineDash: dashed ? [5, 5] : undefined } }
}

function circle(cx: number, cy: number, r: number, fill: string, stroke: string, lineWidth = 1): Graphic {
  return { type: 'circle', silent: true, shape: { cx, cy, r }, style: { fill, stroke, lineWidth } }
}

function polygon(points: number[][], fill: string, stroke = fill): Graphic {
  return { type: 'polygon', silent: true, shape: { points }, style: { fill, stroke } }
}

function stageIcon(items: Graphic[], cx: number, cy: number, kind: string, p: Palette) {
  if (kind === '❄') {
    for (let i = 0; i < 3; i += 1) {
      const angle = i * Math.PI / 3
      const dx = Math.cos(angle) * 11; const dy = Math.sin(angle) * 11
      items.push(line(cx - dx, cy - dy, cx + dx, cy + dy, p.signal, 2))
      ;[-1, 1].forEach((side) => { const ex = cx + dx * side; const ey = cy + dy * side; const bx = cx + dx * side * .56; const by = cy + dy * side * .56; items.push(line(ex, ey, bx - dy * side * .22, by + dx * side * .22, p.signal, 1.3)); items.push(line(ex, ey, bx + dy * side * .22, by - dx * side * .22, p.signal, 1.3)) })
    }
  } else {
    items.push(polygon([[cx, cy + 13], [cx - 10, cy + 7], [cx - 9, cy - 1], [cx - 3, cy - 12], [cx - 2, cy - 2], [cx + 5, cy - 13], [cx + 7, cy - 1], [cx + 12, cy + 7], [cx + 6, cy + 13]], p.violet))
    items.push(polygon([[cx + 1, cy + 9], [cx - 4, cy + 6], [cx - 3, cy + 0], [cx + 1, cy - 5], [cx + 3, cy + 1], [cx + 7, cy + 7]], p.signal))
  }
}

function arrow(items: Graphic[], x1: number, y1: number, x2: number, y2: number, color: string, width = 1.5) {
  items.push(line(x1, y1, x2, y2, color, width))
  const angle = Math.atan2(y2 - y1, x2 - x1)
  const size = 6
  items.push(polygon([[x2, y2], [x2 - size * Math.cos(angle - Math.PI / 6), y2 - size * Math.sin(angle - Math.PI / 6)], [x2 - size * Math.cos(angle + Math.PI / 6), y2 - size * Math.sin(angle + Math.PI / 6)]], color))
}

function stageFrame(items: Graphic[], x: number, y: number, w: number, h: number, index: string, title: string, tag: string, p: Palette) {
  items.push(rect(x, y, w, h, p.panel, p.rule, 12))
  items.push(circle(x + 22, y + 23, 15, p.signal, p.signal))
  items.push(text(x + 22, y + 23, index, '#08101f', '500 14px "DM Mono"', 'center'))
  items.push(text(x + 45, y + 23, title, p.ink, w < 255 ? '500 11px "DM Sans"' : '500 13px "DM Sans"'))
  stageIcon(items, x + w - 26, y + 23, tag, p)
  items.push(line(x + 12, y + 46, x + w - 12, y + 46, p.rule))
}

function drawStage0(items: Graphic[], x: number, y: number, w: number, h: number, p: Palette) {
  stageFrame(items, x, y, w, h, '0', 'Pretrained Dense Video DiT', '❄', p)
  const cx = x + w / 2
  const stackW = Math.min(100, w * .46)
  const stackH = 70
  for (let i = 3; i >= 0; i -= 1) items.push(rect(cx - stackW / 2 + i * 7, y + 91 - i * 8, stackW, stackH, i === 0 ? p.panel : p.panelSoft, p.signal, 6))
  items.push(polygon([[cx - 8, y + 115], [cx - 8, y + 143], [cx + 16, y + 129]], p.signal))
  items.push(text(cx + stackW / 2 + 24, y + 126, '•••', p.quiet, '500 16px "DM Mono"', 'center'))
  const tokenY = y + 228
  const tokenXs = [x + 30, x + w * .32, x + w * .68, x + w - 30]
  tokenXs.forEach((tx) => { items.push(rect(tx - 7, tokenY - 7, 14, 14, p.signalSoft, p.signal, 2)); items.push(line(tx, tokenY + 8, tx, y + 225, p.rule)) })
  items.push(text(cx, tokenY, '···', p.quiet, '500 15px "DM Mono"', 'center'))
  items.push(rect(x + 18, y + 262, w - 36, 94, p.panelSoft, p.signal, 9))
  items.push(text(cx, y + 295, 'Transformer', p.ink, '500 15px "DM Sans"', 'center'))
  items.push(text(cx, y + 320, 'blocks', p.ink, '500 15px "DM Sans"', 'center'))
  tokenXs.forEach((tx) => { items.push(line(tx, y + 356, tx, y + 392, p.rule)); items.push(rect(tx - 7, y + 392, 14, 14, p.violetSoft, p.violet, 2)) })
  items.push(text(cx, y + 399, '···', p.quiet, '500 15px "DM Mono"', 'center'))
}

function drawGrid(items: Graphic[], x: number, y: number, size: number, p: Palette) {
  const cell = size / 5
  for (let row = 0; row < 5; row += 1) for (let col = 0; col < 5; col += 1) {
    const active = col === row || (row === 0 && col === 4) || (row === 4 && col === 1)
    items.push(rect(x + col * cell, y + row * cell, cell - 1, cell - 1, active ? p.signal : p.signalSoft, p.rule, 1))
  }
}

function drawNetwork(items: Graphic[], cx: number, cy: number, radius: number, p: Palette) {
  items.push(circle(cx, cy, radius + 10, 'transparent', p.violet, 1.2))
  const points = [[cx, cy - radius], [cx - radius * .75, cy - radius * .35], [cx + radius * .75, cy - radius * .35], [cx, cy], [cx - radius * .75, cy + radius * .45], [cx + radius * .75, cy + radius * .45], [cx, cy + radius]]
  const edges = [[0,1],[0,2],[1,3],[2,3],[1,4],[2,5],[3,4],[3,5],[4,6],[5,6]]
  edges.forEach(([a, b]) => items.push(line(points[a][0], points[a][1], points[b][0], points[b][1], p.violet, 1)))
  points.forEach(([px, py], i) => items.push(circle(px, py, i === 3 ? 6 : 5, p.panel, i === 3 ? p.signal : p.violet, 1.5)))
}

function drawStage1(items: Graphic[], x: number, y: number, w: number, h: number, p: Palette) {
  stageFrame(items, x, y, w, h, '1', 'Sparse Warm-up', '♨', p)
  const half = w / 2
  items.push(text(x + half * .52, y + 68, 'SPARSE BRANCH', p.muted, '500 9px "DM Mono"', 'center'))
  items.push(text(x + half * 1.5, y + 68, 'COMPENSATION', p.muted, '500 9px "DM Mono"', 'center'))
  const size = Math.min(82, w * .34)
  drawGrid(items, x + half * .52 - size / 2, y + 91, size, p)
  drawNetwork(items, x + half * 1.5, y + 132, Math.min(43, w * .18), p)
  arrow(items, x + half * .52, y + 179, x + half * .82, y + 212, p.quiet)
  arrow(items, x + half * 1.5, y + 179, x + half * 1.18, y + 212, p.quiet)
  items.push(rect(x + w * .28, y + 211, w * .44, 42, p.signalSoft, p.signal, 8))
  items.push(text(x + w / 2, y + 226, 'GATE', p.ink, '500 14px "DM Sans"', 'center'))
  items.push(text(x + w / 2, y + 243, 'near-zero init', p.quiet, '400 8px "DM Mono"', 'center'))
  arrow(items, x + w / 2, y + 255, x + w / 2, y + 278, p.signal)
  items.push(text(x + w / 2, y + 296, 'COARSE MANIFOLD PRIOR', p.muted, '500 9px "DM Mono"', 'center'))
  const bx = x + 20; const by = y + 314; const bw = w - 40; const bh = 92
  for (let row = 0; row < 6; row += 1) for (let col = 0; col < 10; col += 1) {
    const px = (col + .5) / 10; const py = (row + .5) / 6
    const dx = (px - .5) / .52; const dy = (py - .5) / .74
    if (dx * dx + dy * dy < 1) items.push(circle(bx + px * bw, by + py * bh, 3, p.violetSoft, 'transparent'))
  }
  const pts = [[.07,.7],[.19,.43],[.31,.61],[.43,.3],[.55,.55],[.69,.42],[.82,.68],[.94,.48]].map(([px, py]) => [bx + px * bw, by + py * bh])
  items.push(polyline(pts, p.signal, 2)); pts.forEach(([px, py], i) => items.push(circle(px, py, i % 2 ? 4 : 5, i % 2 ? p.violet : p.signal, p.panel, 1)))
  items.push(text(x + w / 2, y + h - 41, 'MULTI-STEP SPARSE', p.signal, '500 11px "DM Mono"', 'center'))
}

function drawMiniTree(items: Graphic[], cx: number, cy: number, p: Palette, sparse = false) {
  const nodes = sparse ? [[0,-31],[-30,-7],[30,-7],[0,10],[-28,34],[28,34]] : [[0,-31],[-34,-9],[34,-9],[0,10],[-34,35],[34,35]]
  const edges = sparse ? [[0,3],[1,3],[2,3],[3,4]] : [[0,3],[1,3],[2,3],[3,4],[3,5]]
  edges.forEach(([a,b]) => items.push(line(cx + nodes[a][0], cy + nodes[a][1], cx + nodes[b][0], cy + nodes[b][1], sparse ? p.signal : p.violet, 1.2)))
  nodes.forEach(([dx,dy], i) => { if (!sparse || i !== 5) items.push(circle(cx + dx, cy + dy, 5, p.panel, sparse ? p.signal : p.violet, 1.4)) })
}

function drawStage2(items: Graphic[], x: number, y: number, w: number, h: number, p: Palette) {
  stageFrame(items, x, y, w, h, '2', 'Trajectory-Mixed Distillation', '♨', p)
  const left = x + w * .27; const right = x + w * .73
  items.push(text(left, y + 65, 'TEACHER · DENSE', p.muted, '500 9px "DM Mono"', 'center')); items.push(text(right, y + 65, 'STUDENT · SPARSE', p.muted, '500 9px "DM Mono"', 'center'))
  drawMiniTree(items, left, y + 117, p); drawMiniTree(items, right, y + 117, p, true)
  const barX = x + 16; const barY = y + 166; const barW = w - 32
  items.push(text(barX, barY - 13, 't = 1', p.violet, '500 10px "DM Mono"')); items.push(text(barX + barW, barY - 13, 't = 0', p.signal, '500 10px "DM Mono"', 'right'))
  items.push(rect(barX, barY, barW, 22, new echarts.graphic.LinearGradient(0, 0, 1, 0, [{ offset: 0, color: p.violet }, { offset: .52, color: 'rgba(122,118,255,.72)' }, { offset: 1, color: p.signal }]) as unknown as string, 'transparent', 5))
  const crossX = barX + barW * .52; items.push(line(crossX, barY - 6, crossX, barY + 30, p.ink, 1, true)); items.push(text(crossX, barY + 39, 't crossover', p.muted, '400 9px "DM Mono"', 'center'))
  items.push(line(x + w / 2, y + 217, x + w / 2, y + h - 24, p.rule, 1, true)); items.push(line(x + 10, y + 217, x + w - 10, y + 217, p.rule, 1, true))
  const sectionY = y + 239
  items.push(text(x + w * .25, sectionY, 'A · CONSISTENCY', p.violet, '500 9px "DM Mono"', 'center')); items.push(text(x + w * .75, sectionY, 'B · DISTRIBUTION', p.signal, '500 9px "DM Mono"', 'center'))
  items.push(text(x + w * .25, sectionY + 22, 'HIGH NOISE', p.ink, '500 11px "DM Sans"', 'center')); items.push(text(x + w * .75, sectionY + 22, 'LOW NOISE', p.ink, '500 11px "DM Sans"', 'center'))
  const lx = x + 18; const lw = w / 2 - 30; const ly = sectionY + 55
  const teacherPts = [[.05,.56],[.24,.3],[.43,.54],[.62,.2],[.82,.43],[.96,.28]].map(([px,py]) => [lx + px * lw, ly + py * 84]); const studentPts = [[.05,.75],[.27,.58],[.5,.72],[.72,.54],[.96,.66]].map(([px,py]) => [lx + px * lw, ly + py * 84])
  items.push(polyline(teacherPts, p.violet, 1.5)); items.push(polyline(studentPts, p.signal, 2)); teacherPts.forEach(([px,py]) => items.push(circle(px, py, 4, p.panel, p.violet, 1.3))); studentPts.forEach(([px,py]) => items.push(circle(px, py, 4.5, p.signal, p.panel, 1)))
  const criticX = x + w * .75; const dotsY = ly + 35
  ;[-24,-14,-4,6,16,26].forEach((dy, i) => { items.push(circle(criticX - 45 + (i % 2) * 9, dotsY + dy, 3.5, p.violet, p.violet)); items.push(circle(criticX + 38 + (i % 2) * 9, dotsY + dy, 3.5, p.signal, p.signal)) })
  items.push(rect(criticX - 22, dotsY - 18, 44, 36, p.panelSoft, p.rule, 6)); items.push(text(criticX, dotsY, 'CRITIC', p.ink, '500 9px "DM Mono"', 'center')); arrow(items, criticX - 33, dotsY, criticX - 17, dotsY, p.quiet); arrow(items, criticX + 33, dotsY, criticX + 17, dotsY, p.quiet)
  items.push(text(criticX - 43, ly + 89, 'detail', p.quiet, '400 8px "DM Mono"', 'center')); items.push(text(criticX + 43, ly + 89, 'fidelity', p.quiet, '400 8px "DM Mono"', 'center'))
  items.push(text(x + w * .25, y + h - 41, 'STRUCTURE + DIVERSITY', p.violet, '500 10px "DM Mono"', 'center')); items.push(text(x + w * .75, y + h - 41, 'FEW-STEP SPARSE', p.signal, '500 10px "DM Mono"', 'center'))
}

function drawQuantBlock(items: Graphic[], x: number, y: number, w: number, h: number, p: Palette, rows = false) {
  if (!rows) { const cell = Math.min(w / 3, h / 3); for (let r = 0; r < 3; r += 1) for (let c = 0; c < 3; c += 1) items.push(rect(x + c * cell, y + r * cell, cell - 1, cell - 1, (r + c) % 3 === 0 ? p.signal : p.signalSoft, p.rule, 1)) }
  else for (let r = 0; r < 3; r += 1) items.push(rect(x, y + r * 14, w, 9, r === 1 ? p.signal : p.signalSoft, p.rule, 1))
}

function flowBox(items: Graphic[], x: number, y: number, w: number, label: string, p: Palette, active = false) {
  items.push(rect(x, y, w, 32, active ? p.signalSoft : p.panelSoft, active ? p.signal : p.rule, 6)); items.push(text(x + w / 2, y + 16, label, active ? p.ink : p.muted, '500 9px "DM Mono"', 'center', 11))
}

function drawStage3(items: Graphic[], x: number, y: number, w: number, h: number, p: Palette) {
  stageFrame(items, x, y, w, h, '3', 'FP8+Fused Kernel', '❄', p)
  items.push(text(x + w / 2, y + 63, 'FP8 QUANTIZATION · INFERENCE', p.signal, '500 9px "DM Mono"', 'center')); items.push(line(x + w / 2, y + 78, x + w / 2, y + 199, p.rule, 1, true))
  items.push(text(x + w * .25, y + 86, 'OFFLINE · ONCE', p.quiet, '500 8px "DM Mono"', 'center')); items.push(text(x + w * .75, y + 86, 'RUNTIME · TOKEN', p.quiet, '500 8px "DM Mono"', 'center'))
  drawQuantBlock(items, x + w * .25 - 29, y + 105, 58, 58, p); drawQuantBlock(items, x + w * .75 - 35, y + 108, 70, 45, p, true)
  items.push(rect(x + w * .25 + 38, y + 105, 9, 58, p.signalSoft, p.signal, 2)); items.push(rect(x + w * .75 + 43, y + 108, 9, 45, p.signalSoft, p.signal, 2))
  items.push(text(x + w * .25, y + 96, 'BF16 W', p.ink, '500 9px "DM Sans"', 'center')); items.push(text(x + w * .75, y + 96, 'BF16 A', p.ink, '500 9px "DM Sans"', 'center'))
  arrow(items, x + w * .25, y + 163, x + w * .25, y + 178, p.quiet); arrow(items, x + w * .75, y + 153, x + w * .75, y + 178, p.quiet)
  items.push(text(x + w * .25, y + 174, 'FP8 W + scale', p.muted, '500 8px "DM Mono"', 'center')); items.push(text(x + w * .75, y + 174, 'FP8 A + scale', p.muted, '500 8px "DM Mono"', 'center'))
  items.push(rect(x + w * .23, y + 187, w * .54, 30, p.signalSoft, p.signal, 6)); items.push(text(x + w / 2, y + 202, 'FP8 E4M3 GEMM', p.ink, '500 10px "DM Mono"', 'center')); items.push(line(x + 10, y + 230, x + w - 10, y + 230, p.rule, 1, true)); items.push(text(x + w / 2, y + 245, 'FUSED KERNEL EXECUTION', p.signal, '500 9px "DM Mono"', 'center'))
  const boxW = w * .68; const boxX = x + (w - boxW) / 2; flowBox(items, boxX, y + 260, boxW, 'Q / K PROJECTION', p); arrow(items, x + w / 2, y + 293, x + w / 2, y + 305, p.quiet); flowBox(items, boxX, y + 306, boxW, 'RMSNORM + RoPE + LAYOUT', p, true)
  const smallW = w * .39; flowBox(items, x + 12, y + 354, smallW, 'BLOCK ROUTING\nSPARSE ATTN', p); flowBox(items, x + w - 12 - smallW, y + 354, smallW, 'LOW-RANK\nCOMPENSATION', p); arrow(items, boxX + boxW * .28, y + 339, x + 12 + smallW / 2, y + 353, p.quiet); arrow(items, boxX + boxW * .72, y + 339, x + w - 12 - smallW / 2, y + 353, p.quiet); flowBox(items, boxX, y + 402, boxW, 'GATE FUSE · ONE PASS', p, true); items.push(text(x + w / 2, y + h - 15, 'REAL ACCELERATION', p.signal, '500 11px "DM Mono"', 'center'))
}

function drawConnector(items: Graphic[], from: { x: number; y: number; w: number; h: number }, to: { x: number; y: number; w: number; h: number }, p: Palette) {
  const sameRow = Math.abs(from.y - to.y) < 2
  if (sameRow) arrow(items, from.x + from.w + 2, from.y + from.h / 2, to.x - 2, to.y + to.h / 2, p.signal, 2.5)
  else { const startX = from.x + from.w / 2; const endX = to.x + to.w / 2; const midY = from.y + from.h + (to.y - from.y - from.h) / 2; items.push(polyline([[startX, from.y + from.h], [startX, midY], [endX, midY], [endX, to.y - 4]], p.signal, 2)); arrow(items, endX, midY, endX, to.y - 4, p.signal, 2) }
}

function drawGainStrip(items: Graphic[], x: number, y: number, w: number, p: Palette) {
  const compact = w < 650; const h = compact ? 94 : 70; items.push(rect(x, y, w, h, p.panel, p.rule, 10))
  const labels = compact ? [['SPARSITY', 'FLOPs'], ['DISTILL.', 'steps'], ['FP8', 'bandwidth'], ['KERNEL', 'overhead'], ['LATENCY', 'speedup']] : [['SPARSITY', 'FLOPs'], ['DISTILLATION', 'steps'], ['FP8', 'bandwidth'], ['KERNEL', 'overhead'], ['REAL LATENCY', 'speedup']]
  const unitW = (w - 24) / labels.length
  labels.forEach(([a, b], i) => {
    const ux = x + 12 + i * unitW; if (i > 0) items.push(line(ux, y + 14, ux, y + h - 14, p.rule))
    const ix = ux + (compact ? unitW / 2 : 23); const iy = compact ? y + 25 : y + 34
    if (i === 0) for (let row = 0; row < 4; row += 1) for (let col = 0; col < 4; col += 1) items.push(circle(ix - 9 + col * 6, iy - 9 + row * 6, 2.5, p.signal, 'transparent'))
    else if (i === 1) { items.push(polyline([[ix - 13, iy - 11], [ix - 13, iy + 10], [ix + 9, iy + 10]], p.violet, 1.5)); items.push(polyline([[ix - 4, iy - 11], [ix + 8, iy - 11], [ix + 8, iy + 1]], p.violet, 1.2, true)); items.push(circle(ix + 9, iy + 10, 4, p.violet, p.violet)) }
    else if (i === 2) { items.push({ type: 'ellipse', silent: true, shape: { cx: ix, cy: iy - 10, rx: 13, ry: 5 }, style: { fill: p.signalSoft, stroke: p.signal, lineWidth: 1.2 } }); items.push(rect(ix - 13, iy - 10, 26, 25, p.signalSoft, p.signal, 0)); items.push({ type: 'ellipse', silent: true, shape: { cx: ix, cy: iy + 15, rx: 13, ry: 5 }, style: { fill: p.panel, stroke: p.signal, lineWidth: 1.2 } }) }
    else if (i === 3) { items.push(rect(ix - 10, iy - 10, 20, 20, p.signalSoft, p.signal, 3)); for (let pin = 0; pin < 3; pin += 1) { items.push(line(ix - 14 + pin * 7, iy - 15, ix - 14 + pin * 7, iy - 10, p.signal)); items.push(line(ix - 14 + pin * 7, iy + 10, ix - 14 + pin * 7, iy + 15, p.signal)) } }
    else items.push(text(ix, iy, '→', p.signal, '500 28px "DM Sans"', 'center'))
    if (compact) { items.push(text(ix, y + 52, a, p.ink, '500 8px "DM Mono"', 'center')); items.push(text(ix, y + 70, b, p.ink, '500 8px "DM Mono"', 'center')) }
    else { items.push(text(ux + 49, y + 27, a, p.ink, '500 9px "DM Mono"')); items.push(text(ux + 49, y + 47, b, p.ink, '500 9px "DM Mono"')); if (i < 4) items.push(text(ux + unitW - 16, y + 35, '↓', p.signal, '500 17px "DM Sans"', 'center')) }
  })
}

function buildOption(width: number): { option: EChartsOption; height: number } {
  const p = readPalette(); const items: Graphic[] = []; const pad = 4; const boxes: Array<{ x: number; y: number; w: number; h: number }> = []
  if (width >= 1020) { const usable = width - pad * 2 - GAP * 3; const ratios = [.205, .225, .31, .26]; let cursor = pad; ratios.forEach((ratio, i) => { const boxW = i === ratios.length - 1 ? width - pad - cursor : Math.floor(usable * ratio); boxes.push({ x: cursor, y: 4, w: boxW, h: BASE_HEIGHT }); cursor += boxW + GAP }) }
  else if (width >= 650) { const boxW = (width - pad * 2 - GAP) / 2; boxes.push({ x: pad, y: 4, w: boxW, h: BASE_HEIGHT }, { x: pad + boxW + GAP, y: 4, w: boxW, h: BASE_HEIGHT }, { x: pad, y: BASE_HEIGHT + 28, w: boxW, h: BASE_HEIGHT }, { x: pad + boxW + GAP, y: BASE_HEIGHT + 28, w: boxW, h: BASE_HEIGHT }) }
  else for (let i = 0; i < 4; i += 1) boxes.push({ x: pad, y: 4 + i * (BASE_HEIGHT + 24), w: width - pad * 2, h: BASE_HEIGHT })
  drawStage0(items, boxes[0].x, boxes[0].y, boxes[0].w, boxes[0].h, p); drawStage1(items, boxes[1].x, boxes[1].y, boxes[1].w, boxes[1].h, p); drawStage2(items, boxes[2].x, boxes[2].y, boxes[2].w, boxes[2].h, p); drawStage3(items, boxes[3].x, boxes[3].y, boxes[3].w, boxes[3].h, p); for (let i = 0; i < boxes.length - 1; i += 1) drawConnector(items, boxes[i], boxes[i + 1], p)
  const stripY = Math.max(...boxes.map((box) => box.y + box.h)) + 12; drawGainStrip(items, pad, stripY, width - pad * 2, p)
  const stripHeight = width < 650 ? 94 : 70
  return { height: stripY + stripHeight + 4, option: { animation: true, animationDuration: 500, animationEasing: 'cubicOut', tooltip: { show: false }, graphic: { elements: items } as never } }
}

function render() {
  if (!host.value || !chart) return
  const width = Math.max(300, Math.floor(host.value.getBoundingClientRect().width)); const { option, height } = buildOption(width); const nextHeight = `${height}px`
  if (host.value.style.height !== nextHeight) host.value.style.height = nextHeight
  chart.resize({ width, height }); chart.setOption(option, { notMerge: true, lazyUpdate: false })
}

onMounted(() => { if (!host.value) return; chart = echarts.init(host.value, undefined, { renderer: 'svg' }); observer = new ResizeObserver(render); observer.observe(host.value); render() })
onBeforeUnmount(() => { observer?.disconnect(); chart?.dispose(); chart = null })
</script>

<template>
  <div class="framework-wrap">
    <div ref="host" class="framework-chart" role="img" aria-label="SparkDiffusion 四阶段框架：预训练稠密 Video DiT、稀疏预热、轨迹混合蒸馏，以及 FP8 量化与融合内核部署。"></div>
    <p class="framework-summary">Dense video DiT is warmed up with sparse and compensation branches, distilled across high- and low-noise trajectories, then deployed with FP8 quantization and fused kernels.</p>
  </div>
</template>

<style scoped>
.framework-wrap { width: 100%; min-width: 0; }
.framework-chart { width: 100%; min-height: 554px; }
.framework-summary { position: absolute; width: 1px; height: 1px; padding: 0; margin: -1px; overflow: hidden; clip: rect(0, 0, 0, 0); white-space: nowrap; border: 0; }
</style>
