<template>
  <div class="resp-textbox"
  v-on="eventStoppers()"
  @click="textBoxClick"
  @mousedown="controlEditMode"
   :style="outerTextBox"
  :tabindex="properties.TabIndex"
  @keydown.enter.self="setContentEditable($event, true)"
  @contextmenu="isEditMode ? openTextContextMenu($event): parentConextMenu($event)"
  @mouseover="updateMouseCursor"
  >
    <textarea
      data-gramm="false"
      ref="textareaRef"
      :style="cssStyleProperty"
      @mouseover="updateMouseCursor"
      @mousemove="mouseMoveEvent"
      :tabindex="properties.TabIndex"
      :maxlength="properties.MaxLength !==0 ? properties.MaxLength : ''"
      :disabled="getDisableValue"
      :title="properties.ControlTipText"
      @dblclick="dblclick($event)"
      @focus="closeTextMenu"
      :readonly="properties.Locked"
      @keydown.escape.exact="releaseEditMode"
      v-cursorDirective="{
        start: textareaRef ? start : data.properties.CursorStartPosition,
        end: textareaRef ? end : data.properties.CursorEndPosition,
        pwdCharType: properties.PasswordChar,
      }"
      @keydown.ctrl="handleCtrl"
      @keydown.shift="handleShiftEnter"
      @keydown.tab.exact="tabKeyBehavior"
      @keydown.enter.exact="enterKeyBehavior"
      @input="
        properties.PasswordChar === ''
          ? textAndValueUpdate($event)
          : handlePasswordChar($event)
      "
      @mousedown.stop="handleMouseDown"
      @keydown.exact="properties.PasswordChar !== '' ? handleDelete($event) : null"
      @blur="handleBlur($event, textareaRef, hideSelectionDiv)"
      @click="handleClick"
      class="text-box-design"
      :value="
        properties.Value
          | passwordFilter(properties.PasswordChar, properties.Value)
      "
      @dragstart="dragBehavior"
    />
    <label ref="autoSizeTextarea"></label>
    <label ref="selectionMarginRef"></label>
  </div>
</template>

<script lang="ts">
import {
  Component,
  Vue,
  Prop,
  Ref,
  Watch,
  Emit,
  Mixins
} from 'vue-property-decorator'
import FdControlVue from '@/api/abstract/FormDesigner/FdControlVue'
import { DirectiveBinding } from 'vue/types/options'
import { EventBus } from '@/FormDesigner/event-bus'

@Component({
  name: 'FDTextBox',
  filters: {
    /**
     * @description filter to show passwordChar instead of original text
     * @function passwordFilter
     * @param value text to be filtered
     * @param password passwordChar value to filter the text
     * @param text  TextBox properties Text Value
     */
    passwordFilter (value: string, password: string, text: string) {
      if (password !== '' && text !== '') {
        let filteredValue: string = ''
        for (let index = 0; index < text.length; index++) {
          filteredValue = filteredValue + password[0]
        }
        return filteredValue
      } else {
        return value
      }
    }
  },
  directives: {
    cursorDirective: {
      /**
       * @description  called after the containing component’s VNode has updated
       * updates selection start and end cursor positon when data model value changes
       * to maintain cursor position
       * @function update Hook Functions i.e, (bind, inserted, update, componentUpdated, unbind)
       * @param event The element the directive is bound to. This can be used to directly manipulate the DOM.
       * @param vnode The virtual node produced by Vue’s compiler
       */
      update (event: HTMLElement, vnode: DirectiveBinding) {
        if (vnode.value.pwdCharType !== '') {
          (event as HTMLFormElement).selectionStart = vnode.value.start;
          (event as HTMLFormElement).selectionEnd = vnode.value.end
        } else {
          return undefined
        }
      }
    }
  }
})
export default class FDTextBox extends Mixins(FdControlVue) {
  @Ref('hideSelectionDiv') readonly hideSelectionDiv!: HTMLDivElement;
  @Ref('autoSizeTextarea') readonly autoSizeTextarea!: HTMLLabelElement;
  @Ref('textareaRef') textareaRef!: HTMLTextAreaElement;
  @Ref('selectionMarginRef') readonly selectionMarginRef!: HTMLLabelElement;

  $el!: HTMLDivElement
  originalText: string = ''
  trimmedText: string = ''
  fitToSizeWhenMultiLine: boolean = false
  start: number = 0
  end: number = 0
  autoSizeAfterEnter: boolean = false
  get outerTextBox () {
    return {
      cursor: this.controlCursor
    }
  }
  mouseMoveEvent (event: MouseEvent) {
    if (this.properties.SelectionMargin && event.offsetX < 11) {
      this.controlCursor = 'context-menu'
    } else {
      this.controlCursor = 'auto'
    }
  }

