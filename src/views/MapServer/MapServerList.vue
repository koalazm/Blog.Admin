<template>
    <section>
        <!--工具条-->
        <toolbar :buttonList="buttonList" @callFunction="callFunction"></toolbar>
        <!--列表-->
        <el-col :span="isPreviewOpen?12:24">
            <el-table :data="mapServers" highlight-current-row 
            @current-change="selectCurrentRow"
            v-loading="listLoading" @selection-change="selsChange"
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
                <el-pagination layout="prev, pager, next" @current-change="handleCurrentChange" :page-size="50"
                            :total="total" style="float:right;">
                </el-pagination>
            </el-col>

        </el-col>
        <el-col :span="isPreviewOpen?12:0">

        </el-col>
         <!--编辑界面-->
        <el-dialog
        title="编辑"
        :visible.sync="editFormVisible"
        v-model="editFormVisible"
        :close-on-click-modal="false"
        >
        <el-form :model="editForm" label-width="80px" :rules="editFormRules" ref="editForm">
            <el-form-item label="服务名称" prop="ServiceName">
            <el-input v-model="editForm.ServiceName" auto-complete="off"></el-input>
            </el-form-item>
            <el-form-item label="服务别名" prop="ServiceAlais">
            <el-input v-model="editForm.ServiceAlais" auto-complete="off"></el-input>
            </el-form-item>
            <el-form-item label="服务类型" prop="ServiceType">
            <el-input v-model="editForm.ServiceType" auto-complete="off"></el-input>
            </el-form-item>
            <el-form-item label="服务地址" prop="ServiceUrl">
            <el-input v-model="editForm.ServiceUrl" auto-complete="off"></el-input>
            </el-form-item>
            <el-form-item label="状态" prop="Enabled">
            <el-select v-model="editForm.Enabled" placeholder="请选择服务状态">
                <el-option
                v-for="item in statusList"
                :key="item.value"
                :label="item.name"
                :value="item.value"
                ></el-option>
            </el-select>
            </el-form-item>
        </el-form>

        <div slot="footer" class="dialog-footer">
            <el-button @click.native="editFormVisible = false">取消</el-button>
            <el-button type="primary" @click.native="editSubmit" :loading="editLoading">提交</el-button>
        </div>
        </el-dialog>

    <!--新增界面-->
    <el-dialog
      title="新增服务"
      :visible.sync="addFormVisible"
      v-model="addFormVisible"
      :close-on-click-modal="false"
    >
      <el-form :model="addForm" label-width="80px" :rules="addFormRules" ref="addForm">
       <el-form-item label="服务名称" prop="ServiceName">
            <el-input v-model="addForm.ServiceName" auto-complete="off"></el-input>
            </el-form-item>
            <el-form-item label="服务别名" prop="ServiceAlais">
            <el-input v-model="addForm.ServiceAlais" auto-complete="off"></el-input>
            </el-form-item>
            <el-form-item label="服务类型" prop="ServiceType">
            <el-input v-model="addForm.ServiceType" auto-complete="off"></el-input>
            </el-form-item>
            <el-form-item label="服务地址" prop="ServiceUrl">
            <el-input v-model="addForm.ServiceUrl" auto-complete="off"></el-input>
            </el-form-item>
            <el-form-item label="状态" prop="Enabled">
            <el-select v-model="addForm.Enabled" placeholder="请选择服务状态">
                <el-option
                v-for="item in statusList"
                :key="item.value"
                :label="item.name"
                :value="item.value"
                ></el-option>
            </el-select>
            </el-form-item>
      </el-form>
      <div slot="footer" class="dialog-footer">
        <el-button @click.native="addFormVisible = false">取消</el-button>
        <el-button type="primary" @click.native="addSubmit" :loading="addLoading">确定</el-button>
      </div>
    </el-dialog>
    </section>
</template>

