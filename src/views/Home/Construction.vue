<template>

  <a-layout>
    <a-layout-content style="padding: 0 50px">

      <a-breadcrumb style="margin: 16px 0">
        <a-breadcrumb-item>首页</a-breadcrumb-item>
        <a-breadcrumb-item>工程项目</a-breadcrumb-item>
      </a-breadcrumb>
    
      <a-layout style="padding: 24px 0; background: #fff">
        <a-layout-content :style="{ padding: '0 24px', minHeight: '420px' }">
          <a-spin :spinning="spinning" size="large" tip="数据加载中...">
            
            <!-- 搜索 -->
            <a-input-search
              v-model:value="value"
              placeholder="请输入工程名进行搜索"
              enter-button="搜索"
              size="large"
              @search="onSearch"
            />

            <!-- 数据列表 -->
            <a-list item-layout="vertical" size="large" :pagination="pagination" :data-source="listData">
              <template #renderItem="{ item }">  
                <a-list-item key="item.title">
                  <!-- 图片 -->
                  <template #extra>
                    <img
                      width="272"
                      alt="logo"
                      src="https://gw.alipayobjects.com/zos/rmsportal/mqaQswcyDLcXyDKnZfES.png"
                    />
                  </template>
                  <!-- 数据 -->
                  <a-list-item-meta :description="item.description">
                    <template #title>
                      <a class="link" @click="detailProject(item)">{{ item.title }}</a>
                    </template>
                    <template #avatar><a-avatar :src="item.avatar" /></template>
                  </a-list-item-meta>
                  <!-- 项目数据列表 -->
                  <ul>
                    <li v-for="(j,index) in item.content" :key="index">
                      <a class="link" @click="detail(j)">{{j.data_name}}</a>
                    </li>
                  </ul>
                </a-list-item>
              </template>
            </a-list>

          </a-spin>
        </a-layout-content>
      </a-layout>

    </a-layout-content>
  </a-layout>

  <!-- 数据详情弹出框 -->
  <a-modal v-model:visible="visibleProject" title="项目详情" @ok="handleOk" width='50%'>
    <p> <b>标题</b> ：{{currentProject.name}}</p>
    <p> <b>时间</b> ：{{currentProject.time}}</p>
    <p> <b>项目描述</b> ：{{currentProject.description}}</p>
    <p> <b>数据数量</b> ：{{currentProject.count}}</p>
  </a-modal>

  <!-- 数据详情弹出框 -->
  <a-modal v-model:visible="visible" title="数据详情" @ok="handleOk" width='50%'>
    <p> <b>标题</b> ：{{currentData.name}}</p>
    <p> <b>类型</b> ：{{currentData.type}}</p>
    <p> <b>时间</b> ：{{currentData.time}}</p>
    <p> <b>比例尺</b> ：{{currentData.scale}}</p>
    <p> <b>坐标</b> ：{{currentData.coord}}</p>
    <p> <b>url</b> ：{{currentData.url}}</p>
    <p> <b>图层</b> ：{{currentData.layerName}}</p>
  </a-modal>


</template>



<script lang="ts">
import { defineComponent, reactive, ref } from "vue"
import { useStore } from "vuex";
import axios from "axios";
import Cookies from "js-cookie";
import { StarOutlined, LikeOutlined, MessageOutlined } from '@ant-design/icons-vue';