  handleMouseDown (event: MouseEvent) {
    if (event.detail !== 2) {
      if (this.properties.SelectionMargin) {
        if (event.offsetX < 11) {
          this.textareaRef.setSelectionRange(this.getSelectionStart, this.getSelectionStart)
        }
      }
    }
  }

  dblclick (e: Event) {
    if (this.isEditMode) {
      let newSelectionStart = 0
      const eTarget = e.target as HTMLTextAreaElement
      for (let i = eTarget.selectionStart; i > 0; i--) {
        if (eTarget.value[i - 1] === ' ' || eTarget.value[i - 1] === undefined) {
          newSelectionStart = i
          break
        }
      }
      this.textareaRef.setSelectionRange(newSelectionStart, eTarget.selectionEnd)
    }
  }
  get getDisableValue () {
    if (this.isRunMode) {
      return this.properties.Enabled === false || this.properties.Locked
    } else if (this.isEditMode) {
      if (this.properties.Locked) {
        return true
      } else {
        return false
      }
    } else {
      return true
    }
  }
  /**
   * @description style object is passed to :style attribute in Textarea tag
   * dynamically changing the styles of the component based on properties
   * @function cssStyleProperty
   *
   */
  get cssStyleProperty () {
    const controlProp = this.properties
    const font: font = controlProp.Font
      ? controlProp.Font
      : {
        FontName: 'Arial',
        FontSize: 10,
        FontItalic: true,
        FontBold: true,
        FontUnderline: true,
        FontStrikethrough: true,
        FontStyle: 'Arial'
      }
    let display = ''
    if (this.isRunMode) {
      display = controlProp.Visible ? controlProp.Width === 0 || controlProp.Height === 0 ? 'none' : 'block' : 'none'
    } else {
      display = controlProp.Width === 0 || controlProp.Height === 0 ? 'none' : 'block'
    }
    return {
      cursor: this.isEditMode ? this.controlCursor : '',
      left: `${controlProp.Left}px`,
      width: `${controlProp.Width}px`,
      height: `${controlProp.Height}px`,
      top: `${controlProp.Top}px`,
      borderColor: controlProp.BorderStyle === 1 ? controlProp.BorderColor : '',
      textAlign:
        controlProp.TextAlign === 0
          ? 'left'
          : controlProp.TextAlign === 1
            ? 'center'
            : 'right',
      backgroundColor: controlProp.BackStyle ? controlProp.BackColor : 'transparent',
      borderLeft: controlProp.BorderStyle === 1 ? '1px solid ' + controlProp.BorderColor : controlProp.SpecialEffect === 2 ? '2px solid gray' : controlProp.SpecialEffect === 3 ? '1.5px solid gray' : controlProp.SpecialEffect === 4 ? '0.5px solid gray' : '',
      borderRight: controlProp.BorderStyle === 1 ? '1px solid ' + controlProp.BorderColor : controlProp.SpecialEffect === 1 ? '2px solid gray' : controlProp.SpecialEffect === 4 ? '1.5px solid gray' : controlProp.SpecialEffect === 3 ? '0.5px solid gray' : '',
      borderTop: controlProp.BorderStyle === 1 ? '1px solid ' + controlProp.BorderColor : controlProp.SpecialEffect === 2 ? '2px solid gray' : controlProp.SpecialEffect === 3 ? '1.5px solid gray' : controlProp.SpecialEffect === 4 ? '0.5px solid gray' : '',
      borderBottom: controlProp.BorderStyle === 1 ? '1px solid ' + controlProp.BorderColor : controlProp.SpecialEffect === 1 ? '2px solid gray' : controlProp.SpecialEffect === 4 ? '1.5px solid gray' : controlProp.SpecialEffect === 3 ? '0.5px solid gray' : '',
      whiteSpace: this.setWhiteSpace(),
      wordBreak:
        controlProp.WordWrap && controlProp.MultiLine ? 'break-word' : 'normal',
      color:
        controlProp.Enabled === true ? controlProp.ForeColor : this.getEnabled,
      fontFamily: (font.FontStyle! !== '') ? this.setFontStyle : font.FontName!,
      fontSize: `${font.FontSize}px`,
      fontStyle: font.FontItalic || this.isItalic ? 'italic' : '',
      textDecoration:
        font.FontStrikethrough === true && font.FontUnderline === true
          ? 'underline line-through'
          : font.FontUnderline
            ? 'underline'
            : font.FontStrikethrough
              ? 'line-through'
              : '',
      textDecorationSkipInk: 'none',
      fontWeight: font.FontBold ? 'bold' : (font.FontStyle !== '') ? this.tempWeight : '',
      fontStretch: (font.FontStyle !== '') ? this.tempStretch : '',
      display: display,
      overflowX: this.properties.AutoSize ? 'hidden' : this.getScrollBarX,
      overflowY: this.properties.AutoSize ? 'hidden' : this.getScrollBarY,
      pointerEvents: this.isEditMode ? 'auto' : 'none',
      paddingLeft: controlProp.SelectionMargin ? '10px' : '2px'
    }
  }

