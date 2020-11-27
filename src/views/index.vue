<template>
  <div class="index">
    <div class="left">
      <div class="top">
        <div class="logo"></div>
        <div class="welCome">
          <p>智能避障拐杖</p>
          <p>盲人大数据分析平台</p>
        </div>
      </div>

      <div class="bottom">
        <div class="pattern">
          <div class="logoIntr">
            <span class="set">已设服务点</span>
            <span class="rec">推荐服务点</span>
          </div>
        </div>
        <Load v-show="load2" :type="2"></Load>
        <div class="heatMapCon" v-show="pattern == 0">
          <span
            v-for="(data, index) in heatMapRegion"
            :key="index"
            :class="{ choosed: heatMapIndex == index }"
            @click="heatMapIndex = index"
            >{{ data == "广州" ? "全广州" : data }}</span
          >
        </div>
        <div class="serviceCon" v-show="pattern == 1">
          <div class="dataCon" v-show="serverId">
            <div class="data">
              <div class="position">
                <span>
                  经度
                  <span>{{
                    parseFloat(chooseMarker.data.longitude).toFixed(2)
                  }}</span>
                </span>
                <span>
                  纬度
                  <span>{{
                    parseFloat(chooseMarker.data.latitude).toFixed(2)
                  }}</span>
                </span>
              </div>
              <div class="choose-data">
                <p>数据1</p>
                <input type="text" v-model="markerDate.num1" />
                <p>数据2</p>
                <input type="text" v-model="markerDate.num2" />
                <p>负责人</p>
                <input type="text" v-model="markerDate.leader" />
                <p>志愿者数量</p>
                <input type="text" v-model="markerDate.volunteerNum" />
              </div>
            </div>
            <div
              v-show="chooseMarker.data.type != 'set'"
              class="add button"
              @click="addMarker()"
            >
              设为服务点
            </div>
            <div
              v-show="chooseMarker.data.type == 'set'"
              class="sure button"
              @click="changeMaeker()"
            >
              确定修改
            </div>
            <div
              v-show="chooseMarker.data.type == 'set'"
              class="delete button"
              @click="deleteMarker()"
            >
              删除服务点
            </div>
          </div>
        </div>
      </div>
    </div>
    <div class="right">
      <div id="map"></div>
      <Load v-show="load1" :type="1"></Load>
    </div>
  </div>
</template>

<script lang="ts">
import { Component, Vue, Watch } from "vue-property-decorator";
import Load from "@/components/loading/loading.vue";
// import Vue from "vue";
// import Component from "vue-class-component";
import position1 from "../assets/position1.png";
// 推荐
import position2 from "../assets/position2.png";
// 已设
import position3 from "../assets/position3.png";
declare var AMap: any;
declare var heatmapData: any;

@Component({ components: { Load } })
export default class index extends Vue {
  // 地图实例
  map: any;
  // 模式
  pattern: number = 0; // 0--热力图， 1--服务点
  // 请求地址
  reqUrl: string = "http://121.196.111.30:10086";
  // 热力图实例
  heatmap: any = null;
  // 显示热力图的所在区域编号
  heatMapIndex: number = 0;
  // 热力图区域信息
  heatMapRegion: any = [
    "广州",
    "花都区",
    "南沙区",
    "增城区",
    "从化区",
    "番禺区",
    "白云区",
    "黄埔区",
    "荔湾区",
    "海珠区",
    "天河区",
    "越秀区",
  ];
  // 区域视野实例（用多边形描绘）
  polygons: any = [];
  // 已设服务点实例
  setMarkerList: any = [];
  // 推荐服务点实例
  recMarkerList: any = [];
  // 鼠标选中的服务点的圆圈标注
  chooseMarkerCircle: any = null;
  // 鼠标选中的服务点
  chooseMarker: any = { data: {} };
  // 鼠标选中的服务点id
  serverId: any = null;
  // 鼠标选中的服务点json数据
  markerDate: any = {
    num1: "",
    num2: "",
    leader: "",
    volunteerNum: "",
  };
  // 地图加载
  load1: any = false;
  // 服务点数据加载
  load2: any = false;
  tagEvent: any = null;

