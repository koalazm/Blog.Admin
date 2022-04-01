<template>
    <section>
        <!--工具条-->
        <el-col :span="6" class="toolbar roles">
            <el-card class="box-card">
                <div slot="header" class="clearfix">
                    <span>工程列表</span>
                    <el-button @click="getProjects" style="float: right; padding: 3px 0" type="text">刷新</el-button>
                </div>
                <div v-for="p in projects" :key="p.Id" @click="handleProjectChange(p.Id)" :class="p.Id==projectid ? 'active':''"
                     class="text item role-item">
                    {{p.ProjectName }}
                </div>
            </el-card>

        </el-col>
        <el-col :span="18" class="toolbar perms morechildren">
            <el-card class="box-card">
                <div slot="header" class="clearfix">
                    <span>地图服务列表</span> 
                    <el-button :loading="loadingSave" @click="saveAssign" style="float: right; padding: 3px 0" type="text">{{loadingSaveStr}}</el-button>
                </div>
                <el-table ref="mapServerTable" :data="mapservers" highlight-current-row
                    v-loading="loadingSave" @selection-change="handleTableSelsChange" 
                    style="width: 100%;">
                    <el-table-column type="selection" width="50">
                    </el-table-column>
                    <el-table-column type="index" width="80">
                    </el-table-column>
                    <el-table-column prop="ServiceName" label="服务名称" width="" sortable>
                    </el-table-column>
                    <el-table-column prop="ServiceAlais" label="服务别名" width="" sortable>
                    </el-table-column>
                    <el-table-column prop="ServiceType" label="服务类型" width="" sortable>
                    </el-table-column>
                    <el-table-column prop="ServiceUrl" label="服务地址" width="" sortable>
                    </el-table-column>
                    <el-table-column prop="Enabled" label="状态" width="" sortable>
                        <template slot-scope="scope">
                            <el-tag
                                    :type="scope.row.Enabled == 1  ? 'success' : 'danger'"
                                    disable-transitions>{{scope.row.Enabled == 1 ? "正常":"禁用"}}
                            </el-tag>
                        </template>
                    </el-table-column>
                </el-table>

                <!--工具条-->
                <el-col :span="24" class="toolbar">
                    <el-pagination layout="prev, pager, next" @current-change="handlePagerChange" :page-size="50"
                                :total="total" style="float:right;">
                    </el-pagination>
                </el-col>
            </el-card>
        </el-col>
    </section>
</template>

<script>
    import util from '../../../util/date'
    import {getProjectListPage, getMapServerListPage, getMapServerIds, addProjectMapServers} from '../../api/api';

    let id = 1000;

    export default {
        data() {
            return {
                page: 1,
                filters: {
                    name: ''
                },
                total: 0,
                sels: [],//列表选中列
                projects: [],
                projectid: 0,
                mapservers: [],
                loadingSaveStr:'保存',
                loadingSave:false,
            }
        },
        methods: {
            //反选
            reverse(ls){
                console.log(this.mapservers);
                console.log(ls);
                if(ls && ls.length){
                    for(let i=0;i<ls.length;i++){
                        let btn = ls[i];
                        let findBtnIndex = this.assignBtns.findIndex(t=>t == btn.value);
                        if(findBtnIndex>-1){
                            this.assignBtns.splice(findBtnIndex,1);
                        }else{
                            this.assignBtns.push(""+btn.value);
                        }
                    }
                }
            },
            //性别显示转换
            formatEnabled: function (row, column) {
                return row.Enabled ? '正常' : '未知';
            },
            formatCreateTime: function (row, column) {
                return (!row.CreateTime || row.CreateTime == '') ? '' : util.formatDate.format(new Date(row.CreateTime), 'yyyy-MM-dd');
            },
            //性别显示转换
            formatSex: function (row, column) {
                return row.sex == 1 ? '男' : row.sex == 0 ? '女' : '未知';
            },
            // 时间格式化
            dateFormat: function (row, column, cellValue, index){
                return format(row[column.property])
            },
            //获取角色列表
            getProjects() {
                getProjectListPage().then((res) => {
                    this.projects = res.data.response.data;
                    this.getMapServers();
                });
            },
            handleTableSelsChange: function (sels) {
                this.sels = sels;
            },
            handlePagerChange(val) {
                this.page = val;
                this.getMapServers();
            },
            //获取用户
            getMapServers() {
                let para = {
                    page: this.page,
                    key: this.filters.name
                };
                this.loadingSave = true;

                getMapServerListPage(para).then((res) => {
                    this.total = res.data.response.dataCount;
                    this.mapservers = res.data.response.data;
                    this.loadingSave = false;
                    this.loadingSaveStr='保存';
                });
            },
            //通过项目id获取用户Id，
            getMapServerIds(pid) {
                this.$refs.mapServerTable.clearSelection();

                let para = {pid: pid}
                getMapServerIds(para).then((res) => {
                    this.loadingSave=false;
                    this.loadingSaveStr='保存';

                    if(res.data.response){
                        this.mapservers.forEach(mapserver=> {
                            res.data.response.forEach(uid => {
                                if(mapserver.Id == uid){
                                    this.$refs.mapServerTable.toggleRowSelection(mapserver);
                                }
                            });
                        });
                    }
                });
            },
            handleProjectChange(pid) {
                this.loadingSave=true;
                this.loadingSaveStr='加载中...';
                this.projectid = pid;
                this.getMapServerIds(pid);
            },
            saveAssign() {
                var mids = [];
                this.sels.forEach(select => {
                    mids.push(select.Id);
                });
                let param = {mids:mids, pid:this.projectid};
                addProjectMapServers(param).then((res) => {
                    if(res.data.success == true){
                        this.$message({
                                message: "数据更新成功",
                                type: 'success'
                            });
                    }else{
                        this.$message({
                                message: "数据更新失败",
                                type: 'error'
                            }); 
                    }
                    
                });
            },
        },
        mounted() {
            this.loadingSave=true;
            this.loadingSaveStr='加载中...';
            this.getProjects();
        }
    }

</script>

<style>
    .role-item {
        cursor: pointer;
        padding: 10px;
    }

    .role-item.active {
        background: #ebf5ff;
    }

    .role-item:hover {
        background: #ebf5ff;
    }

    .custom-tree-node {
        flex: 1;
        display: flex;
        align-items: center;
        justify-content: space-between;
        font-size: 14px;
        padding-right: 8px;
    }

    .text {
        font-size: 14px;
    }

    .clearfix:before,
    .clearfix:after {
        display: table;
        content: "";
    }

    .clearfix:after {
        clear: both
    }

    .box-card {
        width: 90%;
    }


    .morechildren .el-checkbox{
        margin-right: 5px !important;
        float: left;
    }
    .morechildren .el-checkbox-group{
        margin-left: 50px;
        padding: 5px;
    }
    .morechildren .el-tree-node__content{
        height: auto !important;
    }
</style>
