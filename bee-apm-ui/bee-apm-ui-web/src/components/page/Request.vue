<template>
    <div>
        <el-row v-show="isShowTable">
            <el-card>
                <el-form ref="form" :model="form" label-width="80px" width="100%">
                    <el-row :gutter="20">
                        <el-col :span="7">
                            <el-form-item label="环境">
                                <el-select v-model="form.env" placeholder="请选择">
                                    <el-option
                                        v-for="item in envOptions"
                                        :key="item.value"
                                        :label="item.name"
                                        :value="item.value">
                                    </el-option>
                                </el-select>
                            </el-form-item>
                        </el-col>
                        <el-col :span="7">
                            <el-form-item label="应用">
                                <el-select v-model="form.app" placeholder="请选择">
                                    <el-option
                                        v-for="item in appOptions"
                                        :key="item.value"
                                        :label="item.name"
                                        :value="item.value">
                                    </el-option>
                                </el-select>
                            </el-form-item>
                        </el-col>
                        <el-col :span="7">
                            <el-form-item label="排序">
                                <el-select v-model="form.sort" placeholder="请选择">
                                    <el-option key="time" label="时间" value="time"></el-option>
                                    <el-option key="spend" label="耗时" value="spend"></el-option>
                                </el-select>
                            </el-form-item>
                        </el-col>
                        <el-col :span="3" align="right">
                            <el-button @click="queryRequestList(1)" type="primary">查&nbsp;&nbsp;询</el-button>
                        </el-col>
                    </el-row>
                    <el-row :gutter="20">
                        <el-col :span="7">
                            <el-form-item label="gId" style="margin-bottom: 0px">
                                <el-input v-model="form.gid"></el-input>
                            </el-form-item>
                        </el-col>
                        <el-col :span="7">
                            <el-form-item label="IP" style="margin-bottom: 0px">
                                <el-input v-model="form.ip"></el-input>
                            </el-form-item>
                        </el-col>
                    </el-row>
                </el-form>
            </el-card>
            <el-card style="margin-top: -2px">
                <el-row>
                    <el-col :span="24">
                        <ve-histogram :legend-visible='false' :extend="requestCharExtend" :data="requestChartData"
                                      :settings="requestChartSettings"></ve-histogram>
                    </el-col>
                </el-row>
                <el-row>
                    <el-col :span="24">
                        <el-table :data="requestTableData.rows" border stripe>
                            <el-table-column prop="id" label="ID" width="170" fixed>
                            </el-table-column>
                            <el-table-column prop="time" label="时间" width="150" :formatter="timeFormatter" fixed>
                            </el-table-column>
                            <el-table-column prop="gid" label="GID" width="170">
                            </el-table-column>
                            <el-table-column prop="ip" label="IP" width="120">
                            </el-table-column>
                            <el-table-column prop="env" label="环境" width="120">
                            </el-table-column>
                            <el-table-column prop="app" label="应用" width="120">
                            </el-table-column>
                            <el-table-column prop="spend" label="耗时" width="100">
                            </el-table-column>
                            <el-table-column prop="tags.url" label="URL" min-width="250">
                            </el-table-column>
                            <el-table-column label="操作" width="180" align="center" fixed="right">
                                <template slot-scope="_">
                                    <el-button type="text">参数</el-button>
                                    <el-button type="text">返回</el-button>
                                    <el-button type="text" @click="queryCallTree(_.row)">调用链</el-button>
                                </template>
                            </el-table-column>
                        </el-table>
                        <div class="pagination">
                            <el-pagination background @current-change="handlePageChange" layout="prev, pager, next"
                                           :total="requestTableData.pageTotal"
                                           :current-page="requestTableData.currPageNum">
                            </el-pagination>
                        </div>
                    </el-col>
                </el-row>
            </el-card>
        </el-row>
        <el-row v-show="isShowTree">
            <el-card>
                <el-row>
                    <el-col :span="24" align="right" style="padding-bottom: 10px">
                        <el-button type="success">刷新</el-button>
                        <el-button type="primary" @click="backButtonEvent()">返回</el-button>
                    </el-col>
                </el-row>
                <el-row>
                    <el-table
                        :data="tableListData"
                        :row-style="toggleDisplayTr"
                        border stripe
                        class="init_table">
                        <el-table-column
                            label="调用链路"
                            min-width="500"
                            show-overflow-tooltip
                            align="left">
                            <template slot-scope="_">
                                <p :style="`margin-left: ${_.row.__level * 20}px;margin-top:0;margin-bottom:0`"><i
                                    @click="toggleFoldingStatus(_.row)" class="permission_toggleFold"
                                    :class="toggleFoldingClass(_.row)"></i>{{_.row.text}}</p>
                            </template>
                        </el-table-column>

                        <el-table-column
                            align="center"
                            prop="spend"
                            width="60"
                            label="耗时(ms)">
                        </el-table-column>

                        <el-table-column
                            align="center"
                            width="50"
                            label="操作">
                            <template slot-scope="_">
                                <el-button type="text">入参</el-button>
                            </template>
                        </el-table-column>
                    </el-table>
                </el-row>
            </el-card>
        </el-row>
    </div>
