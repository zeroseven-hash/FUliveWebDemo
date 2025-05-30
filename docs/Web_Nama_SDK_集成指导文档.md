# FULiveDemo集成指导文档
# 目录结构
```typescript
├─ nginx // 前端静态服务器配置 用于docker nginx image
├─ public  // 公共静态资源
├─ src 
│   ├─ asset // svg、icon等图片资源
│   ├─ common // 全局的config与变量配置、less全局变量
│   ├─ components // 子组件
│   ├─ Routers // view 页面级组件
│   │   ├─ Agora 声网demo
│   │   ├─ Home 首页
│   │   │   ├─ Mobile.tsx // 移动端页面
│   │   │   ├─ PC.tsx // 电脑端页面
│   ├─ store 全局状态管理state
│   ├─ utils
│   │   ├─ namaSDK //namasdk
│   │   │   ├─ SIMD // 支持SIMD的wasm sdk
│   │   │   ├─ funamawebassembly.js // 不支持SIMD的sdk入口文件
│   │   │   ├─ index.ts // NAMASDK class definded
│   │   │   ├─ polyfills.ts // 补丁
│   │   │   ├─ shared.js // sdk 渲染逻辑上层封装
│   │   │   ├─ worker.js // sdk 启用worker多线程
│   │   ├─ hooks.ts // react 公共hooks
│   │   ├─ tools.ts // 辅助函数
│   ├─ main.tsx // react 入口
│   ├─ router.tsx // react-router 
├─ .gitlab-ci.yml // gitlab ci
├─ index.html
├─ package.json
├─ vite.config.ts // vite 构建工具配置
```
## 关键资源文件
#### 1. 美颜道具：用于加载美颜功能
https://fu-sdk.oss-cn-hangzhou.aliyuncs.com/WebDemo/face_beautification.bundle
#### 2. ai算法道具：用于加载ai算法能力
https://fu-sdk.oss-cn-hangzhou.aliyuncs.com/WebDemo/ai_face_processor_pc.bundle
https://fu-sdk.oss-cn-hangzhou.aliyuncs.com/WebDemo/ai_face_processor.bundle

## 功能集成
namasdk可以作为module被导入，其调用在src/utils/namaSDK/shared.js当中，可以在woker中使用该js，或是在主线程调用。

### 1. 加载sdk
sdk分为simd版本以及非simd版本，simd版本有更好的性能，但是兼容性较差，低版本的浏览器支持不佳。非simd的版本具有更好的兼容性。建议在此处使用fallback逻辑。
```typescript
// src/utils/namaSDK/shared.js
async loadModule({ auth, typeSDK }) {
    let loadDir = typeSDK == "LOW" ? "" : "SIMD/";
    try {
      // 白名单直接进入低版本wasm
      let namaWasm = null;
      if (loadDir) {
        namaWasm = (await import(`./SIMD/funamawebassembly.js`))?.default;
      } else {
        namaWasm = (await import(`./funamawebassembly.js`))?.default;
      }
      this.module = await namaWasm?.();
    } catch (e) {
      console.error("load SIMD/funamawebassembly fail", e);
      if (loadDir) {
        // simd 失败就降级
        const namaWasm = (await import("./funamawebassembly.js"))?.default;
        this.module = await namaWasm();
      }
    }
    
    // 初始化sdk，传入鉴权信息
    var step_res = this.module.fuSetup(auth);
    this.isLoaded = true;
    this.module.fuSetLogLevel(this.module.FULOGLEVEL.FU_LOG_LEVEL_INFO);
    // this.module.fuSetLogLevel(this.module.FULOGLEVEL.FU_LOG_LEVEL_OFF);
    this.getPrepare();
  }
```
我们提供了判断是否使用simd版本的函数，可用于参考：
```typescript
//src/utils/tool.ts
/**
 * 判断是否 不支持Wasm SIMD
 * Chrome < 91
 * Edge < 91
 * Safari < 16.4
 * Firefox < 89
 * @returns boolean
 */
export const notWasmSimdSupported = () => {
  const ua = navigator.userAgent;
  // 正则表达式匹配浏览器名称和版本
  const chromeMatch = ua.match(/Chrome\/(\d+)/);
  const edgeMatch = ua.match(/Edg\/(\d+)/);
  const firefoxMatch = ua.match(/Firefox\/(\d+)/);
  const safariMatch = ua.match(/Version\/([\d.]+).*Safari/i);

  // 检测浏览器版本
  if (chromeMatch && !edgeMatch) {
    const chromeVersion = parseInt(chromeMatch[1], 10);
    return chromeVersion < 91;
  }

  if (edgeMatch) {
    const edgeVersion = parseInt(edgeMatch[1], 10);
    return edgeVersion < 91;
  }

  if (firefoxMatch) {
    const firefoxVersion = parseInt(firefoxMatch[1], 10);
    return firefoxVersion < 89;
  }

  if (safariMatch) {
    const safariVersion = parseFloat(safariMatch[1]); // Safari 的版本可能是小数
    console.log("safariVersion", safariVersion);
    return safariVersion < 16.4;
  }

  // 如果未匹配到任何支持的浏览器，则返回 false
  return false;
};
```
### 2. 加载ai算法能力
使用sdk进行渲染之前，需要提前加载ai算法能力，通过加载ai的bundle来开启ai算法。
```typescript
// src/utils/namaSDK/shared.js
async loadAiBundle(aiBundle) {
    var flag = new this.module.FUAIFACEALGORITHMCONFIG();
    // 根据需要关闭不需要的ai算法，提高加载速度
    flag.value =
      this.module.FUAIFACEALGORITHMCONFIG.FACE_ALGORITHM_ENABLE_ALL.value |
      this.module.FUAIFACEALGORITHMCONFIG.FACE_ALGORITHM_DISABLE_DEL_SPOT
        .value |
      this.module.FUAIFACEALGORITHMCONFIG.FACE_ALGORITHM_DISABLE_ARMESHV2
        .value |
      this.module.FUAIFACEALGORITHMCONFIG.FUAIFACE_DISABLE_RACE.value |
      this.module.FUAIFACEALGORITHMCONFIG.FUAIFACE_DISABLE_LANDMARK_HP_OCCU
        .value;

    this.module.fuSetFaceAlgorithmConfig(flag);
    //加载 ai bundle
    this.module.fuLoadAIModelFromPackage(
      aiBundle,
      this.module.FUAITYPE.FACEPROCESSOR,
    );
    if (!this.module.fuIsAIModelLoaded(this.module.FUAITYPE.FACEPROCESSOR)) {
      console.log("AI model not loaded");
      return;
    }
    console.log("ai bundle loaded");
    this.aiLoaded = true;
    return true;
  }

```

