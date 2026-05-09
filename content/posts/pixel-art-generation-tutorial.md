---
title: "卡牌像素画生成教程：自己动手做素材"
date: 2026-05-09
draft: false
tags: ["游戏开发", "像素画", "Python", "Pillow", "卡牌素材"]
categories: ["技术", "游戏开发"]
summary: "用 Python Pillow 代码生成卡牌像素画的完整教程：环境搭建、像素画原理、单位绘制、卡牌合成、批量生产。"
showToc: true
TocOpen: true
weight: 3
series: ["红警卡牌"]
---

## 一、环境搭建

### 1.1 安装 Pillow

```bash
pip3 install Pillow
```

### 1.2 验证

```bash
python3 -c "from PIL import Image; print('OK')"
```

### 1.3 工作目录

```bash
mkdir -p ~/soft/card-game-assets/sprites
cd ~/soft/card-game-assets/sprites
```

---

## 二、像素画原理

### 2.1 核心概念

像素画就是**在网格上逐像素填充颜色**。每个"像素"放大 4 倍显示，即逻辑坐标 (x, y) 对应画布上 (x×4, y×4) 的 4×4 方块。

```
逻辑坐标(2,3) → 画布坐标(8,12)到(11,15) → 一个4×4的方块
```

### 2.2 基础操作函数

```python
from PIL import Image, ImageDraw, ImageFont

PIXEL = 4  # 每个逻辑像素在画布上占4×4

def px(draw, x, y, color):
    """画一个逻辑像素"""
    draw.rectangle(
        [x*PIXEL, y*PIXEL, (x+1)*PIXEL-1, (y+1)*PIXEL-1],
        fill=color
    )

def px_rect(draw, x1, y1, x2, y2, color):
    """画一个逻辑矩形"""
    draw.rectangle(
        [x1*PIXEL, y1*PIXEL, (x2+1)*PIXEL-1, (y2+1)*PIXEL-1],
        fill=color
    )

def px_circle(draw, cx, cy, r, color):
    """画一个逻辑圆形"""
    for y in range(cy-r, cy+r+1):
        for x in range(cx-r, cx+r+1):
            if (x-cx)**2 + (y-cy)**2 <= r**2:
                px(draw, x, y, color)

def px_line(draw, x1, y1, x2, y2, color):
    """画一条逻辑线"""
    steps = max(abs(x2-x1), abs(y2-y1), 1)
    for i in range(steps+1):
        t = i / steps
        x = int(x1 + t*(x2-x1))
        y = int(y1 + t*(y2-y1))
        px(draw, x, y, color)
```

### 2.3 调色板

```python
PALETTE = {
    # 阵营色
    "soviet_red":       (140, 50, 40),   # 苏军红
    "soviet_dark":      (100, 35, 25),   # 苏军深红
    "allied_blue":      (50, 80, 160),   # 盟军蓝
    "allied_dark":      (35, 55, 120),   # 盟军深蓝
    "empire_purple":    (160, 50, 160),  # 帝国紫
    "empire_dark":      (110, 35, 110),  # 帝国深紫
    
    # 金属色
    "steel":            (120, 130, 140), # 钢铁
    "steel_light":      (160, 170, 180), # 亮钢
    "steel_dark":       (80, 85, 95),    # 深钢
    
    # 皮肤色
    "skin":             (200, 170, 140), # 皮肤
    "skin_dark":        (170, 140, 110), # 深皮肤
    
    # 特效色
    "fire":             (255, 120, 30),  # 火焰
    "fire_bright":      (255, 200, 50),  # 亮火
    "blue_glow":        (80, 180, 255),  # 蓝光（盟军）
    "red_glow":         (255, 80, 80),   # 红光（苏军）
    "purple_glow":      (180, 80, 255),  # 紫光（帝国）
    "gold_glow":        (255, 220, 80),  # 金光
    
    # 基础色
    "white":            (255, 255, 255),
    "black":            (0, 0, 0),
    "bg_dark":          (30, 30, 50),    # 卡牌背景
}
```

