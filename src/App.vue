<script setup>
import { ref, computed, watch, onMounted, nextTick } from 'vue'

// 1. ESTADO DE LA APLICACIÓN
const templates = ref([])
const activeIndex = ref(null)
const variableValues = ref({})
const isSidebarOpen = ref(true)
const currentMode = ref('form') // 'form' o 'config'

const editorRef = ref(null)
const isDraggingOver = ref(false)

// Archivo para importar
const fileInputRef = ref(null)

// Modal de exportación
const isExportModalOpen = ref(false)
const exportScope = ref('all')

// ================= CAMPOS DISPONIBLES =================
const availableFieldTypes = [
  { id: 'texto', label: 'Texto', desc: 'Entrada de texto libre para nombres o datos generales.' },
  { id: 'dinero', label: 'Dinero', desc: 'Monto numérico acompañado del símbolo de moneda $.' },
  { id: 'fecha', label: 'Fecha', desc: 'Selector de calendario para escoger día, mes y año.' },
  { id: 'hora', label: 'Hora', desc: 'Selector de horario para citas y turnos específicos.' },
  { id: 'numero', label: 'Número', desc: 'Campo numérico para cantidades, días o porcentajes.' },
  { id: 'opciones', label: 'Opciones', desc: 'Menú desplegable con alternativas predeterminadas.' }
]

// Modal simplificado de campo
const isFieldModalOpen = ref(false)
const selectedFieldType = ref(null)
const fieldNameInput = ref('')
const fieldNameInputRef = ref(null)

const optionsList = ref([
  { text: '12.00', isDefault: false },
  { text: '24.00', isDefault: true },
  { text: '48.00', isDefault: false }
])

let savedRange = null
let editingBadgeElement = null
let dropIndicatorNode = null

// ================= FORMATEADORES DE DINERO =================
const handleMoneyInput = (varName, event) => {
  let val = event.target.value.replace(/[^0-9.]/g, '')
  const parts = val.split('.')
  
  if (parts.length > 2) {
    val = parts[0] + '.' + parts.slice(1).join('')
  }
  if (parts[1] && parts[1].length > 2) {
    val = parts[0] + '.' + parts[1].slice(0, 2)
  }

  const intFormatted = parts[0].replace(/\B(?=(\d{3})+(?!\d))/g, ',')
  variableValues.value[varName] = parts.length > 1 ? `${intFormatted}.${parts[1]}` : intFormatted
}

const handleMoneyBlur = (varName) => {
  const val = variableValues.value[varName]
  if (!val) return
  const clean = String(val).replace(/,/g, '')
  const num = parseFloat(clean)
  if (!isNaN(num)) {
    variableValues.value[varName] = num.toLocaleString('en-US', {
      minimumFractionDigits: 2,
      maximumFractionDigits: 2
    })
  }
}

// ================= IMPORTAR / EXPORTAR =================
const triggerImport = () => {
  fileInputRef.value?.click()
}

const handleFileImport = (event) => {
  const file = event.target.files?.[0]
  if (!file) return

  const reader = new FileReader()
  reader.onload = (e) => {
    try {
      const data = JSON.parse(e.target.result)
      
      if (Array.isArray(data)) {
        const valid = data.filter(t => t && typeof t.content === 'string')
        if (valid.length > 0) {
          templates.value = [...valid, ...templates.value]
          activeIndex.value = 0
          showToast({ type: 'success', title: `${valid.length} plantilla(s) importada(s)` })
        } else {
          showToast({ type: 'error', title: 'El archivo no contiene plantillas válidas' })
        }
      } else if (data && typeof data.content === 'string') {
        templates.value.unshift({
          title: data.title || 'Plantilla Importada',
          content: data.content
        })
        activeIndex.value = 0
        showToast({ type: 'success', title: 'Plantilla importada con éxito' })
      } else {
        showToast({ type: 'error', title: 'Formato de archivo incompatible' })
      }
    } catch (err) {
      showToast({ type: 'error', title: 'Error al procesar el archivo JSON' })
    }
    event.target.value = ''
  }
  reader.readAsText(file)
}

const executeExport = () => {
  if (exportScope.value === 'current' && !currentTemplate.value) {
    showToast({ type: 'error', title: 'No hay ninguna plantilla seleccionada' })
    return
  }

  const exportData = exportScope.value === 'all' 
    ? templates.value 
    : [currentTemplate.value]

  const dataStr = 'data:text/json;charset=utf-8,' + encodeURIComponent(JSON.stringify(exportData, null, 2))
  const dlAnchor = document.createElement('a')
  dlAnchor.setAttribute('href', dataStr)

  const filename = exportScope.value === 'all'
    ? 'plantillas_todas.json'
    : `plantilla_${(currentTemplate.value?.title || 'export').toLowerCase().replace(/\s+/g, '_')}.json`

  dlAnchor.setAttribute('download', filename)
  document.body.appendChild(dlAnchor)
  dlAnchor.click()
  dlAnchor.remove()

  isExportModalOpen.value = false
  showToast({ type: 'success', title: 'Descarga completada' })
}

// ================= SERIALIZADOR =================
const templateToHtml = (content) => {
  if (!content) return ''
  const escaped = content
    .replace(/&/g, '&amp;')
    .replace(/</g, '&lt;')
    .replace(/>/g, '&gt;')
    .replace(/\n/g, '<br>')

  const regex = /\{\{(?:([a-zA-Z]+):)?([^:{}]+)(?::([^:{}]+))?(?::([^:{}]+))?\}\}/g
  return escaped.replace(regex, (match, type, name, options, defaultVal) => {
    const fieldType = type || 'texto'
    const typeConfig = availableFieldTypes.find(f => f.id === fieldType) || availableFieldTypes[0]
    return createBadgeHtml(typeConfig, name.trim(), options ? options.trim() : '', defaultVal ? defaultVal.trim() : '')
  })
}

const createBadgeHtml = (typeConfig, name, options = '', defaultVal = '') => {
  return `<span contenteditable="false" class="field-chip inline-flex items-center gap-1 px-2 h-[20px] mx-0.5 rounded text-[11px] font-semibold bg-slate-100 text-slate-800 border border-slate-200 select-none cursor-pointer align-middle transition-colors hover:bg-slate-200/80 leading-none" data-var-name="${name}" data-var-type="${typeConfig.id}" data-var-options="${options}" data-var-default="${defaultVal}"><span class="chip-name font-mono pointer-events-none">${name}</span><span class="text-slate-400 font-normal mx-0.5 pointer-events-none select-none">|</span><span class="chip-type text-slate-500 font-normal text-[10px] pointer-events-none select-none">${typeConfig.label.toLowerCase()}</span><span class="chip-remove text-slate-400 hover:text-slate-800 font-bold ml-0.5 text-xs leading-none cursor-pointer" title="Eliminar campo">×</span></span>`
}

