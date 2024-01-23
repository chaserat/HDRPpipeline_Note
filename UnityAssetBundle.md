# UnityAssetBundle

## AssetBundle内容

- 使用AssetBundleBrowser可以查看Bundle基础内容

  - "com.unity.assetbundlebrowser": "https://github.com/Unity-Technologies/AssetBundles-Browser.git",

- 使用Unity安装目录Tools下的工具WebExtract和binary2text可以吧AssetBundle的内容提取到文本文件

- External References

  ```
  External References
  path(1): "Library/unity default resources" GUID: 0000000000000000e000000000000000 Type: 0
  path(2): "archive:/CAB-91155ceb2b588eccabe542ba663bfcf5/CAB-91155ceb2b588eccabe542ba663bfcf5" GUID: 00000000000000000000000000000000 Type: 0
  path(3): "archive:/CAB-c4f86db63bc5ffbd48b4aaa1354ab83c/CAB-c4f86db63bc5ffbd48b4aaa1354ab83c" GUID: 00000000000000000000000000000000 Type: 0
  path(4): "archive:/CAB-e67981e3cab004a88d75ec850fc16d86/CAB-e67981e3cab004a88d75ec850fc16d86" GUID: 00000000000000000000000000000000 Type: 0
  ```

  - 所依赖的外部文件列表

- m_Container

  - 所有可寻址资源对象列表（打包时传入的资源列表），只有简单描述

    ```
     11: 	m_Container  (map) [size: 29, children: 1]
       13: 		size 6 (int)
    		data  (pair) [size: 25, children: 2]
       15: 			first "prefabs/metal_albedo.tex" (string)
       19: 			second  (AssetInfo) [size: 20, children: 3]
       20: 				preloadIndex 2 (int)
       21: 				preloadSize 1 (int)
       22: 				asset  (PPtr<Object>) [size: 12, children: 2]
       23: 					m_FileID 0 (int)
       24: 					m_PathID 8779226390041277365 (SInt64)
    ```

  - 每个资源对应PreloadTable中的一段数据，如材质资源，在PreloadTable会有贴图、Shader等

  - m_Container中存在的资源才能使用AssetBundle.LoadAsse加载

- m_PreloadTable

  - m_Container中每个资源依赖的具体资源，（包含打在其它Bundle的资源）

    ```
        1: 	m_Name "res_mat.ab" (string)
        5: 	m_PreloadTable  (vector) [size: 16, children: 1]
        7: 		size 18 (int)
          data  (PPtr<Object>) [size: 12, children: 2]
        9: 			m_FileID 1 (int)
       10: 			m_PathID -5757993651755961534 (SInt64)
    ```

  - m_FileID：依赖的外部文件的索引（External References），如果是当前文件的资源，m_FileID为0

  - m_PathID：文件内部的对象Id，对应对象的ID字段

  - 通过这2个ID就可以定位到资源对象

  - 运行时这2个ID会跟对象的InstanceId关联

  - 调用AssetBundle.LoadAsset时会加载该对象PreloadTable中的所有所有资源

  - m_PreloadTable里的数据会重复（比如m_Container中的多个资源都依赖了同一个Prefab）

  - 这里的依赖是展开的（比如Prefab会列出它依赖的所有资源，Mesh、贴图等）

  - 如果很多资源都依赖了同一个复杂资源，这里的重复数据会很多

- 资源对象数据

  - 每个资源对象的具体序列化数据

    ```
    ID: -5757993651755961534 (ClassID: 28)     0: Texture2D [size: 115, children: 24 pathID: -400444606]
        1: 	m_Name "Paint5G_AlbedoSmoothness" (string)
        5: 	m_ForcedFallbackFormat 4 (int)
        6: 	m_DownscaleFallback 0 (bool)
        7: 	m_IsAlphaChannelOptional 0 (bool)
        8: 	m_Width 1024 (int)
        9: 	m_Height 1024 (int)
       10: 	m_CompleteImageSize 1398128 (unsigned int)
       11: 	m_MipsStripped 0 (int)
       12: 	m_TextureFormat 48 (int)
       13: 	m_MipCount 11 (int)
    ```

  - ID: -5757993651755961534，这个ID就是该资源的m_PathID

  - 不同Bundle里的同名文件的m_PathID是相同的，猜测这个Hash是用文件名生成的

