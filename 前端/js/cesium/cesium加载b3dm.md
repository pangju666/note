```javascript
const tileset = await Cesium3DTileset.fromUrl("http://172.16.0.14/Scene/Production_6.json",
    {
        maximumScreenSpaceError: 1, // 值越小初始层级越高，最小值为0
    }
);
viewer?.scene.primitives.add(tileset);
viewer?.zoomTo(tileset, new HeadingPitchRange(
    Cesium.Math.toRadians(0),
    Cesium.Math.toRadians(-90),
));
```