const domToTemplateString = (el) => {
  if (!el) return ''
  let text = ''

  const traverse = (node) => {
    if (node.nodeType === Node.TEXT_NODE) {
      text += node.textContent
    } else if (node.nodeType === Node.ELEMENT_NODE) {
      if (node.id === 'drag-drop-caret') return
      if (node.classList && node.classList.contains('field-chip')) {
        const varName = node.getAttribute('data-var-name') || ''
        const varType = node.getAttribute('data-var-type') || 'texto'
        const varOptions = node.getAttribute('data-var-options') || ''
        const varDefault = node.getAttribute('data-var-default') || ''
        
        if (varType === 'opciones' && varOptions) {
          text += `{{opciones:${varName}:${varOptions}:${varDefault}}}`
        } else {
          text += `{{${varType}:${varName}}}`
        }
        return
      }
      if (node.tagName === 'BR') {
        text += '\n'
        return
      }
      for (let child of node.childNodes) {
        traverse(child)
      }
      if (node.tagName === 'DIV' || node.tagName === 'P') {
        text += '\n'
      }
    }
  }

  for (let child of el.childNodes) {
    traverse(child)
  }
  return text.replace(/\u00A0/g, ' ')
}

const syncEditorToTemplate = () => {
  if (!editorRef.value || !currentTemplate.value) return
  currentTemplate.value.content = domToTemplateString(editorRef.value)
}

const syncTemplateToEditor = () => {
  if (!editorRef.value || !currentTemplate.value) return
  editorRef.value.innerHTML = templateToHtml(currentTemplate.value.content)
}

// ================= DRAG & DROP =================
const onDragStartBadge = (e, fieldType) => {
  e.dataTransfer.setData('application/json', JSON.stringify(fieldType))
  e.dataTransfer.effectAllowed = 'copy'
}

const getDropRange = (e) => {
  if (document.caretRangeFromPoint) {
    return document.caretRangeFromPoint(e.clientX, e.clientY)
  } else if (document.caretPositionFromPoint) {
    const pos = document.caretPositionFromPoint(e.clientX, e.clientY)
    if (pos) {
      const r = document.createRange()
      r.setStart(pos.offsetNode, pos.offset)
      r.collapse(true)
      return r
    }
  }
  return null
}

const removeDropIndicator = () => {
  if (dropIndicatorNode && dropIndicatorNode.parentNode) {
    dropIndicatorNode.remove()
  }
  dropIndicatorNode = null
}

const onEditorDragOver = (e) => {
  e.preventDefault()
  isDraggingOver.value = true

  const range = getDropRange(e)
  if (!range || !editorRef.value?.contains(range.startContainer)) return

  if (!dropIndicatorNode) {
    dropIndicatorNode = document.createElement('span')
    dropIndicatorNode.id = 'drag-drop-caret'
    dropIndicatorNode.className = 'inline-block w-[2px] h-4 bg-slate-800 animate-pulse align-middle mx-1 rounded-full pointer-events-none'
  }

  if (dropIndicatorNode.nextSibling !== range.startContainer.childNodes?.[range.startOffset]) {
    range.insertNode(dropIndicatorNode)
  }
}

const onEditorDragLeave = (e) => {
  if (!editorRef.value?.contains(e.relatedTarget)) {
    isDraggingOver.value = false
    removeDropIndicator()
  }
}

const onEditorDrop = (e) => {
  e.preventDefault()
  isDraggingOver.value = false
  const data = e.dataTransfer.getData('application/json')
  if (!data) {
    removeDropIndicator()
    return
  }
  const fieldType = JSON.parse(data)

  if (dropIndicatorNode && dropIndicatorNode.parentNode) {
    const range = document.createRange()
    range.setStartBefore(dropIndicatorNode)
    range.collapse(true)
    savedRange = range
    removeDropIndicator()
  } else {
    savedRange = getDropRange(e)
  }

  openModalForNewField(fieldType)
}

const onClickBadgeButton = (fieldType) => {
  const sel = window.getSelection()
  if (sel.rangeCount > 0 && editorRef.value?.contains(sel.getRangeAt(0).commonAncestorContainer)) {
    savedRange = sel.getRangeAt(0)
  } else {
    savedRange = null
  }
  openModalForNewField(fieldType)
}

const openModalForNewField = (fieldType) => {
  editingBadgeElement = null
  selectedFieldType.value = fieldType
  fieldNameInput.value = ''
  
  if (fieldType.id === 'opciones') {
    optionsList.value = [
      { text: '12.00', isDefault: false },
      { text: '24.00', isDefault: true },
      { text: '48.00', isDefault: false }
    ]
  }

  isFieldModalOpen.value = true
  nextTick(() => {
    fieldNameInputRef.value?.focus()
  })
}

const addOption = () => {
  optionsList.value.push({ text: '', isDefault: optionsList.value.length === 0 })
}

const removeOption = (index) => {
  if (optionsList.value.length <= 1) return
  const wasDefault = optionsList.value[index].isDefault
  optionsList.value.splice(index, 1)
  if (wasDefault && optionsList.value.length > 0) {
    optionsList.value[0].isDefault = true
  }
}

const setDefaultOption = (index) => {
  optionsList.value.forEach((opt, idx) => {
    opt.isDefault = idx === index
  })
}

