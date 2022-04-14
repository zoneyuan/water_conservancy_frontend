<template>
  <a-layout>
    <a-layout-content style="padding: 0 50px">

      <a-breadcrumb style="margin: 16px 0">
        <a-breadcrumb-item>首页</a-breadcrumb-item>
        <a-breadcrumb-item>地理数据编目</a-breadcrumb-item>
      </a-breadcrumb>

      <a-layout style="padding: 24px 0; background: #fff">

        <!-- 左侧 -->
        <a-layout-sider width="200" style="background: #fff">
          <a-menu mode="inline" :openKeys="openKeys" :multiple="true" style="height: 100%">
            <!-- 数据集类型 -->
            <antd-sub-menu :menuParams="menuParams[0]" @addFilter="addFilter">
              <template #icon>
                <svg class="icon" aria-hidden="true">
                  <use xlink:href="#icon-leixing"></use>
                </svg>
              </template>
            </antd-sub-menu>

            <!-- 时间 -->
            <antd-sub-menu :menuParams="menuParams[1]" @addFilter="addFilter">
              <template #icon>
                <svg class="icon" aria-hidden="true">
                  <use xlink:href="#icon-riqi"></use>
                </svg>
              </template>
            </antd-sub-menu>

            <!-- 比例尺 -->
            <antd-sub-menu :menuParams="menuParams[2]" @addFilter="addFilter">
              <template #icon>
                <svg class="icon" aria-hidden="true">
                  <use xlink:href="#icon-biaochi"></use>
                </svg>
              </template>
            </antd-sub-menu>

          </a-menu>
        </a-layout-sider>

        <!-- 右侧 -->
        <a-layout-content :style="{ padding: '0 24px', minHeight: '420px' }">
          <a-spin :spinning="spinning" size="large" tip="数据加载中...">

            <!-- 搜索 -->
            <a-input-search v-model:value="value"  placeholder="请输入关键词进行搜索" enter-button="搜索" 
              size="large" @search="onSearch"/>

            <!-- 已选区域 -->
            <a-select mode="tags" placeholder="请选择" v-model:value="optionsArr" 
              style="width: 100%" :open="false" :bordered="false" disabled >
            </a-select>

            <!-- 数据列表 -->
            <a-list class="geo-data-list" :loading="loading" item-layout="horizontal"
              :data-source="dataList.data.slice(slice.start, slice.end)" >
              <template #renderItem="{ item }">
                <a-list-item>    

                  <!-- 数据详情和下载 -->
                  <template #actions>
                    <a class="link" @click="detail(item)">数据详情</a>
                    <!-- <a class="link" @click="naviToMap(item)">三维展示</a> -->
               <!--      <a class="link" @click="display(item)">在线展示</a> -->
                    <a class="link" @click="naviToMap(item)">在线展示</a>
                    <a v-if="allowFlag" class="link" :dataTitle="item.title" @click="download" >下载</a >
                  </template>

                  <!-- 数据坐标、时间显示 -->
                  <a-list-item-meta :description="item.coord + '; ' + item.time" >
                    <template #title>
                      <a>{{ item.title.split(".")[0] }}</a>
                    </template>
                    <template #avatar>
                      <a-avatar shape="square" size="large" :src="item.imgSrc" />
                    </template>
                  </a-list-item-meta>

                </a-list-item>
              </template>
            </a-list>

            <!-- 分页 -->
            <a-pagination
              v-model:current="currentPage"
              v-model:pageSize="pageSize"
              :total="totalItem"
              :show-total="(total) => `共 ${total} 条`"
              show-size-changer
              @showSizeChange="onShowSizeChange"
              show-quick-jumper
              @change="onChange" />

          </a-spin>
        </a-layout-content>
      </a-layout>


    </a-layout-content>
  </a-layout>


  <!-- 数据详情弹出框 -->
  <a-modal v-model:visible="visible" title="数据详情" @ok="handleOk" width='50%'>
    <p> <b>标题</b> ：{{currentData.title}}</p>
    <p> <b>类型</b> ：{{currentData.type}}</p>
    <p> <b>时间</b> ：{{currentData.time}}</p>
    <p> <b>比例尺</b> ：{{currentData.scale}}</p>
    <p> <b>坐标</b> ：{{currentData.coord}}</p>
    <p> <b>url</b> ：{{currentData.url}}</p>
    <p> <b>图层</b> ：{{currentData.layerName}}</p>
  </a-modal>


