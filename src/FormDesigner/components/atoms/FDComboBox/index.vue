<template>
  <div
    class="custom-select"
    :tabindex="tabindex"
    :style="customSelectObj"
    :title="properties.ControlTipText"
    @click="selectedItem"
    @keydown.enter="setContentEditable($event, true)"
    @keydown.esc="releaseEditMode"
    v-on="eventStoppers()"
  >
    <div
      class="combobox"
      :style="boxStyleObj"
      @mousedown="controlEditMode"
      tabindex="1"
      @click="toFocus"
    >
      <div :class="properties.SelectionMargin ? 'selectionDiv' : ''">
        <span
          v-if="properties.SelectionMargin"
          class="selectionSpan"
          :style="selectionSpanObj"
          @click="setSelection"
        ></span>
        <textarea
          data-gramm="false"
          ref="textareaRef"
          :style="cssStyleProperty"
          wrap="off"
          :tabindex="properties.TabIndex"
          :readonly="getDisableValue || properties.Style === 1"
          :maxlength="properties.MaxLength !==0 ? properties.MaxLength : ''"
          @blur="handleBlur($event, textareaRef, hideSelectionDiv)"
          @click="handleClick($event, textareaRef, hideSelectionDiv)"
          @input="handleTextInput($event)"
          class="text-box-design"
          :value="selectionData[0]"
          @dragstart="dragBehavior"
          @focus="makeTextAreaFocus"
        />
        <div
          ref="hideSelectionDiv"
          @click="divHide($event, textareaRef)"
          :style="divcssStyleProperty"
          :title="properties.ControlTipText"
          class="text-box-design"
        >
          {{ selectionData[0] }}
        </div>
        <label
          ref="autoSizeTextarea"
          class="labelStyle"
          :class="labelStyleObj"
        ></label>
      </div>
      <div
        class="selected"
        :class="[properties.DropButtonStyle === 1 ? 'forArrow' : properties.DropButtonStyle === 2 ? 'forEllipsis' : properties.DropButtonStyle === 3 ? 'forReduce' : 'forPlain']"
        @click="enabledCheck($event)"
        :style="selectedStyleObj"
      ></div>
      <div class="items" :class="{ selectHide: !open }" :style="itemsStyleObj">
        <div class="listStyle" :title="properties.ControlTipText" :style="listStyleObj" @mousedown="scrollingOnListItems">
          <table :style="tableStyleObj" class="table-style" @click="tableClick" ref="comboRef" tabindex="1">
            <thead class="tHeadStyle" v-if="properties.ColumnHeads === true">
              <!-- <tr> -->
            <th class="thClass" :style="colHeadsStyle">
                <template
                  :style="tdStyleObj"
                  v-if="properties.ListStyle === 1"
                  class="tdClass"
                ></template>
                <template
                  v-for="(a, columnIndex) in extraDatas.ColumnHeadsValues"
                >
                  <th
                    v-if="columnIndex < properties.ColumnCount"
                    :key="columnIndex"
                    :style="updateColHeads(columnIndex)"
                  >
                    {{ a }}
                  </th>
                </template>
                <hr class="hrStyle">
            </th>
            <!-- </tr> -->
            </thead>
            <thead v-else></thead>
            <tbody
            class="tBodyStyle"
              :style="tableBodyObj"
              @click="open = false"
            >
              <tr
                :tabindex="index"
                class="tr"
                v-for="(item, index) of tempArray"
                :key="index"
                @mouseenter="handleDrag"
                @keydown="handleExtendArrowKeySelect"
                @blur="clearMatchEntry"
                @mousedown="
                  isRunMode || isEditMode ? handleMultiSelect($event) : ''
                "
              >
                <td
                  :style="tdStyleObj"
                  class="tdClassIn"
                  v-if="
                    properties.ListStyle === 1 && properties.ColumnCount > 0
                  "
                >
                <input name="radio" type="radio" class="inputClass" />
                </td>
                <td
                  class="column-item"
                  v-for="(i, index) in item"
                  :key="index"
                  :style="updateColumnWidths(index)"
                >
                  <template v-if="index < properties.ColumnCount">{{
                    i
                  }}</template>
                </td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>
    </div>
  </div>
</template>

