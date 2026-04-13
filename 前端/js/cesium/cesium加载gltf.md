```javascript
const model = await Model.fromGltfAsync({
    url: "
    modelMatrix: Transforms.eastNorthUpToFixedFrame(
        Cartesian3.fromDegrees(85.0, 30.0, 4000)  // 模型经纬度 + 高程
    ),
    id: "xizang-dem",
    scale: 1.0,
    //minimumPixelSize: 128,  // 保证模型远处可见
    //maximumScale: 20000     // 限制最大缩放
});
viewer.scene.primitives.add(model);
model.readyEvent.addEventListener(() => {
    viewer.camera.setView({
        destination: model.boundingSphere.center,
        orientation: {
            heading: Cesium.Math.toRadians(270),
            pitch: Cesium.Math.toRadians(-45),
            roll: 0
        }
    });

    // 推远一定距离
    const range = model.boundingSphere.radius * 1.9;
    viewer.camera.moveBackward(range);
```