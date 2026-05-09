---
title: "Unity 2D 小游戏开发指南：从零到抖音发布"
date: 2026-05-09
draft: false
tags: ["Unity", "2D游戏", "抖音小游戏", "C#", "游戏开发"]
categories: ["技术", "游戏开发"]
summary: "从 Unity Hub 安装到抖音小游戏发布的完整指南：环境搭建、技术储备、开发流程、抖音适配与上架。"
showToc: true
TocOpen: true
weight: 1
series: ["Unity游戏开发"]
---

## 一、环境搭建

### 1.1 安装 Unity Editor

1. 启动 Unity Hub

```bash
unity-hub
```

2. 安装 Editor：推荐 **Unity 2022.3 LTS**（长期支持，抖音 SDK 兼容性最好）

安装时勾选模块：
- ✅ Linux Build Support (IL2CPP)
- ✅ WebGL Build Support（抖音小游戏核心）
- ✅ Visual Studio / VS Code（代码编辑器）
- ✅ 中文语言包

3. 激活许可证：Unity Hub → 设置 → 许可证 → 个人版（免费，年收入 < 10 万美元）

### 1.2 VS Code 配置

```bash
# 安装 C# 开发工具
code --install-extension ms-dotnettools.csharp
code --install-extension unity.unity-debug
```

Unity → Edit → Preferences → External Editor → 选择 VS Code

### 1.3 项目创建

Unity Hub → 新建项目 → 选择 **2D(Core)** 模板 → 命名 → 创建

> 不要选 2D URP，那个是给高端 2D 的，小游戏用 Core 就够了。

---

## 二、必须储备的技术

### 2.1 C# 基础（1-2 周）

这是 Unity 的脚本语言，必须掌握：

```
优先级    内容                    用途
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
P0      变量、类型、运算符        基础中的基础
P0      if/else、for/while       控制流
P0      函数、参数、返回值        代码组织
P0      类、对象、继承           面向对象
P1      枚举 enum               状态机
P1      协程 IEnumerator         动画、延迟、计时器
P1      委托 delegate / Action   事件系统
P1      泛型 List<T>            数据管理
P2      接口 interface           解耦
P2      序列化 [Serializable]   存档
```

**推荐学习路径**：
1. B站搜"C# 入门"（3-5小时速通）
2. 边做游戏边学，不要纯看视频

### 2.2 Unity 核心概念（1 周）

| 概念 | 说明 | 类比 |
|------|------|------|
| GameObject | 游戏中的物体 | 舞台上的演员 |
| Component | 挂在物体上的功能 | 演员的技能 |
| Transform | 位置/旋转/缩放 | 演员站在哪 |
| MonoBehaviour | 脚本基类 | 演员的剧本 |
| Prefab | 预制体，可复用 | 演员的模板 |
| Scene | 场景 | 舞台布景 |
| Asset | 资源（图片/音频/脚本） | 道具库 |
| Tag / Layer | 标签/层级 | 演员分类 |

### 2.3 Unity 2D 核心（1 周）

```
Sprite          → 2D 图片素材
SpriteRenderer  → 显示图片
Rigidbody2D     → 2D 物理引擎（重力/碰撞）
Collider2D      → 2D 碰撞体
Animator        → 动画状态机
Animation       → 关键帧动画
Canvas          → UI 系统
Tilemap         → 2D 地图编辑
```

### 2.4 必须掌握的 Unity API

```csharp
// 每帧执行
void Update() { }

// 物理帧（固定间隔，处理物理）
void FixedUpdate() { }

// 碰撞检测
void OnCollisionEnter2D(Collision2D col) { }
void OnTriggerEnter2D(Collider2D col) { }

// 输入
Input.GetKeyDown(KeyCode.Space)
Input.GetAxisRaw("Horizontal")

// 移动
transform.Translate(Vector3.right * speed * Time.deltaTime)
rigidbody2D.velocity = new Vector2(moveX * speed, rb.velocity.y)

// 实例化/销毁
Instantiate(prefab, position, rotation)
Destroy(gameObject, delay)

// 查找
GameObject.Find("Name")
GetComponent<T>()
FindObjectOfType<T>()

// 协程（延迟/计时）
StartCoroutine(MyCoroutine());
IEnumerator MyCoroutine() {
    yield return new WaitForSeconds(2f);
    // 2秒后执行
}

// 场景切换
SceneManager.LoadScene("SceneName");

// 存档
PlayerPrefs.SetInt("Score", 100);
PlayerPrefs.GetInt("Score", 0);
```

### 2.5 抖音小游戏特殊要求

| 技术 | 说明 |
|------|------|
| WebGL 构建 | 抖音小游戏基于 WebGL |
| 触屏操作 | 不依赖键盘，全触屏 |
| 包体控制 | 首包 < 20MB，总包 < 200MB |
| 适配比例 | 竖屏 9:16 或横屏 16:9 |
| 抖音 SDK | 登录、分享、广告、录屏 |
| 内存控制 | 手机端 < 1GB 内存占用 |