const onEditorClick = (e) => {
  if (e.target.classList.contains('chip-remove')) {
    e.stopPropagation()
    const chip = e.target.closest('.field-chip')
    if (chip) {
      const prev = chip.previousSibling
      const next = chip.nextSibling

      if (next && next.nodeType === Node.TEXT_NODE) {
        if ((prev && prev.nodeType === Node.TEXT_NODE && /[\s\u00A0]$/.test(prev.textContent)) ||
            /^[\s\u00A0]*[,.\?!:;]/.test(next.textContent)) {
          next.textContent = next.textContent.replace(/^[\s\u00A0]+/, '')
        }
      }

      chip.remove()
      if (editorRef.value) editorRef.value.normalize()
      syncEditorToTemplate()
      showToast({ type: 'success', title: 'Campo eliminado' })
    }
    return
  }

  const chip = e.target.closest('.field-chip')
  if (chip) {
    e.stopPropagation()
    editingBadgeElement = chip
    const varName = chip.getAttribute('data-var-name')
    const typeId = chip.getAttribute('data-var-type') || 'texto'
    const varOptions = chip.getAttribute('data-var-options') || ''
    const varDefault = chip.getAttribute('data-var-default') || ''

    selectedFieldType.value = availableFieldTypes.find(f => f.id === typeId) || availableFieldTypes[0]
    fieldNameInput.value = varName

    if (typeId === 'opciones' && varOptions) {
      const opts = varOptions.split(',')
      optionsList.value = opts.map(opt => ({
        text: opt,
        isDefault: opt === varDefault
      }))
    }

    isFieldModalOpen.value = true
    nextTick(() => {
      fieldNameInputRef.value?.focus()
      fieldNameInputRef.value?.select()
    })
  }
}

const confirmFieldInsertion = () => {
  if (!fieldNameInput.value.trim() || !editorRef.value) return
  const cleanName = fieldNameInput.value.trim().toLowerCase().replace(/\s+/g, '_')

  let cleanOptions = ''
  let defaultVal = ''
  if (selectedFieldType.value?.id === 'opciones') {
    const validOpts = optionsList.value.filter(o => o.text.trim())
    cleanOptions = validOpts.map(o => o.text.trim()).join(',')
    const defOpt = validOpts.find(o => o.isDefault) || validOpts[0]
    defaultVal = defOpt ? defOpt.text.trim() : ''
  }

  if (editingBadgeElement) {
    editingBadgeElement.setAttribute('data-var-name', cleanName)
    editingBadgeElement.setAttribute('data-var-options', cleanOptions)
    editingBadgeElement.setAttribute('data-var-default', defaultVal)
    const label = editingBadgeElement.querySelector('.chip-name')
    if (label) label.textContent = cleanName
    const typeLabel = editingBadgeElement.querySelector('.chip-type')
    if (typeLabel) typeLabel.textContent = selectedFieldType.value.label.toLowerCase()
    
    syncEditorToTemplate()
    isFieldModalOpen.value = false
    showToast({ type: 'success', title: `Renombrado a ${cleanName}` })
    return
  }

  const tempDiv = document.createElement('div')
  tempDiv.innerHTML = createBadgeHtml(selectedFieldType.value, cleanName, cleanOptions, defaultVal)
  const chipNode = tempDiv.firstChild
  const spaceNode = document.createTextNode(' ')

  if (savedRange && editorRef.value.contains(savedRange.commonAncestorContainer)) {
    savedRange.deleteContents()
    savedRange.insertNode(chipNode)
    chipNode.parentNode.insertBefore(spaceNode, chipNode.nextSibling)
  } else {
    editorRef.value.appendChild(chipNode)
    editorRef.value.appendChild(spaceNode)
  }

  syncEditorToTemplate()
  isFieldModalOpen.value = false
  showToast({ type: 'success', title: `Campo ${cleanName} añadido` })

  nextTick(() => {
    if (!editorRef.value) return
    editorRef.value.focus()

    const range = document.createRange()
    range.setStartAfter(spaceNode)
    range.collapse(true)

    const sel = window.getSelection()
    sel.removeAllRanges()
    sel.addRange(range)
  })
}

// ================= SISTEMA DE TOAST =================
const toast = ref({ show: false, type: 'success', title: '', description: '', onConfirm: null, autoClose: true })
const progress = ref(100)
let toastInterval = null

const showToast = ({ type = 'success', title, description = '', onConfirm = null }) => {
  if (toastInterval) clearInterval(toastInterval)
  const totalDuration = description ? 5000 : 2500
  const step = 50
  let elapsed = 0
  progress.value = 100
  toast.value = { show: true, type, title, description, onConfirm, autoClose: type !== 'confirm' }

  if (toast.value.autoClose) {
    toastInterval = setInterval(() => {
      elapsed += step
      progress.value = Math.max(0, 100 - (elapsed / totalDuration) * 100)
      if (elapsed >= totalDuration) closeToast()
    }, step)
  }
}

const closeToast = () => {
  if (toastInterval) clearInterval(toastInterval)
  toast.value.show = false
}

const confirmToastAction = () => {
  if (toast.value.onConfirm) toast.value.onConfirm()
  closeToast()
}

// ================= FOCO AUTOMÁTICO EN EL PRIMER CAMPO =================
const focusFirstFormField = () => {
  nextTick(() => {
    const container = document.getElementById('form-message-container')
    if (container) {
      const firstInput = container.querySelector('input, select')
      if (firstInput) {
        firstInput.focus()
      }
    }
  })
}

// ================= CARGAR AL INICIAR =================
onMounted(() => {
  const saved = localStorage.getItem('mis_plantillas_v15')
  if (saved) {
    templates.value = JSON.parse(saved)
  } else {
    // Plantilla inicial simple y minimalista
    templates.value = [
      {
        title: 'Plantilla de Ejemplo',
        content: 'Hola {{texto:nombre}}, escribe tu mensaje aquí...'
      }
    ]
    saveToLocalStorage()
  }
  activeIndex.value = 0
  nextTick(() => {
    syncTemplateToEditor()
    if (currentMode.value === 'form') focusFirstFormField()
  })
})

const saveToLocalStorage = () => {
  localStorage.setItem('mis_plantillas_v15', JSON.stringify(templates.value))
}

watch(() => templates.value, () => saveToLocalStorage(), { deep: true })

const currentTemplate = computed(() => {
  if (activeIndex.value !== null && templates.value[activeIndex.value]) {
    return templates.value[activeIndex.value]
  }
  return null
})

watch(activeIndex, () => {
  variableValues.value = {}
  nextTick(() => {
    syncTemplateToEditor()
    if (currentMode.value === 'form') focusFirstFormField()
  })
})

watch(currentMode, (newMode) => {
  if (newMode === 'config') {
    nextTick(() => syncTemplateToEditor())
  } else if (newMode === 'form') {
    focusFirstFormField()
  }
})

