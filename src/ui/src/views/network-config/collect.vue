<template>
    <div class="collect-wrapper">
        <div class="title">
            <bk-button type="primary"
            :disabled="!table.checked.length"
            :loading="$loading('collectDataCollection')"
            @click="executionDiscovery">
                {{$t('NetworkDiscovery["执行发现"]')}}
            </bk-button>
            <div class="input-box">
                <input type="text" class="cmdb-form-input" :placeholder="$t('NetworkDiscovery[\'搜索IP、云区域\']')" v-model.trim="filter.text" @keyup.enter="getTableData">
                <i class="bk-icon icon-search" @click="getTableData"></i>
            </div>
        </div>
        <cmdb-table
            class="collect-table"
            :loading="$loading('searchDataCollection')"
            :checked.sync="table.checked"
            :header="table.header"
            :list="table.list"
            :wrapperMinusHeight="240"
            @handleSortChange="handleSortChange"
            @handlePageChange="handlePageChange"
            @handleSizeChange="handleSizeChange"
            @handleCheckAll="handleCheckAll">
            <template slot="id" slot-scope="{ item }">
                <label class="table-checkbox bk-form-checkbox bk-checkbox-small"
                    @click.stop>
                    <input type="checkbox"
                        :value="`${item['bk_cloud_id']}#${item['bk_host_innerip']}#${item['bk_biz_id']}`"
                        v-model="table.checked">
                </label>
            </template>
            <template slot="status" slot-scope="{ item }">
                <div class="status-wrapper" @mouseover="setTooltip($event, item)" @mouseleave="removeTooltip">
                    <template v-if="item.status['report_status'] !== 'normal' || item.status['collector_status'] === 'pending'">
                        <div class="bk-spin-loading bk-spin-loading-mini bk-spin-loading-primary">
                            <div class="rotate rotate1"></div>
                            <div class="rotate rotate2"></div>
                            <div class="rotate rotate3"></div>
                            <div class="rotate rotate4"></div>
                            <div class="rotate rotate5"></div>
                            <div class="rotate rotate6"></div>
                            <div class="rotate rotate7"></div>
                            <div class="rotate rotate8"></div>
                        </div>
                        <span class="text" :id="item['bk_host_innerip']" v-if="item.status['report_status'] !== 'normal'">{{$t('NetworkDiscovery["上报中"]')}}</span>
                        <span class="text" :id="item['bk_host_innerip']" v-else>{{$t('NetworkDiscovery["下发中"]')}}</span>
                    </template>
                    <template v-else-if="item.status['collector_status'] === 'abnormal' || item.status['config_status'] !== 'normal'">
                        <i class="bk-icon icon-circle color-danger" :id="item['bk_host_innerip']"></i>
                        <span class="text">{{$t('NetworkDiscovery["异常"]')}}</span>
                    </template>
                    <template v-else>
                        <i class="bk-icon icon-circle color-success" :id="item['bk_host_innerip']"></i>
                        <span class="text">{{$t('NetworkDiscovery["正常"]')}}</span>
                    </template>
                </div>
            </template>
            <template slot="period" slot-scope="{ item }">
                {{periodMap[item.config.period]}}
            </template>
            <template slot="operation" slot-scope="{ item }">
                <span class="text-primary" @click="showConfig(item)">{{$t('EventPush["配置"]')}}</span>
            </template>
        </cmdb-table>
        <bk-dialog
        class="config-dialog"
        :is-show.sync="configDialog.isShow" 
        :has-header="false"
        :has-footer="false"
        :quick-close="false"
        :close-icon="false"
        padding="0"
        :width="424">
            <div slot="content" class="dialog-content">
                <div class="content-box">
                    <h2 class="title">
                        {{$t('NetworkDiscovery["配置采集器"]')}}
                    </h2>
                    <label>
                        <span>{{$t('NetworkDiscovery["SNMP扫描范围"]')}}</span>
                        <span class="color-danger">*</span>
                        <i class="bk-icon icon-exclamation-circle" v-tooltip="{content: htmlEncode(), classes: 'collect-tooltip'}"></i>
                    </label>
                    <textarea name="scan_range" id="" cols="30" rows="10" v-validate="'required'" v-model.trim="configDialog.scan_range"></textarea>
                    <div v-show="errors.has('scan_range')" class="color-danger">{{ errors.first('scan_range') }}</div>
                    <label>
                        <span>{{$t('NetworkDiscovery["采集频率"]')}}</span>
                        <span class="color-danger">*</span>
                    </label>
                    <bk-selector
                        :list="configDialog.periodList"
                        :selected.sync="configDialog.period"
                    ></bk-selector>
                    <label>
                        <span>{{$t('NetworkDiscovery["团体字"]')}}</span>
                        <span class="color-danger">*</span>
                        <i class="bk-icon icon-exclamation-circle" v-tooltip="'Community String'"></i>
                    </label>
                    <input type="text" name="community" class="cmdb-form-input" v-validate="'required'" v-model.trim="configDialog.community">
                    <div v-show="errors.has('community')" class="color-danger">{{ errors.first('community') }}</div>
                </div>
                <div class="footer">
                    <bk-button type="primary" @click="saveConfig">
                        {{$t('NetworkDiscovery["保存并下发"]')}}
                    </bk-button>
                    <bk-button type="default" @click="hideConfig">
                        {{$t('Common["取消"]')}}
                    </bk-button>
                </div>
            </div>
        </bk-dialog>
        <div class="status-tips" ref='tooltipContent' v-if="tooltip.id">
            <p class="tips-content">{{$t('NetworkDiscovery["采集器状态"]')}}：
                <span :class="tooltip.content.status.collector_status === 'normal' ? 'color-success' : 'color-danger'">
                    {{tooltip.content.status.collector_status === 'normal' ? $t('NetworkDiscovery["正常"]') : $t('NetworkDiscovery["异常"]')}}
                </span>
            </p>
            <p class="tips-content">{{$t('NetworkDiscovery["配置状态"]')}}：
                <span :class="tooltip.content.status.config_status === 'normal' ? 'color-success' : 'color-danger'">
                    {{tooltip.content.status.collector_status === 'normal' ? $t('NetworkDiscovery["正常"]') : $t('NetworkDiscovery["更新失败"]')}}
                </span>
            </p>
            <p class="tips-content">{{$t('NetworkDiscovery["上报状态"]')}}：
                <span :class="{'color-success': tooltip.content.status.report_status === 'normal'}">
                    {{tooltip.content.status.collector_status === 'normal' ? $t('NetworkDiscovery["完成"]') : $t('NetworkDiscovery["上报中"]')}}
                </span>
            </p>
        </div>
    </div>
