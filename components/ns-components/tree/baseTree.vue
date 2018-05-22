<!--基础树-->
<template>
	<div id="baseTree">
		<!--树显示按钮-->
    <ns-icon-svg v-if="buttonShow" @click="treeshow" icon-class="hj" class="zhankai"></ns-icon-svg>
		<!--树区域-->
		<div v-if="treeisShow" class="tree-width house-tree" id="house_tree">
			<!--树搜索框-->
			<ns-icon-svg v-if="seek" @mouseup.native.stop="" icon-class="sousuo" class="tree-search-icon tree_node_icon"></ns-icon-svg>
			<el-select v-if="seek" size="small" v-model="treeInput" :clearable="true" @change="changeData" filterable remote placeholder="请输入关键词" class="tree-search" :remote-method="remoteMethod" :loading="loading" :loading-text="searchTip">
				<el-option v-for="(item,index) in options" :key="index" :label="item.label" :value="item.id">
				</el-option>
			</el-select>
			<div class="tree-line"></div>

			<!--树隐藏按钮-->
      <ns-icon-svg v-if="buttonisShow" @click="treeisHide" icon-class="shouqi1" class="shouqi"></ns-icon-svg>

			<!--树主体-->
			<div class="tree-body">
				<p class="treeTitle" ref="title" v-if="false">{{title}}</p>
				<sortable-tree v-loading="treeloading" element-loading-text="拼命加载中" :data="treeData" childrenAttr="children" mixinParentKey="$parent" @changePosition="changePosition" :dragEnable="true" closeStateKey="$foldClose">
					<!--树item-->
					<template slot-scope="{item}">
						<!--item内容-->
						<div class="custom-tree-content" :title="item.name" :class="{'exitChild': item.children && item.children.length,treeActive:selectEntityTypeId==item.id}">
							<!-- [+] -->
							<ns-icon-svg class="tree_node_icon" v-if="item['$foldClose'] && item.children && item.children.length || item.children && !item.children.length" @click="changeState(item)" icon-class="CombinedShapeCopy1"></ns-icon-svg>
							<!-- [-] -->
							<ns-icon-svg class="tree_node_icon" v-else-if="!item['$foldClose'] && item.children && item.children.length" icon-class="CombinedShape1" @click="changeState(item)" ></ns-icon-svg>
							<!-- [.] -->
							<ns-icon-svg class="tree_node_icon" v-else  icon-class="bushenbuzhan"></ns-icon-svg>

							<span class="tree-text" @click="handleClick(item)">{{item.name}}</span>
							<!-- <span class="tree-text" v-else :id="item.houseId" @click="handleClick($event,item)">{{item.name}}</span> -->
						</div>
						<!--item操作-->
						<el-dropdown class="fnsicon hide" trigger="click" :hide-on-click="true" v-if="showFunction">
							<span class="fnsicon_svg_span">
							<ns-icon-svg icon-class="gengduo" class="tree_node_icon" @click="operate"></ns-icon-svg>
						</span>
							<el-dropdown-menu slot="dropdown">
								<el-dropdown-item @click.native="fnclick(item,'edit')" v-if="!isTreeRoot(item)">
									编辑
								</el-dropdown-item>
								<el-dropdown-item :disabled="hasChildNode(item)" v-if="!isTreeRoot(item)">
									<el-popover placement="top" width="280" trigger="click" >
										<p>该节点删除后不可恢复，确定继续删除吗？</p>
										<div class="prompt-delete">
											<el-button class="delete-sure" type="primary" size="mini" @click.native="deleteDo(item)">确 定</el-button>
											<el-button class="delete-cancel" size="mini" @click="cancel">取 消</el-button>
										</div>
										<div slot="reference">删除</div>
									</el-popover>
								</el-dropdown-item>

								<template v-if="ChildNode.nodeShow">
									<el-dropdown-item>
										<p class="addChildre" style="line-height:36px;">新增子节点</p>
									</el-dropdown-item>
									<template>
										<el-dropdown-item v-for="(n,index) in ChildNode.Inode" :key="index" @click.native="fnclick(item,index)">
											<ns-icon-svg icon-class="dian-copy"></ns-icon-svg>{{n.name}}
										</el-dropdown-item>
									</template>
								</template>
							</el-dropdown-menu>
						</el-dropdown>
					</template>
				</sortable-tree>
			</div>
			<slot name="form"></slot>
		</div>
	</div>
