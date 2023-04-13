<template>
  <div id="map" class="map_container"></div>
  <div id="marker" ref="market"></div>
  <div id="textInfo" ref="testInfo">我是text文本信息</div>
  <div id="popup" class="ol-popup" ref="popup">
    <a href="#" id="popup-closer" class="ol-popup-closer" ref="popupCloser"></a>
    <div id="popup-content" class="popup-content" ref="popupContent"></div>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted } from "vue"
import { Map, View, Feature } from "ol" // 地图实例方法、视图方法
import { Tile as TileLayer, Vector as VectorLayer } from "ol/layer" // 瓦片渲染方法,矢量图层方法
import { Vector, XYZ } from "ol/source" //矢量数据源
import { Point, LineString } from "ol/geom" //几何点、线
import * as control from "ol/control" //视图控件
import { Style, Circle, Fill, Stroke, Text, Icon } from "ol/style.js" //设置样式：圆，填充，描边，文本,图片
import { GeoJSON, MVT } from "ol/format" //格式化数据,数据格式有GeoJSON,MVT等
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
const popup = ref()
const popupCloser = ref()
const popupContent = ref()
const vectorLayer = ref()
const market = ref()
const testInfo = ref()
let gaode = null

const initMap = () => {
  //初始化高德地图：用来渲染底图
  gaode = new TileLayer({
    source: new XYZ({
      url: "http://wprd0{1-4}.is.autonavi.com/appmaptile?lang=zh_cn&size=1&style=7&x={x}&y={y}&z={z}",
      wrapX: false
    })
  })

  //初始化地图放大缩小控件ZoomSlider
  const zoomslider = new control.ZoomSlider({})

  //初始化地图跳转控件ZoomToExtent
  const zoomToExtent = new control.ZoomToExtent({
    extent: [110, 30, 160, 30]
  })

  //初始化比例尺控件scaleline
  const scaleLine = new control.ScaleLine({})

  //初始化全图控件FullScreen
  const fullScreen = new control.FullScreen({})

  //鼠标位置控件mouseposition
  const mousePosition = new control.MousePosition({})

  //全局地图控件overviewmap 显示当前视口中的地图位于全局地图的哪一部分
  //地图旋转控件rotate

  //初始化矢量图层的方法
  vectorLayer.value = new VectorLayer({
    source: new Vector()
  })

  // 初始化地图视图
  const mapView = new View({
    projection: "EPSG:4326", // 坐标系，有EPSG:4326和EPSG:3857，需要高精度和分辨率，选择 EPSG:3857；如果需要标准的地理坐标系，选择 EPSG:4326
    center: [114.3, 30.5], // 坐标,选择EPSG:3857需要fromLonLat( [114.064839, 22.548857])
    minZoom: 0, // 地图缩放最小级别
    maxZoom: 18, // 地图缩放最大级别
    zoom: 12 // 地图缩放级别(打开页面时默认级别)
  })

  // 初始化地图实例
  map.value = new Map({
    target: "map", // 对应页面里 id 为 map 的元素
    layers: [gaode, vectorLayer.value], // 图层
    view: mapView, //视图
    controls: control
      .defaults()
      .extend([zoomToExtent, zoomslider, fullScreen, scaleLine, mousePosition]), //加载地图控件
    interactions: interaction
      .defaults()
      .extend([new interaction.DragRotateAndZoom()])
  })
  //添加交互功能DragBox
  map.value.addInteraction(new interaction.DragBox())

  //设置矢量图形
  drawVectorLayer(map.value)

  //geojson设置点要素
  gjdrawPointFeature(map.value, [114.4, 30.6])
  //geojson设置线和区
  gjdrawLineAndArea(map.value)
  //加载本地geojson
  onloadLocalData(map.value)
  //加载网络geojson
  onloadNetworkData(map.value)

  //地图点击事件和漫游:飞行视角
  map.value.on("click", (evt: any) => {
    //console.log(evt)
    const { coordinate } = evt
    gjdrawPointFeature(map.value, coordinate) //描点

    flyPerspective(map.value, coordinate) //view移动

    //点击地图出现点标记
    addPinotMarket(map.value, coordinate)

    //点击地图出现文本
    addText(map.value, coordinate)
  })

  //制作GIF动画
  drawGif(map.value)

  //点击地图出现Popup弹窗
  addPopup(map.value)

  //解决控件重叠
  document
    .querySelector(".ol-zoomslider")
    ?.setAttribute("style", "pointer-events: auto;top:6.5em")

  //map.addLayer(tileLayer); map.removeOverlay(marker);
}

