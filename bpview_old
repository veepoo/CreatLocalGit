
package com.veepoo.hband.activity.connected;

import android.app.AlertDialog;
import android.content.BroadcastReceiver;
import android.content.Context;
import android.content.DialogInterface;
import android.content.Intent;
import android.content.IntentFilter;
import android.graphics.Bitmap;
import android.graphics.Canvas;
import android.graphics.Color;
import android.os.Handler;
import android.os.Message;
import android.support.v4.content.LocalBroadcastManager;
import android.text.TextUtils;
import android.view.LayoutInflater;
import android.view.View;
import android.widget.ImageView;
import android.widget.LinearLayout;
import android.widget.TextView;
import android.widget.Toast;
import android.widget.ToggleButton;

import com.github.mikephil.charting.charts.LineChart;
import com.github.mikephil.charting.data.Entry;
import com.github.mikephil.charting.data.LineData;
import com.github.mikephil.charting.data.LineDataSet;
import com.orhanobut.logger.Logger;
import com.veepoo.hband.R;
import com.veepoo.hband.activity.BaseActivity;
import com.veepoo.hband.ble.BleBroadCast;
import com.veepoo.hband.ble.BleIntentPut;
import com.veepoo.hband.ble.BleProfile;
import com.veepoo.hband.ble.readmanager.BPHandler;
import com.veepoo.hband.config.SputilVari;
import com.veepoo.hband.modle.BPBean;
import com.veepoo.hband.modle.TimeBean;
import com.veepoo.hband.sql.SqlHelperUtil;
import com.veepoo.hband.util.BaseUtil;
import com.veepoo.hband.util.ConvertHelper;
import com.veepoo.hband.util.ImageUtils;
import com.veepoo.hband.util.SpUtil;
import com.veepoo.hband.view.HomeCircleView;

import java.io.IOException;
import java.util.ArrayList;
import java.util.concurrent.Executors;
import java.util.concurrent.ScheduledExecutorService;
import java.util.concurrent.TimeUnit;

import butterknife.BindColor;
import butterknife.BindString;
import butterknife.BindView;
import butterknife.OnClick;
import cn.sharesdk.framework.ShareSDK;
import cn.sharesdk.onekeyshare.OnekeyShare;

/**
 * Created by timaimee on 2016/8/17.
 */
public class BPDetectActivity extends BaseActivity {
    private final static String TAG = BPDetectActivity.class.getSimpleName();
    private Context mContext = BPDetectActivity.this;
    private final static String TAG_UMENT = "血压测试界面";
    private final static int MAX_VALUE_ADC = 180;
    private final static int MIN_VALUE_ADC = 30;
    private int highAdcValue = 0;
    private int lowAdcValue = 0;
    private long mPressedTime = 0;

    @BindString(R.string.head_title_bp_detect)
    String mStrHeadTitle;


    @BindString(R.string.dpdetect_clicktostart)
    String mStrClickToStart;
    @BindString(R.string.dpdetect_test_invalit)
    String mStrTestInvalit;
    @BindView(R.id.bpdetect_model_normal)
    TextView mModleNormal;
    @BindView(R.id.bpdetect_model_private)
    TextView mModlePrivate;
    @BindView(R.id.detect_start)
    ImageView mDetectButImage;
    @BindView(R.id.bpdetect_undetectview)
    ImageView mUnDetectView;
    @BindView(R.id.bpdetect_circleview)
    HomeCircleView mCircleView;
    @BindView(R.id.middle_progress_tv)
    TextView mMiddleProgressTv;
    @BindView(R.id.bp_adc_chart)
    LineChart mBpChart;
    @BindView(R.id.bpdetect_states)
    TextView mBpState;
    @BindView(R.id.detect_notify)
    ToggleButton notify;

    @BindColor(R.color.white)
    int white;
    @BindColor(R.color.app_color_helper_four)
    int backgroundColor;

    private int modelNormal = 0, modelPrivate = 1;
    private int modelType = 0;

    private boolean isDetecting = false;
    BPHandler mBPHander;

    int progrss = 0;
    ScheduledExecutorService scheduledThreadPool;
    //如果开始跑进度，就不要再启动线程了
    boolean isStartProgress = false;

    ArrayList<String> xVals = new ArrayList<>();
    ArrayList<Entry> values = new ArrayList<>();


    Handler mHandler = new Handler() {
        @Override
        public void handleMessage(Message msg) {
            super.handleMessage(msg);
            if (isDetecting) {
                progrss = progrss + 2;//55秒
                if (progrss <= 100) {
                    mCircleView.setInnerProgress((float) progrss / 100);
                    mMiddleProgressTv.setText(progrss + "%");
                } else {
                    Toast.makeText(mContext, mStrTestInvalit, Toast.LENGTH_SHORT).show();
                    stopDetectView();
                }
            }
        }
    };

