# ActivityRecognition

1. Activity Recognition
Earlier user activity was detected using LocationClient, ActivityRecognitionApi. But these were deprecated recently and now we have to use ActivityRecognitionClient to do the same (hopefully it won’t be deprecated again, but their main intent is providing much efficient api).

Here is code snippet that can be used to detect the activity.

ActivityRecognitionClient mActivityRecognitionClient = new ActivityRecognitionClient(this);
 
Task<Void> task = mActivityRecognitionClient.requestActivityUpdates(
            Constants.DETECTION_INTERVAL_IN_MILLISECONDS,
            mPendingIntent);
 
ActivityRecognitionResult result = ActivityRecognitionResult.extractResult(intent);
ArrayList<DetectedActivity> detectedActivities = (ArrayList) result.getProbableActivities();
 
for (DetectedActivity activity : detectedActivities) {
     Log.e(TAG, "Detected activity: " + activity.getType() + ", " + activity.getConfidence());
}
Let’s start this by creating new project in Android Studio.

2. Creating New Project
android-activity-recognition
1. Create a new project in Android Studio from File ⇒ New Project and select Basic Activity from templates.

2. Download this res folder and add the drawables to your project’s res. This folder contains necessary drawables required for this example project.

3. Open build.gradle file and add play-services-location dependency.

build.gradle
dependencies {
    implementation fileTree(dir: 'libs', include: ['*.jar'])
    implementation 'com.android.support:appcompat-v7:26.1.0'
    //...
 
    compile 'com.google.android.gms:play-services-location:11.6.0'
}
4. Create a class named Constants.java to keep few global variables like activity frequency interval and confidence threshold.

Constants.java
package info.androidhive.activityrecognition; 
public class Constants {
 
    public static final String BROADCAST_DETECTED_ACTIVITY = "activity_intent";
 
    static final long DETECTION_INTERVAL_IN_MILLISECONDS = 30 * 1000;
 
    public static final int CONFIDENCE = 70;
}
5. Create a class named DetectedActivitiesIntentService.java and extend it from IntentService. In onHandleIntent() method, list of probable activities will be retried and broadcasted using LocalBroadcastManager.

DetectedActivitiesIntentService.java
package info.androidhive.activityrecognition;
import android.app.IntentService;
import android.content.Intent;
import android.support.v4.content.LocalBroadcastManager;
import android.util.Log;
 
import com.google.android.gms.location.ActivityRecognitionResult;
import com.google.android.gms.location.DetectedActivity;
 
import java.util.ArrayList;
 
public class DetectedActivitiesIntentService  extends IntentService {
 
    protected static final String TAG = DetectedActivitiesIntentService.class.getSimpleName();
 
    public DetectedActivitiesIntentService() {
        // Use the TAG to name the worker thread.
        super(TAG);
    }
 
    @Override
    public void onCreate() {
        super.onCreate();
    }
 
    @SuppressWarnings("unchecked")
    @Override
    protected void onHandleIntent(Intent intent) {
        ActivityRecognitionResult result = ActivityRecognitionResult.extractResult(intent);
 
        // Get the list of the probable activities associated with the current state of the
        // device. Each activity is associated with a confidence level, which is an int between
        // 0 and 100.
        ArrayList<DetectedActivity> detectedActivities = (ArrayList) result.getProbableActivities();
 
        for (DetectedActivity activity : detectedActivities) {
            Log.e(TAG, "Detected activity: " + activity.getType() + ", " + activity.getConfidence());
            broadcastActivity(activity);
        }
    }
 
    private void broadcastActivity(DetectedActivity activity) {
        Intent intent = new Intent(Constants.BROADCAST_DETECTED_ACTIVITY);
        intent.putExtra("type", activity.getType());
        intent.putExtra("confidence", activity.getConfidence());
        LocalBroadcastManager.getInstance(this).sendBroadcast(intent);
    }
}
6. We need to create another background service class named BackgroundDetectedActivitiesService.java which runs in background and triggers the activities in an interval basis.

BackgroundDetectedActivitiesService.java
package info.androidhive.activityrecognition;
 
import android.app.PendingIntent;
import android.app.Service;
import android.content.Intent;
import android.os.Binder;
import android.os.IBinder;
import android.support.annotation.NonNull;
import android.support.annotation.Nullable;
import android.util.Log;
import android.widget.Toast;
 
import com.google.android.gms.location.ActivityRecognitionClient;
import com.google.android.gms.tasks.OnFailureListener;
import com.google.android.gms.tasks.OnSuccessListener;
import com.google.android.gms.tasks.Task;
 
public class BackgroundDetectedActivitiesService extends Service {
    private static final String TAG = BackgroundDetectedActivitiesService.class.getSimpleName();
 
    private Intent mIntentService;
    private PendingIntent mPendingIntent;
    private ActivityRecognitionClient mActivityRecognitionClient;
 
