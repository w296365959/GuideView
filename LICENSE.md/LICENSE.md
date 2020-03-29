package com.wzm.widget;

import android.app.Activity;
import android.content.Context;
import android.graphics.Bitmap;
import android.graphics.Canvas;
import android.graphics.Color;
import android.graphics.DashPathEffect;
import android.graphics.Paint;
import android.graphics.PorterDuff;
import android.graphics.PorterDuffXfermode;
import android.graphics.RectF;
import android.os.Build;
import android.util.DisplayMetrics;
import android.view.View;
import android.view.ViewTreeObserver;
import android.widget.FrameLayout;
import android.widget.RelativeLayout;

import com.pingougou.baseutillib.R;
import com.pingougou.baseutillib.tools.common.AppManager;
import com.pingougou.baseutillib.tools.log.PPYLog;
import com.pingougou.baseutillib.tools.system.ScreenUtils;

import java.lang.annotation.Documented;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;

import androidx.annotation.IntDef;
import androidx.annotation.StringDef;

/**
 * (Hangzhou)
 *
 * @author: wzm
 * @date :  2020/3/29 14:46
 * Summary: 引导蒙层
 * RectF location = new RectF();
 * ImageView[] views = new ImageView[drawableRes.length];
 * for (int i = 0; i < drawableRes.length; i++) {
 * views[i] = new ImageView(context);
 * views[i].setImageResource(drawableRes[i]);
 * }
 * GuideView.Builder.newInstance(context)
 * .setTargetView(targetView)//目标控件
 * .setTargetRectF(location) //挖空位置（优先级高于 目标控件）
 * .setCustomGuideViews(views) // 添加的图片
 * .setDirections(directions)//添加的图片摆放的位置方向
 * .setOffset(offsetXY)//添加的图片的在directions基础上的偏移位置
 * .setBgColor(context.getResources().getColor(R.color.black_70))//蒙层背景色
 * .setShape(GuideView.OVAL)//挖空的形状
 * .setStrikingMode(GuideView.StrikingMode.STRIKING_HOLLOW)//镂空
 * .setOnclickListener(new GuideView.OnClickCallback() {
 * @Override public void onClickedGuideView(GuideView guideView) {
 * guideView.hide();//点击事件 隐藏控件
 * }
 * })
 * .build()
 * .show();
 */
public class GuideView  extends RelativeLayout implements ViewTreeObserver.OnGlobalLayoutListener{
    private final String TAG = GuideView.class.getSimpleName();
    private int screenW;
    private int screenH;
    private static final String SHOW_GUIDE_PREFIX = "show_guide";
    private Context mContent;
    private int radius;
    private View targetView;
    private boolean isSupportHollow = true;//是否支持镂空
    //引导图
    private View[] customGuideViews;
    //引导图x,y 偏移距离
    private int[][] customGuideViewOffset;
    private Paint mCirclePaint;

    //是否测量完成
    private boolean isMeasured;
    private int[] center;
    private PorterDuffXfermode porterDuffXfermode;
    private Bitmap bitmap;
    private int backgroundColor;
    private Canvas tempCanvas;
    @Direction
    private int direction;
    @Direction
    private int[] directions;
    @MyShape
    private String myShape;
    //镂空模式
    @StrikingMode
    private int strikingMode;
    //目标控件 左上角在屏幕的位置
    private int[] location;
    private RectF rectF;
    private RectF targetRectFs;
    private OnClickCallback onclickListener;
    private OnShowCallback onShowListener;
    private int targetViewWidth;
    private int targetViewHeight;
    private Paint bgPaint;
    private boolean hasShow = false;
    //图片 位置
    public static final int LEFT = 1;
    public static final int TOP = 2;
    public static final int RIGHT = 3;
    public static final int BOTTOM = 4;
    public static final int LEFT_TOP = 5;
    public static final int LEFT_BOTTOM = 6;
    public static final int RIGHT_TOP = 7;
    public static final int RIGHT_BOTTOM = 8;
    public static final int BOTTOM_CENTER = 9;
    public static final int TOP_CENTER = 10;
    public static final int HORIZONTALLY_CENTER = 11;
    public static final int START_LEFT = 12;
    public static final int START_RIGHT = 13;
    public static final int BOTTOM_START_LEFT = 14;
    public static final int END_RIGHT = 15;
    public static final int END_TOP_RIGHT = 16;
    public static final int END_BOTTOM_RIGHT = 17;
    public static final int CENTER_IN_SCREEN = 18;