// ================= MOTOR DE PLANTILLAS =================
const parsedTokens = computed(() => {
  if (!currentTemplate.value?.content) return []
  const content = currentTemplate.value.content
  const regex = /\{\{(?:([a-zA-Z]+):)?([^:{}]+)(?::([^:{}]+))?(?::([^:{}]+))?\}\}/g
  const tokens = []
  let lastIndex = 0
  let match

  while ((match = regex.exec(content)) !== null) {
    if (match.index > lastIndex) {
      tokens.push({ type: 'text', value: content.slice(lastIndex, match.index) })
    }
    const fieldType = match[1] || 'texto'
    const varName = match[2].trim()
    const optionsRaw = match[3] || ''
    const defaultVal = match[4] || ''
    const options = optionsRaw ? optionsRaw.split(',').map(o => o.trim()).filter(Boolean) : []

    if (defaultVal && variableValues.value[varName] === undefined) {
      variableValues.value[varName] = defaultVal
    }

    tokens.push({ 
      type: 'variable', 
      fieldType,
      name: varName,
      options,
      defaultVal
    })
    lastIndex = regex.lastIndex
  }
  if (lastIndex < content.length) {
    tokens.push({ type: 'text', value: content.slice(lastIndex) })
  }
  return tokens
})

const extractedVariables = computed(() => {
  if (!currentTemplate.value) return []
  const regex = /\{\{(?:([a-zA-Z]+):)?([^:{}]+)(?::([^:{}]+))?(?::([^:{}]+))?\}\}/g
  const matches = [...currentTemplate.value.content.matchAll(regex)]
  return [...new Set(matches.map(m => m[2].trim()))] 
})

// Mensaje final
const finalMessage = computed(() => {
  if (!currentTemplate.value) return ''
  let msg = currentTemplate.value.content
  const regex = /\{\{(?:([a-zA-Z]+):)?([^:{}]+)(?::([^:{}]+))?(?::([^:{}]+))?\}\}/g

  msg = msg.replace(regex, (match, type, varName) => {
    const rawVal = variableValues.value[varName]
    if (rawVal === undefined || rawVal === '') return `[${varName}]`

    const fieldType = type || 'texto'

    if (fieldType === 'dinero') {
      const clean = String(rawVal).replace(/,/g, '')
      const num = parseFloat(clean)
      if (!isNaN(num)) {
        const formatted = num.toLocaleString('en-US', {
          minimumFractionDigits: 2,
          maximumFractionDigits: 2
        })
        return `$${formatted}`
      }
      return `$${rawVal}`
    } else if (fieldType === 'fecha') {
      if (/^\d{4}-\d{2}-\d{2}$/.test(rawVal)) {
        const [y, m, d] = rawVal.split('-')
        return `${d}/${m}/${y}`
      }
      return rawVal
    }
    return rawVal
  })

  return msg.replace(/\$\$/g, '$')
})

const getInputWidth = (varName) => {
  const textLen = (variableValues.value[varName] || varName || '').length
  return `${Math.max(75, Math.min(240, textLen * 8 + 20))}px`
}

const createNewTemplate = () => {
  templates.value.unshift({ 
    title: 'Nueva Plantilla',
    content: 'Hola {{texto:nombre}}, escribe tu mensaje aquí...'
  })
  activeIndex.value = 0
  currentMode.value = 'config'
  if (!isSidebarOpen.value) isSidebarOpen.value = true
  nextTick(() => syncTemplateToEditor())
}

const deleteTemplate = (index) => {
  showToast({
    type: 'confirm',
    title: '¿Eliminar plantilla?',
    description: 'Esta acción no se puede deshacer y se borrará de tu memoria local.',
    onConfirm: () => {
      templates.value.splice(index, 1)
      activeIndex.value = templates.value.length > 0 ? 0 : null
      nextTick(() => syncTemplateToEditor())
    }
  })
}

const clearFields = () => {
  variableValues.value = {}
  showToast({ type: 'success', title: 'Campos limpiados' })
  focusFirstFormField()
}

const copyClipboard = async () => {
  if (!finalMessage.value) return
  try {
    await navigator.clipboard.writeText(finalMessage.value)
    showToast({ type: 'success', title: '¡Mensaje copiado al portapapeles!' })
  } catch (err) {
    showToast({ type: 'error', title: 'Error al copiar', description: 'Copia el texto manualmente.' })
  }
}
</script>

