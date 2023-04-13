<template>
  <div id="map" class="map_container" ref="map_ref"></div>
</template>

<script setup lang="ts">
import { ref, onMounted, nextTick } from "vue"
import { Map, View, Feature } from "ol" // 地图实例方法、视图方法
import { Tile as TileLayer, Vector as VectorLayer } from "ol/layer" // 瓦片渲染方法,矢量图层方法
import { Vector, XYZ, Cluster } from "ol/source" //矢量数据源
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
import { stringToGlsl } from "ol/style/expressions"

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

const clusterInit = () => {
  let clusterData = {
    成都市: { center: { lng: 104.061902, lat: 30.609503 } },
    广安市: { center: { lng: 106.619126, lat: 30.474142 } },
    绵阳市: { center: { lng: 104.673612, lat: 31.492565 } },
    雅安市: { center: { lng: 103.031653, lat: 30.018895 } },
    自贡市: { center: { lng: 104.797794, lat: 29.368322 } },
    宜宾市: { center: { lng: 104.610964, lat: 28.781347 } },
    内江市: { center: { lng: 105.064555, lat: 29.581632 } }
  }
  let points = [
    { name: "成都市", value: 85 },
    { name: "绵阳市", value: 36 },
    { name: "广安市", value: 50 },
    { name: "雅安市", value: 555 },
    { name: "自贡市", value: 55 },
    { name: "宜宾市", value: 666 },
    { name: "内江市", value: 777 }
  ]
  // 实现聚合分散方法
  addCluster(clusterData, points, true)
}

const addCluster = (clusterData: any, points: any, flag: boolean) => {
  let source = new Vector()
  for (const key in clusterData) {
    points.forEach((e: any) => {
      if (e.name == key) {
        let point = fromLonLat([
          clusterData[key].center.lng,
          clusterData[key].center.lat
        ])
        var f = new Feature({
          geometry: new Point(point)
        })
        f.set("name", e.name)
        f.set("value", e.value)
        source.addFeature(f)
      }
    })
  }
  let clusterSource = new Cluster({
    distance: parseInt("20", 10), //控制距离小于多少就开始聚合
    source: source
  })

  let layer = new VectorLayer({
    source: clusterSource,
    style: clusterStyle() //条件样式是将样式配置为一个回调函数方法，其参数包含要素本身和分辨率
  })
  map.value.addLayer(layer)
}

//动态样式配置方法
const clusterStyle = () => {
  return (feature: any, solution: any) => {
    var total = 0
    feature.get("features").forEach((value: any, index: any) => {
      total += value.get("value")
    })
    var style = new Style({
      image: new Circle({
        radius: 15,
        stroke: new Stroke({
          color: "blue"
        }),
        fill: new Fill({
          color: "rgba(24,144,255,100)"
        })
      }),
      text: new Text({
        text: total.toString(),
        fill: new Fill({
          color: "#FFF"
        }),
        font: "12px Calibri,sans-serif",
        stroke: new Stroke({
          color: "red",
          width: 5
        })
      })
    })
    return style
  }
}

onMounted(() => {
  initMap()
  clusterInit()
})
</script>
<style lang="less" scoped>
.map_container {
  width: 100vh;
  height: 100vh;
}
</style>