  setWhiteSpace () {
    const controlProp = this.properties
    if (this.isEditMode) {
      if (controlProp.MultiLine) {
        if (controlProp.WordWrap) {
          return 'break-spaces'
        } else {
          return 'nowrap'
        }
      } else {
        return 'nowrap'
      }
    } else {
      if (controlProp.MultiLine) {
        if (controlProp.WordWrap) {
          return 'break-spaces'
        } else {
          return 'pre'
        }
      } else {
        return 'pre'
      }
    }
  }
  /**
   * @description updates the dataModel textBox object properties when user insert/delete text
   * inside textBox when passwordChar is set, updates text and values properties of textBox with entered character
   * @function handlePasswordChar
   * @param event its of type TextEvent
   * @event input
   *
   */
  handlePasswordChar (event: Event) {
    this.start = this.textareaRef.selectionStart
    this.end = this.textareaRef.selectionEnd
    const eventTargetArea = event.target as HTMLTextAreaElement
    const controlPropData = this.properties
    if (event.target instanceof HTMLTextAreaElement) {
      if (!controlPropData.MultiLine) {
        event.target.value = event.target.value.replace(/(\r\n|\n|\r)/gm, '')
        for (let i = 0; i < event.target.value.length; i++) {
          event.target.value = event.target.value.replace(event.target.value[i], this.properties.PasswordChar!)
        }
      }
    }
    let newData
    let text = this.properties.Text!
    let selectionDiff =
      (this.data.properties!.CursorStartPosition as number) !==
      (this.data.properties!.CursorEndPosition as number)
    if (event.target instanceof HTMLTextAreaElement) {
      if (selectionDiff && (event instanceof InputEvent)) {
        // selection
        newData =
        text.substring(0, this.data.properties!.CursorStartPosition as number) + event.data
        text.substring(this.data.properties!.CursorEndPosition as number)
        this.updateDataModel({ propertyName: 'Text', value: newData })
        this.updateDataModel({ propertyName: 'Value', value: newData })
      } else if (text.length < eventTargetArea.value.length && (event instanceof InputEvent)) {
        // insertion
        newData = [
          text.slice(0, this.start - 1),
          event.data,
          text.slice(this.start - 1)
        ].join('')
        this.updateDataModel({ propertyName: 'Text', value: newData })
        this.updateDataModel({ propertyName: 'Value', value: newData })
      } else if (text.length > eventTargetArea.value.length) {
        // deletion
        newData = [
          text.slice(0, eventTargetArea.selectionStart),
          text.slice(eventTargetArea.selectionStart + 1)
        ].join('')
        this.updateDataModel({ propertyName: 'Text', value: newData })
        this.updateDataModel({ propertyName: 'Value', value: newData })
      }
      this.updateDataModel({
        propertyName: 'CursorStartPosition',
        value: eventTargetArea.selectionStart
      })
      this.updateDataModel({
        propertyName: 'CursorEndPosition',
        value: eventTargetArea.selectionEnd
      })
      const controlPropData = this.properties
      if (controlPropData.AutoTab && controlPropData.MaxLength! > 0) {
        if (event.target instanceof HTMLTextAreaElement) {
          if (eventTargetArea.value.length === controlPropData.MaxLength) {
            EventBus.$emit('focusNextControlOnAutoTab')
          }
        }
      }
    } else {
      throw new Error('Expected HTMLTextAreaElement but found different element')
    }
  }
  /**
   * @description style object is passed to :style attribute in div tag
   * dynamically changing the styles of the component based on cssStyleProperty of
   * textarea
   * @function divcssStyleProperty
   *
   */
  get divcssStyleProperty () {
    const styleObject = this.cssStyleProperty
    return {
      ...styleObject,
      display: 'none',
      paddingTop: '2px',
      paddingLeft: '2px',
      cursor: this.controlCursor
    }
  }
  /**
   * @description When user enters ctrl + enter cursor moves to next line
   * @function handleCtrlAndShiftEnter
   * @param el its of type HTMLTextAreaElement
   * @param text new line character
   * @event keydown.enter.ctrl
   */
  handleCtrlAndShiftEnter (el : HTMLTextAreaElement, text: string) {
    el.focus()
    if (el.selectionEnd !== el.selectionStart) {
      const text = this.properties.Value!.toString()
      this.updateDataModel({
        propertyName: 'Value',
        value: text.slice(0, el.selectionStart) + text.slice(el.selectionEnd)
      })
    }
    if (typeof el.selectionStart === 'number' &&
            typeof el.selectionEnd === 'number') {
      const val = el.value
      const selStart = el.selectionStart
      const te = this.properties.Value!.toString()
      this.updateDataModel({
        propertyName: 'Value',
        value: te.slice(0, selStart) + text + te.slice(selStart)
      })
      el.value = val.slice(0, selStart) + text + val.slice(el.selectionEnd)
      const startEndPos = selStart + text.length
      el.selectionStart = el.selectionEnd = startEndPos
      this.textareaRef.blur()
      this.textareaRef.focus()
      this.start = this.end = selStart + 1
    }
    this.autoSizeAfterEnter = true
    this.updateAutoSize()
  }
  /**
   * @description EnterKeyBehavior - if true when enter is pressed while editing the cursor moves to next line
   *  if false the cursor remains in same place
   * @function enterKeyBehavior
   * @param event its of type KeyboardEvent
   * @event keydown.enter
   */
  enterKeyBehavior (event: KeyboardEvent) {
    if (this.properties.MultiLine) {
      if (event.ctrlKey) {
        event.preventDefault()
        this.handleCtrlAndShiftEnter(this.textareaRef, '\n')
        return true
      } else if (event.shiftKey) {
        return true
      } else if (this.properties.EnterKeyBehavior && this.properties.MultiLine) {
        this.handleCtrlAndShiftEnter(this.textareaRef, '\n')
        event.preventDefault()
        return false
      }
    }
    event.preventDefault()
    return false
  }
  handleShiftEnter (event: KeyboardEvent) {
    if (event.keyCode === 13) {
      event.preventDefault()
      if (this.properties.MultiLine) {
        this.handleCtrlAndShiftEnter(this.textareaRef, '\n')
      }
    }
  }
  /**
   * @description  specifies how the control responds to the TAB key
   * when  TabKeyBehavior true in textBox tab spaces are added on press of tab Key
   * when TabKeyBehavior false in textBox pressing tab moves focus to next controls
   * @function tabKeyBehavior
   * @param event its of type MouseEvent
   * @event keydown.tab
   */
  tabKeyBehavior (event: KeyboardEvent) {
    this.start = this.textareaRef.selectionStart + 1
    this.end = this.textareaRef.selectionEnd + 1
    if (this.properties.TabKeyBehavior) {
      const TABKEY = 9
      if (event.target instanceof HTMLTextAreaElement) {
        event.stopPropagation()
        const eventTaget = event.target
        if (this.properties.PasswordChar !== '') {
          const selectionStart = eventTaget.selectionStart
          const selectionEnd = eventTaget.selectionEnd
          const value = this.properties.Value!.toString()
          if (event.keyCode === TABKEY) {
            this.updateDataModel({ propertyName: 'Value', value: value.substring(0, selectionStart) + '\t' + value.substring(selectionEnd) })
            event.preventDefault()
          }
          (event.target).selectionStart = selectionStart + 1;
          (event.target).selectionEnd = (event.target).selectionStart
        } else {
          const selectionStart = eventTaget.selectionStart
          const selectionEnd = eventTaget.selectionEnd
          const value = eventTaget.value
          if (event.keyCode === TABKEY) {
            (event.target).value =
            value.substring(0, selectionStart) +
            '\t' +
            value.substring(selectionEnd)
            event.preventDefault()
          }
          this.textAndValueUpdate(event)
          this.textareaRef.selectionStart = selectionStart + 1
          this.textareaRef.selectionEnd = selectionEnd + 1
        }
      } else {
        throw new Error('Expected HTMLTextAreaElement but found different element')
      }
    } else {
      if (this.isEditMode) {
        event.stopPropagation()
        EventBus.$emit('focusNextControlOnAutoTab')
      }
    }
  }
  /**
   * @description updates the dataModel textBox object properties when user insert/delete text
   * inside textBox, updates text and values properties of textBox
   * @function textAndValueUpdate
   * @param event its of type InputEvent
   * @event input
   *
   */
  textAndValueUpdate (event: Event) {
    const controlPropData = this.properties
    if (controlPropData.AutoTab && controlPropData.MaxLength! > 0) {
      if (event.target instanceof HTMLTextAreaElement) {
        if (event.target.value.length === controlPropData.MaxLength) {
          EventBus.$emit('focusNextControlOnAutoTab')
        }
      }
    }
    if (this.properties.AutoSize) {
      if ((event as InputEvent).data === null && this.properties.MultiLine) {
        this.autoSizeAfterEnter = true
      }
      this.updateAutoSize()
    }
    if (event.target instanceof HTMLTextAreaElement) {
      if (!controlPropData.MultiLine) {
        event.target.value = event.target.value.replace(/(\r\n|\n|\r)/gm, '')
      }
      this.updateDataModel({
        propertyName: 'Value',
        value: (event.target).value
      })
      this.updateDataModel({
        propertyName: 'Text',
        value: (event.target).value
      })
      if (this.properties.ControlSource !== '') {
        this.updateDataModelExtraData({
          propertyName: 'ControlSourceValue',
          value: (event.target).value
        })
      } else {
        return undefined
      }
    } else {
      throw new Error('Expected HTMLTextAreaElement but found different element')
    }
  }