</template>

<script>
import { defineComponent, ref, reactive, computed, watch, h } from "vue";
import { Modal, message } from "ant-design-vue";
import { DownloadOutlined } from "@ant-design/icons-vue";

// import导入组件时不加 { }
import AntdSubMenu from "@/components/AntdSubMenu.vue";

import { useStore } from "vuex";
import { useRouter } from "vue-router";

import axios from "axios";
import Cookies from "js-cookie";

export default defineComponent({
  components: {
    AntdSubMenu,
  },
  setup() {
    const store = useStore();
    const router = useRouter();

    const spinning = ref(false);    // 查找数据量过大数据暂时未显示出来时 spinning变为true
    const currentPage = ref(1);
    const pageSize = ref(10);
    const visible = ref(false);     // 数据详情对话框
    const currentData = reactive({  // 当前数据详情
      title: '',
      type: '',
      time: '',
      scale: '',
      coord: '',
      url: '',
      layerName: ''
    });

    //搜索框文本的值
    const value = ref("");

    //根据用户类别控制是否展示“下载”按钮
    const allowFlag = Cookies.get("usertype") === "administrator";

    //截取返回数据中当前分页范围内的列表项
    const slice = computed(() => {
      return {
        start: (currentPage.value - 1) * pageSize.value,
        end: (currentPage.value - 1) * pageSize.value + pageSize.value,
      };
    });

    const filesArr = ["089057.e00.dwg", "3537.5-575.0.zip", "chuanzha.rvt"];

    let index = 0;

    const onShowSizeChange = (current, size) => {};
    const onChange = () => {};

    watch(currentPage, (newVal, oldValue) => {});
    watch(pageSize, (newVal, oldValue) => {});

    const { menuParams, options, optionsArr, addFilter } = useAddFilter(); // 用户添加过滤、取消过滤
    const { loading, dataList, totalItem, onSearch } = useSearchData( store, value, options); // 用户搜索
 
    const openKeys = computed(() => {
      return Array.from(menuParams, (item) => item.name);
    });

    //数据详情
    const detail = (item) => {
      visible.value = true;
      currentData.title = item.title;
      currentData.type = item.type;
      currentData.time = item.time.slice(5);
      currentData.scale = item.scale;
      currentData.coord = item.coord;
      currentData.url = item.url;
      currentData.layerName = item.layerName;
    };
    const handleOk = (e) => {
      console.log(e);
      visible.value = false;
    }

    // 在线展示
    const display = (item) => {
      const routeUrl = router.resolve({
        path: '/display',
        query: {
          title: item.title,
          type: item.type,
          url: item.url
        }
      });
      window.open(routeUrl.href, '_blank');
    };

    // 三维展示
    const naviToMap = async (item) => {
      if (item.title.split(".")[1] === "dwg") {   // .dwg文件
        spinning.value = true;      // 开始请求
        const res = await axios({
          method: "post",
          url: store.state.baseUrl + "/api/forge",     // http://localhost:4000//api/forge
          data: { fileName: filesArr[index++ % 3], },
        });
        spinning.value = false;
        if (res.status === 200) {
          router.push({
              name: "cadviewer",
              params: {
                access_token: res.data.access_token,
                urn: res.data.urn,
              },
            }).catch((err) => {
              console.log(err);
            });
        }
      } else {   
        router.push({
          name: "three",
          params: {
            serviceUrl: item.url,
            serviceLayerName: item.layerName,
            serviceType: item.type,
          },
        }).catch((err) => {
          console.log(err);
        });
      }
    };

    // 下载
    const download = (event) => {
      const dataTitle = event.target.getAttribute("dataTitle");
      Modal.confirm({
        title: "确认下载?",
        icon: h(DownloadOutlined),
        centered: true,
        content: h("p", [
          h("span", "您选择的数据集是: "),
          h("b", dataTitle),
          h("span", "，是否确认下载？"),
        ]),
        onOk() {
          return new Promise((resolve, reject) => {
            const url = store.state.baseUrl + "/download";
            const params = {
              fileName: dataTitle,
            };
            axios
              .post(url, params)
              .then((res) => {
                if (res.status === 200) {
                  message.success("开始下载!");
                  location.href = `${url}?fileName=${dataTitle}`;
                } else {
                  message.error("文件未找到!");
                }
                resolve();
              })
              .catch((error) => {
                message.error("请求出错，请重试!");
              });
          });
        },
        onCancel() {},
      });
    };

    return {
      spinning,
      currentPage,
      pageSize,
      visible,
      currentData,
      value,
      allowFlag,
      slice,
      
      onShowSizeChange,
      onChange,

      menuParams,
      options,
      optionsArr,
      addFilter,

      loading,
      dataList,
      totalItem,
      onSearch,

      openKeys,

      detail,
      handleOk,
      display, 
      naviToMap,
      download,

    };
  },
});