</template>
<style lang='stylus' rel='stylesheet/stylus' src="../css/tree-table.styl"></style>
<style>
    .tree-anchor {
        line-height: 24px;
        height: 24px;
        width: 100%;
    }
</style>

<script>
    import bus from '../common/bus';
    import Vue from 'vue';
    import '../iconfont/iconfont.css';

    let moment = require("moment");

    export default {
        name: 'request',
        components: {
        },
        data: function () {
            return {
                tableListData: [], //调用链数据
                foldList: [], // 该数组中的值 都会在列表中进行隐藏  死亡名单
                isShowTable: true,
                isShowTree: false,
                requestChartSettings: {},
                requestCharExtend: {
                    'xAxis.0.axisLabel.rotate': 60,
                    series(v) {
                        v.forEach(i => {
                            i.barMaxWidth = 50
                        })
                        return v
                    }
                },
                requestChartData: {
                    columns: ['time', '请求量'],
                    rows: []
                },
                requestTableData: {
                    currPageNum: 1,
                    pageTotal: 150,
                    rows: []
                },
                treeData: [],
                form: {
                    gid: null,
                    env: null,
                    app: null,
                    ip: null,
                    sort: null
                },
                envOptions: [],
                appOptions: []
            }
        },
        created() {
            this.onRefresh();
            this.getPickerDate();
            this.queryEnvGroupList();
            this.queryRequestList(1);
            this.queryAppGroupList();
        },
        methods: {
            onRefresh() {
                let self = this;
                bus.$on("refreshTag", function (val) {
                    console.log("refreshTag =========== " + val);
                    if (val === 'request') {
                        console.log("request refresh.................")
                        self.getPickerDate();
                        self.queryEnvGroupList();
                        self.queryAppGroupList();
                        self.queryRequestList(1);
                    }
                });
            },
            getBeginTime() {
                return moment(this.pickerDate[0]).format('YYYY-MM-DD HH:mm');
            },
            getEndTime() {
                return moment(this.pickerDate[1]).format('YYYY-MM-DD HH:mm');
            },
            getPickerDate() {
                const self = this;
                bus.$on("pickerDateEvent", function (val) {
                    self.pickerDate = val;
                });
                bus.$emit("getPickerDateEvent");
            },
            queryEnvGroupList() {
                const url = "/api/common/getGroupList";
                this.$axios.post(url, {
                    beginTime: this.getBeginTime(),
                    endTime: this.getEndTime(),
                    group: "env"
                }).then((res) => {
                    console.log("==>queryEnvGroupList=%o", res);
                    this.envOptions = res.data.result;
                })
            },
            queryAppGroupList() {
                const url = "/api/common/getGroupList";
                this.$axios.post(url, {
                    beginTime: this.getBeginTime(),
                    endTime: this.getEndTime(),
                    group: "app"
                }).then((res) => {
                    console.log("==>queryAppGroupList=%o", res);
                    this.appOptions = res.data.result;
                })
            },
            // 分页导航
            handlePageChange(val) {
                this.queryRequestList(val);
            },
            // 表格数据
            queryRequestList(pageNum) {
                if (pageNum == 1) {
                    this.getRequestChartData();
                }
                const url = "/api/request/list";
                let params = this.form;
                params.pageNum = pageNum;
                params.beginTime = this.getBeginTime();
                params.endTime = this.getEndTime();
                this.$axios.post(url, params).then((res) => {
                    console.log("==>queryRequestList=%o", res);
                    this.requestTableData.rows = res.data.rows;
                    this.requestTableData.currPageNum = res.data.pageNum;
                    this.requestTableData.pageTotal = res.data.pageTotal;
                })
            },
            // 柱状图数据
            getRequestChartData() {
                const url = "/api/request/chart";
                let params = this.form;
                params.beginTime = this.getBeginTime();
                params.endTime = this.getEndTime();
                this.$axios.post(url, params).then((res) => {
                    console.log("==>getRequestChartData=%o", res);
                    this.requestChartData.rows = res.data.rows;
                })
            },
            timeFormatter(row, column) {
                return row.time.substring(11, 19);
            },
            queryCallTree(row) {
                console.log("==>queryCallTree row=%o", row);
                const url = "/api/request/callTree";
                let params = {gid: row.gid, time: row.time};
                this.$axios.post(url, params).then((res) => {
                    console.log("==>queryCallTree result=%o", res);
                    this.tableListData = this.formatConversion([], res.data.result);
                })
                this.isShowTable = false;
                this.isShowTree = true;
            },
            backButtonEvent() {
                this.isShowTable = true;
                this.isShowTree = false;
            },
            /*********************************
             ** 切换展开 还是折叠
             ** @params: params 当前点击行的数据
             *********************************/
            toggleFoldingStatus(params) {
                this.foldList.includes(params.__identity) ? this.foldList.splice(this.foldList.indexOf(params.__identity), 1) : this.foldList.push(params.__identity)
            },
            /*********************************
             ** 该方法会对每一行数据都做判断 如果foldList 列表中的元素 也存在与当前行的 __family列表中  则该行不展示
             *********************************/
            toggleDisplayTr({row, index}) {
                for (let i = 0; i < this.foldList.length; i++) {
                    let item = this.foldList[i]
                    // 如果foldList中元素存在于 row.__family中，则该行隐藏。  如果该行的自身标识等于隐藏元素，则代表该元素就是折叠点
                    if (row.__family.includes(item) && row.__identity !== item) return 'display:none;'
                }
                return ''
            },
            /*********************************
             ** 如果子集长度为0，则不返回字体图标。
             ** 如果子集长度为不为0，根据foldList是否存在当前节点的标识返回相应的折叠或展开图标
             ** 关于class说明：permission_placeholder返回一个占位符，具体查看class
             ** @params: params 当前行的数据对象
             *********************************/
            toggleFoldingClass(params) {
                let hasChildren = true;
                if (typeof (params.children) == "undefined" || params.children.length === 0) {
                    hasChildren = false;
                }
                return !hasChildren ? 'permission_placeholder' : (this.foldList.indexOf(params.__identity) === -1 ? 'iconfont icon-minus-square-o' : 'iconfont icon-plussquareo')
            },
            /*********************************
             ** 将树形接口数据扁平化
             ** @params: parent 为当前累计的数组  也是最后返回的数组
             ** @params: children 为当前节点仍需继续扁平子节点的数据
             ** @params: index 默认等于0， 用于在递归中进行累计叠加 用于层级标识
             ** @params: family 装有当前包含元素自身的所有父级 身份标识
             ** @params: elderIdentity 父级的  唯一身份标识
             *********************************/
            formatConversion(parent, children, index = 0, family = [], elderIdentity = 'x') {
                // children如果长度等于0，则代表已经到了最低层
                // let page = (this.startPage - 1) * 10
                if (children && children.length > 0) {
                    children.map((x, i) => {
                        // 设置 __level 标志位 用于展示区分层级
                        Vue.set(x, '__level', index)
                        // 设置 __family 为家族关系 为所有父级，包含本身在内
                        Vue.set(x, '__family', [...family, elderIdentity + '_' + i])
                        // 本身的唯一标识  可以理解为个人的身份证咯 一定唯一。
                        Vue.set(x, '__identity', elderIdentity + '_' + i)
                        parent.push(x)
                        // 如果仍有子集，则进行递归
                        if (x.children && x.children.length > 0) this.formatConversion(parent, x.children, index + 1, [...family, elderIdentity + '_' + i], elderIdentity + '_' + i)
                    })
                }
                return parent
            }
        }
    }
</script>


