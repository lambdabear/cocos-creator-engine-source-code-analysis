# Cocos-JS Engine 源码分析

Cocos Creator 是当前非常流行的手游和微信小游戏的开发工具，Cocos Creator 的引擎部分包括 JavaScript 和 C++两个部分，我们团队主要用的是 JS-engine，因此有必要对 JS-engine 进行深入的分析与学习，而分析其源码是一种有效的学习手段。

### 代码核心组成

从入口文件 index.js 来看，引擎的核心还是 cocos2d，源码位于 cocos2d 文件夹，启动引擎为 CCBoot.js，整个引擎的命名空间位于 cc module 之下。而 cocos2d 的重要部分有 CCgame, director, action, node, scene, sprite, render-texture 几个部分。

### CCgame

CCgame 定义了一个核心对象 game，包含了整个游戏的主要状态属性。以下是较为重要的部分：

#### 事件 Event

- EVENT*HIDE:"game*\_on_hide"
- EVENT*SHOW:"game*\_on_show"
- EVENT_GAME_INITED: "game_inited"
- EVENT_RENDERER_INITED: "renderer_inited"

对应于前两个事件要特别注意，源码中给出了重要说明。  
**事件"game_on_hide"**: 请注意，在 WEB 平台，这个事件不一定会 100% 触发，这完全取决于浏览器的回调行为。在原生平台，它对应的是应用被切换到后台事件，下拉菜单和上拉状态栏等不一定会触发这个事件，这取决于系统行为。  
**事件“game_on_show”**: 请注意，在 WEB 平台，这个事件不一定会 100% 触发，这完全取决于浏览器的回调行为。在原生平台，它对应的是应用被切换到前台事件。

#### 渲染类型 RENDER_TYPE

- RENDER_TYPE_CANVAS: 0
- RENDER_TYPE_WEBGL: 1
- RENDER_TYPE_OPENGL: 2

#### 根节点 RootNodes

- \_persistRootNodes: {}
- \_ignoreRemovePersistNode: null

#### Config 键值

```js
CONFIG_KEY: {
    width: "width",
    height: "height",
    // engineDir: "engineDir",
    debugMode: "debugMode",
    exposeClassName: "exposeClassName",
    showFPS: "showFPS",
    frameRate: "frameRate",
    id: "id",
    renderMode: "renderMode",
    registerSystemEvent: "registerSystemEvent",
    jsList: "jsList",
    scenes: "scenes"
}
```

#### 状态 States

- \_paused: true, //whether the game is paused
- \_configLoaded: false, //whether config loaded
- \_isCloning: false, // deserializing or instantiating
- \_prepareCalled: false, //whether the prepare function has been called
- \_prepared: false, //whether the engine has prepared
- \_rendererInitialized: false,
- \_renderContext: null,
- \_intervalId: null, //interval target of main
- \_lastTime: null,
- \_frameTime: null,

#### 场景链表 Scenes list

- \_sceneInfos: []

#### 公共方法 Public Methods

```js
setFrameRate (frameRate) {...}  
step () {
    cc.director.mainLoop();
}
```

step()函数用于执行一帧游戏循环，实际上是调用执行 cc.director.mainLoop 函数，cc.director 是一个管理游戏的逻辑流程的单例对象。它创建和处理主窗口并且管理什么时候执行场景。mainLoop 函数我们在以后的 director 部分再做详解。

```js
pause () {...}
```

源码里的注释说的很清楚：暂停游戏主循环。包含：游戏逻辑，渲染，事件处理，背景音乐和所有音效。这点和只暂停游戏逻辑的 cc.director.pause 不同。

```js
resume () {...}
```

恢复游戏主循环。包含：游戏逻辑，渲染，事件处理，背景音乐和所有音效。

```js
run () {...}
```

运行游戏，并且指定引擎配置和 onStart 的回调。

```js
addPersistRootNode: function (node) {
    if (!cc.Node.isNode(node) || !node.uuid) {
        cc.warnID(3800);
        return;
    }
    var id = node.uuid;
    if (!this._persistRootNodes[id]) {
        var scene = cc.director._scene;
        if (cc.isValid(scene)) {
            if (!node.parent) {
                node.parent = scene;
            }
            else if ( !(node.parent instanceof cc.Scene) ) {
                cc.warnID(3801);
                return;
            }
            else if (node.parent !== scene) {
                cc.warnID(3802);
                return;
            }
            this._persistRootNodes[id] = node;
            node._persistNode = true;
        }
    }
}
```

添加常驻根节点，该节点在场景切换时不会被销毁。看源码会检查该 node 的 parent，如果为空则将当前 scene 赋给 node.parent，如果不为空且不等于当前 scene 则会告警退出。还有一些其它的公共方法，就不做详解了，有兴趣的可以参看源代码。

#### 私有方法 Private Methods

```js
_setAnimFrame () {...}
```