    @Override
    public View initVew() {

        View view = LayoutInflater.from(mContext).inflate(R.layout.activity_dpdetect, null);
        return view;

    }

    @Override
    public void initData() {
        mBPHander = new BPHandler(mContext);
        registerLocalBroadCaster();
        initHeadView(mStrHeadTitle);
        baseImgRight.setVisibility(View.VISIBLE);
        baseImgRight.setImageResource(R.drawable.app_share);
        setChartProperty();
        setData();
        mHeadLayout.setBackgroundColor(backgroundColor);
        changeModelView();
        mCircleView.setCircleProgress(1f);
        mCircleView.setDrawOutCircle(false);
        mCircleView.setInnerProgress(0.0f);
        mBpState.setText(mStrClickToStart);

    }


    @OnClick({R.id.bpdetect_model_normal, R.id.bpdetect_model_private})
    public void onClickEvent(View view) {
        int id = view.getId();
        switch (id) {
            case R.id.bpdetect_model_normal:
                modelType = modelNormal;
                changeModelView();
                break;
            case R.id.bpdetect_model_private:
                modelType = modelPrivate;
                String highPressValues = SpUtil.getString(mContext, SputilVari.BP_SETTING_HIGHT, "0");
                String lowPressValues = SpUtil.getString(mContext, SputilVari.BP_SETTING_LOW, "0");
                if (TextUtils.equals(highPressValues, "0") || TextUtils.equals(lowPressValues, "0")) {
                    isSettingPrivateModel();
                }
                changeModelView();
                break;
        }
    }


    @Override
    protected void onResume() {
        super.onResume();
        if (modelType == modelPrivate) {
            String highPressValues = SpUtil.getString(mContext, SputilVari.BP_SETTING_HIGHT, "0");
            String lowPressValues = SpUtil.getString(mContext, SputilVari.BP_SETTING_LOW, "0");
            if (TextUtils.equals(highPressValues, "0") || TextUtils.equals(lowPressValues, "0")) {
                isSettingPrivateModel();
            }
        }
        onResume(this, TAG_UMENT);
    }

    @BindString(R.string.dpdetect_dialog_setting_title)
    String mStrSettingTitle;
    @BindString(R.string.dpdetect_dialog_setting_content)
    String mStrSettingContent;
    @BindString(R.string.dpdetect_dialog_setting_ok)
    String mStrSettingOk;
    @BindString(R.string.dpdetect_dialog_setting_no)
    String mStrSettingNo;

    private void isSettingPrivateModel() {
        AlertDialog mSettingPrivate = new AlertDialog.Builder(mContext).setTitle(mStrSettingTitle)
                .setIconAttribute(android.R.attr.alertDialogIcon).setCancelable(false)
                .setMessage(mStrSettingContent)
                .setPositiveButton(mStrSettingOk, new DialogInterface.OnClickListener() {
                    @Override
                    public void onClick(DialogInterface dialog, int which) {
                        startActivity(new Intent(mContext, BPSettingActivity.class));

                    }
                }).setNegativeButton(mStrSettingNo, new DialogInterface.OnClickListener() {

                    @Override
                    public void onClick(DialogInterface arg0, int arg1) {
                        modelType = modelNormal;
                        changeModelView();
                    }

                }).create();
        mSettingPrivate.show();
    }


    private void changeModelView() {
        if (modelType == modelPrivate) {
            mModleNormal.setBackgroundResource(R.drawable.button_shape_left_normal);
            mModleNormal.setTextColor(white);

            mModlePrivate.setBackgroundResource(R.drawable.button_shape_right_press);
            mModlePrivate.setTextColor(backgroundColor);

        } else if (modelType == modelNormal) {

            mModleNormal.setBackgroundResource(R.drawable.button_shape_left_press);
            mModleNormal.setTextColor(backgroundColor);

            mModlePrivate.setBackgroundResource(R.drawable.button_shape_right_normal);
            mModlePrivate.setTextColor(white);

        }
    }

    @BindString(R.string.command_ble_disconnect_toast)
    String mNeedConnectBLE;

    @OnClick(R.id.detect_start)
    public void onClickDetect() {
        changeDetectView();
    }

