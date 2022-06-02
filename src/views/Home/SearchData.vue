<template>
  <a-form>
    <a-layout>
      <a-layout-content style="padding: 0 50px">
        <a-breadcrumb style="margin: 16px 0">
          <a-breadcrumb-item>首页</a-breadcrumb-item>
          <a-breadcrumb-item>区域检索</a-breadcrumb-item>
        </a-breadcrumb>
        <a-layout style="padding: 24px 0; background: #fff">
          <a-layout-content :style="{ padding: '0 24px', minHeight: '420px' }">
            <a-spin :spinning="spinning" size="large" tip="数据加载中...">
               <a-row>
              <a-form-item  style="width:10%" v-bind="validateInfos.searchtype">
                <a-select                  
                  v-model:value="modelRef.searchtype" placeholder="请选择查询方式"
                >
                  <a-select-option value="reigon">按地名查询</a-select-option>
                  <a-select-option value="latlon">按经纬度查询</a-select-option>
                </a-select>
              </a-form-item>
              <p> &nbsp;地名：</p>
              <a-form-item style="width:10%" v-bind="validateInfos.reigonname">
                <a-input  v-model:value="modelRef.reigonname" placeholder="请输入所在地名"> </a-input>
              </a-form-item>
<!--               <a-form-item  style="width:10%" v-bind="validateInfos.city">
                <a-select
                  v-model:value="modelRef.city"
                >
                 
                </a-select>
              </a-form-item>
              <a-form-item  style="width:10%" v-bind="validateInfos.county">
                <a-select                  
                  v-model:value="modelRef.county"
                >
                 
                </a-select>
              </a-form-item> -->
              <p> &nbsp;经度：</p>
              <a-form-item style="width:10%" v-bind="validateInfos.lon0">
                <a-input  v-model:value="modelRef.lon0" placeholder="请输入位置"> </a-input>
              </a-form-item>
              {{ " - " }}
              <a-form-item style="width:10%" v-bind="validateInfos.lon1">
                <a-input  v-model:value="modelRef.lon1"> </a-input>
              </a-form-item>
              <p> &nbsp;纬度：</p>
              <a-form-item style="width:10%" v-bind="validateInfos.lat0">
                <a-input  v-model:value="modelRef.lat0"> </a-input>
              </a-form-item>
              {{ " - " }}
              <a-form-item style="width:10%" v-bind="validateInfos.lat1">
                <a-input  v-model:value="modelRef.lat1"> </a-input>
              </a-form-item>
              <p> &nbsp;&nbsp;&nbsp;</p>
              <a-form-item>
                <a-button
                  
                  type="primary"
                  @click.prevent="onSearch"
                  >检索</a-button
                >
              </a-form-item>
               </a-row>

              <a-list
                class="geo-data-list"
                :loading="loading"
                item-layout="horizontal"
                :data-source="dataList.data.slice(slice.start, slice.end)"
              >
                <template #renderItem="{ item }">
                  <a-list-item>
                    <template #actions>
                      <a class="link" @click="detail(item)">数据详情</a>
                      <a class="link" @click="naviToMap(item)">在线展示</a>
                      <a
                        v-if="allowFlag"
                        class="link"
                        :dataTitle="item.title"
                        @click="download"
                        >下载</a
                      >
                    </template>
                    <a-list-item-meta
                      :description="item.coord + '; ' + item.time"
                    >
                      <template #title>
                        <a>{{ item.title}}</a>
                       <!--  .split(".")[0]  -->
                      </template>
                      <template #avatar>
                        <a-avatar
                          shape="square"
                          size="large"
                          :src="item.imgSrc"
                        />
                      </template>
                    </a-list-item-meta>
                  </a-list-item>
                </template>
              </a-list>

              <a-pagination
                v-model:current="currentPage"
                v-model:pageSize="pageSize"
                :total="totalItem"
                :show-total="(total) => `共 ${total} 条`"
                show-size-changer
                @showSizeChange="onShowSizeChange"
                show-quick-jumper
                @change="onChange"
              />
            </a-spin>
          </a-layout-content>
        </a-layout>
      </a-layout-content>
    </a-layout>
  </a-form>
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
import { defineComponent, reactive, toRaw } from "vue"
import { Form, message } from "ant-design-vue"
import { ref,  computed, watch, h } from "vue";
import { Modal} from "ant-design-vue";
import { DownloadOutlined } from "@ant-design/icons-vue";
import IconFont from "assets/js/iconfont";
// import导入组件时不加 { }
import AntdSubMenu from "@/components/AntdSubMenu.vue";
import { useStore } from "vuex";
import { useRouter } from "vue-router";
import axios from "axios";
import Cookies from "js-cookie";
const useForm = Form.useForm