### 3. 加载美颜道具，初始化渲染能力
加载美颜道具，开启美颜能力，设置画布，初始化opengl上下文。
```typescript
 // src/utils/namaSDK/shared.js
 async InitNama({ bundle, canvas }) {
    console.log("InitNama ......");
    // 设置画布
    this.module.canvas = canvas;
    // 在设置画布之后，初始化opengl上下文。上一步为设置画布，此处需要传入画布的id
    this.module.fuInitGLContext("");
    
    //设置当前运行的平台。
    this.module.fuSetPlatform(
      !this.prepare.isPc
        ? this.module.FUPLATFORM.ANDORID
        : this.module.FUPLATFORM.PC,
    );

    // this.module.fuEnableFrameTimeProfile(100, true);
    // call fuDestroyItem() when the item is no longer needed.
    const item = await this.fuCreateItem({
      data: bundle,
      name: beautyBundleName,
    });
    console.log("beauty bundle loaded");
    // 记录美颜道具 的handle
    this.beautyBundle = item;

    // 设置一下最大检测人脸数
    this.module.fuSetMaxFaces(4);
    this.beautyLoaded = true;
  }
```

### 4. 开启渲染
getPrepare函数用于调用上述步骤，每当获取相关参数之后，调用getPrepare，其会根据参数的key和data自动选择上述准备步骤，当所有准备步骤全部完成，并且输入设置完毕，其会调用BeginRender函数开始渲染。
我们封装了简易的渲染器可以接受ImageBitMap,ReadableStream,HTMLVideoElement等作为渲染输入。
```typescript
// src/utils/namaSDK/shared.js
async getPrepare(key, data) {
    if (key) {
      // 没有key就直接检查后续逻辑
      this.prepare[key] = data;
    }
    console.log("run prepare", key, this.prepare, {
      aiInited: this.aiInited,
      inited: this.inited,
      beautyLoaded: this.beautyLoaded,
      aiLoaded: this.aiLoaded,
      isLoaded: this.isLoaded,
    });
    if (!this.isLoaded) return;
    if (this.prepare.aiBundle && !this.aiInited) {
      this.aiInited = true;
      await this.loadAiBundle(this.prepare.aiBundle);
    }
    if (this.prepare.beautyBundle && this.prepare.offscreen) {
      if (!this.inited) {
        this.inited = true;
        await this.InitNama({
          bundle: this.prepare.beautyBundle,
          canvas: this.prepare.offscreen,
        });
      }
      // 会有case 其他都好了 aiload的逻辑还没走，但是渲染要在ai之后
      if (
        this.inited &&
        (this.prepare.videoStream || this.prepare.imageBitmap) &&
        this.aiLoaded &&
        this.beautyLoaded
      ) {
        var pipe = undefined;
        if (this.isUpload) {
          if (this.prepare.imageBitmap || this.prepare.videoStream) {
            pipe = ImageVideoRenderPipe;
          }
        } else if (this.isAgora) {
          if (this.prepare.writableStream && this.prepare.videoStream) {
            pipe = AgoraRenderPipe;
          }
        } else if (this.prepare.videoStream) {
          pipe = NamaRenderPipe;
        }
        if (pipe != undefined) {
          this.BeginRender(pipe);
        }
      }
    }

    if (this.aiLoaded && this.beautyLoaded) {
      if (this.isWorker) {
        postMessage({ type: "inited" }); // 触发默认美颜时机
      } else {
        this.onInited?.();
      }
    }
  }

```
BegenRender函数根据渲染的输入类型，选择相应的渲染器。
```typescript
  // src/utils/namaSDK/shared.js
  BeginRender(pipe) {
    const that = this;
    // render image
    if (this.renderer) {
      this.renderer.destroy();
      this.renderer = null;
    }

    if (pipe === ImageVideoRenderPipe) {
      this.renderer = new ImageRenderer(this.module);
      if (this.prepare.imageBitmap) {
        this.renderer.setImageBuffer(
          this.prepare.imageBitmap,
          this.prepare.imageBitmap.width,
          this.prepare.imageBitmap.height,
          this.flipHorizontal,
        );
        this.renderer.setFps(30);
      } else if (this.prepare.videoStream) {
        this.renderer.setImageBuffer(
          this.prepare.videoStream,
          this.prepare.videoStream.videoWidth,
          this.prepare.videoStream.videoHeight,
          false,
        );
        // for now, use 30 fps to display video;
        this.renderer.setFps(30);
      }
    } else if (pipe === NamaRenderPipe) {
      var stream = undefined;
      var type = undefined;
      if (this.prepare.videoStream instanceof ReadableStream) {
        this.renderer = new StreamRenderer(this.module);
        stream = this.prepare.videoStream;
        this.renderer.setStream(stream, this.isWorker,this.flipHorizontal);
      } else if (this.prepare.videoStream instanceof HTMLVideoElement) {
        this.renderer = new ImageRenderer(this.module);
        this.renderer.setImageBuffer(
          this.prepare.videoStream,
          this.prepare.videoStream.videoWidth,
          this.prepare.videoStream.videoHeight,
          this.flipHorizontal,
        );
        var track = this.prepare.videoStream.srcObject.getVideoTracks()[0];
        var fps = track.getSettings().frameRate;
        this.renderer.setFps(fps);
      } else {
        console.log("unknow stream not support");
        return;
      }
    } else if (pipe === AgoraRenderPipe) {
      this.renderer = new StreamRenderer(this.module);
      stream = this.prepare.videoStream;
      this.renderer.setStream(stream,this.isWorker, this.flipHorizontal);
      if (this.prepare.writableStream) {
        this.renderer.setWritableStream(
          this.prepare.writableStream,
          !this.isPureAgora,
        );
      }
    } else {
      console.log("unknow pipe not support");
      return;
    }

    this.renderer.setOnFpsUpdate((fps, rtt) => {
      that.OnUpdateFPS(fps, rtt);
    });
    this.renderer.setOnBeforeRender(() => {
      that.OnBeforeRender();
      that.cppItems[0] = that.beautyBundle ? that.beautyBundle : 0;
      that.cppItems[1] = that.stickerBundle ? that.stickerBundle : 0;
      that.renderer.setRenderList(
        that.useEmpty ? that.emptyItems : that.cppItems,
      );
      that.renderer.setViewport(
        0,
        0,
        that.module.canvas.width,
        that.module.canvas.height,
      );
    });
    this.renderer.setOnAfterRender(() => {
      that.OnAfterRender();
    });
    this.renderer.beginRender();
  }
}
```
渲染接口：渲染器是对渲染接口的简易封装。namasdk的核心渲染接口有两个
fuRenderTextureToCanvas(...)
fuRenderBufferToCanvas(....)
分别对应buffer输入以及webglTexture输入，在参数中输入贴纸以及美颜道具，该接口会将输入经过渲染，输出到指定的canvas上（初始化上下文时指定的canvas）。通过这两个接口，用户可根据需求自行拓展修改渲染器的实现。

