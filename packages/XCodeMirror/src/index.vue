<template>
  <div class="x-codemirror-wrap">
    <div
      ref="merge"
      v-if="isMerge"
    ></div>
    <textarea
      v-else
      id="refTextarea"
      ref="textarea"
      :placeholder="placeholder"
    ></textarea>
  </div>
</template>

<script>
import _CodeMirror from 'codemirror';

const CodeMirror = window.CodeMirror || _CodeMirror;

let codeMirrorEditorIns = null;
const initCodeMirrorEditor = (type, options) => {
  if (type === 'code') {
    codeMirrorEditorIns = CodeMirror.fromTextArea(document.getElementById("refTextarea"), options);
  } else if (type === 'merge') {
    codeMirrorEditorIns = CodeMirror.fromTextArea(document.getElementById("refTextarea"), options).edit;
  }
};

export default {
  name: 'XCodeMirror',
  props: {
    emit: String,
    placeholder: String,
    code: String,
    options: Object,
    isMerge: Boolean,
    events: Array,
  },
  data() {
    return {
      curCode: '',
    };
  },
  watch: {
    code(newVal) {
      this.setCodeChange(newVal);
    },
    isMerge(newVal) {
      if (newVal) {
        this.$nextTick(() => {
          this.setMerge(newVal);
        })
      } else {
        this.init();
      }
    },
    options: {
      deep: true,
      handler(options) {
        for (const key in options) {
          this.changeOption(key, options[key]);
        }
      },
    }
  },
  mounted() {
    this.init();
    this.bindEvents();
    this.emitTo('ready', codeMirrorEditorIns);
  },
  methods: {
    init() {
      if (this.isMerge) {
        this.initMerge();
      } else {
        this.initCode();
        this.refresh();
      }
      // 支持双向绑定
      codeMirrorEditorIns.on('change', (coder) => {
        this.curCode = coder.getValue();
        if (this.$emit) {
          this.emitTo('change', this.curCode);
        }
      });
    },
    initCode() {
      const options = Object.assign({}, this.options);
      initCodeMirrorEditor('code', options)
      codeMirrorEditorIns.setValue(this.code || this.curCode);
    },
    initMerge() {
      const options = Object.assign({}, this.options);
      initCodeMirrorEditor(options)
    },
    changeOption(opt, value) {
      // 修改编辑器的语法配置
      codeMirrorEditorIns.setOption(opt, value);
    },
    bindEvents() {
      const filterEvent = [];
      const allowEvents = [
        'scroll',
        'changes',
        'beforeChange',
        'cursorActivity',
        'keyHandled',
        'inputRead',
        'electricInput',
        'beforeSelectionChange',
        'viewportChange',
        'swapDoc',
        'gutterClick',
        'gutterContextMenu',
        'focus',
        'blur',
        'refresh',
        'optionChange',
        'scrollCursorIntoView',
        'update'
      ];
      this.events.filter((event) => {
        if (allowEvents.includes(event) && !filterEvent.includes(event)) {
          filterEvent.push(event);
        }
      });
      filterEvent.forEach(event => {
        // 循环事件，并兼容 run-time 事件命名
        codeMirrorEditorIns.on(event, (...args) => {
          this.emitTo(event, { ...args });
        })
      })
    },
    setCodeChange(newVal) {
      const curValue = codeMirrorEditorIns.getValue()
      if (newVal !== curValue) {
        codeMirrorEditorIns.setValue(newVal);
        const scrollInfo = codeMirrorEditorIns.getScrollInfo();
        codeMirrorEditorIns.scrollTo(scrollInfo.left, scrollInfo.top);
      }
    },
    setMerge() {
      const history = codeMirrorEditorIns.doc.history;
      const cleanGeneration = codeMirrorEditorIns.doc.cleanGeneration;
      this.options.value = codeMirrorEditorIns.getValue();
      this.destroy();
      this.init();
      codeMirrorEditorIns.doc.history = history;
      codeMirrorEditorIns.doc.cleanGeneration = cleanGeneration;
    },
    refresh() {
      this.$nextTick(() => {
        codeMirrorEditorIns.refresh();
      })
    },
    emitTo(type, data) {
      this.$emit(this.emit, {
        type,
        data,
      });
    },
    beforeDestroy() {
      this.destroy();
    },
    destroy() {
      const cmDom = codeMirrorEditorIns?.doc?.cm?.getWrapperElement();
      cmDom && cmDom.remove && cmDom.remove();
    },
  },
}
</script>

<style scoped>
.x-codemirror-wrap{
  width: 100%;
}
</style>
