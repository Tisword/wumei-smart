<template>
<div style="padding:6px;">
    <el-card v-show="showSearch" style="margin-bottom:6px;">
        <el-form :model="queryParams" ref="queryForm" :inline="true" label-width="68px" style="margin-bottom:-20px;">
            <el-form-item label="分类名称" prop="categoryName">
                <el-input v-model="queryParams.categoryName" placeholder="请输入产品分类名称" clearable size="small" @keyup.enter.native="handleQuery" />
            </el-form-item>
            <el-form-item label="系统定义" prop="isSys">
                <el-select v-model="queryParams.status" placeholder="请选择状态" clearable size="small">
                    <el-option v-for="dict in dict.type.iot_yes_no" :key="dict.value" :label="dict.label" :value="dict.value" />
                </el-select>
            </el-form-item>
            <el-form-item>
                <el-button type="primary" icon="el-icon-search" size="mini" @click="handleQuery">搜索</el-button>
                <el-button icon="el-icon-refresh" size="mini" @click="resetQuery">重置</el-button>
            </el-form-item>
        </el-form>
    </el-card>

    <el-card style="padding-bottom:100px;">
        <el-row :gutter="10" class="mb8">
            <el-col :span="1.5">
                <el-button type="primary" plain icon="el-icon-plus" size="mini" @click="handleAdd" v-hasPermi="['iot:category:add']">新增</el-button>
            </el-col>
            <el-col :span="1.5">
                <el-button type="success" plain icon="el-icon-edit" size="mini" :disabled="single" @click="handleUpdate" v-hasPermi="['iot:category:edit']">修改</el-button>
            </el-col>
            <el-col :span="1.5">
                <el-button type="danger" plain icon="el-icon-delete" size="mini" :disabled="multiple" @click="handleDelete" v-hasPermi="['iot:category:remove']">删除</el-button>
            </el-col>
            <el-col :span="1.5">
                <el-button type="warning" plain icon="el-icon-download" size="mini" @click="handleExport" v-hasPermi="['iot:category:export']">导出</el-button>
            </el-col>
            <right-toolbar :showSearch.sync="showSearch" @queryTable="getList"></right-toolbar>
        </el-row>

        <el-table v-loading="loading" :data="categoryList" @selection-change="handleSelectionChange" border>
            <el-table-column type="selection" width="55" align="center" />
            <el-table-column label="产品分类名称" align="center" prop="categoryName" />
            <el-table-column label="备注" align="left" prop="remark" min-width="150" />
            <el-table-column label="系统定义" align="center" prop="isSys">
                <template slot-scope="scope">
                    <dict-tag :options="dict.type.iot_yes_no" :value="scope.row.isSys" />
                </template>
            </el-table-column>
            <el-table-column label="显示顺序" align="center" prop="orderNum" />
            <el-table-column label="创建时间" align="center" prop="createTime" width="180">
                <template slot-scope="scope">
                    <span>{{ parseTime(scope.row.createTime, '{y}-{m}-{d}') }}</span>
                </template>
            </el-table-column>
            <el-table-column label="操作" align="center" class-name="small-padding fixed-width" width="150">
                <template slot-scope="scope">
                    <el-button size="small" type="primary" style="padding:5px;" icon="el-icon-edit" @click="handleUpdate(scope.row)" v-hasPermi="['iot:category:edit']">修改</el-button>
                    <el-button size="small" type="danger" style="padding:5px;" icon="el-icon-delete" @click="handleDelete(scope.row)" v-hasPermi="['iot:category:remove']" v-if =" canEdit ? true : (scope.row.isSys == '1' ? false : true)">删除</el-button>
                </template>
            </el-table-column>
        </el-table>

        <pagination v-show="total>0" :total="total" :page.sync="queryParams.pageNum" :limit.sync="queryParams.pageSize" @pagination="getList" />

        <!-- 添加或修改产品分类对话框 -->
        <el-dialog :title="title" :visible.sync="open" width="500px" append-to-body>
            <el-form ref="form" :model="form" :rules="rules" label-width="80px">
                <el-form-item label="分类名称" prop="categoryName">
                    <el-input v-model="form.categoryName" placeholder="请输入产品分类名称" />
                </el-form-item>
                <el-form-item label="显示顺序" prop="orderNum">
                    <el-input v-model="form.orderNum" type="number" placeholder="请输入显示顺序" />
                </el-form-item>
                <el-form-item label="备注" prop="remark">
                    <el-input v-model="form.remark" type="textarea" placeholder="请输入内容" />
                </el-form-item>
            </el-form>
            <div slot="footer" class="dialog-footer">
                <el-button type="primary" @click="submitForm" :disabled=" canEdit ? false : (form.isSys == '1' ? true : false)">确 定</el-button>
                <el-button @click="cancel">取 消</el-button>
            </div>
        </el-dialog>

    </el-card>