    private void changeDetectView() {
        boolean connected = SpUtil.getBoolean(mContext, SputilVari.BLE_IS_CONNECT_DEVICE_SUCCESS, false);
        if (!connected) {
            Toast.makeText(mContext, mNeedConnectBLE, Toast.LENGTH_SHORT).show();
            mBpState.setText(mNeedConnectBLE);
            return;
        }
        if (isDetecting) {
            stopDetectView();
        } else {
            if (notify.isChecked()) {
                startDetectView(false); //有波形
            } else {
                startDetectView(true); //无波形
            }
        }
    }

    private void startDetectView(boolean unnofity) {
        Logger.t(TAG).i("start Detect");
        isDetecting = true;
        isStartProgress = false;
        mDetectButImage.setImageResource(R.drawable.detect_bp_pause);
        mCircleView.setVisibility(View.VISIBLE);
        mBpChart.setVisibility(View.VISIBLE);
        mMiddleProgressTv.setVisibility(View.VISIBLE);

        mModleNormal.setEnabled(false);
        mModlePrivate.setEnabled(false);

        mBpState.setVisibility(View.GONE);
        mUnDetectView.setVisibility(View.GONE);

        mBPHander.readCurrentBp(modelType, unnofity);
        progrss = 0;
        if (scheduledThreadPool == null) {
            scheduledThreadPool = Executors.newSingleThreadScheduledExecutor();
        }


        mBpChart.invalidate();
    }

    private void stopDetectView() {
        Logger.t(TAG).i("stop Detect");
        isDetecting = false;
        isStartProgress = false;
        mMiddleProgressTv.setText("0");
        mDetectButImage.setImageResource(R.drawable.detect_bp_start);
        mBpChart.setVisibility(View.GONE);
        mBpState.setVisibility(View.VISIBLE);
        mBPHander.closeCurrentBp(modelType);
        mModleNormal.setEnabled(true);
        mModlePrivate.setEnabled(true);
        mCircleView.setInnerProgress(0);
        if (scheduledThreadPool != null) {
            scheduledThreadPool.shutdownNow();
            scheduledThreadPool = null;
        }

        if (null != mBpChart) {
            // 一组是7248
            mBpChart.moveViewToX(0);
            mBpChart.setData(mBpChart.getData());
            mBpChart.getData().getXVals().clear();
            mBpChart.getData().getDataSetByIndex(0).clear();
            mBpChart.fitScreen();
            mBpChart.notifyDataSetChanged();
            mBpChart.invalidate();
        }
        progrss = 0;
    }


    class RefreshProgress implements Runnable {

        @Override
        public void run() {
            mHandler.sendMessage(Message.obtain());
        }
    }

    /**
     * 监听本进程的广播
     */
    private void registerLocalBroadCaster() {
        LocalBroadcastManager.getInstance(mContext).registerReceiver(mBPbroadCaster, getFilter());
    }

    private void unRegisterLocalBroadCaster() {
        LocalBroadcastManager.getInstance(mContext).unregisterReceiver(mBPbroadCaster);
    }

    private IntentFilter getFilter() {
        IntentFilter mIntentFilter = new IntentFilter();
        mIntentFilter.addAction(BleBroadCast.DISCONNECTED_DEVICE_SUCCESS);
        mIntentFilter.addAction(BleProfile.BP_OPRATE);
        mIntentFilter.addAction(BleBroadCast.BP_ADC);
        return mIntentFilter;
    }

    /**
     * 收广播
     */
    private final BroadcastReceiver mBPbroadCaster = new BroadcastReceiver() {

        public void onReceive(Context context, Intent intent) {
            String action = intent.getAction();

            if (action.equals(BleBroadCast.DISCONNECTED_DEVICE_SUCCESS)) {
                Logger.t(TAG).i("断开了");
                stopDetectView();
            }
            if (action.equals(BleProfile.BP_OPRATE)) {
                if (isDetecting) {
                    byte[] data = intent.getByteArrayExtra(BleIntentPut.BLE_CMD);
                    Logger.t(TAG).i("接收到血压=" + ConvertHelper.byte2HexForShow(data));
                    showSaveDialog(data);
                }
            }
            if (action.equals(BleBroadCast.BP_ADC)) {
                Logger.t(TAG).i("接收ADC=" + isDetecting);
                byte[] data = intent.getByteArrayExtra(BleIntentPut.BLE_CMD);
                if (data.length != 20) {
                    return;
                }
                int adcViewArr[] = ConvertHelper.byte2HexForAdcTranslateIntArr(data);
                if (isDetecting) {

                    AddADCEntry(adcViewArr);
                }

            }

        }
    };
    @BindString(R.string.dpdetect_dialog_save_title)
    String mStrSaveTitle;
    @BindString(R.string.dpdetect_dialog_save_content)
    String mStrSaveContent;
    @BindString(R.string.dpdetect_dialog_save_ok)
    String mStrSaveOk;
    @BindString(R.string.dpdetect_dialog_save_no)
    String mStrSaveNo;
    @BindString(R.string.watch_is_busy)
    String mDeviceBusy;

