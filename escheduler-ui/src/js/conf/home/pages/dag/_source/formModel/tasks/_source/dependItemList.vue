<template>
  <div class="dep-list-model">
    <div v-for="(el,$index) in dependItemList" class="list" @click="itemIndex = $index">
      <x-select filterable :style="{width:isInstance ? '140px' : '162px'}" :disabled="isDetails" v-model="el.definitionId" @on-change="_onChangeDefinitionId">
        <x-option v-for="item in definitionList" :key="item.value" :value="item.value" :label="item.label">
        </x-option>
      </x-select>
      <x-select filterable :style="{width:isInstance ? '144px' : '156px'}" :disabled="isDetails" v-model="el.depTasks">
        <x-option v-for="item in el.depTasksList || []" :key="item" :value="item" :label="item">
        </x-option>
      </x-select>
      <x-select style="width: 80px;" v-model="el.cycle" :disabled="isDetails" @on-change="_onChangeCycle">
        <x-option v-for="item in cycleList" :key="item.value" :value="item.value" :label="item.label">
        </x-option>
      </x-select>
      <x-select style="width: 116px;" v-model="el.dateValue" :disabled="isDetails">
        <x-option v-for="item in el.dateValueList || []" :key="item.value" :value="item.value" :label="item.label">
        </x-option>
      </x-select>
      <template v-if="isInstance">
        <span class="instance-state">
          <i class="iconfont" :class="'icon-' + el.state" v-if="el.state === 'SUCCESS'" data-toggle="tooltip" data-container="body" :title="$t('成功')">&#xe607;</i>
          <i class="iconfont" :class="'icon-' + el.state" v-if="el.state === 'WAITING'" data-toggle="tooltip" data-container="body" :title="$t('等待')">&#xe62a;</i>
          <i class="iconfont" :class="'icon-' + el.state" v-if="el.state === 'FAILED'" data-toggle="tooltip" data-container="body" :title="$t('失败')">&#xe626;</i>
        </span>
      </template>
      <span class="operation">
        <a href="javascript:" class="delete" @click="!isDetails && _remove($index)">
          <i class="iconfont" :class="_isDetails" data-toggle="tooltip" data-container="body" :title="$t('删除')" >&#xe611;</i>
        </a>
        <a href="javascript:" class="add" @click="!isDetails && _add()" v-if="$index === (dependItemList.length - 1)">
          <i class="iconfont" :class="_isDetails" data-toggle="tooltip" data-container="body" :title="$t('添加')">&#xe636;</i>
        </a>
      </span>
    </div>
  </div>
