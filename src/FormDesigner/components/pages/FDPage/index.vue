<template>
  <div id="app">
    <div class="container">
      <div class="header">
        <Header :userFormId="userFormId" />
      </div>
      <div class="mainbody">
        <Split class="outersplit">
          <SplitArea :size="20" class="splitleft">
            <Split :direction="vertical">
              <SplitArea class="innersplit">
                <div v-if="hideShowTree === false">
                  <div class="sideheader">
                    <span class="sideheader1">
                      <span>Project - VBAProject</span>
                      <div id="mdiv" v-on:click="hideShowTreeBrowser">
                        <div class="mdiv">
                          <div class="md"></div>
                        </div>
                      </div>
                    </span>
                  </div>

                  <div>
                    <ViewCodeLogo class="icon-style" />
                    <ViewObject class="icon-style" />
                    <CloseFolderLogo class="icon-style" />
                  </div>
                  <hr />
                  <div>
                    <div
                      v-for="(userform, userformId) in userformData"
                      :key="userformId"
                      @click="setUserForm(userformId)"
                    >
                      <p>{{ userformId }}</p>
                    </div>
                  </div>
                </div>
              </SplitArea>

              <SplitArea class="innersplit">
                <PropertiesList
                :userFormId="userFormId"/>
                <!-- <label>Prop Name</label>
                <input type="text" v-model="testPropName" />
                <br />
                <label>Prop Value</label>
                <input type="text" v-model="testPropValue" />
                <br />
                <button @click="clickUpdate">Update selected Control</button>
                <br /><br />
                <button @click="clickDelete">Delete selected Control</button> -->
              </SplitArea>
            </Split>
          </SplitArea>

          <SplitArea :size="80" class="splitAreaClass">
            <div class="splitAreaDiv">
              <ResizeUserForm
                :userFormId="userFormId"
                :controlId="userFormId"
                :containerId="userFormId"
              />
            </div>
          </SplitArea>
        </Split>
      </div>
      <div class="fixedSide">
        <ToolBox />
      </div>
      <div>
        <UserformTabOrder />
        <TabStripTabOrder />
        <RenameMultiPageDialog />
        <ErrorPopup />
      </div>
    </div>
  </div>
</template>

<script lang="ts">
import { Component, Vue } from 'vue-property-decorator'
import ToolBox from '@/FormDesigner/components/organisms/FDToolBox/index.vue'
import ResizeUserForm from '@/FormDesigner/components/organisms/FDResizeUserForm/index.vue'
import PropertiesList from '@/FormDesigner/components/organisms/FDPropertyList/index.vue'
import TreeBrowser from '@/FormDesigner/components/organisms/FDTreeBrowser/index.vue'
import Header from '@/FormDesigner/components/organisms/FDHeader/index.vue'
import UserformTabOrder from '@/FormDesigner/components/molecules/FDUserformTabOrder/index.vue'
import TabStripTabOrder from '@/FormDesigner/components/organisms/FDTabStripTabOrderModal/index.vue'
import RenameMultiPageDialog from '@/FormDesigner/components/organisms/FDRenameMultiPageDialog/index.vue'
import ErrorPopup from '@/FormDesigner/components/atoms/FDErrorPopup/index.vue'

import ViewCodeLogo from '../../../../assets/view-code.svg'
import ViewObject from '../../../../assets/view-object.svg'
import CloseFolderLogo from '../../../../assets/close-folder.svg'
import { State, Action } from 'vuex-class'
import { Payload } from 'vuex'
import {
  IupdateControl,
  IdeleteControl,
  IupdateControlExtraData,
  IselectControl
} from '../../../../storeModules/fd/actions'
import { EventBus } from '@/FormDesigner/event-bus'

@Component({
  name: 'FDPage',
  components: {
    ToolBox,
    ResizeUserForm,
    TreeBrowser,
    PropertiesList,
    Header,
    ViewCodeLogo,
    ViewObject,
    CloseFolderLogo,
    UserformTabOrder,
    TabStripTabOrder,
    RenameMultiPageDialog,
    ErrorPopup
  }
})
export default class FDPage extends Vue {
  @State((state) => state.fd.userformData) userformData!: userformData;
  vertical: string = 'vertical';
  display: string = '';
  hideShowTree: boolean = false;
  hideShowTreeBrowser () {
    this.hideShowTree = true
  }
  userFormId: string = 'ID_USERFORM1';
  @State((state: rootState) => state.fd.selectedControls)
  selectedControls!: fdState['selectedControls'];
  @Action('fd/updateControl') updateControl!: (payload: IupdateControl) => void;
  @Action('fd/updateControlExtraData') updateControlExtraData!: (payload: IupdateControlExtraData) => void;
  @Action('fd/deleteControl') deleteControl!: (payload: IdeleteControl) => void;
  @Action('fd/selectControl') selectControl!: (payload: IselectControl) => void;
  testPropName: keyof controlProperties = 'Width';
  testPropValue = '100';

