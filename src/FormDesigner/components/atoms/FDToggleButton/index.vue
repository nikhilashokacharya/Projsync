<template>
  <button
    class="toggle-button"
    :style="styleObj"
    :name="properties.Name"
    :tabindex="properties.TabIndex"
    :title="properties.ControlTipText"
    :runmode="getDisableValue"
    @mousedown="controlEditMode"
    @keydown.enter.prevent="setContentEditable($event, true)"
    @blur="
      () => {
        isClicked = false;
        isFocus = false;
      }
    "
    @click.stop="toggleButtonClick"
  >
    <span v-if="!syncIsEditMode || isRunMode">
      <span>{{ computedCaption.afterbeginCaption }}</span>
      <span class="spanClass">{{ computedCaption.acceleratorCaption }}</span>
      <span>{{ computedCaption.beforeendCaption }}</span>
    </span>
    <FDEditableText
      v-else
      :editable="isRunMode === false && syncIsEditMode"
      :style="editCssObj"
      :caption="properties.Caption"
      @updateCaption="updateCaption"
      @releaseEditMode="releaseEditMode"
    >
    </FDEditableText>
  </button>
</template>

<script lang="ts">
import { Component, Mixins, Watch } from 'vue-property-decorator'
import FdControlVue from '@/api/abstract/FormDesigner/FdControlVue'
import FDEditableText from '@/FormDesigner/components/atoms/FDEditableText/index.vue'

@Component({
  name: 'FDToggleButton',
  components: {
    FDEditableText
  }
})
export default class FDToggleButton extends Mixins(FdControlVue) {
  $el!: HTMLButtonElement;
  isClicked: boolean = true;
  isFocus: boolean = false;
  clickCount: number = 0;

  /**
   * @description getDisableValue checks for the RunMode of the control and then returns after checking for the Enabled
   * and the Locked property
   * @function getDisableValue
   */
  get getDisableValue () {
    if (this.isRunMode) {
      return (
        this.properties.Enabled === false || this.properties.Locked === true
      )
    } else {
      return true
    }
  }

  /**
   * @description toggleButtonClick is a method to check the check the clicked functionality of the button tag.
   * Also It sets the variables isClicked and isFocus based on the Locked property
   * @function toggleButtonClick
   * @param MouseEvent
   */
  toggleButtonClick (e: MouseEvent) {
    if (this.isRunMode) {
      this.clickCount = this.clickCount + 1
      if (this.properties.Locked === false) {
        this.isFocus = true
        this.isClicked = !this.isClicked
        if (this.properties.TripleState) {
          if (this.clickCount % 3 === 0) {
            this.updateDataModel({ propertyName: 'Value', value: '' })
          } else if (this.isClicked) {
            this.updateDataModel({ propertyName: 'Value', value: 'True' })
          } else {
            this.updateDataModel({ propertyName: 'Value', value: 'False' })
          }
        } else if (this.isClicked) {
          this.updateDataModel({ propertyName: 'Value', value: 'True' })
        } else {
          this.updateDataModel({ propertyName: 'Value', value: 'False' })
        }
      } else {
        this.isClicked = false
      }
    }
    this.selectedItem(e)
    if (this.isEditMode) {
      (this.$el.children[0] as HTMLSpanElement).focus()
    }
  }