这个函数的主要作用是根据运行环境设置 window.requestAnimFrame，在 web 环境下，应该就是标准的 window.requestAnimationFrame；关于 requestAnimationFrame 在游戏开发中的应用，可以参考这篇文章[Anatomy of a video game](https://developer.mozilla.org/zh-CN/docs/Games/Anatomy)

```js
_runMainLoop: function () {
    var self = this, callback, config = self.config, CONFIG_KEY = self.CONFIG_KEY,
        director = cc.director,
        skip = true, frameRate = config[CONFIG_KEY.frameRate];

    director.setDisplayStats(config[CONFIG_KEY.showFPS]);

    callback = function () {
        if (!self._paused) {
            self._intervalId = window.requestAnimFrame(callback);
            if (frameRate === 30) {
                if (skip = !skip) {
                    return;
                }
            }
            director.mainLoop();
        }
    };

    self._intervalId = window.requestAnimFrame(callback);
    self._paused = false;
}
```

运行游戏，通过前面设置的 window.requestAnimFrame 函数形成游戏主循环，每帧中运行 director.mainLoop; 关于 mainLoop 函数，会在 director 一章介绍。剩余几个私有方法都是初始化配置信息、初始化渲染器 renderer、初始化事件的函数。

### CCDirector

cc.director 是一个 singleton 对象，包含了一些标准方法，这些方法主要用于创建和处理主窗口并且管理场景的执行。cc.director 是由 cc.Director.\_getInstance 生成的，而这个函数是用 cc.DisplayLinkDirector 这个类构造出 director 的。

#### DisplayLinkDirector

cc.DisplayLinkDirector 是继承于 cc.Director，cc.DisplayLinkDirector 定义了 mainLoop 函数和几个控制动画的方法。我们应该重点关注 mainLoop 函数，因为这是游戏运行的主循环函数，下面看看 mainLoop 的定义

```js
/**
 * Run main loop of director
 */
mainLoop: CC_EDITOR
  ? function(deltaTime, updateAnimate) {
      if (!this._paused) {
        this.emit(cc.Director.EVENT_BEFORE_UPDATE);

        this._compScheduler.startPhase();
        this._compScheduler.updatePhase(deltaTime);

        if (updateAnimate) {
          this._scheduler.update(deltaTime);
        }

        this._compScheduler.lateUpdatePhase(deltaTime);

        this.emit(cc.Director.EVENT_AFTER_UPDATE);
      }

      this.emit(cc.Director.EVENT_BEFORE_VISIT);
      // update the scene
      this._visitScene();
      this.emit(cc.Director.EVENT_AFTER_VISIT);

      // Render
      cc.g_NumberOfDraws = 0;
      cc.renderer.clear();

      cc.renderer.rendering(cc._renderContext);
      this._totalFrames++;

      this.emit(cc.Director.EVENT_AFTER_DRAW);
    }
  : function() {
      if (this._purgeDirectorInNextLoop) {
        this._purgeDirectorInNextLoop = false;
        this.purgeDirector();
      } else if (!this.invalid) {
        // calculate "global" dt
        this.calculateDeltaTime();

        if (!this._paused) {
          this.emit(cc.Director.EVENT_BEFORE_UPDATE);
          // Call start for new added components
          this._compScheduler.startPhase();
          // Update for components
          this._compScheduler.updatePhase(this._deltaTime);
          // Engine update with scheduler
          this._scheduler.update(this._deltaTime);
          // Late update for components
          this._compScheduler.lateUpdatePhase(this._deltaTime);
          // User can use this event to do things after update
          this.emit(cc.Director.EVENT_AFTER_UPDATE);
          // Destroy entities that have been removed recently
          cc.Object._deferredDestroy();
        }

        /* to avoid flickr, nextScene MUST be here: after tick and before draw.
             XXX: Which bug is this one. It seems that it can't be reproduced with v0.9 */
        if (this._nextScene) {
          this.setNextScene();
        }

        this.emit(cc.Director.EVENT_BEFORE_VISIT);
        // update the scene
        this._visitScene();
        this.emit(cc.Director.EVENT_AFTER_VISIT);

        // Render
        cc.g_NumberOfDraws = 0;
        cc.renderer.clear();

        cc.renderer.rendering(cc._renderContext);
        this._totalFrames++;

        this.emit(cc.Director.EVENT_AFTER_DRAW);
        eventManager.frameUpdateListeners();
      }
    };
```

从源码来看，主循环主要的步骤为:

1.  发送即将更新事件 EVENT_BEFORE_UPDATE;
2.  调用组件调度器\_compScheduler 运行所有新注册组件的 start 方法，\_compScheduler 是在初始化 director 时 new ComponentScheduler()生成的对象实例，ComponentScheduler 定义在 component-scheduler.js 文件;
3.  调用\_compScheduler 运行所有注册组件的 update 方法;
4.  运行所有有效的定时器 update，定时器\_scheduler 是在初始化时 new cc.Scheduler()生成的对象实例，cc.Scheduler 的定义在 CCScheduler.js 文件中;
5.  调用\_compScheduler 运行所有注册组件的 lateUpdate 方法;
6.  发送更新完成事件 EVENT_AFTER_UPDATE;
7.  发送场景即将更新事件 EVENT_BEFORE_VISIT;
8.  更新场景;
9.  发送场景更新完成事件 EVENT_AFTER_VISIT;
10. 渲染下一帧;
11. 发送渲染完成事件 EVENT_AFTER_DRAW

#### Director

DisplayLinkDirector 继承于 Director，Director 提供了一些对场景操作的方法

```js
// 暂停正在运行的场景，该暂停只会停止游戏逻辑执行，但是不会停止渲染和 UI 响应。如果想要更彻底得暂停游戏，包含渲染，音频和事件，请使用cc.game.pause
pause() {...}
// 恢复暂停场景的游戏逻辑，如果当前场景没有暂停将没任何事情发生。
resume() {...}
// 暂停当前运行的场景，压入到暂停的场景栈中。
pushScene(scene) {...}
// 立刻切换指定场景。
runSceneImmediate(scene, onBeforeLoadScene, onLaunched) {...}
// 运行指定场景
runScene(scene, onBeforeLoadScene, onLaunched) {...}
// 通过场景名称进行加载场景。
loadScene(sceneName, onLaunched, _onUnloaded) {...}
// 预加载场景，你可以在任何时候调用这个方法。
// 调用完后，你仍然需要通过 `cc.director.loadScene` 来启动场景.
// 就算预加载还没完成，你也可以直接调用 `cc.director.loadScene`，加载完成后场景就会启动。
preloadScene(sceneName, onLoaded) {...}
```
