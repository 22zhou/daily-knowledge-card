# 知识卡片 | 2026-04-21

> 阅读时长：约60分钟

---

## 一、科技热点

### 1.1 今日重磅：宁德时代"超级科技日"发布三大电池技术

今天最受关注的科技新闻是宁德时代在南京举办的2026年"超级科技日"发布会，主题为"极域之约"。这是公司成立以来技术密度最高的一场发布会，集中发布了三大技术产品：

**核心技术突破：**

- **钠离子电池**：降低对锂资源的依赖，成本有望降低30%
- **凝聚态电池**：能量密度突破500Wh/kg，可应用于航空级别场景
- **超快充技术**：充电10分钟、续航800公里，解决充电焦虑

**对行业的影响：**
中国动力电池产业正在从"规模领先"向"技术定义行业标准"跨越，这标志着中国新能源产业进入新阶段。

### 1.2 2026全球6G技术与产业生态大会在南京开幕

大会主题为"极致连接·智能融合·场景共创·产业共赢"，为期3天，来自全球的顶尖专家将共商6G技术创新与产业系统构建。

**大会亮点：**
- 发布6G核心技术成果（太赫兹通信、智能超表面等）
- 推动全球6G统一标准制定
- 明确商用试点计划

### 1.3 AI领域最新动态

**GPT-6完成预训练**
- 上下文窗口：200万Token
- 原生多模态能力
- 相当于可以一次性阅读300万字文本

**国产大模型崛起**
- 阿里千问登顶全球调用榜首
- 百度文心、字节豆包月调用量突破百亿
- 中美顶级模型差距已基本消弭

**人形机器人半马**
北京亦庄人形机器人半程马拉松中，机器人"闪电"以50分26秒夺冠，甚至超越了人类半马世界纪录。

---

## 二、商业风向

### 2.1 程序员副业新机遇（2026版）

**五大高潜力方向：**

| 方向 | 启动难度 | 收益潜力 | 被动收入 |
|------|----------|----------|----------|
| SaaS工具开发 | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | 高 |
| 知识付费/课程 | ⭐⭐ | ⭐⭐⭐ | 中 |
| 技术咨询 | ⭐⭐⭐ | ⭐⭐⭐⭐ | 低 |
| 开源商业化 | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | 中 |
| 垂直AI解决方案 | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | 高 |

**2026年趋势：**
- AI工具开发持续火热
- 低代码/无代码平台插件需求激增
- 垂直领域SaaS工具成为突破口

### 2.2 8000亿特别国债发行

财政部公布2026年超长期特别国债发行安排：
- 总规模：1.3万亿元
- 首批：4月24日发行，1190亿元
- 资金投向：重大战略实施、设备更新、消费升级

---

## 三、程序员英语

### 3.1 常用技术词汇

| 中文 | 英文 | 发音 | 例句 |
|------|------|------|------|
| 涂鸦/手绘 | Doodle | /ˈduːdl/ | Enable doodle mode for freehand drawing. |
| 画布 | Canvas | /ˈkænvəs/ | Draw on a canvas using touch events. |
| 路径 | Path | /pæθ/ | Store the drawing path as a list of points. |
| 笔触 | Stroke | /stroʊk/ | Set stroke width and color. |
| 回退/撤销 | Undo | /ˈʌnduː/ | Implement undo functionality with a stack. |
| 重做 | Redo | /riːˈduː/ | Redo the last undone action. |
| 触摸事件 | Touch Event | /tʌtʃ ɪˈvent/ | Handle touch events for drawing. |
| 贝塞尔曲线 | Bezier Curve | /ˈbezɪeɪ kɜːrv/ | Use Bezier curves for smooth lines. |
| 渲染 | Render | /ˈrendər/ | Render the path to the canvas. |
| 位图 | Bitmap | /ˈbɪtmæp/ | Convert paths to bitmap for export. |

### 3.2 实用短语

```
"It's a piece of cake." - 小菜一碟
"Hit the nail on the head." - 一针见血
"Let's call it a day." - 今天就到这里
"Break the ice." - 打破僵局
"Under the hood." - 底层实现
"Back to the drawing board." - 从头开始
```

---

## 四、每日知识

> **Android图像编辑系列：手绘涂鸦功能从原理到实现**

### 4.1 概述

手绘涂鸦是图像编辑应用的核心功能之一，广泛应用于相册、美颜相机、笔记应用等场景。本文将从原理讲起，详细介绍如何在Android原生开发中实现一个完整的手绘涂鸦功能，包括：

- 自定义View的实现原理
- 触摸事件处理
- 贝塞尔曲线平滑
- 撤销/重做机制
- 画笔样式自定义
- 性能优化

### 4.2 核心原理

#### 4.2.1 Canvas绘图基础

