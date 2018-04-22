---
title: UGUI屏幕适配思考
date: 2018-04-18 19:39:10
tags: Unity3D
---
最近一众“全面屏”手机的出现，让之前的项目遇到很多适配问题，由此考虑如何简便而快捷的适配更多新设备的屏幕尺寸。</br>

## 问题分析

选取iphoneX,iphone 6S,ipad分辨率为例，大体上，会遇到如下问题：
1. 以stretch模式定义控件大小，在不同分辨率下，控件会因为屏幕大小的改变而改变比例。
2. 以长宽+位置定义控件大小，在不同分辨率下，控件常常相互重叠或者超出屏幕。

![1_2](/image/1_2.png)
3. 以Anchors+stretch定义控件大小，控件比例正确，但往往长宽改变造成显示内容改变。

![3](/image/3.png)

## 解题思路

分析上述问题，实际上，所有UI的适配问题，最终可以归结为两类：
1. 每个组件长宽和位置均保持设计原稿大小，屏幕变化后，无法完整显示所有组件。
2. 每个组件长宽和位置均使用屏幕比例表示，屏幕变化后，可以完整显示所有组件，但组件大小和内容比例全部失真。

根据以上问题，合理的屏幕适配效果应该是，每个组件能够完全显示在新的分辨率下，且位置布局保持原本比例，允许一定的长宽位置变化。</br>
做到这点，首先我们需要能够快速更改整个UI布局，因而第一步，需要能够将原始UI转化为数据，然后用数据，在新的分辨率下重新生成UI布局。一般的，每一个UI组件，都包含一个RectTransform控件，因而使用以下代码，可以方便的将一个UI的Anchors布局数据化。这里数据以XML的形式保存方便阅读。
<pre><code>
    private void Bake(RectTransform node)
    {
        if (node.childCount < 1)
            throw new System.Exception("No GUI in this node of hierarchy.");

        var doc = new XmlDocument();
        var declaration = doc.CreateXmlDeclaration("1.0", "UTF-8", null);
        doc.AppendChild(declaration);

        var root = doc.CreateElement("Root");
        Bake(doc, root, node);
        doc.AppendChild(root);

        var path = System.IO.Path.Combine(Application.streamingAssetsPath, "gui");
        path = System.IO.Path.Combine(path, "main.xml");
        doc.Save(path);
        AssetDatabase.Refresh();
    }

    private void Bake(XmlDocument doc, XmlElement elem, RectTransform node)
    {
        for (var i = 0; i < node.childCount; ++i)
        {
            // 把递归移动到堆上免得炸了
            var action = new System.Action&ltXmlDocument, XmlElement, RectTransform&gt(Bake);
            var e = doc.CreateElement("Node");
            elem.AppendChild(e);
            action(doc, e, node.GetChild(i).GetComponent&ltRectTransform&gt());
        }
        var x = doc.CreateAttribute("x");
        x.Value = node.anchoredPosition.x.ToString();
        elem.Attributes.Append(x);
        var y = doc.CreateAttribute("y");
        y.Value = node.anchoredPosition.y.ToString();
        elem.Attributes.Append(y);
        var w = doc.CreateAttribute("width");
        w.Value = node.rect.width.ToString();
        elem.Attributes.Append(w);
        var h = doc.CreateAttribute("height");
        h.Value = node.rect.height.ToString();
        elem.Attributes.Append(h);
        var min = doc.CreateAttribute("anchor-min");
        min.Value = node.anchorMin.ToString();
        elem.Attributes.Append(min);
        var max = doc.CreateAttribute("anchor-max");
        max.Value = node.anchorMax.ToString();
        elem.Attributes.Append(max);
    }
</code></pre>

保存的数据结果类似这样。
![4](/image/4.png)

这里我只保存了变换和位置数据。然后使用以下代码，可以简单的对拥有相同分支结构和节点数量的UIGI进行位置重排。
<pre><code>
    private void Reset(RectTransform rootNode)
    {
        XmlDocument doc = new XmlDocument();
        doc.LoadXml(data.text);

        var root = doc["Root"];
        Reset(rootNode, root);
    }

    private void Reset(RectTransform node, XmlNode data)
    {
        for (int i = 0; i < data.ChildNodes.Count; ++i)
        {
            var ndata = data.ChildNodes[i];
            var n = node.GetChild(i) as RectTransform;
            SetNodeData(n, ndata);
            var action = new System.Action'<'RectTransform, XmlNode'>'(Reset);
            action(n, ndata);
        }
    }

    private RectTransform GenNode(RectTransform parent, XmlNode data)
    {
        var node = new GameObject("Node", typeof(RectTransform));
        var t = node.GetComponent&ltRectTransform&gt();
        t.SetParent(parent);
        SetNodeData(t, data);
        return t;
    }

    private void SetNodeData(RectTransform node, XmlNode data)
    {
        var x = float.Parse(data.Attributes["x"].Value);
        var y = float.Parse(data.Attributes["y"].Value);
        var w = float.Parse(data.Attributes["width"].Value);
        var h = float.Parse(data.Attributes["height"].Value);
        var min = StringToVector2(data.Attributes["anchor-min"].Value);
        var max = StringToVector2(data.Attributes["anchor-max"].Value);

        node.SetSizeWithCurrentAnchors(RectTransform.Axis.Horizontal, w);
        node.SetSizeWithCurrentAnchors(RectTransform.Axis.Vertical, h);
        node.anchoredPosition = new Vector2(x, y);
        node.anchorMin = min;
        node.anchorMax = max;
    }      

    private Vector2 StringToVector2(string str)
    {
        var strs = str.Replace("(", "").Replace(")", "").Split(',');
        return new Vector2(
            float.Parse(strs[0]),
            float.Parse(strs[1]));
    }
</code></pre>

当GUI布局能够数据化后，回顾问题分析中关于各种问题的描述，从直观数学上看，anchors+stretch布局从数值上最正确。然而当屏幕改变后，由于长宽比也发生改变，往往按照比例布局，最终得到的结果很容易造成图像的比例失衡，类似于原本的正方形变成了长方形的问题就变得很普遍。</br>
为了解决这个问题，需要为控件大小引入一个最佳比例的概念。比如100:100的控件，在600x800的显示器中为最佳显示效果，那么不难得到该控件最佳显示比例为1:1。当显示器变更为960x1280时，我们先以控件宽度作为基准，按比例可得100/600x960=160,则在新比例下，控件长宽应为160:160。但有时显示器的长宽比会发生很大变化，导致以宽度作为基准，得到合适的高度比例无法再显示器中显示。比如新的比例为600x90的显示器，按比例计算，100/600x600=100，此时最佳显示比例的控件，高度将高出屏幕。因而在以最佳比例的显示缩放中，仍需要进一步选择长宽中更小的一个作为基准。</br>
目前市面上移动设备最常见的屏幕尺寸大概3类：
1. 全面屏
    * 小米MIX2/2S系列 -- 2160x1080
    * 三星Galaxy s8/s9 -- 2960x1440
    * 华为Mate 10 -- 2560x1440
    * 华为Mate 10 Pro -- 2160x1080
    * iphoneX -- 2436x1125
2. 标准
    * Android VGA -- 800x600/800x480
    * Android HD -- 1920x1080/1280x720
    * iphone 6/6P/7/7P/8/8P -- 1334x750
3. 平板
    * ipad/ipad mini -- 2048x1536
    * ipad pro -- 2732x2048/2224x1668