//设置矢量图形
const drawVectorLayer = (map: any) => {
  // 1、通过几何信息和样式信息构建要素
  var point = new Feature({
    geometry: new Point([114.3, 30.5])
  })
  let style = new Style({
    image: new Circle({
      radius: 10,
      fill: new Fill({
        color: "#ff2d51"
      }),
      stroke: new Stroke({
        color: "#333",
        width: 2
      })
    })
  })
  point.setStyle(style) //将样式给到要素点
  // 2、将要素添加到矢量数据源
  const source = new Vector({
    features: [point]
  })
  // 3、将矢量数据源添加到矢量图层
  const layer = new VectorLayer({
    source
  })
  // 4、将矢量图层添加到地图容器
  map.addLayer(layer)
}

//geojson设置点要素
const gjdrawPointFeature = (map: any, coordinate: number[]) => {
  //1.创建geojson数据
  var data = {
    type: "FeatureCollection", //固定格式
    features: [
      {
        type: "Feature",
        geometry: {
          type: "Point", //点
          coordinates: coordinate
        }
      }
    ]
  }
  // 2、将geojson数据设置给矢量数据源
  const source = new Vector({
    features: new GeoJSON().readFeatures(data)
  })
  // 3、将矢量数据源添加到矢量图层
  const layer = new VectorLayer({
    source
  })
  let style = new Style({
    image: new Circle({
      radius: 10,
      fill: new Fill({
        color: "yellow"
      }),
      stroke: new Stroke({
        color: "#333",
        width: 2
      })
    })
  })
  layer.setStyle(style)
  // 4、将矢量图层添加到地图容器
  map.addLayer(layer)
}

//geojson设置线和区
const gjdrawLineAndArea = (map: any) => {
  //1.创建geojson数据
  var data = {
    type: "FeatureCollection", //固定格式
    features: [
      {
        type: "Feature",
        geometry: {
          type: "LineString", //线
          coordinates: [
            [114.3, 30.5],
            [114.3, 30.6]
          ]
        }
      },
      {
        type: "Feature",
        geometry: {
          type: "Polygon", //区
          coordinates: [
            [
              [114.3, 30.5],
              [114.3, 30.6],
              [114.4, 30.6]
            ]
          ]
        }
      }
    ]
  }
  // 2、将geojson数据设置给矢量数据源
  const source = new Vector({
    features: new GeoJSON().readFeatures(data)
  })
  // 3、将矢量数据源添加到矢量图层
  const layer = new VectorLayer({
    source
  })
  let style = new Style({
    stroke: new Stroke({
      color: "#ff2d51",
      width: 4
    }),
    fill: new Fill({
      color: "rgba(50, 50, 50, 0.3)"
    })
  })
  layer.setStyle(style)
  // 4、将矢量图层添加到地图容器
  map.addLayer(layer)
}

//加载本地geojson
const onloadLocalData = (map: any) => {
  // 1、将本地geojson数据设置给矢量数据源
  const source = new Vector({
    url: "../public/map.geojson",
    format: new GeoJSON()
  })
  // 2、将矢量数据源添加到矢量图层
  const layer = new VectorLayer({
    source
  })
  let style = new Style({
    image: new Circle({
      radius: 10,
      fill: new Fill({
        color: "green"
      }),
      stroke: new Stroke({
        color: "#333",
        width: 2
      })
    })
  })
  layer.setStyle(style)
  // 3、将矢量图层添加到地图容器
  map.addLayer(layer)
}

//加载网络geojson
const onloadNetworkData = (map: any) => {
  // 1、将网络geojson数据设置给矢量数据源
  let source = new Vector({
    url: "https://geo.datav.aliyun.com/areas_v3/bound/geojson?code=420100",
    format: new GeoJSON()
  })
  // 2、将矢量数据源添加到矢量图层
  const layer = new VectorLayer({
    source //矢量数据源的名称不能修改，必须为source
  })
  const style = new Style({
    //填充色
    fill: new Fill({
      color: "rgba(50,50,50,0.4)"
    }),
    //边线颜色
    stroke: new Stroke({
      color: "#ff2d5180",
      width: 2
    })
  })
  layer.setStyle(style)
  // 3、将矢量图层添加到地图容器
  map.addLayer(layer)

  source = new Vector({
    url: "https://geo.datav.aliyun.com/areas_v3/bound/geojson?code=340000",
    format: new GeoJSON()
  })
  const layer1 = new VectorLayer({
    source
  })
  layer1.setStyle(style)
  map.addLayer(layer1)
}