- m_StreamData

  - 二进制流数据，如贴图

## 对象标识
- FileIdentifier（文件标识）
  - pathName
  - type
  - guid
- SerializedObjectIdentifier（全局对象标识） 
  - serializedFileIndex （全局文件索引）
  -  localIdentifierInFile （m_PathID）
- LocalSerializedObjectIdentifier（局部对象标识）
  - localSerializedFileIndex（m_FileID）
  - localIdentifierInFile（m_PathID）
- InstanceId
  - 在内存中生成的实例D，全局唯一
  - int类型，每次递增2，如果溢出会Crash
  - UnityEngine.Objec的成员变量
  - 有一个全局的HashMap（ms_IDToPointer）保存了InstanceId到Object指针的映射
  - AssetBundle反序列化时会生成内部以及依赖的所有对象的InstanceId（PPtr.transfer）
  - 如果AssetBundle卸载再重新加载，内部资源的InstanceId会重新生成
- PPtr
  - PPtr.transfer会从m_FileID和m_PathID拿到对象的IncetanceId，如果没有则分配
  - 第一次使用PPtr对象指针时，如果对象没有加载，会触发对象的加载
  - AssetBundle内的对象引用都是通过PPtr实现的

## 编辑器加载
```
guid -> FileIdentifier
m_ArtifactIDToArtifactMetaInfo = m_DB->OpenMap(txn, "ArtifactIDToArtifactMetaInfo")(CreateSerializedAssetV2)
GetImportedMetaInfo(guid) -> ImportedAssetMetaInfo
PersistentManager::LocalSerializedObjectIdentifierToInstanceID(FileIdentifier, ImportedAssetMetaInfo.objectInfo.localIdentifier)
GUIDPersistentManagerV2::InsertFileIdentifierInternal(FileIdentifier) -> SerializedObjectIdentifier
SerializedObjectIdentifier(globalIndex, localID)
Remapper::GetOrGenerateInstanceID(SerializedObjectIdentifier) -> InstanceId
PPtr::operator T*()
ReadObjectFromPersistentManager
SerializedFile::ReadObject
```
## 打包
```
AssetBundleBuilder::AddBuildAssets -> m_BuildAssets
PersistentManager::InstanceIDToSerializedObjectIdentifier
PPtr::Transfer
InstanceIDToLocalSerializedObjectIdentifier
BuildingPlayerOrAssetBundleInstanceIDResolveCallback
m_BuildAssets.find
PersistentManager::GlobalToLocalSerializedFileIndex -> LocalSerializedObjectIdentifier
localIdentifierInFile是直接赋值的
LocalSerializedObjectIdentifier -> (m_FileID, m_PathID)
```
- AssetBundle InternalName(ArchiveStorageReader.m_DirectoryInfo)
```
MdFourGenerator generator;
generator.Feed(assetBundleFullName.GetName());
Hash128 hash = generator.Finish();
core::string internalName = core::Join("CAB-", Hash128ToString(hash));
// 这个Hash是稳定的，只要Bundle名字不变，每次生成的Hash就是相同的
```
- 场景和其它可寻址资源不能打在同一个Bundle
  - 会报错：Cannot mark assets and scenes in one AssetBundle. AssetBundle name is "xxx".
  - 如果只有场景可寻址，其它资源通过依赖打入Bundle是可以的
- 同一次Build不同AssetBundle不允许加入相同资源
  - 会报错：Asset "xxx" has been assigned to more than one Asset Bundle.
  - AssetBundleVariant也不可以
- AssetBundleVariant
  - 不同变体在内部用相同名字和ID，适合做不同设备适配资源
  - 同一个AssetBundle的不同变体不能同时加载，会报重复加载AssetBundle错误