Android的Canvas是一个强大的2D图形绘制工具，我们可以使用它来完成所有涂鸦相关的功能。

**Canvas核心方法：**

```java
// 绘制路径（涂鸦线条）
public void drawPath(Path path, Paint paint)

// 绘制点
public void drawPoint(float x, float y, Paint paint)

// 绘制位图
public void drawBitmap(Bitmap bitmap, Matrix matrix, Paint paint)
```

#### 4.2.2 触摸事件模型

涂鸦功能的本质是捕获用户的触摸事件，将离散的触摸点连成路径：

```java
@Override
public boolean onTouchEvent(MotionEvent event) {
    float x = event.getX();
    float y = event.getY();
    
    switch (event.getAction()) {
        case MotionEvent.ACTION_DOWN:
            // 用户按下手指，开始新路径
            startPath(x, y);
            return true;
            
        case MotionEvent.ACTION_MOVE:
            // 用户移动手指，扩展路径
            continuePath(x, y);
            return true;
            
        case MotionEvent.ACTION_UP:
            // 用户抬起手指，结束路径
            endPath(x, y);
            return true;
    }
    return false;
}
```

### 4.3 完整实现：DrawingBoardView

#### 4.3.1 完整代码

```java
package com.example.imageeditor.view;

import android.content.Context;
import android.graphics.Bitmap;
import android.graphics.Canvas;
import android.graphics.Color;
import android.graphics.Paint;
import android.graphics.Path;
import android.graphics.PorterDuff;
import android.graphics.PorterDuffXfermode;
import android.util.AttributeSet;
import android.view.MotionEvent;
import android.view.View;

import androidx.annotation.Nullable;

import java.util.ArrayList;
import java.util.List;

/**
 * 自定义涂鸦画板视图
 * 支持：自由绘制、多种画笔样式、撤销/重做、橡皮擦、清空
 */
public class DrawingBoardView extends View {
    
    // ==================== 成员变量 ====================
    
    // 画布和位图
    private Canvas mDrawCanvas;
    private Bitmap mCanvasBitmap;
    
    // 画笔
    private Paint mDrawPaint;
    private Paint mBitmapPaint;
    
    // 当前路径
    private Path mCurrentPath;
    
    // 路径历史（用于撤销/重做）
    private List<DrawPath> mPathHistory = new ArrayList<>();
    private List<DrawPath> mRedoHistory = new ArrayList<>();
    
    // 画笔配置
    private int mPaintColor = Color.BLACK;
    private float mPaintStrokeWidth = 8f;
    private PaintMode mPaintMode = PaintMode.DRAW;
    
    // 触摸相关
    private float mLastX, mLastY;
    private boolean mIsDrawing = false;
    
    // ==================== 枚举定义 ====================
    
    public enum PaintMode {
        DRAW,      // 绘画模式
        ERASER     // 橡皮擦模式
    }
    
    // ==================== 内部类 ====================
    
    /**
     * 路径数据结构，保存路径及其画笔样式
     */
    private static class DrawPath {
        Path path;
        int color;
        float strokeWidth;
        PaintMode mode;
        
        DrawPath(Path path, int color, float strokeWidth, PaintMode mode) {
            this.path = path;
            this.color = color;
            this.strokeWidth = strokeWidth;
            this.mode = mode;
        }
    }
    
    // ==================== 构造方法 ====================
    
    public DrawingBoardView(Context context) {
        super(context);
        init();
    }
    
    public DrawingBoardView(Context context, @Nullable AttributeSet attrs) {
        super(context, attrs);
        init();
    }
    
    public DrawingBoardView(Context context, @Nullable AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
        init();
    }
    
    // ==================== 初始化 ====================
    
    private void init() {
        // 初始化绘画画笔
        mDrawPaint = new Paint();
        mDrawPaint.setAntiAlias(true);           // 抗锯齿
        mDrawPaint.setDither(true);              // 防抖动
        mDrawPaint.setColor(mPaintColor);
        mDrawPaint.setStyle(Paint.Style.STROKE); // 只画线，不填充
        mDrawPaint.setStrokeJoin(Paint.Join.ROUND);  // 线条连接处圆滑
        mDrawPaint.setStrokeCap(Paint.Cap.ROUND);    // 线条起点/终点圆滑
        
        // 位图绘制画笔
        mBitmapPaint = new Paint(Paint.DITHER_FLAG);
        
        // 初始化当前路径
        mCurrentPath = new Path();
    }
    
    // ==================== 尺寸测量 ====================
    
    @Override
    protected void onSizeChanged(int w, int h, int oldw, int oldh) {
        super.onSizeChanged(w, h, oldw, oldh);
        
        // 创建位图（使用ARGB_8888保证高质量）
        mCanvasBitmap = Bitmap.createBitmap(w, h, Bitmap.Config.ARGB_8888);
        mDrawCanvas = new Canvas(mCanvasBitmap);
        
        // 填充白色背景
        mDrawCanvas.drawColor(Color.WHITE);
    }
    
    // ==================== 绘制 ====================
    
    @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);
        
        // 绘制位图
        canvas.drawBitmap(mCanvasBitmap, 0, 0, mBitmapPaint);
        
        // 绘制当前正在画的路径
        if (mIsDrawing && mCurrentPath != null) {
            canvas.drawPath(mCurrentPath, mDrawPaint);
        }
    }
    
    // ==================== 触摸事件处理 ====================
    
    @Override
    public boolean onTouchEvent(MotionEvent event) {
        float x = event.getX();
        float y = event.getY();
        
        switch (event.getAction()) {
            case MotionEvent.ACTION_DOWN:
                touchStart(x, y);
                return true;
                
            case MotionEvent.ACTION_MOVE:
                touchMove(x, y);
                return true;
                
            case MotionEvent.ACTION_UP:
                touchUp(x, y);
                return true;
        }
        
        return false;
    }
    
    /**
     * 开始绘制路径
     */
    private void touchStart(float x, float y) {
        // 清空重做历史（新操作后重做列表要清空）
        mRedoHistory.clear();
        
        // 创建新路径
        mCurrentPath = new Path();
        
        // 配置画笔
        updatePaint();
        mCurrentPath.reset();
        mCurrentPath.moveTo(x, y);
        
        mLastX = x;
        mLastY = y;
        mIsDrawing = true;
    }
    
    /**
     * 继续绘制路径（核心：使用贝塞尔曲线平滑）
     */
    private void touchMove(float x, float y) {
        float dx = Math.abs(x - mLastX);
        float dy = Math.abs(y - mLastY);
        
        // 如果移动距离太小，不处理（防抖动）
        if (dx >= TOUCH_TOLERANCE || dy >= TOUCH_TOLERANCE) {
            // 使用二次贝塞尔曲线，使线条更平滑
            // midPoint是(x,y)和(lastX,lastY)的中点
            float midX = (mLastX + x) / 2;
            float midY = (mLastY + y) / 2;
            
            // quadTo: 从当前点到中点画二次贝塞尔曲线
            mCurrentPath.quadTo(mLastX, mLastY, midX, midY);
            
            mLastX = x;
            mLastY = y;
        }
        
        invalidate(); // 触发重绘
    }
    
    /**
     * 结束绘制路径
     */
    private void touchUp(float x, float y) {
        // 画到最后一点
        mCurrentPath.lineTo(x, y);
        
        // 将路径保存到历史
        DrawPath drawPath = new DrawPath(
            mCurrentPath,
            mPaintColor,
            mPaintStrokeWidth,
            mPaintMode
        );
        mPathHistory.add(drawPath);
        
        // 真正绘制到位图上
        mDrawCanvas.drawPath(mCurrentPath, mDrawPaint);
        
        // 重置状态
        mCurrentPath = new Path();
        mIsDrawing = false;
        
        invalidate();
    }
    
    // ==================== 撤销/重做 ====================
    
    /**
     * 撤销
     */
    public void undo() {
        if (mPathHistory.size() > 0) {
            // 移除最后一个路径
            DrawPath lastPath = mPathHistory.remove(mPathHistory.size() - 1);
            mRedoHistory.add(lastPath);
            
            // 重绘
            redrawCanvas();
        }
    }
    
    /**
     * 重做
     */
    public void redo() {
        if (mRedoHistory.size() > 0) {
            DrawPath redoPath = mRedoHistory.remove(mRedoHistory.size() - 1);
            mPathHistory.add(redoPath);
            
            // 重绘
            redrawCanvas();
        }
    }
    
    /**
     * 清空画布
     */
    public void clearCanvas() {
        mPathHistory.clear();
        mRedoHistory.clear();
        
        // 填充白色背景
        mDrawCanvas.drawColor(Color.WHITE);
        invalidate();
    }
    
    /**
     * 重新绘制整个画布
     */
    private void redrawCanvas() {
        // 清空画布
        mDrawCanvas.drawColor(Color.WHITE);
        
        // 重新绘制所有历史路径
        for (DrawPath drawPath : mPathHistory) {
            mDrawPaint.setColor(drawPath.color);
            mDrawPaint.setStrokeWidth(drawPath.strokeWidth);
            
            // 橡皮擦模式需要特殊处理
            if (drawPath.mode == PaintMode.ERASER) {
                mDrawPaint.setXfermode(new PorterDuffXfermode(PorterDuff.Mode.CLEAR));
            } else {
                mDrawPaint.setXfermode(null);
            }
            
            mDrawCanvas.drawPath(drawPath.path, mDrawPaint);
        }
        
        invalidate();
    }
    
    // ==================== 画笔配置 ====================
    
    /**
     * 更新画笔配置
     */
    private void updatePaint() {
        mDrawPaint.setColor(mPaintColor);
        mDrawPaint.setStrokeWidth(mPaintStrokeWidth);
        
        if (mPaintMode == PaintMode.ERASER) {
            // 橡皮擦：使用CLEAR混合模式
            mDrawPaint.setXfermode(new PorterDuffXfermode(PorterDuff.Mode.CLEAR));
        } else {
            mDrawPaint.setXfermode(null);
        }
    }
    
    /**
     * 设置画笔颜色
     */
    public void setPaintColor(int color) {
        this.mPaintColor = color;
        if (mPaintMode != PaintMode.ERASER) {
            mDrawPaint.setColor(color);
        }
    }
    
    /**
     * 设置画笔粗细
     */
    public void setPaintStrokeWidth(float width) {
        this.mPaintStrokeWidth = width;
        mDrawPaint.setStrokeWidth(width);
    }
    
    /**
     * 设置画笔模式
     */
    public void setPaintMode(PaintMode mode) {
        this.mPaintMode = mode;
        updatePaint();
    }
    
    // ==================== 导出功能 ====================
    
    /**
     * 导出当前画布为Bitmap
     */
    public Bitmap exportBitmap() {
        Bitmap bitmap = Bitmap.createBitmap(getWidth(), getHeight(), Bitmap.Config.ARGB_8888);
        Canvas canvas = new Canvas(bitmap);
        canvas.drawColor(Color.WHITE);
        draw(canvas);
        return bitmap;
    }
    
    /**
     * 导出为PNG
     */
    public void exportToPng(String path) {
        Bitmap bitmap = exportBitmap();
        try {
            java.io.FileOutputStream out = new java.io.FileOutputStream(path);
            bitmap.compress(Bitmap.CompressFormat.PNG, 100, out);
            out.flush();
            out.close();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
    
    // ==================== 辅助方法 ====================
    
    /**
     * 检查是否可以撤销
     */
    public boolean canUndo() {
        return mPathHistory.size() > 0;
    }
    
    /**
     * 检查是否可以重做
     */
    public boolean canRedo() {
        return mRedoHistory.size() > 0;
    }
    
    // 触摸容差（防抖动）
    private static final float TOUCH_TOLERANCE = 4f;
}
```