</template>
<script>
	import SortableTree from './SortableTree.vue';
	export default {
		data() {
			return {
                // 树隐藏显示控制数据
                draggable:true,
                buttonShow: false,
				buttonisShow: true,
				treeisShow: true,
				//树的搜索
				treeInput:"",
				selectEntityTypeId:""
			}
		},
		props: {
			title:{
				type:String
			},
			treeData:{//树数据
                type:Object,
                default:{}
            },
            treeloading: {//加载动画显隐
                type:Boolean,
                default:true
            },
            showFunction: {//操作按钮显隐
                type:Boolean,
                default:true
			},
			ChildNode:{//操作按钮对应显示节点信息
				type:Object,
				default() {
					return {nodeShow:true,Inode:[]}
				} 
			},
			seek:{//搜索框的显隐
				type:Boolean,
				default:true
			},
			loading:{//搜索框加载显隐
				type:Boolean,
				default:false
			},
			searchTip:{//搜索框加载显示的文字
				type:String,
				default:"搜索数据中..."
			},
			options:{//select对应显示的数据
				type:Array,
				default() {
					return []
				}
			}
		},
		methods: {
			//=====================================触发方法===================================//
			//树显示隐藏
			treeisHide() { //隐藏
				this.treeisShow = false;
				this.buttonShow = true;
				this.buttonisShow = false

			},
			treeshow() { //显示
				this.treeisShow = true;
				this.buttonisShow = true;
				this.buttonShow = false;
			},
			changePosition(opts){//节点拖动触发函数
                this.$emit('changePosition', opts)
            },
            changeState (item) {//点击节点左侧图标触发函数
                this.$emit('changeState',item)  
            },
			handleClick(item){//点击节点触发函数
				this.selectEntityTypeId = item.id;
                this.$emit('handleClick',item);
            },
            isTreeRoot(item){//判断是否是跟节点
                if(item.id == 0){
                    return true;
                }else{
                    return false;
                }
            },
			hasChildNode(item){//判断有没有子节点
				if(item.children){
                    return true;
                }else{
                    return false;
                }
			},
			//删除节点
			deleteDo(item) {
				this.$refs.title.click()
				this.$emit('deleteDo', item)
			},
			//取消删除
			cancel() {
				this.$refs.title.click()
			},
			fnclick(item,index){//对节点进行操作时出发函数
				this.$emit('fnclick', item,index)
			},
			operate(){//点击操作图标时触发的函数
				this.$emit('operate')
			},
			changeData(val){//select值改变的时候触发的函数
				this.$emit('changeData',val)
			},
			remoteMethod(query){//搜索输入框值改变的时候触发的函数
				this.$emit('remoteMethod',query)
			},
			sortTree(treeData,order) {
			//根据树节点的编码对树节点进行排序
			var arr = treeData;
			//console.log(arr);
			var compare = function(obj1, obj2) {
				var val1 = obj1.order;
				var val2 = obj2.order;
				if (val1 < val2) {
				return -1;
				} else if (val1 > val2) {
				return 1;
				} else {
				return 0;
				}
			};
				arr.sort(compare);
				//console.log(arr.sort(compare));
				for (var a = 0; a < arr.length; a++) {
					//console.log(arr[a].children != null);
					if (arr[a].children != null) {this.sortTree(arr[a].children);}
				}
			}, 
		},
		components: {
			[SortableTree.name]: SortableTree
		}
	}

	//树上搜索框的一个基本数据格式，select根据label和id的值来展示和给value赋值
	// [{label:'3-1',id:0},{label:'3-2',id:1},{label:'3-3',id:2}]



	//树的基本的数据格式，id, name, children 是必须字段，其他根据业务需要自己随意添加，想使用树的懒加载，只需第一次返回children为一个空的数组即可，点击，节点再去发请求的时候，把请求的数据拼接给对应children就可以了。
	// {
    //                 id:0,
	// 				   name: 'root',
    //                 children: [
    //                 {
    //                     id:1,
    //                     name: '2',
    //                     children: [
    //                     {id:2, name: '2-1' },
    //                     {
    //                         id:3,
    //                         name: '2-2',
    //                         children: []
    //                     }
    //                     ]
    //                 },
    //                 {
    //                     id:4,
    //                     name: '3',
    //                     children: [
    //                     {id:5, name: '3-1' },
    //                     {id:6,name: '3-2' }
    //                     ]
    //                 }
    //                 ]
	// 			}


	//这里是开放出来的表单插槽，可以自己定义自己想要的任何表单
	// <slot name="form"></slot>
</script>
<style>
	@import url("../../../assets/css/Modular/tree/tree.css");
</style>
<style rel="stylesheet/less" lang="scss">
#baseTree{
	.el-popover {
		border-radius: 3px;
		color: #333333;
		border: 1px solid #dadada;
		box-shadow: 0 0 4px 0 rgba(0, 0, 0, 0.22) !important;
		border-radius: 4px 4px 4px 0 0 0 4px;
	}
	.treeActive{
		background: #ccc;
	}
}
	
</style>