  /**
   * @description style object is passed to :style attribute in button tag
   * dynamically changing the styles of the component based on properties
   * @function styleObj
   *
   */
  protected get styleObj (): Partial<CSSStyleDeclaration> {
    const controlProp = this.properties
    const font: font = controlProp.Font
      ? controlProp.Font
      : {
        FontName: 'Arial',
        FontSize: 10,
        FontItalic: true,
        FontBold: true,
        FontUnderline: true,
        FontStrikethrough: true
      }
    let display = ''
    if (this.isRunMode) {
      display = controlProp.Visible ? 'inline-block' : 'none'
    } else {
      display = 'inline-block'
    }
    return {
      left: `${controlProp.Left}px`,
      width: `${controlProp.Width}px`,
      height: `${controlProp.Height}px`,
      top: `${controlProp.Top}px`,
      boxShadow: controlProp.Enabled
        ? controlProp.Value === 'False' || controlProp.Value === 'false'
          ? '1px 1px gray'
          : controlProp.Value === 'True' || controlProp.Value === 'true'
            ? '-1px -1px black'
            : '1px 1px gray'
        : '1px 1px gray',
      backgroundColor: controlProp.BackStyle ? controlProp.BackColor : 'transparent',
      outline: controlProp.Enabled
        ? this.isFocus
          ? '1px dotted black'
          : 'none'
        : 'none',
      outlineOffset: this.isClicked ? '-5px' : '-5px',
      display: display,
      color:
        controlProp.Enabled === true &&
        (controlProp.Value === 'False' ||
          controlProp.Value === 'false' ||
          controlProp.Value === 'True' ||
          controlProp.Value === 'true')
          ? controlProp.ForeColor
          : this.getEnabled,
      cursor:
        controlProp.MousePointer !== 0 || controlProp.MouseIcon !== ''
          ? this.getMouseCursorData
          : 'default',
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
      whiteSpace: controlProp.WordWrap ? 'pre-wrap' : 'pre',
      wordBreak: controlProp.WordWrap ? 'break-all' : 'normal',
      paddingLeft: controlProp.AutoSize ? '0px' : '0px',
      paddingRight: controlProp.WordWrap ? '0px' : '6px',
      textAlign:
        controlProp.TextAlign === 0
          ? 'left'
          : controlProp.TextAlign === 1
            ? 'center'
            : 'right',
      backgroundImage: `url(${controlProp.Picture})`,
      backgroundRepeat: this.getRepeat,
      backgroundPosition: controlProp.Picture ? this.getPosition : '',
      backgroundPositionX: controlProp.Picture ? this.getPositionX : '',
      backgroundPositionY: controlProp.Picture ? this.getPositionY : '',
      borderLeft: controlProp.Value !== 'True' ? '1px solid' : '',
      borderTop: controlProp.Value !== 'True' ? '1px solid' : '',
      borderRight: controlProp.Value === 'True' ? '1px solid' : '',
      borderBottom: controlProp.Value === 'True' ? '1px solid' : '',
      borderTopColor: controlProp.Value !== 'True' ? 'white !important' : '',
      borderLeftColor: controlProp.Value !== 'True' ? 'white !important' : '',
      borderBottomColor: controlProp.Value === 'True' ? 'white !important' : '',
      borderRightColor: controlProp.Value === 'True' ? 'white !important' : ''
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
   * @description watches changes in properties to set autoset when true
   * @function autoSize
   * @param oldVal previous properties data
   * @param newVal  new/changed properties data
   */
  @Watch('properties.AutoSize', { deep: true })
  autoSize (newVal: boolean, oldVal: boolean) {
    // if autoSize is true then height and width value will not get updated
    this.updateAutoSize()
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
   * @description changes width and height when autoSize is true by getting content offsetWidth
   *  and offsetHeight with the help of Ref attribute
   * @function updateAutoSize
   * @override
   */
  updateAutoSize () {
    if (this.properties.AutoSize === true) {
      this.$nextTick(() => {
        this.updateDataModel({
          propertyName: 'Height',
          value:
            (this.$el.childNodes[0] as HTMLSpanElement)
              .offsetHeight + 5
        })
        this.updateDataModel({
          propertyName: 'Width',
          value:
            (this.$el.childNodes[0] as HTMLSpanElement)
              .offsetWidth + 5
        })
      })
    }
  }

  /**
   * @description mounted initializes the values which are required for the component
   */
  mounted () {
    this.$el.focus()
    this.updateAutoSize()
  }
  releaseEditMode (event: KeyboardEvent) {
    this.$el.focus()
    this.setContentEditable(event, false)
  }
}
</script>

<style scoped>
.toggle-button {
  width: 0px;
  height: 0px;
  left: 0px;
  top: 0px;
  box-shadow: 1px 1px gray;
  border: none;
  overflow: hidden;
  outline: none;
  box-sizing: border-box;
}
</style>
