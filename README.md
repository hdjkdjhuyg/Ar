package com.example.flowsense

import android.Manifest
import android.app.*
import android.bluetooth.BluetoothAdapter
import android.content.*
import android.content.pm.PackageManager
import android.hardware.Sensor
import android.hardware.SensorEvent
import android.hardware.SensorEventListener
import android.hardware.SensorManager
import android.location.Location
import android.net.wifi.WifiManager
import android.os.*
import android.provider.Settings
import androidx.core.app.ActivityCompat
import androidx.core.app.NotificationCompat
import androidx.recyclerview.widget.LinearLayoutManager
import androidx.work.*
import com.google.android.gms.location.*
import com.google.gson.Gson
import kotlinx.coroutines.*
import org.tensorflow.lite.Interpreter
import java.io.FileInputStream
import java.nio.MappedByteBuffer
import java.nio.channels.FileChannel
import java.util.*
import java.util.concurrent.TimeUnit
import kotlin.math.abs
import kotlin.math.sqrt

// Main Service for context monitoring and automation
class FlowSenseService : Service() {
    private lateinit var fusedLocationClient: FusedLocationProviderClient
    private lateinit var geofencingClient: GeofencingClient
    private lateinit var sensorManager: SensorManager
    private lateinit var wifiManager: WifiManager
    private lateinit var bluetoothAdapter: BluetoothAdapter
    private lateinit var patternDetector: AdvancedPatternDetector
    private lateinit var notificationManager: NotificationManager
    private var isRunning = false
    private val geofencePendingIntent: PendingIntent by lazy { createGeofencePendingIntent() }

    override fun onBind(intent: Intent?): IBinder? = null