- 注意事项
  - 打包时尽量指定
    - DisableLoadAssetByFileName
    - DisableLoadAssetByFileNameWithExtension
    - 不指定的话加载AssetBundle后会建立2个Map索引资源


## 运行时加载

- AssetBundle.LoadFromFileAsync

```
AssetBundleLoadFromFileAsyncOperation::Execute
GetBackgroundJobQueue().ScheduleJob(LoadArchiveJob, this)
void JobQueue::ExecuteJobFunc
AssetBundleLoadFromFileAsyncOperation::LoadArchiveJob(AssetBundleLoadFromFileAsyncOperation*) 
	AssetBundleLoadFromFileAsyncOperation::LoadArchive
  ArchiveStorageReader::Initialize
  ArchiveStorageHeader::ReadHeader
  ArchiveStorageReader::ReadBlocksAndDirectory
  ArchiveStorageReader.m_DirectoryInfo
    node0: "CAB-92a91385b61fb83e1101b6c0e8dbda29"(SerializedData)
    node1: "CAB-92a91385b61fb83e1101b6c0e8dbda29.resS"(StreamData)
AssetBundleLoadFromAsyncOperation::IntegrateWithPreloadManager
	GetPreloadManager().AddToQueue(this)
PreloadManager::ProcessSingleOperation
AssetBundleLoadFromAsyncOperation::Perform
serializedFilePath = $"archive:/{nodes[0].path}/{nodes[i].path}"
if (!PersistentManager::IsStreamLoaded(serializedFilePath))
PersistentManager::LoadFileStream(serializedFilePath)
nameSpaceID = GUIDPersistentManagerV2::InsertPathNameInternal
	if(!m_ConstantGUIDAssetPathToIndex.find(goodPathName))
	GUIDPersistentManagerV2::InsertFileIdentifierInternal
SerializedFile::ReadHeader
SerializedFile::ReadMetadata
	READ m_Externals(FileIdentifier)
	pathName: "archive:/CAB-c647322316ea566480ea1e94d43b166b/CAB-c647322316ea566480ea1e94d43b166b"
PersistentManager::PostLoadStreamNameSpaceInternal
  ForIn(externalRefs)
  {
      int serializedFileIndex = InsertFileIdentifierInternal(externalRefs[i])
      if(!m_ConstantGUIDAssetPathToIndex.find(goodPathName))
      GUIDPersistentManagerV2::AddFileIdentifier
        PersistentManager::AddStream
        serializedFileIndex = m_IndexToFile.size();
        m_IndexToFile.push_back(file);
        m_ConstantGUIDAssetPathToIndex.insert(goodPathName, serializedFileIndex)
      m_GlobalToLocalNameSpace[nameSpaceID][serializedFileIndex] = i + 1;
      m_LocalToGlobalNameSpace[nameSpaceID][i + 1] = serializedFileIndex;
  }
  m_GlobalToLocalNameSpace[nameSpaceID][nameSpaceID] = 0;
  m_LocalToGlobalNameSpace[nameSpaceID][0] = nameSpaceID;
  // m_IndexToFile、m_Streams、m_GlobalToLocalNameSpace、m_LocalToGlobalNameSpace
  // 这4个数组是一一对应的，索引就是serializedFileIndex
  // m_ConstantGUIDAssetPathToIndex是个hash_map，是后续快速反向查找用的
abID = GetAssetBundleInstanceId
PersistentManager::ReadObjectThreaded(abID)
Remapper::InstanceIDToSerializedObjectIdentifier(InstanceID)
PersistentManager::ReadAndActivateObjectThreaded(InstanceID, SerializedObjectIdentifier)
SerializedFile = PersistentManager::GetSerializedFileIfObjectAvailable(SerializedObjectIdentifier)
PersistentManager::SetActiveNameSpace(identifier.serializedFileIndex);
SerializedFile::ReadObject(AssetBundle)
  AssetBundle::Transfer
  Transfer(m_PreloadTable)
  Transfer(m_Container)
  Transfer(m_MainAsset)
  PPtr::Transfer
  (m_FileID, m_PathID) -> LocalSerializedObjectIdentifier
  PersistentManager::LocalSerializedObjectIdentifierToInstanceID(LocalSerializedObjectIdentifier)
  activeNameSpace = GetActiveNameSpace
  if (localSerializedFileIndex == 0) 
  {
      globalFileIndex = activeNameSpace
  }
  else 
  {
      globalFileIndex = m_LocalToGlobalNameSpace[activeNameSpace].find(localSerializedFileIndex)
  }
  SerializedObjectIdentifier(globalFileIndex, localIdentifier.localIdentifierInFile)
  Remapper::GetOrGenerateInstanceID(SerializedObjectIdentifier)
  	Remapper.m_SerializedObjectToInstanceID.insert
  	Remapper.m_InstanceIDToSerializedObject.insert
PersistentManager::ClearActiveNameSpace
```
  - AssetBundle::LoadAsset
    - 资源加载之后会通过PreloadTable加载所有依赖的资源