### 5. 切换相机，切换输入或停止渲染
如果需要更换相机流，或者停止渲染。调用resetStream(),如果需要重启渲染，再次调用getPrepare开启渲染即可
```typescript
  // src/utils/namaSDK/shared.js
 async resetStream() {
    if (this.renderer)
    {
      this.renderer.clearCanvas();
      this.renderer.destroy();
    }

    if (this.abort_controller) {
      this.abort_controller.abort();
      this.abort_controller = null;
    }
    if (this.textureId) {
      this.module.fuUnRegisterNativeTexture(this.textureId);
      this.textureId = null;
    }
    if (this.requestAnimationId) {
      cancelAnimationFrame(this.requestAnimationId);
    }
    this.prepare.videoStream = null;
    this.prepare.writableStream = null;
    this.prepare.imageBitmap = null;
  }
```
### 6.切换贴纸
切换贴纸需要将贴纸bundle加载，随后调用setSticker，该接口将会加载贴纸道具，生成对应的handle并且输入到fuRender函数中。
```typescript
   // src/utils/namaSDK/shared.js
  async setSticker({ key, bundle }) {
    if (this.loadingSticker) {
      console.log("Sticker:loading, ignore", key);
      return;
    }
    this.changeCreatingStatus(key);
    this.toRenderSticker = { key, bundle };
    const item = await this.fuCreateItem({ data: bundle, name: key });
    if (item) {
      this.module.fuItemSetParamd(item, "rotationMode", 2);
    }
    var currentSticker = this.currentSticker;
    this.currentSticker = this.toRenderSticker;
    this.stickerBundle = item;
    this.toRenderSticker = null;
    if (currentSticker.key) {
      await this.destoryItem(currentSticker.key);
    }
    this.changeCreatingStatus("");
  }
```
### 7.美颜道具参数设置
```typescript
// src/utils/namaSDK/shared.js
// SetParams设置参数，bundleName为道具名，type为menuKey中的值，effectName为参数名
// value为参数值
export enum menuKey {
  beauty,
  sticker,
  effects,
  shaping,
  filter,
}
setParams({ bundleName, type, effectName, value }) {
  if (!this.module) {
    console.error("namaSDK not inited!");
    return;
  }
  const res = this.bundleMap[bundleName];
  if (!res) {
    console.error("fuItem not found!");
    return;
  }
  const item = res;
  if (type === 4) {
    this.module.fuItemSetParams(item, "filter_name", effectName);
    this.module.fuItemSetParamd(item, "filter_level", value);
  } else {
    this.module.fuItemSetParamd(item, effectName, value);
  }
  console.log("set param", effectName, value);
}

```
美颜道具中效果参数以及滤镜参数，默认值参考如下：
```typescript
// src/commom/defaultConfig.tsx
const effectsList: TabOptionItem[] = [
  {
    key: "磨皮",
    label: "磨皮",
    range: [0, 100],
    PcIcon: <PcMopi />,
    MobIcons: [MobMopi, MobMopiChanges, MobMopiActive, MobMopiChangesActive],
    fuKey: "blur_level",
    rawInitValue: 6,
    rawRange: [0, 6],
    reflexType: 3,
    initValue: 55,
  },
  {
    key: "美白",
    label: "美白",
    range: [0, 100],
    PcIcon: <Meibai />,
    MobIcons: [
      MobMeibai,
      MobMeibaiChanges,
      MobMeibaiActive,
      MobMeibaiChangesActive,
    ],
    fuKey: "color_level_mode2",
    rawInitValue: 0.2,
    rawRange: [0, 1],
    initValue: 40,
  },
  {
    key: "红润",
    label: "红润",
    range: [0, 100],
    PcIcon: <Hongrun />,
    MobIcons: [
      MobHongrun,
      MobHongrunChanges,
      MobHongrunActive,
      MobHongrunChangesActive,
    ],
    fuKey: "red_level",
    rawInitValue: 0.5,
    rawRange: [0, 1],
    initValue: 30,
  },
  {
    key: "清晰",
    label: "清晰",
    range: [0, 100],
    PcIcon: <Qingxi />,
    MobIcons: [
      MobQingxi,
      MobQingxiChanges,
      MobQingxiActive,
      MobQingxiChangesActive,
    ],
    fuKey: "clarity",
    rawInitValue: 0,
    rawRange: [0, 1],
    initValue: 0,
  },
  {
    key: "锐化",
    label: "锐化",
    range: [0, 100],
    PcIcon: <Ruihua />,
    MobIcons: [
      MobRuihua,
      MobRuihuaChanges,
      MobRuihuaActive,
      MobRuihuaChangesActive,
    ],
    fuKey: "sharpen",
    rawInitValue: 0.2,
    rawRange: [0, 1],
    initValue: 60,
  },
  {
    key: "五官立体",
    label: "五官立体",
    range: [0, 100],
    PcIcon: <Wuguanliti />,
    MobIcons: [MobWglt, MobWgltChanges, MobWgltActive, MobWgltChangesActive],
    fuKey: "face_threed",
    rawInitValue: 0,
    rawRange: [0, 1],
    initValue: 40,
  },
  {
    key: "亮眼",
    label: "亮眼",
    range: [0, 100],
    PcIcon: <Liangyan />,
    MobIcons: [MobLy, MobLyChanges, MobLyActive, MobLyChangesActive],
    fuKey: "eye_bright",
    rawInitValue: 1,
    rawRange: [0, 1],
    initValue: 30,
  },
  {
    key: "美牙",
    label: "美牙",
    range: [0, 100],
    PcIcon: <Meiya />,
    MobIcons: [
      MobMeiya,
      MobMeiyaChanges,
      MobMeiyaActive,
      MobMeiyaChangesActive,
    ],
    fuKey: "tooth_whiten",
    rawInitValue: 1,
    rawRange: [0, 1],
    initValue: 0,
  },
  {
    key: "去黑眼圈",
    label: "去黑眼圈",
    range: [0, 100],
    PcIcon: <Quheiyanquan />,
    MobIcons: [MobQhyq, MobQhyqChanges, MobQhyqActive, MobQhyqChangesActive],
    fuKey: "",
    rawInitValue: 0,
    rawRange: [0, 1],
    initValue: 40,
  },
  {
    key: "去法令纹",
    label: "去法令纹",
    range: [0, 100],
    PcIcon: <Qufalingwen />,
    MobIcons: [MobQflw, MobQflwChanges, MobQflwActive, MobQflwChangesActive],
    fuKey: "",
    rawInitValue: 0,
    rawRange: [0, 1],
    initValue: 80,
  },
];

const shapingList: TabOptionItem[] = [
  {
    key: "瘦脸",
    label: "瘦脸",
    range: [0, 100],
    PcIcon: (
      <img
        src={shoulian}
        style={{ width: 28, height: 28, objectFit: "cover" }}
      />
    ),
    MobIcons: [
      MobShoulian,
      MobShoulianChanges,
      MobShoulianActive,
      MobShoulianChangesActive,
    ],
    fuKey: "cheek_thinning_mode2",
    rawInitValue: 0,
    rawRange: [0, 1],
    initValue: 0,
  },
  {
    key: "V脸",
    label: "V脸",
    range: [0, 100],
    PcIcon: <Vlian />,
    MobIcons: [
      MobVlian,
      MobVlianChanges,
      MobVlianActive,
      MobVlianChangesActive,
    ],
    fuKey: "cheek_v",
    rawInitValue: 0,
    rawRange: [0, 1],
    initValue: 50,
  },
  {
    key: "窄脸",
    label: "窄脸",
    range: [0, 100],
    PcIcon: <Zhailian />,
    MobIcons: [
      MobZhailian,
      MobZhailianChanges,
      MobZhailianActive,
      MobZhalianChangesActive,
    ],
    fuKey: "cheek_narrow_mode2",
    rawInitValue: 0,
    rawRange: [0, 1],
    initValue: 0,
  },
  {
    key: "短脸",
    label: "短脸",
    range: [0, 100],
    PcIcon: <Duanlian />,
    MobIcons: [
      MobDuanlian,
      MobDuanlianChanges,
      MobDuanlianActive,
      MobDuanlianChangesActive,
    ],
    fuKey: "cheek_short",
    rawInitValue: 0,
    rawRange: [0, 1],
    initValue: 0,
  },
  {
    key: "小脸",
    label: "小脸",
    range: [0, 100],
    PcIcon: <Xiaolian />,
    MobIcons: [
      MobXiaolian,
      MobXianlianChanges,
      MobXiaolianActive,
      MobXiaolianChangesActive,
    ],
    fuKey: "cheek_small_mode2",
    rawInitValue: 0,
    rawRange: [0, 1],
    initValue: 0,
  },
  {
    key: "瘦颧骨",
    label: "瘦颧骨",
    range: [0, 100],
    PcIcon: <Shouegu />,
    MobIcons: [
      MobShoueg,
      MobShouegChange,
      MobShouegActive,
      MobShouegChangesActive,
    ],
    fuKey: "intensity_cheekbones",
    rawInitValue: 0,
    rawRange: [0, 1],
    initValue: 0,
  },
  {
    key: "瘦下颌骨",
    label: "瘦下颌骨",
    range: [0, 100],
    PcIcon: <Shouxiaegu />,
    MobIcons: [
      MobShouxeg,
      MobShouxegChanges,
      MobShouxegActive,
      MobShouxegChangesActive,
    ],
    fuKey: "intensity_lower_jaw",
    rawInitValue: 0,
    rawRange: [0, 1],
    initValue: 10,
  },
  {
    key: "大眼",
    label: "大眼",
    range: [0, 100],
    PcIcon: <Dayan />,
    MobIcons: [MobDayan, MobDayanChanges, MobDayanActive, MobDayanChangeActive],
    fuKey: "",
    rawInitValue: 0.5,
    rawRange: [0, 1],
    initValue: 40,
  },
  {
    key: "圆眼",
    label: "圆眼",
    range: [0, 100],
    PcIcon: <Yuanyan />,
    MobIcons: [
      MobYuanyan,
      MobYuanyanChanges,
      MobYuanyanActive,
      MobYuanyanChangeActive,
    ],
    fuKey: "intensity_eye_circle",
    rawInitValue: 0,
    rawRange: [0, 1],
    initValue: 0,
  },
  {
    key: "下巴",
    label: "下巴",
    range: [-50, 50],
    PcIcon: <Xiaba />,
    MobIcons: [MobXb, MobXbChanges, MobXbActive, MobXbChangeActive],
    fuKey: "intensity_chin_mode2",
    rawInitValue: 0.5,
    rawRange: [0, 1],
    reflexType: 2,
    initValue: 0,
  },
  {
    key: "额头",
    label: "额头",
    range: [-50, 50],
    PcIcon: <Etou />,
    MobIcons: [MobEt, MobEtChanges, MobEtActive, MobEtChangesActive],
    fuKey: "intensity_forehead_mode2",
    rawInitValue: 0.5,
    rawRange: [0, 1],
    reflexType: 2,
    initValue: 0,
  },
  {
    key: "瘦鼻",
    label: "瘦鼻",
    range: [0, 100],
    PcIcon: <Shoubi />,
    MobIcons: [MobSb, MobSbChanges, MobSbActive, MobSbChangesActive],
    fuKey: "intensity_nose_mode2",
    rawInitValue: 0,
    rawRange: [0, 1],
    initValue: 50,
  },
  {
    key: "嘴型",
    label: "嘴型",
    range: [-50, 50],
    PcIcon: <Zuixing />,
    MobIcons: [MobZx, MobZxChanges, MobZxActive, MobZxChangesActive],
    fuKey: "",
    rawInitValue: 0.5,
    rawRange: [0, 1],
    reflexType: 2,
    initValue: 0,
  },
  {
    key: "嘴唇厚度",
    label: "嘴唇厚度",
    range: [-50, 50],
    PcIcon: <Zuibahoudu />,
    MobIcons: [MobZchd, MobZchdChanges, MobZchdActive, MobZchdChangesActive],
    fuKey: "intensity_lip_thick",
    rawInitValue: 0.5,
    rawRange: [0, 1],
    reflexType: 2,
    initValue: 0,
  },
  {
    key: "眼睛位置",
    label: "眼睛位置",
    range: [-50, 50],
    PcIcon: <Yanjingweizhi />,
    MobIcons: [MobYjwz, MobYjwzChanges, MobYjwzActive, MobYjwzChangesActive],
    fuKey: "intensity_eye_height",
    rawInitValue: 0.5,
    rawRange: [0, 1],
    reflexType: 2,
    initValue: 0,
  },
  {
    key: "开眼角",
    label: "开眼角",
    range: [0, 100],
    PcIcon: <Kaiyanjiao />,
    MobIcons: [MobKyj, MobKyjChanges, MobKyjActive, MobKyjChangesActive],
    fuKey: "intensity_canthus",
    rawInitValue: 0,
    rawRange: [0, 1],
    initValue: 0,
  },
  {
    key: "眼睑下至",
    label: "眼睑下至",
    range: [0, 100],
    PcIcon: <Yanjianxiazhi />,
    MobIcons: [MobYjxz, MobYjxzChanges, MobYjxzActive, MobYjxzChangesActive],
    fuKey: "intensity_eye_lid",
    rawInitValue: 0,
    rawRange: [0, 1],
    initValue: 0,
  },
  {
    key: "眼距",
    label: "眼距",
    range: [-50, 50],
    PcIcon: <Yanju />,
    MobIcons: [MobYj, MobYjChanges, MobYjActive, MobYjChangesActive],
    fuKey: "intensity_eye_space",
    rawInitValue: 0.5,
    rawRange: [0, 1],
    reflexType: 2,
    initValue: 0,
  },
  {
    key: "眼睛角度",
    label: "眼睛角度",
    range: [-50, 50],
    PcIcon: <Yanjingjiaodu />,
    MobIcons: [MobYjjd, MobYjjdChanges, MobYjjdActive, MobYjjdChangesActive],
    fuKey: "intensity_eye_rotate",
    rawInitValue: 0.5,
    rawRange: [0, 1],
    reflexType: 2,
    initValue: 0,
  },
  {
    key: "长鼻",
    label: "长鼻",
    range: [-50, 50],
    PcIcon: <Changbi />,
    MobIcons: [MobCb, MobCbChanges, MobCbActive, MobCbChangesActive],
    fuKey: "intensity_long_nose",
    rawInitValue: 0.5,
    rawRange: [0, 1],
    reflexType: 2,
    initValue: 0,
  },
  {
    key: "缩人中",
    label: "缩人中",
    range: [-50, 50],
    PcIcon: <Suorenzhong />,
    MobIcons: [MobSrz, MobSrzChanges, MobSrzActive, MobSrzChangesActive],
    fuKey: "intensity_philtrum",
    rawInitValue: 0.5,
    rawRange: [0, 1],
    reflexType: 2,
    initValue: 0,
  },
  {
    key: "微笑嘴角",
    label: "微笑嘴角",
    range: [0, 100],
    PcIcon: <Weixiaozuijiao />,
    MobIcons: [MobWxzj, MobWxzjChanges, MobWxzjActive, MobWxzjChangesActive],
    fuKey: "intensity_smile",
    rawInitValue: 0,
    rawRange: [0, 1],
    initValue: 35,
  },
  {
    key: "眉毛上下",
    label: "眉毛上下",
    range: [-50, 50],
    PcIcon: <Meimaoshangxia />,
    MobIcons: [MobMmsx, MobMmsxChanges, MobMmsxActive, MobMmsxChangesActive],
    fuKey: "intensity_brow_height",
    rawInitValue: 0.5,
    rawRange: [0, 1],
    reflexType: 2,
    initValue: 0,
  },
  {
    key: "眉间距",
    label: "眉间距",
    range: [-50, 50],
    PcIcon: <Meijianju />,
    MobIcons: [MobMjj, MobMjjChanges, MobMjjActive, MobMjjChangesActive],
    fuKey: "intensity_brow_space",
    rawInitValue: 0.5,
    rawRange: [0, 1],
    reflexType: 2,
    initValue: 0,
  },
  {
    key: "眉毛粗细",
    label: "眉毛粗细",
    range: [-50, 50],
    PcIcon: <Meimaocuxi />,
    MobIcons: [MobMmcx, MobMmcxChanges, MobMmcxActive, MobMmcxChangesActive],
    fuKey: "intensity_brow_thick",
    rawInitValue: 0.5,
    rawRange: [0, 1],
    reflexType: 2,
    initValue: 0,
  },
];
// 实际没用这个initValue
// 后续如果要每个滤镜的初始值不同的话还是需要这个值的
const filterList = [
  {
    key: "ziran1",
    label: "自然",
    index: 1,
    initValue: 0,
    range: [0, 100],
    PcIcon: "",
    MobIcons: [],
  },
  {
    key: "ziran2",
    label: "自然",
    index: 2,
    initValue: 0,
    range: [0, 100],
    PcIcon: "",
    MobIcons: [],
  },
  {
    key: "ziran3",
    label: "自然",
    index: 3,
    initValue: 0,
    range: [0, 100],
    PcIcon: "",
    MobIcons: [],
  },
  {
    key: "ziran4",
    label: "自然",
    index: 4,
    initValue: 0,
    range: [0, 100],
    PcIcon: "",
    MobIcons: [],
  },
  {
    key: "ziran5",
    label: "自然",
    index: 5,
    initValue: 0,
    range: [0, 100],
    PcIcon: "",
    MobIcons: [],
  },
  {
    key: "ziran6",
    label: "自然",
    index: 6,
    initValue: 0,
    range: [0, 100],
    PcIcon: "",
    MobIcons: [],
  },
  {
    key: "ziran7",
    label: "自然",
    index: 7,
    initValue: 0,
    range: [0, 100],
    PcIcon: "",
    MobIcons: [],
  },
  {
    key: "ziran8",
    label: "自然",
    index: 8,
    initValue: 0,
    range: [0, 100],
    PcIcon: "",
    MobIcons: [],
  },
  {
    key: "zhiganhui1",
    label: "质感灰",
    index: 1,
    initValue: 0,
    range: [0, 100],
    PcIcon: "",
    MobIcons: [],
  },
  {
    key: "zhiganhui2",
    label: "质感灰",
    index: 2,
    initValue: 0,
    range: [0, 100],
    PcIcon: "",
    MobIcons: [],
  },
  {
    key: "zhiganhui3",
    label: "质感灰",
    index: 3,
    initValue: 0,
    range: [0, 100],
    PcIcon: "",
    MobIcons: [],
  },
  {
    key: "zhiganhui4",
    label: "质感灰",
    index: 4,
    initValue: 0,
    range: [0, 100],
    PcIcon: "",
    MobIcons: [],
  },
  {
    key: "zhiganhui5",
    label: "质感灰",
    index: 5,
    initValue: 0,
    range: [0, 100],
    PcIcon: "",
    MobIcons: [],
  },
  {
    key: "zhiganhui6",
    label: "质感灰",
    index: 6,
    initValue: 0,
    range: [0, 100],
    PcIcon: "",
    MobIcons: [],
  },
  {
    key: "zhiganhui7",
    label: "质感灰",
    index: 7,
    initValue: 0,
    range: [0, 100],
    PcIcon: "",
    MobIcons: [],
  },
  {
    key: "zhiganhui8",
    label: "质感灰",
    index: 8,
    initValue: 0,
    range: [0, 100],
    PcIcon: "",
    MobIcons: [],
  },
  {
    key: "mitao1",
    label: "蜜桃",
    index: 1,
    initValue: 0,
    range: [0, 100],
    PcIcon: "",
    MobIcons: [],
  },
  {
    key: "mitao2",
    label: "蜜桃",
    index: 2,
    initValue: 0,
    range: [0, 100],
    PcIcon: "",
    MobIcons: [],
  },
  {
    key: "mitao3",
    label: "蜜桃",
    index: 3,
    initValue: 0,
    range: [0, 100],
    PcIcon: "",
    MobIcons: [],
  },
  {
    key: "mitao4",
    label: "蜜桃",
    index: 4,
    initValue: 0,
    range: [0, 100],
    PcIcon: "",
    MobIcons: [],
  },
  {
    key: "mitao5",
    label: "蜜桃",
    index: 5,
    initValue: 0,
    range: [0, 100],
    PcIcon: "",
    MobIcons: [],
  },
  {
    key: "mitao6",
    label: "蜜桃",
    index: 6,
    initValue: 0,
    range: [0, 100],
    PcIcon: "",
    MobIcons: [],
  },
  {
    key: "mitao7",
    label: "蜜桃",
    index: 7,
    initValue: 0,
    range: [0, 100],
    PcIcon: "",
    MobIcons: [],
  },
  {
    key: "mitao8",
    label: "蜜桃",
    index: 8,
    initValue: 0,
    range: [0, 100],
    PcIcon: "",
    MobIcons: [],
  },
  {
    key: "bailiang1",
    label: "白亮",
    index: 1,
    initValue: 0,
    range: [0, 100],
    PcIcon: "",
    MobIcons: [],
  },
  {
    key: "bailiang2",
    label: "白亮",
    index: 2,
    initValue: 0,
    range: [0, 100],
    PcIcon: "",
    MobIcons: [],
  },
  {
    key: "bailiang3",
    label: "白亮",
    index: 3,
    initValue: 0,
    range: [0, 100],
    PcIcon: "",
    MobIcons: [],
  },
  {
    key: "bailiang4",
    label: "白亮",
    index: 4,
    initValue: 0,
    range: [0, 100],
    PcIcon: "",
    MobIcons: [],
  },
  {
    key: "bailiang5",
    label: "白亮",
    index: 5,
    initValue: 0,
    range: [0, 100],
    PcIcon: "",
    MobIcons: [],
  },
  {
    key: "bailiang6",
    label: "白亮",
    index: 6,
    initValue: 0,
    range: [0, 100],
    PcIcon: "",
    MobIcons: [],
  },
  {
    key: "bailiang7",
    label: "白亮",
    index: 7,
    initValue: 0,
    range: [0, 100],
    PcIcon: "",
    MobIcons: [],
  },
  {
    key: "fennen1",
    label: "粉嫩",
    index: 1,
    initValue: 0,
    range: [0, 100],
    PcIcon: "",
    MobIcons: [],
  },
  {
    key: "fennen2",
    label: "粉嫩",
    index: 2,
    initValue: 0,
    range: [0, 100],
    PcIcon: "",
    MobIcons: [],
  },
  {
    key: "fennen3",
    label: "粉嫩",
    index: 3,
    initValue: 0,
    range: [0, 100],
    PcIcon: "",
    MobIcons: [],
  },
  {
    key: "fennen4",
    label: "粉嫩",
    index: 4,
    initValue: 0,
    range: [0, 100],
    PcIcon: "",
    MobIcons: [],
  },
  {
    key: "fennen5",
    label: "粉嫩",
    index: 5,
    initValue: 0,
    range: [0, 100],
    PcIcon: "",
    MobIcons: [],
  },
  {
    key: "fennen6",
    label: "粉嫩",
    index: 6,
    initValue: 0,
    range: [0, 100],
    PcIcon: "",
    MobIcons: [],
  },
  {
    key: "fennen7",
    label: "粉嫩",
    index: 7,
    initValue: 0,
    range: [0, 100],
    PcIcon: "",
    MobIcons: [],
  },
  {
    key: "fennen8",
    label: "粉嫩",
    index: 8,
    initValue: 0,
    range: [0, 100],
    PcIcon: "",
    MobIcons: [],
  },
  {
    key: "lengsediao1",
    label: "冷色调",
    index: 1,
    initValue: 0,
    range: [0, 100],
    PcIcon: "",
    MobIcons: [],
  },
  {
    key: "lengsediao2",
    label: "冷色调",
    index: 2,
    initValue: 0,
    range: [0, 100],
    PcIcon: "",
    MobIcons: [],
  },
  {
    key: "lengsediao3",
    label: "冷色调",
    index: 3,
    initValue: 0,
    range: [0, 100],
    PcIcon: "",
    MobIcons: [],
  },
  {
    key: "lengsediao4",
    label: "冷色调",
    index: 4,
    initValue: 0,
    range: [0, 100],
    PcIcon: "",
    MobIcons: [],
  },
  {
    key: "lengsediao5",
    label: "冷色调",
    index: 5,
    initValue: 0,
    range: [0, 100],
    PcIcon: "",
    MobIcons: [],
  },
  {
    key: "lengsediao6",
    label: "冷色调",
    index: 6,
    initValue: 0,
    range: [0, 100],
    PcIcon: "",
    MobIcons: [],
  },
  {
    key: "lengsediao7",
    label: "冷色调",
    index: 7,
    initValue: 0,
    range: [0, 100],
    PcIcon: "",
    MobIcons: [],
  },
  {
    key: "lengsediao8",
    label: "冷色调",
    index: 8,
    initValue: 0,
    range: [0, 100],
    PcIcon: "",
    MobIcons: [],
  },
  {
    key: "lengsediao9",
    label: "冷色调",
    index: 9,
    initValue: 0,
    range: [0, 100],
    PcIcon: "",
    MobIcons: [],
  },
  {
    key: "lengsediao10",
    label: "冷色调",
    index: 10,
    initValue: 0,
    range: [0, 100],
    PcIcon: "",
    MobIcons: [],
  },
  {
    key: "lengsediao11",
    label: "冷色调",
    index: 11,
    initValue: 0,
    range: [0, 100],
    PcIcon: "",
    MobIcons: [],
  },
  {
    key: "nuansediao1",
    label: "暖色调",
    index: 1,
    initValue: 0,
    range: [0, 100],
    PcIcon: "",
    MobIcons: [],
  },
  {
    key: "nuansediao2",
    label: "暖色调",
    index: 2,
    initValue: 0,
    range: [0, 100],
    PcIcon: "",
    MobIcons: [],
  },
  {
    key: "nuansediao3",
    label: "暖色调",
    index: 3,
    initValue: 0,
    range: [0, 100],
    PcIcon: "",
    MobIcons: [],
  },
  {
    key: "gexing1",
    label: "个性",
    index: 1,
    initValue: 0,
    range: [0, 100],
    PcIcon: "",
    MobIcons: [],
  },
  {
    key: "gexing2",
    label: "个性",
    index: 2,
    initValue: 0,
    range: [0, 100],
    PcIcon: "",
    MobIcons: [],
  },
  {
    key: "gexing3",
    label: "个性",
    index: 3,
    initValue: 0,
    range: [0, 100],
    PcIcon: "",
    MobIcons: [],
  },
  {
    key: "gexing4",
    label: "个性",
    index: 4,
    initValue: 0,
    range: [0, 100],
    PcIcon: "",
    MobIcons: [],
  },
  {
    key: "gexing5",
    label: "个性",
    index: 5,
    initValue: 0,
    range: [0, 100],
    PcIcon: "",
    MobIcons: [],
  },
  {
    key: "gexing6",
    label: "个性",
    index: 6,
    initValue: 0,
    range: [0, 100],
    PcIcon: "",
    MobIcons: [],
  },
  {
    key: "gexing7",
    label: "个性",
    index: 7,
    initValue: 0,
    range: [0, 100],
    PcIcon: "",
    MobIcons: [],
  },
  {
    key: "gexing8",
    label: "个性",
    index: 8,
    initValue: 0,
    range: [0, 100],
    PcIcon: "",
    MobIcons: [],
  },
  {
    key: "gexing9",
    label: "个性",
    index: 9,
    initValue: 0,
    range: [0, 100],
    PcIcon: "",
    MobIcons: [],
  },
  {
    key: "gexing10",
    label: "个性",
    index: 10,
    initValue: 0,
    range: [0, 100],
    PcIcon: "",
    MobIcons: [],
  },
  {
    key: "gexing11",
    label: "个性",
    index: 11,
    initValue: 0,
    range: [0, 100],
    PcIcon: "",
    MobIcons: [],
  },
  {
    key: "xiaoqingxin1",
    label: "小清新",
    index: 1,
    initValue: 0,
    range: [0, 100],
    PcIcon: "",
    MobIcons: [],
  },
  {
    key: "xiaoqingxin2",
    label: "小清新",
    index: 2,
    initValue: 0,
    range: [0, 100],
    PcIcon: "",
    MobIcons: [],
  },
  {
    key: "xiaoqingxin3",
    label: "小清新",
    index: 3,
    initValue: 0,
    range: [0, 100],
    PcIcon: "",
    MobIcons: [],
  },
  {
    key: "xiaoqingxin4",
    label: "小清新",
    index: 4,
    initValue: 0,
    range: [0, 100],
    PcIcon: "",
    MobIcons: [],
  },
  {
    key: "xiaoqingxin5",
    label: "小清新",
    index: 5,
    initValue: 0,
    range: [0, 100],
    PcIcon: "",
    MobIcons: [],
  },
  {
    key: "xiaoqingxin6",
    label: "小清新",
    index: 6,
    initValue: 0,
    range: [0, 100],
    PcIcon: "",
    MobIcons: [],
  },
  {
    key: "heibai1",
    label: "黑白",
    index: 1,
    initValue: 0,
    range: [0, 100],
    PcIcon: "",
    MobIcons: [],
  },
  {
    key: "heibai2",
    label: "黑白",
    index: 2,
    initValue: 0,
    range: [0, 100],
    PcIcon: "",
    MobIcons: [],
  },
  {
    key: "heibai3",
    label: "黑白",
    index: 3,
    initValue: 0,
    range: [0, 100],
    PcIcon: "",
    MobIcons: [],
  },
  {
    key: "heibai4",
    label: "黑白",
    index: 4,
    initValue: 0,
    range: [0, 100],
    PcIcon: "",
    MobIcons: [],
  },
  {
    key: "heibai5",
    label: "黑白",
    index: 5,
    initValue: 0,
    range: [0, 100],
    PcIcon: "",
    MobIcons: [],
  },
];
```