    override fun onCreate() {
        super.onCreate()
        fusedLocationClient = LocationServices.getFusedLocationProviderClient(this)
        geofencingClient = LocationServices.getGeofencingClient(this)
        sensorManager = getSystemService(Context.SENSOR_SERVICE) as SensorManager
        wifiManager = getSystemService(Context.WIFI_SERVICE) as WifiManager
        bluetoothAdapter = BluetoothAdapter.getDefaultAdapter()
        patternDetector = AdvancedPatternDetector(this)
        notificationManager = getSystemService(Context.NOTIFICATION_SERVICE) as NotificationManager
        createNotificationChannel()
        schedulePeriodicPatternAnalysis()
    }

    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        startForeground(1, createNotification())
        if (!isRunning) {
            isRunning = true
            startContextMonitoring()
            checkPermissionsAndPrompt()
        }
        return START_STICKY
    }

    private fun createNotificationChannel() {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            val channel = NotificationChannel(
                "flowsense_channel",
                "FlowSense Service",
                NotificationManager.IMPORTANCE_LOW
            )
            notificationManager.createNotificationChannel(channel)
        }
    }

    private fun createNotification(): Notification {
        val intent = Intent(this, MainActivity::class.java)
        val pendingIntent = PendingIntent.getActivity(this, 0, intent, PendingIntent.FLAG_IMMUTABLE)
        return NotificationCompat.Builder(this, "flowsense_channel")
            .setContentTitle("FlowSense Active")
            .setContentText("Enhancing your device experience")
            .setSmallIcon(R.drawable.ic_notification)
            .setContentIntent(pendingIntent)
            .build()
    }

    private fun createGeofencePendingIntent(): PendingIntent {
        val intent = Intent(this, GeofenceBroadcastReceiver::class.java)
        return PendingIntent.getBroadcast(this, 0, intent, PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE)
    }

    private fun checkPermissionsAndPrompt() {
        val permissions = mutableListOf<String>()
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) {
            permissions.add(Manifest.permission.POST_NOTIFICATIONS)
        }
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.S) {
            permissions.addAll(listOf(Manifest.permission.BLUETOOTH_SCAN, Manifest.permission.BLUETOOTH_CONNECT))
        }
        permissions.addAll(listOf(Manifest.permission.ACCESS_FINE_LOCATION, Manifest.permission.ACCESS_WIFI_STATE, Manifest.permission.CHANGE_WIFI_STATE))
        
        if (permissions.any { ActivityCompat.checkSelfPermission(this, it) != PackageManager.PERMISSION_GRANTED }) {
            val intent = Intent(this, MainActivity::class.java).apply {
                putExtra("prompt_permissions", true)
                addFlags(Intent.FLAG_ACTIVITY_NEW_TASK)
            }
            startActivity(intent)
        }

        if (!Settings.System.canWrite(this)) {
            val intent = Intent(Settings.ACTION_MANAGE_WRITE_SETTINGS).apply {
                data = android.net.Uri.parse("package:$packageName")
                addFlags(Intent.FLAG_ACTIVITY_NEW_TASK)
            }
            startActivity(intent)
        }

        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
            val notificationManager = getSystemService(Context.NOTIFICATION_SERVICE) as NotificationManager
            if (!notificationManager.isNotificationPolicyAccessGranted) {
                val intent = Intent(Settings.ACTION_NOTIFICATION_POLICY_ACCESS_SETTINGS).apply {
                    addFlags(Intent.FLAG_ACTIVITY_NEW_TASK)
                }
                startActivity(intent)
            }
        }
    }

    private fun startContextMonitoring() {
        if (ActivityCompat.checkSelfPermission(this, Manifest.permission.ACCESS_FINE_LOCATION) == PackageManager.PERMISSION_GRANTED) {
            val locationRequest = LocationRequest.create().apply {
                interval = 60000
                fastestInterval = 30000
                priority = LocationRequest.PRIORITY_BALANCED_POWER_ACCURACY
            }
            fusedLocationClient.requestLocationUpdates(locationRequest, locationCallback, Looper.getMainLooper())
            setupGeofences()
        }

        val accelerometer = sensorManager.getDefaultSensor(Sensor.TYPE_ACCELEROMETER)
        sensorManager.registerListener(sensorListener, accelerometer, SensorManager.SENSOR_DELAY_NORMAL)

        registerReceiver(wifiReceiver, IntentFilter(WifiManager.WIFI_STATE_CHANGED_ACTION))
        registerReceiver(bluetoothReceiver, IntentFilter(BluetoothAdapter.ACTION_STATE_CHANGED))
        registerReceiver(batteryReceiver, IntentFilter(Intent.ACTION_BATTERY_CHANGED))
    }

    private fun setupGeofences() {
        val geofences = patternDetector.getKnownLocations().map { loc ->
            Geofence.Builder()
                .setRequestId(loc.id)
                .setCircularRegion(loc.latitude, loc.longitude, 100f)
                .setExpirationDuration(Geofence.NEVER_EXPIRE)
                .setTransitionTypes(Geofence.GEOFENCE_TRANSITION_ENTER or Geofence.GEOFENCE_TRANSITION_EXIT)
                .build()
        }
        if (geofences.isNotEmpty()) {
            val request = GeofencingRequest.Builder()
                .setInitialTrigger(GeofencingRequest.INITIAL_TRIGGER_ENTER)
                .addGeofences(geofences)
                .build()
            if (ActivityCompat.checkSelfPermission(this, Manifest.permission.ACCESS_FINE_LOCATION) == PackageManager.PERMISSION_GRANTED) {
                geofencingClient.addGeofences(request, geofencePendingIntent)
            }
        }
    }

    private fun schedulePeriodicPatternAnalysis() {
        val workRequest = PeriodicWorkRequestBuilder<PatternAnalysisWorker>(15, TimeUnit.MINUTES)
            .setConstraints(Constraints.Builder().setRequiredNetworkType(NetworkType.NOT_REQUIRED).build())
            .build()
        WorkManager.getInstance(this).enqueueUniquePeriodicWork("pattern_analysis", ExistingPeriodicWorkPolicy.KEEP, workRequest)
    }

    private val locationCallback = object : LocationCallback() {
        override fun onLocationResult(result: LocationResult) {
            result.lastLocation?.let { location ->
                patternDetector.updateLocation(location)
            }
        }
    }

    private val sensorListener = object : SensorEventListener {
        override fun onSensorChanged(event: SensorEvent?) {
            event?.let {
                if (it.sensor.type == Sensor.TYPE_ACCELEROMETER) {
                    val movement = sqrt(it.values[0] * it.values[0] + it.values[1] * it.values[1] + it.values[2] * it.values[2])
                    patternDetector.updateMovement(movement > 8.0)
                }
            }
        }

        override fun onAccuracyChanged(sensor: Sensor?, accuracy: Int) {}
    }

    private val wifiReceiver = object : BroadcastReceiver() {
        override fun onReceive(context: Context?, intent: Intent?) {
            val wifiState = wifiManager.wifiState
            patternDetector.updateWifiState(wifiState == WifiManager.WIFI_STATE_ENABLED)
        }
    }

    private val bluetoothReceiver = object : BroadcastReceiver() {
        override fun onReceive(context: Context?, intent: Intent?) {
            val state = bluetoothAdapter.state
            patternDetector.updateBluetoothState(state == BluetoothAdapter.STATE_ON)
        }
    }

    private val batteryReceiver = object : BroadcastReceiver() {
        override fun onReceive(context: Context?, intent: Intent?) {
            val level = intent?.getIntExtra(BatteryManager.EXTRA_LEVEL, -1) ?: -1
            val status = intent?.getIntExtra(BatteryManager.EXTRA_STATUS, -1) ?: -1
            patternDetector.updateBattery(level, status == BatteryManager.BATTERY_STATUS_CHARGING)
        }
    }

    override fun onDestroy() {
        super.onDestroy()
        isRunning = false
        fusedLocationClient.removeLocationUpdates(locationCallback)
        sensorManager.unregisterListener(sensorListener)
        unregisterReceiver(wifiReceiver)
        unregisterReceiver(bluetoothReceiver)
        unregisterReceiver(batteryReceiver)
        geofencingClient.removeGeofences(geofencePendingIntent)
    }
}