```
AssetBundleManager::CollectPreloadDataDependencies(m_Dependencies)
AssetBundleManager::CollectPreloadDataRecursively(m_Dependencies)
AssetBundleManager::CollectPreloadData
AssetBundle::GetPreloadData(m_PreloadTable)
LoadOperation::Perform
PersistentManager::LoadObjectsThreaded(instanceIDs)
PrepareLoadObjects
	Remapper::InstanceIDToSerializedObjectIdentifier(InstanceID)
	m_InstanceIDToSerializedObject.find(instanceID)
PersistentManager::ReadAndActivateObjectThreaded(InstanceID, SerializedObjectIdentifier)
SerializedFile = PersistentManager::GetSerializedFileIfObjectAvailable(SerializedObjectIdentifier)
PersistentManager::SetActiveNameSpace(identifier.serializedFileIndex);
SerializedFile::ReadObject(localIdentifierInFile)
  Texture2D::Transfer
  TransferResourceImage(m_StreamData)
PersistentManager::ClearActiveNameSpace
```
  - PPtr::operator T*()
```
if(Object::IDToPointer(GetInstanceID()) == NULL)
PersistentManager::ReadObject(InstanceID)
PersistentManager::ReadObjectThreaded(InstanceID)
Remapper::InstanceIDToSerializedObjectIdentifier
PersistentManager::ReadAndActivateObjectThreaded(InstanceID, SerializedObjectIdentifier)	
SerializedFile = PersistentManager::GetSerializedFileIfObjectAvailable(SerializedObjectIdentifier)
PersistentManager::SetActiveNameSpace(identifier.serializedFileIndex);
SerializedFile::ReadObject(localIdentifierInFile)
  Texture2D::Transfer
PersistentManager::ClearActiveNameSpace
```

- Unity依赖Bundle是通过文件名Hash对应的
- 反序列化AssetBundle时所有对象的全局文件索引已经确定了

## 资源卸载

- Object.DestroyImmediate(asset, true)
  - 可以完全销毁对象
  - 调用DestroyObjectHighLevel
    - GameObject调用DestroyGameObjectHierarchy
    - 其它对象调用DestroySingleObject
  - 会调用PersistentManager.MakeObjectUnpersistent从Remapper删除InstanceId
  - 而注册ID则是在AssetBundle反序列化时PPtr.transfer中调用LocalSerializedObjectIdentifierToInstanceID做的
  - 所以下次PersistentManager.PrepareLoadObjects时会跳过该资源的加载，返回null
- Resources.UnloadAsset
  - 不能卸载Prefab，会报错
  - UnloadAsset may only be used on individual assets and can not be used on GameObject's / Components / AssetBundles or GameManagers
  - 调用UnloadObject内部直接delete_object_internal
  - 不会调用PersistentManager.MakeObjectUnpersistent
  - 因此InstanceId还在Remapper中，下次加载InstanceId保持不变
- AssetBundle.Unload(true)
  - 内部对象会全部销毁
  - 通过PersistentManager::RemoveObjectsFromPath从Remapper中删除所有InstanceId
  - 下次加载InstanceId会发生变化