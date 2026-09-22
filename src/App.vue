<script setup lang="ts">

import { onBeforeUnmount, onMounted, ref } from 'vue'
import EChart from './components/EChart.vue'
import FrameworkDiagram from './components/FrameworkDiagram.vue'
import {
  headlineOption,
  latencyOption,
  radarOption,
  speedupOption,
  trapOption,
  vbenchOption,
} from './chartOptions'
import { qualityRows } from './data/paperData'

type Demo = {
  id: string
  label: string
  source: string
  prompt: string
}

const menuOpen = ref(false)
const expandedVideo = ref<Demo | null>(null)
const copiedKey = ref<string | null>(null)

/* Static option objects — charts don't react to anything, so build once. */
const latencyOpt = latencyOption()
const speedupOpt = speedupOption()
const headlineOpt = headlineOption()
const trapOpt = trapOption()
const vbenchOpt = vbenchOption()
const radarOpt = radarOption()

const qualityModels = [...new Set(qualityRows.map((r) => r.model))]

type DemoLatency = { h100: string; rtx5090: string }
type DemoMethodSpec = {
  folder: string
  label: string
  note?: string
  ours?: boolean
  latency: DemoLatency
}
type DemoModelSpec = { folder: string; title: string; methods: DemoMethodSpec[] }

// One card per model (NVIDIA Sol-Engine style): title + method strip → one
// prompt → one row of method videos. `folder` is the directory under
// docs/videos-real; each method takes the FIRST file (glob keys are sorted)
// from videos-real/<folder>/<method>/, and the prompt is the FIRST line of
// videos-real/<folder>/prompts.txt.
const demoSpecs: DemoModelSpec[] = [
  {
    title: 'Wan2.1-T2V-1.3B · 480P · 3 Steps',
    folder: 'Wan2.1-T2V-1.3B-480P-90',
    methods: [
      { folder: 'full_attention', label: 'Full Attention', latency: { h100: '92 s', rtx5090: '182 s' } },
      { folder: 'fastwan', label: 'FastWan', note: '90% sparsity · 3 Steps', latency: { h100: '1.2 s', rtx5090: '2.8 s' } },
      { folder: 'turbo_diffusion', label: 'Turbo', note: '90% sparsity · 3 Steps', latency: { h100: '1 s', rtx5090: '2 s' } },
      { folder: 'ours', label: 'Ours', note: '90% sparsity · 3 Steps', ours: true, latency: { h100: '0.6 s', rtx5090: '1.3 s' } },
    ],
  },
  {
    title: 'Wan2.1-T2V-14B · 480P',
    folder: 'Wan2.1-T2V-14B-480P-90',
    methods: [
      { folder: 'full_attention', label: 'Full Attention', latency: { h100: '460 s', rtx5090: '1674 s' } },
      { folder: 'turbo_diffusion', label: 'Turbo', note: '90% sparsity', latency: { h100: '6.2 s', rtx5090: '10.2 s' } },
      { folder: 'ours', label: 'Ours', note: '90% sparsity', ours: true, latency: { h100: '3.6 s', rtx5090: '8.3 s' } },
    ],
  },
  {
    title: 'Wan2.1-I2V-14B · 720P',
    folder: 'Wan2.1-I2V-14B-720P-95',
    methods: [
      { folder: 'full_attention', label: 'Full Attention', latency: { h100: '1757 s', rtx5090: '4769 s' } },
      // { folder: 'ours_top0.05', label: 'Ours (top 5%)', note: '95% sparsity', ours: true },
      { folder: 'ours_top0.03', label: 'Ours (top 3%)', note: '97% sparsity', ours: true, latency: { h100: '8 s', rtx5090: '18 s' } },
    ],
  },
  {
    title: 'Wan2.1-T2V-14B · 720P · 3 Steps',
    folder: 'Wan2.1-T2V-720p-3steps',
    methods: [
      { folder: 'full_attention', label: 'Full Attention', latency: { h100: '1757 s', rtx5090: '4769 s' } },
      { folder: 'fastwan', label: 'FastWan', note: '90% · 3 steps', latency: { h100: '20.5 s', rtx5090: '54.1 s' } },
      { folder: 'turbo_diffusion', label: 'Turbo', note: '90% · 3 steps', latency: { h100: '16 s', rtx5090: '25.3 s' } },
      { folder: 'ours', label: 'Ours', note: '97% · 3 steps', ours: true, latency: { h100: '8 s', rtx5090: '18 s' } },
    ],
  },
  {
    title: 'Wan2.2-T2V-A14B · 720P',
    folder: 'Wan2.2-T2V-A14B-720P-97',
    methods: [
      { folder: 'full_attention', label: 'Full Attention', latency: { h100: '1508 s', rtx5090: '4545 s' } },
      { folder: 'ours', label: 'Ours', note: '97% sparsity', ours: true, latency: { h100: '8 s', rtx5090: '25.1 s' } },
    ],
  },
]