// Geofence Broadcast Receiver
class GeofenceBroadcastReceiver : BroadcastReceiver() {
    override fun onReceive(context: Context?, intent: Intent?) {
        val geofencingEvent = GeofencingEvent.fromIntent(intent ?: return)
        if (geofencingEvent.hasError()) return
        val transition = geofencingEvent.geofenceTransition
        val geofences = geofencingEvent.triggeringGeofences
        context?.let {
            val patternDetector = AdvancedPatternDetector(it)
            geofences.forEach { geofence ->
                patternDetector.handleGeofenceTransition(geofence.requestId, transition == Geofence.GEOFENCE_TRANSITION_ENTER)
            }
        }
    }
}

// Worker for periodic pattern analysis
class PatternAnalysisWorker(appContext: Context, workerParams: WorkerParameters) : CoroutineWorker(appContext, workerParams) {
    override suspend fun doWork(): Result {
        val patternDetector = AdvancedPatternDetector(applicationContext)
        patternDetector.analyzeContext()
        return Result.success()
    }
}

// Advanced Pattern Detector with ML and clustering
class AdvancedPatternDetector(private val context: Context) {
    private val patterns = mutableListOf<ContextPattern>()
    private var currentLocation: Location? = null
    private var isMoving = false
    private var isWifiOn = false
    private var isBluetoothOn = false
    private var batteryLevel = -1
    private var isCharging = false
    private val dbHelper = PatternDatabaseHelper(context)
    private val tfliteInterpreter: Interpreter? by lazy { loadTFLiteModel() }
    private val userFeedback = mutableMapOf<String, Int>() // Pattern ID to feedback score (1=accept, -1=reject)

    fun updateLocation(location: Location) {
        currentLocation = location
    }

    fun updateMovement(moving: Boolean) {
        isMoving = moving
    }

    fun updateWifiState(enabled: Boolean) {
        isWifiOn = enabled
    }

    fun updateBluetoothState(enabled: Boolean) {
        isBluetoothOn = enabled
    }

    fun updateBattery(level: Int, charging: Boolean) {
        batteryLevel = level
        isCharging = charging
    }

    fun handleGeofenceTransition(geofenceId: String, entered: Boolean) {
        val contextSnapshot = ContextSnapshot(
            location = currentLocation,
            time = Calendar.getInstance().get(Calendar.HOUR_OF_DAY),
            isMoving = isMoving,
            isWifiOn = isWifiOn,
            isBluetoothOn = isBluetoothOn,
            batteryLevel = batteryLevel,
            isCharging = isCharging,
            geofenceId = geofenceId
        )
        if (entered) {
            analyzeContext(contextSnapshot)
        }
    }

    fun getKnownLocations(): List<GeofenceLocation> {
        return dbHelper.getKnownLocations()
    }

    private fun loadTFLiteModel(): Interpreter? {
        return try {
            val tfliteModel = loadModelFile()
            Interpreter(tfliteModel)
        } catch (e: Exception) {
            null
        }
    }

    private fun loadModelFile(): MappedByteBuffer {
        val fileDescriptor = context.assets.openFd("pattern_model.tflite")
        val inputStream = FileInputStream(fileDescriptor.fileDescriptor)
        val fileChannel = inputStream.channel
        val startOffset = fileDescriptor.startOffset
        val declaredLength = fileDescriptor.declaredLength
        return fileChannel.map(FileChannel.MapMode.READ_ONLY, startOffset, declaredLength)
    }

    fun analyzeContext(contextSnapshot: ContextSnapshot = ContextSnapshot(
        location = currentLocation,
        time = Calendar.getInstance().get(Calendar.HOUR_OF_DAY),
        isMoving = isMoving,
        isWifiOn = isWifiOn,
        isBluetoothOn = isBluetoothOn,
        batteryLevel = batteryLevel,
        isCharging = isCharging,
        geofenceId = null
    )) {
        // Cluster contexts using simplified K-Means
        val recentContexts = dbHelper.getRecentContexts()
        val clusters = clusterContexts(recentContexts + contextSnapshot)
        val matchedCluster = clusters.find { cluster -> cluster.any { it.isSimilar(contextSnapshot) } }

        if (matchedCluster != null) {
            val pattern = patterns.find { it.context.isSimilar(contextSnapshot) }
            if (pattern != null && userFeedback[pattern.id]?.let { it >= 0 } != false) {
                executeAction(pattern.action, pattern.id)
            } else {
                val predictedAction = predictAction(contextSnapshot)
                val newPattern = ContextPattern(UUID.randomUUID().toString(), contextSnapshot, predictedAction)
                patterns.add(newPattern)
                dbHelper.savePattern(newPattern)
                suggestAction(newPattern)
            }
        } else {
            dbHelper.saveContext(contextSnapshot)
        }
    }

