<script setup lang="ts">
import { onBeforeUnmount, onMounted, ref } from 'vue'
import tinymce from 'tinymce'
import type { Editor } from 'tinymce'
import 'tinymce/icons/default'
import 'tinymce/themes/silver'
import 'tinymce/models/dom'
import 'tinymce/plugins/lists'
import 'tinymce/plugins/link'
import 'tinymce/plugins/code'
import 'tinymce/skins/ui/oxide/skin.min.css'
import 'tinymce/skins/ui/oxide/content.min.css'
import {
  destroyFormulaRendering,
  initFormulaRendering,
  openFormulaEditor,
  renderLatex,
  serializeFormulaContent,
  serializeFormulaLatex,
} from 'formula-editor-modal'
import type { FormulaEditorHandle } from 'formula-editor-modal'

const editorElement = ref<HTMLDivElement>()
const toolbarElement = ref<HTMLDivElement>()
const savedContent = ref('')
const error = ref('')
const outputType = ref<'latex' | 'img'>('latex')
const exportFormat = ref<'html' | 'latex'>('html')
let editor: Editor | undefined
let formulaDialog: FormulaEditorHandle | undefined
let disposed = false

function editFormula(target?: HTMLElement) {
  if (!editor || formulaDialog) return
  const currentEditor = editor
  currentEditor.focus()
  if (!target) currentEditor.selection.collapse(false)
  const bookmark = currentEditor.selection.getBookmark(1)
  const restoreSelection = () => {
    currentEditor.focus()
    currentEditor.selection.moveToBookmark(bookmark)
  }
  formulaDialog = openFormulaEditor({
    latex: target?.getAttribute('data-latex') ?? '',
    outputType: target ? (target.tagName === 'IMG' ? 'img' : 'latex') : outputType.value,
    async onConfirm(result) {
      const html = result.type === 'latex' ? await renderLatex(result.latex) : result.data
      if (disposed) return
      restoreSelection()
      currentEditor.undoManager.transact(() => {
        if (target?.isConnected) currentEditor.selection.select(target)
        currentEditor.insertContent(html)
      })
      currentEditor.dispatch('input')
      currentEditor.dispatch('change')
      formulaDialog = undefined
    },
    onCancel() {
      restoreSelection()
      formulaDialog = undefined
    },
  })
}

function exportContent() {
  if (editor) savedContent.value = exportFormat.value === 'latex'
    ? serializeFormulaLatex(editor.getContent())
    : serializeFormulaContent(editor.getContent())
}

onMounted(async () => {
  try {
    await tinymce.init({
      target: editorElement.value,
      inline: true,
      // Inline 模式让 SDK 在同一 document 中渲染公式的 Shadow DOM。
      fixed_toolbar_container_target: toolbarElement.value,
      toolbar_persist: true,
      menubar: false,
      skin: false,
      content_css: false,
      plugins: 'lists link code',
      toolbar: 'undo redo | blocks | bold italic | bullist numlist | link | formula | code',
      noneditable_class: 'fe-formula',
      extended_valid_elements: 'span[class|data-latex|contenteditable|style],img[class|data-latex|src|alt|width|height|style]',
      setup(instance) {
        editor = instance
        instance.ui.registry.addButton('formula', {
          text: '插入公式',
          tooltip: '打开 LaTeX 公式编辑器',
          onAction: () => editFormula(),
        })
        instance.on('dblclick', event => {
          const node = (event.target as Element).closest<HTMLElement>(
            'span.fe-formula[data-latex], img.fe-formula-image[data-latex]',
          )
          if (node && instance.getBody().contains(node)) {
            event.preventDefault()
            editFormula(node)
          }
        })
        instance.on('init', () => {
          if (disposed) instance.remove()
        })
      },
    })
    if (!disposed) {
      await initFormulaRendering({ renderLatexText: true })
      if (disposed) destroyFormulaRendering()
    }
  } catch (cause) {
    if (!disposed) error.value = `编辑器加载失败：${cause instanceof Error ? cause.message : String(cause)}`
  }
})

onBeforeUnmount(() => {
  disposed = true
  formulaDialog?.destroy()
  editor?.remove()
  destroyFormulaRendering()
})
</script>

<template>
  <main class="workspace">
    <header class="page-header">
      <div>
        <p class="eyebrow">VUE 3 · TINYMCE 6.0</p>
        <h1>LaTeX 公式编辑器</h1>
        <p class="description">编辑正文，点击「插入公式」添加公式，双击已有公式可再次修改。</p>
      </div>
      <div class="page-actions">
        <label>导出格式
          <select v-model="exportFormat" aria-label="导出格式">
            <option value="html">HTML（保留公式图片）</option>
            <option value="latex">LaTeX（公式转为源码）</option>
          </select>
        </label>
        <button type="button" class="export-button" :disabled="!!error" @click="exportContent">导出内容</button>
      </div>
    </header>
    <p v-if="error" role="alert" class="error">{{ error }}</p>
    <div class="formula-options">
      <label>SDK 返回形式
        <select v-model="outputType" aria-label="SDK 返回形式">
          <option value="latex">LaTeX 公式</option>
          <option value="img">IMG 图片（PNG）</option>
        </select>
      </label>
      <span>用于新插入的公式；双击修改时保留原形式。图片包含 LaTeX 源码，可再次编辑。</span>
    </div>
    <section class="editor-card" aria-label="富文本编辑器">
      <div ref="toolbarElement" class="editor-toolbar"></div>
      <div ref="editorElement" class="editor-body" aria-label="正文">
        <h2>在正文中书写公式</h2>
        <p>这里可以输入文字、设置格式，也可以插入数学、物理和化学公式。</p>
        <p>例如：求方程 $x^2-2x-3=0$ 的根。</p>
      </div>
    </section>
    <section v-if="savedContent" class="export-panel">
      <h2>导出结果</h2>
      <textarea aria-label="导出内容" :value="savedContent" readonly spellcheck="false"></textarea>
    </section>
  </main>
</template>
