<template>
  <div
    class="listStyle"
    :style="listStyleObj"
    :title="properties.ControlTipText"
    @click.stop="listBoxClick"
    @mousedown="controlEditMode"
    @click.self="selectListBox"
    :tabindex="properties.TabIndex"
    @keydown="forMatchEntry"
    v-on="eventStoppers()"
    @keydown.esc="setContentEditable($event, false)"
  >
    <table class="table-style" :style="tableStyleObj" ref="listBoxTableRef">
      <thead v-if="properties.ColumnHeads === true" class="theadClass">
        <tr>
          <td
            :style="tdStyleObj"
            v-if="properties.ListStyle === 1"
            class="tdClass"
          ></td>
          <template v-for="(a, columnIndex) in extraDatas.ColumnHeadsValues">
            <td
              v-if="columnIndex < properties.ColumnCount"
              :key="columnIndex"
              :style="updateColHeads(columnIndex)"
            >
              {{ a }}
            </td>
          </template>
        </tr>
      </thead>
      <thead v-else></thead>
      <tbody class="table-body">
        <tr
          :tabindex="index"
          class="tr"
          ref="listStyleRef"
          v-for="(item, index) of extraDatas.RowSourceData"
          :key="index"
          @mouseenter.stop="handleDrag"
          @keydown.stop="handleExtendArrowKeySelect"
          @keydown.esc="releaseEditMode"
          @blur.stop="clearMatchEntry"
          @mousedown="isRunMode || isEditMode ? handleMultiSelect($event) : setInitial($event)"
        >
          <td
            :style="tdStyleObj"
            class="tdClassIn"
            v-if="properties.ListStyle === 1 && properties.ColumnCount > 0"
          >
            <input
              :name="properties.MultiSelect === 2 ? 'checkbox' : 'radio'"
              :type="
                properties.MultiSelect === 1 || properties.MultiSelect === 2
                  ? 'checkbox'
                  : 'radio'
              "
              class="inputClass"
            />
          </td>
          <td
            class="column-item"
            v-for="(i, index) in item"
            :key="index"
            :style="updateColumnWidths(index)"
          >
            <template v-if="index < properties.ColumnCount">{{ i }}</template>
          </td>
        </tr>
      </tbody>
    </table>
  </div>
</template>

<script lang="ts">
import {
  Component,
  Prop,
  Model,
  Emit,
  Mixins,
  Watch,
  Ref
} from 'vue-property-decorator'
import FdControlVue from '@/api/abstract/FormDesigner/FdControlVue'
import { EventBus } from '@/FormDesigner/event-bus'

@Component({
  name: 'FDListBox'
})
export default class FDListBox extends Mixins(FdControlVue) {
  @Ref('listStyleRef') listStyleRef : HTMLTableRowElement[]
  @Ref('listBoxTableRef') listBoxTableRef!: HTMLTableElement
  @Prop() isActivated: boolean

  $el: HTMLDivElement

  selectListBox () {
    this.listBoxTableRef.children[1].children[0].dispatchEvent(new MouseEvent('mousedown'))
  }
  clearMatchEntry () {
    this.updateDataModelExtraData({ propertyName: 'MatchData', value: '' })
  }

  /**
   * @description style object is passed to :style attribute in div tag
   * dynamically changing the styles of the component based on properties
   * @function listStyleObj
   */
  protected get listStyleObj (): Partial<CSSStyleDeclaration> {
    const controlProp = this.properties
    let display = ''
    if (this.isRunMode) {
      display = controlProp.Visible ? 'inline-block' : 'none'
    } else {
      display = 'inline-block'
    }
    return {
      backgroundColor: controlProp.BackColor,
      borderColor: controlProp.BorderColor,
      border: this.properties.BorderStyle
        ? `1px solid ${this.properties.BorderColor}`
        : 'none',
      cursor:
        controlProp.MousePointer !== 0 || controlProp.MouseIcon !== ''
          ? this.getMouseCursorData
          : 'default',
      borderLeft: controlProp.SpecialEffect === 2 ? '2px solid gray' : controlProp.SpecialEffect === 3 ? '1.5px solid gray' : controlProp.SpecialEffect === 4 ? '0.5px solid gray' : '',
      borderRight: controlProp.SpecialEffect === 1 ? '2px solid gray' : controlProp.SpecialEffect === 4 ? '1.5px solid gray' : controlProp.SpecialEffect === 3 ? '0.5px solid gray' : '',
      borderTop: controlProp.SpecialEffect === 2 ? '2px solid gray' : controlProp.SpecialEffect === 3 ? '1.5px solid gray' : controlProp.SpecialEffect === 4 ? '0.5px solid gray' : '',
      borderBottom: controlProp.SpecialEffect === 1 ? '2px solid gray' : controlProp.SpecialEffect === 4 ? '1.5px solid gray' : controlProp.SpecialEffect === 3 ? '0.5px solid gray' : '',
      left: `${controlProp.Left}px`,
      width: `${controlProp.Width}px`,
      height: `${controlProp.Height}px`,
      top: `${controlProp.Top}px`,
      display: display
    }
  }

  /**
   * @description style object is passed to :style attribute in table tag
   * dynamically changing the styles of the component based on properties
   * @function tableStyleObj
   */
  protected get tableStyleObj (): Partial<CSSStyleDeclaration> {
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
    return {
      borderCollapse: 'collapse',
      tableLayout: 'fixed',
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
            'px'
    }
  }

  /**
   * @description style object is passed to :style attribute in td tag
   * dynamically changing the styles of the component based on properties
   * @function tdStyleObj
   */
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