---

## 三、开发流程

### 3.1 推荐的第一款游戏：Flappy Bird 类

为什么选这个：
- 代码量最小（200 行 C#）
- 只需要 3 个素材（鸟、管道、背景）
- 涵盖核心机制：物理、碰撞、计分、UI
- 1-3 天可以完成

### 3.2 完整开发步骤

```
Step 1: 素材准备
  ├── 背景图（960×540 或 1080×1920）
  ├── 角色 Sprite（带动画帧）
  ├── 障碍物/道具
  └── 音效 BGM（免版权：freesound.org）

Step 2: 场景搭建
  ├── 创建 GameObject
  ├── 添加 SpriteRenderer
  ├── 添加 Rigidbody2D / Collider2D
  └── 设置 Tag / Layer

Step 3: 脚本编写
  ├── PlayerController.cs    玩家控制
  ├── ObstacleSpawner.cs     障碍生成
  ├── GameManager.cs         游戏状态/计分
  └── UIManager.cs           UI 交互

Step 4: UI 制作
  ├── Canvas → Screen Space - Overlay
  ├── 开始界面（标题 + 开始按钮）
  ├── 游戏中 HUD（分数）
  └── 结束界面（分数 + 重新开始）

Step 5: 打磨
  ├── 粒子效果（得分/死亡）
  ├── 音效
  ├── 动画（角色飞/旋转）
  └── 难度递增

Step 6: WebGL 构建
  └── File → Build Settings → WebGL → Build
```

### 3.3 典型脚本示例

**PlayerController.cs**：

```csharp
using UnityEngine;

public class PlayerController : MonoBehaviour
{
    public float jumpForce = 5f;
    private Rigidbody2D rb;

    void Start()
    {
        rb = GetComponent<Rigidbody2D>();
    }

    void Update()
    {
        if (Input.GetMouseButtonDown(0))
        {
            rb.velocity = Vector2.zero;
            rb.AddForce(Vector2.up * jumpForce, ForceMode2D.Impulse);
        }
    }

    void OnCollisionEnter2D(Collision2D col)
    {
        GameManager.Instance.GameOver();
    }
}
```

**GameManager.cs**：

```csharp
using UnityEngine;
using UnityEngine.SceneManagement;

public class GameManager : MonoBehaviour
{
    public static GameManager Instance;
    public int score;
    public bool isGameOver;

    void Awake()
    {
        Instance = this;
    }

    public void AddScore()
    {
        if (isGameOver) return;
        score++;
    }

    public void GameOver()
    {
        isGameOver = true;
    }

    public void Restart()
    {
        SceneManager.LoadScene(SceneManager.GetActiveScene().name);
    }
}
```

---

## 四、抖音小游戏适配

### 4.1 抖音小游戏开发流程

```
Unity WebGL 构建
  → 转换为抖音小游戏格式（使用 douyin-convert 工具）
  → 抖音开发者工具调试
  → 提交审核
  → 上架
```

### 4.2 抖音小游戏 SDK

```bash
# 抖音官方 Unity SDK
https://github.com/nicewarm/DouyinMiniGameUnity
```

核心能力：

| 能力 | API | 用途 |
|------|-----|------|
| 登录 | tt.login() | 获取用户身份 |
| 分享 | tt.shareAppMessage() | 社交传播 |
| 广告 | tt.createRewardedVideoAd() | 变现 |
| 录屏 | tt.getGameRecorderManager() | 短视频传播 |
| 排行榜 | tt.setUserCloudStorage() | 竞技社交 |
| 支付 | tt.pay() | 内购 |

### 4.3 构建优化

```javascript
// 抖音小游戏对 WebGL 构建的特殊要求

// 1. 压缩格式：Brotli
// Unity → Project Settings → Player → WebGL → Compression Format → Brotli

// 2. 代码剥离：High
// Unity → Project Settings → Player → Managed Stripping Level → High

// 3. IL2CPP（必须）
// Unity → Project Settings → Player → Scripting Backend → IL2CPP

// 4. 包体优化
// - 使用 Sprite Atlas 打包图集
// - 压缩纹理（ASTC/ETC2）
// - 移除未使用的模块
// - Addressables 按需加载
```

### 4.4 抖音开发者账号

1. 注册：https://developer.open-douyin.com/
2. 创建小游戏应用
3. 获取 AppID
4. 下载抖音开发者工具（仅 Windows/Mac，Linux 需用 Windows 虚拟机或借电脑）

> **注意：抖音开发者工具目前只有 Windows 和 Mac 版本，Linux 不支持。** 构建好的 WebGL 包需要在 Windows/Mac 上用开发者工具转换和调试。

---

## 五、技术储备路线图

### 5.1 时间规划

