```
import {
  Cartesian2,
  Cartographic,
  EllipsoidGeodesic,
  HeightReference,
  SceneTransforms,
} from "cesium";
import * as turf from "@turf/turf";
import { randomPoint } from "@turf/random";
import voronoi from "@turf/voronoi";

/**
 * 计算线段的表面距离
 * @param startPosition  -线段起点的世界坐标
 * @param endPosition    -线段终点的世界坐标
 * @param viewer
 * @returns {number} 表面距离
 */
export function calculateDistance(startPosition, endPosition, viewer) {
  let resultDistance = 0;
  const startPoint = SceneTransforms.worldToWindowCoordinates(
    viewer.scene,
    startPosition,
  );
  const endPoint = SceneTransforms.worldToWindowCoordinates(
    viewer.scene,
    endPosition,
  );
  if (typeof startPoint === "undefined" || typeof endPoint === "undefined") {
    return Number.NaN;
  }
  const sampleWindowPoints = [startPosition];
  const length = Math.sqrt(
    Math.pow(endPoint.x - startPoint.x, 2) +
      Math.pow(endPoint.y - startPoint.y, 2),
  );
  for (let ii = 50; ii <= length; ii++) {
    const tempPoint = findWindowPositionByPixelInterval(
      startPoint,
      endPoint,
      ii,
    );
    const tempPosition = viewer.scene.globe.pick(
      viewer.camera.getPickRay(tempPoint),
      viewer.scene,
    );
    if (tempPosition) {
      sampleWindowPoints.push(tempPosition);
    }
  }
  sampleWindowPoints.push(endPosition);
  for (let jj = 0; jj < sampleWindowPoints.length - 1; jj++) {
    resultDistance += calculateDetailSurfaceLength(
      sampleWindowPoints[jj + 1],
      sampleWindowPoints[jj],
    );
  }
  return resultDistance;
}

/**
 * 获取线段上距起点一定距离出的线上点坐标（屏幕坐标）
 * @param startPosition  -线段起点（屏幕坐标）
 * @param endPosition -线段终点（屏幕坐标）
 * @param interval -距起点距离
 * @returns {Cartesian2} 结果坐标（屏幕坐标）
 */
function findWindowPositionByPixelInterval(
  startPosition,
  endPosition,
  interval,
) {
  let result = new Cartesian2(0, 0);
  const length = Math.sqrt(
    Math.pow(endPosition.x - startPosition.x, 2) +
      Math.pow(endPosition.y - startPosition.y, 2),
  );
  if (length < interval) {
    return result;
  } else {
    const x =
      (interval / length) * (endPosition.x - startPosition.x) + startPosition.x;
    const y =
      (interval / length) * (endPosition.y - startPosition.y) + startPosition.y;
    result.x = x;
    result.y = y;
  }
  return result;
}

/**
 * 计算细分后的，每一小段的笛卡尔坐标距离（也就是大地坐标系距离）
 * @param startPosition -每一段线段起点
 * @param endPosition -每一段线段终点
 * @returns {number} 表面距离
 */
function calculateDetailSurfaceLength(startPosition, endPosition) {
  let innerS = 0;
  const cartographicStart = Cartographic.fromCartesian(startPosition);
  const cartographicEnd = Cartographic.fromCartesian(endPosition);
  const geoD = new EllipsoidGeodesic();
  geoD.setEndPoints(cartographicStart, cartographicEnd);
  innerS = geoD.surfaceDistance;
  innerS = Math.sqrt(
    Math.pow(innerS, 2) +
      Math.pow(cartographicStart.height - cartographicEnd.height, 2),
  );
  return innerS;
}

/**
 * 计算线段的表面面积
 * @param positions 区域坐标集合
 * @param viewer
 * @returns {number} 面积
 */
export function calculateArea(positions, viewer) {
  const windowCoordinates = positions.map((position) =>
    SceneTransforms.worldToWindowCoordinates(viewer.scene, position),
  );
  let result = 0;
  const bounds = getBounds(windowCoordinates);
  const points = randomPoint(50, {
    bbox: [bounds[0], bounds[1], bounds[2], bounds[3]],
  });
  const mainPoly = cartesian2ToTurfPolygon(windowCoordinates);
  const voronoiPolygons = voronoi(points, {
    bbox: [bounds[0], bounds[1], bounds[2], bounds[3]],
  });
  voronoiPolygons.features.forEach((element) => {
    const intersectPoints = intersect(mainPoly, element.geometry);
    result += calculateDetailSurfaceArea(intersectPoints, viewer);
  });
  return result;
}

/**
 * 计算细分后的，每一个三角形的面积
 * @param positions 区域坐标集合
 * @param viewer
 * @returns {number} 面积
 */
function calculateDetailSurfaceArea(positions, viewer) {
  let worldPositions = [];
  positions.forEach((element) => {
    worldPositions.push(pickCartesian(viewer, element).cartesian);
  });
  return getArea(worldPositions);
}

function getArea(positions) {
  const x = [0];
  const y = [0];
  const geodesic = new EllipsoidGeodesic();
  //角度转化为弧度(rad)
  const radiansPerDegree = Math.PI / 180.0;
  if (positions.length > 0) {
    //数组x,y分别按顺序存储各点的横、纵坐标值
    for (let i = 0; i < positions.length - 1; i++) {
      const p1 = positions[i];
      const p2 = positions[i + 1];
      const point1cartographic = Cartographic.fromCartesian(p1);
      const point2cartographic = Cartographic.fromCartesian(p2);
      geodesic.setEndPoints(point1cartographic, point2cartographic);
      const s = Math.sqrt(
        Math.pow(geodesic.surfaceDistance, 2) +
          Math.pow(point2cartographic.height - point1cartographic.height, 2),
      );
      const lat1 = point2cartographic.latitude * radiansPerDegree;
      const lon1 = point2cartographic.longitude * radiansPerDegree;
      const lat2 = point1cartographic.latitude * radiansPerDegree;
      const lon2 = point1cartographic.longitude * radiansPerDegree;
      let angle = -Math.atan2(
        Math.sin(lon1 - lon2) * Math.cos(lat2),
        Math.cos(lat1) * Math.sin(lat2) -
          Math.sin(lat1) * Math.cos(lat2) * Math.cos(lon1 - lon2),
      );
      if (angle < 0) {
        angle += Math.PI * 2.0;
      }

      y.push(Math.sin(angle) * s + y[i]);
      x.push(Math.cos(angle) * s + x[i]);
    }

    let sum = 0;
    for (let i = 0; i < x.length - 1; i++) {
      sum += x[i] * y[i + 1] - x[i + 1] * y[i];
    }

    return Math.abs(sum + x[x.length - 1] * y[0] - x[0] * y[y.length - 1]) / 2;
  }
  return 0;
}

function pickCartesian(viewer, windowPosition) {
  //根据窗口坐标，从场景的深度缓冲区中拾取相应的位置，返回笛卡尔坐标。
  const cartesianModel = viewer.scene.pickPosition(windowPosition);

  //场景相机向指定的鼠标位置（屏幕坐标）发射射线
  const ray = viewer.camera.getPickRay(windowPosition);
  //获取射线与三维球相交的点（即该鼠标位置对应的三维球坐标点，因为模型不属于球面的物体，所以无法捕捉模型表面）
  const cartesianTerrain = viewer.scene.globe.pick(ray, viewer.scene);

  const result = {};
  if (
    typeof cartesianModel !== "undefined" ||
    typeof cartesianTerrain !== "undefined"
  ) {
    result.cartesian = cartesianModel || cartesianTerrain;
    result.CartesianModel = cartesianModel;
    result.cartesianTerrain = cartesianTerrain;
    result.windowCoordinates = windowPosition.clone();
    //坐标不一致，证明是模型，采用绝对高度。否则是地形，用贴地模式。
    result.altitudeMode =
      cartesianModel?.z.toFixed(0) !== cartesianTerrain?.z.toFixed(0)
        ? HeightReference.NONE
        : HeightReference.CLAMP_TO_GROUND;
  }
  return result;
}

function getBounds(points) {
  let bounds = [];
  let left = Number.MAX_VALUE;
  let right = Number.MIN_VALUE;
  let top = Number.MAX_VALUE;
  let bottom = Number.MIN_VALUE;
  points.forEach((element) => {
    left = Math.min(left, element.x);
    right = Math.max(right, element.x);
    top = Math.min(top, element.y);
    bottom = Math.max(bottom, element.y);
  });
  bounds.push(left);
  bounds.push(top);
  bounds.push(right);
  bounds.push(bottom);
  return bounds;
}

function cartesian2ToTurfPolygon(positions) {
  const coordinates = [[]];
  positions.forEach((element) => {
    coordinates[0].push([element.x, element.y]);
  });
  coordinates[0].push([positions[0].x, positions[0].y]);
  const polygon = turf.polygon(coordinates);
  return polygon.geometry;
}

function intersect(poly1, poly2) {
  const intersection = turf.intersect(poly1, poly2);
  if (intersection?.geometry !== undefined) {
    return turfPolygonToCartesian2Arr(intersection?.geometry);
  } else {
    return [];
  }
}

function turfPolygonToCartesian2Arr(geometry) {
  const positionArr = [];
  geometry.coordinates.forEach((pointArr) => {
    pointArr.forEach((point) => {
      positionArr.push(new Cartesian2(point[0], point[1]));
    });
  });
  positionArr.push(
    new Cartesian2(
      geometry.coordinates[0][0][0],
      geometry.coordinates[0][0][1],
    ),
  );
  return positionArr;
}

```