<script>
    import util from '../../../util/date'
    import {getMapServerListPage, addMapServer, batchRemoveMapServer, editMapServer, removeMapServer } from '../../api/api';
    import { getButtonList } from "../../promissionRouter";
    import Toolbar from "../../components/Toolbar";

    export default {
        components: { Toolbar },
        data() {
            return {
                filters: {
                    name: ''
                },
                mapServers: [],
                statusList: [
                    { name: "激活", value: true },
                    { name: "禁用", value: false }
                ],
                isPreviewOpen: false,
                total: 0,
                buttonList: [],
                currentRow: null,
                page: 1,
                listLoading: false,
                sels: [],//列表选中列
                
                editFormVisible: false, //编辑界面是否显示
                editLoading: false,
                editFormRules: {
                    Name: [{ required: true, message: "请输入服务名称", trigger: "blur" }],
                    Enabled: [{ required: true, message: "请选择状态", trigger: "blur" }]
                },
                //编辑界面数据
                editForm: {
                    Id: 0,
                    ServiceName: "",
                    ServiceAlais: "",
                    ServiceType: "",
                    ServiceUrl: "",
                    Enabled: false
                },

                addFormVisible: false, //新增界面是否显示
                addLoading: false,
                addFormRules: {
                    Name: [{ required: true, message: "请输入服务名称", trigger: "blur" }],
                    Enabled: [{ required: true, message: "请选择状态", trigger: "blur" }]
                },
                //新增界面数据
                addForm: {
                    ServiceName: "",
                    ServiceAlais: "",
                    ServiceType: "",
                    ServiceUrl: "",
                    Enabled: false
                },
                //projectName: "",
                //uploadURL: "/api/zrzyfile/uploadfile",
                //uploadFiles: [] //上传的文件列表
            }
        },
        methods: {
            selectCurrentRow(val) {
                this.currentRow = val;
            },
            callFunction(item) {
                this.filters = {
                    name: item.search
                };
                this[item.Func].apply(this, item);
            },
            handleCurrentChange(val) {
                this.page = val;
                this.getMapServers();
            },
            //获取用户列表
            getMapServers() {
                let para = {
                    page: this.page,
                    key: this.filters.name
                };
                this.listLoading = true;
                //NProgress.start();
                getMapServerListPage(para).then((res) => {

                    this.total = res.data.response.dataCount;
                    this.mapServers = res.data.response.data;
                    this.listLoading = false;
                    //NProgress.done();
                });
            },
            //预览
            handlePreview(){
                let row = this.currentRow;
                if (!row) {
                    this.$message({
                    message: "请选择要预览的服务！",
                    type: "error"
                    });

                    return;
                }

                this.isPreviewOpen = !this.isPreviewOpen

            },
            //删除
            handleDel(){
                let row = this.currentRow;
                if (!row) {
                    this.$message({
                    message: "请选择要删除的一行数据！",
                    type: "error"
                    });

                    return;
                }
                this.$confirm('确认删除该记录吗?', '提示', {
                    type: 'warning'
                }).then(() => {
                    this.listLoading = true;
                    //NProgress.start();
                    let para = {id: row.Id};
                    removeMapServer(para).then((res) => {
                        console.log(res);
                        if (util.isEmt.format(res)) {
                            this.listLoading = false;
                            return;
                        }
                        this.listLoading = false;
                        //NProgress.done();
                        if (res.data.success) {
                            this.$message({
                                message: '删除成功',
                                type: 'success'
                            });

                        } else {
                            this.$message({
                                message: res.data.msg,
                                type: 'error'
                            });
                        }

                        this.getMapServers();
                    });
                }).catch(() => {

                });
            },
            //显示编辑界面
            handleEdit() {
                let row = this.currentRow;
                if (!row) {
                    this.$message({
                    message: "请选择要编辑的一行数据！",
                    type: "error"
                    });
                    return;
                }

                this.editFormVisible = true;
                this.editForm = Object.assign({}, row);
            },
            //显示新增界面
            handleAdd() {
                this.addFormVisible = true;
                this.addForm = {
                    ServiceName: "",
                    ServiceAlais: "",
                    ServiceType: "",
                    ServiceUrl: "",
                    Enabled: false,
                    CreateBy: ""
                };
            },
            //编辑
            editSubmit: function() {
            this.$refs.editForm.validate(valid => {
                if (valid) {
                this.$confirm("确认提交吗？", "提示", {}).then(() => {
                    this.editLoading = true;
                    //NProgress.start();
                    let para = Object.assign({}, this.editForm);
                    editMapServer(para).then(res => {
                    if (util.isEmt.format(res)) {
                        this.editLoading = false;
                        return;
                    }

                    if (res.data.success) {
                        this.editLoading = false;
                        //NProgress.done();
                        this.$message({
                        message: res.data.msg,
                        type: "success"
                        });
                        this.$refs["editForm"].resetFields();
                        this.editFormVisible = false;
                        this.getMapServerListPage();
                    } else {
                        this.$message({
                        message: res.data.msg,
                        type: "error"
                        });
                    }
                    });
                });
                }
            });
            },
            addSubmit: function(){
                let _this = this;
                this.$refs.addForm.validate((valid) => {
                    if (valid) {
                        this.$confirm('确认提交吗？', '提示', {}).then(() => {
                            this.addLoading = true;
                            //NProgress.start();
                            let para = Object.assign({}, this.addForm);

                            para.CreateTime = util.formatDate.format(new Date(), 'yyyy-MM-dd');
                            para.ModifyTime = para.CreateTime;
                            para.IsDeleted = false;

                            var user = JSON.parse(window.localStorage.user);

                            if (user && user.uID > 0) {
                                para.CreateId = user.uID;
                                para.CreateBy = user.uRealName;
                            } else {
                                this.$message({
                                    message: '用户信息为空，先登录',
                                    type: 'error'
                                });
                                _this.$router.replace(_this.$route.query.redirect ? _this.$route.query.redirect : "/");
                            }


                            addMapServer(para).then((res) => {
                                if (util.isEmt.format(res)) {
                                    this.addLoading = false;
                                    return;
                                }
                                if (res.data.success) {
                                    this.addLoading = false;
                                    //NProgress.done();
                                    this.$message({
                                        message: res.data.msg,
                                        type: 'success'
                                    });
                                    this.$refs['addForm'].resetFields();
                                    this.addFormVisible = false;
                                    this.getMapServers();
                                }
                                else {
                                    this.$message({
                                        message: res.data.msg,
                                        type: 'error'
                                    });

                                }

                            });

                        });
                    }
                });
            },
            selsChange: function (sels) {
                this.sels = sels;
            },
            //批量删除
            batchRemove: function () {

                var ids = this.sels.map(item => item.uID).toString();
                this.$confirm('确认删除选中记录吗？', '提示', {
                    type: 'warning'
                }).then(() => {
                    this.listLoading = true;
                    //NProgress.start();
                    let para = {ids: ids};

                    batchRemoveMapServer(para).then((res) => {
                        this.listLoading = false;
                        //NProgress.done();
                        this.$message({
                            message: '该功能未开放',
                            type: 'warning'
                        });
                    });
                }).catch(() => {

                });
            },
        },
        mounted() {
            var result = this.getMapServers();
            console.log(result);
            let routers = window.localStorage.router
            ? JSON.parse(window.localStorage.router)
            : [];
            this.buttonList = getButtonList(this.$route.path, routers);
            // console.log(this.$route.path)
            // console.log(routers);
        }
    }

</script>

<style scoped>

</style>