<template>
  <div class="min-h-screen bg-[#e8ecf1] p-4 lg:p-8 flex items-center justify-center font-sans text-slate-800">
    
    <!-- Input oculto para importar JSON -->
    <input 
      type="file" 
      ref="fileInputRef" 
      accept=".json" 
      @change="handleFileImport" 
      class="hidden" 
    />

    <div class="w-full max-w-[1350px] h-[90vh] bg-white rounded-[2.5rem] shadow-2xl overflow-hidden flex relative">
      
      <!-- ================= 1. BARRA LATERAL IZQUIERDA (PLANTILLAS + IMPORT/EXPORT) ================= -->
      <aside 
        class="bg-[#f9fafc] border-r border-slate-100 flex flex-col flex-shrink-0 transition-all duration-300 ease-in-out relative z-10 overflow-hidden"
        :class="isSidebarOpen ? 'w-80' : 'w-0'"
      >
        <div class="w-80 flex flex-col h-full">
          <!-- Cabecera -->
          <div class="h-24 px-6 flex items-center justify-between border-b border-slate-100/60">
            <div>
              <h2 class="text-base font-bold text-slate-800">Tus Plantillas</h2>
              <p class="text-[11px] text-slate-400 mt-0.5">{{ templates.length }} guardadas</p>
            </div>
            
            <div class="flex items-center gap-1.5">
              <button 
                @click="createNewTemplate"
                class="w-7 h-7 rounded-full bg-[#1a1b26] text-white flex items-center justify-center hover:bg-slate-800 transition-transform hover:scale-105 active:scale-95 cursor-pointer shadow-sm"
                title="Nueva Plantilla"
              >
                <svg class="w-3.5 h-3.5 pointer-events-none" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2.5" d="M12 4v16m8-8H4"></path></svg>
              </button>

              <button 
                @click="isSidebarOpen = false"
                class="w-7 h-7 rounded-full hover:bg-slate-200 text-slate-400 hover:text-slate-700 flex items-center justify-center transition-colors cursor-pointer"
                title="Ocultar menú"
              >
                <svg class="w-3.5 h-3.5 pointer-events-none" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M11 19l-7-7 7-7m8 14l-7-7 7-7"></path></svg>
              </button>
            </div>
          </div>

          <!-- Barra de Importar y Exportar -->
          <div class="px-6 py-2 border-b border-slate-100/80 bg-white flex items-center justify-between text-xs font-semibold text-slate-600">
            <button 
              @click="triggerImport" 
              class="hover:text-slate-900 transition-colors flex items-center gap-1 cursor-pointer py-1"
              title="Importar archivo JSON"
            >
              <svg class="w-3.5 h-3.5 text-slate-500" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M4 16v1a3 3 0 003 3h10a3 3 0 003-3v-1m-4-8l-4-4m0 0L8 8m4-4v12"></path></svg>
              Importar
            </button>
            
            <span class="text-slate-200">|</span>
            
            <button 
              @click="isExportModalOpen = true" 
              class="hover:text-slate-900 transition-colors flex items-center gap-1 cursor-pointer py-1"
              title="Exportar archivo JSON"
            >
              <svg class="w-3.5 h-3.5 text-slate-500" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M4 16v1a3 3 0 003 3h10a3 3 0 003-3v-1m-4-4l-4 4m0 0l-4-4m4 4V4"></path></svg>
              Exportar
            </button>
          </div>

          <!-- Lista de Plantillas -->
          <div class="flex-1 overflow-y-auto p-4 space-y-1.5">
            <div 
              v-for="(tpl, index) in templates" :key="index"
              @click="activeIndex = index"
              class="group px-4 py-3 rounded-2xl cursor-pointer transition-all duration-300 flex justify-between items-center"
              :class="activeIndex === index 
                ? 'bg-white shadow-[0_10px_30px_-10px_rgba(0,0,0,0.08)] border border-slate-100' 
                : 'hover:bg-white/60 border border-transparent'"
            >
              <span class="font-bold text-slate-800 truncate text-sm flex-1">{{ tpl.title || 'Plantilla Sin Nombre' }}</span>
              
              <button 
                @click.stop="deleteTemplate(index)"
                class="text-slate-300 hover:text-red-500 transition-colors p-1 cursor-pointer hover:scale-110 active:scale-95 ml-2"
                title="Eliminar plantilla"
              >
                <svg class="w-4 h-4 pointer-events-none" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M19 7l-.867 12.142A2 2 0 0116.138 21H7.862a2 2 0 01-1.995-1.858L5 7m5 4v6m4-6v6m1-10V4a1 1 0 00-1-1h-4a1 1 0 00-1 1v3M4 7h16"></path></svg>
              </button>
            </div>
          </div>
        </div>
      </aside>

      <!-- ================= 2. ZONA PRINCIPAL ================= -->
      <main class="flex-1 bg-white flex flex-col h-full relative z-0 min-w-0">
        
        <!-- Header Top -->
        <header class="h-24 px-8 lg:px-10 flex items-center justify-between border-b border-slate-50 flex-shrink-0">
          <div class="flex items-center gap-4">
            <button 
              v-if="!isSidebarOpen"
              @click="isSidebarOpen = true"
              class="w-9 h-9 rounded-full border border-slate-200 flex items-center justify-center text-slate-600 hover:bg-slate-50 transition-all cursor-pointer active:scale-95 shadow-sm"
              title="Mostrar Plantillas"
            >
              <svg class="w-4 h-4 pointer-events-none" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M4 6h16M4 12h16M4 18h16"></path></svg>
            </button>

            <div>
              <h1 class="text-xl font-bold text-slate-800 truncate max-w-xs md:max-w-md">
                {{ currentTemplate?.title || 'App Mensajes' }}
              </h1>
              <p class="text-xs text-slate-400 mt-0.5">
                {{ currentMode === 'form' ? 'Modo Formulario' : 'Modo Configuración' }}
              </p>
            </div>
          </div>
          
          <!-- Selector de Modo -->
          <div class="flex items-center gap-3">
            <div class="bg-slate-100 p-1 rounded-2xl flex items-center gap-1 border border-slate-200/60 shadow-inner">
              <button 
                @click="currentMode = 'form'"
                class="px-4 py-2 rounded-xl text-xs font-bold transition-all cursor-pointer flex items-center gap-1.5"
                :class="currentMode === 'form' 
                  ? 'bg-white text-slate-900 shadow-sm' 
                  : 'text-slate-500 hover:text-slate-800'"
              >
                <svg class="w-3.5 h-3.5" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9 12h6m-6 4h6m2 5H7a2 2 0 01-2-2V5a2 2 0 012-2h5.586a1 1 0 01.707.293l5.414 5.414a1 1 0 01.293.707V19a2 2 0 01-2 2z"></path></svg>
                Formulario
              </button>

              <button 
                @click="currentMode = 'config'"
                class="px-4 py-2 rounded-xl text-xs font-bold transition-all cursor-pointer flex items-center gap-1.5"
                :class="currentMode === 'config' 
                  ? 'bg-white text-slate-900 shadow-sm' 
                  : 'text-slate-500 hover:text-slate-800'"
              >
                <svg class="w-3.5 h-3.5" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M10.325 4.317c.426-1.756 2.924-1.756 3.35 0a1.724 1.724 0 002.573 1.066c1.543-.94 3.31.826 2.37 2.37a1.724 1.724 0 001.065 2.572c1.756.426 1.756 2.924 0 3.35a1.724 1.724 0 00-1.066 2.573c.94 1.543-.826 3.31-2.37 2.37a1.724 1.724 0 00-2.572 1.065c-.426 1.756-2.924 1.756-3.35 0a1.724 1.724 0 00-2.573-1.066c-1.543.94-3.31-.826-2.37-2.37a1.724 1.724 0 00-1.065-2.572c-1.756-.426-1.756-2.924 0-3.35a1.724 1.724 0 001.066-2.573c-.94-1.543.826-3.31 2.37-2.37.996.608 2.296.07 2.572-1.065z"></path><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M15 12a3 3 0 11-6 0 3 3 0 016 0z"></path></svg>
                Configurar
              </button>
            </div>
          </div>
        </header>

        <!-- Contenido Central -->
        <div class="flex-1 overflow-y-auto px-8 lg:px-10 py-8" v-if="currentTemplate">
          
          <!-- ================= MODO A: FORMULARIO ================= -->
          <div v-if="currentMode === 'form'" class="space-y-6">
            <div class="bg-white rounded-[2rem] p-8 shadow-[0_8px_30px_rgb(0,0,0,0.04)] border border-slate-100 flex flex-col justify-between min-h-[380px]">
              <div>
                <div class="mb-5 pb-3 border-b border-slate-100/60">
                  <p class="text-xs font-medium text-slate-500">Completa los campos integrados en el texto.</p>
                </div>

                <!-- CONTENEDOR CON LEADING-7 (28px) -->
                <div id="form-message-container" class="text-slate-700 text-sm leading-7 whitespace-pre-wrap font-normal">
                  <template v-for="(token, idx) in parsedTokens" :key="idx">
                    <span v-if="token.type === 'text'">{{ token.value }}</span>

                    <!-- DINERO -->
                    <span v-else-if="token.fieldType === 'dinero'" class="inline-flex items-center px-2 h-[26px] mx-0.5 bg-slate-50 border border-slate-200 hover:border-slate-300 rounded-md text-xs font-semibold text-slate-800 shadow-xs focus-within:ring-2 focus-within:ring-slate-200 focus-within:border-slate-400 focus-within:bg-white transition-all align-middle">
                      <span class="text-xs font-bold text-slate-400 mr-1 select-none">$</span>
                      <input 
                        :value="variableValues[token.name]"
                        @input="handleMoneyInput(token.name, $event)"
                        @blur="handleMoneyBlur(token.name)"
                        type="text"
                        inputmode="decimal"
                        :placeholder="token.name"
                        :style="{ width: getInputWidth(token.name) }"
                        class="bg-transparent border-none outline-none text-slate-800 text-xs font-semibold placeholder:text-slate-400 text-left min-w-[55px]"
                      />
                    </span>

                    <!-- FECHA DATEPICKER -->
                    <span v-else-if="token.fieldType === 'fecha'" class="inline-block mx-0.5 align-middle">
                      <input 
                        v-model="variableValues[token.name]"
                        type="date"
                        class="px-2 h-[26px] bg-slate-50 border border-slate-200 hover:border-slate-300 rounded-md text-xs font-semibold text-slate-800 outline-none focus:bg-white focus:ring-2 focus:ring-slate-200 focus:border-slate-400 shadow-xs cursor-pointer"
                      />
                    </span>

                    <!-- OPCIONES MÚLTIPLES (SELECT) -->
                    <span v-else-if="token.fieldType === 'opciones'" class="inline-block mx-0.5 align-middle">
                      <select 
                        v-model="variableValues[token.name]"
                        class="px-2 h-[26px] bg-slate-50 border border-slate-200 hover:border-slate-300 rounded-md text-xs font-semibold text-slate-800 outline-none focus:bg-white focus:ring-2 focus:ring-slate-200 focus:border-slate-400 shadow-xs cursor-pointer"
                      >
                        <option value="" disabled>{{ token.name }}</option>
                        <option v-for="opt in token.options" :key="opt" :value="opt">{{ opt }}</option>
                      </select>
                    </span>

                    <!-- HORA -->
                    <span v-else-if="token.fieldType === 'hora'" class="inline-block mx-0.5 align-middle">
                      <input 
                        v-model="variableValues[token.name]"
                        type="time"
                        class="px-2 h-[26px] bg-slate-50 border border-slate-200 hover:border-slate-300 rounded-md text-xs font-semibold text-slate-800 outline-none focus:bg-white focus:ring-2 focus:ring-slate-200 focus:border-slate-400 shadow-xs cursor-pointer"
                      />
                    </span>

                    <!-- NÚMERO / TEXTO -->
                    <span v-else class="inline-block mx-0.5 align-middle">
                      <input 
                        v-model="variableValues[token.name]"
                        :type="token.fieldType === 'numero' ? 'number' : 'text'"
                        :placeholder="token.name"
                        :style="{ width: getInputWidth(token.name) }"
                        class="px-2 h-[26px] bg-slate-50 border border-slate-200 hover:border-slate-300 rounded-md text-xs font-semibold text-slate-800 text-center outline-none focus:bg-white focus:ring-2 focus:ring-slate-200 focus:border-slate-400 shadow-xs transition-all placeholder:text-slate-400"
                      />
                    </span>
                  </template>
                </div>
              </div>

              <!-- Pie de Acciones -->
              <div class="mt-8 pt-6 border-t border-slate-100/80 flex items-center justify-between">
                <button 
                  v-if="extractedVariables.length > 0"
                  @click="clearFields"
                  class="text-xs font-semibold text-slate-400 hover:text-slate-700 transition-colors cursor-pointer px-3 py-2 rounded-xl hover:bg-slate-50 border border-transparent hover:border-slate-200"
                >
                  Limpiar campos
                </button>
                <div v-else></div>

                <button 
                  @click="copyClipboard"
                  class="bg-[#1a1b26] cursor-pointer text-white font-bold py-3 px-8 rounded-xl hover:bg-slate-800 transition-all hover:scale-105 active:scale-95 shadow-md flex items-center justify-center gap-2 text-sm"
                >
                  Copiar Mensaje
                  <svg class="w-4 h-4 pointer-events-none" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M8 5H6a2 2 0 00-2 2v12a2 2 0 002 2h10a2 2 0 002-2v-1M8 5a2 2 0 002 2h2a2 2 0 002-2M8 5a2 2 0 012-2h2a2 2 0 012 2m0 0h2a2 2 0 012 2v3m2 4H10m0 0l3-3m-3 3l3 3"></path></svg>
                </button>
              </div>
            </div>
          </div>

          <!-- ================= MODO B: CONFIGURACIÓN ================= -->
          <div v-else class="flex flex-col lg:flex-row gap-6">
            
            <!-- Columna Izquierda: Editor -->
            <div class="flex-1 bg-white rounded-[2rem] p-8 shadow-[0_8px_30px_rgb(0,0,0,0.04)] border border-slate-100">
              
              <div class="flex items-center justify-between mb-6 pb-4 border-b border-slate-100/60">
                <h3 class="font-bold text-base text-slate-800">Editor de Plantilla</h3>
                <button 
                  @click="currentMode = 'form'"
                  class="bg-slate-900 text-white text-xs font-semibold px-4 py-2 rounded-xl hover:bg-slate-800 transition-all cursor-pointer shadow-xs"
                >
                  Probar en Formulario →
                </button>
              </div>

              <!-- Título -->
              <div class="mb-6">
                <label class="block text-xs font-bold text-slate-400 uppercase tracking-wider mb-2">Nombre de la plantilla</label>
                <input 
                  v-model="currentTemplate.title" 
                  class="w-full text-xl font-bold text-slate-800 bg-slate-50/70 border border-slate-200/80 rounded-2xl px-5 py-3 outline-none focus:bg-white focus:ring-2 focus:ring-slate-200 transition-all placeholder-slate-300"
                  placeholder="Ej: Mensaje de bienvenida"
                />
              </div>

              <!-- Editor -->
              <div>
                <label class="block text-xs font-bold text-slate-400 uppercase tracking-wider mb-2">Cuerpo del mensaje</label>
                <div 
                  ref="editorRef"
                  contenteditable="true"
                  @input="syncEditorToTemplate"
                  @dragover="onEditorDragOver"
                  @dragleave="onEditorDragLeave"
                  @drop="onEditorDrop"
                  @click="onEditorClick"
                  class="w-full min-h-[220px] bg-slate-50/70 border rounded-2xl p-6 text-slate-800 text-sm leading-7 outline-none transition-all cursor-text focus:bg-white"
                  :class="isDraggingOver 
                    ? 'border-slate-800 ring-2 ring-slate-800/10 bg-slate-100/50' 
                    : 'border-slate-200/80 focus:ring-2 focus:ring-slate-200'"
                ></div>
              </div>

            </div>

            <!-- Columna Derecha: Sidebar de Campos -->
            <div class="w-full lg:w-72 bg-white rounded-[2rem] p-6 shadow-[0_8px_30px_rgb(0,0,0,0.04)] border border-slate-100 flex-shrink-0">
              <div class="mb-4 pb-3 border-b border-slate-100/60">
                <h3 class="font-bold text-sm text-slate-800">Campos</h3>
                <p class="text-[11px] text-slate-400 mt-0.5">Arrastra o haz clic para agregar al texto.</p>
              </div>

              <div class="space-y-2.5">
                <div
                  v-for="fieldType in availableFieldTypes"
                  :key="fieldType.id"
                  draggable="true"
                  @dragstart="onDragStartBadge($event, fieldType)"
                  @click="onClickBadgeButton(fieldType)"
                  class="p-3.5 rounded-2xl border border-slate-200/80 bg-slate-50/60 hover:bg-white hover:border-slate-300 hover:shadow-sm transition-all cursor-grab active:cursor-grabbing select-none"
                >
                  <div class="flex items-start gap-2.5">
                    <div class="w-6 h-6 rounded-lg bg-white border border-slate-200/80 flex items-center justify-center flex-shrink-0 mt-0.5 text-slate-700">
                      <!-- Texto -->
                      <svg v-if="fieldType.id === 'texto'" class="w-3.5 h-3.5" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M4 6h16M4 12h16M4 18h7"/></svg>
                      <!-- Dinero -->
                      <svg v-else-if="fieldType.id === 'dinero'" class="w-3.5 h-3.5" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 8c-1.657 0-3 .895-3 2s1.343 2 3 2 3 .895 3 2-1.343 2-3 2m0-8V6m0 8v2m8-4a8 8 0 11-16 0 8 8 0 0116 0z"/></svg>
                      <!-- Fecha -->
                      <svg v-else-if="fieldType.id === 'fecha'" class="w-3.5 h-3.5" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M8 7V3m8 4V3m-9 8h10M5 21h14a2 2 0 002-2V7a2 2 0 00-2-2H5a2 2 0 00-2 2v12a2 2 0 002 2z"/></svg>
                      <!-- Hora -->
                      <svg v-else-if="fieldType.id === 'hora'" class="w-3.5 h-3.5" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 8v4l3 3m6-3a9 9 0 11-18 0 9 9 0 0118 0z"/></svg>
                      <!-- Número -->
                      <svg v-else-if="fieldType.id === 'numero'" class="w-3.5 h-3.5" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M7 20l4-16m2 16l4-16M6 9h14M4 15h14"/></svg>
                      <!-- Opciones -->
                      <svg v-else class="w-3.5 h-3.5" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M8 9l4 4 4-4m-8 6h8"/></svg>
                    </div>

                    <div class="flex-1">
                      <h4 class="text-xs font-bold text-slate-800">{{ fieldType.label }}</h4>
                      <p class="text-[11px] text-slate-500 mt-0.5 leading-snug">{{ fieldType.desc }}</p>
                    </div>
                  </div>
                </div>
              </div>
            </div>

          </div>

        </div>
      </main>

    </div>

    <!-- ================= MODAL DE EXPORTACIÓN ================= -->
    <div 
      v-if="isExportModalOpen"
      class="fixed inset-0 bg-slate-900/40 backdrop-blur-xs flex items-center justify-center z-50 p-4 transition-all"
      @click.self="isExportModalOpen = false"
    >
      <div class="bg-white rounded-2xl p-6 max-w-sm w-full shadow-2xl border border-slate-100">
        <h3 class="text-sm font-bold text-slate-800 mb-4">Exportar Plantillas</h3>

        <div class="space-y-3 mb-6">
          <label class="flex items-center gap-3 p-3 rounded-xl border border-slate-200 hover:bg-slate-50 cursor-pointer transition-colors" :class="{'border-slate-800 bg-slate-50/80': exportScope === 'all'}">
            <input 
              type="radio" 
              name="exportScope" 
              value="all" 
              v-model="exportScope" 
              class="w-4 h-4 text-slate-800 accent-slate-800 cursor-pointer"
            />
            <div>
              <p class="text-xs font-bold text-slate-800">Todas las plantillas</p>
              <p class="text-[11px] text-slate-400">Exporta las {{ templates.length }} plantillas guardadas</p>
            </div>
          </label>

          <label class="flex items-center gap-3 p-3 rounded-xl border border-slate-200 hover:bg-slate-50 cursor-pointer transition-colors" :class="{'border-slate-800 bg-slate-50/80': exportScope === 'current'}">
            <input 
              type="radio" 
              name="exportScope" 
              value="current" 
              v-model="exportScope" 
              class="w-4 h-4 text-slate-800 accent-slate-800 cursor-pointer"
            />
            <div class="overflow-hidden">
              <p class="text-xs font-bold text-slate-800">Solo la seleccionada</p>
              <p class="text-[11px] text-slate-400 truncate max-w-[200px]">{{ currentTemplate?.title || 'Sin título' }}</p>
            </div>
          </label>
        </div>

        <div class="flex items-center gap-2">
          <button 
            @click="isExportModalOpen = false"
            class="flex-1 py-2 rounded-xl border border-slate-200 text-slate-600 text-xs font-semibold hover:bg-slate-50 transition-colors cursor-pointer"
          >
            Cancelar
          </button>
          <button 
            @click="executeExport"
            class="flex-1 py-2 rounded-xl bg-[#1a1b26] text-white text-xs font-semibold hover:bg-slate-800 transition-transform active:scale-95 cursor-pointer shadow-sm"
          >
            Descargar JSON
          </button>
        </div>
      </div>
    </div>

    <!-- ================= MODAL SIMPLE PARA NOMBRAR CAMPO ================= -->
    <div 
      v-if="isFieldModalOpen"
      class="fixed inset-0 bg-slate-900/40 backdrop-blur-xs flex items-center justify-center z-50 p-4 transition-all"
      @click.self="isFieldModalOpen = false"
    >
      <div class="bg-white rounded-2xl p-6 max-w-sm w-full shadow-2xl border border-slate-100">
        
        <h3 class="text-sm font-bold text-slate-800 mb-4">
          {{ editingBadgeElement ? 'Editar campo' : 'Nombre del campo' }}
        </h3>

        <div class="space-y-4 mb-6">
          <div>
            <label class="block text-xs font-semibold text-slate-500 mb-1.5">Nombre</label>
            <input 
              ref="fieldNameInputRef"
              v-model="fieldNameInput"
              @keydown.enter.prevent="confirmFieldInsertion"
              @keydown.esc="isFieldModalOpen = false"
              type="text" 
              class="w-full bg-slate-50 border border-slate-200 rounded-xl px-3.5 py-2 text-sm font-medium text-slate-800 outline-none focus:bg-white focus:ring-2 focus:ring-slate-300 transition-all font-mono"
            />
          </div>

          <!-- Opciones Múltiples -->
          <div v-if="selectedFieldType?.id === 'opciones'">
            <label class="block text-xs font-semibold text-slate-500 mb-2">Opciones</label>
            
            <div class="space-y-2 max-h-56 overflow-y-auto pr-1">
              <div v-for="(opt, idx) in optionsList" :key="idx" class="flex items-center gap-2">
                <span class="text-slate-300 text-xs select-none">⠿</span>
                
                <input 
                  type="radio" 
                  name="defaultOption"
                  :checked="opt.isDefault"
                  @change="setDefaultOption(idx)"
                  class="w-4 h-4 text-slate-800 accent-slate-800 cursor-pointer"
                  title="Marcar como valor predeterminado"
                />

                <input 
                  v-model="opt.text"
                  @keydown.enter.prevent="confirmFieldInsertion"
                  type="text" 
                  class="flex-1 bg-white border border-slate-200 rounded-lg px-3 py-1.5 text-xs text-slate-800 outline-none focus:border-slate-800 transition-colors font-mono"
                />

                <button 
                  @click="removeOption(idx)"
                  class="w-6 h-6 rounded-full border border-slate-200 text-slate-400 hover:text-slate-800 hover:border-slate-400 flex items-center justify-center transition-colors cursor-pointer text-xs"
                  title="Eliminar opción"
                >
                  -
                </button>
              </div>
            </div>

            <button 
              @click="addOption"
              class="text-xs font-semibold text-slate-600 hover:text-slate-900 mt-2.5 flex items-center gap-1 cursor-pointer"
            >
              + Agregar opción
            </button>
          </div>
        </div>

        <div class="flex items-center gap-2">
          <button 
            @click="isFieldModalOpen = false"
            class="flex-1 py-2 rounded-xl border border-slate-200 text-slate-600 text-xs font-semibold hover:bg-slate-50 transition-colors cursor-pointer"
          >
            Cancelar
          </button>
          <button 
            @click="confirmFieldInsertion"
            class="flex-1 py-2 rounded-xl bg-[#1a1b26] text-white text-xs font-semibold hover:bg-slate-800 transition-transform active:scale-95 cursor-pointer shadow-sm"
          >
            Aceptar
          </button>
        </div>

      </div>
    </div>

    <!-- ================= NOTIFICACIÓN TOAST ================= -->
    <div 
      class="fixed bottom-8 right-8 z-50 transition-all duration-300 transform"
      :class="toast.show ? 'translate-y-0 opacity-100' : 'translate-y-8 opacity-0 pointer-events-none'"
    >
      <div class="w-[360px] max-w-[calc(100vw-2rem)] bg-white rounded-2xl shadow-[0_20px_50px_-15px_rgba(0,0,0,0.15)] border border-slate-200/80 overflow-hidden">
        <div class="p-5">
          <div class="flex items-center justify-between" :class="{'mb-3': toast.description || toast.type === 'confirm'}">
            <div class="flex items-center gap-3">
              <div 
                class="w-6 h-6 rounded-full flex items-center justify-center border-2 flex-shrink-0"
                :class="{
                  'border-emerald-500 text-emerald-500': toast.type === 'success',
                  'border-rose-500 text-rose-500': toast.type === 'error',
                  'border-amber-500 text-amber-500': toast.type === 'confirm'
                }"
              >
                <svg v-if="toast.type === 'success'" class="w-3.5 h-3.5 stroke-[3]" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" d="M5 13l4 4L19 7"></path></svg>
                <svg v-if="toast.type === 'error'" class="w-3.5 h-3.5 stroke-[3]" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" d="M6 18L18 6M6 6l12 12"></path></svg>
                <span v-if="toast.type === 'confirm'" class="text-xs font-bold leading-none">!</span>
              </div>
              
              <h4 class="font-bold text-slate-800 text-sm leading-snug">{{ toast.title }}</h4>
            </div>

            <button @click="closeToast" class="text-slate-400 hover:text-slate-600 cursor-pointer p-0.5" title="Cerrar">
              <svg class="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M6 18L18 6M6 6l12 12"></path></svg>
            </button>
          </div>

          <p v-if="toast.description" class="text-slate-500 text-xs leading-relaxed mb-4 pl-9">
            {{ toast.description }}
          </p>

          <div v-if="toast.type === 'confirm'" class="pl-9 flex gap-2">
            <button 
              @click="confirmToastAction" 
              class="px-4 py-1.5 bg-rose-500 hover:bg-rose-600 text-white rounded-lg text-xs font-semibold shadow-xs transition-colors cursor-pointer active:scale-95"
            >
              Eliminar
            </button>
            <button 
              @click="closeToast" 
              class="px-4 py-1.5 border border-slate-200 hover:bg-slate-50 text-slate-700 rounded-lg text-xs font-semibold shadow-xs transition-colors cursor-pointer active:scale-95"
            >
              Cancelar
            </button>
          </div>
        </div>

        <div 
          v-if="toast.autoClose" 
          class="h-1 bg-emerald-500 transition-all ease-linear"
          :style="{ width: `${progress}%` }"
        ></div>
      </div>
    </div>

  </div>
</template>