    /**
     * 定义GuideView相对于targetView的方位
     */
    @Documented
    @IntDef({LEFT, TOP, RIGHT, BOTTOM,
            LEFT_TOP, LEFT_BOTTOM,
            RIGHT_TOP, RIGHT_BOTTOM,
            BOTTOM_CENTER, TOP_CENTER,
            HORIZONTALLY_CENTER, START_LEFT,
            START_RIGHT, BOTTOM_START_LEFT,
            END_RIGHT, END_TOP_RIGHT,
            END_BOTTOM_RIGHT, CENTER_IN_SCREEN})
    @Retention(RetentionPolicy.SOURCE)
    public @interface  Direction {
    }



    /**
     * 定义目标控件的形状。圆形，矩形,椭圆
     * (椭圆已验证，其他形状 暂未完成)
     */
    @Documented
    @StringDef({MyShape.CIRCULAR, MyShape.RECTANGULAR_ROUND,MyShape.RECTANGULAR, MyShape.OVAL})
    @Retention(RetentionPolicy.SOURCE)
    public @interface MyShape {
        //挖洞 形状
        public static final String CIRCULAR = "circular";
        public static final String RECTANGULAR_ROUND = "rectangular_round";
        public static final String RECTANGULAR = "rectangular";
        public static final String OVAL = "oval";
    }

    @Documented
    @IntDef({StrikingMode.STRIKING_NONE, StrikingMode.STRIKING_HOLLOW, StrikingMode.STRIKING_DASH_LINE})
    @Retention(RetentionPolicy.SOURCE)
    public @interface StrikingMode {
        int STRIKING_NONE = 0;
        int STRIKING_HOLLOW = 1;
        int STRIKING_DASH_LINE = 2;
    }

    public GuideView(Context context) {
        super(context);
        this.mContent = context;
        tempCanvas = new Canvas();
        // 背景画笔
        bgPaint = new Paint();
        // targetView 的透明圆形画笔
        mCirclePaint = new Paint();
        porterDuffXfermode = new PorterDuffXfermode(PorterDuff.Mode.CLEAR);    //SRC_OUT或者CLEAR都可以
        //透明效果
        mCirclePaint.setXfermode(porterDuffXfermode);
        mCirclePaint.setAntiAlias(true);
        rectF = new RectF();
        initView(context);
    }