  @Watch('properties.Font.FontSize', { deep: true })
  autoSizeValidateOnFontChange () {
    if (this.properties.AutoSize) {
      this.updateAutoSize()
    }
  }

  @Watch('properties.WordWrap', { deep: true })
  autoSizeValidateOnWordWrapChange () {
    if (this.properties.AutoSize) {
      this.updateAutoSize()
    }
  }

  /**
   * @description dragBehavior - if true when dragging
   *  if false the cursor remains in same place
   * @function dragBehavior
   * @param event its of type KeyboardEvent
   * @event dragStart
   */
  dragBehavior (e: Event) {
    if (this.properties.DragBehavior) {
      return true
    }
    e.preventDefault()
  }

  @Watch('isEditMode')
  editModeValidation () {
    if (this.properties.AutoSize && !this.isEditMode) {
      this.updateAutoSize()
    }
    if (this.textareaRef) {
      this.originalText = this.textareaRef.value
      this.trimmedText = this.originalText.replace(/(\r\n|\n|\r)/gm, ',')
    }
  }

  @Watch('properties.MultiLine')
  multiLineValidate () {
    if (this.textareaRef) {
      this.originalText = this.properties.Value!.toString()
      this.trimmedText = this.originalText.replace(/(\r\n|\n|\r)/gm, '¶')
      if (this.properties.MultiLine === false && this.properties.PasswordChar !== '') {
        this.updateDataModel({ propertyName: 'Text', value: '' })
      }
      if (this.properties.MultiLine) {
        this.trimmedText = this.originalText.replace(/¶/g, '\n')
        this.updateDataModel({ propertyName: 'Value', value: this.trimmedText })
      } else {
        this.updateDataModel({ propertyName: 'Value', value: this.trimmedText })
      }
    }
    if (this.properties.AutoSize) {
      this.fitToSizeWhenMultiLine = true
      this.updateAutoSize()
    }
  }

