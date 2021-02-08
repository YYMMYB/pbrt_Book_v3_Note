# 03 pbrt 代码结构概览

| 基类         | 源码目录      | 章节  |
| ------------ | ------------- | ----- |
| `Shape`      | shapes/       | 3.1   |
| `Aggregate`  | accelerators/ | 4.2   |
| `Camera`     | cameras/      | 6.1   |
| `Sampler`    | samplers/     | 7.2   |
| `Filter`     | filters/      | 7.8   |
| `Material`   | materials/    | 9.2   |
| `Texture`    | textures/     | 10.3  |
| `Medium`     | media/        | 11.3  |
| `Light`      | lights/       | 12.2  |
| `Integrator` | integrators/  | 1.3.3 |

## 整个流程

1. 首先解析场景文件包括：
   1. 几何形状
   2. 材质
   3. 灯光
   4. 相机
2. 得到 `Scene` 与 `Integrator` 的实例
3. 执行主渲染循环, 最终用 `Integrator::Render()` 生成图像

参考源码时注意切换分支到 book !!!
主函数 `main()` 在 main\pbrt.cpp 中, 执行过程如下:

1. `options = 命令行参数`
2. `pbrtInit(options);`
3. 解析文件, 最终用 `RenderOptions::MakeScene()` 创建 `Scene` 实例.
4. 通过 `pbrtWorldEnd()` 调用 `RenderOptions::MakeIntegrator()` 创建 `Integrator`, 生成图像.

## 场景表示

- `Scene` 在 core/scene.h, core/scene.cpp 中
  - `List<Light>` 一个场景才有一个灯光列表, 而非每个物体都有自己的灯光列表.
    因为这样更符合物理规律.
  - `aggregate : Primitive`, `aggregate`是比较特殊的`Primitive`
    - 只保存了场景中物体的**引用**.
    - 但也实现了 `Primitive` 接口.
    - 使用高速数据结构, 避免对于在射线远处物体, 进行多余的射线检测.

场景中所有的几何物体, 都用某种`Primitive` 表示, 且内部包括:

- `Shap` 描述形状
- `Material` 描述"颜色"(即外观)

这些几何物体最后会被收集到`aggregate`中.

在场景都构建完后, 即构造函数的最后, 会向每个灯光发个消息, 对某些灯光会很有用:

```cpp
for (const auto &light : lights) light->Preprocess(*this);
```

`Scene::IntersectP()` 通常比 `Scene::Intersect()` 快, 因为不需要计算最近的交点.

## Integrator 接口 与 SamplerIntegrator

渲染图片的过程是通过 `Integrator` 完成的(第14,15,16章).

- `Integrator`在 core/integrator.h, core/integrator.cpp 中
  在 integrators/ 文件夹中有各种实现.
  - `Render()`

