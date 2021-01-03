<template>
  <button
    class="commandbutton"
    :style="styleObj"
    :name="properties.Name"
    :tabindex="properties.TabIndex"
    :title="properties.ControlTipText"
    :runmode="getDisableValue"
    @blur="
      () => {
        isClicked = false;
      }
    "
    @mousedown="controlEditMode"
    @keydown.enter.prevent="setContentEditable($event, true)"
    @click.stop="commandButtonClick"
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
import { Component, Emit, Mixins, Prop, Watch } from 'vue-property-decorator'
import FdControlVue from '@/api/abstract/FormDesigner/FdControlVue'
import FDEditableText from '@/FormDesigner/components/atoms/FDEditableText/index.vue'

@Component({
  name: 'FDCommandButton',
  components: {
    FDEditableText
  }
})
export default class FDCommandButton extends Mixins(FdControlVue) {
  $el!: HTMLButtonElement;
  isClicked: boolean = false;
  isContentEditable: boolean = false;

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
   * @description commandButtonClick is a method to check the check the clicked functionality of the button tag.
   * Also It sets the variable isClicked based on the Locked property
   * @function commandButtonClick
   *
   */
  commandButtonClick (e: Event) {
    if (!this.isRunMode) {
      this.selectedItem(e)
    }
    if (this.isActivated) {
      if (this.properties.Locked) {
        this.isClicked = false
      } else {
        this.isClicked = true
      }
    }
    if (this.isEditMode) {
      (this.$el.children[0] as HTMLSpanElement).focus()
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
          value: (this.$el.childNodes[0] as HTMLSpanElement).offsetHeight + 5
        })
        this.updateDataModel({
          propertyName: 'Width',
          value: (this.$el.childNodes[0] as HTMLSpanElement).offsetWidth + 5
        })
      })
    }
  }

  /**
   * @description style object is passed to :style attribute in button tag
   * dynamically changing the styles of the component based on properties
   * @function styleObj
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
      ...(!controlProp.AutoSize && this.renderSize),
      ...this.baseStyle,
      left: `${controlProp.Left}px`,
      width: `${controlProp.Width}px`,
      height: `${controlProp.Height}px`,
      top: `${controlProp.Top}px`,
      borderTopColor: controlProp.Default ? 'black' : controlProp.BackColor,
      borderBottomColor: controlProp.Default ? 'black' : controlProp.BackColor,
      borderLeftColor: controlProp.Default ? 'black' : controlProp.BackColor,
      borderRightColor: controlProp.Default ? 'black' : controlProp.BackColor,
      outline: controlProp.Enabled
        ? controlProp.TakeFocusOnClick && this.isClicked
          ? '1px dotted black'
          : 'none'
        : 'none',
      outlineOffset:
        controlProp.TakeFocusOnClick && this.isClicked ? '-5px' : '0px',
      display: display,
      backgroundColor: controlProp.BackStyle ? controlProp.BackColor : 'transparent',
      color:
        controlProp.Enabled === true ? controlProp.ForeColor : this.getEnabled,
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
      backgroundImage: `url(${controlProp.Picture})`,
      backgroundRepeat: this.getRepeat,
      backgroundPosition: controlProp.Picture ? this.getPosition : '',
      backgroundPositionX: controlProp.Picture ? this.getPositionX : '',
      backgroundPositionY: controlProp.Picture ? this.getPositionY : ''
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
   * @description watches changes in propControlData to set autoset when true
   * @function autoSize
   * @param oldVal previous propControlData data
   * @param newVal  new/changed propControlData data
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
.commandbutton {
  overflow: hidden;
  display: inline-block;
  box-sizing: border-box;
  width: 0px;
  height: 0px;
  left: 0px;
  top: 0px;
}
.commandbutton[runmode]:active {
  border-style: outset !important;
}
</style>