### 4.4 使用示例

#### 4.4.1 在XML布局中使用

```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical">
    
    <!-- 颜色选择栏 -->
    <LinearLayout
        android:id="@+id/colorPicker"
        android:layout_width="match_parent"
        android:layout_height="50dp"
        android:orientation="horizontal"
        android:gravity="center" />
    
    <!-- 画板 -->
    <com.example.imageeditor.view.DrawingBoardView
        android:id="@+id/drawingBoard"
        android:layout_width="match_parent"
        android:layout_height="0dp"
        android:layout_weight="1" />
    
    <!-- 工具栏 -->
    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="56dp"
        android:orientation="horizontal"
        android:gravity="center">
        
        <ImageButton
            android:id="@+id/btnUndo"
            android:layout_width="48dp"
            android:layout_height="48dp"
            android:src="@drawable/ic_undo"
            android:contentDescription="撤销" />
            
        <ImageButton
            android:id="@+id/btnRedo"
            android:layout_width="48dp"
            android:layout_height="48dp"
            android:src="@drawable/ic_redo"
            android:contentDescription="重做" />
            
        <ImageButton
            android:id="@+id/btnEraser"
            android:layout_width="48dp"
            android:layout_height="48dp"
            android:src="@drawable/ic_eraser"
            android:contentDescription="橡皮擦" />
            
        <ImageButton
            android:id="@+id/btnClear"
            android:layout_width="48dp"
            android:layout_height="48dp"
            android:src="@drawable/ic_clear"
            android:contentDescription="清空" />
    </LinearLayout>
    
</LinearLayout>
```