export default defineComponent({
    components: {
    StarOutlined,
    LikeOutlined,
    MessageOutlined,
  },
  setup() { 
    const store = useStore();

    const spinning = ref(false);    // 查找数据量过大数据暂时未显示出来时 spinning变为true
    const currentPage = ref(1);
    const pageSize = ref(10);

    const visibleProject = ref(false)
    const currentProject = reactive({
      name: '',
      time: '',
      description: '',
      count: ''
    })

    const visible = ref(false);     // 数据详情对话框
    const currentData = reactive({  // 当前数据详情
      name: '',
      type: '',
      time: '',
      scale: '',
      coord: '',
      url: '',
      layerName: ''
    });

    //搜索框文本的值
    const value = ref('');
    const listData = reactive([])   // 项目信息 
    const { loading, dataList, totalItem, onSearch } = useSearchData(store, value , listData); // 用户搜索


    //根据用户类别控制是否展示“下载”按钮
    const allowFlag = Cookies.get("usertype") === "administrator";

    const pagination = {
      onChange: (page: number) => {
        console.log(page);
      },
      pageSize: 3,
    };


    // 项目详情
    const detailProject = (item) => {
      visibleProject.value = true;
      currentProject.name = item.title;
      currentProject.time = item.time;
      currentProject.description = item.description;
      currentProject.count = item.count;
    }
    // 数据详情
    const detail = (j) => {
      visible.value = true;
      currentData.name = j.data_name;
      currentData.type = j.data_type;
      currentData.time = j.data_time;
      currentData.scale = j.data_scale;
      currentData.coord = j.data_coord;
      currentData.url = j.data_url;
      currentData.layerName = j.data_layer_name;
    };
    const handleOk = (e) => {
      console.log(e);
      visible.value = false;
      visibleProject.value = false;
    }

    return {
      spinning,
      currentPage,
      pageSize,
      visibleProject,
      currentProject,
      visible,
      currentData,

      value,
      listData,
      loading, 
      dataList, 
      totalItem, 
      onSearch,

      allowFlag,
      pagination,

      detailProject,
      detail,
      handleOk
    }
  },

});


function useSearchData(store, value, listData) {
  const loading = ref(false);  // 数据加载图标
  const dataList = reactive({  // 数据列表，初始化为空
    data: store.state.data,
  });
  const totalItem = ref();     // 数据总数目

  // 搜索，得到数据列表 dataList
  const onSearch = async (searchValue) => {
    loading.value = true;
    value.value = "";
    store.state.data = [];
    
    // 1、查询项目信息
    const url = store.state.baseUrl + "/projectInfo";
    const params = {
      params: {
        name: searchValue
      }
    }
    const res = await axios.get(url, params);
    if(res.data.length) {  // 查到项目
      var project_id = []
      var projectInfo = res.data;
      projectInfo.forEach(i => {
        project_id.push(i['project_id'])
      });
      
      // 2、根据项目id查询相应数据信息
      const url2 = store.state.baseUrl + '/dataInfoByProject'
      var params2 = {
        params: {
        }
      }
      project_id.forEach(i => {
        params2['params']['id'+i] = i
      })
      const res2 = await axios.get(url2, params2);
      if(res2.data.length) {
        res2.data.forEach( i => {
          projectInfo.forEach( j => {
            if(i['project_id']==j['project_id']) {
              j['project_data'] += JSON.stringify(i)
              j['project_data'] += ','
            }
          })
        });
      }
      // 将项目存入仓库中
      store.state.data = res.data.map((item) => {
        return {
          project_id: item.project_id,
          project_name: item.project_name,
          project_time: item.project_time,
          project_description: item.project_description,
          project_data: item.project_data
        }
      })
      console.log('项目数量：' + store.state.data.length);

      dataList.data = store.state.data;
      totalItem.value = res.data.length;
      loading.value = false;

      listData.splice(0)   // 清空项目数据

      for (let i = 0; i < store.state.data.length; i++) {
        // 项目中数据
        var projectData = []
        var tempData = store.state.data[i].project_data.split('},')
        tempData.pop()
        tempData.forEach(j => {
          j = j.concat('}')
          projectData.push(JSON.parse(j))
        })

        // 项目信息
        listData.push({
          href: '#',
          title: store.state.data[i].project_name,
          time: store.state.data[i].project_time,
          avatar: 'https://joeschmoe.io/api/v1/random',
          description: store.state.data[i].project_description,
          content: projectData,
          count: projectData.length
        });
     }

    } else {  // 没查到 
      store.state.data = [];
      dataList.data = store.state.data;
      totalItem.value = 0;
      loading.value = false;
    }
  };
  return { loading, dataList, totalItem, onSearch };
}



</script>





<style>

</style>