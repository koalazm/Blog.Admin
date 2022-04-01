<template>
    <section>
        <!--工具条-->
     <toolbar :buttonList="buttonList" @callFunction="callFunction"></toolbar>
        <!--列表-->
        <el-table :data="files" highlight-current-row 
        @current-change="selectCurrentRow"
        v-loading="listLoading" @selection-change="selsChange"
                  style="width: 100%;">
            <el-table-column type="selection" width="50">
            </el-table-column>
            <el-table-column type="index" width="80">
            </el-table-column>
            <el-table-column prop="FileName" label="文件名称" width="" sortable>
            </el-table-column>
            <el-table-column prop="FileSize" label="文件大小" width="" sortable>
                 <template slot-scope="scope">
                     <span>{{scope.row.FileSize}} MB</span>
                </template>
            </el-table-column>
            <el-table-column prop="FileType" label="文件类型" width="" sortable>
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

         <!--编辑界面-->
        <el-dialog
        title="编辑"
        :visible.sync="editFormVisible"
        v-model="editFormVisible"
        :close-on-click-modal="false"
        >
        <el-form :model="editForm" label-width="80px" :rules="editFormRules" ref="editForm">
            <el-form-item label="文件名" prop="FileName">
            <el-input v-model="editForm.FileName" auto-complete="off"></el-input>
            </el-form-item>
            <el-form-item label="文件大小" prop="FileSize">
            <el-input v-model="editForm.FileSize" auto-complete="off"></el-input>
            </el-form-item>
            <el-form-item label="状态" prop="Enabled">
            <el-select v-model="editForm.Enabled" placeholder="请选择文件状态">
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
      title="上传文件"
      :visible.sync="addFormVisible"
      v-model="addFormVisible"
      :close-on-click-modal="false"
    >
      <el-form :model="addForm" label-width="80px" :rules="addFormRules" ref="addForm">
         <el-upload 
            class="upload-css"
            :file-list="uploadFiles"
            ref="upload"
            :on-success="upLoadSuccess"
            :on-error="upLoadError"
            :action="uploadURL"
            :auto-upload="false">
            <el-button slot="trigger" size="small" type="primary">选取文件</el-button>
            <el-button style="margin-left: 10px;" size="small" type="success" @click="submitUpload">上传到服务器</el-button>
            <div slot="tip" class="el-upload__tip">只能上传pdf文件，且不超过50MB</div>
        </el-upload>
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
    import {getFileListPage, batchRemoveFile, editFile, removeFile, downloadFile, getFilePath } from '../../api/api';
    import { getButtonList } from "../../promissionRouter";
    import Toolbar from "../../components/Toolbar";

    export default {
        components: { Toolbar },
        data() {
            return {
                filters: {
                    name: ''
                },
                files: [],
                statusList: [
                    { name: "激活", value: true },
                    { name: "禁用", value: false }
                ],
                total: 0,
                buttonList: [],
                currentRow: null,
                page: 1,
                listLoading: false,
                sels: [],//列表选中列
                
                editFormVisible: false, //编辑界面是否显示
                editLoading: false,
                editFormRules: {
                    Name: [{ required: true, message: "请输入文件名称", trigger: "blur" }],
                    Enabled: [{ required: true, message: "请选择状态", trigger: "blur" }]
                },
                //编辑界面数据
                editForm: {
                    Id: 0,
                    FileName: "",
                    FileSize: "",
                    Enabled: false
                },

                addFormVisible: false, //新增界面是否显示
                addLoading: false,
                addFormRules: {
                    Name: [{ required: true, message: "请输入文件名称", trigger: "blur" }],
                    Enabled: [{ required: true, message: "请选择状态", trigger: "blur" }]
                },
                //新增界面数据
                addForm: {
                    FileName: "",
                    FileSize: "",
                    Enabled: false
                },
                projectName: "",
                uploadURL: "/api/zrzyfile/uploadfile",
                uploadFiles: [] //上传的文件列表
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
                this.getFiles();
            },
            //获取用户列表
            getFiles() {
                let para = {
                    page: this.page,
                    key: this.filters.name
                };
                this.listLoading = true;
                //NProgress.start();
                getFileListPage(para).then((res) => {

                    this.total = res.data.response.dataCount;
                    this.files = res.data.response.data;
                    this.listLoading = false;
                    //NProgress.done();
                });
            },
            handleDownload(){
                let row = this.currentRow;
                if (!row) {
                    this.$message({
                    message: "请选择需要下载的文件！",
                    type: "error"
                    });

                    return;
                }
                let para = {filename: row.GuidName};
               downloadFile(para).then((res)=>{
                    let blob = new Blob([res.data]);
                    let url = window.URL.createObjectURL(blob);
                    let aLink = document.createElement("a");
                    aLink.style.display = "none";
                    aLink.href = url;
                    aLink.setAttribute("download", row.FileName);                            4
                    document.body.appendChild(aLink);
                    aLink.click();
                    document.body.removeChild(aLink); //下载完成移除元素
                    window.URL.revokeObjectURL(url); //释放掉blob对象
               });
            },
            //删除
            handleDel: function() {
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
                    removeFile(para).then((res) => {
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

                        this.getFiles();
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
                CreateBy: "",
                Name: "",
                Enabled: ""
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
                    editFile(para).then(res => {
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
                        this.getFiles();
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

                    batchRemoveFile(para).then((res) => {
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
            
            //文件上传
            submitUpload() {
                this.$refs.upload.submit();
            },
            //文件上传成功时的钩子
            upLoadSuccess(response, file, fileList) {
                
                if (response.success) {
                    this.$message({
                        message: response.msg,
                        type: "success"
                    });
                    this.addFormVisible = false;
                    this.getFiles();
                } else {
                    this.$message({
                        message: response.msg,
                        type: "error"
                    });
                }
                
            },
            //文件上传失败时的钩子
            upLoadError(response, file, fileList) {
                this.$message({
                    message: response.msg,
                    type: "error"
                });
            }
        },
        mounted() {
            this.getFiles();

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