#### 4.4.2 Activity中使用

```java
public class DrawingActivity extends AppCompatActivity {
    
    private DrawingBoardView mDrawingBoard;
    private LinearLayout mColorPicker;
    
    // 预设颜色数组
    private int[] mColors = {
        Color.BLACK,      // 黑色
        Color.WHITE,      // 白色
        Color.RED,        // 红色
        Color.YELLOW,     // 黄色
        Color.GREEN,      // 绿色
        Color.BLUE,       // 蓝色
        Color.parseColor("#FF69B4"),  // 粉色
        Color.parseColor("#9370DB")    // 紫色
    };
    
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_drawing);
        
        initViews();
        initColorPicker();
        initListeners();
    }
    
    private void initViews() {
        mDrawingBoard = findViewById(R.id.drawingBoard);
        mColorPicker = findViewById(R.id.colorPicker);
        
        // 设置初始画笔颜色
        mDrawingBoard.setPaintColor(mColors[0]);
        mDrawingBoard.setPaintStrokeWidth(8f);
    }
    
    private void initColorPicker() {
        // 动态创建颜色按钮
        for (int color : mColors) {
            View colorView = createColorView(color);
            mColorPicker.addView(colorView);
        }
    }
    
    private View createColorView(final int color) {
        View view = new View(this);
        LinearLayout.LayoutParams params = new LinearLayout.LayoutParams(40, 40);
        params.setMargins(8, 8, 8, 8);
        view.setLayoutParams(params);
        view.setBackgroundColor(color);
        
        // 添加圆形边框
        GradientDrawable drawable = new GradientDrawable();
        drawable.setShape(GradientDrawable.OVAL);
        drawable.setColor(color);
        drawable.setStroke(3, Color.GRAY);
        view.setBackground(drawable);
        
        view.setOnClickListener(v -> {
            mDrawingBoard.setPaintMode(DrawingBoardView.PaintMode.DRAW);
            mDrawingBoard.setPaintColor(color);
            
            // 更新选中状态
            updateSelectedColor(view);
        });
        
        return view;
    }
    
    private void initListeners() {
        // 撤销按钮
        findViewById(R.id.btnUndo).setOnClickListener(v -> {
            mDrawingBoard.undo();
        });
        
        // 重做按钮
        findViewById(R.id.btnRedo).setOnClickListener(v -> {
            mDrawingBoard.redo();
        });
        
        // 橡皮擦按钮
        findViewById(R.id.btnEraser).setOnClickListener(v -> {
            mDrawingBoard.setPaintMode(DrawingBoardView.PaintMode.ERASER);
        });
        
        // 清空按钮
        findViewById(R.id.btnClear).setOnClickListener(v -> {
            new AlertDialog.Builder(this)
                .setTitle("确认清空")
                .setMessage("确定要清空画布吗？")
                .setPositiveButton("确定", (dialog, which) -> {
                    mDrawingBoard.clearCanvas();
                })
                .setNegativeButton("取消", null)
                .show();
        });
    }
    
    private void updateSelectedColor(View selectedView) {
        // 重新绘制所有颜色按钮的边框
        for (int i = 0; i < mColorPicker.getChildCount(); i++) {
            View child = mColorPicker.getChildAt(i);
            GradientDrawable drawable = (GradientDrawable) child.getBackground();
            
            if (child == selectedView) {
                drawable.setStroke(6, Color.BLUE); // 选中状态
            } else {
                drawable.setStroke(3, Color.GRAY); // 普通状态
            }
        }
    }
    
    // 保存涂鸦
    private void saveDrawing() {
        String filePath = getExternalFilesDir(null) + "/drawing_" 
            + System.currentTimeMillis() + ".png";
        mDrawingBoard.exportToPng(filePath);
        
        Toast.makeText(this, "已保存到: " + filePath, Toast.LENGTH_LONG).show();
    }
}
```