</div>
</template>

<script>
import {
    listCategory,
    getCategory,
    delCategory,
    addCategory,
    updateCategory
} from "@/api/iot/category";

export default {
    name: "Category",
    dicts: ["iot_yes_no"],
    data() {
        return {
            // 判断是否获取到修改权限
            canEdit:false,
            // 遮罩层
            loading: true,
            // 选中数组
            ids: [],
            // 非单个禁用
            single: true,
            // 非多个禁用
            multiple: true,
            // 显示搜索条件
            showSearch: true,
            // 总条数
            total: 0,
            // 产品分类表格数据
            categoryList: [],
            // 弹出层标题
            title: "",
            // 是否显示弹出层
            open: false,
            // 查询参数
            queryParams: {
                pageNum: 1,
                pageSize: 10,
                categoryName: null,
                tenantName: null,
                isSys: null,
            },
            // 表单参数
            form: {},
            // 表单校验
            rules: {
                categoryName: [{
                    required: true,
                    message: "产品分类名称不能为空",
                    trigger: "blur"
                }],
                tenantId: [{
                    required: true,
                    message: "租户ID不能为空",
                    trigger: "blur"
                }],
                tenantName: [{
                    required: true,
                    message: "租户名称不能为空",
                    trigger: "blur"
                }],
                isSys: [{
                    required: true,
                    message: "是否系统通用不能为空",
                    trigger: "blur"
                }],
            }
        };
    },
    created() {
        this.getList();
        this.init();
    },
    methods: {
      init(){
        if (this.$store.state.user.roles.indexOf("admin") !== -1){
          this.canEdit = true
        }
      },
        /** 查询产品分类列表 */
        getList() {
            this.loading = true;
            if (this.$store.state.user.roles.indexOf("admin") === -1){
              this.queryParams.tenantName = this.$store.state.user.name
            }
            listCategory(this.queryParams).then(response => {
                this.categoryList = response.rows;
                this.total = response.total;
                this.loading = false;
            });
        },
        // 取消按钮
        cancel() {
            this.open = false;
            this.reset();
        },
        // 表单重置
        reset() {
            this.form = {
                categoryId: null,
                categoryName: null,
                tenantId: null,
                tenantName: null,
                isSys: null,
                parentId: null,
                orderNum: null,
                delFlag: null,
                createBy: null,
                createTime: null,
                updateBy: null,
                updateTime: null,
                remark: null
            };
            this.resetForm("form");
        },
        /** 搜索按钮操作 */
        handleQuery() {
            this.queryParams.pageNum = 1;
            this.getList();
        },
        /** 重置按钮操作 */
        resetQuery() {
            this.resetForm("queryForm");
            this.handleQuery();
        },
        // 多选框选中数据
        handleSelectionChange(selection) {
            this.ids = selection.map(item => item.categoryId)
            this.single = selection.length !== 1
            this.multiple = !selection.length
        },
        /** 新增按钮操作 */
        handleAdd() {
            this.reset();
            this.open = true;
            this.title = "添加产品分类";
        },
        /** 修改按钮操作 */
        handleUpdate(row) {
            this.reset();
            const categoryId = row.categoryId || this.ids
            getCategory(categoryId).then(response => {
                this.form = response.data;
                this.open = true;
                this.title = "修改产品分类";
            });
        },
        /** 提交按钮 */
        submitForm() {
            this.$refs["form"].validate(valid => {
                if (valid) {
                    if (this.form.categoryId != null) {
                        updateCategory(this.form).then(response => {
                            this.$modal.msgSuccess("修改成功");
                            this.open = false;
                            this.getList();
                        });
                    } else {
                        addCategory(this.form).then(response => {
                            this.$modal.msgSuccess("新增成功");
                            this.open = false;
                            this.getList();
                        });
                    }
                }
            });
        },
        /** 删除按钮操作 */
        handleDelete(row) {
            const categoryIds = row.categoryId || this.ids;
            let msg="";
            this.$modal.confirm('是否确认删除产品分类编号为"' + categoryIds + '"的数据项？').then(function () {
                return delCategory(categoryIds).then(response => {
                    msg=response.msg;
                });
            }).then(() => {
                this.getList();
                this.$modal.msgSuccess(msg);
            }).catch(() => {});
        },
        /** 导出按钮操作 */
        handleExport() {
            this.download('iot/category/export', {
                ...this.queryParams
            }, `category_${new Date().getTime()}.xlsx`)
        }
    }
};
</script>