export default defineComponent({
  components: {
    AntdSubMenu,
  },
  setup() {
    const store = useStore();
    const router = useRouter();
    const spinning = ref(false);
    const currentPage = ref(1);
    const pageSize = ref(10);
    const modelRef = reactive({
      searchtype: "",
      reigonname:"",
      lat0:"",
      lat1:"",
      lon0:"",
      lon1:""
    })
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
    //输入框文本值
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
     const rulesRef = reactive({
      searchtype: [
        {
          required: true,
          message: "",
        },
      ],
      reigonname: [
        {
          required: true,
          message: "",
        },
      ],
      lat0: [
        {
          required: true,
          message: "",
        },
      ],
      lat1: [
        {
          required: true,
          message: "",
        },
      ],
      lon0: [
        {
          required: true,
          message: "",
        },
      ],
      lon1: [
        {
          required: true,
          message: "",
        },
      ],
    })
     const options = toRaw(modelRef)
    const { resetFields, validate, validateInfos } = useForm(modelRef, rulesRef)

    const filesArr = ["089057.e00.dwg", "3537.5-575.0.zip", "chuanzha.rvt"];

    let index = 0;

    const onShowSizeChange = (current, size) => {};

    const onChange = () => {};

    watch(currentPage, (newVal, oldValue) => {});

    watch(pageSize, (newVal, oldValue) => {});

    /* const openKeys = computed(() => {
      return Array.from(menuParams, (item) => item.name);
    }); */
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
    const naviToMap = async (item) => {
      if (item.title.split(".")[1] === "dwg") {
        spinning.value = true;
        const res = await axios({
          method: "post",
          url: store.state.baseUrl + "/api/forge",
          data: {
            fileName: filesArr[index++ % 3],
          },
        });
        spinning.value = false;
        if (res.status === 200) {
          router
            .push({
              name: "cadviewer",
              params: {
                access_token: res.data.access_token,
                urn: res.data.urn,
              },
            })
            .catch((err) => {
              console.log(err);
            });
        }
      } else {
        router
          .push({
            name: "three",
            params: {
              serviceUrl: item.url,
              serviceLayerName: item.layerName,
              serviceType: item.type,
            },
          })
          .catch((err) => {
            console.log(err);
          });
      }
    };

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

    /*  const { menuParams, options, optionsArr, addFilter } = useAddFilter(); */
    const { loading, dataList, totalItem, onSearch } = useSearchData(
      store,
      value,
      options
    );

    return {
      spinning,
      currentPage,
      pageSize,
      modelRef,
      visible,
      currentData,
      value,
      allowFlag,
      slice,
      onShowSizeChange,
      onChange,
      /* openKeys, */
      detail,
      handleOk,
      naviToMap,
      download,
      validateInfos,
      resetFields,
      /*  menuParams,
      options,
      optionsArr,
      addFilter, */
      loading,
      dataList,
      totalItem,
      onSearch,
    };
  },
});

/* function useAddFilter() {
  const menuParams = reactive([
    {
      name: "sub1",
      title: "数据集类型",
      types: [
        { type: "DEM", id: 1 },
        { type: "DOM", id: 2 },
        { type: "Vector", id: 3 },
        { type: "Model", id: 4 },
      ],
    },
    {
      name: "sub2",
      title: "时间",
      types: [
        { type: "2020年以后", id: 5 },
        { type: "2019年", id: 6 },
        { type: "2017年", id: 7 },
        { type: "2017年以前", id: 8 },
      ],
    },
    {
      name: "sub3",
      title: "比例尺",
      types: [
        { type: "1:10000", id: 9 },
        { type: "1:5000", id: 10 },
        { type: "1:2000", id: 11 },
        { type: "> 1:2000", id: 12 },
      ],
    },
  ]);

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

  const addFilter = (filter, key) => {
    let existIndex;
    switch (key) {
      case "sub1":
        existIndex = options.option.types.findIndex(
          (value) => value === filter
        );
        if (existIndex === -1) {
          options.option.types.push(filter);
        } else {
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
        existIndex = options.option.scale.findIndex(
          (value) => value === filter
        );
        if (existIndex === -1) {
          options.option.scale.push(filter);
        } else {
          options.option.scale.splice(existIndex, 1);
        }
        break;
    }
  };

  return { menuParams, options, optionsArr, addFilter };
} */

function useSearchData(store, value, options) {
  const loading = ref(false);

  const dataList = reactive({
    data: store.state.data,
  });

  const totalItem = ref();

  const onSearch = async () => {
    loading.value = true;
    value.value = "";
    store.state.data = [];

    const url = store.state.baseUrl + "/search";
    const params = {
      params: {
        type: options.searchtype,
        reigonnamea: options.reigonname,
        lat0a: options.lat0,
        lat1a: options.lat1,
        lon0a: options.lon0,
        lon1a: options.lon1,
      },
    };

    const res = await axios.get(url, params);
    if (res.data.length) {
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
    } else {
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