### 4.5 进阶功能：画笔样式选择器

```java
/**
 * 画笔样式配置类
 */
public class BrushConfig {
    // 画笔类型
    public enum BrushType {
        ROUND,      // 圆形（铅笔）
        SQUARE,     // 方形
        CALLIGRAPHY // 书法笔
    }
    
    private BrushType type;
    private int color;
    private float strokeWidth;
    private float opacity;  // 透明度 0.0 - 1.0
    
    public BrushConfig(BrushType type, int color, float strokeWidth, float opacity) {
        this.type = type;
        this.color = color;
        this.strokeWidth = strokeWidth;
        this.opacity = opacity;
    }
    
    /**
     * 创建Paint对象
     */
    public Paint createPaint() {
        Paint paint = new Paint();
        paint.setAntiAlias(true);
        paint.setDither(true);
        paint.setColor(color);
        paint.setStyle(Paint.Style.STROKE);
        paint.setStrokeJoin(Paint.Join.ROUND);
        paint.setStrokeCap(Paint.Cap.ROUND);
        paint.setStrokeWidth(strokeWidth);
        
        // 设置透明度
        paint.setAlpha((int) (opacity * 255));
        
        // 根据类型设置不同的笔触
        switch (type) {
            case SQUARE:
                paint.setStrokeCap(Paint.Cap.SQUARE);
                break;
            case CALLIGRAPHY:
                paint.setStrokeCap(Paint.Cap.BUTT);
                // 书法笔需要设置路径效果
                paint.setPathEffect(new ComposePathEffect(
                    new CornerPathEffect(strokeWidth / 2),
                    null
                ));
                break;
        }
        
        return paint;
    }
}

/**
 * 书法笔效果
 */
public class CalligraphyBrush {
    private float angle; // 笔倾斜角度
    
    public CalligraphyBrush(float angle) {
        this.angle = angle;
    }
    
    public Path createStrokePath(List<PointF> points) {
        Path path = new Path();
        if (points.size() < 2) return path;
        
        path.moveTo(points.get(0).x, points.get(0).y);
        
        for (int i = 1; i < points.size(); i++) {
            PointF p = points.get(i);
            
            // 计算笔触宽度变化（模拟书法粗细变化）
            float strokeWidth = calculateStrokeWidth(p);
            
            // 根据角度和宽度绘制椭圆作为笔触
            // ...
        }
        
        return path;
    }
}
```