    private fun clusterContexts(contexts: List<ContextSnapshot>): List<List<ContextSnapshot>> {
        // Simplified K-Means clustering
        val k = 3 // Number of clusters
        val centroids = contexts.take(k).toMutableList()
        repeat(10) { // Iterate 10 times for convergence
            val clusters = mutableListOf<MutableList<ContextSnapshot>>().apply { repeat(k) { add(mutableListOf()) } }
            contexts.forEach { context ->
                val closestCentroidIndex = centroids.indices.minByOrNull { computeDistance(context, centroids[it]) } ?: 0
                clusters[closestCentroidIndex].add(context)
            }
            centroids.forEachIndexed { index, _ ->
                if (clusters[index].isNotEmpty()) {
                    centroids[index] = averageContext(clusters[index])
                }
            }
        }
        return clusters.filter { it.isNotEmpty() }
    }

    private fun computeDistance(c1: ContextSnapshot, c2: ContextSnapshot): Double {
        val weights = mapOf("location" to 0.4, "time" to 0.3, "moving" to 0.1, "wifi" to 0.1, "bluetooth" to 0.05, "battery" to 0.05)
        var distance = 0.0
        c1.location?.let { loc1 -> c2.location?.let { loc2 -> distance += weights["location"]!! * loc1.distanceTo(loc2) } }
        distance += weights["time"]!! * abs(c1.time - c2.time)
        distance += weights["moving"]!! * if (c1.isMoving == c2.isMoving) 0.0 else 1.0
        distance += weights["wifi"]!! * if (c1.isWifiOn == c2.isWifiOn) 0.0 else 1.0
        distance += weights["bluetooth"]!! * if (c1.isBluetoothOn == c2.isBluetoothOn) 0.0 else 1.0
        distance += weights["battery"]!! * abs(c1.batteryLevel - c2.batteryLevel) / 100.0
        return distance
    }

    private fun averageContext(contexts: List<ContextSnapshot>): ContextSnapshot {
        val avgLat = contexts.mapNotNull { it.location?.latitude }.average()
        val avgLon = contexts.mapNotNull { it.location?.longitude }.average()
        val avgTime = contexts.map { it.time }.average().toInt()
        val mostCommonMoving = contexts.groupBy { it.isMoving }.maxByOrNull { it.value.size }?.key ?: false
        val mostCommonWifi = contexts.groupBy { it.isWifiOn }.maxByOrNull { it.value.size }?.key ?: false
        val mostCommonBluetooth = contexts.groupBy { it.isBluetoothOn }.maxByOrNull { it.value.size }?.key ?: false
        val avgBattery = contexts.map { it.batteryLevel }.average().toInt()
        val mostCommonCharging = contexts.groupBy { it.isCharging }.maxByOrNull { it.value.size }?.key ?: false
        return ContextSnapshot(
            location = Location("").apply { latitude = avgLat; longitude = avgLon },
            time = avgTime,
            isMoving = mostCommonMoving,
            isWifiOn = mostCommonWifi,
            isBluetoothOn = mostCommonBluetooth,
            batteryLevel = avgBattery,
            isCharging = mostCommonCharging
        )
    }

    private fun predictAction(context: ContextSnapshot): String {
        tfliteInterpreter?.let { interpreter ->
            val input = floatArrayOf(
                context.location?.latitude?.toFloat() ?: 0f,
                context.location?.longitude?.toFloat() ?: 0f,
                context.time.toFloat(),
                if (context.isMoving) 1f else 0f,
                if (context.isWifiOn) 1f else 0f,
                if (context.isBluetoothOn) 1f else 0f,
                context.batteryLevel.toFloat(),
                if (context.isCharging) 1f else 0f
            )
            val output = Array(1) { FloatArray(3) } // 3 possible actions: reading, meeting, battery_saver
            interpreter.run(input, output)
            return when (output[0].indexOfMax()) {
                0 -> "reading_mode"
                1 -> "meeting_mode"
                2 -> "battery_saver_mode"
                else -> "default_mode"
            }
        } ?: run {
            return when {
                context.time >= 19 && context.isCharging && !context.isMoving -> "reading_mode"
                context.isBluetoothOn && context.isMoving -> "meeting_mode"
                context.batteryLevel < 20 && !context.isCharging -> "battery_saver_mode"
                else -> "default_mode"
            }
        }
    }

    private fun FloatArray.indexOfMax(): Int = indices.maxByOrNull { this[it] } ?: 0

    private fun suggestAction(pattern: ContextPattern) {
        val intent = Intent(context, MainActivity::class.java).apply {
            putExtra("suggested_pattern", Gson().toJson(pattern))
            addFlags(Intent.FLAG_ACTIVITY_NEW_TASK)
        }
        context.startActivity(intent)
    }