    private void initView(Context mContext) {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.JELLY_BEAN_MR1) {
            DisplayMetrics dm = new DisplayMetrics();
            Activity appCompatActivity = AppManager.getAppManager().currentActivity();
            if (appCompatActivity != null) {
                appCompatActivity.getWindowManager().getDefaultDisplay().getRealMetrics(dm);
                screenW = dm.widthPixels;
                screenH = dm.heightPixels;
            } else {
                screenW = ScreenUtils.getWidth(mContext);
                screenH = ScreenUtils.getHeight(mContext);
            }
        } else {
            screenW = ScreenUtils.getWidth(mContext);
            screenH = ScreenUtils.getHeight(mContext);
        }
    }

    public int[] getLocation() {
        return location;
    }

    public void setLocation(int[] location) {
        this.location = location;
    }

    public int getRadius() {
        return radius;
    }

    public void setRadius(int radius) {
        this.radius = radius;
    }

    public void setDirection(@Direction int direction) {
        this.direction = direction;
    }

    public void setDirections(@Direction int... directions) {
        this.directions = directions;
    }

    public void setShape(@MyShape String shape) {
        this.myShape = shape;
    }

    public void setBgColor(int background_color) {
        this.backgroundColor = background_color;
    }

    public void setTargetView(View targetView) {
        this.targetView = targetView;
    }

    public int[] getCenter() {
        return center;
    }

    public void setCenter(int[] center) {
        this.center = center;
    }


    public void setCustomGuideViews(View... imgView) {
        customGuideViews = imgView;
    }

    public void setCustomGuideViewOffset(int[]... offsetXY) {
        customGuideViewOffset = offsetXY;
    }

    public void setSupportHollow(boolean supportHollow) {
        isSupportHollow = supportHollow;
    }

    public void setStrikingMode(@StrikingMode int strikingMode){
        this.strikingMode = strikingMode;
    }

    private String generateUniqId(View v) {
        return SHOW_GUIDE_PREFIX + v.getId();
    }

    public void setOnclickListener(OnClickCallback onclickListener) {
        this.onclickListener = onclickListener;
    }

    public void setOnSomethingAfterShow(OnShowCallback onShowListener) {
        this.onShowListener = onShowListener;
    }

    /**
     * 点击退出
     */
    private void setClickInfo() {
        setOnClickListener(new OnClickListener() {
            @Override
            public void onClick(View v) {
                if (onclickListener != null) {
                    onclickListener.onClickedGuideView((GuideView) v);
                }

            }
        });
    }

    public boolean isHasShow() {
        return hasShow;
    }

    public void show() {
        if (hasShow) {
            return;
        }
        hasShow = true;
        if (targetView != null) {
            targetView.getViewTreeObserver().addOnGlobalLayoutListener(this);
        }
        this.setBackgroundResource(R.color.transparent);
        this.bringToFront();  //设置在最上层
        if (this.getParent() == null) {
            ((FrameLayout) ((Activity) mContent).getWindow().getDecorView()).addView(this);
        }
        if (onShowListener != null) {
            onShowListener.doSomethingAfterShow(this);
        }
    }

    public void hide() {
        PPYLog.v(TAG, "hide");
        if (customGuideViews != null) {
            targetView.getViewTreeObserver().removeOnGlobalLayoutListener(this);
            this.removeAllViews();
            ((FrameLayout) ((Activity) mContent).getWindow().getDecorView()).removeView(this);
        }
        hasShow = false;
    }

    /**
     * 获得targetView 或targetRectFs 的宽高
     *
     * @return
     */
    private int[] getTargetViewSize() {
        int[] location = {-1, -1};
        if (isMeasured) {
            if (targetRectFs != null) {
                location[0] = (int) (targetRectFs.right - targetRectFs.left);
                location[1] = (int) (targetRectFs.bottom - targetRectFs.top);
            } else {
                location[0] = targetView.getWidth();
                location[1] = targetView.getHeight();
            }
        }
        return location;
    }

    /**
     * 获得targetView 或targetRectFs的半径
     *
     * @return
     */
    private int getTargetViewRadius() {
        if (isMeasured) {
            int[] size = getTargetViewSize();
            int x = size[0];
            int y = size[1];

            return (int) (Math.sqrt(x * x + y * y) / 2);
        }
        return -1;
    }

    @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);
        PPYLog.v(TAG, "onDraw");
        if (!isMeasured)
            return;
        if (targetView == null)
            return;
        drawBackground(canvas);
    }


    private void drawBackground(Canvas canvas) {
        PPYLog.v(TAG, "drawBackground");
        // 先绘制bitmap，再将bitmap绘制到屏幕
        bitmap = Bitmap.createBitmap(canvas.getWidth(), canvas.getHeight(), Bitmap.Config.ARGB_8888);
        tempCanvas.setBitmap(bitmap);
        if (backgroundColor != 0) {
            bgPaint.setColor(backgroundColor);
        } else {
            bgPaint.setColor(getResources().getColor(R.color.trans_black_d));
        }
        // 绘制屏幕背景
        tempCanvas.drawRect(0, 0, tempCanvas.getWidth(), tempCanvas.getHeight(), bgPaint);

        boolean supportTargetStriking;
        switch (strikingMode) {
            case StrikingMode.STRIKING_DASH_LINE:
                //透明效果
                mCirclePaint.setStyle(Paint.Style.STROKE);
                mCirclePaint.setStrokeWidth(3);
                mCirclePaint.setPathEffect(new DashPathEffect(new float[] {10, 10}, 0));
                mCirclePaint.setXfermode(null);
                mCirclePaint.setAntiAlias(true);
                mCirclePaint.setColor(Color.WHITE);
                supportTargetStriking = true;
                break;
            case StrikingMode.STRIKING_HOLLOW:
                //透明效果
                mCirclePaint.setXfermode(porterDuffXfermode);
                mCirclePaint.setAntiAlias(true);
                supportTargetStriking = true;
                break;
            case StrikingMode.STRIKING_NONE:
            default:
                supportTargetStriking = false;
                break;
        }
        if (supportTargetStriking) {
            if (myShape != null) {
                switch (myShape) {
                    case MyShape.CIRCULAR://圆形
                        tempCanvas.drawCircle(center[0], center[1], radius, mCirclePaint);
                        break;
                    case MyShape.RECTANGULAR_ROUND://圆角矩形
                        if (targetRectFs != null) {
                            tempCanvas.drawRoundRect(targetRectFs, radius, radius, mCirclePaint);
                        } else {
                            rectF.left = location[0] + 5;
                            rectF.top = center[1] - targetViewHeight / 2 + 1;
                            rectF.right = location[0] + targetViewWidth - 5;
                            rectF.bottom = center[1] + targetViewHeight / 2 - 1;
                            tempCanvas.drawRoundRect(rectF, radius, radius, mCirclePaint);
                        }
                        break;
                    case MyShape.RECTANGULAR://矩形
                        if (targetRectFs != null) {
                            tempCanvas.drawRoundRect(targetRectFs, 0, 0, mCirclePaint);
                        } else {
                            rectF.left = location[0] + 5;
                            rectF.top = center[1] - targetViewHeight / 2 + 1;
                            rectF.right = location[0] + targetViewWidth - 5;
                            rectF.bottom = center[1] + targetViewHeight / 2 - 1;
                            tempCanvas.drawRoundRect(rectF, radius, 0, mCirclePaint);
                        }
                        break;
                    case MyShape.OVAL://椭圆
                        if (targetRectFs == null) {
                            rectF.left = location[0];
                            rectF.top = location[1];
                            rectF.right = location[0] + targetViewWidth;
                            rectF.bottom = location[1] + targetViewHeight;
                            tempCanvas.drawOval(rectF, mCirclePaint);
                        } else {
                            tempCanvas.drawOval(targetRectFs, mCirclePaint);
                        }
                        break;
                    default:
                        break;
                }
            } else {
                tempCanvas.drawCircle(center[0], center[1], radius, mCirclePaint);
            }
        }

        // 绘制到屏幕
        canvas.drawBitmap(bitmap, 0, 0, bgPaint);
        bitmap.recycle();
    }


    @Override
    public void onGlobalLayout() {
        if (isMeasured)
            return;
        if (targetView.getHeight() > 0 && targetView.getWidth() > 0) {
            isMeasured = true;
            targetViewWidth = targetView.getWidth();
            targetViewHeight = targetView.getHeight();
        }

        // 获取targetView的中心坐标
        if (center == null) {
            // 获取左上角坐标
            location = new int[2];
            targetView.getLocationInWindow(location);
            center = new int[2];
            // 获取中心坐标
            center[0] = location[0] + targetView.getWidth() / 2;
            center[1] = location[1] + targetView.getHeight() / 2;
        }
        // 获取targetView或targetRectFs外切圆半径
        if (radius == 0) {
            radius = getTargetViewRadius();
        }

        //文字图片和提示图片
        createView();

    }

    //文字图片和我知道啦图片一起放
    private void createView() {
        PPYLog.v(TAG, "createView");
        if (customGuideViews != null && customGuideViewOffset != null && directions != null) {
            if (customGuideViews.length != customGuideViewOffset.length) {
                throw new RuntimeException("customGuideViews.length is not equals customGuideViewOffset.length!");
            }
            if (customGuideViews.length != directions.length) {
                throw new RuntimeException("customGuideViews.length is not equals directions.length!");
            }
            int left = center[0] - targetViewWidth / 2;
            int right = center[0] + targetViewWidth / 2;
            int top = center[1] - targetViewHeight / 2;
            int bottom = center[1] + targetViewHeight / 2;
            if (targetRectFs != null) {
                left = (int) targetRectFs.left;
                right = (int) targetRectFs.right;
                top = (int) targetRectFs.top;
                bottom = (int) targetRectFs.bottom;
            }
            for (int i = 0; i < customGuideViews.length; i++) {
                if (customGuideViews[i] == null || customGuideViews[i].getParent() != null) {
                    PPYLog.w(TAG, "guideView img==null Or  The specified child already has a parent!");
                    continue;
                }
                LayoutParams viewParams = new LayoutParams(LayoutParams.WRAP_CONTENT, LayoutParams.WRAP_CONTENT);
                customGuideViews[i].setLayoutParams(viewParams);
                //先测量，再求宽高
                customGuideViews[i].measure(0, 0);
                int customWidth = customGuideViews[i].getMeasuredWidth();
                int customHeight = customGuideViews[i].getMeasuredHeight();
                int offX = customGuideViewOffset[i][0];
                int offY = customGuideViewOffset[i][1];
                switch (directions[i]) {
                    case BOTTOM_START_LEFT:
                        //在targetView底部，屏幕左对齐
                        viewParams.setMargins(offX, bottom + offY, 0, 0);
                        break;
                    case START_LEFT:
                        //与targetView同一水平线，屏幕左对齐
                        viewParams.setMargins(offX, bottom - customHeight + offY, 0, 0);
                        break;
                    case START_RIGHT:
                        //与targetView同一水平线，与屏幕右对齐
                        viewParams.setMargins(screenW - customWidth + offX, bottom - customHeight + offY, 0, 0);
                        break;
                    case END_RIGHT:
                        //与targetView同一水平线，与控件右对齐
                        viewParams.setMargins(right - customWidth + offX, bottom - customHeight + offY, 0, 0);
                        break;
                    case END_TOP_RIGHT:
                        //在targetView顶部，与targetView右对齐，
                        viewParams.setMargins(right - customWidth + offX, top - customHeight + offY, 0, 0);
                        break;
                    case END_BOTTOM_RIGHT:
                        //在targetView底部，与targetView右对齐，
                        viewParams.setMargins(right - customWidth + offX, bottom + offY, 0, 0);
                        break;
                    case TOP:
                        // 在targetView顶部（与targetView左对齐）
                        viewParams.setMargins(left + offX, top - customHeight + offY, 0, 0);
                        break;
                    case BOTTOM:
                        // 在targetView底部（与targetView左对齐）
                        viewParams.setMargins(left + offX, bottom + offY, 0, 0);
                        break;
                    case LEFT:
                        // 与targetView同一水平线,在targetView左边
                        viewParams.setMargins(left - customWidth + offX, bottom - customHeight + offY, 0, 0);
                        break;
                    case RIGHT:
                        // 与targetView同一水平线,在targetView右边
                        viewParams.setMargins(right + offX, bottom - customHeight + offY, 0, 0);
                        break;
                    case LEFT_TOP:
                        //在targetView左上角
                        viewParams.setMargins(left - customWidth + offX, top - customHeight + offY, 0, 0);
                        break;
                    case RIGHT_TOP:
                        //在targetView右上角
                        viewParams.setMargins(right + offX, top - customHeight + offY, 0, 0);
                        break;
                    case LEFT_BOTTOM:
                        //在targetView左下角
                        viewParams.setMargins(left - customWidth + offX, bottom + offY, 0, 0);
                        break;
                    case RIGHT_BOTTOM:
                        //在targetView右下角
                        viewParams.setMargins(right + offX, bottom + offY, 0, 0);
                        break;
                    case TOP_CENTER:
                        //在targetView顶部 同时相对屏幕宽居中
                        viewParams.setMargins(screenW / 2 - customWidth / 2 + offX, top - customHeight + offY, 0, 0);
                        break;
                    case HORIZONTALLY_CENTER:
                        //与targetView底部 在同一水平线上 居中
                        viewParams.setMargins(screenW / 2 - customWidth / 2 + offX, bottom - customHeight + offY, 0, 0);
                        break;
                    case BOTTOM_CENTER:
                        //在targetView下方，同时在屏幕水平方向居中
                        viewParams.setMargins(screenW / 2 - customWidth / 2 + offX, bottom + offY, 0, 0);
                        break;
                    case CENTER_IN_SCREEN:
                        //在屏幕中间
                        viewParams.setMargins(screenW / 2 - customWidth / 2 + offX, screenH / 2 - customHeight / 2 + offY, 0, 0);
                        break;
                    default:
                        viewParams.setMargins(screenW / 2 - customWidth / 2, bottom + offY, 0, 0);
                        break;
                }
                this.addView(customGuideViews[i], viewParams);
            }
        } else {
            PPYLog.w(TAG, "not set tips view!");
        }
    }


    public interface OnShowCallback {
        void doSomethingAfterShow(GuideView guideView);
    }

    /**
     * GuideView点击Callback
     */
    public interface OnClickCallback {
        void onClickedGuideView(GuideView guideView);
    }

    public static class Builder {
        static GuideView guiderView;
        static Builder instance = new Builder();
        Context mContext;

        private Builder() {
        }

        public Builder(Context ctx) {
            mContext = ctx;
        }

        public static Builder newInstance(Context ctx) {
            guiderView = new GuideView(ctx);
            return instance;
        }

        /**
         * 设置目标view
         */
        public Builder setTargetView(View target) {
            guiderView.setTargetView(target);
            return instance;
        }

        /**
         * 挖空区域（优先级高于targetView）
         *
         * @param rectFs
         * @return
         */
        public Builder setTargetRectF(RectF rectFs) {
            guiderView.targetRectFs = rectFs;
            return instance;
        }

        /**
         * 设置蒙层背景颜色
         * getResources().getColor(R.color.black_70)
         */
        public Builder setBgColor(int color) {
            guiderView.setBgColor(color);
            return instance;
        }

        /**
         * 设置文字和图片View 在目标view的位置
         */
        public Builder setDirction(@Direction int dir) {
            return setDirections(dir);
        }

        public Builder setDirections(@Direction int... dir) {
            guiderView.setDirections(dir);
            return instance;
        }

        /**
         * 设置绘制形状
         */
        public Builder setShape(@MyShape String shape) {
            guiderView.setShape(shape);
            return instance;
        }

        /**
         * 设置圆半径
         *
         * @param radius
         * @return
         */
        public Builder setRadius(int radius) {
            guiderView.setRadius(radius);
            return instance;
        }


        public Builder setCustomGuideViews(View... imgView) {
            guiderView.setCustomGuideViews(imgView);
            return instance;
        }

        public Builder setOffset(int[]... offsetXY) {
            guiderView.setCustomGuideViewOffset(offsetXY);
            return instance;
        }

        public Builder supportHollow(boolean supportHollow) {
            guiderView.setSupportHollow(supportHollow);
            return instance;
        }

        public Builder setStrikingMode(@StrikingMode int strikingMode) {
            guiderView.setStrikingMode(strikingMode);
            return instance;
        }

        /**
         * 点击监听
         */
        public Builder setOnclickListener(final OnClickCallback callback) {
            guiderView.setOnclickListener(callback);
            return instance;
        }

        public Builder setTags(final Object tag) {
            guiderView.setTag(tag);
            return instance;
        }

        public GuideView build() {
            guiderView.setClickInfo();
            return guiderView;
        }

    }
}
