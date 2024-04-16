## Vite 流程分析

### 流程
- 处理 config，内置插件，hooks等
- 开启 服务，启动服务端 ws
- 

### watcher.on('add', onFileAddUnlink)

```js
const onFileAddUnlink = async (file: string) => {
  file = normalizePath(file);
  await handleFileAddUnlink(file, server);
  await onHMRUpdate(file, true);
};

/**
 * 处理文件添加和删除的逻辑
 * @param file 要处理的文件路径
 * @param server Vite 开发服务器实例
 */
export async function handleFileAddUnlink(
  file: string,
  server: ViteDevServer
): Promise<void> {
  // 获取受影响的模块列表，包括当前文件以及所有受当前文件影响的模块
  const modules = [...(server.moduleGraph.getModulesByFile(file) || [])];

  // 获取所有受当前文件影响的全局模块
  modules.push(...getAffectedGlobModules(file, server));

  // 如果受影响的模块列表不为空，则更新这些模块的信息
  if (modules.length > 0) {
    // 获取当前文件在项目中的相对路径
    const shortName = getShortName(file, server.config.root);
    // 更新受影响模块的信息，包括模块名、模块列表、更新时间等
    updateModules(
      shortName,
      // 去除重复的模块
      unique(modules),
      // 获取当前时间作为更新时间
      Date.now(),
      // 传入 Vite 开发服务器实例
      server
    );
  }
}
```

### onChange

```js
watcher.on("change", async (file) => {
  moduleGraph.onFileChange(file);
  await onHMRUpdate(file, false);
});
```

### 同过以上可知，文件有变化，先修改 moduleGraph，再 通知 onHMRUpdate

```js
// 1. server 端
// server 端监听文件变更 最后一站 : updateModules
// updateModules 核心代码

updates.push(
    ...[...boundaries].map(({ boundary, acceptedVia }) => ({
    type: `${boundary.type}-update` as const,
    timestamp,
    path: normalizeHmrUrl(boundary.url),
    explicitImportRequired:
        boundary.type === 'js'
        ? isExplicitImportRequired(acceptedVia.url)
        : undefined,
    acceptedPath: normalizeHmrUrl(acceptedVia.url),
    })),
)

ws.send({
    type: 'update',
    updates,
  })

// 2. 客户端ws收到update后执行的片段
// ws 在启动服务阶段，整合config时 插入到 html的  script:/@vite/client
// 此处的 payload.updates和 上面的 updates是同一内容： updates 中的每一项代表着一个模块的更新信息
payload.updates.map(async (update): Promise<void> => {
    if (update.type === 'js-update') {
    return queueUpdate(fetchUpdate(update))
}

/**
 * 根据提供的更新信息，获取模块的更新。
 * @param {Update} update - 包含模块路径、接受路径、时间戳和是否需要显式导入等更新信息。
 * @returns {Function | undefined} - 执行更新逻辑的函数，如果找不到模块则返回undefined。
 */
async function fetchUpdate({
  path,
  acceptedPath,
  timestamp,
  explicitImportRequired,
}: Update) {
  // 检查模块是否存在于 hotModulesMap 中
  const mod = hotModulesMap.get(path)
  if (!mod) {
    // 如果找不到模块，则提前返回
    return
  }

  let fetchedModule: ModuleNamespace | undefined
  const isSelfUpdate = path === acceptedPath

  // 在重新导入模块之前确定符合条件的回调函数
  const qualifiedCallbacks = mod.callbacks.filter(({ deps }) =>
    deps.includes(acceptedPath),
  )

  // 如果模块正在更新或具有符合条件的回调函数
  if (isSelfUpdate || qualifiedCallbacks.length > 0) {
    // 处置与接受路径关联的现有数据
    const disposer = disposeMap.get(acceptedPath)
    if (disposer) await disposer(dataMap.get(acceptedPath))

    // 将接受路径拆分为路径和查询部分
    const [acceptedPathWithoutQuery, query] = acceptedPath.split(`?`)
    try {
      // 动态导入更新的模块
      fetchedModule = await import(
        /* @vite-ignore */
        base +
          acceptedPathWithoutQuery.slice(1) +
          `?${explicitImportRequired ? 'import&' : ''}t=${timestamp}${
            query ? `&${query}` : ''
          }`
      )
    } catch (e) {
      // 如果获取更新失败，则记录警告
      warnFailedFetch(e, acceptedPath)
    }
  }

  // 返回执行更新逻辑的函数
  return () => {
    for (const { deps, fn } of qualifiedCallbacks) {
      fn(
        deps.map(
          // 如果匹配接受路径，则将每个依赖项映射到获取的模块
          (dep) => (dep === acceptedPath ? fetchedModule : undefined)
        )
      )
    }
  }
}

/**
 * queueUpdate 用于缓冲由相同源更改触发的多个热更新，以确保它们按照发送的顺序被调用。首先，它将热更新承诺添加到队列中。然后，如果没有正在处理的热更新，它将标记为正在处理，并等待当前微任务队列执行完毕。接着，它将标记为未处理，并复制队列以备份，并清空原队列。最后，它执行所有热更新的回调函数。
 */

/**
 * 缓冲由相同源更改触发的多个热更新，以便它们按照发送的顺序被调用。
 * （否则由于 HTTP 请求往返，顺序可能不一致）
 * @param {Promise<(() => void) | undefined>} p - 表示要排队的热更新的承诺。
 * @returns {Promise<void>} - 表示排队热更新的异步承诺。
 */
async function queueUpdate(p: Promise<(() => void) | undefined>) {
  // 将热更新承诺添加到队列中
  queued.push(p)

  // 如果没有正在处理的热更新
  if (!pending) {
    // 标记为正在处理
    pending = true
    // 等待当前微任务队列执行完毕
    await Promise.resolve()
    // 标记为未处理
    pending = false
    // 复制队列并清空原队列
    const loading = [...queued]
    queued = []
    // 执行所有热更新
    ;(await Promise.all(loading)).forEach((fn) => fn && fn())
  }
}



```

