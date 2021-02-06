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
3. 执行主渲染循环, `Integrator::Render()` 生成图像

## 场景表示

参考源码时注意切换分支到 book !!!
主函数 `main()` 在 main\pbrt.cpp 中, 执行过程如下:

1. `options = 命令行参数`
2. `pbrtInit(options);`
3. 解析文件, 最终用 `RenderOptions::MakeScene()` 生成 `Scene` 实例.

`Scene` 定义在  core/scene.h, core/scene.cpp 中, 结构如下:

- `Scene`
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
