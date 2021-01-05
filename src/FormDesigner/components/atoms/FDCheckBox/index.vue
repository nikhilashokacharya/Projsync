<template>
  <div
    :title="properties.ControlTipText"
    class="outer-check"
    :style="cssStyleProperty"
    @click.stop="checkBoxClick"
    @keydown.enter.prevent="setContentEditable($event, true)"
    :tabindex="properties.TabIndex"
  >
    <label class="control" :style="controlStyleObj" v-if="properties.Alignment === 1">
      <input
        @change="handleChange($event, checkboxInput)"
        ref="checkboxInput"
        :name="properties.Name"
        :tabindex="properties.TabIndex"
        :disabled="getDisableValue"
        type="checkbox"
        class="control-input visually-hidden" />
      <span
        class="control-indicator"
        :style="controlIndicatorStyleObj"
        ref="spanRef"
      ></span
    ></label>
    <div :style="pictureDivStyle" v-if="properties.Alignment === 1">
      <div ref="divAutoSize" :style="divcssStyleProperty">
        <span
          v-if="!syncIsEditMode || isRunMode"
          @click="isRunMode && makeChecked($event)"
        >
          <span>{{ computedCaption.afterbeginCaption }}</span>
          <span class="spanStyle">{{
            computedCaption.acceleratorCaption
          }}</span>
          <span>{{ computedCaption.beforeendCaption }}</span>
        </span>
        <FDEditableText
          v-else
          ref="checkBoxSpanRef"
          :editable="isRunMode === false && syncIsEditMode"
          :caption="properties.Caption"
          :style="editCssObj"
          @updateCaption="updateCaption"
          @releaseEditMode="releaseEditMode"
        >
        </FDEditableText>
      </div>
    </div>
    <div :style="pictureDivStyle" v-if="properties.Alignment === 0">
      <div ref="divAutoSize" :style="divcssStyleProperty">
        <span
          v-if="!syncIsEditMode || isRunMode"
          @click="isRunMode && makeChecked($event)"
        >
          <span>{{ computedCaption.afterbeginCaption }}</span>
          <span class="spanStyle">{{
            computedCaption.acceleratorCaption
          }}</span>
          <span>{{ computedCaption.beforeendCaption }}</span>
        </span>
        <FDEditableText
          v-else
          ref="checkBoxSpanRef"
          :editable="isRunMode === false && syncIsEditMode"
          :caption="properties.Caption"
          :style="editCssObj"
          @updateCaption="updateCaption"
          @releaseEditMode="releaseEditMode"
        >
        </FDEditableText>
      </div>
    </div>
    <label class="control" :style="controlStyleObj" v-if="properties.Alignment === 0">
      <input
        @change="handleChange($event, checkboxInput)"
        ref="checkboxInput"
        :name="properties.Name"
        :tabindex="properties.TabIndex"
        :disabled="getDisableValue"
        type="checkbox"
        class="control-input visually-hidden" />
      <span
        class="control-indicator"
        :style="controlIndicatorStyleObj"
        ref="spanRef"
      ></span
    ></label>
  </div>
</template>

<script lang="ts">
import { Component, Ref, Mixins, Watch, Vue } from 'vue-property-decorator'
import FdControlVue from '@/api/abstract/FormDesigner/FdControlVue'
import FDEditableText from '@/FormDesigner/components/atoms/FDEditableText/index.vue'

@Component({
  name: 'FDCheckBox',
  components: {
    FDEditableText
  }
})
export default class FDCheckBox extends Mixins(FdControlVue) {
  @Ref('checkboxInput') checkboxInput!: HTMLInputElement;
  @Ref('divAutoSize') autoSizecheckbox!: HTMLDivElement;
  @Ref('spanRef') spanRef!: HTMLSpanElement;
  @Ref('checkBoxSpanRef') checkBoxSpanRef!: FDEditableText
  $el: HTMLDivElement
  alignItem: boolean = false

  get controlStyleObj () {
    const controlProp = this.properties
    return {
      position: 'sticky',
      top: `${controlProp.Height! / 2 - 10}px`
    }
  }
  /**
   * @description  watches Enabled property and the sets the backgroundColor
   * @function checkEnabled
   */
  @Watch('properties.Enabled', {
    deep: true
  })
  checkEnabled (newVal: boolean, oldVal: boolean) {
    if (!this.properties.Enabled) {
      this.spanRef.style.backgroundColor = 'rgba(220, 220, 220, 1)'
    } else {
      this.spanRef.style.backgroundColor = 'white'
    }
  }