  /**
   * @description watches changes in properties for Value
   * @function ValueData
   * @param oldVal previous properties data
   * @param newVal  new/changed properties data
   */
  @Watch('properties.Value', { deep: true })
  ValueData (newVal: string, oldVal: string) {
    if (
      this.properties.BoundColumn! > 0 &&
      this.properties.BoundColumn! < this.extraDatas.RowSourceData!.length
    ) {
      let tempData = [...this.extraDatas.RowSourceData!]
      let tempBoundColumn = this.properties.BoundColumn! - 1
      for (let i = 0; i < this.extraDatas.RowSourceData!.length; i++) {
        if (newVal !== '' && tempData[i][tempBoundColumn] !== newVal && this.listStyleRef[i].style.backgroundColor === 'rgb(59, 122, 231)') {
          this.listStyleRef[i].style.backgroundColor = ''
        }
        if (tempData[i][tempBoundColumn] === newVal) {
          this.clearOptionBGColorAndChecked(this.tempListBoxComboBoxEvent)
          this.listStyleRef[i].style.backgroundColor = 'rgb(59, 122, 231)'
          this.updateDataModel({ propertyName: 'Text', value: newVal })
        }
        if (newVal !== '' && tempData[i][tempBoundColumn] !== newVal && this.listStyleRef[i].style.backgroundColor === 'rgb(59, 122, 231)') {
          this.listStyleRef[i].style.backgroundColor = ''
        }
      }
      if (tempData![0][this.properties.BoundColumn! - 1] === newVal) {
        this.updateDataModel({ propertyName: 'Value', value: newVal })
      } else {
        this.updateDataModel({ propertyName: 'Value', value: '' })
      }
    } else {
      return undefined
    }
  }

  /**
   * @description mounted initializes the values which are required for the component
   */
  mounted () {
    this.$el.focus()
    var event = new MouseEvent('mousedown.stop')
    this.setInitial(event)
    const initialRowSourceData = this.extraDatas.RowSourceData!
    this.updateDataModel({ propertyName: 'ControlSource', value: '' })
    if (initialRowSourceData && initialRowSourceData.length === 0) {
      this.updateDataModel({ propertyName: 'TopIndex', value: -1 })
    } else {
      this.updateDataModel({ propertyName: 'TopIndex', value: 0 })
    }
  }

  /**
   * @description watches changes in properties for MultiSelect
   * @function multiSelectCheck
   * @param oldVal previous properties data
   * @param newVal  new/changed properties data
   */
  @Watch('properties.MultiSelect', { deep: true })
  multiSelectCheck (newVal: number, oldVal: number) {
    if (this.tempListBoxComboBoxEvent) {
      this.clearOptionBGColorAndChecked(this.tempListBoxComboBoxEvent)
    }
  }

  /**
   * @description watches changes in properties for ListStyle
   * @function listCheck
   * @param oldVal previous properties data
   * @param newVal  new/changed properties data
   */
  @Watch('properties.ListStyle', { deep: true })
  listCheck (newVal: number, oldVal: number) {
    this.clearOptionBGColorAndChecked(this.tempListBoxComboBoxEvent)
  }

  @Watch('isEditMode', { deep: true })
  isEditCheck (newVal: boolean, oldVal: boolean) {
    if (this.isEditMode) {
      this.$el.click()
    }
  }

  forMatchEntry (event: KeyboardEvent) {
    if (event.key === 'Enter' && event.keyCode === 13) {
      this.setContentEditable(event, true)
    }
    if (this.isEditMode) {
      this.listStyleRef[0].click()
      this.handleExtendArrowKeySelect(event)
    }
    if (event.key === 'Escape' && event.keyCode === 27) {
      this.releaseEditMode(event)
    }
    if (event.key === 'Delete') {
      this.deleteItem(event)
    }
  }

  @Watch('properties.ControlSource')
  controlSourceUpdate () {
    if (this.properties.ControlSource !== '') {
      for (let i = 0; i < this.extraDatas.RowSourceData!.length; i++) {
        this.listStyleRef[i].style.backgroundColor = ''
        if (this.extraDatas.RowSourceData![i][this.properties.BoundColumn! - 1] === this.extraDatas.ControlSourceValue!) {
          const listRow = this.listStyleRef[i]
          listRow.style.backgroundColor = 'rgb(59, 122, 231)'
        }
      }
    }
  }
  releaseEditMode (event: KeyboardEvent) {
    this.$el.focus()
    this.setContentEditable(event, false)
  }
  eventStoppers () {
    const eventStop = (event: Event) => event.stopPropagation()
    return this.isEditMode === false ? null : {
      keydown: eventStop
    }
  }
  listBoxClick (e: MouseEvent) {
    this.selectedItem(e)
    if (!this.isActivated) {
      EventBus.$emit('focusUserForm')
    }
  }
}
</script>

<style scoped>
.listStyle {
  width: 200px;
  height: 200px;
  background-color: lightgray;
  border: 1px solid gray;
  overflow: auto;
  box-sizing: border-box;
  /* box-shadow: -1px -1px lightgray; */
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
  /* padding: 3px; */
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
  /* margin: 4px;  */
  margin-left: 7px;
  margin-top: 0px;
  font-family: Arial, Helvetica, sans-serif;
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
.table-style {
  width: 100%;
}
.theadClass {
  border-bottom: 1px solid;
  white-space: nowrap;
}
.tdClass {
  width: 15px;
  border-right: 1px solid;
}
.tdClassIn {
  width: 20px;
}
.inputClass {
  margin: 0;
}
</style>