  @Watch('properties.AutoSize')
  setAutoHeightWidth () {
    if (this.properties.AutoSize) {
      this.updateAutoSize()
      if (this.properties.MultiLine) {
        this.fitToSizeWhenMultiLine = true
      }
    }
  }
  /**
   * @override
   */
  updateAutoSize () {
    if (this.properties.AutoSize === true) {
      this.$nextTick(() => {
        if (this.properties.Value === '') {
          this.updateDataModel({
            propertyName: 'Height',
            value: parseInt(this.textareaRef.style.fontSize) + 15
          })
          this.updateDataModel({
            propertyName: 'Width',
            value: 15
          })
        } else {
          if (!this.properties.MultiLine) {
            const textareaRef: HTMLTextAreaElement = this.textareaRef
            // replication of stype attribute to Label tag for autoSize property to work
            let tempLabel: HTMLLabelElement = this.autoSizeTextarea
            tempLabel.style.display = 'inline'
            if (this.properties.MultiLine && !this.fitToSizeWhenMultiLine) {
              tempLabel.style.display = 'inline-block'
              if (this.properties.SelectionMargin) {
                tempLabel.style.width = textareaRef.clientWidth - 12 + 'px'
              } else {
                tempLabel.style.width = textareaRef.clientWidth - 4 + 'px'
              }
            }
            tempLabel.innerText = this.textareaRef.value
            tempLabel.style.fontFamily = textareaRef.style.fontFamily
            tempLabel.style.fontStretch = textareaRef.style.fontStretch
            tempLabel.style.fontStyle = textareaRef.style.fontStyle
            tempLabel.style.fontSize = parseInt(textareaRef.style.fontSize) + 'px'
            tempLabel.style.whiteSpace = 'pre'
            tempLabel.style.wordBreak = textareaRef.style.wordBreak
            tempLabel.style.fontWeight = textareaRef.style.fontWeight
            const initWid = this.textareaRef.offsetWidth
            this.updateDataModel({
              propertyName: 'Width',
              value: tempLabel.offsetWidth + 14
            })

            this.updateDataModel({
              propertyName: 'Height',
              value: parseInt(this.textareaRef.style.fontSize) + 16
            })
            tempLabel.style.display = 'none'
            tempLabel.innerText = ''
          } else {
            const textareaRef: HTMLTextAreaElement = this.textareaRef
            // replication of stype attribute to Label tag for autoSize property to work
            let tempLabel: HTMLLabelElement = this.autoSizeTextarea
            tempLabel.style.display = 'inline-block'
            if (this.properties.SelectionMargin) {
              tempLabel.style.width = textareaRef.clientWidth - 12 + 'px'
            } else {
              tempLabel.style.width = textareaRef.clientWidth - 4 + 'px'
            }
            tempLabel.innerText = textareaRef.value
            tempLabel.style.fontFamily = textareaRef.style.fontFamily
            tempLabel.style.fontStretch = textareaRef.style.fontStretch
            tempLabel.style.fontStyle = textareaRef.style.fontStyle
            tempLabel.style.fontSize = parseInt(textareaRef.style.fontSize) + 'px'
            if (!this.properties.WordWrap) {
              tempLabel.style.whiteSpace = 'pre'
              tempLabel.style.wordBreak = textareaRef.style.wordBreak
            } else {
              tempLabel.style.whiteSpace = 'break-spaces'
              tempLabel.style.wordBreak = 'break-word'
            }
            let initHeight = tempLabel.offsetHeight
            let initWidth = tempLabel.offsetWidth
            tempLabel.style.whiteSpace = ''
            tempLabel.style.wordBreak = ''
            for (let i = initWidth; i >= 0; i--) {
              tempLabel.style.width = i + 'px'
              if (tempLabel.offsetHeight > initHeight || tempLabel.scrollWidth > tempLabel.offsetWidth) {
                this.updateDataModel({
                  propertyName: 'Width',
                  value: i + (this.properties.SelectionMargin ? tempLabel.offsetWidth === 0 ? (parseInt(this.textareaRef.style.fontSize) - 15) : 15 : 7)
                })
                if (this.fitToSizeWhenMultiLine) {
                  this.fitToSizeWhenMultiLine = false
                  this.updateDataModel({
                    propertyName: 'Width',
                    value: i + (this.properties.SelectionMargin ? tempLabel.offsetWidth === 0 ? (parseInt(this.textareaRef.style.fontSize) - 15) : 15 : 7)
                  })
                }
                break
              }
            }
            tempLabel.style.whiteSpace = 'break-spaces'
            tempLabel.style.wordBreak = 'break-word'
            if (this.autoSizeAfterEnter) {
              this.updateDataModel({
                propertyName: 'Height',
                value: initHeight + 25
              })
              this.autoSizeAfterEnter = false
            } else {
              this.updateDataModel({
                propertyName: 'Height',
                value: initHeight + 15
              })
            }
            tempLabel.style.display = 'none'
            tempLabel.innerText = ''
          }
        }
        this.textareaRef.scrollTop = 0
      })
    } else {
      return undefined
    }
  }
  async handleCtrl (event: KeyboardEvent) {
    const key = event.keyCode
    const controlProp = this.properties
    let copiedText = this.properties.Value!.toString()
    let selectionStart = this.textareaRef.selectionStart
    let selectionEnd = this.textareaRef.selectionEnd
    if (event.keyCode === 86 && controlProp.PasswordChar !== '') {
      let selectStart = this.textareaRef.selectionStart
      let selectEnd = this.textareaRef.selectionEnd
      const text = await navigator.clipboard.readText()
      let newText = ''
      if (controlProp.MaxLength! > 0) {
        let remLength = controlProp.MaxLength! - controlProp.Value!.toString().length
        if (text.length > remLength) {
          newText = copiedText.slice(0, selectStart) + text.slice(0, remLength) + copiedText.slice(selectEnd)
        } else {
          newText = copiedText.slice(0, selectStart) + text + copiedText.slice(selectEnd)
        }
      } else {
        newText = copiedText.slice(0, selectStart) + text + copiedText.slice(selectEnd)
      }
      this.updateDataModel({
        propertyName: 'Text',
        value: newText
      })
    }
    if (key === 88) {
      if (controlProp.PasswordChar !== '') {
        const text = controlProp.Value!.toString().slice(selectionStart, selectionEnd)
        navigator.clipboard.writeText(text)
        selectionStart += 1
        let textAfterCut = copiedText.slice(0, selectionStart - 1) + copiedText.slice(selectionEnd)
        this.start = selectionStart - 1
        this.end = selectionStart - 1
        this.updateDataModel({
          propertyName: 'Text',
          value: textAfterCut
        })
        this.textareaRef.selectionStart = selectionStart
        this.textareaRef.selectionEnd = selectionStart
      }
    }
    if (key === 67) {
      if (controlProp.PasswordChar !== '') {
        let selectionStart = this.textareaRef.selectionStart
        let selectionEnd = this.textareaRef.selectionEnd
        const text = controlProp.Value!.toString().slice(selectionStart, selectionEnd)
        navigator.clipboard.writeText(text)
      }
    }
    if (key === 13) {
      this.enterKeyBehavior(event)
    }
  }