    public void showSaveDialog(final byte[] data) {
        if (data.length < 5) {
            stopDetectView();
            return;
        }
        int arrInt[] = ConvertHelper.byte2HexToIntArr(data);
        final int highValue = arrInt[1];
        final int lowValue = arrInt[2];
        int progress = arrInt[3];
        int state = arrInt[4];
        if (data.length >= 6) {
            int isHaveProgress = arrInt[5];
            refreshProgress(isHaveProgress, progress);
        }

        if (state == 1 || state == 2 || state == 4 || state == 5) {
            mMiddleProgressTv.setText("0");
            mBpState.setText(mDeviceBusy);
            stopDetectView();
            return;
        }
        if (highValue == 0 && lowValue == 0) {
            //状态空闲下，00 00不处理
            return;
        }
        stopDetectView();
        if (!isAngioVailt(highValue, lowValue)) {
            mMiddleProgressTv.setText("0");
            mBpState.setText(mStrTestInvalit);
            return;
        }
        mMiddleProgressTv.setText(highValue + "/" + lowValue);
        AlertDialog mDisalog = new AlertDialog.Builder(mContext).setTitle(mStrSaveTitle)
                .setIconAttribute(android.R.attr.alertDialogIcon).setCancelable(false)
                .setMessage(String.format(mStrSaveContent, arrInt[1] + "/" + arrInt[2]))
                .setPositiveButton(mStrSaveOk, new DialogInterface.OnClickListener() {
                    @Override
                    public void onClick(DialogInterface dialog, int which) {
                        // 绑定不一样的手表时把今天的数据清除了
                        saveTheBP(highValue, lowValue);

                    }
                }).setNegativeButton(mStrSaveNo, new DialogInterface.OnClickListener() {

                    @Override
                    public void onClick(DialogInterface arg0, int arg1) {
                        mBpState.setText(getState(highValue, lowValue));
                    }

                }).create();
        mDisalog.show();
    }

    /**
     * 更新进度显示
     *
     * @param isHaveProgress
     */
    private void refreshProgress(int isHaveProgress, int progress) {
        if (isHaveProgress == 0) {
            Logger.t(TAG).i("接收到血压,返回假进度");
            if (!isStartProgress) {
                scheduledThreadPool.scheduleAtFixedRate(new RefreshProgress(), 0, 1100, TimeUnit.MILLISECONDS);
                isStartProgress = true;
            }
        } else if (isHaveProgress == 1) {

            if (progress <= 100) {
                Logger.t(TAG).i("接收到血压,返回真实");
                mCircleView.setInnerProgress((float) progress / 100);
                mMiddleProgressTv.setText(progress + "%");
            }
        } else {
            Logger.t(TAG).i("接收到血压,返回假进度");
            if (!isStartProgress) {
                scheduledThreadPool.scheduleAtFixedRate(new RefreshProgress(), 0, 1100, TimeUnit.MILLISECONDS);
                isStartProgress = true;
            }
        }
    }


    private void saveTheBP(int high, int low) {

        String account = SqlHelperUtil.getUserbean().getAccount();
        String mac = SpUtil.getString(mContext, SputilVari.BLE_LAST_CONNECT_ADDRESS, "");
        boolean isBind = SpUtil.getBoolean(mContext, SputilVari.BLE_IS_BINDER, false);
        TimeBean timeBean = new TimeBean();
        timeBean.setYear(TimeBean.getSysYear());
        timeBean.setMonth(TimeBean.getSysMonth());
        timeBean.setDay(TimeBean.getSysDay());
        timeBean.setHour(TimeBean.getSysHour());
        timeBean.setMinute(TimeBean.getSysMiute());

        BPBean bpBean = new BPBean(account, timeBean, high, low, true, mac, isBind);
        mBpState.setText(getState(high, low));
        timeBean.save();
        bpBean.save();
    }

    @BindString(R.string.dpdetect_bplow)
    String mStrbplow;
    @BindString(R.string.dpdetect_bphigh)
    String mStrbphigh;
    @BindString(R.string.dpdetect_bpnormal)
    String mStrbpnormal;
    @BindString(R.string.dpdetect_bpinvalit)
    String mStrinvalit;