</template>
<script>
  import _ from 'lodash'
  import { cycleList, dateValueList } from './commcon'
  import disabledState from '@/module/mixin/disabledState'
  export default {
    name: 'dep-list',
    data () {
      return {
        list: [],
        definitionList: [],
        cycleList: cycleList,
        isInstance: false,
        itemIndex: null
      }
    },
    mixins: [disabledState],
    props: {
      dependItemList: Array,
      index: Number
    },
    model: {
      prop: 'dependItemList',
      event: 'dependItemListEvent'
    },
    methods: {
      /**
       * add task
       */
      _add () {
        // btn loading
        this.isLoading = true
        // dependItemList index
        let is = (value) => _.some(this.dependItemList, { definitionId: value })
        let noArr = _.filter(this.definitionList, v => !is(v.value))
        let value = noArr[0] && noArr[0].value || null
        let val = value || this.definitionList[0].value
        // add task list
        this._getDependItemList(val).then(depTasksList => {
          this.$nextTick(() => {
            this.$emit('dependItemListEvent', _.concat(this.dependItemList, this._rtNewParams(val, depTasksList)))
          })
        })
        // remove tooltip
        this._removeTip()
      },
      /**
       * remove task
       */
      _remove (i) {
        this.dependItemList.splice(i, 1)
        this._removeTip()

        if (!this.dependItemList.length) {
          this.$emit('on-delete-all', {
            index: this.index
          })
        }
      },
      /**
       * get processlist
       */
      _getProcessList () {
        return new Promise((resolve, reject) => {
          this.definitionList = _.map(_.cloneDeep(this.store.state.dag.processListS), v => {
            return {
              value: v.id,
              label: v.name
            }
          })
          resolve()
        })
      },
      /**
       * get dependItemList
       */
      _getDependItemList (ids, is = true) {
        return new Promise((resolve, reject) => {
          if (is) {
            this.store.dispatch('dag/getProcessTasksList', { processDefinitionId: ids }).then(res => {
              resolve(['ALL'].concat(_.map(res, v => v.name)))
            })
          } else {
            this.store.dispatch('dag/getTaskListDefIdAll', { processDefinitionIdList: ids }).then(res => {
              resolve(res)
            })
          }
        })
      },
      /**
       * change process get dependItemList
       */
      _onChangeDefinitionId ({ value }) {
        // get depItem list data
        this._getDependItemList(value).then(depTasksList => {
          let item = this.dependItemList[this.itemIndex]
          // init set depTasks All
          item.depTasks = 'ALL'
          // set dependItemList item data
          this.$set(this.dependItemList, this.itemIndex, this._rtOldParams(value, depTasksList, item))
        })
      },
      _onChangeCycle ({ value }) {
        let list = _.cloneDeep(dateValueList[value])
        this.$set(this.dependItemList[this.itemIndex], 'dateValue', list[0].value)
        this.$set(this.dependItemList[this.itemIndex], 'dateValueList', list)
      },
      _rtNewParams (value, depTasksList) {
        return {
          definitionId: value,
          depTasks: 'ALL',
          depTasksList: depTasksList,
          cycle: 'day',
          dateValue: 'last1Days',
          dateValueList: _.cloneDeep(dateValueList['day']),
          state: ''
        }
      },
      _rtOldParams (value, depTasksList, item) {
        return {
          definitionId: value,
          depTasks: item.depTasks || 'ALL',
          depTasksList: depTasksList,
          cycle: item.cycle,
          dateValue: item.dateValue,
          dateValueList: _.cloneDeep(dateValueList[item.cycle]),
          state: item.state
        }
      },
      /**
       * remove tip
       */
      _removeTip () {
        $('body').find('.tooltip.fade.top.in').remove()
      }
    },
    watch: {
    },
    beforeCreate () {
    },
    created () {
      // is type projects-instance-details
      this.isInstance = this.router.history.current.name === 'projects-instance-details'
      // get processlist
      this._getProcessList().then(() => {
        if (!this.dependItemList.length) {
          let value = this.definitionList[0].value
          this._getDependItemList(value).then(depTasksList => {
            this.$emit('dependItemListEvent', _.concat(this.dependItemList, this._rtNewParams(value, depTasksList)))
          })
        } else {
          // get definitionId ids
          let ids = _.map(this.dependItemList, v => v.definitionId).join(',')
          // get item list
          this._getDependItemList(ids, false).then(res => {
            _.map(this.dependItemList, (v, i) => {
              this.$set(this.dependItemList, i, this._rtOldParams(v.definitionId, ['ALL'].concat(_.map(res[v.definitionId] || [], v => v.name)), v))
            })
          })
        }
      })
    },
    mounted () {
    },
    components: {}
  }
</script>

<style lang="scss" rel="stylesheet/scss">
  .dep-list-model {
    position: relative;
    min-height: 32px;
    .list {
      margin-bottom: 6px;
      .operation {
        padding-left: 4px;
        a {
          i {
            font-size: 18px;
            vertical-align: middle;
          }
        }
        .delete {
          color: #ff0000;
        }
        .add {
          color: #0097e0;
        }
      }
    }
    .instance-state {
      display: inline-block;
      width: 24px;
      .iconfont {
        font-size: 20px;
        vertical-align: middle;
        cursor: pointer;
        margin-left: 6px;
        &.icon-SUCCESS {
          color: #33cc00;
        }
        &.icon-WAITING {
          color: #888888;
        }
        &.icon-FAILED {
          color: #F31322;
        }
      }
    }
  }
</style>