---

## 三、绘制步兵

### 3.1 步兵模板

步兵是 16×16 逻辑像素，放大后 64×64 像素：

```python
def gen_infantry(filename, faction, gun_len=4, has_gun=True,
                  helmet_extra=None):
    W, H = 16, 16  # 逻辑像素尺寸
    img = Image.new('RGBA', (W*PIXEL, H*PIXEL), (0,0,0,0))
    d = ImageDraw.Draw(img)
    
    body_c = PALETTE[faction]
    dark_c = PALETTE[faction.replace("_red","_dark").replace("_blue","_dark").replace("_purple","_dark")]
    cx = W // 2
    
    # 1. 头部（圆形）
    px_circle(d, cx, 4, 2, PALETTE["skin"])
    
    # 2. 头盔
    px_rect(d, cx-3, 1, cx+3, 3, body_c)
    if helmet_extra:  # 特殊头盔装饰
        for p in helmet_extra:
            px(d, p[0]+cx, p[1], dark_c)
    
    # 3. 身体
    px_rect(d, cx-3, 6, cx+3, 11, body_c)
    
    # 4. 阵营徽章
    badge_c = {"soviet_red": PALETTE["red_glow"],
               "allied_blue": PALETTE["gold_glow"],
               "empire_purple": PALETTE["purple_glow"]}[faction]
    px(d, cx, 8, badge_c)
    px(d, cx-1, 8, badge_c)
    
    # 5. 腿
    px_rect(d, cx-3, 12, cx-1, 15, PALETTE["brown_dark"])
    px_rect(d, cx+1, 12, cx+3, 15, PALETTE["brown_dark"])
    
    # 6. 武器
    if has_gun:
        px_line(d, cx+4, 7, cx+4+gun_len, 7, PALETTE["steel"])
        px_line(d, cx+4, 8, cx+4+gun_len, 8, PALETTE["steel"])
    
    img.save(filename)
```

### 3.2 调用示例

```python
# 美国大兵（1费白卡）
gen_infantry("gi_soldier.png", "allied_blue", gun_len=4)

# 动员兵（1费白卡）
gen_infantry("mobilize_soldier.png", "soviet_red", gun_len=4)

# 狙击手（3费蓝卡，长枪）
gen_infantry("sniper.png", "allied_blue", gun_len=8)

# 磁暴步兵（3费蓝卡，特殊头盔）
gen_infantry("tesla_soldier.png", "soviet_red", gun_len=5,
             helmet_extra=[(0,2)])  # 头盔顶部装饰

# 间谍（3费紫卡，无武器）
gen_infantry("spy.png", "allied_blue", has_gun=False)

# 帝国武士（1费白卡）
gen_infantry("samurai.png", "empire_purple", gun_len=5)
```

### 3.3 自定义技巧

| 想做的效果 | 怎改参数 |
|-----------|---------|
| 更大的枪 | `gun_len=8` |
| 双枪 | 画两条 `px_line` |
| 无武器（法师类） | `has_gun=False` |
| 特殊头盔 | `helmet_extra=[(x,y),(x,y)]` |
| 不同肤色 | 改 `PALETTE["skin"]` |

---

## 四、绘制载具

### 4.1 坦克模板

坦克是 32×32 逻辑像素，放大后 128×128 像素：