  // 初始化
  init(): void {
    this.map = new AMap.Map("map", {
      zoom: 14,
      center: [113.39, 23.05],
    });
    this.map.plugin([
      "AMap.Heatmap",
      "AMap.ElasticMarker",
      "AMap.DistrictSearch",
    ]);
    let option = {
      radius: 25,
      opacity: [0, 0.5],
      gradient: {
        0.5: "rgb(0, 0, 255)",
        0.65: "rgb(117, 211, 248)",
        0.7: "rgb(0, 255, 0)",
        0.9: "rgb(255, 234, 0)",
        1.0: "rgb(255, 0, 0)",
      },
    };
    this.heatmap = new AMap.Heatmap(this.map, option);
  }

  // 显示热力图
  getHeatMap() {
    this.changeView(this.heatMapRegion[this.heatMapIndex]);
    if (this.heatMapRegion[this.heatMapIndex] == "广州") {
      const message: any = this.$Message.loading({
        content: "热力图加载中...",
        duration: 0,
      });
      (this as any).$axios
        .get("http://121.196.111.30/gps/data/flow_data.json")
        .then((res: any) => {
          setTimeout(message, 0);
          this.heatmap.show();
          this.heatmap.setDataSet({ data: res.data });
          this.$Message.success("获取成功");
        })
        .catch((err: any) => {
          setTimeout(message, 0);
          console.log(err);
          this.$Message.error("获取失败");
        });
    } else {
      this.load1 = false;
      this.$Message.success(`已切换到${this.heatMapRegion[this.heatMapIndex]}`);
    }
  }

  // 热力图模块，改变地图视野
  changeView(districtName: string) {
    this.deleteView();
    var district = new AMap.DistrictSearch({ extensions: "all" });
    district.search(districtName, (status: any, result: any) => {
      if (status == "complete") {
        var bounds = result.districtList[0].boundaries;
        if (bounds) {
          for (let i = 0; i < bounds.length; i++) {
            var polygon = new AMap.Polygon({
              map: this.map,
              strokeWeight: 0,
              strokeColor: "rgba(1,1,1,0)",
              fillColor: "#80d8ff",
              fillOpacity: 0,
              path: bounds[i],
            });
            this.polygons.push(polygon);
          }
          this.map.setFitView();
        }
      }
    });
  }

  // 热力图模块，删除地图视野
  deleteView() {
    for (let i = 0; i < this.polygons.length; i++) {
      this.polygons[i].setMap(null);
    }
    this.polygons = [];
  }

  // 获取服务点数据
  getMarker() {
    // 获取推荐服务点
    this.load1 = true;
    (this as any).$axios
      .get(this.reqUrl + "/api/data/listLocation/0")
      .then((res: any) => {
        console.log("推荐服务点list", res.data);
        let recMarkerDatas = res.data.data;
        let len1 = recMarkerDatas.length;
        for (let i = 0; i < len1; i++) {
          this.setMarker(recMarkerDatas[i], "rec");
        }
        // 获取设置的服务点
        (this as any).$axios
          .get(this.reqUrl + "/api/data/serverPoints")
          .then((res: any) => {
            console.log("已设服务点list", res.data);
            let setMarkerDatas = res.data.data;
            let len2 = setMarkerDatas.length;
            for (let i = 0; i < len2; i++) {
              this.setMarker(setMarkerDatas[i], "set");
            }
            this.load1 = false;
            this.map.setFitView();
          })
          .catch((err: any) => {
            console.log(err);
          });
      })
      .catch((err: any) => {
        console.log(err);
      });

    return;
  }

  // 显示服务点
  setMarker(
    obj: {
      serverId: number | string;
      longitude: number | string;
      latitude: number | string;
    },
    type: string
  ) {
    // serverId 可以用来请求marker的其他数据
    let marker = new AMap.Marker({
      position: [obj.longitude, obj.latitude],
      icon: type == "rec" ? position2 : position3, // 图标
      offset: new AMap.Pixel(-25, -25),
    });
    this.bindMarkerData(marker, obj, type);
    if (type == "set") {
      this.setMarkerList.push(marker);
    } else {
      this.recMarkerList.push(marker);
    }
    marker.on("click", this.clickMarker);
    this.map.add([marker]);
    return marker;
  }

  // 给服务点绑定具体信息
  bindMarkerData(marker: any, obj: any, type: string) {
    marker.data = {
      serverId: obj.serverId,
      longitude: obj.longitude,
      latitude: obj.latitude,
      type: type,
    };
  }