    fun recordFeedback(patternId: String, accepted: Boolean) {
        userFeedback[patternId] = userFeedback.getOrDefault(patternId, 0) + if (accepted) 1 else -1
        dbHelper.updatePatternFeedback(patternId, userFeedback[patternId]!!)
        if (userFeedback[patternId]!! <= -3) {
            patterns.removeAll { it.id == patternId }
            dbHelper.deletePattern(patternId)
        }
    }

    private fun executeAction(action: String, patternId: String) {
        when (action) {
            "reading_mode" -> {
                setDoNotDisturb(true)
                adjustBrightness(0.3f)
                adjustSoundMode(AudioManager.RINGER_MODE_SILENT)
                openApp("com.example.reader")
                scheduleContextualReminder("Reading time", "Enjoy your reading session")
            }
            "meeting_mode" -> {
                setDoNotDisturb(true)
                enableBluetooth()
                sendNotification("Meeting mode activated")
            }
            "battery_saver_mode" -> {
                enableBatterySaver()
                disableUnnecessaryServices()
                sendNotification("Battery saver mode activated")
            }
        }
        dbHelper.logActionExecution(patternId, action)
    }

    private fun setDoNotDisturb(enable: Boolean) {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
            val notificationManager = context.getSystemService(Context.NOTIFICATION_SERVICE) as NotificationManager
            if (notificationManager.isNotificationPolicyAccessGranted) {
                notificationManager.setInterruptionFilter(
                    if (enable) NotificationManager.INTERRUPTION_FILTER_PRIORITY
                    else NotificationManager.INTERRUPTION_FILTER_ALL
                )
            }
        }
    }

    private fun adjustBrightness(level: Float) {
        if (Settings.System.canWrite(context)) {
            val settings = Settings.System.getInt(context.contentResolver, Settings.System.SCREEN_BRIGHTNESS_MODE)
            if (settings == Settings.System.SCREEN_BRIGHTNESS_MODE_AUTOMATIC) {
                Settings.System.putInt(context.contentResolver, Settings.System.SCREEN_BRIGHTNESS_MODE, Settings.System.SCREEN_BRIGHTNESS_MODE_MANUAL)
            }
            Settings.System.putInt(context.contentResolver, Settings.System.SCREEN_BRIGHTNESS, (level * 255).toInt())
        }
    }

    private fun adjustSoundMode(mode: Int) {
        val audioManager = context.getSystemService(Context.AUDIO_SERVICE) as AudioManager
        audioManager.ringerMode = mode
    }

    private fun openApp(packageName: String) {
        val intent = context.packageManager.getLaunchIntentForPackage(packageName)
        if (intent != null) {
            context.startActivity(intent)
        }
    }

    private fun enableBluetooth() {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.S && ActivityCompat.checkSelfPermission(context, Manifest.permission.BLUETOOTH_CONNECT) == PackageManager.PERMISSION_GRANTED) {
            bluetoothAdapter.enable()
        }
    }

    private fun enableBatterySaver() {
        val powerManager = context.getSystemService(Context.POWER_SERVICE) as PowerManager
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP && !powerManager.isPowerSaveMode) {
            val intent = Intent(Settings.ACTION_BATTERY_SAVER_SETTINGS).apply {
                addFlags(Intent.FLAG_ACTIVITY_NEW_TASK)
            }
            context.startActivity(intent)
        }
    }

    private fun disableUnnecessaryServices() {
        if (ActivityCompat.checkSelfPermission(context, Manifest.permission.CHANGE_WIFI_STATE) == PackageManager.PERMISSION_GRANTED) {
            wifiManager.isWifiEnabled = false
        }
    }

    private fun sendNotification(message: String) {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU && ActivityCompat.checkSelfPermission(context, Manifest.permission.POST_NOTIFICATIONS) != PackageManager.PERMISSION_GRANTED) {
            return
        }
        val notification = NotificationCompat.Builder(context, "flowsense_channel")
            .setContentTitle("FlowSense Action")
            .setContentText(message)
            .setSmallIcon(R.drawable.ic_notification)
            .build()
        notificationManager.notify(Random().nextInt(), notification)
    }

    private fun scheduleContextualReminder(title: String, message: String) {
        val alarmManager = context.getSystemService(Context.ALARM_SERVICE) as AlarmManager
        val intent = Intent(context, ReminderBroadcastReceiver::class.java).apply {
            putExtra("title", title)
            putExtra("message", message)
        }
        val pendingIntent = PendingIntent.getBroadcast(context, Random().nextInt(), intent, PendingIntent.FLAG_IMMUTABLE)
        alarmManager.setExact(AlarmManager.RTC_WAKEUP, System.currentTimeMillis() + 60000, pendingIntent)
    }
}