    IBinder mBinder = new BackgroundDetectedActivitiesService.LocalBinder();
 
    public class LocalBinder extends Binder {
        public BackgroundDetectedActivitiesService getServerInstance() {
            return BackgroundDetectedActivitiesService.this;
        }
    }
 
    public BackgroundDetectedActivitiesService() {
 
    }
 
    @Override
    public void onCreate() {
        super.onCreate();
        mActivityRecognitionClient = new ActivityRecognitionClient(this);
        mIntentService = new Intent(this, DetectedActivitiesIntentService.class);
        mPendingIntent = PendingIntent.getService(this, 1, mIntentService, PendingIntent.FLAG_UPDATE_CURRENT);
        requestActivityUpdatesButtonHandler();
    }
 
    @Nullable
    @Override
    public IBinder onBind(Intent intent) {
        return mBinder;
    }
 
    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        super.onStartCommand(intent, flags, startId);
        return START_STICKY;
    }
 
    public void requestActivityUpdatesButtonHandler() {
        Task<Void> task = mActivityRecognitionClient.requestActivityUpdates(
                Constants.DETECTION_INTERVAL_IN_MILLISECONDS,
                mPendingIntent);
 
        task.addOnSuccessListener(new OnSuccessListener<Void>() {
            @Override
            public void onSuccess(Void result) {
                Toast.makeText(getApplicationContext(),
                        "Successfully requested activity updates",
                        Toast.LENGTH_SHORT)
                        .show();
            }
        });
 
        task.addOnFailureListener(new OnFailureListener() {
            @Override
            public void onFailure(@NonNull Exception e) {
                Toast.makeText(getApplicationContext(),
                        "Requesting activity updates failed to start",
                        Toast.LENGTH_SHORT)
                        .show();
            }
        });
    }
 
    public void removeActivityUpdatesButtonHandler() {
        Task<Void> task = mActivityRecognitionClient.removeActivityUpdates(
                mPendingIntent);
        task.addOnSuccessListener(new OnSuccessListener<Void>() {
            @Override
            public void onSuccess(Void result) {
                Toast.makeText(getApplicationContext(),
                        "Removed activity updates successfully!",
                        Toast.LENGTH_SHORT)
                        .show();
            }
        });
 
        task.addOnFailureListener(new OnFailureListener() {
            @Override
            public void onFailure(@NonNull Exception e) {
                Toast.makeText(getApplicationContext(), "Failed to remove activity updates!",
                        Toast.LENGTH_SHORT).show();
            }
        });
    }
 
    @Override
    public void onDestroy() {
        super.onDestroy();
        removeActivityUpdatesButtonHandler();
    }
}
7. Open AndroidManifest.xml and add ACTIVITY_RECOGNITION permission. Also add DetectedActivitiesIntentService and BackgroundDetectedActivitiesService services.
 
AndroidManifest.xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="info.androidhive.activityrecognition">
 
    <uses-permission android:name="com.google.android.gms.permission.ACTIVITY_RECOGNITION" />
 
    <application
        android:allowBackup="true"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:supportsRtl="true"
        android:theme="@style/AppTheme">
        <activity android:name=".MainActivity">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
 
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
 
        <!-- Service that provides activity recognition data. Setting the android:exported attribute
        to "false" stops other apps from starting this service, even when using an explicit
        intent. -->
        <service
            android:name=".DetectedActivitiesIntentService"
            android:exported="false" />
 
        <service android:name=".BackgroundDetectedActivitiesService"></service>
    </application>
 
</manifest>
3. Receiving Activity Updates
Now we have everything ready. Let’s see how to receive the user activity updates in main activity or in any other activity.

8. Open the layout file main activity (activity_main.xml) and below layout code. Here we are adding an ImageView (to display an icon representing the activity) and TextView (to display the activity). We also need two buttons to start and start the activity recognition service.

activity_main.xml
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:padding="16dp"
    tools:context="info.androidhive.activityrecognition.MainActivity">
 
    <LinearLayout
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_centerHorizontal="true"
        android:orientation="vertical">
 
        <ImageView
            android:id="@+id/img_activity"
            android:layout_width="wrap_content"
            android:layout_height="200dp"
            android:tint="#606060" />
 
        <TextView
            android:id="@+id/txt_activity"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_gravity="center_horizontal"
            android:textAllCaps="true"
            android:textColor="@color/colorPrimary"
            android:textSize="18dp"
            android:textStyle="bold" />
 
        <TextView
            android:id="@+id/txt_confidence"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_gravity="center_horizontal"
            android:layout_marginTop="10dp"
            android:textAllCaps="true"
            android:textSize="14dp" />
 
    </LinearLayout>
 
    <Button
        android:id="@+id/btn_start_tracking"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_alignParentBottom="true"
        android:layout_alignParentLeft="true"
        android:text="Start Tracking" />
 
    <Button
        android:id="@+id/btn_stop_tracking"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_alignParentBottom="true"
        android:layout_alignParentRight="true"
        android:text="Stop Tracking" />