const allVideos = import.meta.glob('../docs/videos-real/*/*/*.mp4', {
  eager: true,
  import: 'default',
}) as Record<string, string>

// Prompts live per model: videos-real/<model>/prompts.txt, one prompt per
// gallery row (Nth line ↔ Nth row). query: '?raw' makes the glob return the
// file TEXT — without it .txt is treated as a static asset and you get the
// bundled URL instead of the content. Missing file or short file falls back
// to a placeholder so a row label never blanks.
const promptsByModel = import.meta.glob('../docs/videos-real/*/prompts.txt', {
  eager: true,
  query: '?raw',
  import: 'default',
}) as Record<string, string>

const modelPrompts = (model: string) => {
  const raw = Object.entries(promptsByModel).find(([p]) => p.includes(`/${model}/`))?.[1] ?? ''
  return raw.split(/\r?\n/).filter((line) => line.trim() !== '')
}

// All videos of videos-real/<model>/<method>/ in sorted order — glob keys
// are sorted paths, so row N pairs the Nth file of every method.
const videosOf = (model: string, method: string) =>
  Object.entries(allVideos)
    .filter(([p]) => p.includes(`/${model}/${method}/`))
    .map(([, source]) => source)

// Prompt line N (0-based) of videos-real/<model>/prompts.txt.
const modelPrompt = (model: string, index: number) =>
  modelPrompts(model)[index] || `[ Add the original input prompt for ${model} row ${index + 1} ]`

// Two prompt rows per model card — the Nth row pairs the Nth video of each
// method with the Nth prompt line.
const ROWS_PER_MODEL = 2

type DemoCell = { method: string; label: string; note?: string; ours: boolean; latency: DemoLatency; source: string }
type DemoRow = { prompt: string; cells: DemoCell[] }
type DemoBlock = { folder: string; title: string; methodLabels: string[]; rows: DemoRow[] }

const demoBlocks: DemoBlock[] = demoSpecs
  .map((spec) => {
    const byMethod = spec.methods.map((method) => ({
      ...method,
      sources: videosOf(spec.folder, method.folder),
    }))
    // Videos land in docs/videos-real progressively — surface gaps in the
    // console so a silently half-empty card is easy to spot in dev.
    const missing = byMethod.filter((m) => m.sources.length === 0).map((m) => m.folder)
    if (missing.length) {
      console.warn(`[demos] ${spec.folder}: no videos found for ${missing.join(', ')}`)
    }

    const found = byMethod.filter((m) => m.sources.length > 0)
    const rowCount = Math.min(
      ROWS_PER_MODEL,
      Math.max(0, ...found.map((m) => m.sources.length)),
    )
    const rows: DemoRow[] = Array.from({ length: rowCount }, (_, index) => ({
      prompt: modelPrompt(spec.folder, index),
      cells: found
        .filter((m) => m.sources[index])
        .map((m) => ({
          method: m.folder,
          label: m.label,
          note: m.note,
          ours: m.ours ?? false,
          latency: m.latency,
          source: m.sources[index],
        })),
    }))

    return {
      folder: spec.folder,
      title: spec.title,
      methodLabels: found.map((m) => m.label),
      rows,
    }
  })
  .filter((block) => block.rows.length > 0 && block.rows.some((row) => row.cells.length > 0))

