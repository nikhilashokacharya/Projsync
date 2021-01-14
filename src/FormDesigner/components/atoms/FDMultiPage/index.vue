<template>
  <div>
    <div
      class="outer-page"
      :style="pageStyleObj"
      :title="properties.ControlTipText"
      @mousedown="multiPageMouseDown"
      @click.stop="
        !isEditMode ? selectedItem : addControlObj($event, selectedPageID)
      "
      @mouseup="dragSelectorControl($event)"
      @contextmenu.stop="handleContextMenu"
      @keydown.delete.stop.exact="deleteMultiPage"
      @keyup.stop="selectMultipleCtrl($event, false)"
      :tabindex="properties.TabIndex"
    >
      <div
        class="pages"
        :style="styleTabsObj"
        :title="properties.ControlTipText"
        v-if="controls.length > 0"
      >
        <div class="move" ref="scrolling" :style="styleMoveObj">
          <div
            ref="controlTabsRef"
            class="page"
            v-for="(value, key) in controls"
            :key="key"
            :style="getTabStyle"
          >
            <FDControlTabs
              @setValue="setValue"
              @isChecked="isChecked"
              :setFontStyle="setFontStyle"
              @tempStretch="tempStretch"
              @deleteMultiPageControl="
                (event) => {
                  deleteMultiPageControl(event);
                }
              "
              :data="data"
              :pageValue="value"
              :indexValue="key"
              :pageData="pageData(value).properties"
              :isRunMode="isRunMode"
              :isEditMode="isEditMode"
              :isItalic="isItalic"
              :tempStretch="tempStretch"
              :tempWeight="tempWeight"
              :tempWidth="tempWidth"
              ref="controlTab"
            />
          </div>
        </div>
        <div
          class="content"
          :style="styleContentObj"
          ref="contentRef"
          :title="properties.ControlTipText"
          @scroll="updateScrollingLeftTop"
        >
          <div
            v-if="controls.includes(selectedPageID)"
            :style="containerDivStyle"
            :title="properties.ControlTipText"
            :tabindex="properties.TabIndex"
            @keydown.delete.exact="deleteMultiPageControl"
            @keydown.ctrl.exact.stop="selectMultipleCtrl($event, true)"
            @keydown.ctrl.stop="handleKeyDown"
            @keydown.enter.exact="setContentEditable($event, true)"
            @keydown.shift.exact.stop="selectMultipleCtrl($event, true)"
            @contextmenu.stop="
              showContextMenu($event, userFormId, controlId, 'container', isEditMode)
            "
          >
            <Container
              :contextMenuType="contextMenuType"
              :viewMenu="viewMenu"
              :userFormId="userFormId"
              :controlId="selectedPageID"
              :containerId="selectedPageID"
              :isEditMode="isEditMode"
              :title="properties.ControlTipText"
              :left="left"
              :top="top"
              :width="properties.Width"
              :height="properties.Height"
              ref="containerRef"
              @closeMenu="closeMenu"
              @openMenu="
                (e, parentID, controlID, type, mode) =>
                  showContextMenu(e, parentID, controlID, type, mode)
              "
            />
          </div>
        </div>
        <div></div>
        <div :style="getScrollButtonStyleObj">
          <button class="left-button" @click="leftmove"></button>
          <button class="right-button" @click="rightmove"></button>
        </div>
      </div>
    </div>
    <div
      id="right-click-menu"
      ref="multipage"
      :tabindex="0"
      @blur.stop="closeContextMenu"
      :style="{
        top: top,
        left: left,
        display: multiPageContextMenu ? 'block' : 'none',
      }"
    >
      <ContextMenu
        :values="contextMenuValue"
        :controlId="controlId"
        :selectedTab="updatedValue"
        :data="data"
        :userFormId="userFormId"
      />
    </div>
  </div>
</template>

<script lang="ts">
import { Component, Prop, Ref, Watch } from 'vue-property-decorator'
import FdControlVue from '@/api/abstract/FormDesigner/FdControlVue'
import { State, Action } from 'vuex-class'
import FdContainerVue from '@/api/abstract/FormDesigner/FdContainerVue'
import { controlProperties } from '@/FormDesigner/controls-properties'
import ContextMenu from '../FDContextMenu/index.vue'
import { tabsContextMenu } from '../../../models/tabsContextMenu'
import Vue from 'vue'
import {
  KeyValueProp,
  ScrollBarProp
} from '@/FormDesigner/controls-properties-types'
import FDControlTabs from '@/FormDesigner/components/atoms/FDControlTabs/index.vue'
import Container from '@/FormDesigner/components/organisms/FDContainer/index.vue'
import { EventBus } from '@/FormDesigner/event-bus'

