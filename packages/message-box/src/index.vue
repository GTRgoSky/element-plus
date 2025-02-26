<template>
  <transition name="fade-in-linear" @after-leave="$emit('vanish')">
    <el-overlay
      v-show="visible"
      :z-index="state.zIndex"
      :overlay-class="['is-message-box', modalClass]"
      :mask="modal"
      @click.self="handleWrapperClick"
    >
      <div
        ref="root"
        v-trap-focus
        :aria-label="title || 'dialog'"
        aria-modal="true"
        :class="[
          'el-message-box',
          customClass,
          { 'el-message-box--center': center },
        ]"
      >
        <div
          v-if="title !== null && title !== undefined"
          class="el-message-box__header"
        >
          <div class="el-message-box__title">
            <div
              v-if="icon && center"
              :class="['el-message-box__status', icon]"
            ></div>
            <span>{{ title }}</span>
          </div>
          <button
            v-if="showClose"
            type="button"
            class="el-message-box__headerbtn"
            aria-label="Close"
            @click="
              handleAction(distinguishCancelAndClose ? 'close' : 'cancel')
            "
            @keydown.enter="
              handleAction(distinguishCancelAndClose ? 'close' : 'cancel')
            "
          >
            <i class="el-message-box__close el-icon-close"></i>
          </button>
        </div>
        <div class="el-message-box__content">
          <div class="el-message-box__container">
            <div
              v-if="icon && !center && hasMessage"
              :class="['el-message-box__status', icon]"
            ></div>
            <div v-if="hasMessage" class="el-message-box__message">
              <slot>
                <p v-if="!dangerouslyUseHTMLString">{{ message }}</p>
                <p v-else v-html="message"></p>
              </slot>
            </div>
          </div>
          <div v-show="showInput" class="el-message-box__input">
            <el-input
              ref="inputRef"
              v-model="state.inputValue"
              :type="inputType"
              :placeholder="inputPlaceholder"
              :class="{ invalid: state.validateError }"
              @keydown.prevent.enter="handleInputEnter"
            />
            <div
              class="el-message-box__errormsg"
              :style="{
                visibility: !!state.editorErrorMessage ? 'visible' : 'hidden',
              }"
            >
              {{ state.editorErrorMessage }}
            </div>
          </div>
        </div>
        <div class="el-message-box__btns">
          <el-button
            v-if="showCancelButton"
            :loading="state.cancelButtonLoading"
            :class="[cancelButtonClass]"
            :round="roundButton"
            size="small"
            @click="handleAction('cancel')"
            @keydown.enter="handleAction('cancel')"
          >
            {{ state.cancelButtonText || t('el.messagebox.cancel') }}
          </el-button>
          <el-button
            v-show="showConfirmButton"
            ref="confirmRef"
            :loading="state.confirmButtonLoading"
            :class="[confirmButtonClasses]"
            :round="roundButton"
            :disabled="state.confirmButtonDisabled"
            size="small"
            @click="handleAction('confirm')"
            @keydown.enter="handleAction('confirm')"
          >
            {{ state.confirmButtonText || t('el.messagebox.confirm') }}
          </el-button>
        </div>
      </div>
    </el-overlay>
  </transition>
</template>
<script lang="ts">
import {
  defineComponent,
  nextTick,
  onMounted,
  onBeforeUnmount,
  computed,
  watch,
  reactive,
  ref,
} from 'vue'
import ElButton from '@element-plus/button'
import ElInput from '@element-plus/input'
import { t } from '@element-plus/locale'
import { Overlay as ElOverlay } from '@element-plus/overlay'
import { useModal, useLockScreen, useRestoreActive, usePreventGlobal } from '@element-plus/hooks'
import { TrapFocus } from '@element-plus/directives'
import PopupManager from '@element-plus/utils/popup-manager'
import { on, off } from '@element-plus/utils/dom'
import { EVENT_CODE } from '@element-plus/utils/aria'

import type { ComponentPublicInstance, PropType } from 'vue'
import type { Action, MessageBoxState } from './message-box.type'

const TypeMap: Indexable<string> = {
  success: 'success',
  info: 'info',
  warning: 'warning',
  error: 'error',
}