    private String getState(int high, int low) {
        int result_low = -1, result_normal = 0, result_high = 1, result_err = 255;
        int state = BaseUtil.getBloodLevel(high, low);
        if (state == result_low) {
            return mStrbplow;
        } else if (state == result_high) {
            return mStrbphigh;
        } else if (state == result_normal) {
            return mStrbpnormal;
        } else {
            return mStrinvalit;
        }
    }

    private void setData() {
        values.clear();
        xVals.clear();
        LineDataSet dateSet = new LineDataSet(values, "");
        LineData data = new LineData(xVals, dateSet);

        dateSet.setDrawCircleHole(false);
        dateSet.setDrawCircles(false);
        dateSet.setHighlightEnabled(false);
        dateSet.setDrawHighlightIndicators(false);
        dateSet.setDrawValues(false);
        dateSet.setColor(Color.WHITE);

        mBpChart.setData(data);
    }

    private void AddADCEntry(int xValue[]) {
        LineData data = mBpChart.getData();
        if (data != null) {
            LineDataSet set = data.getDataSetByIndex(0);
            for (int i = 0; i < xValue.length; i++) {

                data.addXValue(data.getXValCount() + 1 + "");
                int value = xValue[i] / 10;
                if (value > highAdcValue) {
                    highAdcValue = value;
                }
                if (value < lowAdcValue) {
                    lowAdcValue = value;
                }
                data.addEntry(new Entry(value, set.getEntryCount()), 0);

            }
            Logger.t(TAG).i("data.getXValCount()=" + data.getXValCount());

            mBpChart.notifyDataSetChanged();
            mBpChart.setVisibleXRangeMaximum(1024);
//            if (data.getXValCount() > 1024) {
            mBpChart.moveViewToX(data.getXValCount() - 256);
//            }
            mBpChart.getAxisLeft().setAxisMaxValue(highAdcValue + 10);
            mBpChart.getAxisLeft().setAxisMinValue(lowAdcValue - 10);
            mBpChart.invalidate();
        }
    }


    private void setChartProperty() {
        mBpChart.setDrawGridBackground(false);
        mBpChart.getDescription().setEnabled(false);
        mBpChart.setNoDataText("");

        mBpChart.setScaleEnabled(false);
        mBpChart.setScaleYEnabled(false);


        mBpChart.getAxisLeft().setAxisMaxValue(MAX_VALUE_ADC);
        mBpChart.getAxisLeft().setAxisMinValue(MIN_VALUE_ADC);

        mBpChart.getAxisLeft().setDrawLabels(false);
        mBpChart.getAxisLeft().setDrawAxisLine(false);
        mBpChart.getAxisLeft().setDrawGridLines(false);

        mBpChart.getAxisRight().setDrawAxisLine(false);
        mBpChart.getAxisRight().setDrawLabels(false);
        mBpChart.getAxisRight().setDrawGridLines(false);

        mBpChart.getXAxis().setDrawAxisLine(false);
        mBpChart.getXAxis().setDrawLabels(false);
        mBpChart.getXAxis().setDrawGridLines(false);

        mBpChart.setDrawBorders(false);

        mBpChart.getLegend().setEnabled(false);

    }


    /**
     * 判断血压是否有有效 [60-300].[20-200]
     */
    private boolean isAngioVailt(int high, int low) {
        boolean flag = true;
        if (high < 60 || high > 300) {
            flag = false;
        }
        if (low < 20 || low > 200) {
            flag = false;
        }
        return flag;
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        stopDetectView();
        unRegisterLocalBroadCaster();
    }

    @BindView(R.id.detect_bp_layout)
    LinearLayout rootview;

    @OnClick(R.id.head_img_right)
    public void onClick() {
        Logger.t(TAG).e("Click Share");
        long mNowTime = System.currentTimeMillis();//获取第一次按键时间
        if ((mNowTime - mPressedTime) > 2000) {//比较两次按键时间差
            showShareView();
            mPressedTime = mNowTime;
        }
    }

    private void showShareView() {
        new Thread(new Runnable() {
            @Override
            public void run() {
                Bitmap myShot = Bitmap.createBitmap(rootview.getWidth(), rootview.getHeight(), Bitmap.Config.ARGB_8888);
                Canvas canvas = new Canvas(myShot);
                try {
                    rootview.draw(canvas);
                    ImageUtils.saveImage(mContext, SputilVari.SHARE_PNG_PATH, myShot);
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        }).start();
        ShareSDK.initSDK(mContext);
        OnekeyShare oks = new OnekeyShare();
        oks.disableSSOWhenAuthorize();
        oks.setImagePath(SputilVari.SHARE_PNG_PATH);
        oks.show(mContext);
    }


    @Override
    public void onPause() {
        super.onPause();
        onPause(this, TAG_UMENT);
    }


}
