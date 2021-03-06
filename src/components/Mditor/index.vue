<template>
  <div class="mditor">

    <div class="mditor-controls">
      <div class="tabNav">
        <span
        v-for="tab in tabs"
        class="tabNav__tab"
        :class="{ active: mode === tab }"
        :key="tab"
        @click="mode = tab">
          {{ tab }}
        </span>
      </div>

      <div class="toolbar">
       <capture-dropdown />
      </div>
    </div>

    <textarea 
      v-if="mode === 'write'"
      class="mditor__textArea custom-scroll"
      ref='textarea'
      spellcheck="false"
      :value="value"
      :placeholder="placeholder"
      :style="overflow"
      @input="handleInput"
      @keydown.tab.prevent="indentText">
    </textarea>

    <div 
      v-else
      v-html="parsedText"      
      class="mditor__preview markdown-preview custom-scroll"       
      :style="overflow"
    />
    
  </div>
</template>

<script>
import marked from 'marked';
import hljs from 'highlight.js';

import CaptureDropdown from './CaptureDropdown';

marked.setOptions({
  highlight: code => hljs.highlightAuto(code).value,
  breaks: true,
});

export default {
  name: 'Mditor',
  props: {
    value: {
      type: String,
      default: '',
    },
    placeholder: {
      type: String,
      default: 'Write your contents',
    },
    minHeight: {
      type: Number,
      default: 100,
    },
    maxHeight: {
      type: Number,
      default: 450,
    },
  },
  data: () => ({
    tabs: ['write', 'preview'],
    mode: 'write',
  }),
  computed: {
    parsedText() {
      return marked(this.value) || 'No contents for preview';
    },
    overflow() {
      return {
        minHeight: `${this.minHeight}px`,
        maxHeight: `${this.maxHeight}px`,
      };
    },
  },
  watch: {
    mode(v) {
      if (v === 'write') {
        this.$nextTick(() => {
          this.autoHeight();
        });
      }
    },
  },
  methods: {
    handleInput(e) {
      this.autoHeight();
      this.$emit('input', e.target.value);
    },

    autoHeight() {
      const target = this.$refs.textarea;
      target.style.height = 'auto';
      target.style.height = `${target.scrollHeight + 10}px`;
    },

    indentText(e) {
      const {
        selectionStart: startPos,
        selectionEnd: endPos,
        value: inputVal,
      } = e.target;

      // 2 spaces
      const indent = '  ';

      let selectionPrev = inputVal.substring(0, startPos);
      let selection = inputVal.substring(startPos, endPos);
      const selectionNext = inputVal.substring(endPos);

      // if text selection
      if (startPos !== endPos) {
        // find line start position
        const lineStartPos = startPos - selectionPrev.split('\n').pop().length;

        let startIndentLength = indent.length;
        let endIndentLength = indent.length;

        // indent
        if (!e.shiftKey) {
          selectionPrev = selectionPrev.substring(0, lineStartPos) +
                          indent +
                          selectionPrev.substring(lineStartPos, startPos);

          // line break => line break + indent
          selection = selection.replace(/\n/g, `\n${indent}`);

        // unindent - shift + tab
        } else {
          // line start with indent
          if (inputVal.substr(lineStartPos, 1) === ' ') {
            startIndentLength = -startIndentLength;

            // Indent is in start of selection
            if (lineStartPos === startPos) {
              startIndentLength = 0;
              endIndentLength = 0;
              selection = selection.substring(indent.length);

            // Indent is before selection
            } else {
              endIndentLength = -endIndentLength;
              selectionPrev = selectionPrev.substring(0, lineStartPos) +
                              selectionPrev.substring(lineStartPos + indent.length);
            }
          } else {
            startIndentLength = 0;
            endIndentLength = 0;
          }

          selection = selection.replace(new RegExp(`\n${indent}`, 'g'), '\n');
        }

        // set indented value
        e.target.value = selectionPrev + selection + selectionNext;

        e.target.selectionStart = startPos + startIndentLength;
        e.target.selectionEnd = startPos + selection.length + endIndentLength;

      // just tab not selection
      } else {
        e.target.value = selectionPrev + indent + selectionNext;
        e.target.selectionStart = startPos + indent.length;
        e.target.selectionEnd = startPos + indent.length;
      }
    },
  },
  created() {
    this.$nextTick(() => {
      this.autoHeight();
    });
  },
  components: {
    CaptureDropdown,
  },
};
</script>
