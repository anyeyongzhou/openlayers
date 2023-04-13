<template>
  <div id="map" class="map_container" ref="map_ref"></div>
</template>

<script setup lang="ts">
import { ref, onMounted, nextTick } from "vue"
import { Map, View, Feature } from "ol" // 地图实例方法、视图方法
import { Tile as TileLayer, Vector as VectorLayer } from "ol/layer" // 瓦片渲染方法,矢量图层方法
import { Vector, XYZ } from "ol/source" //矢量数据源
import { Point, LineString } from "ol/geom" //几何点、线
import * as control from "ol/control" //视图控件
import { Style, Circle, Fill, Stroke, Text, Icon } from "ol/style.js" //设置样式：圆，填充，描边，文本,图片
import { GeoJSON, MVT } from "ol/format" //格式化数据,数据格式有GeoJSON,MVT等
import gif from "../public/gif_1.gif"
//地图投影的相关的方法，
//get：获取指定代码的 Projection 对象；
//toLonLat：将坐标转换为经度/纬度；
//当投影EPSG:3857时选择fromLonLat方法能将坐标从经度/纬度转换为其他投影;
//transform将坐标从源投影转换为目标投影;等等，具体见文档
import {
  get as getProjection,
  toLonLat,
  fromLonLat,
  transform
} from "ol/proj.js"
import * as interaction from "ol/interaction" //地图交互功能
import Overlay from "ol/Overlay" //覆盖物
import {
  toStringHDMS,
  add,
  createStringXY,
  format,
  rotate,
  toStringXY
} from "ol/coordinate" //对经纬度坐标进行处理：toStringHDMS将坐标抓换为经纬度

const map = ref()
const map_ref = ref()
let gaode = null

const initMap = () => {
  //初始化高德地图：用来渲染底图
  gaode = new TileLayer({
    source: new XYZ({
      url: "http://wprd0{1-4}.is.autonavi.com/appmaptile?lang=zh_cn&size=1&style=7&x={x}&y={y}&z={z}",
      wrapX: false
    })
  })

  // 初始化地图视图
  const mapView = new View({
    center: fromLonLat([104.912777, 34.730746]),
    zoom: 4.5
  })

  // 初始化地图实例
  map.value = new Map({
    target: "map", // 对应页面里 id 为 map 的元素
    layers: [gaode], // 图层
    view: mapView //视图
  })
}

onMounted(() => {
  initMap()
})
</script>
<style lang="less" scoped>
.map_container {
  width: 100vh;
  height: 100vh;
}
</style>