### 4.6 性能优化技巧

```java
/**
 * 性能优化版本：使用缓存减少重绘
 */
public class OptimizedDrawingBoardView extends View {
    
    // 脏区域标记，只重绘变化部分
    private Rect mDirtyRect = new Rect();
    
    @Override
    protected void onDraw(Canvas canvas) {
        if (mDirtyRect.isEmpty()) {
            // 全量绘制
            canvas.drawBitmap(mCanvasBitmap, 0, 0, mBitmapPaint);
            canvas.drawPath(mCurrentPath, mDrawPaint);
        } else {
            // 只绘制脏区域
            canvas.save();
            canvas.clipRect(mDirtyRect);
            canvas.drawBitmap(mCanvasBitmap, 0, 0, mBitmapPaint);
            canvas.drawPath(mCurrentPath, mDrawPaint);
            canvas.restore();
        }
    }
    
    // 在touchMove中更新脏区域
    private void touchMove(float x, float y) {
        // ... 贝塞尔曲线逻辑
        
        // 更新脏区域
        mDirtyRect.set(
            (int) Math.min(mLastX, x) - (int) mPaintStrokeWidth,
            (int) Math.min(mLastY, y) - (int) mPaintStrokeWidth,
            (int) Math.max(mLastX, x) + (int) mPaintStrokeWidth,
            (int) Math.max(mLastY, y) + (int) mPaintStrokeWidth
        );
        
        invalidate(mDirtyRect);
    }
    
    @Override
    public void invalidate(Rect dirty) {
        mDirtyRect.set(dirty);
        super.invalidate(dirty);
    }
}
```

### 4.7 小结

本文详细介绍了Android手绘涂鸦功能的实现：

1. **核心原理**：捕获触摸事件，将离散的点连成贝塞尔曲线
2. **关键类**：自定义View + Canvas + Paint + Path
3. **贝塞尔曲线**：使用quadTo实现平滑的曲线效果
4. **撤销/重做**：维护两个List（历史和重做栈）
5. **橡皮擦**：使用PorterDuffXfermode的CLEAR模式
6. **性能优化**：使用脏区域标记，只重绘变化部分

这些技术同样适用于：
- 签名功能
- 涂鸦贴纸
- 手写笔记
- 图像标注

---

## 五、软考备考

### 5.1 系统架构设计师：软件架构风格（一）

**软件架构风格**是软件工程中的重要概念，描述了组织组件、组件之间的交互以及组件与环境之间关系的模式。

#### 5.1.1 五大架构风格

| 架构风格 | 核心特点 | 典型应用 |
|----------|----------|----------|
| **数据流风格** | 数据在组件间顺序流动 | 批处理、管道-过滤器 |
| **调用返回风格** | 过程调用连接组件 | 主程序-子程序、层次结构 |
| **独立组件风格** | 组件独立，通过消息通信 | 进程通信、事件驱动 |
| **虚拟机风格** | 虚拟环境执行 | 解释器、基于规则系统 |
| **仓库风格** | 中央数据存储 + 组件 | 数据库、黑板系统 |

#### 5.1.2 数据流风格详解

**批处理风格**
- 组件一次性全部处理输入
- 组件之间无交互
- 适合大规模数据处理

**管道-过滤器风格**
- 数据流像水流过管道
- 过滤器负责转换数据
- 可并行处理

```java
// 伪代码示例：管道-过滤器
Input -> Filter1 -> Filter2 -> Filter3 -> Output

// Filter1: 数据转换
class Filter1 {
    InputStream read() {
        return transform(previous.read());
    }
}
```

#### 5.1.3 调用返回风格

**主程序-子程序风格**
```
MainProgram
    ├── SubRoutine1
    │   ├── SubRoutine1_1
    │   └── SubRoutine1_2
    └── SubRoutine2
```

