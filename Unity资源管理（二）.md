# Unity资源管理（二）

## 整包方案

### Prefab加载

- Prefab加载之后，会加载并创建所有依赖的资源
- Object.DestroyImmediate(prefab, true)
  - 可以完全销毁对象
  - 内部调用DestroyGameObjectHierarchy
  - 会调用PersistentManager.MakeObjectUnpersistent
  - 会从Remapper删除InstanceId
  - 而注册ID则是在AssetBundle反序列化时PPtr.transfer中调用LocalSerializedObjectIdentifierToInstanceID做的
  - 所以下次PersistentManager.PrepareLoadObjects时会跳过该资源的加载，返回null

- Resources.UnloadAsset
  - 不能卸载Prefab，会报错
  - UnloadAsset may only be used on individual assets and can not be used on GameObject's / Components / AssetBundles or GameManagers
  - 卸载其它资源不会调用PersistentManager.MakeObjectUnpersistent
  - 因此InstanceId还在Remapper中，下载加载InstanceId保持不变

- AssetBundle.Unload(true)
  - 内部对象会全部卸载
  - InstanceId也会全部回收

- 贴图
  - 如果单独卸载Prefab依赖的贴图，贴图会丢失
  - 再次加载、重新实例化不会恢复引用
- 材质
  - 材质卸载之后，如果当前场景中还有引用的话会在PrepareMeshRenderNodes中触发PPtr的重新加载材质
  - 加载材质后会在Material.EnsurePropertiesExist中触发重新加载或设置贴图（由propertiesValid控制只处理一次）
- 目前来看Prefab只能通过AssetBundle.Unload(true)来卸载
- Prefab依赖的资源可以单独卸载
  - 贴图、材质、动作等内置对象会重新映射资源
  - MonoBehaviour引用的资源不会自动映射


### 热更Patch

- 同一次Build不同AssetBundle不允许加入相同资源
  - 报错Asset "xxx" has been assigned to more than one Asset Bundle.
  - AssetBundleVariant也不可以

- AssetBundleVariant
  - 不同变体在内部用相同名字和ID，适合做不同设备适配资源
  - 不同变体的同一个AssetBundle不能同时加载，会报重复加载AssetBundle错误

- 通过阅读源码，Unity依赖Bundle是通过文件名Hash对应的
- 反序列化AssetBundle时所有对象的全局文件索引已经确定了
- 不修改源码没办法将原始资源的依赖资源指向新Bundle

## 结论
- 内存控制
  - 普通资源手动控制加载、卸载
  - Prefab依赖资源手动加载、卸载
  - Prefab资源不单独释放，跟随Bundle释放
  - 有MonoBehaviour引用资源的预制考虑使用反射重新关联资源
- 热更Patch
  - 基础资源（贴图、Mesh、动画等）
    - 整包打包
    - 热更把改变的资源新打一个Bundle
    - 复杂资源也重新打包，应用依赖
  - 复杂资源（有依赖的资源，如Prefab、Material）
    - 拷贝到包外，做二进制Patch
    - 或者将依赖改变资源的所有资源全部打进Patch包
- 或者修改Unity源码
  - Prefab支持释放、重新加载
  - AssetBundle支持资源覆盖