</template>

<script>
    import { mapActions } from 'vuex'
    export default {
        data () {
            return {
                filter: {
                    text: ''
                },
                table: {
                    header: [{
                        id: 'id',
                        type: 'checkbox'
                    }, {
                        id: 'bk_cloud_name',
                        name: this.$t('Hosts["云区域"]')
                    }, {
                        id: 'bk_host_innerip',
                        name: this.$t('Common["内网IP"]')
                    }, {
                        id: 'status',
                        name: this.$t('ProcessManagement["状态"]')
                    }, {
                        id: 'version',
                        name: `${this.$t('NetworkDiscovery["版本"]')}`
                    }, {
                        id: 'period',
                        name: this.$t('NetworkDiscovery["采集频率"]')
                    }, {
                        id: 'report_total',
                        name: this.$t('NetworkDiscovery["采集统计"]')
                    }, {
                        id: 'operation',
                        name: this.$t('Association["操作"]'),
                        sortable: false
                    }],
                    list: [],
                    checked: [],
                    pagination: {
                        count: 0,
                        size: 10,
                        current: 1
                    },
                    defaultSort: '-last_time',
                    sort: '-last_time'
                },
                configDialog: {
                    isShow: false,
                    period: '',
                    scan_range: '',
                    community: 'public',
                    bk_host_innerip: '',
                    bk_cloud_id: '',
                    periodList: [{
                        id: '∞',
                        name: this.$t('NetworkDiscovery["手动"]')
                    }, {
                        id: '12H',
                        name: this.$t('NetworkDiscovery["12H"]')
                    }, {
                        id: '24H',
                        name: this.$t('NetworkDiscovery["24H"]')
                    }, {
                        id: '7D',
                        name: this.$t('NetworkDiscovery["7D"]')
                    }]
                },
                tooltip: {
                    instance: null,
                    content: null,
                    id: ''
                },
                periodMap: {
                    '∞': this.$t('NetworkDiscovery["手动"]'),
                    '12H': this.$t('NetworkDiscovery["12H"]'),
                    '24H': this.$t('NetworkDiscovery["24H"]'),
                    '7D': this.$t('NetworkDiscovery["7D"]')
                }
            }
        },
        created () {
            this.getTableData()
        },
        methods: {
            ...mapActions('netDataCollection', [
                'searchDataCollection',
                'collectDataCollection',
                'updateDataCollection'
            ]),
            executionDiscovery () {
                let params = {
                    collectors: []
                }
                this.table.checked.map(key => {
                    let keyArr = key.split('#')
                    params.collectors.push({
                        bk_cloud_id: Number(keyArr[0]),
                        bk_host_innerip: keyArr[1],
                        bk_biz_id: Number(keyArr[2])
                    })
                })
                this.collectDataCollection({params, config: {requestId: 'collectDataCollection'}})
            },
            showConfig (item) {
                this.configDialog.scan_range = item.config['scan_range'] === null ? '' : item.config['scan_range'].join('\n')
                this.configDialog.bk_host_innerip = item['bk_host_innerip']
                this.configDialog.bk_cloud_id = item['bk_cloud_id']
                this.configDialog.bk_biz_id = item['bk_biz_id']
                this.configDialog.period = item.config.period
                this.configDialog.community = item.config.community
                this.configDialog.isShow = true
            },
            hideConfig () {
                this.configDialog.isShow = false
            },
            async saveConfig () {
                if (!await this.$validator.validateAll()) {
                    return
                }
                let params = {
                    bk_cloud_id: this.configDialog['bk_cloud_id'],
                    bk_host_innerip: this.configDialog['bk_host_innerip'],
                    bk_biz_id: this.configDialog['bk_biz_id'],
                    config: {
                        scan_range: this.configDialog.scan_range.split(/\n|;|；|,|，/),
                        period: this.configDialog.period,
                        community: this.configDialog.community
                    }
                }
                await this.updateDataCollection({params, config: {requestId: 'updateDataCollection'}})
                this.hideConfig()
                this.getTableData()
            },
            removeTooltip () {
                this.tooltip.instance && this.tooltip.instance.destroy()
            },
            setTooltip (event, item) {
                this.tooltip.content = item
                this.tooltip.id = item['bk_host_innerip']
                this.$nextTick(() => {
                    this.tooltip.instance && this.tooltip.instance.destroy()
                    this.tooltip.instance = this.$tooltips({
                        duration: -1,
                        theme: 'light',
                        zIndex: 9999,
                        container: document.body,
                        target: document.getElementById(item['bk_host_innerip'])
                    })
                    this.tooltip.instance.$el.append(this.$refs.tooltipContent)
                })
            },
            htmlEncode () {
                let temp = document.createElement('div')
                temp.innerHTML = `${this.$t('NetworkDiscovery["填写格式"]')}&lt;/br&gt;${this.$t('NetworkDiscovery["指定IP"]')}：192.168.1.1&lt;/br&gt;IP ${this.$t('NetworkDiscovery["范围"]')}：192.168.1.1-192.168.1.200&lt;/br&gt;cidr ip ${this.$t('NetworkDiscovery["范围"]')}：192.168.1.1/32`
                let output = temp.innerText
                temp = null
                return output
            },
            async getTableData () {
                let pagination = this.table.pagination
                let params = {
                    query: this.filter.text,
                    page: {
                        start: (pagination.current - 1) * pagination.size,
                        limit: pagination.size,
                        sort: this.table.sort
                    }
                }
                const res = await this.searchDataCollection({params, config: {requestId: 'searchDataCollection'}})
                this.table.pagination.count = res.count
                this.table.list = res.info
                if (res.info.length) {
                    let index = this.table.header.findIndex(header => header.id === 'version')
                    this.table.header[index] = {
                        id: 'version',
                        name: `${this.$t('NetworkDiscovery["版本"]')}(${this.$t('NetworkDiscovery["最新"]')}${res.info[0]['latest_ersion']})`
                    }
                    this.table.header.splice()
                }
            },
            handleSortChange (sort) {
                this.table.sort = sort
                this.handlePageChange(1)
            },
            handleSizeChange (size) {
                this.table.pagination.size = size
                this.handlePageChange(1)
            },
            handlePageChange (page) {
                this.table.pagination.current = page
                this.getTableData()
            },
            handleCheckAll () {
                this.table.checked = this.table.list.map(item => `${item['bk_cloud_id']}#${item['bk_host_innerip']}#${item['bk_biz_id']}`)
            }
        }
    }