  created () {
    const propData: controlData = this.data
    if (propData.properties.ControlSource !== '') {
      const controlSourceValue = propData.extraDatas!.ControlSourceValue
      this.updateDataModel({
        propertyName: 'Value',
        value: controlSourceValue
      })
      this.updateDataModel({ propertyName: 'Text', value: controlSourceValue })
    }
  }

  /**
  * @description watches ControlSource property
  * @function updateControlSourceValue
  * @param oldVal previous string value
  * @param newVal  new/changed string value
  */
  @Watch('properties.ControlSource', { deep: true })
  updateControlSourceValue (newVal: string, oldVal: string) {
    const propData: controlData = this.data
    if (propData.properties.ControlSource !== '') {
      const controlSourceValue = propData.extraDatas!.ControlSourceValue
      this.updateDataModel({
        propertyName: 'Value',
        value: controlSourceValue
      })
      this.updateDataModel({ propertyName: 'Text', value: controlSourceValue })
    } else {
      this.updateDataModel({
        propertyName: 'Value',
        value: ''
      })
      this.updateDataModel({ propertyName: 'Text', value: '' })
    }
  }
  /**
   * @description keep tracks of key press and selectionStart and selectionEnd
   * updates extra property CursorStartPosition and CursorEndPosition which is required
   * when user insert, update or delete text in textBox
   * @function handleDelete
   * @param event it is of type KeyboardEvent
   * @event keydown
   */
  handleDelete (event: KeyboardEvent) {
    if (event.keyCode !== 8) {
      this.handlePasswordChar(event)
    }
    if (event.target instanceof HTMLTextAreaElement) {
      const text = this.properties.Value!.toString()
      if (event.keyCode === 8 || event.keyCode === 46) {
        if ((event.target).selectionStart! === (event.target).selectionEnd!) {
          this.updateDataModel({
            propertyName: 'Text',
            value: text.slice(0, (event.target).selectionStart! - 1) + text.slice((event.target).selectionEnd! - 1)
          })
        } else {
          this.updateDataModel({
            propertyName: 'Text',
            value: text.slice(0, (event.target).selectionStart!) + text.slice((event.target).selectionEnd!)
          })
          this.start = (event.target).selectionStart!
          this.end = (event.target).selectionStart!
          event.preventDefault()
        }
      } else {
        return undefined
      }
    } else {
      throw new Error('Expected HTMLTextAreaElement but found different element')
    }
  }
  /**
   * @description  show selection when TextBox loses focus
   * when HideSelection is false selection is show if user selects any text
   * @function handleBlur
   * @event blur
   *
   */
  handleBlur (
    event: TextEvent,
    textareaRef: HTMLTextAreaElement,
    hideSelectionDiv: HTMLDivElement
  ) {
    this.getSelectionStart = this.textareaRef.selectionStart
    this.getSelectionEnd = this.textareaRef.selectionEnd
    const selection = window.getSelection()!
    if (!this.isEditMode) {
      if (selection.rangeCount >= 1) {
        for (var i = 0; i < selection.rangeCount; i++) {
          selection.removeRange(selection.getRangeAt(i))
        }
      }
    }
  }
  /**
   *@description hides the div when focus comes to textarea when hideSelection
   * properties is false
   * @function handleClick
   * @param event its of FocusEvent
   * @event click
   */
  handleClick (event: MouseEvent) {
    const selectionStart = this.textareaRef.selectionStart
    const selectionEnd = this.textareaRef.selectionEnd
    if (event.offsetX < 11) {
      this.selectionMargin(selectionStart)
    }
  }
  selectionMargin (start: number) {
    if (this.properties.SelectionMargin === true) {
      this.$nextTick(() => {
        const textareaRef: HTMLTextAreaElement = this.textareaRef
        // replication of stype attribute to Label tag for autoSize property to work
        let tempLabel: HTMLLabelElement = this.selectionMarginRef
        if (this.properties.MultiLine) {
          tempLabel.style.display = 'inline-block'
          tempLabel.style.width = textareaRef.clientWidth - 12 + 'px'
        }
        tempLabel.style.whiteSpace = 'nowrap'
        tempLabel.style.fontFamily = textareaRef.style.fontFamily
        tempLabel.style.fontStretch = textareaRef.style.fontStretch
        tempLabel.style.fontStyle = textareaRef.style.fontStyle
        tempLabel.style.whiteSpace = textareaRef.style.whiteSpace
        tempLabel.style.wordBreak = textareaRef.style.wordBreak
        tempLabel.style.fontSize = parseInt(textareaRef.style.fontSize) + 'px'
        let initHeight = 0
        let finalIndex = 0
        let isSpacepresent = false
        for (let i = start; i < this.textareaRef.value.length; i++) {
          tempLabel.innerText = tempLabel.innerText + textareaRef.value[i]
          if (i === start) {
            initHeight = tempLabel.clientHeight
          } else if (textareaRef.value[i] === '\n') {
            finalIndex = i - 1
            tempLabel.innerText = ''
            tempLabel.style.display = 'none'
            break
          } else if (tempLabel.clientHeight > initHeight) {
            if (tempLabel.innerText[i + 1] === ' ') {
              finalIndex = i
              tempLabel.innerText = ''
              tempLabel.style.display = 'none'
              break
            } else {
              for (let j = i; j >= 0; j--) {
                if (tempLabel.innerText[j] === ' ' || tempLabel.innerText[j] === '\n') {
                  isSpacepresent = true
                  break
                }
              }
              if (isSpacepresent) {
                for (let j = i; j >= 0; j--) {
                  if (tempLabel.innerText[j] === ' ') {
                    finalIndex = j - 1
                    break
                  }
                }
              } else {
                finalIndex = i - 1
              }
            }
            tempLabel.innerText = ''
            tempLabel.style.display = 'none'
            break
          } else if (i === this.textareaRef.value.length - 1) {
            finalIndex = i
            tempLabel.innerText = ''
            tempLabel.style.display = 'none'
          }
        }
        if (!isSpacepresent) {
          if (finalIndex < start) {
            this.textareaRef.setSelectionRange(start, start + 1)
            tempLabel.innerText = ''
            tempLabel.style.display = 'none'
          } else {
            this.textareaRef.setSelectionRange(start, finalIndex + 1)
          }
        } else {
          this.textareaRef.setSelectionRange(start, start + finalIndex + 1)
        }
      })
    }
  }
  /**
   * @description hides div instead of textarea when hideSelection is false
   * when hideSelection properties is true textarea is shown
   * when hideSelection properties is false div is shown
   * @function divHide
   * @param event its of type MouseEvent
   * @event click
   */
  divHide (event: MouseEvent, textareaRef: HTMLTextAreaElement) {
    if (this.isActivated) {
      if (event.target instanceof HTMLSpanElement || event.target instanceof HTMLDivElement) {
        (event.target).style.display = 'none'
        textareaRef.style.display = 'block'
        if (
          (event.target).tagName === 'SPAN' &&
      (event.target).parentNode!.nodeName === 'DIV'
        ) {
          ((event.target)
            .parentNode as HTMLElement).style.display = 'none'
        }
        textareaRef.focus()
        textareaRef.selectionStart = textareaRef.selectionEnd
      }
    }
  }

