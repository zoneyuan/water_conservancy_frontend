<template>
  <a-layout>
    <a-layout-content style="padding: 0 50px">

      <a-breadcrumb style="margin: 16px 0">
        <a-breadcrumb-item>数据展示</a-breadcrumb-item>
        <a-breadcrumb-item>当前数据：{{title}}</a-breadcrumb-item>
      </a-breadcrumb>

      <a-layout style="padding: 24px 0; background: #fff">
        <a-layout-content :style="{ padding: '0 24px', minHeight: '420px' }">
          <div id="cesiumContainer"></div>
        </a-layout-content>
      </a-layout>

    </a-layout-content>
  </a-layout>
</template>

<script>
import { defineComponent, onMounted } from "vue";
import { useRoute } from "vue-router";
import { useStore } from "vuex";
import axios from "axios";
import initCesium from "assets/js/initCesium";
import * as Cesium from "cesium";

export default defineComponent({
  name: "three",
  setup() {
    const route = useRoute();
    const store = useStore();

    const title = route.query.title.split('.')[0];
    const type = route.query.url;
    const url = route.query.url;

    onMounted(() => {
      const viewer = initCesium(
        Cesium,
        store.state.cesiumToken,
        store.state.mapboxToken,
        "cesiumContainer"
      );

      // if (url) {  // 根据路由传过来的类型，进行不同的加载方式
      //   switch (type) {
      //     case "Vector":         // 矢量数据 GeoJson
      //       axios({
      //         methods: "GET",
      //         url: url,
      //       }).then((res) => {
      //           const dataSource = Cesium.GeoJsonDataSource.load(res.data);
      //           viewer.dataSources.add(dataSource);
      //           viewer.flyTo(dataSource, 1.0);
      //         }).catch((error) => {
      //           console.log("error: ", error);
      //         });
      //       break;
      //     case "Model":          // 三维模型 3D Tiles
      //       const tileset = viewer.scene.primitives.add(
      //         new Cesium.Cesium3DTileset({
      //           url: "http://localhost:4000/geodata/3dtiles/zhanlangguan/tileset.json",
      //         })
      //       );
      //       tileset.readyPromise.then(function (tileset) {
      //         viewer.camera.viewBoundingSphere(
      //           tileset.boundingSphere,
      //           new Cesium.HeadingPitchRange(0, -0.5, 0)
      //         );
      //       });
      //       break;
      //     default:               // DEM、DOM  影像 
      //       const provider = new Cesium.WebMapServiceImageryProvider({
      //         url: route.params.serviceUrl,
      //         layers: route.params.serviceLayerName,
      //         parameters: {
      //           service: "WMS",
      //           version: "1.3.0",
      //           request: "GetMap",
      //           transparent: true,
      //           format: "image/png",
      //           // crs: "EPSG:3857",
      //           // bbox: "13037099.5443,3406033.9804,13054221.4387,3421321.386",
      //         },
      //       });
      //       const imageryLayer = viewer.imageryLayers.addImageryProvider(provider);
      //       viewer.camera.flyTo({
      //         destination: Cesium.Cartesian3.fromDegrees(
      //           117.1882,
      //           29.1838,
      //           40000
      //         ),
      //         orientation: {
      //           heading: Cesium.Math.toRadians(0.0),
      //           pitch: Cesium.Math.toRadians(-75.0),
      //         },
      //       });
      //       break;
      //   }
      // }
    
    });


    return {
      title
    };

  },
});
</script>

<style scoped>
#cesiumContainer {
  min-height: 700px;
  justify-content: center;
  position: relative;
}
</style>
