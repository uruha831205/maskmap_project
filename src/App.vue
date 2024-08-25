<script setup>
import { RouterLink, RouterView } from "vue-router";
import citynames from "./assets/CityCountyData.json";
import city_area_c from "./assets/city_area_c.json";
import { ref, watch, watchEffect, onMounted, computed } from "vue";
import axios from "axios";
import L from "leaflet";
import "leaflet/dist/leaflet.css";
import Icon_hos from "@/assets/icon/hos.svg";

import default_icon_shadow from "leaflet/dist/images/marker-shadow.png";
import default_icon from "leaflet/dist/images/marker-icon-2x.png";

let openStreetMap; //地圖
let city_area_data = city_area_c; //縣市和行政區的經緯度

const all_data = ref([]); // 從 第三方API取得 醫院整所資料
const selected_city = ref(""); // 下拉式選單選擇的縣市
const selected_area = ref(""); // 下拉式選單選擇的行政區

//test : 自訂地圖標示 icon
var defalut_Icon = L.icon({
  iconUrl: default_icon, // 圖片來源
  iconSize: [25, 40], // icon大小
  iconAnchor: [13, 40], // icon錨點
  // iconAnchor: [22, 80],
  shadowUrl: default_icon_shadow, // 陰影圖片來源
  shadowAnchor: [13, 40], // 陰影錨點
  popupAnchor: [0, -40], // 訊息錨點 : 點擊icon後訊息框顯示的位置
});

//自訂地圖標示 icon
var myIcon = L.icon({
  iconUrl: Icon_hos, //圖片來源
  iconSize: [30, 30], //icon大小
  // iconAnchor: [22, 80],
  popupAnchor: [0, -20], //點擊icon後訊息框顯示的位置
});

//返回被選取的縣市和區域的醫院診所之座標
const rtn_datas = computed(() => {
  const choose_datas = ref([]);

  choose_datas.value = all_data.value.filter(
    (data) => data.properties.county === selected_city.value.CityName
  );
  if (selected_area.value != "") {
    choose_datas.value = choose_datas.value.filter(
      (data) => data.properties.town === selected_area.value.AreaName
    );
  }
  //return choose_datas.value.map((data) => data.geometry.coordinates); // 從資料集中選出特定資料
  return choose_datas.value;
});

//若縣市有變更，selected_area變數自動清空 & 執行設置選擇地點的中心經緯度和ICON
watch(selected_city, () => {
  selected_area.value = "";
  set_center_coordinate("city", openStreetMap, 11);
  add_coordinates();
});

//若區域有變更，執行設置選擇地點中心經緯度和ICON
watch(selected_area, () => {
  set_center_coordinate("area", openStreetMap, 14);
  add_coordinates();
});

//設置選擇地點的中心經、緯度
function set_center_coordinate(choose, openStreetMap, zoom_num) {
  let rtn_data = "";
  switch (choose) {
    case "city":
      rtn_data = city_area_data.find(
        (item) => item.city === selected_city.value.CityName
      );
      break;
    case "area":
      rtn_data = city_area_data.find(
        (item) =>
          item.city === selected_city.value.CityName &&
          item.area === selected_area.value.AreaName
      );
      break;
  }
  if (rtn_data != null) {
    openStreetMap.setView([rtn_data.lat, rtn_data.lng], zoom_num); //畫面移動至該區的座標
  }
}

//新增ICON
function add_coordinates() {
  if (rtn_datas.value != "") {
    //清空之前新增的ICON
    openStreetMap.eachLayer((layer) => {
      if (layer instanceof L.Marker) {
        openStreetMap.removeLayer(layer);
      }
    });
    //走訪選取的座標資訊並建立ICON
    for (let item of rtn_datas.value) {
      L.marker([item.geometry.coordinates[1], item.geometry.coordinates[0]], {
        icon: myIcon,
      }).addTo(openStreetMap).bindPopup(`<p><h1>${item.properties.name}<h1></p>
        <h2>口罩剩餘數量<br>成人 : <span style="color:red;">${item.properties.mask_adult}</span><br>小孩 : <span style="color:red;">${item.properties.mask_child}</span><br></h2>
        <h3>${item.properties.phone}</h3>
        <h3>${item.properties.address}</h3>
        <h3>最後更新時間:${item.properties.updated}</h3>
          `);
    }
  }
}