```python
def gen_tank(filename, faction, turret_w=8, gun_len=6,
              double_gun=False, heavy=False):
    W, H = 32, 32
    img = Image.new('RGBA', (W*PIXEL, H*PIXEL), (0,0,0,0))
    d = ImageDraw.Draw(img)
    
    body_c = PALETTE[faction]
    cx = W // 2
    bw = 12 if heavy else 10  # 车体宽度
    
    # 1. 履带
    track_y = H - 6
    px_rect(d, cx-bw//2-1, track_y, cx+bw//2+1, H-2,
            PALETTE["steel_dark"])
    # 履带细节（每隔2像素一条线）
    for x in range(cx-bw//2, cx+bw//2+1):
        if x % 2 == 0:
            px(d, x, track_y+1, PALETTE["steel"])
    
    # 2. 车体
    bh = 10 if heavy else 8
    px_rect(d, cx-bw//2, track_y-bh, cx+bw//2, track_y-1, body_c)
    
    # 3. 车体线条细节
    for x in range(cx-bw//2+1, cx+bw//2):
        if x % 3 == 0:
            px(d, x, track_y-bh+2, PALETTE[faction+"_dark"])
    
    # 4. 炮塔
    th = 4 if heavy else 3
    px_rect(d, cx-turret_w//2, track_y-bh-th,
            cx+turret_w//2, track_y-bh-1,
            PALETTE["turret"])
    
    # 5. 火炮
    if double_gun:  # 天启坦克双管
        px_rect(d, cx+turret_w//2, track_y-bh-th+1,
                cx+turret_w//2+gun_len, track_y-bh-th+2,
                PALETTE["steel_light"])
        px_rect(d, cx-turret_w//2-gun_len, track_y-bh-th+1,
                cx-turret_w//2, track_y-bh-th+2,
                PALETTE["steel_light"])
        # 炮口火焰
        px(d, cx+turret_w//2+gun_len, track_y-bh-th+1,
           PALETTE["fire"])
        px(d, cx-turret_w//2-gun_len-1, track_y-bh-th+1,
           PALETTE["fire"])
    else:  # 单管
        px_rect(d, cx+turret_w//2, track_y-bh-th+1,
                cx+turret_w//2+gun_len, track_y-bh-th+2,
                PALETTE["steel_light"])
        # 炮口特效（蓝光=盟军，火焰=苏军）
        if faction == "allied_blue":
            px(d, cx+turret_w//2+gun_len, track_y-bh-th+1,
               PALETTE["blue_glow"])
        else:
            px(d, cx+turret_w//2+gun_len, track_y-bh-th+1,
               PALETTE["fire"])
    
    img.save(filename)
```

### 4.2 调用示例

```python
# 轻坦克（2费白卡）
gen_tank("light_tank.png", "allied_blue", turret_w=6, gun_len=4)

# 光棱坦克（5费紫卡，长炮蓝光）
gen_tank("prism_tank.png", "allied_blue", turret_w=6, gun_len=8)

# 天启坦克（6费紫卡，重甲双管）
gen_tank("apocalypse.png", "soviet_red", turret_w=8,
         gun_len=6, double_gun=True, heavy=True)

# 幻影坦克（4费蓝卡）
gen_tank("mirage.png", "allied_blue", turret_w=5, gun_len=5)

# V3火箭（4费紫卡，超长炮管）
gen_tank("v3.png", "soviet_red", turret_w=4, gun_len=12)
```

### 4.3 自定义技巧

| 想做的效果 | 怎改参数 |
|-----------|---------|
| 重型坦克 | `heavy=True` |
| 双管炮 | `double_gun=True` |
| 长炮管 | `gun_len=12` |
| 大炮塔 | `turret_w=10` |
| 盟军蓝光 | faction=`"allied_blue"` |
| 苏军火焰 | faction=`"soviet_red"` |

---

## 五、绘制飞行器

### 5.1 飞艇模板

