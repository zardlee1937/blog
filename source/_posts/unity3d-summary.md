---
title: Unity3D总结-AssetBundle
date: 2018-04-12 12:48:36
tags: Unity3D
---

## 打包

#### 接口

1. 在Unity3D 2017中，Assetstore提供了官方的AssetBundle可视化工具 -- AssetBundle Browser。可以使用图形化界面方便的打包AssetBundle。
2. 通过BuildPipline.BuildAssetBundle()函数打包，例如

<pre>
    <code>
    // 以下代码在Unity3d Assets菜单下添加一个Build AssetBundls按钮来打包AssetBundle
    [MenuItem("Assets/Build AssetBundls")]
    public static void Build()
    {
        var bundleList = new List&ltAssetBundleBuild&gt();
        bundleList.Add(new AssetBundleBuild
        {
            addressableNames = new string[]
            {
                "path/to/assets/",
            },
            assetNames = new string[]
            {
                "name/to/assets",
            },
            assetBundleName = "Texture",
            assetBundleVariant = "",
        });
        BuildPipeline.BuildAssetBundles(
            Path.Combine(Path.Combine(Application.streamingAssetsPath, "ABs"), "Custom"),
            BuildAssetBundleOptions.ForceRebuildAssetBundle,
            BuildTarget.Android);
    }
    </code>
</pre>

#### 策略

1. 打包AssetBundle需要根据项目的需求选择策略。一般的，大的资源分配策略有下：
    * 根据资源所属场景分配
    * 根据资源业务相关性分配
2. 场景不仅仅是指Unity3D的Level，是指资源出现的场景。比如一幅地图上用到的所有模型，纹理，声音。再比如一个UI页面上用到的图集，Sprite。
3. 业务相关性，则是根据所属对象和对象出现以及使用的情况分。比如进入一间房子需要显示桌子椅子和其他室内部件，这些部件所关联的Prefab，纹理，材质等等。

---

## 加载

#### 加载AssetBundle

1. 加载的核心接口一共三个
    * WWW = new WWW(Path) / UnityWebRequest.GetAssetBundle (两种方式实际都是network streaming形式，这里统称Web方式)
    * AssetBundle.LoadFromFile(Path)
    * AssetBundle.LoadFromMemoery(BinaryData)
2. Web方式可以使用coutoutine形式，从一个文件路径加载bundle。加载过程异步，并且会创建多个WebStream，适合同时加载多个小块资源。需要注意，使用完成后通过释放Request或WWW来释放WebStream。当加载AssetBundle时，Web方式与NetworkAssetBundle.LoadFromFile速度相差无几，100次平均下来差距在2-8ms左右。例如：

<pre><code>
    // 以下代码加载速度与AssetBundle.LoadFromFile相近
    private IEnumerator WWWLoader()
    {
        var loader = new WWW(Path.Combine(path, n));
        yield return loader;
        loader.Dispose();
        loader = null;
    }
    // 基本等同于
    private IEnumerator WebRequest()
    {
        var request = UnityWebRequest.GetAssetBundle(Path.Combine(path, n));
        yield return request.SendWebRequest();
        var bundle = DownloadHandlerAssetBundle.GetContent(request);
        request = null;
    }
</code></pre>

    PS:WebRequest中，无法通过Dispose接口主动释放WebStream，最终会通过Finalize来释放。一定程度上或许会造成GC压力。可以自行测试确定效率。

3. AssetBundle.LoadFromFile(Path)允许从一个文件中加载AssetBundle。如果该文件来源于网络，则需要先下载该文件到本地存储器，然后再加载。每次加载文件必然会开启一个FileStream，整体开销同正常程序的IO操作。一般情况下，这是加载AssetBundle最快的方式。

<pre><code>
    // 以下代码使用FileStream加载一个AssetBundle
    private void Load(string bundleFilePath)
    {
        var bundle = AssetBundle.LoadFromFile(bundlePath);
    }
</code></pre>

    PS:值得注意的是，AssetBundle.LoadFromFile会创建一个FileStream，连续调用多次可能会引起GC峰值。

4. AssetBundle.LoadFromMemoery(BinaryData)正常情况是几种方式中加载最慢的一个。但在实际使用中，程序本质上需要的是AssetBundle中的Resources，AssetBundle本身则并不是很需要。保有一份AssetBundle必然会造成内存开销，因此一些场景中，加载完AssetBundle中的Resources之后，需要释放掉Bundle本身。AssetBundle.LoadFromMemoery(BinaryData)适用于加载已经加载过，或者需要重复加载/卸载的AssetBundle。