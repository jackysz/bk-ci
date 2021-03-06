<template>
    <section class="bk-form pipeline-setting base" v-if="!isLoading">
        <div class="setting-container">
            <form-field :required="true" :label="$t('name')" :is-error="errors.has(&quot;name&quot;)" :error-msg="errors.first(&quot;name&quot;)">
                <input class="bk-form-input" :placeholder="$t('settings.namePlaceholder')" v-model="pipelineSetting.pipelineName" name="name" v-validate.initial="&quot;required|max:40&quot;" />
            </form-field>

            <form-field :required="false" :label="$t('settings.group')" v-if="tagGroupList.length">
                <div class="form-group form-group-inline">
                    <div :class="grouInlineCol"
                        v-for="(filter, index) in tagGroupList"
                        :key="index">
                        <label class="group-title">{{filter.name}}</label>
                        <bk-select :value="labelValues[index]"
                            @selected="handleLabelSelect(index, arguments)"
                            @clear="handleLabelSelect(index, [[]])"
                            multiple
                        >
                            <bk-option v-for="(option, oindex) in filter.labels" :key="oindex" :id="option.id" :name="option.name">
                            </bk-option>
                        </bk-select>
                    </div>
                </div>
            </form-field>

            <form-field :label="$t('desc')" :is-error="errors.has(&quot;desc&quot;)" :error-msg="errors.first(&quot;desc&quot;)">
                <textarea name="desc" v-model="pipelineSetting.desc" :placeholder="$t('settings.descPlaceholder')" class="bk-form-textarea" v-validate.initial="&quot;max:100&quot;"></textarea>
            </form-field>

            <form-field :label="$t('settings.runLock')" class="opera-lock-radio">
                <bk-radio-group v-model="pipelineSetting.runLockType">
                    <bk-radio v-for="(entry, key) in runTypeList" :key="key" :value="entry.value" class="view-radio">{{ entry.label }}</bk-radio>
                </bk-radio-group>
            </form-field>
            <div class="bk-form-item opera-lock" v-if="pipelineSetting.runLockType === 'SINGLE'">
                <div class="bk-form-content">
                    <div class="opera-lock-item">
                        <label class="opera-lock-label">{{ $t('settings.largestNum') }}：</label>
                        <div class="bk-form-control control-prepend-group control-append-group">
                            <input type="text" name="maxQueueSize" :placeholder="$t('settings.itemPlaceholder')" class="bk-form-input" v-validate.initial="&quot;required|numeric|max_value:20|min_value:0&quot;" v-model.number="pipelineSetting.maxQueueSize">
                            <div class="group-box group-append">
                                <div class="group-text">{{ $t('settings.item') }}</div>
                            </div>
                            <p v-if="errors.has('maxQueueSize')" class="is-danger">{{errors.first("maxQueueSize")}}</p>
                        </div>
                    </div>
                    <div class="opera-lock-item">
                        <label class="opera-lock-label">{{ $t('settings.lagestTime') }}：</label>
                        <div class="bk-form-control control-prepend-group control-append-group">
                            <input type="text" name="waitQueueTimeMinute" :placeholder="$t('settings.itemPlaceholder')" class="bk-form-input" v-validate.initial="'required|numeric|max_value:1440|min_value:1'" v-model.number="pipelineSetting.waitQueueTimeMinute">
                            <div class="group-box group-append">
                                <div class="group-text">{{ $t('settings.minutes') }}</div>
                            </div>
                            <p v-if="errors.has('waitQueueTimeMinute')" class="is-danger">{{errors.first("waitQueueTimeMinute")}}</p>
                        </div>
                    </div>
                </div>
            </div>

            <div class="handle-btn" style="margin: 30px 0 0 146px">
                <bk-button @click="savePipelineSetting()" theme="primary" :disabled="isDisabled || noPermission">{{ $t('save') }}</bk-button>
                <bk-button @click="exit">{{ $t('cancel') }}</bk-button>
            </div>
        </div>
    </section>
</template>