**层次结构风格**
```
┌─────────────────────────────┐
│      表示层(Presentation)  │ <- 用户界面
├─────────────────────────────┤
│      业务层(Business)       │ <- 业务逻辑
├─────────────────────────────┤
│      数据层(Data Access)    │ <- 数据访问
├─────────────────────────────┤
│      基础层(Infrastructure) │ <- 数据库、网络
└─────────────────────────────┘
```

#### 5.1.4 核心考点

1. **识别架构风格**：给定场景描述，识别属于哪种风格
2. **优缺点分析**：每种风格的优势和劣势
3. **适用场景**：哪种场景适合哪种架构风格

**历年真题要点：**
- 管道-过滤器风格强调**解耦**和**可扩展**
- 层次风格强调**关注点分离**
- 事件驱动强调**松耦合**和**异步处理**

---

## 六、面试题

### 6.1 Java并发编程

**题目： synchronized和ReentrantLock有什么区别？**

```java
// synchronized示例
public class Counter {
    private int count = 0;
    
    public synchronized void increment() {
        count++;
    }
    
    public synchronized int getCount() {
        return count;
    }
}

// ReentrantLock示例
public class CounterWithLock {
    private int count = 0;
    private final ReentrantLock lock = new ReentrantLock();
    
    public void increment() {
        lock.lock();
        try {
            count++;
        } finally {
            lock.unlock(); // 必须放在finally中
        }
    }
    
    public int getCount() {
        lock.lock();
        try {
            return count;
        } finally {
            lock.unlock();
        }
    }
}
```

**关键区别：**

| 特性 | synchronized | ReentrantLock |
|------|--------------|---------------|
| **锁获取** | 隐式获取/释放 | 显式获取/释放 |
| **公平锁** | 不支持 | 支持（fair参数） |
| **可中断** | 不可中断 | 可中断（lockInterruptibly） |
| **尝试获取** | 不支持 | 支持（tryLock） |
| **多条件** | 不支持 | 支持（newCondition） |
| **性能** | JVM优化，自动优化 | 需要手动释放 |

```java
// ReentrantLock的高级用法
ReentrantLock lock = new ReentrantLock(true); // true=公平锁

// 1. tryLock - 尝试获取锁，非阻塞
if (lock.tryLock()) {
    try {
        // 操作
    } finally {
        lock.unlock();
    }
} else {
    // 获取失败，执行其他逻辑
}

// 2. tryLock带超时
if (lock.tryLock(5, TimeUnit.SECONDS)) {
    try {
        // 操作
    } finally {
        lock.unlock();
    }
}

// 3. 可中断锁
lock.lockInterruptibly();
try {
    // 操作
} finally {
    lock.unlock();
}

// 4. 多条件Condition
Condition notFull = lock.newCondition();
Condition notEmpty = lock.newCondition();

// 生产者
notFull.await();  // 等待不满
// 生产
notEmpty.signal(); // 通知不空

// 消费者
notEmpty.await(); // 等待不空
// 消费
notFull.signal();  // 通知不满
```

### 6.2 Android面试题

**题目：Android中如何避免内存泄漏？**

```java
/**
 * 常见的内存泄漏场景及解决方案
 */

// 1. 静态引用Context
// 错误写法
public class BadExample {
    private static Context context; // 永远不会被GC
    private static Activity activity; // Activity无法被销毁
    
    public void setActivity(Activity a) {
        this.activity = a; // 内存泄漏！
    }
}

// 正确写法：使用Application Context
public class GoodExample {
    private Context context;
    
    public void setContext(Context context) {
        // 使用Application Context而不是Activity Context
        this.context = context.getApplicationContext();
    }
}

// 2. Handler内存泄漏
// 错误写法
public class BadHandlerActivity extends Activity {
    private Handler handler = new Handler() {
        @Override
        public void handleMessage(Message msg) {
            // 持有Activity引用
        }
    };
    
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        // 发送延迟消息
        handler.sendEmptyMessageDelayed(1, 100000); // 100秒后执行
    }
}

// 正确写法：使用静态内部类 + WeakReference
public class GoodHandlerActivity extends Activity {
    
    // 使用静态内部类，不持有外部类引用
    private static class SafeHandler extends Handler {
        private final WeakReference<Activity> activityRef;
        
        public SafeHandler(Activity activity) {
            this.activityRef = new WeakReference<>(activity);
        }
        
        @Override
        public void handleMessage(Message msg) {
            Activity activity = activityRef.get();
            if (activity != null) {
                // 安全使用activity
            }
        }
    }
    
    private SafeHandler handler;
    
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        handler = new SafeHandler(this);
        
        // 清理：移除所有回调和消息
        handler.removeCallbacksAndMessages(null);
    }
    
    @Override
    protected void onDestroy() {
        super.onDestroy();
        handler.removeCallbacksAndMessages(null);
    }
}

// 3. 监听器未注销
public class NetworkActivity extends Activity {
    private RequestQueue queue;
    private Request request;
    
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        queue = Volley.newRequestQueue(this);
        
        // 添加监听器
        request.addListener(new Listener<String>() {
            @Override
            public void onResponse(String response) {
                // 处理响应
            }
        });
    }
    
    @Override
    protected void onDestroy() {
        super.onDestroy();
        // 必须取消请求，否则Activity无法被GC
        if (request != null) {
            request.cancel();
        }
    }
}

// 4. 资源未关闭
public class ResourceActivity extends Activity {
    private Cursor cursor;
    private SQLiteDatabase db;
    private InputStream inputStream;
    
    @Override
    protected void onDestroy() {
        super.onDestroy();
        
        // 关闭Cursor
        if (cursor != null && !cursor.isClosed()) {
            cursor.close();
        }
        
        // 关闭数据库
        if (db != null && db.isOpen()) {
            db.close();
        }
        
        // 关闭流
        if (inputStream != null) {
            try {
                inputStream.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }
}

// 5. 使用WeakHashMap自动回收
public class CacheManager {
    // 当Entry不再被外部引用时，会被自动回收
    private Map<String, Bitmap> cache = new WeakHashMap<>();
    
    public void put(String key, Bitmap bitmap) {
        cache.put(key, bitmap);
    }
    
    public Bitmap get(String key) {
        return cache.get(key);
    }
}
```