  mounted () {
    this.$el.focus({
      preventScroll: true
    })
  }
  releaseEditMode (event: KeyboardEvent) {
    this.$el.focus({
      preventScroll: true
    })
    this.setContentEditable(event, false)
  }
  eventStoppers () {
    const eventStop = (event: Event) => event.stopPropagation()
    return this.isEditMode === false ? null : {
      keydown: eventStop
    }
  }
  @Watch('properties.TextAlign')
  textAlignValidate () {
    if (this.properties.AutoSize) {
      this.updateAutoSize()
    }
  }
  @Watch('properties.ScrollBars')
  scrollBarValidate () {
    if (this.properties.AutoSize) {
      this.updateAutoSize()
    }
  }
  @Watch('properties.Enabled')
  enabledValidate () {
    if (this.properties.AutoSize) {
      this.updateAutoSize()
    }
  }
  @Watch('properties.SelectionMargin')
  selectionMarginValidate () {
    if (this.properties.AutoSize) {
      this.updateAutoSize()
    }
  }
  textBoxClick (event: MouseEvent) {
    if (this.toolBoxSelectControl === 'Select') {
      event.stopPropagation()
    }
    Vue.nextTick(() => {
      if (this.isEditMode) {
        this.textareaRef.blur()
        this.textareaRef.focus()
      }
    })
  }
  closeTextMenu () {
    EventBus.$emit('closeMenu')
  }
}
</script>

<style scoped>
.text-box-design {
  width: 0px;
  height: 0px;
  left: 0px;
  top: 0px;
  resize: none;
  overflow: hidden;
  box-sizing: border-box;
  caret-color: black;
}
.text-box-design:focus {
  outline: none;
}
.text-box-design::selection {
  background: lightblue;
}
</style>