//漫游
const flyPerspective = (map: any, position: number[]) => {
  map.getView().animate({
    center: position,
    zoom: 10,
    duration: 2000
  })
}

//在画布中制作 GIF 动画
const drawGif = (map: any) => {}

//点击地图出现Popup弹窗
const addPopup = (map: any) => {
  // 使用变量存储弹窗所需的 DOM 对象
  let container = popup.value
  let closer = popupCloser.value
  let content = popupContent.value

  // 创建一个弹窗 Overlay 对象
  const overlay = new Overlay({
    element: container, //绑定 Overlay 对象和 DOM 对象的
    autoPan: {
      animation: {
        duration: 250
      }
    }
  })
  // 将弹窗添加到 map 地图中
  map.addOverlay(overlay)

  /**
   * 为弹窗添加一个响应关闭的函数
   */
  closer!.onclick = () => {
    overlay.setPosition(undefined)
    closer!.blur()
    return false
  }
  /**
   * 添加单击map 响应函数来处理弹窗动作
   */
  map.on("singleclick", (evt: any) => {
    //使用EPSG:4326
    let coordinate = evt.coordinate
    //使用EPSG:3857
    //let coordinate = transform(evt.coordinate, "EPSG:3857", "EPSG:4326")
    // 点击尺 （这里是尺(米)，并不是经纬度）;
    let hdms = toStringHDMS(evt.coordinate) // 转换为经纬度显示
    content!.innerHTML = `
        <p>你点击了这里：</p>
        <p>经纬度：<p><code> ${hdms}  </code> <p>
        <p>坐标：</p>X:${coordinate[0]} &nbsp;&nbsp; Y: ${coordinate[1]}`
    overlay.setPosition(evt.coordinate) //把 overlay 显示到指定的 x,y坐标
  })
}

//overlay实现点击地图出现点标记
const addPinotMarket = (map: any, coordinate: number[]) => {
  let marketElement = market.value
  let markerOverLay = new Overlay({
    position: coordinate,
    positioning: "center-center",
    element: marketElement,
    stopEvent: false
  })
  map.addOverlay(markerOverLay)
}

//overlay实现text文本信息
const addText = (map: any, coordinate: number[]) => {
  const textElement = testInfo.value
  var textInfo = new Overlay({
    position: coordinate,
    offset: [20, -20],
    element: textElement
  })
  map.addOverlay(textInfo)
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
#map .ol-zoomslider {
  top: 6.5em;
}
.ol-popup {
  position: absolute;
  background-color: white;
  -webkit-filter: drop-shadow(0 1px 4px rgba(0, 0, 0, 0.2));
  filter: drop-shadow(0 1px 4px rgba(0, 0, 0, 0.2));
  padding: 15px;
  border-radius: 10px;
  border: 1px solid #cccccc;
  bottom: 12px;
  left: -50px;
}
.ol-popup:after,
.ol-popup:before {
  top: 100%;
  border: solid transparent;
  content: " ";
  height: 0;
  width: 0;
  position: absolute;
  pointer-events: none;
}
.ol-popup:after {
  border-top-color: white;
  border-width: 10px;
  left: 48px;
  margin-left: -10px;
}
.ol-popup:before {
  border-top-color: #cccccc;
  border-width: 11px;
  left: 48px;
  margin-left: -11px;
}
.ol-popup-closer {
  text-decoration: none;
  position: absolute;
  top: 2px;
  right: 8px;
}
.popup-content {
  width: 400px;
}
.ol-popup-closer:after {
  content: "✖";
}
#marker {
  width: 20px;
  height: 20px;
  background: red;
  border-radius: 50%;
}
#textInfo {
  width: 200px;
  height: 40px;
  line-height: 40px;
  background: burlywood;
  color: yellow;
  text-align: center;
  font-size: 20px;
}
</style>