```python
def gen_airship(filename, faction):
    W, H = 32, 32
    img = Image.new('RGBA', (W*PIXEL, H*PIXEL), (0,0,0,0))
    d = ImageDraw.Draw(img)
    
    cx = W // 2
    body_c = PALETTE[faction]
    
    # 1. 气球（大圆）
    px_circle(d, cx, 8, 8, body_c)
    px_circle(d, cx, 8, 6, PALETTE[faction+"_dark"])
    
    # 2. 炸弹舱
    px_rect(d, cx-5, 16, cx+5, 20, PALETTE["steel_dark"])
    
    # 3. 引擎
    px_rect(d, cx-10, 18, cx-6, 22, PALETTE["steel_dark"])
    px_rect(d, cx+6, 18, cx+10, 22, PALETTE["steel_dark"])
    px(d, cx-8, 20, PALETTE["fire_bright"])  # 引擎火焰
    px(d, cx+8, 20, PALETTE["fire_bright"])
    
    # 4. 炸弹
    px(d, cx, 22, PALETTE["fire"])
    
    # 5. 阵营标志
    badge = {"soviet_red": PALETTE["red_glow"],
             "allied_blue": PALETTE["gold_glow"]}[faction]
    px(d, cx, 6, badge)
    px(d, cx-1, 7, badge); px(d, cx, 7, badge)
    px(d, cx+1, 7, badge)
    
    img.save(filename)
```

---

## 六、绘制建筑

### 6.1 建筑模板

```python
def gen_building(filename, btype, faction):
    W, H = 32, 32
    img = Image.new('RGBA', (W*PIXEL, H*PIXEL), (0,0,0,0))
    d = ImageDraw.Draw(img)
    
    cx = W // 2
    body_c = PALETTE[faction]
    
    if btype == "tower":  # 防御塔（光棱塔/磁暴线圈）
        px_rect(d, cx-5, H-8-4, cx+5, H-8, PALETTE["steel"])
        px_rect(d, cx-3, H-8-8, cx+3, H-8-4, body_c)
        glow = {"allied_blue": PALETTE["blue_glow"],
                "soviet_red": PALETTE["red_glow"]}[faction]
        px_circle(d, cx, H-8-6, 2, glow)
    
    elif btype == "mine":  # 矿场
        px_rect(d, cx-6, H-8, cx+6, H-2, PALETTE["steel_dark"])
        px_rect(d, cx-3, H-8-4, cx+3, H-8, PALETTE["gold_glow"])
        px_rect(d, cx-5, H-8+2, cx+5, H-8+4, body_c)
    
    elif btype == "barracks":  # 兵营/工厂
        px_rect(d, cx-5, H-8, cx+5, H-2, body_c)
        px_rect(d, cx-2, H-8-4, cx+2, H-8, PALETTE["steel_dark"])
        px(d, cx, H-8-2, PALETTE["fire"])
    
    elif btype == "superweapon":  # 超武（核弹/天气）
        px_rect(d, cx-5, H-8-4, cx+5, H-8, PALETTE["steel"])
        px_rect(d, cx-4, H-8-10, cx+4, H-8-4, body_c)
        glow = {"allied_blue": PALETTE["blue_glow"],
                "soviet_red": PALETTE["red_glow"]}[faction]
        px_circle(d, cx, H-8-8, 3, glow)
    
    img.save(filename)
```

---

## 七、卡牌合成

### 7.1 稀有度边框色

```python
RARITY_COLORS = {
    "白": (200, 200, 200),  # 普通
    "蓝": (50, 130, 255),   # 精良
    "紫": (160, 50, 220),   # 稀有
    "金": (255, 200, 50),   # 传说
    "红": (255, 50, 50),    # 史诗
}
```

### 7.2 合成函数

