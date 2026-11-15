[Windows-批量替换车牌.md](https://github.com/user-attachments/files/34871371/Windows-.md)
Batch-replace-numbers

这个skill可以一键的替换车牌号等一连串图片里的数据，你只需要准备好一个记事本里面存入数据，脚本运行后它会让你选要替换数据范围的四个角，选定之后它会一键替换，仅需几秒。
This tool allows you to replace data—such as license plate numbers—across a series of images with a single click. You simply need to prepare a text file containing the data; once the script is running, it will prompt you to select the four corners defining the area where the data should be replaced. After you’ve made your selection, the replacement is performed instantly—taking only a few seconds.

一、这套方法能做到什么

- 同一张照片，批量替换不同车牌号码。
- 只修改你框选的车牌区域，输出 PNG，区域外像素保持不变。
- 自动按 `001_沪FS9308.png` 这样的名称保存，不会覆盖原图。
- 第一次点选四个角，后续可以复用位置。
- 图片和号码都在本机处理，不需要上传。

效果说明：这是一版适合当前黄底黑字车牌的基础工具。它会自动清除旧文字、尽量保留底色和光影，但复杂污渍、螺丝附近可能留下痕迹。默认使用 Windows 黑体，并非原车牌专用字体；先检查两张样图，满意后再批量导出。

如果你要求图片其他部分与最初照片一致，请使用 最初未经过 AI 修改的原图 作为底图。

 二、安装一次运行环境

1. 从 [Python 官网](https://www.python.org/downloads/windows/) 安装 Windows 版 Python，推荐 3.12 或 3.13。
2. 安装时勾选 Add python.exe to PATH。
3. 按 `Win + R`，输入 `cmd`，回车。
4. 执行：

```bat
py -m pip install pillow opencv-python numpy
```

如果提示找不到 `py`，可以把后面的命令中的 `py` 改为 `python` 再试。

 三、准备号码文件

桌面新建一个文件夹，例如 `批量换车牌`，放入原图。

再用记事本建立 `车牌号码.txt`，**每行一个号码，不要表头**：

```text
沪FS9308
沪FS9309
沪FS9310
```

如果号码在 Excel 中，直接复制整列，粘贴到记事本即可。保存编码选择 UTF-8。

这版脚本接受与当前车牌相同的格式：1个汉字＋1个字母＋5个字母或数字。

 四、保存脚本

打开记事本，把下面代码全部复制进去。

保存到同一文件夹，文件名为 `batch_plate.py`；“保存类型”选择 所有文件，避免保存成 `.py.txt`。

```python
import json
import os
import re
from pathlib import Path
import tkinter as tk
from tkinter import filedialog, messagebox

import cv2
import numpy as np
from PIL import Image, ImageDraw, ImageFont, ImageOps, ImageTk

def choose_corners(root, image):
    """依次点击车牌左上、右上、右下、左下。"""
    root.deiconify()
    root.title("依次点击：左上 → 右上 → 右下 → 左下；右键重选")

    scale = min(1000 / image.width, 720 / image.height, 1)
    size = (round(image.width * scale), round(image.height * scale))
    preview = ImageTk.PhotoImage(
        image.resize(size, Image.Resampling.LANCZOS)
    )

    canvas = tk.Canvas(
        root, width=size[0], height=size[1], highlightthickness=0
    )
    canvas.pack()
    canvas.create_image(0, 0, anchor="nw", image=preview)

    points = []
    ready = tk.BooleanVar(root, False)

    def reset(event=None):
        points.clear()
        canvas.delete("mark")

    def click(event):
        if len(points) >= 4:
            return
        points.append([event.x / scale, event.y / scale])
        canvas.create_oval(
            event.x - 4, event.y - 4,
            event.x + 4, event.y + 4,
            fill="red", outline="white", tags="mark"
        )
        canvas.create_text(
            event.x + 12, event.y - 10,
            text=str(len(points)), fill="red", tags="mark"
        )
        if len(points) == 4:
            ready.set(True)

    canvas.bind("<Button-1>", click)
    canvas.bind("<Button-3>", reset)
    root.wait_variable(ready)
    canvas.destroy()
    root.withdraw()
    return np.asarray(points, dtype=np.float32)

def main(root):
    image_name = filedialog.askopenfilename(
        parent=root,
        title="选择原始卡车照片",
        filetypes=[
            ("图片", "*.jpg *.jpeg *.png *.bmp"),
            ("所有文件", "*.*")
        ]
    )
    if not image_name:
        return

    number_name = filedialog.askopenfilename(
        parent=root,
        title="选择车牌号码 TXT 文件",
        filetypes=[("文本文件", "*.txt")]
    )
    if not number_name:
        return

    raw = Path(number_name).read_bytes()
    try:
        text = raw.decode("utf-8-sig")
    except UnicodeDecodeError:
        text = raw.decode("gb18030")

    numbers = [
        re.sub(r"[\s·.]", "", line).upper()
        for line in text.splitlines()
        if line.strip()
    ]
    if not numbers:
        raise ValueError("号码文件为空。")

    pattern = r"[\u4e00-\u9fff][A-Z][A-Z0-9]{5}"
    invalid = [n for n in numbers if not re.fullmatch(pattern, n)]
    if invalid:
        raise ValueError(
            "以下号码格式不匹配，请检查是否带有表头或序号：\n"
            + "\n".join(invalid[:10])
        )

    fonts_dir = Path(os.environ.get("WINDIR", r"C:\Windows")) / "Fonts"
    font_path = next(
        (
            fonts_dir / name
            for name in ("simhei.ttf", "msyh.ttc", "simsun.ttc")
            if (fonts_dir / name).exists()
        ),
        None
    )
    if font_path is None:
        raise ValueError("未找到中文字体，请安装黑体或微软雅黑。")
    font = ImageFont.truetype(str(font_path), 110)

    image_path = Path(image_name)
    with Image.open(image_path) as source:
        icc = source.info.get("icc_profile")
        image = ImageOps.exif_transpose(source).convert("RGB")

    base = np.asarray(image).copy()
    height, width = base.shape[:2]
    config = image_path.with_suffix(image_path.suffix + ".plate.json")

    reuse = config.exists() and messagebox.askyesno(
        "复用位置",
        "这张照片已有车牌位置记录，是否直接复用？",
        parent=root
    )
    if reuse:
        points = np.asarray(
            json.loads(config.read_text(encoding="utf-8")),
            dtype=np.float32
        )
    else:
        points = choose_corners(root, image)

    if (
        points.shape != (4, 2)
        or not cv2.isContourConvex(points)
        or abs(cv2.contourArea(points)) < 20
    ):
        raise ValueError(
            "车牌四角选择不正确，请重新运行，"
            "按左上、右上、右下、左下的顺序点击。"
        )

    config.write_text(
        json.dumps(points.tolist()), encoding="utf-8"
    )

    # 把倾斜车牌展开为正面模板。
    plate_w, plate_h = 440, 140
    flat_points = np.float32([
        [0, 0],
        [plate_w - 1, 0],
        [plate_w - 1, plate_h - 1],
        [0, plate_h - 1]
    ])
    to_flat = cv2.getPerspectiveTransform(points, flat_points)
    to_photo = cv2.getPerspectiveTransform(flat_points, points)

    flat = cv2.warpPerspective(
        base, to_flat, (plate_w, plate_h),
        borderMode=cv2.BORDER_REPLICATE
    )

    # 遮罩只检测牌内较暗的旧文字，尽量保留黄色底纹。
    gray = cv2.cvtColor(flat, cv2.COLOR_RGB2GRAY)
    erase = (gray < 150).astype(np.uint8) * 255
    erase = cv2.dilate(erase, np.ones((3, 3), np.uint8))
    erase[:12, :] = 0
    erase[-12:, :] = 0
    erase[:, :10] = 0
    erase[:, -10:] = 0
    clean = cv2.inpaint(flat, erase, 5, cv2.INPAINT_TELEA)

    # 最终只允许修改四角围成的区域。
    region = np.zeros((height, width), dtype=np.uint8)
    cv2.fillConvexPoly(
        region, np.rint(points).astype(np.int32), 255
    )
    inside = region > 0

    output_dir = image_path.parent / (image_path.stem + "_换牌结果")
    output_dir.mkdir(exist_ok=True)
    save_options = {"icc_profile": icc} if icc else {}

    for index, number in enumerate(numbers, start=1):
        plate = Image.fromarray(clean.copy())

        # 逐字排版，字符等间距，字形压缩为近似车牌比例。
        margin = 18
        cell_w = (plate_w - 2 * margin) / len(number)
        for position, char in enumerate(number):
            left, top, right, bottom = font.getbbox(char)
            glyph = Image.new("L", (right - left, bottom - top), 0)
            ImageDraw.Draw(glyph).text(
                (-left, -top), char, font=font, fill=255
            )
            glyph = glyph.resize(
                (46, 100), Image.Resampling.LANCZOS
            )
            x = round(margin + position * cell_w + (cell_w - 46) / 2)
            plate.paste((22, 22, 18), (x, 22), glyph)

        # 根据原车牌四角恢复透视。
        warped = cv2.warpPerspective(
            np.asarray(plate), to_photo, (width, height),
            flags=cv2.INTER_LINEAR,
            borderMode=cv2.BORDER_REPLICATE
        )

        result = base.copy()
        result[inside] = warped[inside]

        destination = output_dir / f"{index:03d}_{number}.png"
        Image.fromarray(result).save(destination, **save_options)
        print(f"[{index}/{len(numbers)}] 已保存：{destination.name}")

    messagebox.showinfo(
        "完成",
        f"已生成 {len(numbers)} 张图片。\n\n保存位置：\n{output_dir}",
        parent=root
    )

if __name__ == "__main__":
    root = tk.Tk()
    root.withdraw()
    try:
        main(root)
    except Exception as error:
        messagebox.showerror("处理失败", str(error), parent=root)
        print("处理失败：", error)
    finally:
        try:
            root.destroy()
        except tk.TclError:
            pass
```

 五、开始批量生成

1. 打开存放脚本的文件夹。
2. 点击资源管理器顶部的**地址栏**，输入 `cmd`，回车。
3. 执行：

```bat
py batch_plate.py
```

4. 在弹出的窗口中选择原图，再选择 `车牌号码.txt`。
5. 在预览图里，依次点击**车牌外边缘的左上、右上、右下、左下**四个角，尽量不要包含周围车身。
6. 点完第四个角后自动开始，命令行会显示处理进度。

图片保存在原图旁边的 `原图文件名_换牌结果` 文件夹中。