//從第三方API 取得各縣市區的藥局診所資料
async function fetchPharmacies() {
  try {
    const response = await axios.get(
      "https://raw.githubusercontent.com/kiang/pharmacies/master/json/points.json"
    );
    all_data.value = response.data.features; // 根據 JSON 結構選擇正確的資料
  } catch (error) {
    console.error("Error fetching pharmacies:", error);
  }
}

onMounted(() => {
  //載入後先取得各縣市區的藥局診所資料
  fetchPharmacies();

  openStreetMap = L.map("map", {
    //畫面顯示的座標
    center: [25.042474, 121.513729],
    // 可以嘗試改變 zoom 的數值 : 縮放地圖大小
    // 筆者嘗試後覺得 18 的縮放比例是較適當的查詢範圍
    zoom: 18,
  });

  L.tileLayer("https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png", {
    attribution:
      'Map data &copy; <a href="https://www.openstreetmap.org/">OpenStreetMap</a> contributors, <a href="https://creativecommons.org/licenses/by-sa/2.0/">CC-BY-SA</a>, Imagery © <a href="https://www.mapbox.com/">Mapbox</a>',
    maxZoom: 20,
  }).addTo(openStreetMap);

  var marker1 = L.marker([25.042474, 121.513739], { icon: defalut_Icon }).addTo(
    openStreetMap
  );
  //var marker2 = L.marker([25.031753, 121.520215]).addTo(openStreetMap);

  var circle = L.circle([25.042474, 121.51373], {
    color: "red", //外框
    fillColor: "red", //內圈，#色碼也可以
    fillOpacity: 0.4, //內圈透明度
    radius: 50, //半徑
  }).addTo(openStreetMap);

  //移除
  //circle.remove(openStreetMap);

  // 多邊形
  var polygon = L.polygon([
    [25.041474, 121.51273], //左下
    [25.043474, 121.51273], //右下
    [25.043474, 121.51473], //右上
    [25.041474, 121.51473], //左上
  ]).addTo(openStreetMap);

  //未知
  var popup = L.popup()
    .setLatLng([25.044474, 121.513799])
    .setContent("I am a standalone popup.")
    .openOn(openStreetMap);

  marker1.bindPopup("Hello world!<br>I am a popup.").openPopup(); // bindPopup() : 自訂訊息框內容，openPopup() : 地圖打開後訊息框自動開啟
  // circle.bindPopup("I am a circle.").openPopup();
  // polygon.bindPopup("I am a polygon.").openPopup();
});
</script>

<template>
  <div class="layout">
    <div class="select-group">
      <h1 style="white-space: nowrap">口罩地圖</h1>
      <div class="select-city">
        <label for="city">縣市：</label>
        <select id="city" v-model="selected_city">
          <option value="" disabled>請選擇縣市</option>
          <option
            :value="cityname"
            v-for="cityname in citynames"
            :key="cityname.CityName"
          >
            {{ cityname.CityName }}
          </option>
        </select>
      </div>

      <div class="select-section">
        <label for="section">區域：</label>
        <select id="section" v-model="selected_area">
          <option value="" disabled>請選擇區域</option>
          <option
            :value="sectionname"
            v-for="sectionname in selected_city.AreaList"
            :key="sectionname.AreaName"
          >
            {{ sectionname.AreaName }}
          </option>
        </select>
      </div>
      <div style="white-space: nowrap">
        目前選擇縣市 : {{ selected_city.CityName }}
      </div>
      <div style="white-space: nowrap">
        目前選擇區域 : {{ selected_area.AreaName }}
      </div>
    </div>

    <div class="map-group">
      <div id="map"></div>
    </div>
  </div>
</template>

<style scoped>
.layout {
  display: flex;
}

.select-city,
.select-section {
  white-space: nowrap;
}

.select-group {
  width: fit-content;
  height: 100vh;
  background-color: aqua;
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 5px;
}

.map-group {
  width: 100vw;
  height: 100vh;
  background-color: aquamarine;
}

.map-group #map {
  width: 100%;
  height: 100%;
}

@media screen and (max-width: 1230px) and (min-width: 430px) {
  .layout {
    flex-direction: column;
    height: 100vh;
  }

  .select-group {
    width: 100%;
    height: fit-content;
  }

  .map-group {
    width: 100%;
  }
}

@media screen and (max-width: 430px) {
  .layout {
    flex-direction: column-reverse;
    height: 100vh;
  }

  .select-group {
    width: 100%;
    height: fit-content;
  }

  .map-group {
    width: 100%;
  }
}
</style>