  /**
   * @description  watches Value property and the sets the checked
   * @function verifyValue
   */
  @Watch('properties.Value', {
    deep: true
  })
  verifyValue () {
    if (this.isRunMode) {
      if (this.properties.Enabled && !this.properties.Locked) {
        this.handleValue(this.properties.Value! as string)
      }
    } else {
      this.handleValue(this.properties.Value! as string)
    }
  }

  handleValue (newVal: string) {
    let tempValue = newVal.toLowerCase()
    const checkDiv = this.checkboxInput
    if (!isNaN(parseInt(newVal))) {
      if (parseInt(newVal) === 0) {
        this.spanRef.style.backgroundColor = 'white'
        checkDiv.checked = false
      } else {
        this.spanRef.style.backgroundColor = 'white'
        checkDiv.checked = true
      }
    } else if (tempValue === 'true') {
      this.spanRef.style.backgroundColor = 'white'
      checkDiv.checked = true
    } else if (tempValue === 'false') {
      this.spanRef.style.backgroundColor = 'white'
      checkDiv.checked = false
    } else {
      checkDiv.checked = true
      this.spanRef.style.backgroundColor = 'rgba(220, 220, 220, 1)'
    }
  }
  /**
   * @description  makeChecked controls the checked of the control in RunMode
   * @function makeChecked
   */
  makeChecked () {
    if (!this.getDisableValue) {
      this.tripleState += 1
      const checkDiv = this.checkboxInput
      if (!this.properties.TripleState) {
        checkDiv.checked = !checkDiv.checked
        const tempval = checkDiv.checked
        this.updateDataModel({ propertyName: 'Value', value: tempval })
      } else {
        if (this.tripleState % 3 === 0) {
          checkDiv.checked = true
          this.spanRef.style.backgroundColor = 'rgba(220, 220, 220, 1)'
        } else {
          this.spanRef.style.backgroundColor = 'white'
          checkDiv.checked = !checkDiv.checked
          this.updateDataModel({
            propertyName: 'Value',
            value: checkDiv.checked
          })
        }
      }
    }
  }

  /**
   * @description getDisableValue checks for the RunMode of the control and then returns after checking for the Enabled
   * and the Locked property
   * @function getDisableValue
   */
  get getDisableValue () {
    if (this.isRunMode) {
      return this.properties.Enabled === false || this.properties.Locked
    } else {
      return true
    }
  }

  /**
   * @description style object is passed to :style attribute in span tag
   * dynamically changing the styles of the component based on properties
   * @function controlIndicatorStyleObj
   *
   */
  get controlIndicatorStyleObj () {
    const controlProp = this.properties
    return {
      boxShadow:
        controlProp.SpecialEffect === 0 ? '0px 0px gray' : '-1px -1px gray'
    }
  }

  /**
   * @description style object is passed to :style attribute in label tag
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
        FontSize: 20,
        FontItalic: true,
        FontBold: true,
        FontUnderline: true,
        FontStrikethrough: true
      }
    let display = ''
    if (this.isRunMode) {
      display = controlProp.Visible ? 'grid' : 'none'
    } else {
      display = 'grid'
    }
    return {
      left: `${controlProp.Left}px`,
      width: `${controlProp.Width}px`,
      height: `${controlProp.Height}px`,
      top: `${controlProp.Top}px`,
      backgroundColor: controlProp.BackStyle
        ? controlProp.BackColor
        : 'transparent',
      borderColor: controlProp.BorderColor,
      border: this.getBorderStyle,
      whiteSpace: controlProp.WordWrap ? 'pre-wrap' : 'pre',
      wordBreak: controlProp.WordWrap ? 'break-all' : 'normal',
      color:
        controlProp.Enabled === true ? controlProp.ForeColor : this.getEnabled,
      cursor:
        controlProp.MousePointer !== 0 || controlProp.MouseIcon !== ''
          ? this.getMouseCursorData
          : 'default',
      fontFamily: font.FontStyle! !== '' ? this.setFontStyle : font.FontName!,
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
      fontWeight: font.FontBold
        ? 'bold'
        : font.FontStyle !== ''
          ? this.tempWeight
          : '',
      fontStretch: font.FontStyle !== '' ? this.tempStretch : '',

      display: display,
      overflow: 'hidden',
      gridTemplateColumns: controlProp.Alignment === 1 ? '12px auto' : 'auto 12px',
      gridTemplateRows: '100%',
      gap: '2px',
      alignItems: font.FontSize! > 17 ? 'center' : '',
      alignContent: 'center',
      boxShadow: 'none'
    }
  }

  /**
   * @description style object is passed to :style attribute in div tag
   * dynamically changing the styles of the component based on properties
   * @function divcssStyleProperty
   *
   */
  get divcssStyleProperty () {
    const controlProp = this.properties
    return {
      overflow: 'hidden',
      height: !this.isEditMode ? '100%' : '',
      display: 'flex',
      justifyContent:
        controlProp.TextAlign === 0
          ? 'flex-start'
          : controlProp.TextAlign === 1
            ? 'center'
            : 'flex-end',
      alignItems: 'center',
      textAlign:
        controlProp.TextAlign === 0
          ? 'start'
          : controlProp.TextAlign === 1
            ? 'center'
            : 'end',
      position: !this.isEditMode ? 'relative' : 'absolute'
    }
  }