// Reminder Broadcast Receiver
class ReminderBroadcastReceiver : BroadcastReceiver() {
    override fun onReceive(context: Context?, intent: Intent?) {
        val title = intent?.getStringExtra("title") ?: return
        val message = intent.getStringExtra("message") ?: return
        val notificationManager = context?.getSystemService(Context.NOTIFICATION_SERVICE) as NotificationManager
        val notification = NotificationCompat.Builder(context, "flowsense_channel")
            .setContentTitle(title)
            .setContentText(message)
            .setSmallIcon(R.drawable.ic_notification)
            .build()
        notificationManager.notify(Random().nextInt(), notification)
    }
}

// Data classes
data class ContextSnapshot(
    val location: Location?,
    val time: Int,
    val isMoving: Boolean,
    val isWifiOn: Boolean,
    val isBluetoothOn: Boolean,
    val batteryLevel: Int,
    val isCharging: Boolean,
    val geofenceId: String? = null
) {
    fun isSimilar(other: ContextSnapshot): Boolean {
        return (location?.distanceTo(other.location ?: return false) ?: Float.MAX_VALUE < 100) &&
                abs(time - other.time) <= 1 &&
                isMoving == other.isMoving &&
                isWifiOn == other.isWifiOn &&
                isBluetoothOn == other.isBluetoothOn &&
                abs(batteryLevel - other.batteryLevel) < 10 &&
                geofenceId == other.geofenceId
    }
}

data class ContextPattern(val id: String, val context: ContextSnapshot, val action: String) {
    fun matches(currentContext: ContextSnapshot): Boolean = context.isSimilar(currentContext)
}

data class GeofenceLocation(val id: String, val latitude: Double, val longitude: Double)

// Database Helper
class PatternDatabaseHelper(context: Context) : SQLiteOpenHelper(context, "flowsense.db", null, 2) {
    override fun onCreate(db: SQLiteDatabase) {
        db.execSQL("CREATE TABLE contexts (id INTEGER PRIMARY KEY, location_lat REAL, location_lon REAL, time INTEGER, is_moving INTEGER, is_wifi_on INTEGER, is_bluetooth_on INTEGER, battery_level INTEGER, is_charging INTEGER, geofence_id TEXT)")
        db.execSQL("CREATE TABLE patterns (id TEXT PRIMARY KEY, context_id INTEGER, action TEXT, feedback_score INTEGER)")
        db.execSQL("CREATE TABLE locations (id TEXT PRIMARY KEY, latitude REAL, longitude REAL)")
        db.execSQL("CREATE TABLE action_logs (id INTEGER PRIMARY KEY, pattern_id TEXT, action TEXT, timestamp INTEGER)")
    }

    override fun onUpgrade(db: SQLiteDatabase, oldVersion: Int, newVersion: Int) {
        db.execSQL("DROP TABLE IF EXISTS contexts")
        db.execSQL("DROP TABLE IF EXISTS patterns")
        db.execSQL("DROP TABLE IF EXISTS locations")
        db.execSQL("DROP TABLE IF EXISTS action_logs")
        onCreate(db)
    }

    fun saveContext(context: ContextSnapshot) {
        val db = writableDatabase
        val values = ContentValues().apply {
            put("location_lat", context.location?.latitude ?: 0.0)
            put("location_lon", context.location?.longitude ?: 0.0)
            put("time", context.time)
            put("is_moving", if (context.isMoving) 1 else 0)
            put("is_wifi_on", if (context.isWifiOn) 1 else 0)
            put("is_bluetooth_on", if (context.isBluetoothOn) 1 else 0)
            put("battery_level", context.batteryLevel)
            put("is_charging", if (context.isCharging) 1 else 0)
            put("geofence_id", context.geofenceId)
        }
        db.insert("contexts", null, values)
    }

    fun savePattern(pattern: ContextPattern) {
        val db = writableDatabase
        val contextValues = ContentValues().apply {
            put("location_lat", pattern.context.location?.latitude ?: 0.0)
            put("location_lon", pattern.context.location?.longitude ?: 0.0)
            put("time", pattern.context.time)
            put("is_moving", if (pattern.context.isMoving) 1 else 0)
            put("is_wifi_on", if (pattern.context.isWifiOn) 1 else 0)
            put("is_bluetooth_on", if (pattern.context.isBluetoothOn) 1 else 0)
            put("battery_level", pattern.context.batteryLevel)
            put("is_charging", if (pattern.context.isCharging) 1 else 0)
            put("geofence_id", pattern.context.geofenceId)
        }
        val contextId = db.insert("contexts", null, contextValues)
        val patternValues = ContentValues().apply {
            put("id", pattern.id)
            put("context_id", contextId)
            put("action", pattern.action)
            put("feedback_score", 0)
        }
        db.insert("patterns", null, patternValues)
    }

    fun updatePatternFeedback(patternId: String, score: Int) {
        val db = writableDatabase
        val values = ContentValues().apply { put("feedback_score", score) }
        db.update("patterns", values, "id = ?", arrayOf(patternId))
    }