```
第 1 周：C# 基础 + Unity 界面熟悉
  ├── C# 语法速通（变量/循环/类/继承）
  ├── Unity 界面操作（Hierarchy/Inspector/Scene/Game）
  └── 跟着做一个简单 Demo

第 2 周：2D 游戏核心
  ├── 物理系统（Rigidbody2D / Collider2D）
  ├── 动画系统（Animator / Animation）
  ├── UI 系统（Canvas / Button / Text）
  └── 完成 Flappy Bird 克隆

第 3 周：完整游戏
  ├── 素材制作（Aseprite / Piskel）
  ├── 音效集成
  ├── 关卡设计
  ├── 存档系统
  └── 打磨发布

第 4 周：抖音适配
  ├── WebGL 构建
  ├── 抖音 SDK 集成
  ├── 开发者工具调试
  └── 提交审核
```

### 5.2 技能树

```
Unity 2D 小游戏开发
├── 编程
│   ├── C# 基础 ★★★★★
│   ├── 面向对象 ★★★★
│   ├── 设计模式（单例/状态机/观察者）★★★
│   └── 协程/异步 ★★★
├── Unity 引擎
│   ├── 2D 物理系统 ★★★★★
│   ├── 动画系统 ★★★★
│   ├── UI 系统 ★★★★★
│   ├── 音频系统 ★★★
│   ├── 粒子系统 ★★★
│   └── 场景管理 ★★★
├── 美术
│   ├── 像素画基础 ★★★
│   ├── Sprite 动画帧 ★★★
│   ├── Tilemap 地图 ★★★
│   └── UI 设计 ★★★
└── 发布
    ├── WebGL 构建 ★★★★★
    ├── 抖音 SDK ★★★★
    ├── 包体优化 ★★★★
    └── 性能优化 ★★★
```

---

## 六、免费素材资源

| 资源 | 网址 | 说明 |
|------|------|------|
| Kenney | kenney.nl/assets | 最全的免费 2D 素材 |
| OpenGameArt | opengameart.org | 开源游戏素材 |
| Itch.io | itch.io/game-assets/free | 独立游戏素材 |
| Freesound | freesound.org | 免费音效 |
| Piskel | piskelapp.com | 在线像素画编辑器 |
| Aseprite | aseprite.org | 像素画神器（付费） |
| Unity Asset Store | assetstore.unity.com | 免费专区 |

---

## 七、避坑指南

### 7.1 常见新手坑

| 坑 | 解法 |
|----|------|
| 碰撞不生效 | 检查两个物体都有 Collider2D，至少一个有 Rigidbody2D |
| 触发器 vs 碰撞体 | Trigger 穿透用 OnTriggerEnter2D，碰撞用 OnCollisionEnter2D |
| 移动卡顿 | 物理移动放 FixedUpdate，用 velocity 而不是 AddForce |
| UI 点击穿透 | EventSystem 组件不能删 |
| 手机触屏无效 | 用 Input.GetMouseButtonDown(0)，触屏和鼠标通用 |
| 包体太大 | Sprite Atlas + 代码剥离 + 压缩纹理 |
| WebGL 黑屏 | 检查浏览器控制台报错，通常是内存不足 |

### 7.2 抖音小游戏特殊坑

| 坑 | 解法 |
|----|------|
| 开发者工具无 Linux 版 | 借 Windows 电脑或用虚拟机 |
| WebGL 构建失败 | 切换 IL2CPP + 清 Library 重新构建 |
| 首包超 20MB | Addressables 分包加载 |
| 分享功能不生效 | 需要在抖音后台配置分享素材 |
| 审核被拒 | 先看抖音小游戏审核规范，注意无版号不能内购 |

---

## 八、推荐学习资源

### 8.1 视频

| 资源 | 说明 |
|------|------|
| B站 M_Studio | 最适合新手的 Unity 2D 教程 |
| B站 麦扣 M_Studio | 2D RPG 系列教程 |
| Unity Learn 官方 | 免费系统课程 |
| YouTube Brackeys | 经典英文教程（虽已停更但仍经典） |

### 8.2 文档

| 资源 | 网址 |
|------|------|
| Unity 官方文档 | docs.unity3d.com |
| 抖音小游戏文档 | developer.open-douyin.com/docs |
| C# 官方教程 | learn.microsoft.com/zh-cn/dotnet/csharp |

### 8.3 推荐做游戏顺序

```
1. Flappy Bird（1-3天）     → 学物理、碰撞、UI
2. 2048（2-3天）            → 学逻辑、动画、UI
3. 消消乐（1-2周）          → 学算法、特效、关卡
4. 跑酷游戏（1-2周）        → 学对象池、地图生成、难度曲线
5. 自己的创意（2-4周）      → 综合实战
```

> **最重要的是：尽快发布第一个游戏，哪怕很丑很简陋。发布比完美更重要。**
