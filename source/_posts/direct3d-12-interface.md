---
title: Directx12 接口总结-概念篇
date: 2018-03-28 12:38:02
tags: D3D
---

## Design Philosophy

dx12由于开放了更底层的硬件接口，使开发者可以完全控制计算任务什么时候提交到gpu，因而在使用上比起之前的接口有两改进：
1. 没有单一的immidiate context，因此可以使用多线程handle多个context。
2. 设置好的一次渲染过程可以重复使用。
dx12的设计，使cpu/gpu之间更像client/server之间的关系。cpu在本地分配堆存储数据，通过api允许的接口申请gpu管辖的内存，通过公用数据结构的形式上传cpu保持的数据和代码到gpu，再由gpu计算渲染结果返回公用内存，最后通过一套输出接口（dxgi）输出到显示器。同时cpu也可以访问计算后的结果。
dx12的内存模型，也更加接近于由开发者申请提交，cpu填充，开发者调度使用，一改以往完全由后台调度的策略。开发者需要为正确的使用和调用内存对象做更多工作，同时也能因此获得更高的app执行效率。这之间的关系就好比是c# gc和cpp alloc。

## Components

### Command Queue and Command List

为了实现能够多线程提交和复用渲染，需要从结构上更改d3d app的gpu渲染工作流。dx12针对之前的结构，做出了三个重大改进：
1. 去除immediate context。使用command list代替原本的immediate context。每个command list包含原本api中的图源和渲染状态，可以在单独线程中存在。
2. app现在可以控制gpu如何组织调用渲染过程。使得相同的渲染过程可以重复调用。
3. app现在允许控制什么时候如何提交渲染过程。

#### Command list

command list允许app保存一个绘制或者状态改变的调用过程，并在晚些时候通过gpu来执行这个过程。
* 为了充分利用现代硬件的功能，dx12加入了second level of command list。被称作bundles。
* first level command list(direct command list)可以被直接引用，bundles则允许app在direct command list中打包一些小型api commands。
* bundles在创建时会被尽可能的预处理以提高之后的执行效率。
* bundles可以被包含在多个direct command list中，并被同个的command list多次执行。
* 使用bundles可以提高cpu单线程效率。由于bundles是被预处理并可以多次提交的，它对其中捆绑的操作有一定要求。
* 如果一个command list已经确认执行完毕。在提交新的执行请求前，它可以被重复多次执行。
* 在gpu上执行工作，app需要有效的通过command queue提交command list。
* 一个direct command list可以被直接提交并多次执行，但是在下次提交执行之前，app不知道前次的command list是否执行完毕。
* bundles没有并发限制可以被执行多次。但bundles不能通过command queue直接提交。
* 任何线程都可以在任何时间向任意的command queue提交command list，运行时会按照预先排列好的提交顺序，自动序列化command queue中的command list。

#### Command Queue

command queue用来提交并执行command list。这种架构方式允许开发者更高效的使用cpu和gpu。
* command queue的使用可以让开发者避免由于意外的同步造成的效率损失。
* command queue的使用可以让开发者在更高级别的同步发生时，使用更加高效（多个线程同时提交）精确（使用下标专门提交某个command list）的方式。这意味着运行时和图形驱动将在工程化的并发中花费更少的时间。
* command queue的使用可以使大消耗的操作更有效率。

command queue的结构使接口达成了这些改进:
1. 增加并发：app可以在进行前台工作（如渲染）时，同时进行更深层次的后台工作（如解码视频编码）。
2. 同步执行低优先级的gpu工作：command queue结构允许gpu在一个非同步线程中无锁的执行低优先级gpu工作和原子操作。
3. 高优先级工作：command queue的设计允许脚本打断3d渲染工作，去做少量高优先级计算，以便cpu能够尽早获得结果。
4. 每个command queue只能够保存并提交一个类型的command list。

### Descriptor Heaps (DH)