<script lang="ts">
import {
  Component,
  Vue,
  Prop,
  Mixins,
  Watch,
  Ref
} from 'vue-property-decorator'
import FdControlVue from '@/api/abstract/FormDesigner/FdControlVue'
import { Mutation, Action, Getter } from 'vuex-class'
@Component({
  name: 'FDComboBox'
})
export default class FDComboBox extends Mixins(FdControlVue) {
  $el!: HTMLDivElement;
  @Ref('textareaRef') textareaRef: HTMLTextAreaElement;
  @Ref('autoSizeTextarea') readonly autoSizeTextarea!: HTMLLabelElement;
  @Ref('hideSelectionDiv') readonly hideSelectionDiv!: HTMLDivElement;
  @Ref('comboRef') comboRef!: HTMLTableElement
  private options = ['hello'];
  private tabindex = 0;
  eTargetValue:string = ''
  tempArray: Array<Array<string>> = [];
  open: boolean = false;
  multiselect = [];
  selectionStart: number = 0;
  selectionEnd: number = 0;
  tempInputValue: string = '';
  tempWidth: string = '0px';
  isScrolling: boolean = false;
  get getDisableValue () {
    if (this.isRunMode || this.isEditMode) {
      return this.properties.Enabled === false || this.properties.Locked
    } else {
      return true
    }
  }

  updateColumnValue (index: number) {
    return this.updateColumnWidths(index)
  }

  toFocus () {
    if (this.isEditMode) {
      this.isVisible = true
    } else {
      this.isVisible = false
    }
  }
  tableClick (e: Event) {
    this.tempListBoxComboBoxEvent = e
  }

  @Watch('properties.Font.FontSize', { deep: true })
  autoSizeValidateOnFontChange () {
    if (this.properties.AutoSize) {
      this.updateAutoSize()
    }
  }

  handleTextInput (e: Event) {
    if (this.properties.AutoSize) {
      this.updateAutoSize()
    }
    if (e.target instanceof HTMLTextAreaElement) {
      this.open = true
      const tempEvent = e.target
      this.eTargetValue = e.target.value
      this.comboRef.children[1].children[0].dispatchEvent(new MouseEvent('mousedown'))
      this.comboRef.children[1].children[0].dispatchEvent(new KeyboardEvent('keydown', { key: e.target.value }))
      this.selectionData[0] = this.eTargetValue
      if (this.properties.MatchEntry !== 0) {
        this.textareaRef.focus()
      }
      if ((this.properties.MatchEntry === 0)) {
        for (let i = 0; i < this.tempArray.length; i++) {
          if (this.tempArray[i][0][0] === this.textareaRef.value[0]) {
            this.textareaRef.value = this.tempArray[i][0]
            this.updateDataModel({ propertyName: 'Text', value: this.tempArray[i][0] })
            break
          }
        }
        this.textareaRef.setSelectionRange(0, this.textareaRef.value.length, 'forward')
      }
    } else {
      throw new Error('target is not instance of div element')
    }
  }

  makeTextAreaFocus () {
    if (!this.getDisableValue) {
      if (this.properties.ShowDropButtonWhen !== 0) {
        this.open = true
      }
    }
  }

  setSelection () {
    const setSelectionRef = this.textareaRef
    setSelectionRef.focus()
    setSelectionRef.select()
  }
  clearMatchEntry () {
    this.updateDataModelExtraData({ propertyName: 'MatchData', value: '' })
  }

  protected get selectionSpanObj (): Partial<CSSStyleDeclaration> {
    const controlProp = this.properties
    return {
      backgroundColor: controlProp.BackStyle ? controlProp.BackColor : 'transparent'
    }
  }
  protected get tableBodyObj (): Partial<CSSStyleDeclaration> {
    const controlProp = this.properties
    return {
      width: `${controlProp.Width}px !important`
    }
  }

  protected get colHeadsStyle (): Partial<CSSStyleDeclaration> {
    const controlProp = this.properties
    return {
      backgroundColor: controlProp.BackColor,
      width: '100%'
    }
  }