  /**
   * @description style object is passed to :style attribute in div tag
   * dynamically changing the styles of the component based on properties
   * @function pictureDivStyle
   *
   */
  get pictureDivStyle () {
    const controlProp = this.properties
    return {
      height: '100%',
      display: !this.isEditMode ? 'table-cell' : 'flex',
      alignItems: this.alignItem ? 'baseline' : 'center',
      backgroundImage: `url(${controlProp.Picture})`,
      backgroundRepeat: this.getRepeat,
      backgroundPosition: this.getPosition,
      backgroundPositionX: this.getPositionX,
      backgroundPositionY: this.getPositionY,
      position: 'relative'
    }
  }

  get setAlignment () {
    return {
      editMode: this.isEditMode,
      caption: this.properties.Caption
    }
  }

  @Watch('setAlignment', { deep: true })
  editableTextVerify () {
    if (this.isEditMode) {
      Vue.nextTick(() => {
        if (this.isEditMode && this.checkBoxSpanRef.$el.clientHeight > this.properties.Height!) {
          this.alignItem = true
        } else {
          this.alignItem = false
        }
      })
    }
  }

  /**
   * @description style object is passed to :style attribute in tag
   * dynamically changing the styles of the component based on properties
   * @function editCssObj
   *
   */
  protected get editCssObj (): Partial<CSSStyleDeclaration> {
    const controlProp = this.properties
    return {
      backgroundImage: 'none'
    }
  }

  /**
   * @override
   */
  @Watch('properties.AutoSize', {
    deep: true
  })
  updateAutoSize () {
    if (this.properties.AutoSize) {
      this.$nextTick(() => {
        let divRef: HTMLDivElement = this.autoSizecheckbox
        const offsetWidth = (divRef.childNodes[0] as HTMLSpanElement)
          .offsetWidth
        const offsetHeight = (divRef.childNodes[0] as HTMLSpanElement)
          .offsetHeight
        // Value 10 for checkbox, 4 for Gap style value, and offsetHeight and offsetWidth + 1
        this.updateDataModel({
          propertyName: 'Width',
          value: 10 + 4 + offsetWidth + 1
        })
        this.updateDataModel({
          propertyName: 'Height',
          value: 10 + 4 + offsetHeight + 1
        })
      })
    } else {
      return undefined
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

  @Watch('properties.Caption', { deep: true })
  autoSizeValidateOnCaptionChange () {
    if (this.properties.AutoSize) {
      this.updateAutoSize()
    }
  }

  /**
   * @description  sets controlSource if present and updates Value property
   * @function controlSource
   */
  mounted () {
    this.verifyValue()
    this.$el.focus()
    this.controlSource()
  }
  releaseEditMode (event: KeyboardEvent) {
    this.$el.focus()
    this.setContentEditable(event, false)
  }
  checkBoxClick (event: MouseEvent) {
    this.selectedItem(event)
    if (this.isEditMode) {
      (this.checkBoxSpanRef.$el as HTMLSpanElement).focus()
    }
  }
}
</script>

<style scoped>
* {
  box-sizing: border-box;
}

.outer-check {
  height: 0px;
  width: 0px;
  left: 0px;
  top: 0px;
  min-width: 12px;
  min-height: 13px;
  background-color: rgb(238, 238, 238);
  box-shadow: -1px -1px gray;
  overflow: hidden;
  align-items: center;
  box-sizing: border-box;
}

.visually-hidden {
  border: 0;
  clip: rect(0, 0, 0, 0);
  height: 1px;
  margin: -1px;
  overflow: hidden;
  padding: 0;
  position: absolute;
  width: 1px;
}

.control {
  display: inline-flex;
  position: sticky;
  top: 47%;
}

.control-indicator {
  width: 10px;
  height: 10px;
  margin: 1px;
  background-color: white;
  border: 1px inset grey;
}

.control-input:checked ~ .control-indicator {
  background-image: url(../../../../assets/checkmark.png);
  background-size: 10px;
  background-position: center;
  background-repeat: no-repeat;
}

.spanStyle {
  text-decoration: underline;
  text-underline-position: under;
}

.menu {
  width: 10%;
  margin: 0 auto;
}

.main {
  width: 90%;
  margin: 0 auto;
}
</style>