<script>
    import { mapActions, mapState, mapGetters } from 'vuex'
    import FormField from '@/components/AtomPropertyPanel/FormField.vue'
    export default {
        components: {
            FormField
        },
        props: {
            isDisabled: {
                type: Boolean,
                default: false
            }
        },
        data () {
            return {
                noPermission: false,
                isEditing: false,
                isLoading: true,
                resetFlag: false,
                runTypeList: [
                    {
                        label: this.$t('settings.runningOption.multiple'),
                        value: 'MULTIPLE'
                    },
                    {
                        label: this.$t('settings.runningOption.lock'),
                        value: 'LOCK'
                    },
                    {
                        label: this.$t('settings.runningOption.single'),
                        value: 'SINGLE'
                    }
                ]
            }
        },
        computed: {
            ...mapState('soda', [
                'pipelineSetting'
            ]),
            ...mapGetters({
                'tagGroupList': 'pipelines/getTagGroupList'
            }),
            projectId () {
                return this.$route.params.projectId
            },
            pipelineId () {
                return this.$route.params.pipelineId
            },
            templateId () {
                return this.$route.params.templateId
            },
            routeName () {
                return this.$route.name
            },
            grouInlineCol () {
                const classObj = {}
                let key = 'group-inline '
                key += this.tagGroupList.length < 3 ? this.tagGroupList.length < 2 ? '' : 'column-2' : 'column-3'
                classObj[key] = true
                return classObj
            },
            labelValues () {
                const labels = this.pipelineSetting.labels
                return this.tagGroupList.map((tag) => {
                    const currentLables = tag.labels || []
                    const value = []
                    currentLables.forEach((label) => {
                        const index = labels.findIndex((item) => (item === label.id))
                        if (index > -1) value.push(label.id)
                    })
                    return value
                })
            }
        },
        watch: {
            pipelineSetting: {
                deep: true,
                handler: function (newVal, oldVal) {
                    // 无权限灰掉保存按钮
                    if (this.pipelineSetting.hasPermission !== undefined && this.pipelineSetting.hasPermission === false) {
                        this.noPermission = true
                    } else {
                        this.noPermission = false
                    }
                    this.isLoading = false
                    if (!this.isEditing && JSON.stringify(oldVal) !== '{}' && newVal !== null && !this.resetFlag) {
                        this.isEditing = true
                    }
                    this.resetFlag = false
                    this.isStateChange()
                }
            }
        },
        created () {
            this.requestTemplateSetting(this.$route.params)
            this.requestGrouptLists()
        },
        methods: {
            ...mapActions('soda', [
                'requestTemplateSetting'
            ]),
            handleLabelSelect (index, arg) {
                let labels = []
                this.labelValues.forEach((value, valueIndex) => {
                    if (valueIndex === index) labels = labels.concat(arg[0])
                    else labels = labels.concat(value)
                })
                this.pipelineSetting.labels = labels
            },
            isStateChange () {
                this.$emit('setState', {
                    isLoading: this.isLoading,
                    isEditing: this.isEditing
                })
            },
            exit () {
                this.$emit('cancel')
            },
            /** *
             * 获取标签及其分组
             */
            async requestGrouptLists () {
                const { $store } = this
                let res
                try {
                    res = await $store.dispatch('pipelines/requestGetGroupLists', {
                        projectId: this.projectId
                    })
                    $store.commit('pipelines/updateGroupLists', res)
                    this.dataList = this.tagGroupList
                } catch (err) {
                    this.$showTips({
                        message: err.message || err,
                        theme: 'error'
                    })
                }
            },
            async savePipelineSetting () {
                if (this.errors.any()) return
                this.isDisabled = true
                let result
                let resData
                try {
                    const { pipelineSetting } = this
                    Object.assign(pipelineSetting, { projectId: this.projectId })
                    resData = await this.$ajax.put(`/process/api/user/templates/projects/${this.projectId}/templates/${this.templateId}/settings`, pipelineSetting)

                    if (resData && resData.data) {
                        this.$showTips({
                            message: `${pipelineSetting.pipelineName}${this.$t('updateSuc')}`,
                            theme: 'success'
                        })
                        this.isEditing = false
                        this.isStateChange()
                        result = true
                    } else {
                        this.$showTips({
                            message: `${pipelineSetting.pipelineName}${this.$t('updateFail')}`,
                            theme: 'error'
                        })
                    }
                } catch (err) {
                    if (this.routeName !== 'templateSetting' && err.code === 403) { // 没有权限执行
                        this.$showAskPermissionDialog({
                            noPermissionList: [{
                                resource: `${this.$t('pipeline')}: ${this.pipelineSetting.pipelineName}`,
                                option: this.$t('edit')
                            }],
                            applyPermissionUrl: `${PERM_URL_PIRFIX}/backend/api/perm/apply/subsystem/?client_id=pipeline&project_code=${this.projectId}&service_code=pipeline&role_manager=pipeline:${this.pipelineId}`
                        })
                    } else {
                        this.$showTips({
                            message: err.message || err,
                            theme: 'error'
                        })
                    }
                    result = false
                }
                this.isDisabled = false
                return result
            }
        }
    }
</script>

<style lang="scss">
    @import './../../../scss/conf.scss';

    .pipeline-setting.base{
        position:absolute;
        width: 100%;
        padding:  20px;
        & .setting-container{
            width: 60%;
            min-width: 880px;
        }
         .bk-form-item{
             margin-bottom: 30px;
             & .bk-form-content .bk-form-radio{
                display: block;
             }
        }
        .form-group-inline {
            font-size: 0;
            &:after {
                display: block;
                content: '';
                height: 0;
                width: 0;
                clear: both;
            }
            .group-inline  {
                float: left;
                width: 100%;
                margin-right: 6px;
                .group-title {
                    display: inline-block;
                    max-width: 100%;
                    // margin-bottom: 5px;
                    line-height: 34px;
                    font-size: 14px;
                    overflow: hidden;
                    text-overflow: ellipsis;
                    white-space: nowrap;
                    color: #63656E;
                }
                &.column-2 {
                    width: calc((100% - 6px) / 2);
                }
                &.column-3 {
                    width: calc((100% - 12px) / 3);
                }
                &:last-child {
                    margin-right: 0;
                }
            }
        }
        .form-group-link {
            width: 100%;
        }
        .opera-lock-radio {
            margin-bottom: 0;
            .view-radio {
                margin-top: 8px;
            }
        }
        .opera-lock {
            margin-top: 0;
            .bk-form-content {
                margin-left: 180px;
                padding: 7px 0;
            }
            .opera-lock-item {
                display: inline-block;
                .opera-lock-label {
                    display: inline-block;
                    font-size: 14px;
                }
                + .opera-lock-item {
                    margin-left: 80px;
                }
            }
            .bk-form-control {
                position: relative;
                display: inline-table;
                width: auto;
                background-color: #f2f4f8;
                color: #63656e;
            }
            .is-danger {
                position: absolute;
                top: 100%;
                left: 0;
                font-size: 12px;
                color: $failColor;
            }
            .bk-form-input {
                width: 80px;
                border-radius: 0;
            }
            .group-box {
                vertical-align: middle;
                display: table-cell;
                position: relative;
                border: 1px solid #c4c6cc;
                border-left: none;
                border-radius: 0 2px 2px 0;
            }
            .group-text {
                color: #63656e;
                padding: 0 15px;
                white-space: nowrap;
                font-size: 14px;
            }
        }
    }
</style>