// Hero background video — reads the FIRST (sorted) mp4 from the dedicated
// docs/videos-hero/ folder. Falls back to the first demo video when the
// folder is empty so the hero never goes blank mid-production.
const heroVideos = import.meta.glob('../docs/videos-hero/*.mp4', {
  eager: true,
  import: 'default',
}) as Record<string, string>

const heroDemo = (() => {
  const [heroPath, heroSource] = Object.entries(heroVideos)[0] ?? []
  if (heroSource) return { label: `Hero · ${heroPath.split('/').pop()}`, source: heroSource }
  for (const spec of demoSpecs) {
    for (const method of spec.methods) {
      const source = videosOf(spec.folder, method.folder)[0]
      if (source) return { label: `${spec.folder} / ${method.label}`, source }
    }
  }
  return null
})()
const paperUrl = 'https://arxiv.org/abs/2609.23153'

function openVideo(demo: Demo | null) {
  if (!demo) return
  expandedVideo.value = demo
}

function closeVideo() {
  expandedVideo.value = null
}

// Mirrors the in-page anchor links — relies on html { scroll-behavior: smooth }
// so the reduced-motion media query still wins for users who opt out.
function scrollToId(id: string) {
  document.getElementById(id)?.scrollIntoView()
}

const citation = `@misc{liu2026sparkdiffusionmitigatinghighsparsitytrap,
      title={SparkDiffusion: Mitigating the High-Sparsity Trap --- A Unified Framework for up to $265\\times$ Single-GPU Acceleration of Visual Generation},
      author={Yuxi Liu and Haoyu Li and Zekun Zhang and Tengxu Sun and Yixiang Cai and Jiayong Li and Yifei Xia and Tianle Liu and Baole Ai and Ang Wang and Jiamang Wang and Lin Qu and Kai Zhang and Kun Yuan and Bin Cui},
      year={2026},
      eprint={2609.23153},
      archivePrefix={arXiv},
      primaryClass={cs.CV},
      url={https://arxiv.org/abs/2609.23153},
}`

// Related work from this group — same BibTeX style as the primary entry.
const relatedWork = `@misc{zhang2026rolarotarypositionedlowranklinear,
      title={RoLA: Rotary-Positioned Low-Rank Linear Attention for Efficient Diffusion Transformers},
      author={Zekun Zhang and Yixiang Cai and Yuxi Liu and Tengxu Sun and Tianle Liu and Zhoutong Wu and Haoyu Li and Baole Ai and Ang Wang and Jiamang Wang and Lin Qu and Kun Yuan},
      year={2026},
      eprint={2609.06712},
      archivePrefix={arXiv},
      primaryClass={cs.CV},
      url={https://arxiv.org/abs/2609.06712},
}

@misc{liu2026ropeslr3dropedrivensparselowrank,
      title={RoPeSLR: 3D RoPE-driven Sparse-LowRank Attention for Efficient Diffusion Transformers},
      author={Yuxi Liu and Zekun Zhang and Yixiang Cai and Renjia Deng and Yutong He and Kun Yuan},
      year={2026},
      eprint={2605.20659},
      archivePrefix={arXiv},
      primaryClass={cs.CV},
      url={https://arxiv.org/abs/2605.20659},
}

@misc{liu2026crossdistillbalancingqualitydiversity,
      title={CrossDistill: Balancing Quality and Diversity via Trajectory-Level Hybrid Few-Step Distillation},
      author={Yuxi Liu and Haoyu Li and Yixiang Cai and Tengxu Sun and Zekun Zhang and Baole Ai and Ang Wang and Jiamang Wang and Lin Qu and Kun Yuan and Kai Zhang},
      year={2026},
      eprint={2609.14725},
      archivePrefix={arXiv},
      primaryClass={cs.CV},
      url={https://arxiv.org/abs/2609.14725},
}`

