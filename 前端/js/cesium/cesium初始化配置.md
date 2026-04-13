```
viewer = new Viewer("cesium-container", {
    animation: false, //左下角的动画仪表盘
    baseLayerPicker: false, //右上角的图层选择按钮
    geocoder: false, //geo编码器
    homeButton: false, //home按钮
    sceneModePicker: false, //模式切换按钮
    timeline: false, //底部的时间轴
    navigationHelpButton: false, //右上角的帮助按钮，
    fullscreenButton: false, //右下角的全屏按钮
    selectionIndicator: false, //原生自带绿色选择框，双击显示的绿框
    infoBox: false, //点击要素之后显示的信息窗口
    sceneMode: SceneMode.SCENE2D,
    useBrowserRecommendedResolution: true, // 启用高分辨率支持
    orderIndependentTranslucency: false, // 禁用与透明相关的深度排序
    // 启用透明背景
    contextOptions: {
        webgl: {
            alpha: true // 启用 WebGL 的透明支持
        }
    },
    skyBox: false, // 禁用天空盒
    skyAtmosphere: false,   // 禁用大气层效果
    baseLayer: false, // 禁用基础图层
});

// 隐藏球体
viewer.scene.globe.show = false;
// 设置球体颜色
viewer.scene.globe.baseColor = Color.TRANSPARENT;
// 去除场景背景
viewer.scene.backgroundColor = Color.TRANSPARENT;

// 禁止鼠标操作相机
const cameraController = viewer.scene.screenSpaceCameraController;
cameraController.enableInputs = false;

// 禁止双击缩放
const handler = viewer.screenSpaceEventHandler;
handler.removeInputAction(Cesium.ScreenSpaceEventType.LEFT_DOUBLE_CLICK);
```