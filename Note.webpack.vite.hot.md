## Webapck 和 Vite HMR 的区别


### HMR 在开发中的作用
- 快速开发迭代：HMR 可以使开发者在修改代码后立即看到修改的效果，而无需手动刷新页面。这极大地加速了开发迭代的速度，提高了开发效率。
- 保持应用状态：HMR 在替换模块时会尽量保持应用的状态，例如保留页面上的输入内容、滚动位置等。这样可以避免每次模块更新时都需要重新操作页面，提升了开发体验。
- 模块级别更新：HMR 可以实现模块级别的更新，只更新发生变化的模块，而不是整个页面。这样可以减少不必要的资源加载和渲染，提高了页面加载速度。
- 减少开发成本：由于无需手动刷新页面，开发者可以专注于代码编写和调试，减少了开发过程中的重复操作，降低了开发成本

## 常用的工具，Vite HMR 和 Webpack HMR， 共同点都要建立devserver，server-ws，client-ws，实现资源请求和 客户端-服务端 通知推送通信

### Vite HMR 的特点
- 基于 import.meta,和插件 给模块注入 import.meta.hot = createHotContext()
- import.meta.hot上注册 accept 事件，传入回掉函数自定义实现代码生效方案 import.meta.hot.accept(deps, hot => {______})
- 启动服务时同时启动chokidar.watch监听，文件变动会执行 moduleGraph更新，收集 updates 信息并附带通知客户端
- 客户端收到update消息，通过update中的 url获取模块 （fetchUpdate（path）），得到模块后触发 fn(fetchedModule), fn 为前面注入的import.meta.hot.accept中的回掉函数，实现模块的更新和最新逻辑的执行， css-style注入式的也遵循这个方案；css-link的有所不同，会拷贝 oldLink，重置href为新链接（主要更新timestamp），把newLink插入成功删掉oldLink