cpu修改创建，用来保存和提交descriptor。
* 可以创建完整的heap来保存提交新的descriptor。也可以重新分配空间来修改原有的提交顺序。
* 在被command list引用之前，可以直接由cpu编辑修改。但当引用DH的command list被提交后，该DH则不能被修改。
* dx12必须通过DH来访问descriptor。
* DH可以通过以下策略进行管理：
    1. 为下次draw call填充一个fresh area。在每次command list访问时，移动DT pointer到fresh开头。优点：可以避免记录特定DT pointer的位置。缺点：DH上很可能有许多重复的descriptor，当渲染相同或近似场景时，DH的内存将很快被用完。对于那些在cpu上记录，在gpu上渲染的DH，这种策略需要避免产生访问冲突。(动态填充策略)
    2. 预先根据descriptor的需要创建DH作为一个场景的一部分，之后在绘制时仅仅设置DT即可。
    3. 将DH当作一个包含所有需要的descriptor和其所在位置的大数组。draw call通过index去访问固定区段的heap内容。（预先填充策略）
* 确保root constants and root descriptors通过read/write来访问，而非完整的重新创建，可以在大多数硬件上进一步提升DH性能。

#### Descriptor

用gpu非公开的数据格式描述一个gpu对象的小区块。包含以下四类（PS：其中1，2，3可以在同个heap中存在，4则需要单独heap保存。）：
* Shader Resource Views (SRVs)
* Unordered Access Views (UAVs)
* Constant Buffer Views (CBVs)
* Samplers
硬件驱动不会保有descriptor。驱动仅仅绑定render target以保证swap chain工作正常。由于硬件不会保有descriptor对象，每个对象需要保有驱动所需要的资源地址以便硬件访问。

#### Descriptor Table（DT）

用来引用DH上一段descriptor的结构。使硬件可以使用更加快捷轻量的方式访问drescriptor。DT以在图形管线生成一个root signature并使用索引的方式来引用DT保有的资源。

### Graphic Pipeline State

#### The Direct3d 12 Pipelines

在说明d3d12 pipeline之前，首先描述一下dx11的render pipeline。本质上，dx12的pipeline是针对该版本的扩展。</br>

d3d11 pipeline大体分为以下阶段：
1. Input Assmebler
    * 使用primitive type组织primitive data，提供其他阶段使用。
    * 链接系统默认值以便帮助shader更加有效的执行。
2. Vertex Shader
    * 处理从IA阶段传入的顶点数据。
    * 每个顶点产生一次数据，计算并产生一次输出传给下个阶段。（理论上有毒少个顶点就会有多少次Vertex Shader调用
    * VS一次最大允许16个32位浮点数的传入和传出。
    * VS必须存在并且必须传出一个值。
3. Tessellation
    * 实际用来细分区面，在早期管线中，这部分由cpu实现。现在将它移动到gpu。
    * Tessellation做的事，是顶点和面数较小的图元上，通过计算增加顶点和面数，并传给下个阶段。
    * Tessellation阶段的意义在于，可以减少数据的输入和上载数量，但必然的会消耗gpu运算效率。
    * Tessellation阶段一般由硬件实现，它有两个相关的可编程阶段
        1. Hull shader
            * 每个三角面片调用一次。把输入的低面数的控制点，扩展成更高的面数的控制点。比如3个控制点的三角形扩展成9个控制点的三个三角形。从而构成更多的表面细节。
            * 单次最多输入1-32个控制点。最多输出1-32个控制点，输出的控制点数跟镶嵌参数(tessellation factors)无关。从Hull输出的控制点和面片常量(path constant data)可以被Domain消耗。输出的Tessellation Factors挥别Tesselator使用。
            * Tessellation factors决定面片被细分的程度。
        2. Domain shader
            * Tesselator每输出一个纹理坐标运行一次。
            * 使用Hull输出的图元控制点。
            * 输出顶点位置。
4. Geometry Shader
5. Rasterizer
6. Pixel Shader
7. Output Merge