</script>

<style lang="scss" scoped>
    .collect-wrapper {
        padding-top: 20px;
        >.title {
            .input-box {
                position: relative;
                float: right;
                input {
                    width: 260px;
                }
                .icon-search {
                    position: absolute;
                    top: 9px;
                    right: 8px;
                    font-size: 18px;
                    color: $cmdbTableBorderColor;
                }
            }
        }
        .collect-table {
            margin-top: 20px;
            background: #fff;
            .status-wrapper {
                .bk-icon {
                    font-weight: bold;
                }
                .bk-icon,
                .bk-spin-loading,
                .text {
                    vertical-align: middle;
                }
            }
        }
        .config-dialog {
            .dialog-content {
                .content-box {
                    padding: 20px;
                    >h2 {
                        color: #333948;
                        font-size: 22px;
                        line-height: 1;
                    }
                    >label {
                        display: block;
                        margin: 15px 0 5px;
                        span,
                        i {
                            vertical-align: middle;
                        }
                    }
                    >textarea {
                        width: 100%;
                        height: 80px;
                        border-color: $cmdbBorderColor;
                        resize: none;
                        outline: none;
                        border-radius: 2px;
                    }
                    .info {
                        margin-top: 20px;
                        background: #fff3da;
                        border-radius: 2px;
                        width: 100%;
                        padding-left: 20px;
                        height: 42px;
                        line-height: 40px;
                        font-size: 0;
                        border: 1px solid #ffc947;
                        .bk-icon {
                            position: relative;
                            top: -1px;
                            margin-right: 10px;
                            color: #ffc947;
                            font-size: 20px;
                        }
                        span {
                            font-size: 14px;
                            vertical-align: middle;
                        }
                    }
                }
                .footer {
                    border-top: 1px solid #e5e5e5;
                    padding-right: 20px;
                    text-align: right;
                    font-size: 0;
                    background: #fafbfd;
                    height: 54px;
                    line-height: 54px;
                    .bk-default {
                        margin-left: 10px;
                    }
                }
            }
        }
    }
    .status-tips {
        padding: 5px 10px;
        font-size: 12px;
    }
</style>

<style lang="scss">
    .collect-tooltip {
        .tooltip-inner {
            max-width: 300px;
        }
    }
</style>