</RelativeLayout>
9. Finally open MainActivity.java and do the below necessary changes.

> startTracking() and stopTracking() methods start or stops the user activity updates by toggling the service state.

> BroadcastReceiver is used to receive the activity updates from the IntentService.

> handleUserActivity() receives the user activity updates and displays the information on UI when the confidence value exceeds the threshold confidence value.

> In onResume() and onPause() methods the local broadcast receiver is registered and removed.

MainActivity.java
package info.androidhive.activityrecognition;
 
import android.content.BroadcastReceiver;
import android.content.Context;
import android.content.Intent;
import android.content.IntentFilter;
import android.support.v4.content.LocalBroadcastManager;
import android.support.v7.app.AppCompatActivity;
import android.os.Bundle;
import android.util.Log;
import android.view.View;
import android.widget.Button;
import android.widget.ImageView;
import android.widget.TextView;
 
import com.google.android.gms.location.DetectedActivity;
 
public class MainActivity extends AppCompatActivity {
 
    private String TAG = MainActivity.class.getSimpleName();
    BroadcastReceiver broadcastReceiver;
 
    private TextView txtActivity, txtConfidence;
    private ImageView imgActivity;
    private Button btnStartTrcking, btnStopTracking;
 
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
 
        txtActivity = findViewById(R.id.txt_activity);
        txtConfidence = findViewById(R.id.txt_confidence);
        imgActivity = findViewById(R.id.img_activity);
        btnStartTrcking = findViewById(R.id.btn_start_tracking);
        btnStopTracking = findViewById(R.id.btn_stop_tracking);
 
        btnStartTrcking.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View view) {
                startTracking();
            }
        });
 
        btnStopTracking.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View view) {
                stopTracking();
            }
        });
 
        broadcastReceiver = new BroadcastReceiver() {
            @Override
            public void onReceive(Context context, Intent intent) {
                if (intent.getAction().equals(Constants.BROADCAST_DETECTED_ACTIVITY)) {
                    int type = intent.getIntExtra("type", -1);
                    int confidence = intent.getIntExtra("confidence", 0);
                    handleUserActivity(type, confidence);
                }
            }
        };
 
        startTracking();
    }
 
    private void handleUserActivity(int type, int confidence) {
        String label = getString(R.string.activity_unknown);
        int icon = R.drawable.ic_still;
 
        switch (type) {
            case DetectedActivity.IN_VEHICLE: {
                label = getString(R.string.activity_in_vehicle);
                icon = R.drawable.ic_driving;
                break;
            }
            case DetectedActivity.ON_BICYCLE: {
                label = getString(R.string.activity_on_bicycle);
                icon = R.drawable.ic_on_bicycle;
                break;
            }
            case DetectedActivity.ON_FOOT: {
                label = getString(R.string.activity_on_foot);
                icon = R.drawable.ic_walking;
                break;
            }
            case DetectedActivity.RUNNING: {
                label = getString(R.string.activity_running);
                icon = R.drawable.ic_running;
                break;
            }
            case DetectedActivity.STILL: {
                label = getString(R.string.activity_still);
                break;
            }
            case DetectedActivity.TILTING: {
                label = getString(R.string.activity_tilting);
                icon = R.drawable.ic_tilting;
                break;
            }
            case DetectedActivity.WALKING: {
                label = getString(R.string.activity_walking);
                icon = R.drawable.ic_walking;
                break;
            }
            case DetectedActivity.UNKNOWN: {
                label = getString(R.string.activity_unknown);
                break;
            }
        }
 
        Log.e(TAG, "User activity: " + label + ", Confidence: " + confidence);
 
        if (confidence > Constants.CONFIDENCE) {
            txtActivity.setText(label);
            txtConfidence.setText("Confidence: " + confidence);
            imgActivity.setImageResource(icon);
        }
    }
 
    @Override
    protected void onResume() {
        super.onResume();
 
        LocalBroadcastManager.getInstance(this).registerReceiver(broadcastReceiver,
                new IntentFilter(Constants.BROADCAST_DETECTED_ACTIVITY));
    }
 
    @Override
    protected void onPause() {
        super.onPause();
 
        LocalBroadcastManager.getInstance(this).unregisterReceiver(broadcastReceiver);
    }
 
    private void startTracking() {
        Intent intent1 = new Intent(MainActivity.this, BackgroundDetectedActivitiesService.class);
        startService(intent1);
    }
 
    private void stopTracking() {
        Intent intent = new Intent(MainActivity.this, BackgroundDetectedActivitiesService.class);
        stopService(intent);
    }
}
Now run the app and test it once. You might want to Walk, Run or Drive if want to see other states.