```python
def make_card(unit_img, rarity, cost, name, rarity_label):
    W, H = 256, 384
    card = Image.new('RGBA', (W, H), (0,0,0,0))
    d = ImageDraw.Draw(card)
    
    rc = RARITY_COLORS[rarity]
    
    # 1. 背景
    d.rectangle([8, 8, 248, 376], fill=(30,30,50))
    
    # 2. 稀有度边框
    d.rectangle([0, 0, 255, 383], outline=rc, width=6)
    d.rectangle([4, 4, 251, 379], outline=rc, width=2)
    
    # 3. 顶部稀有度条
    d.rectangle([8, 8, 248, 28], fill=rc)
    
    # 4. 底部信息条
    d.rectangle([8, 360, 248, 376], fill=rc)
    
    # 5. 费用圆圈（左上角）
    d.ellipse([12, 12, 52, 52], fill=(30,30,50), outline=rc, width=2)
    
    # 6. 费用数字
    try:
        font = ImageFont.truetype(
            "/usr/share/fonts/truetype/dejavu/DejaVuSans-Bold.ttf", 24)
    except:
        font = ImageFont.load_default()
    d.text((22, 16), str(cost), fill=(255,255,255), font=font)
    
    # 7. 单位图片（居中放大）
    uw, uh = unit_img.size
    scale = min(220/uw, 200/uh)
    u_scaled = unit_img.resize(
        (int(uw*scale), int(uh*scale)), Image.NEAREST)
    ux = (W - u_scaled.width) // 2
    uy = 80
    card.paste(u_scaled, (ux, uy), u_scaled)
    
    # 8. 卡牌名称
    font_name = ImageFont.truetype(
        "/usr/share/fonts/truetype/dejavu/DejaVuSans-Bold.ttf", 16)
    d.text((W//2 - len(name)*5, 300), name,
           fill=(255,255,255), font=font_name)
    
    # 9. 稀有度标签
    font_tag = ImageFont.truetype(
        "/usr/share/fonts/truetype/dejavu/DejaVuSans.ttf", 12)
    d.text((W//2 - len(rarity_label)*3, 330), rarity_label,
           fill=(180,180,180), font=font_tag)
    
    return card
```

### 7.3 完整制作一张卡

```python
# 步骤1: 生成单位像素画
gen_tank("my_tank.png", "soviet_red", turret_w=8,
         gun_len=6, double_gun=True, heavy=True)

# 步骤2: 读取并合成卡牌
unit = Image.open("my_tank.png")
card = make_card(unit, "紫", 6, "天启坦克", "稀有🟪")
card.save("card_apocalypse.png")
```

---

## 八、批量生产

### 8.1 卡牌配置表

把所有卡牌定义写成数据表，批量生成：

```python
CARDS = [
    # (输出文件名, 类型, 参数字典, 稀有度, 费用, 名称, 稀有度标签)
    ("card_gi",         "infantry", {"faction":"allied_blue"},             "白", 1, "美国大兵",     "普通⬜"),
    ("card_mobilize",   "infantry", {"faction":"soviet_red"},              "白", 1, "动员兵",       "普通⬜"),
    ("card_apocalypse", "tank",     {"faction":"soviet_red","heavy":True,
                                     "turret_w":8,"gun_len":6,
                                     "double_gun":True},                   "紫", 6, "天启坦克",     "稀有🟪"),
    ("card_kirov",      "airship",  {"faction":"soviet_red"},              "金", 7, "基洛夫飞艇",   "传说🟨"),
    ("card_nuke",       "building", {"btype":"superweapon",
                                     "faction":"soviet_red"},              "金", 9, "核弹井",       "传说🟨"),
]

for card_id, unit_type, kwargs, rarity, cost, name, rarity_label in CARDS:
    # 生成单位像素画
    if unit_type == "infantry":
        gen_infantry(f"unit_{card_id}.png", **kwargs)
    elif unit_type == "tank":
        gen_tank(f"unit_{card_id}.png", **kwargs)
    elif unit_type == "airship":
        gen_airship(f"unit_{card_id}.png", **kwargs)
    elif unit_type == "building":
        gen_building(f"unit_{card_id}.png", **kwargs)
    
    # 合成卡牌
    unit = Image.open(f"unit_{card_id}.png")
    card = make_card(unit, rarity, cost, name, rarity_label)
    card.save(f"{card_id}.png")
```

### 8.2 总览图

