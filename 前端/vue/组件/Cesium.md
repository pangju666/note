```javascript
<script setup>
import * as Cesium from "cesium";
import {
  BoundingSphere,
  Cartesian2,
  Cartesian3,
  Color,
  CustomDataSource,
  DistanceDisplayCondition,
  Ellipsoid,
  HorizontalOrigin,
  Ion,
  ScreenSpaceEventHandler,
  ScreenSpaceEventType,
  VerticalOrigin,
  Viewer,
} from "cesium";
import { onMounted, watch } from "vue";
import {
  ArrayUtils,
  NumberUtils,
  ObjectUtils,
  StringUtils,
} from "pangju-utils";

//import "cesium/Build/Cesium/Widgets/widgets.css";
import {
  LOCATION_TYPE,
  POLYGON_TYPE,
  POLYLINE_TYPE,
  SPHERE_LOCATION_TYPE,
} from "@/utils/constant.js";
import { calculateArea, calculateDistance } from "@/utils/geo.js";

const props = defineProps({
  accessToken: {
    type: String,
    required: true,
  },
  config: {
    type: Object,
    required: true,
  },
  data: {
    type: Array,
    required: false,
    default: () => [],
  },
  minPolygonDisplayHeight: {
    type: Number,
    required: false,
    default: undefined,
  },
  minPolylineDisplayHeight: {
    type: Number,
    required: false,
    default: undefined,
  },
  typeField: {
    type: String,
    required: false,
    default: "type",
  },
  coordinatesField: {
    type: String,
    required: false,
    default: "coordinates",
  },
  polygonAreaField: {
    type: String,
    required: false,
    default: "area",
  },
  polylineDistanceField: {
    type: String,
    required: false,
    default: "distance",
  },
  idField: {
    type: String,
    required: false,
    default: "id",
  },
  nameField: {
    type: String,
    required: false,
    default: "name",
  },
  minZoomDistance: {
    type: Number,
    required: false,
    default: undefined,
  },
  maxZoomDistance: {
    type: Number,
    required: false,
    default: undefined,
  },
  labelFontSize: {
    type: Number,
    required: false,
    default: 18,
  },
  labelFontFamily: {
    type: String,
    required: false,
    default: "sans-serif",
  },
  labelLength: {
    type: Number,
    required: false,
    default: 10,
  },
  labelColor: {
    type: String,
    required: false,
    default: "#fff",
  },
  drawColor: {
    type: String,
    required: false,
    default: "#eb4b16",
  },
  polylineColor: {
    type: String,
    required: false,
    default: "#fdc85f",
  },
  polygonColor: {
    type: String,
    required: false,
    default: "#D42E2E",
  },
  polylineWidth: {
    type: Number,
    required: false,
    default: 1.0,
  },
  polygonAlpha: {
    type: Number,
    required: false,
    default: 0.25,
  },
  pointPixelSize: {
    type: Number,
    required: false,
    default: 5,
  },
  drawLocationIcon: {
    type: String,
    required: true,
  },
  initCameraLongitude: {
    type: Number,
    required: false,
    default: undefined,
  },
  initCameraLatitude: {
    type: Number,
    required: false,
    default: undefined,
  },
  initCameraHeight: {
    type: Number,
    required: false,
    default: 1000000,
  },
  clusteredLocationIcon: {
    type: [String, Function],
    required: true,
  },
  locationIcon: {
    type: Function,
    required: true,
  },
  locationLabelIcon: {
    type: Function,
    required: true,
  },
  timerTimeout: {
    type: Number,
    default: 500,
  },
});

const emits = defineEmits([
  "ready",
  "click-entity",
  "camera-move-end",
  "draw-start",
]);

let viewer = null;
let ellipsoid = null;
let handler = null;
let timer = null;

const entityDataMap = new Map();

const locationDataSource = new CustomDataSource("locationDataSource");
const polylineDataSource = new CustomDataSource("polylineDataSource");
const polygonDataSource = new CustomDataSource("polygonDataSource");

onMounted(async () => {
  Ion.defaultAccessToken = props.accessToken;
  viewer = new Viewer("cesium-container", props.config);
  ellipsoid = viewer.scene.globe.ellipsoid;

  // 关闭鼠标操作惯性
  const screenSpaceCameraController = viewer.scene.screenSpaceCameraController;
  screenSpaceCameraController.inertiaSpin = 0;
  screenSpaceCameraController.inertiaTranslate = 0;
  screenSpaceCameraController.inertiaZoom = 0;

  if (ObjectUtils.nonNull(props.minZoomDistance)) {
    viewer.scene.screenSpaceCameraController.minimumZoomDistance =
      props.minZoomDistance;
  }
  if (ObjectUtils.nonNull(props.maxZoomDistance)) {
    viewer.scene.screenSpaceCameraController.maximumZoomDistance =
      props.maxZoomDistance;
  }
  if (
    ObjectUtils.allNotNull(props.initCameraLongitude, props.initCameraLatitude)
  ) {
    viewer.camera.setView({
      destination: Cartesian3.fromDegrees(
        props.initCameraLongitude,
        props.initCameraLatitude,
        props.initCameraHeight,
      ),
    });
  }

  initHandler();

  /**
   * 添加相机移动结束事件，也就是鼠标移动结束事件
   * 由于相机自身有惯性，所以提前关闭惯性会有更好的体验
   * 但是这本身并不影响该功能
   * */
  viewer.camera.moveEnd.addEventListener(() => {
    //2D下会可能拾取不到坐标，extend返回undefined,所以做以下转换
    let canvas = viewer.scene.canvas;
    let upperLeft = new Cartesian2(0, 0); //canvas左上角坐标转2d坐标
    let lowerRight = new Cartesian2(canvas.clientWidth, canvas.clientHeight); //canvas右下角坐标转2d坐标

    let ellipsoid = viewer.scene.globe.ellipsoid;
    let upperLeft3 = viewer.camera.pickEllipsoid(upperLeft, ellipsoid); //2D转3D世界坐标
    let lowerRight3 = viewer.camera.pickEllipsoid(lowerRight, ellipsoid); //2D转3D世界坐标
    if (ObjectUtils.isNull(upperLeft3) || ObjectUtils.isNull(lowerRight3)) {
      return;
    }

    let upperLeftCartographic =
      viewer.scene.globe.ellipsoid.cartesianToCartographic(upperLeft3); //3D世界坐标转弧度
    let lowerRightCartographic =
      viewer.scene.globe.ellipsoid.cartesianToCartographic(lowerRight3); //3D世界坐标转弧度
    if (
      ObjectUtils.isNull(upperLeftCartographic) ||
      ObjectUtils.isNull(lowerRightCartographic)
    ) {
      return;
    }

    let minLongitude = Cesium.Math.toDegrees(upperLeftCartographic.longitude); //弧度转经纬度
    let minLatitude = Cesium.Math.toDegrees(upperLeftCartographic.latitude); //弧度转经纬度

    let maxLongitude = Cesium.Math.toDegrees(lowerRightCartographic.longitude); //弧度转经纬度
    let maxLatitude = Cesium.Math.toDegrees(lowerRightCartographic.latitude); //弧度转经纬度

    const height = viewer.camera.positionCartographic.height;
    locationDataSource.clustering.enabled = height > props.initCameraHeight;

    emits(
      "camera-move-end",
      { longitude: minLongitude, latitude: minLatitude },
      { longitude: maxLongitude, latitude: maxLatitude },
      height,
    );
  });

  locationDataSource.clustering.enabled = true;
  locationDataSource.clustering.pixelRange = 50;
  locationDataSource.clustering.minimumClusterSize = 2;
  locationDataSource.clustering.clusterEvent.addEventListener(
    (clusteredEntities, cluster) => {
      cluster.label.show = false;
      cluster.billboard.show = true;
      cluster.billboard.image =
        typeof props.clusteredLocationIcon === "string"
          ? props.clusteredLocationIcon
          : props.clusteredLocationIcon(clusteredEntities);

      clusteredEntities.forEach((entity) => {
        entity.isClusterd = true;
      });
      resetTimer();
    },
  );

  await viewer.dataSources.add(locationDataSource);
  await viewer.dataSources.add(polylineDataSource);
  await viewer.dataSources.add(polygonDataSource);

  emits("ready");
});

watch(
  () => props.data,
  (newVal) => {
    clearEntities();
    if (ArrayUtils.isNotEmpty(newVal)) {
      renderEntities(newVal);
      if (!locationDataSource.clustering.enabled || newVal.length <= 100) {
        resetTimer();
      }
    }
  },
  { deep: true },
);

const resetTimer = () => {
  if (ObjectUtils.nonNull(timer)) {
    clearTimeout(timer);
  }
  timer = setTimeout(() => {
    locationDataSource.entities.values.forEach((entity) => {
      if (!entity.isClusterd) {
        entity.billboard.image = props.locationLabelIcon(entity.data);
      }
    });
  }, 500);
};
const initHandler = () => {
  destroyHandler();
  handler = new ScreenSpaceEventHandler(viewer.scene.canvas);
  handler.setInputAction((e) => {
    const entity = viewer.scene.pick(e.position);
    if (ObjectUtils.nonNull(entity?.id?.id)) {
      const data = entityDataMap.get(entity.id.id);
      if (ObjectUtils.nonNull(data)) {
        emits("click-entity", entity.id.id, data);
      } else {
        emits("click-entity", entity.id.id, entity.id.description._value);
      }
    }
  }, ScreenSpaceEventType.LEFT_CLICK);
};
const destroyHandler = () => {
  if (ObjectUtils.nonNull(handler)) {
    handler.destroy(); //关闭事件句柄
    handler = null;
  }
};

const renderLocation = (longitude, latitude, label, data) => {
  let entity = {
    position: Cartesian3.fromDegrees(longitude, latitude),
    billboard: {
      image: props.locationIcon(data),
      verticalOrigin: VerticalOrigin.BOTTOM,
    },
    data: data,
  };
  entity = locationDataSource.entities.add(entity);
  if (ObjectUtils.nonNull(entity?.id)) {
    entityDataMap.set(entity.id, data);
  }
  return entity;
};
const renderPolyline = (positions, label, distance, data) => {
  const coordinates = [];

  for (let i = 0; i < positions.length; i++) {
    polylineDataSource.entities.add({
      position: Cartesian3.fromDegrees(
        positions[i].longitude,
        positions[i].latitude,
      ),
      point: {
        color: Color.fromCssColorString(props.polylineColor),
        pixelSize: props.pointPixelSize,
      },
    });
    coordinates.push(positions[i].longitude, positions[i].latitude);
  }

  const entity = polylineDataSource.entities.add({
    polyline: {
      positions: Cartesian3.fromDegreesArray(coordinates),
      width: props.polylineWidth,
      material: Color.fromCssColorString(props.polylineColor),
    },
  });

  const labelText = formatLabelText(label, props.labelLength);
  const distanceText = ObjectUtils.nonNull(distance)
    ? `（距离：${distance}cm）`
    : "";
  const polylinePositions = entity.polyline.positions._value;
  const centerPosition = BoundingSphere.fromPoints(polylinePositions).center;
  entity.position = Ellipsoid.WGS84.scaleToGeodeticSurface(centerPosition);
  entity.label = {
    distanceDisplayCondition: ObjectUtils.nonNull(
      props.minPolylineDisplayHeight,
    )
      ? new DistanceDisplayCondition(0, props.minPolylineDisplayHeight)
      : undefined,
    font: `${props.labelFontSize}px ${props.labelFontFamily}`,
    color: Color.fromCssColorString(props.labelColor),
    text: labelText + distanceText,
  };

  if (ObjectUtils.nonNull(entity?.id)) {
    entityDataMap.set(entity.id, data);
  }
  return entity;
};
const renderPolygon = (positions, label, area, data) => {
  const coordinates = [];

  for (let i = 0; i < positions.length; i++) {
    // 绘制顶点
    polygonDataSource.entities.add({
      position: Cartesian3.fromDegrees(
        positions[i].longitude,
        positions[i].latitude,
      ),
      point: {
        color: Color.fromCssColorString(props.polygonColor),
        pixelSize: props.pointPixelSize,
      },
    });
    const position = Cartesian3.fromDegrees(
      positions[i].longitude,
      positions[i].latitude,
    );
    coordinates.push(position);

    // 绘制连接线
    if (i > 0) {
      const startPosition = Cartesian3.fromDegrees(
        positions[i - 1].longitude,
        positions[i - 1].latitude,
      );
      polygonDataSource.entities.add({
        polyline: {
          positions: [startPosition, position],
          width: props.polylineWidth,
          material: Color.fromCssColorString(props.polygonColor),
        },
      });
    }
  }

  const startPosition = Cartesian3.fromDegrees(
    positions[0].longitude,
    positions[0].latitude,
  );
  const endPosition = Cartesian3.fromDegrees(
    positions[positions.length - 1].longitude,
    positions[positions.length - 1].latitude,
  );
  polygonDataSource.entities.add({
    polyline: {
      positions: [startPosition, endPosition],
      width: props.polylineWidth,
      material: Color.fromCssColorString(props.polygonColor),
    },
  });

  const entity = polygonDataSource.entities.add({
    polygon: {
      hierarchy: coordinates,
      material: Color.fromAlpha(
        Color.fromCssColorString(props.polygonColor),
        props.polygonAlpha,
      ),
    },
  });

  const labelText = formatLabelText(label, props.labelLength);
  const areaText = ObjectUtils.nonNull(area) ? `（面积：${area}cm²）` : "";
  const polyPositions = entity.polygon.hierarchy._value.positions;
  const centerPosition = BoundingSphere.fromPoints(polyPositions).center;
  entity.position = Ellipsoid.WGS84.scaleToGeodeticSurface(centerPosition);
  entity.label = {
    distanceDisplayCondition: ObjectUtils.nonNull(props.minPolygonDisplayHeight)
      ? new DistanceDisplayCondition(0, props.minPolygonDisplayHeight)
      : undefined,
    text: labelText + areaText,
    color: Color.fromCssColorString(props.labelColor),
    font: `${props.labelFontSize}px ${props.labelFontFamily}`,
  };

  if (ObjectUtils.nonNull(entity?.id)) {
    entityDataMap.set(entity.id, data);
  }
  return entity;
};
const renderBillboard = (
  longitude,
  latitude,
  image,
  label,
  flyTo = false,
  data = null,
) => {
  const entity = viewer.entities.add({
    position: Cartesian3.fromDegrees(longitude, latitude),
    billboard: {
      image: image,
      verticalOrigin: VerticalOrigin.BOTTOM,
    },
    description: data,
  });
  if (StringUtils.isNotBlank(label)) {
    entity.label = {
      text: formatLabelText(label, props.labelLength),
      font: `${props.labelFontSize}px ${props.labelFontFamily}`,
      horizontalOrigin: HorizontalOrigin.CENTER,
      pixelOffset: new Cartesian2(0.0, 10),
    };
  }
  if (flyTo) {
    viewer.flyTo(entity);
  }
  return entity;
};
const renderEntities = (data) => {
  for (let item of data) {
    switch (item[props.typeField]) {
      case POLYLINE_TYPE:
        renderPolyline(
          item[props.coordinatesField],
          item[props.nameField],
          item[props.polylineDistanceField],
          item,
        );
        break;
      case POLYGON_TYPE:
        renderPolygon(
          item[props.coordinatesField],
          item[props.nameField],
          item[props.polygonAreaField],
          item,
        );
        break;
      default: {
        renderLocation(
          ArrayUtils.get(item[props.coordinatesField], 0)?.longitude,
          ArrayUtils.get(item[props.coordinatesField], 0)?.latitude,
          item[props.nameField],
          item,
        );
        break;
      }
    }
  }
};
const clearEntities = () => {
  entityDataMap.clear();
  locationDataSource.entities.removeAll();
  polylineDataSource.entities.removeAll();
  polygonDataSource.entities.removeAll();
};
const removeEntity = (entity, type = null) => {
  switch (type) {
    case POLYLINE_TYPE:
      polylineDataSource.entities.remove(entity);
      break;
    case POLYGON_TYPE:
      polygonDataSource.entities.remove(entity);
      break;
    case LOCATION_TYPE:
    case SPHERE_LOCATION_TYPE:
      locationDataSource.entities.remove(entity);
      break;
    default:
      viewer.entities.remove(entity);
      break;
  }
};
const flyTo = (longitude, latitude) => {
  viewer.camera.flyTo({
    destination: Cartesian3.fromDegrees(
      longitude,
      latitude,
      Math.max(props.initCameraHeight - 100, 100),
    ),
  });
};
const formatLabelText = (labelText, length) => {
  if (StringUtils.isBlank(labelText)) {
    return StringUtils.EMPTY;
  }
  return labelText.length > length
    ? `${labelText.substring(0, length)}...`
    : labelText;
};

let tmpBillboard = null;
let tmpPositions = [];
let tmpCoordinatesArr = [];
let tmpPoints = [];
let tmpPolylines = [];
let tmpPolygon = null;

const draw = (type) => {
  emits("draw-start");
  destroyDraw();
  destroyHandler();

  // 开启深度检测
  //viewer.scene.globe.depthTestAgainstTerrain = true;
  handler = new ScreenSpaceEventHandler(viewer.scene.canvas);
  switch (type) {
    case POLYLINE_TYPE:
      //鼠标移动事件
      //handler.setInputAction(() => {}, ScreenSpaceEventType.MOUSE_MOVE);
      //左键点击操作
      handler.setInputAction((click) => {
        //调用获取位置信息的接口
        let ray = viewer.camera.getPickRay(click.position);
        let position = viewer.scene.globe.pick(ray, viewer.scene);

        const cartographic = ellipsoid.cartesianToCartographic(position);
        const latitude = Cesium.Math.toDegrees(cartographic.latitude);
        const longitude = Cesium.Math.toDegrees(cartographic.longitude);
        tmpCoordinatesArr.push({ latitude, longitude });
        tmpPositions.push(position);

        //调用绘制点的接口
        const point = drawPoint(position);
        tmpPoints.push(point);
        if (tmpPositions.length > 1) {
          let polyline = drawPolyline([
            tmpPositions[tmpPositions.length - 2],
            tmpPositions[tmpPositions.length - 1],
          ]);
          tmpPolylines.push(polyline);
        }
      }, ScreenSpaceEventType.LEFT_CLICK);
      //右键点击操作
      handler.setInputAction(() => {
        if (tmpPolylines.length > 0) {
          viewer.entities.remove(tmpPolylines.pop());
        }
        if (tmpPoints.length > 0) {
          viewer.entities.remove(tmpPoints.pop());
          tmpPositions.pop();
          tmpCoordinatesArr.pop();
        }
      }, ScreenSpaceEventType.RIGHT_CLICK);
      break;
    case POLYGON_TYPE:
      //鼠标移动事件
      //handler.setInputAction(() => {}, ScreenSpaceEventType.MOUSE_MOVE);
      //左键点击操作
      handler.setInputAction((click) => {
        //调用获取位置信息的接口
        let ray = viewer.camera.getPickRay(click.position);
        let position = viewer.scene.globe.pick(ray, viewer.scene);

        const cartographic = ellipsoid.cartesianToCartographic(position);
        const latitude = Cesium.Math.toDegrees(cartographic.latitude);
        const longitude = Cesium.Math.toDegrees(cartographic.longitude);
        tmpCoordinatesArr.push({ latitude, longitude });
        tmpPositions.push(position);

        //调用绘制点的接口
        let point = drawPoint(position);
        tmpPoints.push(point);
        if (tmpPositions.length > 1) {
          let polyline = drawPolyline([
            tmpPositions[tmpPositions.length - 2],
            tmpPositions[tmpPositions.length - 1],
          ]);
          tmpPolylines.push(polyline);
        }
        if (tmpPositions.length > 2) {
          if (ObjectUtils.nonNull(tmpPolygon)) {
            viewer.entities.remove(tmpPolygon);
          }
          tmpPolygon = drawPolygon(tmpPositions);
        }
      }, ScreenSpaceEventType.LEFT_CLICK);
      //右键点击操作
      handler.setInputAction(() => {
        if (tmpPolylines.length > 0) {
          viewer.entities.remove(tmpPolylines.pop());
        }
        if (tmpPoints.length > 0) {
          viewer.entities.remove(tmpPoints.pop());
          tmpPositions.pop();
          tmpCoordinatesArr.pop();
        }
        if (ObjectUtils.nonNull(tmpPolygon)) {
          viewer.entities.remove(tmpPolygon);
          if (tmpPositions.length > 2) {
            tmpPolygon = drawPolygon(tmpPositions);
          }
        }
      }, ScreenSpaceEventType.RIGHT_CLICK);
      break;
    default:
      // 监听鼠标左键
      handler.setInputAction((event) => {
        // 从相机位置通过windowPosition 世界坐标中的像素创建一条射线。返回Cartesian3射线的位置和方向。
        let ray = viewer.camera.getPickRay(event.position);
        // 查找射线与渲染的地球表面之间的交点。射线必须以世界坐标给出。返回Cartesian3对象
        let position = viewer.scene.globe.pick(ray, viewer.scene);

        const cartographic = ellipsoid.cartesianToCartographic(position);
        const latitude = Cesium.Math.toDegrees(cartographic.latitude);
        const longitude = Cesium.Math.toDegrees(cartographic.longitude);

        if (ObjectUtils.nonNull(tmpBillboard)) {
          viewer.entities.remove(tmpBillboard);
        }
        tmpCoordinatesArr = [{ latitude, longitude }];
        tmpBillboard = drawBillboard(position);
      }, ScreenSpaceEventType.LEFT_CLICK);
      break;
  }
};
const drawBillboard = (position) => {
  return viewer.entities.add({
    position: position,
    billboard: {
      image: props.drawLocationIcon,
      verticalOrigin: VerticalOrigin.BOTTOM,
    },
  });
};
const drawPoint = (position) => {
  return viewer.entities.add({
    position: position,
    point: {
      color: Color.fromCssColorString(props.drawColor),
      pixelSize: props.pointPixelSize,
    },
  });
};
const drawPolyline = (positions) => {
  if (positions.length < 1) {
    return null;
  }
  return viewer.entities.add({
    polyline: {
      positions: positions,
      width: props.polylineWidth,
      material: Color.fromCssColorString(props.drawColor),
    },
  });
};
const drawPolygon = (positions) => {
  if (positions.length < 2) {
    return null;
  }
  return viewer.entities.add({
    polygon: {
      hierarchy: positions,
      material: Color.fromAlpha(
        Color.fromCssColorString(props.drawColor),
        props.polygonAlpha,
      ),
    },
  });
};
const destroyDraw = () => {
  for (let entity of tmpPoints) {
    viewer.entities.remove(entity);
  }
  for (let entity of tmpPolylines) {
    viewer.entities.remove(entity);
  }
  if (ObjectUtils.nonNull(tmpPolygon)) {
    viewer.entities.remove(tmpPolygon);
  }
  if (ObjectUtils.nonNull(tmpBillboard)) {
    viewer.entities.remove(tmpBillboard);
  }

  tmpPositions = [];
  tmpCoordinatesArr = [];
  tmpPoints = [];
  tmpPolylines = [];
  tmpPolygon = null;
  tmpBillboard = null;
};
const saveDraw = (type) => {
  const result = { coordinates: [...tmpCoordinatesArr], type };
  try {
    if (type === POLYLINE_TYPE && tmpPositions.length === 2) {
      result.distance = NumberUtils.toFixed(
        calculateDistance(
          tmpPositions[0],
          tmpPositions[tmpPositions.length - 1],
          viewer,
        ),
        2,
      );
    } else if (type === POLYGON_TYPE) {
      const area = calculateArea(tmpPositions, viewer);
      if (!Number.isNaN(area)) {
        result.area = NumberUtils.toFixed(area, 2);
      }
    }
  } catch (e) {
    console.log(e);
  }

  initHandler();
  tmpPositions = [];
  tmpCoordinatesArr = [];
  return result;
};
const revokeDraw = (type) => {
  switch (type) {
    case POLYLINE_TYPE:
      if (tmpPolylines.length > 0) {
        viewer.entities.remove(tmpPolylines.pop());
      }
      if (tmpPoints.length > 0) {
        viewer.entities.remove(tmpPoints.pop());
        tmpPositions.pop();
        tmpCoordinatesArr.pop();
      }
      break;
    case POLYGON_TYPE:
      if (tmpPolylines.length > 0) {
        viewer.entities.remove(tmpPolylines.pop());
      }
      if (tmpPoints.length > 0) {
        viewer.entities.remove(tmpPoints.pop());
        tmpPositions.pop();
        tmpCoordinatesArr.pop();
      }
      if (ObjectUtils.nonNull(tmpPolygon)) {
        viewer.entities.remove(tmpPolygon);
        if (tmpPositions.length > 2) {
          tmpPolygon = drawPolygon(tmpPositions);
        }
      }
      break;
    default:
      break;
  }
};

defineExpose({
  renderBillboard,
  removeEntity,
  destroyDraw,
  flyTo,
  draw,
  saveDraw,
  revokeDraw,
});
</script>

<template>
  <div class="full-size">
    <div id="cesium-container" class="full-size"></div>
  </div>
</template>

<style lang="less" scoped></style>

<style lang="less">
.cesium-credit-logoContainer {
  display: none !important;
}
</style>

```