  // 获取服务点具体JSON数据
  getMarkerData() {
    if (!this.serverId) {
      this.load2 = false;
      return;
    }
    if (this.chooseMarker.data.type == "rec") {
      this.markerDate = {
        num1: "",
        num2: "",
        leader: "",
        volunteerNum: "",
      };
      this.load2 = false;
      return;
    }
    (this as any).$axios
      .get(this.reqUrl + "/api/test/select/" + this.serverId)
      .then((res: any) => {
        console.log("获取json", res.data);
        this.load2 = false;
        if (res.data.data && res.data.data.data) {
          this.markerDate = JSON.parse(res.data.data.data);
        } else {
          this.markerDate = {
            num1: "",
            num2: "",
            leader: "",
            volunteerNum: "",
          };
        }
      })
      .catch((err: any) => {
        console.log(err);
      });
  }

  // 服务点点击事件
  clickMarker(event: any) {
    if (this.load1 || this.load2) {
      alert("操作频繁！");
      return;
    }
    let marker = event.target;
    // 再次点击取消选择
    if (this.tagEvent == marker) {
      this.map.remove(this.chooseMarkerCircle);
      this.tagEvent = null;
      this.pattern = 0;
      return;
    }
    this.pattern = 1;
    this.tagEvent = marker;
    this.chooseMarker = marker;
    this.serverId = marker.data.serverId;
    this.load2 = true;
    this.getMarkerData();
    if (this.chooseMarkerCircle) {
      this.map.remove(this.chooseMarkerCircle);
    }
    this.chooseMarkerCircle = new AMap.CircleMarker({
      center: new AMap.LngLat(marker.data.longitude, marker.data.latitude), // 圆心位置
      radius: 40, // 圆半径
      fillOpacity: 0,
      strokeColor: "#222", // 描边颜色
      strokeWeight: 5, // 描边宽度
    });
    this.map.add(this.chooseMarkerCircle);
  }

  // 添加服务点
  addMarker() {
    let judge = confirm("您确定设定改点为新的服务点吗？");
    if (!judge) {
      return;
    }
    this.load2 = true;
    let param = {
      longitude: this.chooseMarker.data.longitude,
      latitude: this.chooseMarker.data.latitude,
    };
    (this as any).$axios
      .post(this.reqUrl + "/api/data/addServerPoint", param)
      .then((res: any) => {
        console.log("添加", res.data);
        // 在地图添加一个服务点
        let obj = {
          serverId: res.data.data.serverId,
          longitude: this.chooseMarker.data.longitude,
          latitude: this.chooseMarker.data.latitude,
        };
        this.chooseMarker = this.setMarker(obj, "set");
        this.serverId = obj.serverId;
        // this.load2 = false;
        this.changeMaekerData();
      })
      .catch((err: any) => {
        console.log(err);
      });
  }

  // 修改服务点json数据
  changeMaekerData() {
    let param = {
      volunteerNum: this.markerDate.volunteerNum,
      leader: this.markerDate.leader,
      num1: this.markerDate.num1,
      num2: this.markerDate.num2,
    };
    (this as any).$axios
      .post(this.reqUrl + "/api/test/update", {
        id: this.serverId,
        data: JSON.stringify(param),
      })
      .then((res: any) => {
        console.log("修改json", res.data);
        this.getMarkerData();
      })
      .catch((err: any) => {
        console.log(err);
      });
  }

  // 删除服务点
  deleteMarker() {
    let judge = confirm("您确定删除这个服务点吗？");
    if (!judge) {
      return;
    }
    this.load2 = true;
    (this as any).$axios
      .post(this.reqUrl + "/api/data/deleteServerPoint", {
        serverId: this.serverId,
      })
      .then((res: any) => {
        console.log("删除", res.data);
        this.serverId = null;
        this.map.remove(this.chooseMarker);
        this.map.remove(this.chooseMarkerCircle);
        this.load2 = false;
        this.pattern = 0;
      })
      .catch((err: any) => {
        alert(err);
      });
  }

  // 修改服务点
  changeMaeker() {
    let judge = confirm("您确定修改这个服务点吗？");
    if (!judge) {
      return;
    }
    this.load2 = true;
    this.changeMaekerData();
  }

  mounted() {
    this.init();
    this.getHeatMap();
    this.getMarker();
  }
  
  @Watch("heatMapIndex")
  getCount(newVal: number) {
    this.getHeatMap();
  }
}
</script>

<style lang="scss">
@import "index.scss";
</style> 