```python
# 生成所有卡牌的总览网格图
cards = [f for f in sorted(os.listdir(".")) if f.startswith("card_")]
cols = 8
rows = (len(cards) + cols - 1) // cols
CW, CH, GAP = 256, 384, 4
W = cols * CW + (cols-1) * GAP
H = rows * CH + (rows-1) * GAP

grid = Image.new('RGBA', (W, H), (20, 20, 30, 255))
for i, f in enumerate(cards):
    card = Image.open(f)
    x = (i % cols) * (CW + GAP)
    y = (i // cols) * (CH + GAP)
    grid.paste(card, (x, y))

grid.save("all_cards_grid.png")
```

---

## 九、进阶技巧

### 9.1 添加动画帧

```python
def gen_infantry_walk_frame(frame, filename, faction):
    """步兵走路动画帧，frame=0/1/2/3"""
    img = gen_infantry_raw(faction)
    d = ImageDraw.Draw(img)
    cx = 8
    
    # 根据帧号调整腿的位置
    offsets = [[0,0], [-1,1], [0,0], [1,-1]]  # 4帧循环
    off = offsets[frame]
    
    # 重画腿（带偏移，模拟走路）
    px_rect(d, cx-3+off[0], 12, cx-1+off[0], 15, PALETTE["brown_dark"])
    px_rect(d, cx+1-off[0], 12, cx+3-off[0], 15, PALETTE["brown_dark"])
    
    img.save(filename)
```

### 9.2 导入 Unity Sprite Sheet

```python
# 把多帧动画拼成 Sprite Sheet
frames = [Image.open(f"walk_{i}.png") for i in range(4)]
sheet_w = frames[0].width * len(frames)
sheet = Image.new('RGBA', (sheet_w, frames[0].height), (0,0,0,0))
for i, f in enumerate(frames):
    sheet.paste(f, (i * frames[0].width, 0))
sheet.save("soldier_walk_sheet.png")
```

Unity 中导入后设置：
- Sprite Mode: Multiple
- Pixel Per Unit: 64
- Filter Mode: Point (no-filter) ← **像素画必须用这个**
- 切片: Automatic 或手动指定每帧 64×64

### 9.3 修改调色板快速换色

```python
# 把所有苏军单位换成沙漠涂装（新赛季主题）
DESERT_PALETTE = {
    "soviet_red": (160, 140, 100),   # 沙漠黄替代红色
    "soviet_dark": (120, 100, 70),
}

# 批量替换
for f in os.listdir("."):
    if f.endswith(".png"):
        img = Image.open(f)
        pixels = img.load()
        for y in range(img.height):
            for x in range(img.width):
                p = pixels[x, y]
                if p[:3] == (140, 50, 40):
                    pixels[x, y] = (160, 140, 100, p[3])
                elif p[:3] == (100, 35, 25):
                    pixels[x, y] = (120, 100, 70, p[3])
        img.save(f"desert_{f}")
```

---

## 十、常见问题

| 问题 | 解决 |
|------|------|
| 字体找不到 | `apt install fonts-dejavu-core` 或用 `ImageFont.load_default()` |
| 图片模糊 | Unity 导入时 Filter Mode 设 **Point(no-filter)** |
| 透明背景不对 | 用 `Image.new('RGBA', ...)` 创建，`(0,0,0,0)` 表示透明 |
| 颜色不一致 | 统一用 PALETTE 字典，不要硬编码 RGB |
| 批量生成慢 | 47张卡 < 5秒，不影响 |

---

## 十一、源码位置

所有生成代码在：

```
/home/jackray/soft/card-game-assets/sprites/
```

关键文件：
- 生成脚本: `/tmp/gen_all_cards.py`（批量生成47张卡）
- 总览图: `all_cards_grid.png`
- 单张卡牌: `card_*.png`
- 单位原图: `apocalypse_tank.png`, `prism_tank.png` 等