export default defineComponent({
  name: 'ElMessageBox',
  components: {
    ElButton,
    ElInput,
    ElOverlay,
  },
  directives: {
    TrapFocus,
  },
  props: {
    beforeClose: {
      type: Function as PropType<(action: Action, state: MessageBoxState, doClose: () => void) => any>,
      default: undefined,
    },
    callback: Function,
    cancelButtonText: {
      type: String,
    },
    cancelButtonClass: String,
    center: Boolean,
    closeOnClickModal: {
      type: Boolean,
      default: true,
    },
    closeOnPressEscape: {
      type: Boolean,
      default: true,
    },
    closeOnHashChange: {
      type: Boolean,
      default: true,
    },
    confirmButtonText: {
      type: String,
    },
    confirmButtonClass: String,
    container: {
      type: String, // default append to body
      default: 'body',
    },
    customClass: String,
    dangerouslyUseHTMLString: Boolean,
    distinguishCancelAndClose: Boolean,
    iconClass: String,
    inputPattern: {
      type: Object as PropType<RegExp>,
      default: () => undefined,
      validator: (val: unknown) => (val instanceof RegExp || val === 'undefined'),
    },
    inputPlaceholder: {
      type: String,
    },
    inputType: {
      type: String,
      default: 'text',
    },
    inputValue: {
      type: String,
    },
    inputValidator: {
      type: Function as PropType<(...args: any[]) => boolean | string>,
      default: null,
    },
    inputErrorMessage: String,
    lockScroll: {
      type: Boolean,
      default: true,
    },
    message: String,
    modalFade: { // implement this feature
      type: Boolean,
      default: true,
    },
    modalClass: String,
    modal: {
      type: Boolean,
      default: true,
    },
    roundButton: Boolean,
    showCancelButton: Boolean,
    showConfirmButton: {
      type: Boolean,
      default: true,
    },
    showClose: {
      type: Boolean,
      default: true,
    },
    type: String,
    title: String,
    showInput: Boolean,
    zIndex: Number,
  },
  emits: ['vanish', 'action'],
  setup(props, { emit }) {
    // const popup = usePopup(props, doClose)
    const visible = ref(false)
    // s represents state
    const state = reactive({
      action: '' as Action,
      inputValue: props.inputValue,
      confirmButtonLoading: false,
      cancelButtonLoading: false,
      cancelButtonText: props.cancelButtonText,
      confirmButtonDisabled: false,
      confirmButtonText: props.confirmButtonText,
      editorErrorMessage: '',
      // refer to: https://github.com/ElemeFE/element/commit/2999279ae34ef10c373ca795c87b020ed6753eed
      // seemed ok for now without this state.
      // isOnComposition: false, // temporary remove
      validateError: false,
      zIndex: PopupManager.nextZIndex(),
    })
    const icon = computed(() => props.iconClass || (props.type && TypeMap[props.type] ? `el-icon-${TypeMap[props.type]}` : ''))
    const hasMessage = computed(() => !!props.message)
    const inputRef = ref<ComponentPublicInstance>(null)
    const confirmRef = ref<ComponentPublicInstance>(null)

    const confirmButtonClasses = computed(() => `el-button--primary ${props.confirmButtonClass}`)

    watch(() => state.inputValue, async val => {
      await nextTick()
      if (props.type === 'prompt' && val !== null) {
        validate()
      }
    }, { immediate: true })

    watch(() => visible.value, val => {
      if (val) {
        if (props.type === 'alert' || props.type === 'confirm') {
          nextTick().then(() => { confirmRef.value?.$el?.focus?.() })
        }
        state.zIndex = PopupManager.nextZIndex()
      }
      if (props.type !== 'prompt') return
      if (val) {
        nextTick().then(() => {
          if (inputRef.value && inputRef.value.$el) {
            getInputElement().focus()
          }
        })
      } else {
        state.editorErrorMessage = ''
        state.validateError = false
      }
    })

    onMounted(async () => {
      await nextTick()
      if (props.closeOnHashChange) {
        on(window, 'hashchange', doClose)
      }
    })

    onBeforeUnmount(() => {
      if (props.closeOnHashChange) {
        off(window, 'hashchange', doClose)
      }
    })

    function doClose() {
      if (!visible.value) return
      visible.value = false
      nextTick(() => {
        if (state.action) emit('action', state.action)
      })
    }

    const handleWrapperClick = () => {
      if (props.closeOnClickModal) {
        handleAction(props.distinguishCancelAndClose ? 'close' : 'cancel')
      }
    }

    const handleInputEnter = () => {
      if (props.inputType !== 'textarea') {
        return handleAction('confirm')
      }
    }

    const handleAction = (action: Action) => {
      if (props.type === 'prompt' && action === 'confirm' && !validate()) {
        return
      }

      state.action = action

      if (props.beforeClose) {
        props.beforeClose?.(action, state, doClose)
      } else {
        doClose()
      }
    }

    const validate = () => {
      if (props.type === 'prompt') {
        const inputPattern = props.inputPattern
        if (inputPattern && !inputPattern.test(state.inputValue || '')) {
          state.editorErrorMessage = props.inputErrorMessage || t('el.messagebox.error')
          state.validateError = true
          return false
        }
        const inputValidator = props.inputValidator
        if (typeof inputValidator === 'function') {
          const validateResult = inputValidator(state.inputValue)
          if (validateResult === false) {
            state.editorErrorMessage = props.inputErrorMessage || t('el.messagebox.error')
            state.validateError = true
            return false
          }
          if (typeof validateResult === 'string') {
            state.editorErrorMessage = validateResult
            state.validateError = true
            return false
          }
        }
      }
      state.editorErrorMessage = ''
      state.validateError = false
      return true
    }

    const getInputElement = () => {
      const inputRefs = inputRef.value.$refs
      return (inputRefs.input || inputRefs.textarea) as HTMLElement
    }

    const handleClose = () => {
      handleAction('close')
    }

    // when close on press escape is disabled, pressing esc should not callout
    // any other message box and close any other dialog-ish elements
    // e.g. Dialog has a close on press esc feature, and when it closes, it calls
    // props.beforeClose method to make a intermediate state by callout a message box
    // for some verification or alerting. then if we allow global event liek this
    // to dispatch, it could callout another message box.
    if (props.closeOnPressEscape) {
      useModal({
        handleClose,
      }, visible)
    } else {
      usePreventGlobal(visible, 'keydown', (e: KeyboardEvent) => e.code === EVENT_CODE.esc)
    }

    // locks the screen to prevent scroll
    if (props.lockScroll) {
      useLockScreen(visible)
    }

    // restore to prev active element.
    useRestoreActive(visible)

    return {
      state,
      visible,
      hasMessage,
      icon,
      confirmButtonClasses,
      inputRef,
      confirmRef,
      doClose, // for outside usage
      handleClose, // for out side usage
      handleWrapperClick,
      handleInputEnter,
      handleAction,
      t,
    }
  },
})
</script>