### 添加 hot update fn

```js
//  vite打包模块时先要给 当前模块插入 一段代码，该代码功能包含
// 1. 创建 import.hot = createHotContext()
// 2. 编写 import.hot.accept(dep, fn) 方法， 执行的是以下 acceptDeps 方法。 这个 fn就是 fetchUpdate 里执行 fn，既import.hot.accept的回掉

function acceptDeps(deps: string[], callback: HotCallback["fn"] = () => {}) {
  const mod: HotModule = hotModulesMap.get(ownerPath) || {
    id: ownerPath,
    callbacks: [],
  };
  mod.callbacks.push({
    deps,
    fn: callback,
  });
  hotModulesMap.set(ownerPath, mod);
}
```


### hot 例子
1. css例子，
```js
// request url:
// http://localhost:5173/src/components/hello-world/HelloWorld.vue?vue&type=style&index=0&scoped=20b92c3c&lang.css
import {createHotContext as __vite__createHotContext} from "/@vite/client";
import.meta.hot = __vite__createHotContext("/src/components/hello-world/HelloWorld.vue?vue&type=style&index=0&scoped=20b92c3c&lang.css");
import {updateStyle as __vite__updateStyle, removeStyle as __vite__removeStyle} from "/@vite/client"
const __vite__id = "/Users/wagnxx/dt/playing/vite/vite-app/src/components/hello-world/HelloWorld.vue?vue&type=style&index=0&scoped=20b92c3c&lang.css"
const __vite__css = ".red[data-v-20b92c3c]{\n    color: red;\n}\n"
__vite__updateStyle(__vite__id, __vite__css)
import.meta.hot.accept()
export default __vite__css
import.meta.hot.prune(()=>__vite__removeStyle(__vite__id))


```
- 插入时机，解析阶段用cssPostPlugin处理
```js
const cssContent = await getContentWithSourcemap(css)
const code = [
    `import { updateStyle as __vite__updateStyle, removeStyle as __vite__removeStyle } from ${JSON.stringify(
    path.posix.join(config.base, CLIENT_PUBLIC_PATH),
    )}`,
    `const __vite__id = ${JSON.stringify(id)}`,
    `const __vite__css = ${JSON.stringify(cssContent)}`,
    `__vite__updateStyle(__vite__id, __vite__css)`,
    // css modules exports change on edit so it can't self accept
    `${
    modulesCode ||
    `import.meta.hot.accept()\nexport default __vite__css`
    }`,
    `import.meta.hot.prune(() => __vite__removeStyle(__vite__id))`,
].join('\n')
return { code, map: { mappings: '' } }
```
- __vite__updateStyle 直接更新 style.textContent = __vite__css
- hot.prune(()=>__vite__removeStyle(__vite__id)) 注册 prune 事件
```js
const hot = {
    ...,
    prune(cb) {
      pruneMap.set(ownerPath, cb)
    },

}

```

- importAnalysisPlugin 会处理 当updateModuleGraph后会对 没有被引用的模块进行收集，再通知客户端 prune事件，此处就是删除对应的 style标签
```js
const prunedImports = await moduleGraph.updateModuleInfo(
    importerModule,
    importedUrls,
    importedBindings,
    normalizedAcceptedUrls,
    isPartiallySelfAccepting ? acceptedExports : null,
    isSelfAccepting,
    ssr,
)
if (hasHMR && prunedImports) {
    handlePrunedModules(prunedImports, server)
}
export function handlePrunedModules(
  mods: Set<ModuleNode>,
  { ws }: ViteDevServer,
): void {
  // update the disposed modules' hmr timestamp
  // since if it's re-imported, it should re-apply side effects
  // and without the timestamp the browser will not re-import it!
  const t = Date.now()
  mods.forEach((mod) => {
    mod.lastHMRTimestamp = t
    debugHmr(`[dispose] ${colors.dim(mod.file)}`)
  })
  ws.send({
    type: 'prune',
    paths: [...mods].map((m) => m.url),
  })
}

case 'prune':
    notifyListeners('vite:beforePrune', payload)
    // After an HMR update, some modules are no longer imported on the page
    // but they may have left behind side effects that need to be cleaned up
    // (.e.g style injections)
    // TODO Trigger their dispose callbacks.
    payload.paths.forEach((path) => {
    const fn = pruneMap.get(path)
    if (fn) {
        fn(dataMap.get(path))
    }
    })
    break
```