### Webpack HMR 的特点
- 添加entry
```js
    if (this.options.hot === "only") {
      additionalEntries.push(require.resolve("webpack/hot/only-dev-server"));
    } else if (this.options.hot) {
      additionalEntries.push(require.resolve("webpack/hot/dev-server"));
    }
```
- 监听 hooks
```js
 this.compiler.hooks.invalid.tap("webpack-dev-server", // ...  this.sendMessage(this.webSocketServer.clients, "invalid");
 this.compiler.hooks.done.tap("webpack-dev-server",  // ...   this.sendStats(this.webSocketServer.clients, this.getStats(stats));


sendStats(clients, stats, force) {
    const shouldEmit =
        !force &&
        stats &&
        (!stats.errors || stats.errors.length === 0) &&
        (!stats.warnings || stats.warnings.length === 0) &&
        this.currentHash === stats.hash;

    if (shouldEmit) {
        this.sendMessage(clients, "still-ok");
        return;
    }

    this.currentHash = stats.hash;
    this.sendMessage(clients, "hash", stats.hash);
    // 错误，警告 ...
    else
         this.sendMessage(clients, "ok");
}
```
- 从上一步可以看出，invalid和done事件 最终都会执行 sendMessage， 并且客户端我们只需关心三个事件 ：invalid， hash，still-ok, ok。先看看sendMessage 做了哪些事呢
```js
  sendMessage(clients, type, data, params) {
    for (const client of clients) {
      // `sockjs` uses `1` to indicate client is ready to accept data
      // `ws` uses `WebSocket.OPEN`, but it is mean `1` too
      if (client.readyState === 1) {
        client.send(JSON.stringify({ type, data, params }));
      }
    }
  }
// 由此可知 invalid， hash，still-ok 三个事件分别传递值为：

// invalid： undefined

// hash： stats.hash

// still-ok： undefined

// ok: undefined

```
- 在客户端的 client接受不同的指令，hash指令更新hash，ok指令准备进入更新逻辑
```js
{
    invalid: function invalid() {
        log.info("App updated. Recompiling...");

        // Fixes #1042. overlay doesn't clear if errors are fixed but warnings remain.
        if (options.overlay) {
        overlay.send({
            type: "DISMISS"
        });
        }
        sendMessage("Invalid");
    },
  hash: function hash(_hash) {
    status.previousHash = status.currentHash;
    status.currentHash = _hash;
  },
"still-ok": function stillOk() {
    log.info("Nothing changed.");
    if (options.overlay) {
      overlay.send({
        type: "DISMISS"
      });
    }
    sendMessage("StillOk");
  },
  ok: function ok() {
    sendMessage("Ok");
    if (options.overlay) {
      overlay.send({
        type: "DISMISS"
      });
    }
    reloadApp(options, status);
  },
}

function reloadApp() {
    if (hot && allowToHot) {
    log.info("App hot update...");
    hotEmitter.emit("webpackHotUpdate", status.currentHash);
    if (typeof self !== "undefined" && self.window) {
      // broadcast update to window
      self.postMessage("webpackHotUpdate".concat(status.currentHash), "*");
    }
  }
}
```
- webpack/hot/dev-sever 收到 webpackHotUpdate, 开始 hot.check, 也就是HotModuleReplacement.runtime.hotCheck
```js
// 定义 hotCheck 函数，用于检查模块更新情况并执行相应的处理
function hotCheck(applyOnUpdate) {
    // 如果当前状态不是 "idle"，则抛出错误
    if (currentStatus !== "idle") {
        throw new Error("check() is only allowed in idle status");
    }
    // 设置状态为 "check"，然后继续执行后续操作
    return setStatus("check")
        // 调用 $hmrDownloadManifest$ 方法获取模块更新信息
        // 替换方法详看下方 runtimeGlobal——（1）
        .then($hmrDownloadManifest$)
        // 处理模块更新信息
        .then(function (update) {
            // 如果没有更新信息，直接返回相应的状态
            if (!update) {
                return setStatus(applyInvalidatedModules() ? "ready" : "idle").then(
                    function () {
                        return null;
                    }
                );
            }

            // 设置状态为 "prepare"，准备进行模块更新
            return setStatus("prepare").then(function () {
                // 初始化 updatedModules 数组和 currentUpdateApplyHandlers 数组
                var updatedModules = [];
                currentUpdateApplyHandlers = [];

                // 并发处理更新的模块
                return Promise.all(
                    // 遍历更新处理器列表，处理模块更新
                    // 替换方法详看下方 runtimeGlobal——（2）
                    Object.keys($hmrDownloadUpdateHandlers$).reduce(function (
                        promises,
                        key
                    ) {
                        // 调用对应的更新处理器，处理模块更新
                        // 在一般情况下，__webpack_require__.hmrC 上只会挂载一个 JSONP 回调函数，用于接收模块更新时发送的新代码。当客户端接收到新的模块更新时，JSONP 回调函数会被触发，执行更新后的模块代码。这样就完成了模块的热更新过程。
                        // 也就是 key = ‘jsonp’
                        $hmrDownloadUpdateHandlers$[key](
                            update.c,
                            update.r,
                            update.m,
                            promises,
                            currentUpdateApplyHandlers,
                            updatedModules
                        );
                        
                        /**
                         * $hmrDownloadUpdateHandlers$[key] 这个方法主要执行 promises.push(loadUpdateChunk(chunkId, updatedModulesList));
                         * 这里有个执行顺序需要明确一下：
                         * 1. loadUpdateChunk 在JsonpChunkLoadingRuntimeModule生成， 内创建一个promise任务，设置 waitingUpdateResolves[chunkId] = resolve 不在此函数内调用，
                         * 正常走jsonp方式请求资源 updateModuleHandler
                         * 2.得到 updateModuleHandler 载入到页面会自动执行 updateModuleHandler(), 即 self["webpackHotUpdatewebpack_hmr"] 这个函数
                         * 3. 在 self["webpackHotUpdatewebpack_hmr"] 尾部会执行waitingUpdateResolves[chunkId]，也就是说 loadUpdateChunk的promise状态终于变成 success了， 
                         * 
                         */
                          // 此时 promise收集结束，可以返回，不过应该说【undefined】,不过也没关系，后续也不需要这个值 
                        return promises;
                    }, [])
                ).then(function () {
                    // 等待阻塞 Promise 完成
                    return waitForBlockingPromises(function () {
                        // 如果有传入 applyOnUpdate 参数，则执行更新
                        if (applyOnUpdate) {
                          /**
                           *  internalApply 方法中执行 dispose,apply,大概流程是：
                           * 
                           *  result = applyHandler(options) // {dispose,apply} 详情见下方 applyHandler 完整代码
                           *  result.dispose() // 用于清理和释放旧模块的资源
                           *  result.apply() // 将新模块的代码应用到页面中，从而完成热模块替换的过程
                           */
                            return internalApply(applyOnUpdate);
                        } else {
                            // 否则，将状态设置为 "ready"，表示更新准备就绪
                            return setStatus("ready").then(function () {
                                return updatedModules;
                            });
                        }
                    });
                });
            });
        });
}


```
- runtimeGloabal
```js
// (1) exports.hmrDownloadManifest = "__webpack_require__.hmrM";
__webpack_require__.hmrM = ()=>{
    /******/
    if (typeof fetch === "undefined")
        throw new Error("No browser support: need fetch API");
    /******/
    return fetch(__webpack_require__.p + __webpack_require__.hmrF()).then((response)=>{
        /******/
        if (response.status === 404)
            return;
        // no update available
        /******/
        if (!response.ok)
            throw new Error("Failed to fetch update manifest " + response.statusText);
        /******/
        return response.json();
        /******/
    }
    );
    /******/
}


/**
 * array with handler functions to download chunk updates
 */
// （2） exports.hmrDownloadUpdateHandlers = "__webpack_require__.hmrC";
  __webpack_require__.hmrC.jsonp = function(/******/
  chunkIds, /******/
  removedChunks, /******/
  removedModules, /******/
  promises, /******/
  applyHandlers, /******/
  updatedModulesList/******/
  ) {
      /******/
      applyHandlers.push(applyHandler);
      /******/
      currentUpdateChunks = {};
      /******/
      currentUpdateRemovedChunks = removedChunks;
      /******/
      currentUpdate = removedModules.reduce(function(obj, key) {
          /******/
          obj[key] = false;
          /******/
          return obj;
          /******/
      }, {});
      /******/
      currentUpdateRuntime = [];
      /******/
      chunkIds.forEach(function(chunkId) {
          /******/
          if (/******/
          __webpack_require__.o(installedChunks, chunkId) && /******/
          installedChunks[chunkId] !== undefined/******/
          ) {
              /******/
              promises.push(loadUpdateChunk(chunkId, updatedModulesList));
              /******/
              currentUpdateChunks[chunkId] = true;
              /******/
          } else {
              /******/
              currentUpdateChunks[chunkId] = false;
              /******/
          }
          /******/
      });
      /******/
      if (__webpack_require__.f) {
          /******/
          __webpack_require__.f.jsonpHmr = function(chunkId, promises) {
              /******/
              if (/******/
              currentUpdateChunks && /******/
              __webpack_require__.o(currentUpdateChunks, chunkId) && /******/
              !currentUpdateChunks[chunkId]/******/
              ) {
                  /******/
                  promises.push(loadUpdateChunk(chunkId));
                  /******/
                  currentUpdateChunks[chunkId] = true;
                  /******/
              }
              /******/
          }
          ;
          /******/
      }
      /******/
  }




self["webpackHotUpdatewebpack_hmr"] = 
(chunkId,moreModules,runtime) => {
  /******/
  for (var moduleId in moreModules) {
      /******/
      if (__webpack_require__.o(moreModules, moduleId)) {
          /******/
          currentUpdate[moduleId] = moreModules[moduleId];
          /******/
          if (currentUpdatedModulesList)
              currentUpdatedModulesList.push(moduleId);
          /******/
      }
      /******/
  }
  /******/
  if (runtime)
      currentUpdateRuntime.push(runtime);
  /******/
  if (waitingUpdateResolves[chunkId]) {
      /******/
      waitingUpdateResolves[chunkId]();
      /******/
      waitingUpdateResolves[chunkId] = undefined;
      /******/
  }
  /******/
}


function applyHandler(options) {
  /******/
  if (__webpack_require__.f)
      delete __webpack_require__.f.jsonpHmr;
  /******/
  currentUpdateChunks = undefined;
  /******/
  function getAffectedModuleEffects(updateModuleId) {
      /******/
      var outdatedModules = [updateModuleId];
      /******/
      var outdatedDependencies = {};
      /******/
      /******/
      var queue = outdatedModules.map(function(id) {
          /******/
          return {
              /******/
              chain: [id],
              /******/
              id: id/******/
          };
          /******/
      });
      /******/
      while (queue.length > 0) {
          /******/
          var queueItem = queue.pop();
          /******/
          var moduleId = queueItem.id;
          /******/
          var chain = queueItem.chain;
          /******/
          var module = __webpack_require__.c[moduleId];
          /******/
          if (/******/
          !module || /******/
          (module.hot._selfAccepted && !module.hot._selfInvalidated)/******/
          )
              /******/
              continue;
          /******/
          if (module.hot._selfDeclined) {
              /******/
              return {
                  /******/
                  type: "self-declined",
                  /******/
                  chain: chain,
                  /******/
                  moduleId: moduleId/******/
              };
              /******/
          }
          /******/
          if (module.hot._main) {
              /******/
              return {
                  /******/
                  type: "unaccepted",
                  /******/
                  chain: chain,
                  /******/
                  moduleId: moduleId/******/
              };
              /******/
          }
          /******/
          for (var i = 0; i < module.parents.length; i++) {
              /******/
              var parentId = module.parents[i];
              /******/
              var parent = __webpack_require__.c[parentId];
              /******/
              if (!parent)
                  continue;
              /******/
              if (parent.hot._declinedDependencies[moduleId]) {
                  /******/
                  return {
                      /******/
                      type: "declined",
                      /******/
                      chain: chain.concat([parentId]),
                      /******/
                      moduleId: moduleId,
                      /******/
                      parentId: parentId/******/
                  };
                  /******/
              }
              /******/
              if (outdatedModules.indexOf(parentId) !== -1)
                  continue;
              /******/
              if (parent.hot._acceptedDependencies[moduleId]) {
                  /******/
                  if (!outdatedDependencies[parentId])
                      /******/
                      outdatedDependencies[parentId] = [];
                  /******/
                  addAllToSet(outdatedDependencies[parentId], [moduleId]);
                  /******/
                  continue;
                  /******/
              }
              /******/
              delete outdatedDependencies[parentId];
              /******/
              outdatedModules.push(parentId);
              /******/
              queue.push({
                  /******/
                  chain: chain.concat([parentId]),
                  /******/
                  id: parentId/******/
              });
              /******/
          }
          /******/
      }
      /******/
      /******/
      return {
          /******/
          type: "accepted",
          /******/
          moduleId: updateModuleId,
          /******/
          outdatedModules: outdatedModules,
          /******/
          outdatedDependencies: outdatedDependencies/******/
      };
      /******/
  }
  /******/
  /******/
  function addAllToSet(a, b) {
      /******/
      for (var i = 0; i < b.length; i++) {
          /******/
          var item = b[i];
          /******/
          if (a.indexOf(item) === -1)
              a.push(item);
          /******/
      }
      /******/
  }
  /******/
  /******/
  // at begin all updates modules are outdated
  /******/
  // the "outdated" status can propagate to parents if they don't accept the children
  /******/
  var outdatedDependencies = {};
  /******/
  var outdatedModules = [];
  /******/
  var appliedUpdate = {};
  /******/
  /******/
  var warnUnexpectedRequire = function warnUnexpectedRequire(module) {
      /******/
      console.warn(/******/
      "[HMR] unexpected require(" + module.id + ") to disposed module"/******/
      );
      /******/
  };
  /******/
  /******/
  for (var moduleId in currentUpdate) {
      /******/
      if (__webpack_require__.o(currentUpdate, moduleId)) {
          /******/
          var newModuleFactory = currentUpdate[moduleId];
          /******/
          /** @type {TODO} */
          /******/
          var result;
          /******/
          if (newModuleFactory) {
              /******/
              result = getAffectedModuleEffects(moduleId);
              /******/
          } else {
              /******/
              result = {
                  /******/
                  type: "disposed",
                  /******/
                  moduleId: moduleId/******/
              };
              /******/
          }
          /******/
          /** @type {Error|false} */
          /******/
          var abortError = false;
          /******/
          var doApply = false;
          /******/
          var doDispose = false;
          /******/
          var chainInfo = "";
          /******/
          if (result.chain) {
              /******/
              chainInfo = "\nUpdate propagation: " + result.chain.join(" -> ");
              /******/
          }
          /******/
          switch (result.type) {
              /******/
          case "self-declined":
              /******/
              if (options.onDeclined)
                  options.onDeclined(result);
              /******/
              if (!options.ignoreDeclined)
                  /******/
                  abortError = new Error(/******/
                  "Aborted because of self decline: " + /******/
                  result.moduleId + /******/
                  chainInfo/******/
                  );
              /******/
              break;
              /******/
          case "declined":
              /******/
              if (options.onDeclined)
                  options.onDeclined(result);
              /******/
              if (!options.ignoreDeclined)
                  /******/
                  abortError = new Error(/******/
                  "Aborted because of declined dependency: " + /******/
                  result.moduleId + /******/
                  " in " + /******/
                  result.parentId + /******/
                  chainInfo/******/
                  );
              /******/
              break;
              /******/
          case "unaccepted":
              /******/
              if (options.onUnaccepted)
                  options.onUnaccepted(result);
              /******/
              if (!options.ignoreUnaccepted)
                  /******/
                  abortError = new Error(/******/
                  "Aborted because " + moduleId + " is not accepted" + chainInfo/******/
                  );
              /******/
              break;
              /******/
          case "accepted":
              /******/
              if (options.onAccepted)
                  options.onAccepted(result);
              /******/
              doApply = true;
              /******/
              break;
              /******/
          case "disposed":
              /******/
              if (options.onDisposed)
                  options.onDisposed(result);
              /******/
              doDispose = true;
              /******/
              break;
              /******/
          default:
              /******/
              throw new Error("Unexception type " + result.type);
              /******/
          }
          /******/
          if (abortError) {
              /******/
              return {
                  /******/
                  error: abortError/******/
              };
              /******/
          }
          /******/
          if (doApply) {
              /******/
              appliedUpdate[moduleId] = newModuleFactory;
              /******/
              addAllToSet(outdatedModules, result.outdatedModules);
              /******/
              for (moduleId in result.outdatedDependencies) {
                  /******/
                  if (__webpack_require__.o(result.outdatedDependencies, moduleId)) {
                      /******/
                      if (!outdatedDependencies[moduleId])
                          /******/
                          outdatedDependencies[moduleId] = [];
                      /******/
                      addAllToSet(/******/
                      outdatedDependencies[moduleId], /******/
                      result.outdatedDependencies[moduleId]/******/
                      );
                      /******/
                  }
                  /******/
              }
              /******/
          }
          /******/
          if (doDispose) {
              /******/
              addAllToSet(outdatedModules, [result.moduleId]);
              /******/
              appliedUpdate[moduleId] = warnUnexpectedRequire;
              /******/
          }
          /******/
      }
      /******/
  }
  /******/
  currentUpdate = undefined;
  /******/
  /******/
  // Store self accepted outdated modules to require them later by the module system
  /******/
  var outdatedSelfAcceptedModules = [];
  /******/
  for (var j = 0; j < outdatedModules.length; j++) {
      /******/
      var outdatedModuleId = outdatedModules[j];
      /******/
      var module = __webpack_require__.c[outdatedModuleId];
      /******/
      if (/******/
      module && /******/
      (module.hot._selfAccepted || module.hot._main) && /******/
      // removed self-accepted modules should not be required
      /******/
      appliedUpdate[outdatedModuleId] !== warnUnexpectedRequire && /******/
      // when called invalidate self-accepting is not possible
      /******/
      !module.hot._selfInvalidated/******/
      ) {
          /******/
          outdatedSelfAcceptedModules.push({
              /******/
              module: outdatedModuleId,
              /******/
              require: module.hot._requireSelf,
              /******/
              errorHandler: module.hot._selfAccepted/******/
          });
          /******/
      }
      /******/
  }
  /******/
  /******/
  var moduleOutdatedDependencies;
  /******/
  /******/
  return {
      /******/
      dispose: function() {
          /******/
          currentUpdateRemovedChunks.forEach(function(chunkId) {
              /******/
              delete installedChunks[chunkId];
              /******/
          });
          /******/
          currentUpdateRemovedChunks = undefined;
          /******/
          /******/
          var idx;
          /******/
          var queue = outdatedModules.slice();
          /******/
          while (queue.length > 0) {
              /******/
              var moduleId = queue.pop();
              /******/
              var module = __webpack_require__.c[moduleId];
              /******/
              if (!module)
                  continue;
              /******/
              /******/
              var data = {};
              /******/
              /******/
              // Call dispose handlers
              /******/
              var disposeHandlers = module.hot._disposeHandlers;
              /******/
              for (j = 0; j < disposeHandlers.length; j++) {
                  /******/
                  disposeHandlers[j].call(null, data);
                  /******/
              }
              /******/
              __webpack_require__.hmrD[moduleId] = data;
              /******/
              /******/
              // disable module (this disables requires from this module)
              /******/
              module.hot.active = false;
              /******/
              /******/
              // remove module from cache
              /******/
              delete __webpack_require__.c[moduleId];
              /******/
              /******/
              // when disposing there is no need to call dispose handler
              /******/
              delete outdatedDependencies[moduleId];
              /******/
              /******/
              // remove "parents" references from all children
              /******/
              for (j = 0; j < module.children.length; j++) {
                  /******/
                  var child = __webpack_require__.c[module.children[j]];
                  /******/
                  if (!child)
                      continue;
                  /******/
                  idx = child.parents.indexOf(moduleId);
                  /******/
                  if (idx >= 0) {
                      /******/
                      child.parents.splice(idx, 1);
                      /******/
                  }
                  /******/
              }
              /******/
          }
          /******/
          /******/
          // remove outdated dependency from module children
          /******/
          var dependency;
          /******/
          for (var outdatedModuleId in outdatedDependencies) {
              /******/
              if (__webpack_require__.o(outdatedDependencies, outdatedModuleId)) {
                  /******/
                  module = __webpack_require__.c[outdatedModuleId];
                  /******/
                  if (module) {
                      /******/
                      moduleOutdatedDependencies = /******/
                      outdatedDependencies[outdatedModuleId];
                      /******/
                      for (j = 0; j < moduleOutdatedDependencies.length; j++) {
                          /******/
                          dependency = moduleOutdatedDependencies[j];
                          /******/
                          idx = module.children.indexOf(dependency);
                          /******/
                          if (idx >= 0)
                              module.children.splice(idx, 1);
                          /******/
                      }
                      /******/
                  }
                  /******/
              }
              /******/
          }
          /******/
      },
      /******/
      apply: function(reportError) {
          /******/
          // insert new code
          /******/
          for (var updateModuleId in appliedUpdate) {
              /******/
              if (__webpack_require__.o(appliedUpdate, updateModuleId)) {
                  /******/
                  __webpack_require__.m[updateModuleId] = appliedUpdate[updateModuleId];
                  /******/
              }
              /******/
          }
          /******/
          /******/
          // run new runtime modules
          /******/
          for (var i = 0; i < currentUpdateRuntime.length; i++) {
              /******/
              currentUpdateRuntime[i](__webpack_require__);
              /******/
          }
          /******/
          /******/
          // call accept handlers
          /******/
          for (var outdatedModuleId in outdatedDependencies) {
              /******/
              if (__webpack_require__.o(outdatedDependencies, outdatedModuleId)) {
                  /******/
                  var module = __webpack_require__.c[outdatedModuleId];
                  /******/
                  if (module) {
                      /******/
                      moduleOutdatedDependencies = /******/
                      outdatedDependencies[outdatedModuleId];
                      /******/
                      var callbacks = [];
                      /******/
                      var errorHandlers = [];
                      /******/
                      var dependenciesForCallbacks = [];
                      /******/
                      for (var j = 0; j < moduleOutdatedDependencies.length; j++) {
                          /******/
                          var dependency = moduleOutdatedDependencies[j];
                          /******/
                          var acceptCallback = /******/
                          module.hot._acceptedDependencies[dependency];
                          /******/
                          var errorHandler = /******/
                          module.hot._acceptedErrorHandlers[dependency];
                          /******/
                          if (acceptCallback) {
                              /******/
                              if (callbacks.indexOf(acceptCallback) !== -1)
                                  continue;
                              /******/
                              callbacks.push(acceptCallback);
                              /******/
                              errorHandlers.push(errorHandler);
                              /******/
                              dependenciesForCallbacks.push(dependency);
                              /******/
                          }
                          /******/
                      }
                      /******/
                      for (var k = 0; k < callbacks.length; k++) {
                          /******/
                          try {
                              /******/
                              callbacks[k].call(null, moduleOutdatedDependencies);
                              /******/
                          } catch (err) {
                              /******/
                              if (typeof errorHandlers[k] === "function") {
                                  /******/
                                  try {
                                      /******/
                                      errorHandlers[k](err, {
                                          /******/
                                          moduleId: outdatedModuleId,
                                          /******/
                                          dependencyId: dependenciesForCallbacks[k]/******/
                                      });
                                      /******/
                                  } catch (err2) {
                                      /******/
                                      if (options.onErrored) {
                                          /******/
                                          options.onErrored({
                                              /******/
                                              type: "accept-error-handler-errored",
                                              /******/
                                              moduleId: outdatedModuleId,
                                              /******/
                                              dependencyId: dependenciesForCallbacks[k],
                                              /******/
                                              error: err2,
                                              /******/
                                              originalError: err/******/
                                          });
                                          /******/
                                      }
                                      /******/
                                      if (!options.ignoreErrored) {
                                          /******/
                                          reportError(err2);
                                          /******/
                                          reportError(err);
                                          /******/
                                      }
                                      /******/
                                  }
                                  /******/
                              } else {
                                  /******/
                                  if (options.onErrored) {
                                      /******/
                                      options.onErrored({
                                          /******/
                                          type: "accept-errored",
                                          /******/
                                          moduleId: outdatedModuleId,
                                          /******/
                                          dependencyId: dependenciesForCallbacks[k],
                                          /******/
                                          error: err/******/
                                      });
                                      /******/
                                  }
                                  /******/
                                  if (!options.ignoreErrored) {
                                      /******/
                                      reportError(err);
                                      /******/
                                  }
                                  /******/
                              }
                              /******/
                          }
                          /******/
                      }
                      /******/
                  }
                  /******/
              }
              /******/
          }
          /******/
          /******/
          // Load self accepted modules
          /******/
          for (var o = 0; o < outdatedSelfAcceptedModules.length; o++) {
              /******/
              var item = outdatedSelfAcceptedModules[o];
              /******/
              var moduleId = item.module;
              /******/
              try {
                  /******/
                  item.require(moduleId);
                  /******/
              } catch (err) {
                  /******/
                  if (typeof item.errorHandler === "function") {
                      /******/
                      try {
                          /******/
                          item.errorHandler(err, {
                              /******/
                              moduleId: moduleId,
                              /******/
                              module: __webpack_require__.c[moduleId]/******/
                          });
                          /******/
                      } catch (err2) {
                          /******/
                          if (options.onErrored) {
                              /******/
                              options.onErrored({
                                  /******/
                                  type: "self-accept-error-handler-errored",
                                  /******/
                                  moduleId: moduleId,
                                  /******/
                                  error: err2,
                                  /******/
                                  originalError: err/******/
                              });
                              /******/
                          }
                          /******/
                          if (!options.ignoreErrored) {
                              /******/
                              reportError(err2);
                              /******/
                              reportError(err);
                              /******/
                          }
                          /******/
                      }
                      /******/
                  } else {
                      /******/
                      if (options.onErrored) {
                          /******/
                          options.onErrored({
                              /******/
                              type: "self-accept-errored",
                              /******/
                              moduleId: moduleId,
                              /******/
                              error: err/******/
                          });
                          /******/
                      }
                      /******/
                      if (!options.ignoreErrored) {
                          /******/
                          reportError(err);
                          /******/
                      }
                      /******/
                  }
                  /******/
              }
              /******/
          }
          /******/
          /******/
          return outdatedModules;
          /******/
      }/******/
  };
  /******/
}

```