    fun deletePattern(patternId: String) {
        val db = writableDatabase
        db.delete("patterns", "id = ?", arrayOf(patternId))
    }

    fun logActionExecution(patternId: String, action: String) {
        val db = writableDatabase
        val values = ContentValues().apply {
            put("pattern_id", patternId)
            put("action", action)
            put("timestamp", System.currentTimeMillis())
        }
        db.insert("action_logs", null, values)
    }

    fun getRecentContexts(): List<ContextSnapshot> {
        val contexts = mutableListOf<ContextSnapshot>()
        val db = readableDatabase
        val cursor = db.query("contexts", null, null, null, null, null, "id DESC", "100")
        while (cursor.moveToNext()) {
            contexts.add(
                ContextSnapshot(
                    location = Location("").apply {
                        latitude = cursor.getDouble(cursor.getColumnIndexOrThrow("location_lat"))
                        longitude = cursor.getDouble(cursor.getColumnIndexOrThrow("location_lon"))
                    },
                    time = cursor.getInt(cursor.getColumnIndexOrThrow("time")),
                    isMoving = cursor.getInt(cursor.getColumnIndexOrThrow("is_moving")) == 1,
                    isWifiOn = cursor.getInt(cursor.getColumnIndexOrThrow("is_wifi_on")) == 1,
                    isBluetoothOn = cursor.getInt(cursor.getColumnIndexOrThrow("is_bluetooth_on")) == 1,
                    batteryLevel = cursor.getInt(cursor.getColumnIndexOrThrow("battery_level")),
                    isCharging = cursor.getInt(cursor.getColumnIndexOrThrow("is_charging")) == 1,
                    geofenceId = cursor.getString(cursor.getColumnIndexOrThrow("geofence_id"))
                )
            )
        }
        cursor.close()
        return contexts
    }

    fun getKnownLocations(): List<GeofenceLocation> {
        val locations = mutableListOf<GeofenceLocation>()
        val db = readableDatabase
        val cursor = db.query("locations", null, null, null, null, null, null)
        while (cursor.moveToNext()) {
            locations.add(
                GeofenceLocation(
                    id = cursor.getString(cursor.getColumnIndexOrThrow("id")),
                    latitude = cursor.getDouble(cursor.getColumnIndexOrThrow("latitude")),
                    longitude = cursor.getDouble(cursor.getColumnIndexOrThrow("longitude"))
                )
            )
        }
        cursor.close()
        return locations
    }

    fun saveLocation(location: GeofenceLocation) {
        val db = writableDatabase
        val values = ContentValues().apply {
            put("id", location.id)
            put("latitude", location.latitude)
            put("longitude", location.longitude)
        }
        db.insert("locations", null, values)
    }
}

// Main Activity with enhanced UI
class MainActivity : AppCompatActivity() {
    private lateinit var binding: ActivityMainBinding
    private lateinit var patternAdapter: PatternAdapter
    private val patternDetector by lazy { AdvancedPatternDetector(this) }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        binding = ActivityMainBinding.inflate(layoutInflater)
        setContentView(binding.root)

        setupRecyclerView()
        requestPermissions()

        val serviceIntent = Intent(this, FlowSenseService::class.java)
        startForegroundService(serviceIntent)

        intent.getStringExtra("suggested_pattern")?.let { patternJson ->
            val pattern = Gson().fromJson(patternJson, ContextPattern::class.java)
            showSuggestedPatternDialog(pattern)
        }