async function copyBibtex(key: string, text: string) {
  try {
    await navigator.clipboard.writeText(text)
    copiedKey.value = key
    window.setTimeout(() => {
      if (copiedKey.value === key) copiedKey.value = null
    }, 1800)
  } catch {
    // Clipboard access may be disabled on some preview deployments.
  }
}

function onKeydown(event: KeyboardEvent) {
  if (event.key === 'Escape') closeVideo()
}

onMounted(() => window.addEventListener('keydown', onKeydown))
onBeforeUnmount(() => window.removeEventListener('keydown', onKeydown))
</script>

<template>
  <main>
    <section id="top" class="hero">
      <video
        class="hero-film"
        :src="heroDemo?.source"
        autoplay
        muted
        loop
        playsinline
        preload="auto"
        aria-label="Project demo video"
      ></video>
      <div class="hero-shade" aria-hidden="true"></div>
      <div class="hero-grain" aria-hidden="true"></div>

      <nav class="nav" aria-label="Primary navigation">
        <a class="brand" href="#top" aria-label="Project home">
          <span>SparkDiffusion</span>
        </a>

        <button
          class="menu-button"
          type="button"
          :aria-expanded="menuOpen"
          aria-label="Toggle navigation"
          @click="menuOpen = !menuOpen"
        >
          <span></span><span></span>
        </button>

        <div class="nav-links" :class="{ 'is-open': menuOpen }">
          <a href="#abstract" @click="menuOpen = false">Abstract</a>
          <a href="#method" @click="menuOpen = false">Method</a>
          <a href="#benchmarks" @click="menuOpen = false">Results</a>
          <a href="#demos" @click="menuOpen = false">Demos</a>
          <a href="#citation" @click="menuOpen = false">Citation</a>
          <a class="nav-action" :href="paperUrl" target="_blank" rel="noreferrer" @click="menuOpen = false">
            <svg class="nav-icon" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.8" aria-hidden="true">
              <path d="M14.5 2.75H6.8a1.8 1.8 0 0 0-1.8 1.8v14.9a1.8 1.8 0 0 0 1.8 1.8h10.4a1.8 1.8 0 0 0 1.8-1.8V7.85L14.5 2.75Z" />
              <path d="M14 2.75v5.5h5M8.5 12h7M8.5 15.5h7" />
            </svg>
            <span>Paper</span>
          </a>
          <a class="nav-action" href="https://github.com/AlibabaResearch/SparkDiffusion" target="_blank" rel="noreferrer" @click="menuOpen = false">
            <svg class="nav-icon github-icon" viewBox="0 0 24 24" fill="currentColor" aria-hidden="true">
              <path d="M12 2.2a10 10 0 0 0-3.16 19.49c.5.09.68-.22.68-.48v-1.86c-2.78.61-3.37-1.18-3.37-1.18-.45-1.16-1.11-1.47-1.11-1.47-.91-.62.07-.61.07-.61 1 .07 1.54 1.04 1.54 1.04.9 1.55 2.35 1.1 2.93.84.09-.66.35-1.1.64-1.36-2.22-.25-4.56-1.11-4.56-4.94 0-1.09.39-1.98 1.03-2.68-.1-.25-.45-1.27.1-2.65 0 0 .84-.27 2.75 1.02A9.55 9.55 0 0 1 12 6.52c.85 0 1.7.12 2.5.34 1.91-1.29 2.75-1.02 2.75-1.02.55 1.38.2 2.4.1 2.65.64.7 1.03 1.59 1.03 2.68 0 3.84-2.35 4.69-4.58 4.94.36.31.68.9.68 1.81v2.8c0 .27.18.58.69.48A10 10 0 0 0 12 2.2Z" />
            </svg>
            <span>Code</span>
          </a>
        </div>
      </nav>

      <div class="hero-copy page-width">
        <p class="kicker">SparkDiffusion · 2027</p>
        <h1>SparkDiffusion:<br /><span>Mitigating the High-Sparsity Trap — A Unified Framework for up to 265× Single-GPU Acceleration of Visual Generation</span></h1>
        <p class="hero-lede">
          A unified post-training framework — compensated sparse attention, trajectory-mixed
          distillation, and fused FP8 deployment — that turns dense video DiTs into high-sparsity,
          few-step generators with up to 265× measured end-to-end speedup on a single GPU.
        </p>
        <p class="authors">
          Yuxi Liu<sup>*1,3</sup> <i>·</i> Haoyu Li<sup>*2,3</sup> <i>·</i> Zekun Zhang<sup>*1</sup> <i>·</i> Tengxu Sun<sup>†3</sup> <i>·</i> Yixiang Cai<sup>1</sup> <i>·</i> Jiayong Li<sup>3,5</sup> <i>·</i> Yifei Xia<sup>6</sup> <i>·</i> Tianle Liu<sup>4</sup> <i>·</i> Baole Ai<sup>3</sup> <i>·</i> Ang Wang<sup>3</sup> <i>·</i> Jiamang Wang<sup>3</sup> <i>·</i> Lin Qu<sup>3</sup> <i>·</i> Kai Zhang<sup>2</sup> <i>·</i> Kun Yuan<sup>†1</sup> <i>·</i> Bin Cui<sup>†6</sup>
        </p>
        <p class="affiliation">
          <sup>1</sup>Peking University, Melon Group <i>·</i> <sup>2</sup>Tsinghua University <i>·</i> <sup>3</sup>Alibaba Group <i>·</i> <sup>4</sup>University of Electronic Science and Technology of China <i>·</i> <sup>5</sup>Harbin Institute of Technology <i>·</i> <sup>6</sup>Peking University
        </p>
        <p class="affiliation-note">* Equal contribution &nbsp;·&nbsp; † Corresponding author</p>
      </div>

      <button
        v-if="heroDemo"
        class="expand-button hero-expand"
        type="button"
        aria-label="Expand hero video"
        title="Expand video"
        @click="openVideo({ id: 'hero', label: heroDemo.label, source: heroDemo.source, prompt: 'Hero background video' })"
      >
        <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.9" aria-hidden="true">
          <path d="M15 3h6v6M21 3l-7 7M9 21H3v-6M3 21l7-7" />
        </svg>
      </button>

      <button
        class="scroll-cue"
        type="button"
        aria-label="Continue to abstract"
        @click="scrollToId('abstract')"
      ><span>Let's go SparkDiffusion</span><b aria-hidden="true"></b></button>
    </section>

    <section id="abstract" class="abstract-section page-width">
      <div class="abstract-title">
        <h2>Abstract</h2>
      </div>
      <div class="abstract-copy">
        <p>
          Video diffusion transformers are expensive because attention dominates long spatiotemporal
          token sequences. We identify the <b>high-sparsity trap</b>: at extreme attention sparsity,
          step-local training losses keep decreasing while terminal generation quality stagnates or
          degrades. The trap is one of supervision: the dominant terminal errors originate in the
          high-noise structure-generation stage, and terminal-aligned training corrects terminal
          errors that substantially extended step-local training cannot. This yields a simple
          staging principle: first adapt the sparse architecture into a coarse prior, then correct
          the terminal distribution. We instantiate the principle as <b>SparkDiffusion</b>, a
          unified acceleration framework for visual generation that combines a short sparse warm-up,
          few-step trajectory-mixed distillation, and FP8 quantization with fused kernels.
          SparkDiffusion sustains 97% attention sparsity with strong visual quality on long-sequence
          720P generation across Wan2.1/Wan2.2 backbones and T2V/I2V tasks, and 90% sparsity on
          Wan2.1-T2V-1.3B-480P. With 3-step CFG-free inference, SparkDiffusion achieves a
          265× end-to-end speedup over the 50-step CFG dense baseline for Wan2.1-T2V-14B-720P on a
          single RTX 5090 (220× on H100), and denoises a Wan2.1-T2V-1.3B-480P video in 1.3 s.
        </p>
        <div class="impact-strip" aria-label="Key results from the paper">
          <div><strong>2 6 5 ×</strong><span>end-to-end speedup on Wan2.1-T2V-14B-720P at 97% sparsity · 3-step, RTX 5090 (220× on H100)</span></div>
          <div><strong>1 . 3 s</strong><span>per Wan2.1-T2V-1.3B-480P video on a single RTX 5090 (0.6 s on H100)</span></div>
          <div><strong>9 7 %</strong><span>attention sparsity sustained on long-sequence 720P T2V/I2V generation</span></div>
        </div>
      </div>
    </section>

    <section id="method" class="method-section page-width">
      <div class="demos-heading">
        <div>
          <h2>Method</h2>
        </div>
      </div>

      <figure class="figure-card method-figure">
        <div class="figure-head">
          <h3>SparkDiffusion framework overview(Simplified version)</h3>
          <p class="figure-head-note">
           For the complete framework overview, see the
            <a class="figure-paper-link" :href="paperUrl" target="_blank" rel="noreferrer">paper ↗</a>.
          </p>
        </div>
        <FrameworkDiagram />
        <figcaption class="figure-note">
          Two post-training stages and one deployment step. Sparse warm-up adapts the dense backbone
          into a coarse prior; trajectory-mixed distillation corrects the terminal distribution with
          high-noise PCM-style consistency and low-noise DMD-style distribution matching (3-step
          student: one PCM step + two DMD steps); FP8 quantization with fused kernels converts the
          saved computation into wall-clock speedup. Default instantiation: RoLA as the sparse
          module, the CrossDistill schedule for distillation.
        </figcaption>
      </figure>
    </section>

    <section id="benchmarks" class="bench-section page-width">
      <div class="demos-heading">
        <div>
          <h2>Results</h2>
        </div>
      </div>

      <!-- Latency & speedup (paper Figures 2 + 3) -->
      <figure class="figure-card">
        <div class="figure-head">
          <h3>End-to-end latency &amp; speedup</h3>
          <span class="figure-tag">Figures 2–3</span>
        </div>

        <div class="figure-strip" aria-label="Headline speedup numbers">
          <div><strong>265×</strong><span>max measured speedup · Wan2.1-T2V-14B-720P @ 97% · 3-step · RTX 5090</span></div>
          <div><strong>220×</strong><span>Wan2.1-T2V-14B-720P @ 97% · 3-step · H100</span></div>
          <div><strong>189×</strong><span>Wan2.2-T2V-A14B-720P @ 97% sparsity · H100</span></div>
        </div>

        <div class="chart-duo chart-duo-stack">
          <div class="chart-slot">
            <p class="chart-slot-label">(a) Latency per video · log scale</p>
            <EChart :option="latencyOpt" height="330px" />
          </div>
          <div class="chart-slot">
            <p class="chart-slot-label">(b) Speedup over Full Attention</p>
            <EChart :option="speedupOpt" height="330px" />
          </div>
          <div class="chart-slot chart-slot-wide">
            <p class="chart-slot-label">
              Headline — Wan2.1-T2V-14B-720P @ 97% sparsity · RTX 5090 · 81 frames (5.06 s clip)
            </p>
            <EChart :option="headlineOpt" height="190px" />
          </div>
        </div>

        <figcaption class="figure-note">
          Suffixes 90 / 97 denote attention sparsity. Full Attention = 50-step CFG (NFE 100) in
          BF16 with FlashAttention; SparkDiffusion = 3-step CFG-free (NFE 3) with the full stack.
          Speedup = Full Attention latency ÷ SparkDiffusion latency, rounded to the nearest
          integer. Wan2.2 RTX 5090 latency includes high-noise / low-noise expert switching
          overhead; on H100 both experts stay resident, so no switching cost is incurred.
        </figcaption>
      </figure>

      <!-- High-sparsity trap (paper Figure 4) -->
      <figure class="figure-card">
        <div class="figure-head">
          <h3>The high-sparsity trap</h3>
          <span class="figure-tag">Figure 4</span>
        </div>
        <EChart :option="trapOpt" height="340px" />
        <figcaption class="figure-note">
          Oracle correction on Wan2.1-T2V-14B-480P (50-step sampler, 32 prompts × 4 seeds):
          replacing the sparse student's velocity with the dense teacher's inside one matched noise
          window. At 97% sparsity, fixing the five highest-noise steps removes most of the terminal
          error while fixing the five lowest-noise steps leaves it essentially unchanged — the
          dominant failure is injected during high-noise structure generation and accumulated by
          later steps.
        </figcaption>
      </figure>

      <!-- Quality (paper Table 2) -->
      <figure class="figure-card">
        <div class="figure-head">
          <h3>Generation quality — VBench &amp; VBench-2.0</h3>
          <span class="figure-tag">Table 2</span>
        </div>

        <div class="chart-duo">
          <div class="chart-slot">
            <p class="chart-slot-label">VBench Total (zoomed 80–85)</p>
            <EChart :option="vbenchOpt" height="300px" />
          </div>
          <div class="chart-slot">
            <p class="chart-slot-label">VBench-2.0 capabilities · Wan2.1-T2V-14B</p>
            <EChart :option="radarOpt" height="300px" />
          </div>
        </div>

        <div class="bench-table-wrap quality-wrap">
          <table class="bench-table quality-table">
            <thead>
              <tr>
                <th>Model</th>
                <th>Method</th>
                <th>VBench ↑</th>
                <th>VB2 Total ↑</th>
                <th>Creat. ↑</th>
                <th>Common. ↑</th>
                <th>Control. ↑</th>
                <th>Human Fid. ↑</th>
                <th>Physics ↑</th>
                <th>Sparsity</th>
              </tr>
            </thead>
            <tbody>
              <template v-for="model in qualityModels" :key="model">
                <tr
                  v-for="(row, i) in qualityRows.filter((r) => r.model === model)"
                  :key="model + row.method + row.sparsity"
                  :class="{ 'is-ours': row.method === 'SparkDiffusion', 'is-first': i === 0 }"
                >
                  <td v-if="i === 0" :rowspan="qualityRows.filter((r) => r.model === model).length">{{ model }}</td>
                  <td>{{ row.method }}</td>
                  <td class="qnum qnum-strong">{{ row.vbench.toFixed(2) }}</td>
                  <td class="qnum">{{ row.vb2.toFixed(2) }}</td>
                  <td class="qnum">{{ row.creativity.toFixed(2) }}</td>
                  <td class="qnum">{{ row.commonsense.toFixed(2) }}</td>
                  <td class="qnum">{{ row.controllability.toFixed(2) }}</td>
                  <td class="qnum">{{ row.humanFidelity.toFixed(2) }}</td>
                  <td class="qnum">{{ row.physics.toFixed(2) }}</td>
                  <td class="qnum">{{ row.sparsity }}</td>
                </tr>
              </template>
            </tbody>
          </table>
        </div>

        <figcaption class="figure-note">
          Wan2.1-T2V-1.3B evaluated at 480 × 832; Wan2.1-T2V-14B and Wan2.2-T2V-A14B at
          720 × 1280, all with 81-frame generation. Baselines run at their strongest official
          configuration (90% sparsity); SparkDiffusion rows are measured with the full deployment
          stack (W8A8 FP8 with fused kernels) active. At matched 90% sparsity, SparkDiffusion
          improves over all baselines on every reported metric on Wan2.1-T2V-14B-720P; pushed to
          97% it stays within ~0.9 VBench points of the dense baseline while running orders of
          magnitude faster, and exceeds every baseline on VBench-2.0 Human Fidelity.
        </figcaption>
      </figure>
    </section>

    <section id="demos" class="demos-section page-width">
      <div class="demos-heading">
        <div>
          <h2>Demos</h2>
        </div>
      </div>

      <div
        v-for="block in demoBlocks"
        :key="block.folder"
        class="model-block"
        :data-model="block.folder"
      >
        <div class="model-head">
          <h3 class="model-title">{{ block.title }}</h3>
          <span class="model-methods">{{ block.methodLabels.join(' · ') }}</span>
        </div>

        <div v-for="(row, rowIndex) in block.rows" :key="block.folder + rowIndex" class="model-row">
          <p class="model-prompt">{{ row.prompt }}</p>

          <!-- Column count follows the method count (2–4) via --cols; the media
               queries below still collapse it to 2 / 1 columns on small screens. -->
          <div class="demo-row-videos" :style="{ '--cols': row.cells.length }">
            <figure
              v-for="cell in row.cells"
              :key="block.folder + rowIndex + cell.method"
              class="video-card"
            >
              <div class="demo-frame">
                <video :src="cell.source" autoplay muted loop playsinline preload="metadata"></video>
                <button
                  type="button"
                  class="expand-button"
                  :aria-label="`Expand ${cell.label} video — ${row.prompt}`"
                  :title="cell.label"
                  @click.stop="openVideo({ id: block.folder + rowIndex + cell.method, label: `${block.title} · ${cell.label}`, source: cell.source, prompt: row.prompt })"
                >
                  <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.9" aria-hidden="true">
                    <path d="M15 3h6v6M21 3l-7 7M9 21H3v-6M3 21l7-7" />
                  </svg>
                </button>
              </div>
              <figcaption class="vid-meta" :class="{ 'is-ours': cell.ours }">
                <div class="vid-primary">
                  <span class="t">{{ cell.label }}</span>
                  <span v-if="cell.note" class="r">{{ cell.note }}</span>
                </div>
                <div class="vid-latency" aria-label="Inference latency">
                  <span>H100 <strong>{{ cell.latency.h100 }}</strong></span>
                  <span>RTX 5090 <strong>{{ cell.latency.rtx5090 }}</strong></span>
                </div>
              </figcaption>
            </figure>
          </div>
        </div>
      </div>
    </section>

    <section id="citation" class="citation-section">
      <div class="page-width citation-layout">
        <div class="citation-intro">
          <h2>Citation</h2>
        </div>
        <div class="citation-card">
          <div class="citation-card-top">
            <div><span>BIBTEX / 2026</span></div>
            <button type="button" class="copy-button" @click="copyBibtex('ours', citation)">
              <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.8" aria-hidden="true"><rect x="8" y="8" width="11" height="12" rx="1.5" /><path d="M5 16V5.5A1.5 1.5 0 0 1 6.5 4H16" /></svg>
              {{ copiedKey === 'ours' ? 'Copied' : 'Copy citation' }}
            </button>
          </div>
          <pre><code>{{ citation }}</code></pre>
          <div class="citation-card-foot"><span>PLEASE CITE THIS WORK</span><span>↗</span></div>
        </div>

        <div class="citation-card citation-card-related">
          <div class="citation-card-top">
            <div><span>RELATED WORK / BIBTEX</span></div>
            <button type="button" class="copy-button" @click="copyBibtex('related', relatedWork)">
              <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.8" aria-hidden="true"><rect x="8" y="8" width="11" height="12" rx="1.5" /><path d="M5 16V5.5A1.5 1.5 0 0 1 6.5 4H16" /></svg>
              {{ copiedKey === 'related' ? 'Copied' : 'Copy all' }}
            </button>
          </div>
          <pre><code>{{ relatedWork }}</code></pre>
          <div class="citation-card-foot"><span>RELATED WORK FROM THIS EFFORT</span><span>↗</span></div>
        </div>
      </div>
    </section>

    <footer class="footer page-width">
      <a href="#top">Back to top ↑</a>
    </footer>

    <div v-if="expandedVideo" class="lightbox" role="dialog" aria-modal="true" aria-label="Expanded video" @click="closeVideo">
      <button type="button" class="lightbox-close" aria-label="Close expanded video" @click="closeVideo">
        <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" aria-hidden="true"><path d="M18 6 6 18M6 6l12 12" /></svg>
      </button>
      <div class="lightbox-stage" @click.stop>
        <video :src="expandedVideo.source" autoplay muted loop playsinline controls></video>
        <p>{{ expandedVideo.id === 'hero' ? 'Hero background video' : expandedVideo.label }}</p>
      </div>
    </div>
  </main>
</template>