### 6.3 算法题

**题目：合并两个有序链表**

```java
/**
 * 合并两个有序链表
 * LeetCode 21
 * 
 * 时间复杂度: O(m + n)
 * 空间复杂度: O(1)
 */
public class ListNode {
    int val;
    ListNode next;
    
    ListNode(int val) {
        this.val = val;
    }
}

public class MergeTwoLists {
    
    /**
     * 方法一：迭代法
     */
    public ListNode mergeTwoLists(ListNode l1, ListNode l2) {
        // 创建哑节点，简化边界处理
        ListNode dummy = new ListNode(0);
        ListNode current = dummy;
        
        while (l1 != null && l2 != null) {
            if (l1.val <= l2.val) {
                current.next = l1;
                l1 = l1.next;
            } else {
                current.next = l2;
                l2 = l2.next;
            }
            current = current.next;
        }
        
        // 处理剩余部分
        current.next = (l1 != null) ? l1 : l2;
        
        return dummy.next;
    }
    
    /**
     * 方法二：递归法
     */
    public ListNode mergeTwoListsRecursive(ListNode l1, ListNode l2) {
        // 递归终止条件
        if (l1 == null) {
            return l2;
        }
        if (l2 == null) {
            return l1;
        }
        
        // 递归选择较小节点
        if (l1.val <= l2.val) {
            l1.next = mergeTwoListsRecursive(l1.next, l2);
            return l1;
        } else {
            l2.next = mergeTwoListsRecursive(l1, l2.next);
            return l2;
        }
    }
    
    /**
     * 测试代码
     */
    public static void main(String[] args) {
        MergeTwoLists solution = new MergeTwoLists();
        
        // 创建链表 1->2->4
        ListNode l1 = new ListNode(1);
        l1.next = new ListNode(2);
        l1.next.next = new ListNode(4);
        
        // 创建链表 1->3->4
        ListNode l2 = new ListNode(1);
        l2.next = new ListNode(3);
        l2.next.next = new ListNode(4);
        
        // 合并
        ListNode result = solution.mergeTwoLists(l1, l2);
        
        // 打印结果: 1->1->2->3->4->4
        while (result != null) {
            System.out.print(result.val + "->");
            result = result.next;
        }
    }
}
```

---

## 七、金句

> **"The best way to predict the future is to create it."**
> 
> 预测未来的最好方式，就是去创造它。
> 
> —— Peter Drucker

---

## 📚 今日总结

| 板块 | 核心收获 |
|------|----------|
| 科技热点 | 宁德时代三大电池技术突破、6G大会开幕、GPT-6完成预训练 |
| 商业风向 | SaaS工具、知识付费、技术咨询仍是程序员副业热门方向 |
| 程序员英语 | 涂鸦相关词汇：doodle、canvas、path、stroke、undo、redo |
| 每日知识 | Android手绘涂鸦功能完整实现，含贝塞尔曲线、撤销/重做 |
| 软考备考 | 软件架构风格分类，重点掌握数据流和调用返回风格 |
| 面试题 | synchronized vs ReentrantLock、Android内存泄漏、链表合并 |
| 金句 | 预测未来的最好方式是创造它 |

---

*知识卡片 | 2026-04-21 | 阅读时长约60分钟*