        binding.btnAddPattern.setOnClickListener { showAddPatternDialog() }
        binding.btnViewLogs.setOnClickListener { showActionLogs() }
    }

    private fun setupRecyclerView() {
        patternAdapter = PatternAdapter(patternDetector.getRecentContexts().map { ContextPattern(UUID.randomUUID().toString(), it, "default_mode") }) { pattern, accepted ->
            patternDetector.recordFeedback(pattern.id, accepted)
        }
        binding.rvPatterns.apply {
            adapter = patternAdapter
            layoutManager = LinearLayoutManager(this@MainActivity)
        }
    }

    private fun requestPermissions() {
        val permissions = mutableListOf(
            Manifest.permission.ACCESS_FINE_LOCATION,
            Manifest.permission.ACCESS_WIFI_STATE,
            Manifest.permission.CHANGE_WIFI_STATE
        )
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.S) {
            permissions.addAll(listOf(Manifest.permission.BLUETOOTH_SCAN, Manifest.permission.BLUETOOTH_CONNECT))
        }
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) {
            permissions.add(Manifest.permission.POST_NOTIFICATIONS)
        }
        ActivityCompat.requestPermissions(this, permissions.toTypedArray(), 100)
    }

    private fun showSuggestedPatternDialog(pattern: ContextPattern) {
        AlertDialog.Builder(this)
            .setTitle("Suggested Automation")
            .setMessage("Activate ${pattern.action} for this context?")
            .setPositiveButton("Accept") { _, _ ->
                patternDetector.recordFeedback(pattern.id, true)
                Toast.makeText(this, "${pattern.action} activated", Toast.LENGTH_SHORT).show()
            }
            .setNegativeButton("Decline") { _, _ ->
                patternDetector.recordFeedback(pattern.id, false)
            }
            .setNeutralButton("Customize") { _, _ -> showCustomizePatternDialog(pattern) }
            .show()
    }

    private fun showAddPatternDialog() {
        val dialogView = layoutInflater.inflate(R.layout.dialog_add_pattern, null)
        // Implement dialog for manual pattern creation
        AlertDialog.Builder(this)
            .setView(dialogView)
            .setTitle("Add Custom Pattern")
            .setPositiveButton("Save") { _, _ ->
                // Save custom pattern to database
                Toast.makeText(this, "Pattern saved", Toast.LENGTH_SHORT).show()
            }
            .setNegativeButton("Cancel", null)
            .show()
    }

    private fun showCustomizePatternDialog(pattern: ContextPattern) {
        val dialogView = layoutInflater.inflate(R.layout.dialog_customize_pattern, null)
        // Implement dialog for customizing pattern actions
        AlertDialog.Builder(this)
            .setView(dialogView)
            .setTitle("Customize Pattern")
            .setPositiveButton("Save") { _, _ ->
                // Update pattern in database
                Toast.makeText(this, "Pattern updated", Toast.LENGTH_SHORT).show()
            }
            .setNegativeButton("Cancel", null)
            .show()
    }

    private fun showActionLogs() {
        // Implement action log display using RecyclerView
        Toast.makeText(this, "Action logs displayed", Toast.LENGTH_LONG).show()
    }
}

// Pattern Adapter for RecyclerView
class PatternAdapter(
    private var patterns: List<ContextPattern>,
    private val onFeedback: (ContextPattern, Boolean) -> Unit
) : RecyclerView.Adapter<PatternAdapter.PatternViewHolder>() {
    class PatternViewHolder(val binding: ItemPatternBinding) : RecyclerView.ViewHolder(binding.root)

    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): PatternViewHolder {
        val binding = ItemPatternBinding.inflate(LayoutInflater.from(parent.context), parent, false)
        return PatternViewHolder(binding)
    }

    override fun onBindViewHolder(holder: PatternViewHolder, position: Int) {
        val pattern = patterns[position]
        with(holder.binding) {
            tvPatternDescription.text = "Action: ${pattern.action}, Time: ${pattern.context.time}, Location: ${pattern.context.location?.latitude}"
            btnAccept.setOnClickListener { onFeedback(pattern, true) }
            btnReject.setOnClickListener { onFeedback(pattern, false) }
        }
    }

    override fun getItemCount(): Int = patterns.size

    fun updatePatterns(newPatterns: List<ContextPattern>) {
        patterns = newPatterns
        notifyDataSetChanged()
    }
}

// View Binding classes (placeholders)
class ActivityMainBinding {
    lateinit var root: View
    lateinit var rvPatterns: RecyclerView
    lateinit var btnAddPattern: Button
    lateinit var btnViewLogs: Button

    companion object {
        fun inflate(inflater: LayoutInflater): ActivityMainBinding {
            return ActivityMainBinding()
        }
    }
}

class ItemPatternBinding {
    lateinit var root: View
    lateinit var tvPatternDescription: TextView
    lateinit var btnAccept: Button
    lateinit var btnReject: Button

    companion object {
        fun inflate(inflater: LayoutInflater, parent: ViewGroup, attach: Boolean): ItemPatternBinding {
            return ItemPatternBinding()
        }
    }
}

// Unit Tests
class PatternDetectorTest {
    @Test
    fun testContextSimilarity() {
        val context1 = ContextSnapshot(Location("").apply { latitude = 10.0; longitude = 20.0 }, 19, true, true, true, 50, true)
        val context2 = ContextSnapshot(Location("").apply { latitude = 10.1; longitude = 20.1 }, 19, true, true, true, 48, true)
        assertTrue(context1.isSimilar(context2))
    }

    @Test
    fun testClustering() {
        val detector = AdvancedPatternDetector(ApplicationProvider.getApplicationContext())
        val contexts = listOf(
            ContextSnapshot(Location("").apply { latitude = 10.0; longitude = 20.0 }, 19, true, true, true, 50, true),
            ContextSnapshot(Location("").apply { latitude = 10.1; longitude = 20.1 }, 19, true, true, true, 48, true),
            ContextSnapshot(Location("").apply { latitude = 50.0; longitude = 60.0 }, 8, false, false, false, 20, false)
        )
        val clusters = detector.clusterContexts(contexts)
        assertEquals(2, clusters.size)
    }
}

// AndroidManifest.xml (not code, but critical configuration)