@Component({
  name: 'FDMultiPage',
  components: {
    ContextMenu,
    FDControlTabs,
    Container: () =>
      import('@/FormDesigner/components/organisms/FDContainer/index.vue')
  }
})
export default class FDMultiPage extends FdContainerVue {
  @State((state) => state.fd.userformData) userformData!: userformData;
  @Prop() isEditMode: boolean;
  @Prop({ required: true, type: String }) public userFormId!: string;
  @Ref('scrolling') scrolling: HTMLDivElement;
  @Ref('contentRef') contentRef: HTMLDivElement;
  @Ref('containerRef') readonly containerRef!: Container;
  @Ref('multipage') multipage: HTMLDivElement;
  @Ref('controlTabsRef') controlTabsRef: HTMLDivElement[];
  @Ref('controlTab') controlTab: FDControlTabs[];

  viewMenu: boolean = false;
  top: string = '0px';
  left: string = '0px';
  contextMenuValue: Array<IcontextMenu> = tabsContextMenu;
  updatedValue: number = 0;
  selectedPageID: string = '';
  multiPageContextMenu: boolean = false;
  tempWidth: number = 0;
  tempHeight: number = 0;
  multiRowCount: number = 1;
  isScrollVisible = false;

  closeMenu () {
    this.viewMenu = false
  }

  @Watch('selectedPageData.properties.ScrollLeft')
  updateScrollLeft () {
    if (this.selectedPageData) {
      this.scrollLeftTop(this.selectedPageData!)
    }
  }

  @Watch('selectedPageData.properties.ScrollTop')
  updateScrollTop () {
    if (this.selectedPageData) {
      this.scrollLeftTop(this.selectedPageData!)
    }
  }
  /**
   * @description sets scrollbar left and top position
   * @function scrollLeftTop
   * @param controlData propControlData passed as input
   */
  scrollLeftTop (controlData: controlData) {
    if (this.selectedPageData) {
      const scrollLeft: number = controlData!.properties.ScrollLeft!
      const scrollTop: number = controlData!.properties.ScrollTop!
      console.log(this.$refs)
      if (scrollLeft > 0) {
        (this.contentRef as IScrollRef).scrollLeft = scrollLeft
      }
      if (scrollTop > 0) {
        (this.contentRef as IScrollRef).scrollTop = scrollTop
      }
    }
  }
  focusPage () {
    if (typeof this.properties.Value === 'number' && this.controls.length > 0) {
      const value: number = this.properties.Value as number;
      (this.controlTab[value].$el.children[1] as HTMLSpanElement).focus()
    } else {
      (this.$el.children[0] as HTMLDivElement).focus()
    }
  }
  closeContextMenu () {
    this.multiPageContextMenu = false
    this.focusPage()
  }
  /**
   * @description takes a single page value based on the value of the control
   * @function pageData
   *
   */
  pageData (value: string): controlData {
    return this.userformData[this.userFormId][value]
  }