- `SamplerIntegrator` 通过采样流, 来产生图像.
  - `Preprocess()` 可选是否实现, 作用是在场景初始化完成后, 去做一些依赖于场景的计算
    (如, 依赖于光源总数的高速数据结构, 预计算整个场景的 radiance 分布).
  - `Sampler`
    - 选出需要进行光线跟踪的点
    - 还有, 选出计算 <dev id="reref_1">[BSDF积分](#ref_1)</dev> 所需的位置. 如, 面光源上选择哪些点来计算光照(第7章).
  - `Camera` 控制视图, 镜头的参数(第6章)
    - `Film` 存储图像. 写入文件或显示到屏幕(第7.9章)

## 主渲染循环

在创建了 `Scene` 与 `Integrator` 后, 就会执行 `Integrator::Render()`, 进行渲染.
![SamplerIntegrator::Render() 的数据流](img/Class%20Relationships.svg)
其中 `Li()` 会计算某条光线(Ray)到达画面的辐射(radiance) *关于辐射这些概念这种东西, 参考[这里](A_辐射.md)*

渲染时, 会将图像分解为多个像素的 tile, 每个 tile 都可以并行计算

`SamplerIntegrator::Render()` 的流程:
1. `Preprocess(scene, *sampler);`
2. 并行渲染每个 tile
3. 保存整个图像

渲染 tile 的流程

1. 计算 tile 总数
2. 对每个 tile 并行渲染
   1. 声明当前 tile 的 arena.
      `Li()`方法的实现通常将需要为每个辐射度计算临时分配少量内存.
      大量的内存分配结果很容易使系统的常规内存分配例程（例如malloc（）或new）不堪重负, 该例程必须维护并同步精心设计的内部数据结构, 以跟踪处理器之间的空闲内存区.
      我们把 `arena : MemoryArena` 传入 `Li()`, 来管理内存.
      `MemoryArena` 的实例不允许被未经同步的多个线程访问.
   3. 取得 `tileSampler : Sampler`, 通过克隆.
   4. 计算当前 tile 的采样边界.
   5. 取得当前 tile 的 `filmTile : FilmTile`.
      `FilmTile` 会提供一个小缓存, 来存储当前 tile 中每个像素的值.
   6. 渲染 tile 中的每个像素.
      `tileSampler->StartPixel(pixel)` 开始采样. `pixel : Point2i`
      `tileSampler->StartNextSample()` 判断是否还需要继续采样.
      1. 初始化 `cameraSample : CameraSample = tileSampler->GetCameraSample(pixel)`, 它会记录产生图片上哪些位置需要产生射线.
      2. 产生当前采样的射线.
         `rayWeight` 表示当前射线的权重. 复杂的相机模型中, 到达胶片边缘 可能 比到达中心的射线, 权重更小.
      3. 用`Li()`计算当前射线的辐射, 这是个纯虚函数, 每个子类必须实现该方法.
          会返回`Spectrum`, 表示射线起点的 入射辐射(incident radiance).
          其具有如下参数:
         1. ray
         2. scene
         3. sampler: 通过蒙特卡洛积分计算光传输方程(light transport equation)时要用的采样器.
         4. arena: 快速分配临时的内存. integrator 会假设在每次`Li()`执行完后, 这部分内存会被快速释放. 
            所以不应该用这个来分配比当前射线的生命周期更长的内存.
         5. depth: 在执行这次`Li()`之前, 光线以及从相机反射了多少次.
      4. 然后就可以根据辐射更新图像了, 通过`FilmTile::AddSample()`
      5. 最后释放内存, 通过`MemoryArena::Reset()`
   7. 将`filmTile`交给最终图像, 通过`Film’s MergeFilmTile()`

`Camera::GenerateRay()` 会根据采样点来产生射线.
`Camera::GenerateRayDifferential()` 会给出 在图像上有一个像素差异的采样, 所产生的射线之间的信息.
Ray differentials 用于从第10章中某些纹理函数中获得更好的结果, 
从而可以计算纹理相对于像素间距的变化速度，这是纹理抗锯齿的关键组成部分.

# Whitted 射线追踪的 Integrator

这里实现基于 Whitted’s ray-tracing algorithm 的Integrator. 这个算法, 对于反射和折射光会进行精确计算, 但是无法计算间接照明.
代码在 integrators/whitted.h, integrators/whitted.cpp 

该算法会递归的计算反射与折射, 直到到达最大深度`WhittedIntegrator::maxDepth`.
数据流如图所示:
![WhittedIntegrator 的数据流](img/Surface%20Integration%20Class%20Relationships.svg)


----
- <dev id="ref_1">[BSDF的积分](#reref_1)</dev>
{% math %}L_{0}\left(\mathrm{p}, \omega_{0}\right)=L_{\mathrm{e}}\left(\mathrm{p}, \omega_{0}\right)+\int_{\mathrm{S}^{2}} f\left(\mathrm{p}, \omega_{0}, \omega_{\mathrm{i}}\right) L_{\mathrm{i}}\left(\mathrm{p}, \omega_{\mathrm{i}}\right)\left|\cos \theta_{\mathrm{i}}\right| \mathrm{d} \omega_{\mathrm{i}}{% endmath %}