  scrollingOnListItems (e : Event) {
    this.isScrolling = true
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
    if (this.isScrolling) {
      this.open = true
      this.textareaRef.focus()
    } else {
      this.open = false
    }
    this.isScrolling = false
    if (this.properties.ShowDropButtonWhen === 1) {
      this.isVisible = false
    }
    if (this.properties.EnterFieldBehavior === 1 && event.target instanceof HTMLTextAreaElement) {
      const eventTarget = event.target
      let tempField = this.tempInputValue.slice(
        eventTarget.selectionStart - eventTarget.selectionEnd
      )
      this.selectionStart = eventTarget.selectionStart
      this.selectionEnd = eventTarget.selectionEnd
    }
    if (!this.properties.HideSelection && textareaRef && event.target instanceof HTMLTextAreaElement) {
      const eventTarget = event.target

      hideSelectionDiv.style.display = 'block'
      hideSelectionDiv.style.height = this.properties.Height! + 'px'
      hideSelectionDiv.style.width = this.properties.Width! + 'px'
      textareaRef.style.display = 'none'
      let textarea = eventTarget.value
      let firstPart =
        textarea.slice(0, eventTarget.selectionEnd) +
        '</span>' +
        textarea.slice(eventTarget.selectionEnd + Math.abs(0))
      let text =
        firstPart.slice(0, eventTarget.selectionStart) +
        "<span style='background-color:lightblue'>" +
        firstPart.slice(eventTarget.selectionStart + Math.abs(0))
      hideSelectionDiv.innerHTML = text
    }
    if (this.properties.MatchRequired && textareaRef) {
      const arrayCheck = this.extraDatas.RowSourceData!.findIndex(
        (element) => element[0] === this.tempInputValue
      )
      if (arrayCheck === -1) {
        this.textareaRef.focus()
      } else {
        return undefined
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
  handleClick (
    event: TextEvent,
    textareaRef: HTMLTextAreaElement,
    hideSelectionDiv: HTMLDivElement
  ) {
    if (!this.properties.HideSelection) {
      hideSelectionDiv.style.display = 'none'
    } else {
      return undefined
    }
    if (this.properties.EnterFieldBehavior === 0) {
      this.textareaRef.focus()
      this.textareaRef.select()
    } else if (this.properties.EnterFieldBehavior === 1) {
    } else {
      return undefined
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
    } else {
      throw new Error('event.target is not an instance of Span or Div Element')
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

  /**
   * @override
   */
  @Watch('properties.AutoSize', { deep: true })
  updateAutoSize () {
    if (this.properties.AutoSize === true) {
      this.updateDataModel({ propertyName: 'SelectionMargin', value: false })
      this.$nextTick(() => {
        const textareaRef: HTMLTextAreaElement = this.textareaRef
        // replication of stype attribute to Label tag for autoSize property to work
        let tempLabel: HTMLLabelElement = this.autoSizeTextarea
        tempLabel.style.display = 'inline'
        tempLabel.style.fontStyle = textareaRef.style.fontStyle
        tempLabel.style.fontSize = parseInt(textareaRef.style.fontSize) + 'px'
        tempLabel.style.whiteSpace = textareaRef.style.whiteSpace
        tempLabel.style.wordBreak = textareaRef.style.wordBreak
        tempLabel.style.fontWeight = textareaRef.style.fontWeight
        tempLabel.style.width = ((this.textareaRef.value.length + 1) * parseInt(textareaRef.style.fontSize)) + 'px'
        tempLabel.style.height = textareaRef.style.height
        tempLabel.innerText = textareaRef.value
        this.updateDataModel({
          propertyName: 'Width',
          value: tempLabel.offsetWidth > 20 ? tempLabel.offsetWidth + 21 : tempLabel.offsetWidth + 25
        })
        this.updateDataModel({
          propertyName: 'Height',
          value: tempLabel.offsetHeight + 5
        })
        tempLabel.innerText = ''
        tempLabel.style.display = 'none'
        this.selectionData[0] = this.eTargetValue
      })
    } else {
      return undefined
    }
  }

  protected get listStyleObj (): Partial<CSSStyleDeclaration> {
    const controlProp = this.properties
    return {
      height: !controlProp.ColumnHeads ? (controlProp.ListRows! > 0 && controlProp.ListRows! < this.extraDatas.RowSourceData!.length ? (controlProp.ListRows! * (controlProp.Font!.FontSize! + 3) + 'px') : '') : (controlProp.ListRows! > 0 && controlProp.ListRows! < this.extraDatas.RowSourceData!.length ? ((controlProp.ListRows! + 1) * (controlProp.Font!.FontSize! + 3) + 'px') : ''),
      backgroundColor: controlProp.BackColor
    }
  }

  protected get tableStyleObj (): Partial<CSSStyleDeclaration> {
    const controlProp = this.properties
    const font: font = controlProp.Font
      ? controlProp.Font
      : {
        FontName: 'Arial',
        FontSize: 20,
        FontItalic: true,
        FontBold: true,
        FontUnderline: true,
        FontStrikethrough: true
      }
    return {
      borderCollapse: 'collapse',
      tableLayout: 'fixed',
      backgroundColor: controlProp.BackStyle ? controlProp.BackColor : 'transparent',
      color: controlProp.ForeColor,
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
      textUnderlinePosition: 'under',
      fontWeight: font.FontBold ? 'bold' : (font.FontStyle !== '') ? this.tempWeight : '',
      fontStretch: (font.FontStyle !== '') ? this.tempStretch : '',
      width:
      controlProp.ColumnWidths === ''
        ? `${controlProp.Width}px`
        : `${controlProp.Width}px` +
          parseInt(controlProp.ColumnWidths!) +
          'px',
      outline: 'none'
    }
  }

  protected get cssStyleProperty (): Partial<CSSStyleDeclaration> {
    const controlProp = this.properties
    const font: font = controlProp.Font
      ? controlProp.Font
      : {
        FontName: 'Arial',
        FontSize: 20,
        FontItalic: true,
        FontBold: true,
        FontUnderline: true,
        FontStrikethrough: true
      }
    return {
      left: `${controlProp.Left}px`,
      width: controlProp.ShowDropButtonWhen === 0 ? `${controlProp.Width! - 4}px` : controlProp.ShowDropButtonWhen === 1 && this.isVisible === false ? `${controlProp.Width! - 4}px` : controlProp.SelectionMargin ? `${controlProp.Width! - 30}px` : `${controlProp.Width! - 25}px`,
      height: `${controlProp.Height! - 5}px`,
      top: `${controlProp.Top}px`,
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
      textUnderlinePosition: 'under',
      fontWeight: font.FontBold ? 'bold' : (font.FontStyle !== '') ? this.tempWeight : '',
      fontStretch: (font.FontStyle !== '') ? this.tempStretch : '',
      backgroundColor: controlProp.BackStyle ? controlProp.BackColor : 'transparent',
      color: controlProp.ForeColor,
      textAlign:
        controlProp.TextAlign === 0
          ? 'left'
          : controlProp.TextAlign === 1
            ? 'center'
            : 'right',
      cursor:
        controlProp.MousePointer !== 0 || controlProp.MouseIcon !== ''
          ? this.getMouseCursorData
          : 'default'
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
      whiteSpace: 'break-spaces',
      width: `${this.properties.Width! - 10}px`,
      height: `${this.properties.Height!}px`
    }
  }

  protected get customSelectObj (): Partial<CSSStyleDeclaration> {
    const controlProp = this.properties
    let display = ''
    if (this.isRunMode) {
      display = controlProp.Visible ? 'inline-block' : 'none'
    } else {
      display = 'inline-block'
    }
    return {
      display: display
    }
  }
  protected get tdStyleObj (): Partial<CSSStyleDeclaration> {
    const controlProp = this.properties
    return {
      textAlign:
        controlProp.TextAlign === 0
          ? 'left'
          : controlProp.TextAlign === 2
            ? 'right'
            : 'center'
    }
  }
  setWidth () {
    this.tempWidth = `${this.properties.Width! + 20}px`
    return this.tempWidth
  }
  @Watch('properties.Value', { deep: true })
  ValueData (newVal: string, oldVal: string) {
    if (
      this.properties.BoundColumn! > 0 &&
      this.properties.BoundColumn! < this.extraDatas.RowSourceData!.length
    ) {
      let tempData = [...this.extraDatas.RowSourceData!]
      if (tempData![0][this.properties.BoundColumn! - 1] === newVal) {
        this.updateDataModel({ propertyName: 'Value', value: newVal })
      } else {
        this.updateDataModel({ propertyName: 'Value', value: '' })
      }
    }
    if (newVal !== '') {
      this.selectionData[0] = newVal
      this.updateDataModel({ propertyName: 'Text', value: newVal })
    }
  }

  @Watch('properties.SelectionMargin', { deep: true })
  checkSelectionMargin (newVal: boolean, oldVal: boolean) {
    this.selectionData[0] = this.eTargetValue
  }
  releaseEditMode (event: KeyboardEvent) {
    this.$el.focus()
    this.setContentEditable(event, false)
  }

  mounted () {
    this.$el.focus()
    const initialRowSourceData = this.extraDatas.RowSourceData!
    if (initialRowSourceData) {
      this.tempArray = initialRowSourceData
    }
    this.updateDataModel({ propertyName: 'ControlSource', value: '' })
    if (initialRowSourceData && initialRowSourceData.length === 0) {
      this.updateDataModel({ propertyName: 'TopIndex', value: -1 })
    } else {
      this.updateDataModel({ propertyName: 'TopIndex', value: 0 })
    }
  }

  @Watch('properties.ControlSource')
  controlSourceUpdate () {
    if (this.properties.ControlSource !== '') {
      this.selectionData[0] = this.extraDatas.ControlSourceValue!
    }
  }
  expandWidth () {
    if (this.properties.ShowDropButtonWhen === 0) {
      return 'hidden'
    } else if (this.properties.ShowDropButtonWhen === 1) {
      if (this.isVisible) {
        return 'visible'
      } else {
        return 'hidden'
      }
    }
  }

  protected get labelStyleObj (): Partial<CSSStyleDeclaration> {
    const controlProp = this.properties
    const font: font = controlProp.Font
      ? controlProp.Font
      : {
        FontName: 'Arial',
        FontSize: 20,
        FontItalic: true,
        FontBold: true,
        FontUnderline: true,
        FontStrikethrough: true
      }
    return {
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
      fontWeight: font.FontBold ? 'bold' : (font.FontStyle !== '') ? this.tempWeight : '',
      fontStretch: (font.FontStyle !== '') ? this.tempStretch : ''

    }
  }

  protected get boxStyleObj (): Partial<CSSStyleDeclaration> {
    const controlProp = this.properties
    return {
      border:
        controlProp.BorderStyle === 0
          ? 'none'
          : '1px solid ' + controlProp.BorderColor,
      cursor:
        controlProp.MousePointer !== 0 || controlProp.MouseIcon !== ''
          ? this.getMouseCursorData
          : 'default',
      borderLeft: controlProp.SpecialEffect === 2 ? '2px solid gray' : controlProp.SpecialEffect === 3 ? '1.5px solid gray' : controlProp.SpecialEffect === 4 ? '0.5px solid gray' : '',
      borderRight: controlProp.SpecialEffect === 1 ? '2px solid gray' : controlProp.SpecialEffect === 4 ? '1.5px solid gray' : controlProp.SpecialEffect === 3 ? '0.5px solid gray' : '',
      borderTop: controlProp.SpecialEffect === 2 ? '2px solid gray' : controlProp.SpecialEffect === 3 ? '1.5px solid gray' : controlProp.SpecialEffect === 4 ? '0.5px solid gray' : '',
      borderBottom: controlProp.SpecialEffect === 1 ? '2px solid gray' : controlProp.SpecialEffect === 4 ? '1.5px solid gray' : controlProp.SpecialEffect === 3 ? '0.5px solid gray' : '',
      display: 'grid',
      gridTemplateColumns: `${controlProp.Width! - 20}px` + ' 21px',
      gridTemplateRows: `${controlProp.Height!}px`,
      outline: 'none'
    }
  }

  protected get itemsStyleObj (): Partial<CSSStyleDeclaration> {
    const controlProp = this.properties
    return {
      width: (parseInt(controlProp.ListWidth!) > 0) ? `${controlProp.ListWidth}px` : `${controlProp.Width!}px`
    }
  }
  protected get selectedStyleObj (): Partial<CSSStyleDeclaration> {
    const controlProp = this.properties
    return {
      visibility: controlProp.ShowDropButtonWhen === 2 ? 'visible' : this.expandWidth(),
      backgroundPosition:
        controlProp.DropButtonStyle === 1 ? 'center' : 'bottom',
      backgroundSize: controlProp.DropButtonStyle === 1 ? '12px 12px' : controlProp.DropButtonStyle === 2 ? '8px 8px' : controlProp.DropButtonStyle === 3 ? '9px 14px' : '',
      cursor:
        controlProp.MousePointer !== 0 || controlProp.MouseIcon !== ''
          ? this.getMouseCursorData
          : 'default'
    }
  }
  enabledCheck (e: MouseEvent) {
    if (this.isRunMode || this.isEditMode) {
      if (this.properties.Enabled) {
        if (!this.properties.Locked) {
          if (this.open) {
            this.textareaRef.focus()
            this.open = false
          } else {
            this.open = true
          }
        }
      } else {
        this.open = false
      }
    } else {
      this.open = false
    }
  }
  eventStoppers () {
    const eventStop = (event: Event) => event.stopPropagation()
    return this.isEditMode === false ? null : {
      keydown: eventStop
    }
  }
}
</script>

<style scoped>
.custom-select {
  position: relative;
  text-align: left;
  outline: none;
  box-sizing: border-box;
  /* line-height: 10px; specify the drop down height */
}
.selectionSpan {
  width: 5px;
}
.selectionDiv {
  display: grid;
  grid-template-columns: 5px auto;
}
.selected {
  background-color: lightgray;
  border: 1px solid #858586;
  border-left: 0px;
  color: black;
  background-size: 40%;
  background-position: center;
  background-repeat: no-repeat;
  cursor: pointer;
  width: 20px;
  height: calc(100% - 2px);
}

.items {
  color: black;
  background-color: white;
  overflow: hidden;
  left: 0;
  right: 0;
  width: calc(100% + 20px);
}
.item {
  color: black;
  cursor: pointer;
  border-left: 1px solid black;
  border-right: 1px solid black;
}
.item:hover {
  background-color: #0380fc;
  color: white;
  border: 1px dotted black;
}
.selectHide {
  display: none;
}
.combobox {
  display: grid;
  grid-template-columns: auto 20px;
}
.columnHeads {
  height: 19.2px;
  border: 1px solid black;
  border-top: 0px;
  display: grid;
  grid-template-columns: 100%;
}
.listStyle {
  width: 100%;
  height: 100%;
  background-color: lightgray;
  border: 1px solid gray;
  overflow: auto;
}
.list-outer {
  border: 0.1px solid lightgray;
  width: 164px;
  height: 130px;
  overflow-y: auto;
  box-shadow: -1px -1px grey;
}
.tr {
  outline: none;
}
.tr:hover {
  background-color: rgb(59, 122, 231);
}
.ul {
  display: grid;
  grid-template-columns: 100%;
  list-style-type: none;
  margin-block-start: 0em;
  margin-block-end: 0em;
  margin-inline-start: 0px;
  margin-inline-end: 0px;
  padding-inline-start: 0px;
}
.li {
  margin: 1px;
  text-align: left;
  font-size: 14px;
  height: 16px;
  border: 0.5px solid white;
  cursor: pointer;
}
.li:hover {
  margin: 1px;
  border-color: rgb(52, 52, 255);
  border-radius: 2px;
  background-color: rgb(59, 122, 231);
}
.fa {
  margin-left: 4px;
  margin-top: 3px;
}
.span {
  margin-left: 7px;
  margin-top: 0px;
  font-family: Arial, Helvetica, sans-serif;
}
.labelStyle {
  white-space: nowrap;
}
.div {
  display: grid;
  grid-template-columns: 1fr;
}
.listbox {
  display: grid;
  grid-template-columns: 100%;
  height: 100px;
  width: 300px;
  background-color: lightgray;
  border: 1px solid gray;
}
.columnHeads {
  height: 19.2px;
  border: 1px solid black;
  display: grid;
  grid-template-columns: 100%;
}
.optionsItems {
  display: grid;
  grid-template-columns: 100%;
}
.text-box-design {
  resize: none;
  overflow: hidden;
  border: none;
  font-family: monospace;
}
.text-box-design:focus {
  outline: none;
}
.text-box-design::selection {
  background: lightblue;
}
.table-style {
  width: 100%;
}
.thClass {
  position: sticky;
  top: 0;
  overflow: hidden;
  text-decoration: underline;
  white-space: nowrap;
}
.tdClass {
  width: 15px;
  border-right: 1px solid;
}
.tdClassIn {
  width: 10px;
}
.inputClass {
  margin: 0;
}
.hrStyle {
  margin: 0px;
}
.forArrow {
  background-image: url('../../../../assets/sort-down.png');
}
.forEllipsis {
  background-image: url('../../../../assets/ellipsis.png');
}
.forReduce {
  background-image: url('../../../../assets/minus.png'),url('../../../../assets/minus.png');
}
.forPlain {
  background-image: none;
}
</style>
