<template>
<div :style="outerDivStyle" class="outerDivClass">
  <fieldset
    class="fieldset"
    :style="cssStyleProperty"
    :title="properties.ControlTipText"
    :tabindex="properties.TabIndex"
    @keydown.ctrl.exact.stop="selectMultipleCtrl($event,true)"
    @keydown.ctrl.stop="handleKeyDown"
    @keydown.shift.exact.stop="selectMultipleCtrl($event,true)"
    @keydown.delete.stop.exact="deleteFrame"
    @keydown.enter.exact="setContentEditable($event, true)"
    @click.stop="!isEditMode ? selectedItem : addControlObj($event)"
    @contextmenu.stop="showContextMenu($event, userFormId, controlId, 'container', isEditMode)"
    @mousedown="frameMouseDown"
    @mouseup="dragSelectorControl($event)"
    @keyup.stop="selectMultipleCtrl($event, false)"
  >
    <legend :style="legendCssStyleProperty">{{ properties.Caption }}</legend>
    <div :style="scrollSize" ref="frame" @scroll="updateScrollingLeftTop">
    <Container
      :contextMenuType="contextMenuType"
      :viewMenu="viewMenu"
      :userFormId="userFormId"
      :controlId="controlId"
      :containerId="controlId"
      :isEditMode="isEditMode"
      :left="left"
      :top="top"
      ref="containerRef"
      @closeMenu="closeMenu"
      @openMenu="(e, parentID, controlID, type, mode) => showContextMenu(e, parentID, controlID, type, mode)"
    />
  </div>
  </fieldset>
</div>
</template>

<script lang="ts">
import { Component, Emit, Prop, Ref, Watch } from 'vue-property-decorator'
import FdControlVue from '@/api/abstract/FormDesigner/FdControlVue'
import { Action } from 'vuex-class'
import Vue from 'vue'
import FdContainerVue from '@/api/abstract/FormDesigner/FdContainerVue'
import Container from '@/FormDesigner/components/organisms/FDContainer/index.vue'
import { controlProperties } from '@/FormDesigner/controls-properties'
import { EventBus } from '@/FormDesigner/event-bus'
@Component({
  name: 'FDFrame',
  components: {
    Container: () =>
      import('@/FormDesigner/components/organisms/FDContainer/index.vue')
  }
})
export default class FDFrame extends FdContainerVue {
  @Ref('containerRef') readonly containerRef!: Container;
  @Ref('frame') readonly frame!: HTMLDivElement;
  @Prop({ required: true, type: Boolean }) public readonly isEditMode: boolean;
  mode: boolean = false;

  /**
   * @description Returns string value for CSS background style
   * @function createBackgroundString
   */
  protected get createBackgroundString () {
    return `url(${this.properties.Picture})`
  }
  /**
   * @description Returns string value for CSS background style for dotted patten
   * @function getSampleDotPattern
   */
  protected get getSampleDotPattern () {
    const dotSize = 1
    const dotSpace = 10
    return {
      backgroundPosition: `7px 7px`,
      backgroundImage: `radial-gradient(${this.properties.ForeColor} 11%, transparent 10%)`,
      backgroundSize: `${dotSpace}px ${dotSpace}px`
    }
  }

  mounted () {
    this.scrollLeftTop(this.data)
  }

  @Watch('properties.ScrollLeft')
  updateScrollLeft () {
    this.scrollLeftTop(this.data)
  }

  @Watch('properties.ScrollTop')
  updateScrollTop () {
    this.scrollLeftTop(this.data)
  }

  @Watch('properties.Visible')
  updateViisible () {
    this.updateEditMode(false)
  }

  @Emit('updateEditMode')
  updateEditMode (val: boolean) {
    return val
  }

  /**
   * @description sets scrollbar left and top position
   * @function scrollLeftTop
   * @param controlData propControlData passed as input
   */
  scrollLeftTop (controlData: controlData) {
    const scrollLeft: number = this.properties.ScrollLeft!
    const scrollTop: number = this.properties.ScrollTop!
    if (scrollLeft > 0) {
      (this.frame as IScrollRef).scrollLeft = scrollLeft
    }
    if (scrollTop > 0) {
      (this.frame as IScrollRef).scrollTop = scrollTop
    }
  }

  /**
   * @description style object is passed to :style attribute in div tag
   * dynamically changing the styles of the component based on propControlData
   * @function outerDivStyle
   *
   */
  protected get outerDivStyle (): Partial<CSSStyleDeclaration> {
    const controlProp = this.data.properties
    return {
      backgroundColor: controlProp.BackColor,
      width: `${controlProp.Width}px`,
      height: `${controlProp.Height}px`,
      boxShadow: controlProp.SpecialEffect ? this.getSpecialEffectData : 'none',
      borderLeft: controlProp.BorderStyle
        ? `0.3px solid ${controlProp.BorderColor}`
        : '0.3px solid gray',
      borderRight: controlProp.BorderStyle
        ? `0.3px solid ${controlProp.BorderColor}`
        : '0.3px solid gray',
      borderBottom: controlProp.BorderStyle
        ? `0.3px solid ${controlProp.BorderColor}`
        : '0.3px solid gray',
      overflow: 'hidden'
    }
  }