  get outerDivStyleObj () {
    return {
      width: `${this.properties.Width!}px`,
      height: `${this.properties.Width!}px`
    }
  }
  /**
   * @description style object is passed to :style attribute in div tag
   * dynamically changing the styles of the component based on data
   * @function pageStyleObj
   *
   */
  get pageStyleObj () {
    const controlProp = this.properties
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
      backgroundColor: controlProp.BackColor,
      display: display,
      borderLeft: '2px solid whitesmoke',
      borderBottom: controlProp.TabOrientation === 0 ? '1px solid gray' : ''
    }
  }

  /**
   * @description style object is passed to :style attribute in div tag
   * dynamically changing the styles of the component based on data
   * @function styleTabsObj
   *
   */
  protected get styleTabsObj (): Partial<CSSStyleDeclaration> {
    const controlProp = this.properties
    return {
      overflow: 'hidden',
      display: 'flex',
      justifyContent: controlProp.TabOrientation === 3 ? 'flex-end' : '',
      height: `${controlProp.Height!}px`
    }
  }

  /**
   * @description style object is passed to :style attribute in div tag
   * dynamically changing the styles of the component based on data
   * @function styleMoveObj
   *
   */
  protected get styleMoveObj (): Partial<CSSStyleDeclaration> {
    const controlProp = this.properties
    return {
      alignSelf: controlProp.TabOrientation === 1 ? 'flex-end' : '',
      float: controlProp.TabOrientation === 3 ? 'right' : '',
      whiteSpace: controlProp.MultiRow === true ? 'break-spaces' : 'nowrap',
      zIndex: controlProp.MultiRow === true ? '100' : '',
      display: controlProp.Style === 2 ? 'none' : 'inline-block',
      height:
        controlProp.TabOrientation === 2 || controlProp.TabOrientation === 3
          ? this.isScrollVisible
            ? `${controlProp.Height! - 54}px`
            : `${controlProp.Height}px`
          : '',
      width:
        controlProp.TabOrientation === 2 || controlProp.TabOrientation === 3
          ? ''
          : !this.isScrollVisible
            ? `${controlProp.Width}px`
            : `${controlProp.Width! - 60}px`,
      overflow: 'hidden'
    }
  }

  /**
   * @description style object is passed to :style attribute in div tag
   * dynamically changing the styles of the component based on data
   * @function containerDivStyle
   *
   */
  get containerDivStyle () {
    if (this.selectedPageData) {
      let zoomVal = this.selectedPageData
        ? this.selectedPageData.properties.Zoom! / 100
        : ''
      return {
        height: '100%',
        width: '100%',
        position: 'relative',
        backgroundImage: `url(${this.selectedPageData.properties.Picture})`,
        backgroundSize:
          this.selectedPageData.properties.Picture === ''
            ? ''
            : this.getSizeMode,
        backgroundRepeat: this.getRepeatData,
        backgroundPosition:
          this.selectedPageData.properties.Picture === ''
            ? ''
            : this.getPosition,
        zoom: zoomVal + ''
      }
    }
  }
  /**
   * @description style object is passed to :style attribute in button tags
   * dynamically changing the styles of the component based on data
   * @function getScrollButtonStyleObj
   *
   */
  protected get getScrollButtonStyleObj (): Partial<CSSStyleDeclaration> {
    const controlProp = this.properties
    const tabsLength =
      this.properties.TabFixedWidth! > 0
        ? this.controls.length * this.properties.TabFixedWidth! +
          10 * this.controls!.length
        : this.properties.Font!.FontSize! < 36
          ? this.properties.Font!.FontSize! * 3.5 * this.controls!.length
          : this.properties.Font!.FontSize! * 2.3 * this.controls!.length
    const tabsHeight =
      this.properties.TabFixedHeight! > 0
        ? this.controls.length * this.properties.TabFixedHeight! +
          this.properties.Font!.FontSize! * this.controls!.length
        : this.properties.Font!.FontSize! * 2.3 * this.controls!.length
    return {
      position: 'absolute',
      zIndex: '3',
      marginTop:
        controlProp.TabOrientation === 2 || controlProp.TabOrientation === 3
          ? `${controlProp.Height! - 30}px`
          : controlProp.TabOrientation === 1
            ? `${controlProp.Height! - 22}px`
            : '0px',
      transform:
        controlProp.TabOrientation === 2
          ? 'rotate(90deg)'
          : this.transformScrollButtonStyle,
      display: !this.properties.MultiRow
        ? this.isScrollVisible
          ? 'block'
          : 'none'
        : 'none',
      right:
        controlProp.TabOrientation === 3
          ? '-14px'
          : controlProp.TabOrientation === 2
            ? `${controlProp.Width! - 40}px`
            : '0px',
      top: '0px'
    }
  }

  scrollButtonVerify () {
    this.isScrollVisible = false
    let sum = 0
    Vue.nextTick(() => {
      if (
        this.properties.TabOrientation === 0 ||
        this.properties.TabOrientation === 1
      ) {
        if (this.scrolling && !this.properties.MultiRow) {
          for (let i = 0; i < this.scrolling.children.length; i++) {
            const a = this.scrolling.children[i] as HTMLDivElement
            sum += a.offsetWidth
          }
          const tabsLength = sum
          if (tabsLength > this.properties.Width!) {
            this.isScrollVisible = true
          } else {
            this.isScrollVisible = false
          }
        }
      } else {
        if (this.scrolling && !this.properties.MultiRow) {
          for (let i = 0; i < this.scrolling.children.length; i++) {
            const a = this.scrolling.children[i] as HTMLDivElement
            sum += a.offsetHeight
          }
          const tabsHeight = sum
          if (tabsHeight > this.properties.Height!) {
            this.isScrollVisible = true
          } else {
            this.isScrollVisible = false
          }
        }
      }
    })
  }

  /**
   * @description takes the index Value and pageValue and set the Value property
   * @function isChecked
   *
   */
  isChecked (value: selectedTab) {
    if (this.isEditMode) {
      this.updatedValue = value.indexValue
      this.selectedPageID = value.pageValue
      this.updateDataModel({ propertyName: 'Value', value: value.indexValue })
      this.selectControl({
        userFormId: this.userFormId,
        select: {
          container: this.getContainerList(this.selectedPageID),
          selected: [this.selectedPageID]
        }
      })
      this.focusPage()
    }
  }

  /**
   * @description takes the ref of the div and determines the scrollLeft and scrollTop
   * @function leftmove
   *
   */
  leftmove () {
    const scrollRef = this.scrolling
    if (
      this.properties.TabOrientation === 0 ||
      this.properties.TabOrientation === 1
    ) {
      scrollRef.scrollLeft! -= 50
    } else {
      scrollRef.scrollTop! -= 50
    }
  }

  /**
   * @description takes the ref of the div and determines the scrollLeft and scrollTop
   * @function rightmove
   *
   */
  rightmove () {
    const scrollRef = this.scrolling
    let tempScrollTop = scrollRef.scrollTop!
    if (
      this.properties.TabOrientation === 0 ||
      this.properties.TabOrientation === 1
    ) {
      scrollRef.scrollLeft! += 50
    } else {
      tempScrollTop += 50
      scrollRef.scrollTop = tempScrollTop
    }
  }

  /**
   * @description takes the index Value and sets the Value property
   * @function setValue
   *
   */
  setValue (value: number) {
    this.updatedValue = value
    this.updateDataModel({ propertyName: 'Value', value: value })
    return true
  }
  /**
   * @description style object is passed to :style attribute in div tag
   * dynamically changing the styles of the component based on data
   * @function getTabStyle
   *
   */
  protected get getTabStyle (): Partial<CSSStyleDeclaration> {
    const controlProp = this.properties
    return {
      display:
        controlProp.TabOrientation === 0 || controlProp.TabOrientation === 1
          ? 'inline-block'
          : 'block',
      height:
        controlProp.TabFixedHeight! > 0
          ? `${controlProp.TabFixedHeight! + 10}px`
          : ''
    }
  }

  /**
   * @description getRepeat returns string value from
   * controlProperties.extraDataRepeatProp
   * @function getRepeat
   */
  protected get getRepeatData (): string {
    if (this.selectedPageData) {
      const picture = this.selectedPageData.properties.Picture!
      const pictureTiling = this.selectedPageData.properties.PictureTiling!
      const pictureSizeMode = this.selectedPageData.properties.PictureSizeMode!
      return controlProperties.getRepeatDataProp(
        picture,
        pictureTiling,
        pictureSizeMode
      )
    } else {
      return ''
    }
  }

  /**
   * @description style object is passed to :style attribute in div tag
   * dynamically changing the styles of the component based on data
   * @function styleContentObj
   *
   */
  protected get styleContentObj (): Partial<CSSStyleDeclaration> {
    const controlProp = this.properties
    return {
      position: 'absolute',
      display:
        controlProp.Height! < controlProp.TabFixedHeight!
          ? 'none'
          : controlProp.Width! < controlProp.TabFixedWidth!
            ? 'none'
            : controlProp.Width! < 30 || controlProp.Height! < 30
              ? 'none'
              : 'block',
      top:
        controlProp.Style !== 2
          ? controlProp.TabOrientation === 0
            ? controlProp.MultiRow
              ? (this.tempHeight + 12) * this.multiRowCount + 'px'
              : controlProp.TabFixedHeight! > 0
                ? controlProp.TabFixedHeight! + 10 + 'px'
                : controlProp.TabFixedHeight! === 0
                  ? this.tempHeight + 16 + 'px'
                  : '33px'
            : '0px'
          : '0px',
      height:
        controlProp.Style !== 2
          ? controlProp.TabOrientation === 0 || controlProp.TabOrientation === 1
            ? controlProp.MultiRow
              ? controlProp.Height! -
                (this.tempHeight + 12) * this.multiRowCount +
                'px'
              : controlProp.TabFixedHeight! > 0
                ? controlProp.TabOrientation === 0
                  ? controlProp.Height! - controlProp.TabFixedHeight! - 10 + 'px'
                  : controlProp.Height! - controlProp.TabFixedHeight! - 5 + 'px'
                : controlProp.TabFixedHeight! === 0
                  ? controlProp.Font!.FontSize! === 72
                    ? controlProp.Height! - controlProp.Font!.FontSize! - 18 + 'px'
                    : controlProp.Height! - controlProp.Font!.FontSize! - 16 + 'px'
                  : controlProp.TabOrientation === 1
                    ? `${controlProp.Height! - 21}px`
                    : `${controlProp.Height! - 35}px`
            : `${controlProp.Height! - 1}px`
          : `${controlProp.Height! - 1}px`,
      width:
        controlProp.Style !== 2
          ? controlProp.TabOrientation === 0 || controlProp.TabOrientation === 1
            ? `${controlProp.Width! - 3}px`
            : controlProp.TabFixedWidth! > 0
              ? controlProp.Width! - controlProp.TabFixedWidth! - 10 + 'px'
              : controlProp.TabFixedWidth! === 0
                ? controlProp.TabOrientation === 2 ||
              controlProp.TabOrientation === 3
                  ? `${controlProp.Width! - this.tempWidth - 13}px`
                  : controlProp.Width! - controlProp.Font!.FontSize! - 20 + 'px'
                : 'calc(100% - 44px)'
          : `${controlProp.Width! - 3}px`,
      left:
        controlProp.Style !== 2
          ? controlProp.TabOrientation === 2
            ? controlProp.TabFixedWidth! > 0
              ? controlProp.TabFixedWidth! + 12 + 'px'
              : controlProp.TabFixedWidth! === 0
                ? controlProp.TabOrientation === 2 ||
                controlProp.TabOrientation === 3
                  ? `${this.tempWidth + 12}px`
                  : controlProp.Font!.FontSize! + 20 + 'px'
                : '40px'
            : '0px'
          : '0px',
      padding: '0px',
      overflow: 'hidden',
      overflowX: this.getScrollBarPage ? this.getScrollBarPage.overflowX! : '',
      overflowY: this.getScrollBarPage ? this.getScrollBarPage.overflowY! : '',
      backgroundColor: 'rgb(238,238,238)',
      backgroundImage:
        'radial-gradient(circle, rgb(0, 0, 0) 0.5px, rgba(0, 0, 0, 0) 0.2px) !important',
      backgroundSize: '9px 10px',
      boxShadow:
        controlProp.TabOrientation === 0 ? '1px 0px gray' : '1px 1px gray'
    }
  }

  /**
   * @description getPosition returns string value from
   * controlProperties.picturePositionProp
   * @function getPosition
   * @returns string value
   */
  protected get getPosition () {
    if (this.selectedPageData) {
      const picture = this.selectedPageData.properties.Picture!
      const pictureAlignment = this.selectedPageData.properties
        .PictureAlignment!
      const pictureSizeMode = this.selectedPageData.properties.PictureSizeMode!
      return controlProperties.getPositionProp(
        picture,
        pictureAlignment,
        pictureSizeMode
      )
    } else {
      return ''
    }
  }

  /**
   * @description getSizeMode returns string value from
   * controlProperties.pictureSizeModeProp
   * @function getSizeMode
   * @returns string value
   */
  protected get getSizeMode (): string {
    if (this.selectedPageData) {
      const index: number = this.selectedPageData.properties.PictureSizeMode!
      return controlProperties.getSizeModeProp(index)
    } else {
      return ''
    }
  }

  /**
   * @description getScrollBarPage returns propData based on keepScrollBar and scrollBar values
   * @function getScrollBarPage
   */
  get getScrollBarPage () {
    if (this.selectedPageData) {
      const keepScrollBar: number = this.selectedPageData.properties
        .KeepScrollBarsVisible!
      const scrollBar: number = this.selectedPageData.properties.ScrollBars!
      return controlProperties.setScrollBarProp(keepScrollBar, scrollBar)
    } else {
      return ''
    }
  }

  /**
   * @description returns the selected page
   * dynamically based on the selectedPageID
   * @function selectedPageData
   *
   */
  get selectedPageData () {
    if (this.selectedPageID) {
      return this.userformData[this.userFormId][this.selectedPageID]
    } else {
      return ''
    }
  }

  /**
   * @description watches changes in propControlData to set autoset when true
   * @function isScrollUsed
   * @param oldVal previous propControlData data
   * @param newVal  new/changed propControlData data
   */
  @Watch('properties.Width')
  isScrollUsed (newVal: number, oldVal: number) {
    if (this.properties.MultiRow) {
      const initialLength = this.controls.length!
      const len = (this.tempWidth + 12) * initialLength
      if (len - this.properties.Width! >= 0) {
        if (this.multiRowCount <= this.controls.length!) {
          const a = Math.ceil(len / this.properties.Width!)
          this.multiRowCount = a
          if (this.properties.Width! <= (this.tempWidth + 12) * 2) {
            this.multiRowCount = a + 2
          }
        } else if (newVal > oldVal) {
          this.multiRowCount = this.controls.length!
          const a = Math.ceil(len / this.properties.Width!)
          this.multiRowCount = a
        }
      } else {
        this.multiRowCount = 1
      }
    } else {
      this.scrollButtonVerify()
    }
  }

  @Watch('properties.TabOrientation')
  orientValidate () {
    this.scrollButtonVerify()
  }

  @Watch('properties.Height')
  heightValidate () {
    this.scrollButtonVerify()
  }

  @Watch('properties.TabFixedWidth')
  tabFixedWidthValidate () {
    this.scrollButtonVerify()
  }

  @Watch('properties.TabFixedWidth')
  tabFixedHeightValidate () {
    this.scrollButtonVerify()
  }

  /**
   * @description watches changes in FontSize of Font
   * @function checkFontValue
   * @param oldVal previous properties data
   * @param newVal  new/changed properties data
   */
  @Watch('properties.Font.FontSize')
  checkFontValue (newVal: number, oldVal: number) {
    this.calculateWidthHeight()
    this.scrollButtonVerify()
  }

  /**
   * @description watches changes in selectedPageData to set the caption
   * @function captionValue
   * @param oldVal previous selectedPageData data
   * @param newVal  new/changed selectedPageData data
   */
  @Watch('selectedPageData.properties.Caption')
  captionValue (newVal: string, oldVal: string) {
    this.calculateWidthHeight()
    this.scrollButtonVerify()
  }

  calculateWidthHeight () {
    const that = this
    if (this.controlTabsRef) {
      const divElement = this.controlTabsRef
      let tempWidth = 0
      let tempHeight = 0
      Vue.nextTick(function () {
        for (let i = 0; i < divElement.length; i++) {
          const ele = divElement[i].children[0].children[1]
            .children[0] as HTMLInputElement
          if (ele.offsetWidth > tempWidth) {
            tempWidth = ele.offsetWidth
          }
          if (ele.offsetHeight > tempHeight) {
            tempHeight = ele.offsetHeight
          }
        }
        that.tempWidth = tempWidth
        that.tempHeight = tempHeight
      })
    }
  }
  mounted () {
    this.scrollButtonVerify()
    this.scrollLeftTop(this.data)
    this.selectedPageID = this.controls[0]
    this.calculateWidthHeight()
    // this.focusPage()
  }
  dragSelectorControl (event: MouseEvent) {
    this.selectedControlArray = []
    if (
      this.selectedPageData &&
      this.controls.length > 0 &&
      this.controls.includes(this.selectedPageID)
    ) {
      this.selectedAreaStyle = this.containerRef.dragSelector.selectAreaStyle
      this.calSelectedAreaStyle(event, this.selectedPageData)
    }
  }

  multiPageMouseDown (e: MouseEvent) {
    EventBus.$emit('isEditMode', this.isEditMode)
    this.selectedItem(e)
    if (this.selMultipleCtrl === false && this.activateCtrl === false) {
      const selContainer = this.selectedControls[this.userFormId].container[0]
      const selected = this.selectedControls[this.userFormId].selected
      if (this.controls.length > 0 && selected.length === 1) {
        this.selectControl({
          userFormId: this.userFormId,
          select: {
            container: this.getContainerList(this.selectedPageID),
            selected: [this.selectedPageID]
          }
        })
      }
      if (selContainer === this.controlId) {
        if (this.selMultipleCtrl === false && this.activateCtrl === false) {
          this.selectControl({
            userFormId: this.userFormId,
            select: {
              container: this.getContainerList(this.selectedPageID),
              selected: [this.selectedPageID]
            }
          })
        }
      }
    }
  }
  showContextMenu (
    e: MouseEvent,
    parentID: string,
    controlID: string,
    type: string,
    mode: boolean
  ) {
    e.preventDefault()
    const selected = this.selectedControls[this.userFormId].selected
    if (selected[0] === this.controlId && this.controls.length > 0) {
      this.changeSelect(this.controls[0])
    }
    this.multiPageContextMenu = false
    this.openMenu(e, parentID, controlID, type, mode)
    Vue.nextTick(() => this.containerRef.contextmenu.focus())
  }
  handleKeyDown (event: KeyboardEvent) {
    this.containerRef.refContextMenu.updateAction(event)
  }
  changeSelect (control: string) {
    this.selectControl({
      userFormId: this.userFormId,
      select: {
        container: this.getContainerList(control),
        selected: [control]
      }
    })
  }
  handleContextMenu (e: MouseEvent) {
    e.preventDefault()
    const userData = this.userformData[this.userFormId]
    let selectedPage = -1
    this.top = `${e.offsetY}px`
    this.left = `${e.offsetX}px`
    if (this.isEditMode) {
      for (const val of this.contextMenuValue) {
        if (val.id === 'ID_DELETEPAGE' || val.id === 'ID_RENAME') {
          val.disabled = this.controls.length === 0
        } else if (val.id === 'ID_MOVE') {
          val.disabled = this.controls.length <= 1
        }
      }
      this.multiPageContextMenu = true
      Vue.nextTick(() => this.multipage.focus())
      this.viewMenu = false
    }
  }
  deleteMultiPage (event: KeyboardEvent) {
    if (this.controlId === this.selectedControls[this.userFormId].selected[0]) {
      this.deleteItem(event)
    } else {
      this.handleKeyDown(event)
    }
  }
  updateScrollingLeftTop (e: MouseEvent) {
    const refName = this.contentRef
    this.updateControl({
      userFormId: this.userFormId,
      controlId: this.selectedPageID,
      propertyName: 'ScrollLeft',
      value: refName.scrollLeft
    })
    this.updateControl({
      userFormId: this.userFormId,
      controlId: this.selectedPageID,
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

  created () {
    EventBus.$on('updateMultiPageValue', this.updateValue)
  }
  destroyed () {
    EventBus.$off('updateMultiPageValue')
  }
  updateValue () {
    {
      const userData = this.userformData[this.userFormId]
      let selectedPage = -1
      if (this.controls.length > 0) {
        selectedPage = this.controls.findIndex(
          (val) => this.properties.Value === userData[val].properties.Index
        )
      }
      if (this.data.controls.length > 0 && selectedPage !== -1) {
        this.selectedPageID = this.controls[selectedPage]
        this.changeSelect(this.controls[selectedPage])
      } else {
        this.changeSelect(this.controlId)
      }
    }
  }
  deleteMultiPageControl (event: KeyboardEvent) {
    const userData = this.userformData[this.userFormId]
    if (
      this.selectedPageID &&
      this.userformData[this.userFormId][this.selectedPageID].controls.length <=
        0
    ) {
      event.stopPropagation()
      this.deleteItem(event)
    }
  }
}
</script>

<style scoped>
.outer-page {
  background-color: rgb(238, 238, 238);
  overflow-y: hidden;
  overflow-x: hidden;
  box-sizing: border-box;
  width: 0px;
  height: 0px;
  left: 0px;
  top: 0px;
  cursor: default;
  position: sticky;
}
.pages {
  /* display: grid; */
  margin: 0;
  /* width: calc(100%);
  height: calc(100%); */
  white-space: nowrap;
  overflow-x: hidden;
  overflow-y: hidden;
}
.left-button {
  position: relative;
  outline: none;
  background-image: url("../../../../assets/left-arrow-img.png");
  background-size: 30%;
  background-position: center;
  background-repeat: no-repeat;
  border: 2px solid white;
  border-right-color: gray;
  border-bottom-color: gray;
  top: 3px;
  right: 15px;
  width: 22px;
  height: 18px;
  padding: 0;
  margin: 0;
  overflow: hidden;
  z-index: 5;
}
.right-button {
  position: relative;
  outline: none;
  background-image: url("../../../../assets/right-arrow-img.png");
  background-size: 30%;
  background-position: center;
  background-repeat: no-repeat;
  border: 2px solid white;
  border-right-color: gray;
  border-bottom-color: gray;
  top: 3px;
  right: 15px;
  width: 22px;
  height: 18px;
  padding: 0;
  margin: 0;
  overflow: hidden;
  z-index: 5;
}
.move {
  display: grid;
}
.page {
  vertical-align: top;
  z-index: 1;
  overflow: hidden;
}
.scroll-page {
  z-index: 2;
}
.page label {
  border: 0.1px solid white;
  background-color: rgb(238, 238, 238);
  /* display: inline-block; */
  padding: 5px 5px 5px 5px;
  margin: 0;
  cursor: pointer;
  position: relative;
  top: 0px;
}
.page [type="radio"] {
  display: none;
}
::-webkit-scrollbar.move {
  display: none;
  width: 0;
  height: 1em;
  background-color: rgb(238, 238, 238);
}
::-webkit-scrollbar.content {
  width: 0;
  height: 1em;
  background-color: rgb(238, 238, 238);
}

::-webkit-scrollbar-button {
  background: rgb(238, 238, 238);
  height: 20px;
  border: 1px solid lightgray;
  border-right-color: gray;
  border-bottom-color: gray;
}

/* Up */
::-webkit-scrollbar-button:single-button:horizontal:decrement {
  background-image: url("../../../../assets/triangle-up-img.png");
  transform: rotate(90deg);
  background-size: 10px;
  background-position: center;
  background-repeat: no-repeat;
  /* border-color: lightgrey; */
}

/* Down */
::-webkit-scrollbar-button:single-button:horizontal:increment {
  background-image: url("../../../../assets/triangle-down-img.png");
  background-size: 10px;
  background-position: center;
  background-repeat: no-repeat;
  /* border-color: lightgrey; */
}

::-webkit-scrollbar-track-piece {
  width: 0px;
}

::-webkit-scrollbar-thumb {
  background-color: darkgrey;
  outline: 1px solid slategrey;
  height: 5px;
}

.page .content {
  position: absolute;
  white-space: normal;
  top: 23px;
  left: 0px;
  /* background: rgb(238, 238, 238); */
  background-color: white;
  background-size: 9px 10px;
  background-image: radial-gradient(
    circle,
    rgb(0, 0, 0) 0.5px,
    rgba(0, 0, 0, 0) 0.2px
  );
  height: 100px;
  right: 0;
  bottom: 0;
  padding: 20px;
  padding-right: 10px;
  width: calc(100% - 35px);
  height: calc(100% - 89px);
  border: 0.1px solid white;
  box-shadow: 2px 1px gray;
}

.page [type="radio"]:checked ~ label ~ .content {
  z-index: 1;
}
.content {
  overflow: auto;
}
#right-click-menu {
  background: #fafafa;
  border: 1px solid #bdbdbd;
  box-shadow: 0 2px 2px 0 rgba(0, 0, 0, 0.14), 0 3px 1px -2px rgba(0, 0, 0, 0.2),
    0 1px 5px 0 rgba(0, 0, 0, 0.12);
  display: block;
  list-style: none;
  margin: 0;
  padding: 0;
  position: absolute;
  width: 100px;
  z-index: 999999;
}

#right-click-menu li {
  border-bottom: 1px solid #e0e0e0;
  margin: 0;
  padding: 5px 5px;
}

#right-click-menu li:last-child {
  border-bottom: none;
}

#right-click-menu li:hover {
  background: #1e88e5;
  color: #fafafa;
}

.spanClass {
  text-decoration: underline;
  text-underline-position: under;
}
:focus {
  outline: none;
}
</style>