// 完成添加过滤、取消过滤
function useAddFilter() {
  const menuParams = reactive([
    {
      name: "sub1",
      title: "数据集类型",
      types: [
        { type: "DEM", id: 1 },
        { type: "DOM", id: 2 },
        { type: "Vector", id: 3 },
        { type: "倾斜模型", id: 4 },
        { type: "BIM", id: 5 },
      ],
    },
    {
      name: "sub2",
      title: "时间",
      types: [
        { type: "2020年以后", id: 5 },
        { type: "2018年 - 2019年", id: 6 },
        { type: "2017年 - 2018年", id: 7 },
        { type: "2017年以前", id: 8 },
      ],
    },
    {
      name: "sub3",
      title: "比例尺",
      types: [
        { type: "< 1:10000", id: 9 },
        { type: "1:10000 - 1:5000", id: 10 },
        { type: "1:5000 - 1:2000", id: 11 },
        { type: "> 1:2000", id: 12 },
      ],
    },
  ]);

  // 已选
  const options = reactive({
    option: {
      types: [],
      date: [],
      scale: [],
    },
  });


  const optionsArr = computed(() => {
    return options.option.types.concat(
      options.option.date,
      options.option.scale
    );
  });

  // 实时更新options数组
  const addFilter = (filter, key) => {
    let existIndex;
    switch (key) {
      case "sub1":
        // 判断options数组中是否已有filter
        existIndex = options.option.types.findIndex( (value) => value === filter);
        if (existIndex === -1) {  // 没有就添加
          options.option.types.push(filter); 
        } else { // 有就取消选择
          options.option.types.splice(existIndex, 1);
        }
        break;
      case "sub2":
        existIndex = options.option.date.findIndex((value) => value === filter);
        if (existIndex === -1) {
          options.option.date.push(filter);
        } else {
          options.option.date.splice(existIndex, 1);
        }
        break;
      case "sub3":
        existIndex = options.option.scale.findIndex( (value) => value === filter);
        if (existIndex === -1) {
          options.option.scale.push(filter);
        } else {
          options.option.scale.splice(existIndex, 1);
        }
        break;
    }
  };

  return { menuParams, options, optionsArr, addFilter };
}

// 搜索
function useSearchData(store, value, options) {
  const loading = ref(false);  // 数据加载图标
  const dataList = reactive({  // 数据列表，初始化为空
    data: store.state.data,
  });
  const totalItem = ref();     // 数据总数目

  // 搜索，得到数据列表 dataList
  const onSearch = async (searchValue) => {
    loading.value = true;
    value.value = "";
    store.state.data = [];  // 清空

    const url = store.state.baseUrl + "/data";
    const params = {
      params: {
        // 输入搜索关键字
        name: searchValue,
        // 选择的类型
        typesFilter: options.option.types,
        dateFilter: options.option.date,
        scaleFilter: options.option.scale,
      },
    };

    const res = await axios.get(url, params);  // http://localhost:4000/data
    if (res.data.length) {  // 查询后存入仓库 store.state.data 中
      store.state.data = res.data.map((item) => {
        return {
          title: item.data_name,
          type: item.data_type,
          time: `更新时间:${item.data_time}`,
          scale: item.data_scale,
          coord: item.data_coord,
          url: item.data_url,
          // 根据数据类型确定显示图片
          imgSrc: store.state.imgSrc.find((img) => img.name === item.data_type).src, // xxx.src
          layerName: item.data_layer_name,
        };
      }); 
      dataList.data = store.state.data;
      totalItem.value = res.data.length;
      loading.value = false;
    } else {  // 没查到
      store.state.data = [];
      dataList.data = store.state.data;
      totalItem.value = 0;
      loading.value = false;
    }
  };

  return { loading, totalItem, dataList, onSearch };
}

</script>



<style scoped>
.ant-list :deep(a.link) {
  color: #40a9ff;
}
</style>