  /**
   * @description style object is passed to :style attribute in fieldset tag
   * dynamically changing the styles of the component based on propControlData
   * @function cssStyleProperty
   *
   */
  protected get cssStyleProperty (): Partial<CSSStyleDeclaration> {
    const controlProp = this.data.properties
    const font: font = this.properties.Font
      ? this.properties.Font
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
      display = controlProp.Visible ? 'block' : 'none'
    } else {
      display = 'block'
    }
    return {
      position: 'relative',
      width: `${controlProp.Width! - 3}px`,
      height: `${controlProp.Height}px`,
      marginLeft: '2px',
      padding: '0px',
      outline: 'none',
      cursor: controlProp.MousePointer !== 0 || controlProp.MouseIcon !== ''
        ? `${this.getMouseCursorData} !important`
        : 'default !important',
      fontFamily: font.FontStyle! !== '' ? this.setFontStyle : font.FontName!,
      fontSize: `${font.FontSize}px`,
      fontStyle: font.FontItalic || this.isItalic ? 'italic' : '',
      textDecoration: font.FontStrikethrough === true && font.FontUnderline === true
        ? 'underline line-through'
        : font.FontUnderline
          ? 'underline'
          : font.FontStrikethrough
            ? 'line-through'
            : '',
      fontWeight: font.FontBold ? 'bold'
        : font.FontStyle !== ''
          ? this.tempWeight
          : '',
      fontStretch: font.FontStyle !== '' ? this.tempStretch : '',
      borderTop: controlProp.BorderStyle
        ? `1px solid ${controlProp.BorderColor}`
        : '1px solid gray',
      borderLeft: 'none',
      borderRight: 'none',
      borderBottom: 'none',
      backgroundImage: controlProp.Picture === ''
        ? this.getSampleDotPattern.backgroundImage
        : this.createBackgroundString,
      backgroundSize: controlProp.Picture === ''
        ? this.getSampleDotPattern.backgroundSize
        : this.getSizeMode,
      backgroundColor: controlProp.Picture !== '' ? '' : controlProp.BackColor,
      backgroundRepeat: this.getRepeatData,
      backgroundPosition: controlProp.Picture !== ''
        ? this.getPosition
        : this.getSampleDotPattern.backgroundPosition,
      display: display,
      zoom: `${controlProp.Zoom}%`,
      whiteSpace: 'nowrap',
      textOverflow: 'ellipsis',
      maxWidth: `${controlProp.Width!}px`
    }
  }
  /**
   * @description style object is passed to :style attribute in legend tag
   * dynamically changing the styles of the component based on propControlData
   * @function legendCssStyleProperty
   *
   */
  protected get legendCssStyleProperty (): Partial<CSSStyleDeclaration> {
    const controlProp = this.data.properties
    return {
      position: 'sticky',
      top: '0px',
      color:
        controlProp.Enabled === true ? controlProp.ForeColor : this.getEnabled,
      background: controlProp.BackColor,
      whiteSpace: 'pre',
      wordBreak: 'normal',
      overflow: 'hidden',
      maxWidth: `${controlProp.Width! - 20}px`
    }
  }
  get scrollSize (): Partial<CSSStyleDeclaration> {
    const controlProp = this.data.properties!
    return {
      width: `${controlProp.Width! - 3}px`,
      height: `${controlProp.Height! - 10}px`,
      overflowX: this.getScrollBarX,
      overflowY: this.getScrollBarY
    }
  }

  showContextMenu (e: MouseEvent, parentID: string, controlID: string, type: string, mode: boolean) {
    this.openMenu(e, parentID, controlID, type, mode)
    Vue.nextTick(() => this.containerRef.contextmenu.focus())
  }
  closeMenu () {
    this.viewMenu = false
  }

  dragSelectorControl (event: MouseEvent) {
    this.selectedControlArray = []
    this.selectedAreaStyle = this.containerRef.dragSelector.selectAreaStyle
    this.calSelectedAreaStyle(event, this.data)
  }

  frameMouseDown (e: MouseEvent) {
    EventBus.$emit('isEditMode', this.isEditMode)
    this.selectedItem(e)
    const selContainer = this.selectedControls[this.userFormId].container[0]
    if (selContainer === this.controlId) {
      this.deActiveControl()
    } else {
      return null
    }
  }

  /**
   * @description To perform ContextMenu actions(for example: selectAll, paste etc.) on UserForm  and Control
   * @function handleKeyDown
   * @param event  - it is of type MouseEvent
   * @event keydown
   */
  handleKeyDown (event: KeyboardEvent) {
    this.containerRef.refContextMenu.updateAction(event)
  }

  deleteFrame (event: KeyboardEvent) {
    if (this.controlId === this.selectedControls[this.userFormId].selected[0]) {
      this.deleteItem(event)
    } else {
      this.handleKeyDown(event)
    }
  }
  updateScrollingLeftTop (e: MouseEvent) {
    const refName = this.frame
    this.updateControl({
      userFormId: this.userFormId,
      controlId: this.controlId,
      propertyName: 'ScrollLeft',
      value: refName.scrollLeft
    })
    this.updateControl({
      userFormId: this.userFormId,
      controlId: this.controlId,
      propertyName: 'ScrollTop',
      value: refName.scrollTop
    })
  }
  selectMultipleCtrl (event: KeyboardEvent, val: boolean) {
    if (event.key === 'Control' && event.keyCode === 17) {
      this.selMultipleCtrl = val
      EventBus.$emit('selectMultipleCtrl', val)
    } else if (event.key === 'Shift' && event.keyCode === 16) {
      this.activateCtrl = val
      EventBus.$emit('actMultipleCtrl', val)
    }
  }
}
</script>

<style scoped>
.fieldset {
  box-sizing: border-box;
  margin: 0px;
  user-select: none;
}
</style>