  clickUpdate () {
    const selected = this.selectedControls[this.userFormId].selected
    for (let i = 0; i < selected.length; i++) {
      this.updateControl({
        userFormId: this.userFormId,
        controlId: selected[i],
        propertyName: this.testPropName,
        value: this.testPropValue
      })
    }
  }
  clickDelete () {
    const selected = this.selectedControls[this.userFormId].selected
    for (let i = 0; i < selected.length; i++) {
      this.deleteControl({
        userFormId: this.userFormId,
        parentId: this.selectedControls[this.userFormId].container[0],
        targetId: selected[i]
      })
    }
  }
  setUserForm (userformId: string) {
    this.userFormId = userformId
    this.selectControl({
      userFormId: this.userFormId,
      select: {
        container: [this.userFormId],
        selected: [this.userFormId]
      }
    })
  }
  mounted () {
    this.setUserForm(this.userFormId)
  }
}
</script>

<style scoped>
#app {
  font-family: Tahoma;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  color: #2c3e50;
  font-size: 14px;
}

.splitleft {
  width: auto;
  z-index: 1;
  top: 5;
  left: 5;
  background-color: white;
  overflow-x: hidden;
  border: 2px solid grey;
}
.outersplit {
  height: 83%;
  width: 85%;
  position: fixed;
}
.innersplit {
  border: 1px solid grey;
  position: relative;
  overflow: hidden;
}
.sideheader1 {
  display: grid;
  grid-template-columns: 1fr 25px;
  /* width: 250px; */
  background-color: rgb(142, 191, 231);
  margin-bottom: 8px;
  text-overflow: ellipsis;
  overflow: hidden;
  white-space: nowrap;
}
.sideheader {
  height: 22px;
  text-align: left;
  background-color: rgb(142, 191, 231);
}

.right {
  right: 0;
  background-color: #80888e;
  height: 100%;
  width: 75%;
  position: absolute;
  z-index: 1;
  overflow-x: hidden;
  /* padding-top: 20px; */
}
.container {
  width: 100%;
  height: 600px;
  max-height: 500px;
}
.header {
  /* height: 40px; */
  height: 10%;
  border-bottom: 1px solid #eee;
  background-color: #ffffff;
}
#mdiv {
  float: right;
  margin: auto;
  position: relative;
  top: 0px;
  left: 0px;
  width: 20px;
  height: 16px;
  background-color: lightgray;
  border: outset;
}

.mdiv {
  height: 17px;
  width: 2px;
  margin-left: 60px;
  background-color: black;
  transform: rotate(45deg);
  z-index: 1;
  position: absolute;
  right: 8px;
}

.md {
  height: 17px;
  width: 2px;
  background-color: black;
  transform: rotate(90deg);
  z-index: 2;
  position: absolute;
  right: 0px;
}
.commandbutton-element {
  border: 1px solid gray;
  border-radius: 3px;
  background-color: white;
  min-width: 50px;
  max-width: 112px;
  width: fit-content;
  height: fit-content;
  min-height: 20px;
  padding: 0px 5px 0px 5px;
  overflow: hidden;
}
.dialog-action-button {
  background-color: #f2eded;
  width: 79px;
  height: 35px;
  padding-left: 5px;
  padding-right: 5px;
  padding-top: 8px;
  padding-bottom: 8px;
  border: 1px solid gray;
  border-radius: 3px;
  overflow: hidden;
}
:focus {
  outline: none;
}
.mainbody {
  margin-top: 37px;
}
.fixedSide {
  /*  position: relative; */
  width: 13%;
  float: right;
}
.icon-style {
  margin-top: 7px;
  margin-left: 5px;
}
.splitAreaClass {
  background: gray;
  overflow: auto;
}
.splitAreaDiv {
  position: relative;
  padding: 5px;
}
</style>
