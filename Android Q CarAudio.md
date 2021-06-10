# Android Q CarAudio 

## 一、Q的行为变更



[TOC]



### 1.1动态路由支持

在Car的领域里，使用设置audioUseDynamicRouting属性在config.xml来打开动态音频路由，默认为false，谷歌建议打开

/packages/services/Car/service/res/values/config.xml

![157513624973](C:\Users\GW00175635\Desktop\Android Q CarAudio.assets\1577513624973.png)

**dynamic routing 开启**

Audio devices 按照（zones）区域进行分组

最少有一个通用的的Zone（域），扩展第二个Zone例如RSE（Reat？Rear Seat Entertainment）后排娱乐

每一个域中的audio devices都被分层volume groups用来控制音频

按照AudioAttributes里的usage制定音频到audio device

**dynamic routing 关闭**

只有一个通用的域

每一个音频组都代表一个Stream Type，然后调用AudioManager进行设置音量

初始化设置详见3.3 3.4内容

### 1.2动态路由区域AOSP示例

![1577934466517](Android Q CarAudio.assets\1577934466517.png)

![1577935328040](Android Q CarAudio.assets\1577935328040.png)

传统音量组分配

![1577956022847](Android Q CarAudio.assets\1577956022847.png)

### 1.3车辆音频焦点

CarZoneAudioFocus

单独管理每一个zone的音频焦点，并和UID进行交互

具体策略详见四、音频焦点AudioFocus



## 二、对外接口CarAudioManager

### 2.1 API列表

| 返回值                  | 方法                        | 参数                                                         | 描述                                                         |
| ----------------------- | --------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| boolean                 | isDynamicRoutingEnabled     |                                                              | 返回是否动态路由是否可用的                                   |
| void                    | setGroupVolume              | int groupId, int index, int flags                            | 设置组音量的音量值到primary zone                             |
| void                    | setGroupVolume              | int zoneId, int groupId, int index, int flags                | 设置组音量的音量值                                           |
| int                     | getGroupMaxVolume           | int groupId                                                  | 获取通用域的传入groupId最大组音量                            |
| int                     | getGroupMaxVolume           | int zoneId, int groupId                                      | 获取传入zoneId域里的groupId最大组音量                        |
| int                     | getGroupMinVolume           | int groupId                                                  | 获取通用域的传入groupId最小组音量                            |
| int                     | getGroupMinVolume           | int zoneId, int groupId                                      | 获取zoneId域里的groupId最小组音量                            |
| int                     | getGroupVolume              | int groupId                                                  | 获取通用域的传入groupId音量值                                |
| int                     | getGroupVolume              | int zoneId, int groupId                                      | 获取传入zoneId域里的groupId音量值                            |
| void                    | setFadeTowardFront          | float value                                                  | 设置前后音量偏移，0.0是平衡，1.0是前面                       |
| void                    | setBalanceTowardRight       | float value                                                  | 设置左右音量偏移，0.0是平衡，1.0是右面                       |
| String[]                | getExternalSources          |                                                              | 获取外部音源，除麦克风外的输入设备（警报音、DVD、收音机等）  |
| CarAudio<br>PatchHandle | createAudioPatch            | String sourceAddress,            @AudioAttributes.AttributeUsage int usage, int gainInMillibels | 通过getExternalSources给出的input port，创建一个外部音源到output的补丁，返回一个CarAudioPatchHandle |
| void                    | releaseAudioPatch           | CarAudioPatchHandle patch                                    | 释放input port和output的关联                                 |
| int                     | getVolumeGroupCount         |                                                              | 获取通用域的可用音量组数目                                   |
| int                     | getVolumeGroupCount         | int zoneId                                                   | 获取zoneId指定域的可用音量组数目                             |
| int                     | getVolumeGroupIdForUsage    | @AudioAttributes.AttributeUsage int usage                    | 获取传入音频用例对应的音量组Id                               |
| int                     | getVolumeGroupIdForUsage    | int zoneId @AudioAttributes.AttributeUsage int usage         | 获取zoneId指定域传入音频用例对应的音量组Id                   |
| int[]                   | getAudioZoneIds             |                                                              | 获取所有音频域的id                                           |
| int                     | getZoneIdForUid             | int uid                                                      | 获取uid映射的zoneId，没有映射返回primaryId                   |
| boolean                 | setZoneIdForUid             | int zoneId, int uid                                          | 设置zoneId和uid的映射                                        |
| boolean                 | clearZoneIdForUid           | int uid                                                      | 清除uid的映射                                                |
| int                     | getZoneIdForDisplay         | Display display                                              | 获取指定display的zoneId，没有找到返回primaryId               |
| int                     | getZoneIdForDisplayPortId   | byte displayPortId                                           | 获取指定display端口ID所对应的zoneId，没有找到返回primaryId   |
| int[]                   | getUsagesForVolumeGroupId   | int groupId                                                  | 获取通用域里指定groupId所有的音频用例                        |
| int[]                   | getUsagesForVolumeGroupId   | int zoneId, int groupId                                      | 获取指定zoneId域里指定groupId所有的音频用例                  |
| void                    | registerCarVolumeCallback   | CarVolumeCallback callback                                   | 注册音量callback,添加到CarAudioManager维护的Callback组里，有onGroupVolumeChanged和onMasterMuteChanged的回调 |
| void                    | unregisterCarVolumeCallback | CarVolumeCallback callback                                   | 注销音量callback，从Callback组里删除                         |

### 2.2CarAudioManager的总结

CarAudiManager的构造方法里通过AIDL方式获取CarAudioService，并注册音量监听。并将音量监回调到其他注册CarAudioManager的Callback。

CarAudioManager实现了CarManagerBase的接口，即实现了onCarDisconnected()方法，在onCarDisConnected里注销CarAudioService里的音量监听。

需要Car.PERMISSION_CAR_CONTROL_AUDIO_VOLUME权限

获取CarAudioManager的步骤，首先要获取Car，然后mCar.getCarManager(Car.AUDIO_SERVICE);

```java
import android.car.Car;

Car mCar;
CarAudioManager mAudioManager;
private void setup(){
    mCar = Car.createCar(mContext, mConnectionCallbacks); 
    mCar.connect();
}

private ServiceConnection mConnectionCallbacks = new ServiceConnection() {
    @Override
    public void onServiceConnected(ComponentName componentName, IBinder iBinder) {
        Log.d(TAG, "onServiceConnected: ");
        mCarAudioManager = (CarAudioManager) mCar.getCarManager(Car.AUDIO_SERVICE);
    }

    @Override
    public void onServiceDisconnected(ComponentName componentName) {
        Log.d(TAG, "onServiceDisconnected: ");
    }
};
```

CarAudioManager初始化过程，以及连接状态的维护

```java
onCarDisconnectedpublic Object getCarManager(String serviceName) {
    CarManagerBase manager;
    ICar service = getICarOrThrow();
    synchronized (mCarManagerLock) {
        manager = mServiceMap.get(serviceName);
        if (manager == null) {
            try {
                IBinder binder = service.getCarService(serviceName);
                if (binder == null) {
                    Log.w(CarLibLog.TAG_CAR, "getCarManager could not get binder for service:" +
                          serviceName);
                    return null;
                }
                manager = createCarManager(serviceName, binder);
                if (manager == null) {
                    Log.w(CarLibLog.TAG_CAR,
                          "getCarManager could not create manager for service:" +
                          serviceName);
                    return null;
                }
                //用来回调onCarDisconnected
                mServiceMap.put(serviceName, manager);
            } catch (RemoteException e) {
                throw e.rethrowFromSystemServer();
            }
        }
    }
    return manager;
}
private CarManagerBase createCarManager(String serviceName, IBinder binder) {
    CarManagerBase manager = null;
    switch (serviceName) {
        case AUDIO_SERVICE:
            manager = new CarAudioManager(binder, mContext, mEventHandler);
            break;
    }
}
private void tearDownCarManagers() {
    synchronized (mCarManagerLock) {
        for (CarManagerBase manager: mServiceMap.values()) {
            //回调所有的CarManagerBase
            manager.onCarDisconnected();
        }
        mServiceMap.clear();
    }
}
```



## 三、内部服务CarAudioService

### 3.1CarAudioService总述



### 3.2CarAudioService初始化

CarAudioService构造函数

```java
    public CarAudioService(Context context) {
        mContext = context;
        //用于读取通话状态
        mTelephonyManager = (TelephonyManager) mContext.getSystemService(Context.TELEPHONY_SERVICE);
        //CarAudioManger音量控制最后调用到AudioManager
        mAudioManager = (AudioManager) mContext.getSystemService(Context.AUDIO_SERVICE);
        //从配置里读取是否支持动态路由
        mUseDynamicRouting = mContext.getResources().getBoolean(R.bool.audioUseDynamicRouting);
        //是否保持主静音状态
        mPersistMasterMuteState = mContext.getResources().getBoolean(
                R.bool.audioPersistMasterMuteState);
        //uid 到 zoneId的映射map表
        mUidToZoneMap = new HashMap<>();
    }
```

CarAudioService实现了CarServiceBase的init、release、dump三个接口、vehicleHalReconnected、dumpMetrics空方法的重写。CarAudioManagerService在ICarImpl构造函数创建，init里初始化

```java
public ICarImpl(Context serviceContext, IVehicle vehicle, SystemInterface systemInterface,
            CanBusErrorNotifier errorNotifier, String vehicleInterfaceName) {
    //创建CarAudioService        
    mCarAudioService = new CarAudioService(serviceContext);
    mSystemStateControllerService = new SystemStateControllerService(
        serviceContext, mCarAudioService, this);
    
    allServices.add(mCarAudioService);
    mAllServices = allServices.toArray(new CarServiceBase[allServices.size()]);
}
    @MainThread
    void init() {
        mBootTiming = new TimingsTraceLog(VHAL_TIMING_TAG, Trace.TRACE_TAG_HAL);
        mHal.init();
        for (CarServiceBase service : mAllServices) {
            //CarAudioService 初始化
            service.init();
        }
        mSystemInterface.reconfigureSecondaryDisplays();
    }
```

CarAudioService的init方法

```java
@Override
    public void init() {
        synchronized (mImplLock) {
            if (mUseDynamicRouting) {
                // Enumerate all output bus device ports
                // 获取所有的output device ports
                AudioDeviceInfo[] deviceInfos = mAudioManager.getDevices(
                        AudioManager.GET_DEVICES_OUTPUTS);
                if (deviceInfos.length == 0) {
                    Log.e(CarLog.TAG_AUDIO, "No output device available, ignore");
                    return;
                }
                SparseArray<CarAudioDeviceInfo> busToCarAudioDeviceInfo = new SparseArray<>();
                for (AudioDeviceInfo info : deviceInfos) {
                    Log.v(CarLog.TAG_AUDIO, String.format("output id=%d address=%s type=%s",
                            info.getId(), info.getAddress(), info.getType()));
                    //枚举Type bus 类型 创建 CarAudioDeviceInfo 整合放入busNumber和carInfo的数字映射里
                    if (info.getType() == AudioDeviceInfo.TYPE_BUS) {
                        final CarAudioDeviceInfo carInfo = new CarAudioDeviceInfo(info);
                        // See also the audio_policy_configuration.xml,
                        // the bus number should be no less than zero.
                        if (carInfo.getBusNumber() >= 0) {
                            busToCarAudioDeviceInfo.put(carInfo.getBusNumber(), carInfo);
                            Log.i(CarLog.TAG_AUDIO, "Valid bus found " + carInfo);
                        }
                    }
                }
                // 根据 busToCarAudioDeviceInfo设置动态路由
                setupDynamicRouting(busToCarAudioDeviceInfo);
            } else {
                Log.i(CarLog.TAG_AUDIO, "Audio dynamic routing not enabled, run in legacy mode");
                //不支持就设置传统音量监听
                setupLegacyVolumeChangedListener();
            }

            // Restore master mute state if applicable
            if (mPersistMasterMuteState) {
                boolean storedMasterMute = Settings.Global.getInt(mContext.getContentResolver(),
                        VOLUME_SETTINGS_KEY_MASTER_MUTE, 0) != 0;
                setMasterMute(storedMasterMute, 0);
            }
        }
    }
```

### 3.3设置动态路由setupDynamicRouting

1.动态路由加载这个两个文件  "/vendor/etc/car_audio_configuration.xml", "/system/etc/car_audio_configuration.xml"

2.如果都没有就是传统模式，加载/packages/services/Car/service/res/xml/car_volume_groups.xml

最后都是一个CarAudioZone的组，CarAudioZone[] mCarAudioZones;

CarAudioZone通过getVolumeGroups就能获取所有的CarVolumeGroup,用于查询

```java
private void setupDynamicRouting(SparseArray<CarAudioDeviceInfo> busToCarAudioDeviceInfo) {
        final AudioPolicy.Builder builder = new AudioPolicy.Builder(mContext);
        builder.setLooper(Looper.getMainLooper());
/*
            "/vendor/etc/car_audio_configuration.xml",
            "/system/etc/car_audio_configuration.xml"
            getAudioConfigurationPath 优先选第一个
*/
        mCarAudioConfigurationPath = getAudioConfigurationPath();
        if (mCarAudioConfigurationPath != null) {
            try (InputStream inputStream = new FileInputStream(mCarAudioConfigurationPath)) {
                CarAudioZonesHelper zonesHelper = new CarAudioZonesHelper(mContext, inputStream,
                        busToCarAudioDeviceInfo);
                mCarAudioZones = zonesHelper.loadAudioZones();
            } catch (IOException | XmlPullParserException e) {
                throw new RuntimeException("Failed to parse audio zone configuration", e);
            }
        } else {
            // In legacy mode, context -> bus mapping is done by querying IAudioControl HAL.
            final IAudioControl audioControl = getAudioControl();
            if (audioControl == null) {
                throw new RuntimeException(
                        "Dynamic routing requested but audioControl HAL not available");
            }
            //传统模式使用 R.xml.car_volume_groups里的配置
            CarAudioZonesHelperLegacy legacyHelper = new CarAudioZonesHelperLegacy(mContext,
                    R.xml.car_volume_groups, busToCarAudioDeviceInfo, audioControl);
            mCarAudioZones = legacyHelper.loadAudioZones();
        }
        for (CarAudioZone zone : mCarAudioZones) {
            if (!zone.validateVolumeGroups()) {
                throw new RuntimeException("Invalid volume groups configuration");
            }
            // Ensure HAL gets our initial value
            zone.synchronizeCurrentGainIndex();
            /*  get当前音量再设置回去
                void synchronizeCurrentGainIndex() {
                    for (CarVolumeGroup group : mVolumeGroups) {
                        group.setCurrentGainIndex(group.getCurrentGainIndex());
                    }
                }
            */
            Log.v(CarLog.TAG_AUDIO, "Processed audio zone: " + zone);
        }

        // Setup dynamic routing rules by usage
        final CarAudioDynamicRouting dynamicRouting = new CarAudioDynamicRouting(mCarAudioZones);
        // 传入AudioPolicy builder
        /* 遍历zones
           	遍历groups
        		遍历group里的busNumber，得到mixFormat 从CarAudioDeviceInfo
        		创建mixingRuleBuilder
        			遍历bus对应的context，
        				遍历context对应的usage，创建新的AudioAttributes setUsage，
        				并将这个AudioAttributes addRule到mixingRuleBuilder里
   				根据mixFormat和mixingRuleBuilder创建audioMix，在传入的AudioPolicy builder里addMix
    
        */
    	dynamicRouting.setupAudioDynamicRouting(builder);

        // Attach the {@link AudioPolicyVolumeCallback}
    	//设置音量回调监听 onVolumeAdjustment
    	/* AudioManager.ADJUST_LOWER  AudioManager.ADJUST_RAISE
    	   AudioManager.ADJUST_MUTE   AudioManager.ADJUST_UNMUTE
    	   AudioManager.ADJUST_TOGGLE_MUTE AudioManager.ADJUST_SAME
    	*/
        builder.setAudioPolicyVolumeCallback(mAudioPolicyVolumeCallback);
	   //使用CarAudioFocus
        if (sUseCarAudioFocus) {
            // Configure our AudioPolicy to handle focus events.
            // This gives us the ability to decide which audio focus requests to accept and bypasses
            // the framework ducking logic.
            mFocusHandler = new CarZonesAudioFocus(mAudioManager,
                    mContext.getPackageManager(),
                    mCarAudioZones);
            //mFocusHandler 是继承AudioPolicy.AudioPolicyFocusListener的CarZonesAudioFocus实例
            //即从来接管该AudioPolicy的AudioFocus的策略制定
            builder.setAudioPolicyFocusListener(mFocusHandler);
            builder.setIsAudioFocusPolicy(true);
        }

        mAudioPolicy = builder.build();
        if (sUseCarAudioFocus) {
            // Connect the AudioPolicy and the focus listener
            mFocusHandler.setOwningPolicy(this, mAudioPolicy);
        }
	    //将该AudioPolicy注册到AudioManager里
        int r = mAudioManager.registerAudioPolicy(mAudioPolicy);
        if (r != AudioManager.SUCCESS) {
            throw new RuntimeException("registerAudioPolicy failed " + r);
        }
    }
```

### 3.4配置文件

#### 3.4.1car_audio_configuration.xml 

动态路由的配置

源码中的位置/device/generic/car/emulator/audio/car_audio_configuration.xml

```xml
<?xml version="1.0" encoding="utf-8"?>
<!--
  Defines the audio configuration in a car, including
    - Audio zones
    - Display to audio zone mappings
    - Context to audio bus mappings
    - Volume groups
  in the car environment.
-->
<carAudioConfiguration version="1">
    <zones>
        <zone name="primary zone" isPrimary="true">
            <volumeGroups>
                <group>
                    <device address="bus0_media_out">
                        <context context="music"/>
                    </device>
                    <device address="bus3_call_ring_out">
                        <context context="call_ring"/>
                    </device>
                    <device address="bus6_notification_out">
                        <context context="notification"/>
                    </device>
                    <device address="bus7_system_sound_out">
                        <context context="system_sound"/>
                    </device>
                </group>
                <group>
                    <device address="bus1_navigation_out">
                        <context context="navigation"/>
                    </device>
                    <device address="bus2_voice_command_out">
                        <context context="voice_command"/>
                    </device>
                </group>
                <group>
                    <device address="bus4_call_out">
                        <context context="call"/>
                    </device>
                </group>
                <group>
                    <device address="bus5_alarm_out">
                        <context context="alarm"/>
                    </device>
                </group>
            </volumeGroups>
            <displays>
                <display port="0"/>
            </displays>
            <!-- to specify displays associated with this audio zone, use the following tags
                <displays>
                    <display port="1"/>
                    <display port="2"/>
                </displays>
                where port is the physical port of the display (See DisplayAddress.Phyisical)
            -->
        </zone>
        <zone name="rear seat zone">
            <volumeGroups>
                <group>
                    <device address="bus100_rear_seat">
                        <context context="music"/>
                        <context context="navigation"/>
                        <context context="voice_command"/>
                        <context context="call_ring"/>
                        <context context="call"/>
                        <context context="alarm"/>
                        <context context="notification"/>
                        <context context="system_sound"/>
                    </device>
                </group>
            </volumeGroups>
            <displays>
                <display port="1"/>
            </displays>
        </zone>
    </zones>
</carAudioConfiguration>
```

参考/device/generic/car/emulator/audio/car_audio_configuration.xml里的配置

主要分为通用和后座，通用里分了4组媒体，导航语音，通话，闹钟。后座只有一个

#### 3.4.2audio_policy_configuration.xml

device列表的 bus对应

路径/device/generic/car/emulator/audio/audio_policy_configuration.xml

```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<audioPolicyConfiguration version="1.0" xmlns:xi="http://www.w3.org/2001/XInclude">
    <!-- version section contains a “version” tag in the form “major.minor” e.g version=”1.0” -->

    <!-- Global configuration Decalaration -->
    <globalConfiguration speaker_drc_enabled="true"/>

    <!-- Modules section:
        There is one section per audio HW module present on the platform.
        Each module section will contains two mandatory tags for audio HAL “halVersion” and “name”.
        The module names are the same as in current .conf file:
                “primary”, “A2DP”, “remote_submix”, “USB”
        Each module will contain the following sections:
        “devicePorts”: a list of device descriptors for all input and output devices accessible via this
        module.
        This contains both permanently attached devices and removable devices.
            "gain": constraints applied to the millibel values:
                - maxValueMB >= minValueMB
                - defaultValueMB >= minValueMB && defaultValueMB <= maxValueMB
                - (maxValueMB - minValueMB) % stepValueMB == 0
                - (defaultValueMB - minValueMB) % stepValueMB == 0
        “mixPorts”: listing all output and input streams exposed by the audio HAL
        “routes”: list of possible connections between input and output devices or between stream and
        devices.
            "route": is defined by an attribute:
                -"type": <mux|mix> means all sources are mutual exclusive (mux) or can be mixed (mix)
                -"sink": the sink involved in this route
                -"sources": all the sources than can be connected to the sink via vis route
        “attachedDevices”: permanently attached devices.
        The attachedDevices section is a list of devices names. The names correspond to device names
        defined in <devicePorts> section.
        “defaultOutputDevice”: device to be used by default when no policy rule applies
    -->
    <modules>
        <!-- Primary Audio HAL -->
        <module name="primary" halVersion="3.0">
            <attachedDevices>
                <!-- One bus per context -->
                <item>bus0_media_out</item>
                <item>bus1_navigation_out</item>
                <item>bus2_voice_command_out</item>
                <item>bus3_call_ring_out</item>
                <item>bus4_call_out</item>
                <item>bus5_alarm_out</item>
                <item>bus6_notification_out</item>
                <item>bus7_system_sound_out</item>
                <item>bus100_rear_seat</item>
                <item>Built-In Mic</item>
                <item>Built-In Back Mic</item>
                <item>Echo-Reference Mic</item>
                <item>FM Tuner</item>
            </attachedDevices>
            <defaultOutputDevice>bus0_media_out</defaultOutputDevice>
            <mixPorts>
                <mixPort name="mixport_bus0_media_out" role="source"
                        flags="AUDIO_OUTPUT_FLAG_PRIMARY">
                    <profile name="" format="AUDIO_FORMAT_PCM_16_BIT"
                             samplingRates="48000"
                             channelMasks="AUDIO_CHANNEL_OUT_STEREO"/>
                </mixPort>
                <mixPort name="mixport_bus1_navigation_out" role="source"
                        flags="AUDIO_OUTPUT_FLAG_PRIMARY">
                    <profile name="" format="AUDIO_FORMAT_PCM_16_BIT"
                             samplingRates="48000"
                             channelMasks="AUDIO_CHANNEL_OUT_STEREO"/>
                </mixPort>
                <mixPort name="mixport_bus2_voice_command_out" role="source"
                        flags="AUDIO_OUTPUT_FLAG_PRIMARY">
                    <profile name="" format="AUDIO_FORMAT_PCM_16_BIT"
                             samplingRates="48000"
                             channelMasks="AUDIO_CHANNEL_OUT_STEREO"/>
                </mixPort>
                <mixPort name="mixport_bus3_call_ring_out" role="source"
                        flags="AUDIO_OUTPUT_FLAG_PRIMARY">
                    <profile name="" format="AUDIO_FORMAT_PCM_16_BIT"
                             samplingRates="48000"
                             channelMasks="AUDIO_CHANNEL_OUT_STEREO"/>
                </mixPort>
                <mixPort name="mixport_bus4_call_out" role="source"
                        flags="AUDIO_OUTPUT_FLAG_PRIMARY">
                    <profile name="" format="AUDIO_FORMAT_PCM_16_BIT"
                             samplingRates="48000"
                             channelMasks="AUDIO_CHANNEL_OUT_STEREO"/>
                </mixPort>
                <mixPort name="mixport_bus5_alarm_out" role="source"
                        flags="AUDIO_OUTPUT_FLAG_PRIMARY">
                    <profile name="" format="AUDIO_FORMAT_PCM_16_BIT"
                             samplingRates="48000"
                             channelMasks="AUDIO_CHANNEL_OUT_STEREO"/>
                </mixPort>
                <mixPort name="mixport_bus6_notification_out" role="source"
                        flags="AUDIO_OUTPUT_FLAG_PRIMARY">
                    <profile name="" format="AUDIO_FORMAT_PCM_16_BIT"
                             samplingRates="48000"
                             channelMasks="AUDIO_CHANNEL_OUT_STEREO"/>
                </mixPort>
                <mixPort name="mixport_bus7_system_sound_out" role="source"
                        flags="AUDIO_OUTPUT_FLAG_PRIMARY">
                    <profile name="" format="AUDIO_FORMAT_PCM_16_BIT"
                             samplingRates="48000"
                             channelMasks="AUDIO_CHANNEL_OUT_STEREO"/>
                </mixPort>
                <mixPort name="mixport_bus100_rear_seat" role="source"
                        flags="AUDIO_OUTPUT_FLAG_PRIMARY">
                    <profile name="" format="AUDIO_FORMAT_PCM_16_BIT"
                             samplingRates="48000"
                             channelMasks="AUDIO_CHANNEL_OUT_STEREO"/>
                </mixPort>
                <mixPort name="primary input" role="sink">
                    <profile name="" format="AUDIO_FORMAT_PCM_16_BIT"
                             samplingRates="8000,11025,12000,16000,22050,24000,32000,44100,48000"
                             channelMasks="AUDIO_CHANNEL_IN_MONO,AUDIO_CHANNEL_IN_STEREO,AUDIO_CHANNEL_IN_FRONT_BACK"/>
                </mixPort>
                <mixPort name="mixport_tuner0" role="sink">
                    <profile name="" format="AUDIO_FORMAT_PCM_16_BIT"
                             samplingRates="48000"
                             channelMasks="AUDIO_CHANNEL_IN_STEREO"/>
                </mixPort>
            </mixPorts>
            <devicePorts>
                <devicePort tagName="bus0_media_out" role="sink" type="AUDIO_DEVICE_OUT_BUS"
                        address="bus0_media_out">
                    <profile name="" format="AUDIO_FORMAT_PCM_16_BIT"
                            samplingRates="48000" channelMasks="AUDIO_CHANNEL_OUT_STEREO"/>
                    <gains>
                        <gain name="" mode="AUDIO_GAIN_MODE_JOINT"
                                minValueMB="-3200" maxValueMB="600" defaultValueMB="0" stepValueMB="100"/>
                    </gains>
                </devicePort>
                <devicePort tagName="bus1_navigation_out" role="sink" type="AUDIO_DEVICE_OUT_BUS"
                        address="bus1_navigation_out">
                    <profile name="" format="AUDIO_FORMAT_PCM_16_BIT"
                            samplingRates="48000" channelMasks="AUDIO_CHANNEL_OUT_STEREO"/>
                    <gains>
                        <gain name="" mode="AUDIO_GAIN_MODE_JOINT"
                                minValueMB="-3200" maxValueMB="600" defaultValueMB="0" stepValueMB="100"/>
                    </gains>
                </devicePort>
                <devicePort tagName="bus2_voice_command_out" role="sink" type="AUDIO_DEVICE_OUT_BUS"
                        address="bus2_voice_command_out">
                    <profile name="" format="AUDIO_FORMAT_PCM_16_BIT"
                            samplingRates="48000" channelMasks="AUDIO_CHANNEL_OUT_STEREO"/>
                    <gains>
                        <gain name="" mode="AUDIO_GAIN_MODE_JOINT"
                                minValueMB="-3200" maxValueMB="600" defaultValueMB="0" stepValueMB="100"/>
                    </gains>
                </devicePort>
                <devicePort tagName="bus3_call_ring_out" role="sink" type="AUDIO_DEVICE_OUT_BUS"
                        address="bus3_call_ring_out">
                    <profile name="" format="AUDIO_FORMAT_PCM_16_BIT"
                            samplingRates="48000" channelMasks="AUDIO_CHANNEL_OUT_STEREO"/>
                    <gains>
                        <gain name="" mode="AUDIO_GAIN_MODE_JOINT"
                                minValueMB="-3200" maxValueMB="600" defaultValueMB="0" stepValueMB="100"/>
                    </gains>
                </devicePort>
                <devicePort tagName="bus4_call_out" role="sink" type="AUDIO_DEVICE_OUT_BUS"
                        address="bus4_call_out">
                    <profile name="" format="AUDIO_FORMAT_PCM_16_BIT"
                            samplingRates="48000" channelMasks="AUDIO_CHANNEL_OUT_STEREO"/>
                    <gains>
                        <gain name="" mode="AUDIO_GAIN_MODE_JOINT"
                                minValueMB="-3200" maxValueMB="600" defaultValueMB="0" stepValueMB="100"/>
                    </gains>
                </devicePort>
                <devicePort tagName="bus5_alarm_out" role="sink" type="AUDIO_DEVICE_OUT_BUS"
                        address="bus5_alarm_out">
                    <profile name="" format="AUDIO_FORMAT_PCM_16_BIT"
                            samplingRates="48000" channelMasks="AUDIO_CHANNEL_OUT_STEREO"/>
                    <gains>
                        <gain name="" mode="AUDIO_GAIN_MODE_JOINT"
                                minValueMB="-3200" maxValueMB="600" defaultValueMB="0" stepValueMB="100"/>
                    </gains>
                </devicePort>
                <devicePort tagName="bus6_notification_out" role="sink" type="AUDIO_DEVICE_OUT_BUS"
                        address="bus6_notification_out">
                    <profile name="" format="AUDIO_FORMAT_PCM_16_BIT"
                            samplingRates="48000" channelMasks="AUDIO_CHANNEL_OUT_STEREO"/>
                    <gains>
                        <gain name="" mode="AUDIO_GAIN_MODE_JOINT"
                                minValueMB="-3200" maxValueMB="600" defaultValueMB="0" stepValueMB="100"/>
                    </gains>
                </devicePort>
                <devicePort tagName="bus7_system_sound_out" role="sink" type="AUDIO_DEVICE_OUT_BUS"
                        address="bus7_system_sound_out">
                    <profile name="" format="AUDIO_FORMAT_PCM_16_BIT"
                            samplingRates="48000" channelMasks="AUDIO_CHANNEL_OUT_STEREO"/>
                    <gains>
                        <gain name="" mode="AUDIO_GAIN_MODE_JOINT"
                                minValueMB="-3200" maxValueMB="600" defaultValueMB="0" stepValueMB="100"/>
                    </gains>
                </devicePort>
                <devicePort tagName="bus100_rear_seat" role="sink" type="AUDIO_DEVICE_OUT_BUS"
                        address="bus100_rear_seat">
                    <profile name="" format="AUDIO_FORMAT_PCM_16_BIT"
                            samplingRates="48000" channelMasks="AUDIO_CHANNEL_OUT_STEREO"/>
                    <gains>
                        <gain name="" mode="AUDIO_GAIN_MODE_JOINT"
                                minValueMB="-3200" maxValueMB="600" defaultValueMB="0" stepValueMB="100"/>
                    </gains>
                </devicePort>
                <devicePort tagName="Built-In Mic" type="AUDIO_DEVICE_IN_BUILTIN_MIC" role="source">
                    <profile name="" format="AUDIO_FORMAT_PCM_16_BIT"
                            samplingRates="8000,11025,12000,16000,22050,24000,32000,44100,48000"
                            channelMasks="AUDIO_CHANNEL_IN_MONO,AUDIO_CHANNEL_IN_STEREO,AUDIO_CHANNEL_IN_FRONT_BACK"/>
                </devicePort>
                <devicePort tagName="Built-In Back Mic" type="AUDIO_DEVICE_IN_BACK_MIC" role="source">
                    <profile name="" format="AUDIO_FORMAT_PCM_16_BIT"
                            samplingRates="8000,11025,12000,16000,22050,24000,32000,44100,48000"
                            channelMasks="AUDIO_CHANNEL_IN_MONO,AUDIO_CHANNEL_IN_STEREO,AUDIO_CHANNEL_IN_FRONT_BACK"/>
                </devicePort>
                <devicePort tagName="Echo-Reference Mic" type="AUDIO_DEVICE_IN_ECHO_REFERENCE" role="source">
                    <profile name="" format="AUDIO_FORMAT_PCM_16_BIT"
                            samplingRates="8000,11025,12000,16000,22050,24000,32000,44100,48000"
                            channelMasks="AUDIO_CHANNEL_IN_MONO,AUDIO_CHANNEL_IN_STEREO,AUDIO_CHANNEL_IN_FRONT_BACK"/>
                </devicePort>
                <devicePort tagName="FM Tuner" type="AUDIO_DEVICE_IN_FM_TUNER" role="source"
                        address="tuner0">
                    <profile name="" format="AUDIO_FORMAT_PCM_16_BIT"
                            samplingRates="48000" channelMasks="AUDIO_CHANNEL_IN_STEREO"/>
                    <gains>
                        <gain name="" mode="AUDIO_GAIN_MODE_JOINT"
                                minValueMB="-3200" maxValueMB="600" defaultValueMB="0" stepValueMB="100"/>
                    </gains>
                </devicePort>
            </devicePorts>
            <!-- route declaration, i.e. list all available sources for a given sink -->
            <routes>
                <route type="mix" sink="bus0_media_out" sources="mixport_bus0_media_out"/>
                <route type="mix" sink="bus1_navigation_out" sources="mixport_bus1_navigation_out"/>
                <route type="mix" sink="bus2_voice_command_out" sources="mixport_bus2_voice_command_out"/>
                <route type="mix" sink="bus3_call_ring_out" sources="mixport_bus3_call_ring_out"/>
                <route type="mix" sink="bus4_call_out" sources="mixport_bus4_call_out"/>
                <route type="mix" sink="bus5_alarm_out" sources="mixport_bus5_alarm_out"/>
                <route type="mix" sink="bus6_notification_out" sources="mixport_bus6_notification_out"/>
                <route type="mix" sink="bus7_system_sound_out" sources="mixport_bus7_system_sound_out"/>
                <route type="mix" sink="bus100_rear_seat" sources="mixport_bus100_rear_seat"/>
                <route type="mix" sink="primary input" sources="Built-In Mic,Built-In Back Mic,Echo-Reference Mic"/>
                <route type="mix" sink="mixport_tuner0" sources="FM Tuner"/>
            </routes>

        </module>

        <!-- A2dp Audio HAL -->
        <xi:include href="a2dp_audio_policy_configuration.xml"/>

        <!-- Usb Audio HAL -->
        <xi:include href="usb_audio_policy_configuration.xml"/>

        <!-- Remote Submix Audio HAL -->
        <xi:include href="r_submix_audio_policy_configuration.xml"/>

    </modules>
    <!-- End of Modules section -->

    <!-- Volume section -->

    <xi:include href="audio_policy_volumes.xml"/>
    <xi:include href="default_volume_tables.xml"/>

    <!-- End of Volume section -->
    <!-- End of Modules section -->

</audioPolicyConfiguration>
```

#### 3.4.3car_volume_group.xml

不支持动态路由的使用这个配置

路径/packages/services/Car/service/res/xml/car_volume_groups.xml

谷歌建议使用新的

```xml
<!--
  This configuration is replaced by car_audio_configuration.xml

  Notes on backward compatibility
  - A new audioUseUnifiedConfiguration flag is added, and the default value
  is false
  - If OEM does not explicitly set audioUseUnifiedConfiguration to be true,
  car_volume_groups.xml will be used, CarAudioService also queries
  IAudioControl HAL (getBusForContext)
  - Otherwise, CarAudioService loads the new car_audio_configuration.xml

  ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

  Defines the all available volume groups for volume control in a car.
  One can overlay this configuration to customize the groups.

  This configuration will be populated by CarAudioService and
  surfaced to Car Settings App and/or other volume control interfaces.

  Certain constraints applied to this configuration
    - One context should not appear in two groups
    - All contexts are assigned
    - One bus should not appear in two groups
    - All gain controllers (set on each bus) in one group have same step value

  It is fine that there are buses that do not appear in any group, those buses
  may be reserved for other usages.

  Important note: when overlaying this configuration,
  make sure the resources are in the same package as CarAudioService.
-->
<volumeGroups xmlns:car="http://schemas.android.com/apk/res-auto"
        car:isDeprecated="true">
    <group>
        <context car:context="music"/>
        <context car:context="call_ring"/>
        <context car:context="notification"/>
        <context car:context="system_sound"/>
    </group>
    <group>
        <context car:context="navigation"/>
        <context car:context="voice_command"/>
    </group>
    <group>
        <context car:context="call"/>
    </group>
    <group>
        <context car:context="alarm"/>
    </group>
</volumeGroups>
```



### 3.5CarAudioService功能调用

CarAudioService extends ICarAudio.Stub

CarAudioManager提供的API，实际是对CarAudioServive的跨进程调用

需要Car.PERMISSION_CAR_CONTROL_AUDIO_VOLUME 权限

#### 3.5.1groupVolume相关

```java
    void setGroupVolume(int zoneId, int groupId, int index, int flags);
    int getGroupMaxVolume(int zoneId, int groupId);
    int getGroupMinVolume(int zoneId, int groupId);
    int getGroupVolume(int zoneId, int groupId); 
    int getVolumeGroupCount(int zoneId);
    int getVolumeGroupIdForUsage(int zoneId, int usage);
    int[] getUsagesForVolumeGroupId(int zoneId, int groupId);
```

**groupId的获取**

通过getVolumeGroupIdForUsage对传入的usage和zoneId返回对应的groupID，简单的遍历查找

传统模式zoneId为primaryID

```java
CarVolumeGroup[] groups = mCarAudioZones[zoneId].getVolumeGroups();
for (int i = 0; i < groups.length; i++) {
    int[] contexts = groups[i].getContexts();
    for (int context : contexts) {
        if (getContextForUsage(usage) == context) {
            return i;
        }
    }
}
```

getVolumeGroupCount(int zoneId)

```java
mCarAudioZones[zoneId].getVolumeGroupCount();
```

getUsagesForVolumeGroupId(int zoneId, int groupId)

1传统模式

```java
new int[] { CarAudioDynamicRouting.STREAM_TYPE_USAGES[groupId] };
```

2支持动态路由

通过CarVolumeGroup group = getCarVolumeGroup(zoneId, groupId);得到volumeGroup

group->contexts->usages查询

**组音量设置和获取**需要动态路由feature的判定

不支持动态路由即传统模式是调用AudioManger的相关参数,将groupId转化成Stream

```java
    static final int[] STREAM_TYPES = new int[] {
            AudioManager.STREAM_MUSIC,
            AudioManager.STREAM_ALARM,
            AudioManager.STREAM_RING
    };

mAudioManager.setStreamVolume(CarAudioDynamicRouting.STREAM_TYPES[groupId], index, flags);
 
mAudioManager.getStreamMaxVolume(CarAudioDynamicRouting.STREAM_TYPES[groupId]);
                        
mAudioManager.getStreamMinVolume(CarAudioDynamicRouting.STREAM_TYPES[groupId]);

mAudioManager.getStreamVolume(CarAudioDynamicRouting.STREAM_TYPES[groupId]);
```

支持动态路由的，首先通过getCarVolumeGroup(zoneId, groupId)获取对应的group,然后在对对应的group进行设置

```java
    private CarVolumeGroup getCarVolumeGroup(int zoneId, int groupId) {
        Preconditions.checkNotNull(mCarAudioZones);
        Preconditions.checkArgumentInRange(zoneId, 0, mCarAudioZones.length - 1,
                "zoneId out of range: " + zoneId);
        return mCarAudioZones[zoneId].getVolumeGroup(groupId);
    }
            CarVolumeGroup group = getCarVolumeGroup(zoneId, groupId);
            group.setCurrentGainIndex(index);

            CarVolumeGroup group = getCarVolumeGroup(zoneId, groupId);
            return group.getMaxGainIndex();

            CarVolumeGroup group = getCarVolumeGroup(zoneId, groupId);
            return group.getMinGainIndex();

            CarVolumeGroup group = getCarVolumeGroup(zoneId, groupId);
            return group.getCurrentGainIndex();
```



#### 3.5.2zoneId相关

```java
    int[] getAudioZoneIds();
    int getZoneIdForUid(int uid);
    boolean setZoneIdForUid(int zoneId, int uid);
    boolean clearZoneIdForUid(int uid);
    int getZoneIdForDisplayPortId(byte displayPortId);
```

1 getAudioZoneIds是将mCarAudioZones里ZoneId整合成数组返回

zoneId和Uid的联动，分配AudioFocus

2 setZoneIdForUid、getZoneIdForUid和clearZoneIdForUid就是对mUidToZoneMap查询维护，然后进行音频焦点AudioFocus的管理，详见四CarAudioFocus

3 getZoneIdForDisplayPortId是通过遍历mCarAudioZones，对比查询PhysicalDisplayAddresses

```java
CarAudioZone zone = mCarAudioZones[index];
List<DisplayAddress.Physical> displayAddresses = zone.getPhysicalDisplayAddresses();
```



#### 3.5.3前后左右平衡

```java
    void setFadeTowardFront(float value);
    void setBalanceTowardRight(float value);
```

这两个功能是调用的hal层的AudioControl，具体实现在hal层以下

```java
final IAudioControl audioControlHal = getAudioControl();
audioControlHal.setFadeTowardFront(value);

audioControlHal.setBalanceTowardRight(value);
```



#### 3.5.4外部音源相关

```java
    String[] getExternalSources();
    CarAudioPatchHandle createAudioPatch(in String sourceAddress, int usage, int gainInMillibels);
    void releaseAudioPatch(in CarAudioPatchHandle patch);
```

通过getExternalSources得到外部音源的sourceAddress，然后通过createAudioPatch进行创建补丁AudioPatch得到CarAudioPatchHandle，CarAudioPatchHandle用于releaseAudioPatch

详见五、外部音源AudioPatch（创建过程中会根据usage对应group的当前音量进行设置，）

#### 3.5.5音量监听相关

```java
    void registerVolumeCallback(in IBinder binder);
    void unregisterVolumeCallback(in IBinder binder);
```

AudioService里有一个mVolumeCallbackContainer的集合register和unregister都是对这个集合的操作

```java
mVolumeCallbackContainer.addBinder(ICarVolumeCallback.Stub.asInterface(binder));
mVolumeCallbackContainer.removeBinder(ICarVolumeCallback.Stub.asInterface(binder));
```

**传统模式的音量监听回调是在init方法，通过广播监听音量变化，然后循环遍历binder集合进行回调**

```java
public void init() {
	if (mUseDynamicRouting) {
    }else{
        setupLegacyVolumeChangedListener();
    }
}
private void setupLegacyVolumeChangedListener() {
    IntentFilter intentFilter = new IntentFilter();
    intentFilter.addAction(AudioManager.VOLUME_CHANGED_ACTION);
    intentFilter.addAction(AudioManager.MASTER_MUTE_CHANGED_ACTION);
    mContext.registerReceiver(mLegacyVolumeChangedReceiver, intentFilter);
}
private final BroadcastReceiver mLegacyVolumeChangedReceiver = new BroadcastReceiver() {
    @Override
    public void onReceive(Context context, Intent intent) {
        final int zoneId = CarAudioManager.PRIMARY_AUDIO_ZONE;
        switch (intent.getAction()) {
            case AudioManager.VOLUME_CHANGED_ACTION:
                int streamType = intent.getIntExtra(AudioManager.EXTRA_VOLUME_STREAM_TYPE, -1);
                int groupId = getVolumeGroupIdForStreamType(streamType);
                if (groupId == -1) {
                    Log.w(CarLog.TAG_AUDIO, "Unknown stream type: " + streamType);
                } else {
                    callbackGroupVolumeChange(zoneId, groupId, 0);
                }
                break;
            case AudioManager.MASTER_MUTE_CHANGED_ACTION:
                callbackMasterMuteChange(zoneId, 0);
                break;
        }
    }
};
private void callbackGroupVolumeChange(int zoneId, int groupId, int flags) {
    for (BinderInterfaceContainer.BinderInterface<ICarVolumeCallback> callback :
         mVolumeCallbackContainer.getInterfaces()) {
        try {
            callback.binderInterface.onGroupVolumeChanged(zoneId, groupId, flags);
        } catch (RemoteException e) {
            Log.e(CarLog.TAG_AUDIO, "Failed to callback onGroupVolumeChanged", e);
        }
    }
}
//callbackMasterMuteChange 同理
```

**动态路由的音量监听是在setupDynamicRouting里，通过AudioPolicy的builder设置的mAudioPolicyVolumeCallback，之后再循环遍历其他监听binder集合**

```java
private void setupDynamicRouting(SparseArray<CarAudioDeviceInfo> busToCarAudioDeviceInfo) {
        final AudioPolicy.Builder builder = new AudioPolicy.Builder(mContext);
        builder.setAudioPolicyVolumeCallback(mAudioPolicyVolumeCallback);
}
    private final AudioPolicy.AudioPolicyVolumeCallback mAudioPolicyVolumeCallback =
            new AudioPolicy.AudioPolicyVolumeCallback() {
        @Override
        public void onVolumeAdjustment(int adjustment) {
            final int usage = getSuggestedAudioUsage();
            Log.v(CarLog.TAG_AUDIO,
                    "onVolumeAdjustment: " + AudioManager.adjustToString(adjustment)
                            + " suggested usage: " + AudioAttributes.usageToString(usage));
            // TODO: Pass zone id into this callback.
            final int zoneId = CarAudioManager.PRIMARY_AUDIO_ZONE;
            final int groupId = getVolumeGroupIdForUsage(zoneId, usage);
            final int currentVolume = getGroupVolume(zoneId, groupId);
            final int flags = AudioManager.FLAG_FROM_KEY | AudioManager.FLAG_SHOW_UI;
            switch (adjustment) {
                case AudioManager.ADJUST_LOWER:
                    int minValue = Math.max(currentVolume - 1, getGroupMinVolume(zoneId, groupId));
                    setGroupVolume(zoneId, groupId, minValue , flags);
                    break;
                case AudioManager.ADJUST_RAISE:
                    int maxValue =  Math.min(currentVolume + 1, getGroupMaxVolume(zoneId, groupId));
                    setGroupVolume(zoneId, groupId, maxValue, flags);
                    break;
                case AudioManager.ADJUST_MUTE:
                    setMasterMute(true, flags);
                    callbackMasterMuteChange(zoneId, flags);
                    break;
                case AudioManager.ADJUST_UNMUTE:
                    setMasterMute(false, flags);
                    callbackMasterMuteChange(zoneId, flags);
                    break;
                case AudioManager.ADJUST_TOGGLE_MUTE:
                    setMasterMute(!mAudioManager.isMasterMute(), flags);
                    callbackMasterMuteChange(zoneId, flags);
                    break;
                case AudioManager.ADJUST_SAME:
                default:
                    break;
            }
        }
    };
```

## 四、音频焦点AudioFocus

车辆音频焦点是Q上新特性，涉及到的文件有

CarAudioFocus.java CarZonesAudioFocus.java

### 4.1CarAudioFocus的初始化

在3.3 设置动态路由时有一步是，设置AudioFocus，得到CarZonesAudioFocus设置进AudioPolicy的builder里，并将AudioPolicy设置到mFocusHandler里，双向设置。

```java
        if (sUseCarAudioFocus) {
            // Configure our AudioPolicy to handle focus events.
            // This gives us the ability to decide which audio focus requests to accept and bypasses
            // the framework ducking logic.
            mFocusHandler = new CarZonesAudioFocus(mAudioManager,
                    mContext.getPackageManager(),
                    mCarAudioZones);
            builder.setAudioPolicyFocusListener(mFocusHandler);
            builder.setIsAudioFocusPolicy(true);
        }

        mAudioPolicy = builder.build();
        if (sUseCarAudioFocus) {
            // Connect the AudioPolicy and the focus listener
            mFocusHandler.setOwningPolicy(this, mAudioPolicy);
        }
```

通过AudioManger PackageManager和CarAudioZones的数组构建CarZonesAudioFocus，在CarZonesAudioFocus里为每一个CarAudioZone都创建了一个CarAudioFocus并存入mFocusZones的map映射表里

 mFocusHandler.setOwningPolicy(this, mAudioPolicy)，就是把Audiopolicy的引用传入到每一个CarAudioFocus里

最后带着CarZonesAudioFocus的回调引用的AudioPolicy通过AudioManager设置到了AudioService里了，可想而知就是截获AudioFocus的回调。

```java
    CarZonesAudioFocus(AudioManager audioManager,
            PackageManager packageManager,
            @NonNull CarAudioZone[] carAudioZones) {
        //Create the zones here, the policy will be set setOwningPolicy,
        // which is called right after this constructor.

        Preconditions.checkNotNull(carAudioZones);
        Preconditions.checkArgument(carAudioZones.length != 0,
                "There must be a minimum of one audio zone");

        //Create focus for all the zones
        for (CarAudioZone audioZone : carAudioZones) {
            Log.d(CarLog.TAG_AUDIO,
                    "CarZonesAudioFocus adding new zone " + audioZone.getId());
            CarAudioFocus zoneFocusListener = new CarAudioFocus(audioManager, packageManager);
            mFocusZones.put(audioZone.getId(), zoneFocusListener);
        }
    }
//CarAudioFocus构造方法里就暂存了AudioManger和PackageManager
    CarAudioFocus(AudioManager audioManager, PackageManager packageManager) {
        mAudioManager = audioManager;
        mPackageManager = packageManager;
    }

 //mFocusHandler.setOwningPolicy(this, mAudioPolicy);
    void setOwningPolicy(CarAudioService audioService, AudioPolicy parentPolicy) {
        mCarAudioService = audioService;
        mAudioPolicy = parentPolicy;

        for (int zoneId : mFocusZones.keySet()) {
            mFocusZones.get(zoneId).setOwningPolicy(mCarAudioService, mAudioPolicy);
        }
    }
//CarAudioFocus.java
    public void setOwningPolicy(CarAudioService audioService, AudioPolicy parentPolicy) {
        mCarAudioService = audioService;
        mAudioPolicy     = parentPolicy;
    }
```

CarZonesAudioFocus继承了AudioPolicyFocusListener，并重写了音频焦点的请求和放弃两个方法，getFocusForAudioFocusInfo就是通过AudioFocusInfo找到准确的zoneId，通过zoneId在mFocusZones去到准确CarAudioFocus，向下分发onAudioFocusRequest和onAudioFocusAbandon的回调

```java
class CarZonesAudioFocus extends AudioPolicy.AudioPolicyFocusListener{
    @Override
    public void onAudioFocusRequest(AudioFocusInfo afi, int requestResult) {
        CarAudioFocus focus = getFocusForAudioFocusInfo(afi);
        focus.onAudioFocusRequest(afi, requestResult);
    }
    @Override
    public void onAudioFocusAbandon(AudioFocusInfo afi) {
        CarAudioFocus focus = getFocusForAudioFocusInfo(afi);
        focus.onAudioFocusAbandon(afi);
    }
    
    private CarAudioFocus getFocusForAudioFocusInfo(AudioFocusInfo afi) {
        //getFocusForAudioFocusInfo defaults to returning default zoneId
        //if uid has not been mapped, thus the value returned will be
        //default zone focus
        //通过UID查找存在CarAudioService里的ZoneId 
        int zoneId = mCarAudioService.getZoneIdForUid(afi.getClientUid());

        // If the bundle attribute for AUDIOFOCUS_EXTRA_REQUEST_ZONE_ID has been assigned
        // Use zone id from that instead.
        Bundle bundle = afi.getAttributes().getBundle();
		//获取AudioFocusInfo里带着的bundleZoneId，判别下优先取bundleZoneId
        if (bundle != null) {
            int bundleZoneId =
                bundle.getInt(CarAudioManager.AUDIOFOCUS_EXTRA_REQUEST_ZONE_ID,
                              -1);
            // check if the zone id is within current zones bounds
            if ((bundleZoneId >= 0)
                && (bundleZoneId < mCarAudioService.getAudioZoneIds().length)) {
                zoneId = bundleZoneId;
            }
        }
		//map里获取focus
        CarAudioFocus focus =  mFocusZones.get(zoneId);
        return focus;
    }
}
```

### 4.2CarAudioFocus的策略

CarAudioFocus是以zone为单位的，即一个zone里的AudioFocus在一个策略下进行管理

#### 4.2.1音频策略表 

```java
// Values for the internal interaction matrix we use to make focus decisions
    static final int INTERACTION_REJECT     = 0;    // Focus not granted
    static final int INTERACTION_EXCLUSIVE  = 1;    // Focus granted, others loose focus
    static final int INTERACTION_CONCURRENT = 2;    // Focus granted, others keep focus


    // TODO:  Make this an overlayable resource...
    //  MUSIC           = 1,        // Music playback
    //  NAVIGATION      = 2,        // Navigation directions
    //  VOICE_COMMAND   = 3,        // Voice command session
    //  CALL_RING       = 4,        // Voice call ringing
    //  CALL            = 5,        // Voice call
    //  ALARM           = 6,        // Alarm sound from Android
    //  NOTIFICATION    = 7,        // Notifications
    //  SYSTEM_SOUND    = 8,        // User interaction sounds (button clicks, etc)
    private static int sInteractionMatrix[][] = {
        // Row selected by playing sound (labels along the right)
        // Column selected by incoming request (labels along the top)
        // Cell value is one of INTERACTION_REJECT, INTERACTION_EXCLUSIVE, INTERACTION_CONCURRENT
/*第几行为当前占用的音频焦点类型，第几列为要申请的焦点类型，0是拒绝，1是独享其他丢失，2是共享其他不丢失       
  	如sInteractionMatrix[2][3] 即在导航占用焦点的情况下，语音请求焦点，对应的策略是1，语音要独享焦点
*/
        // Invalid, Music, Nav, Voice, Ring, Call, Alarm, Notification, System
        {  0,       0,     0,   0,     0,    0,    0,     0,            0 }, // Invalid
        {  0,       1,     2,   1,     1,    1,    1,     2,            2 }, // Music
        {  0,       2,     2,   1,     2,    1,    2,     2,            2 }, // Nav
        {  0,       2,     0,   2,     1,    1,    0,     0,            0 }, // Voice
        {  0,       0,     2,   2,     2,    2,    0,     0,            2 }, // Ring
        {  0,       0,     2,   0,     2,    2,    2,     2,            0 }, // Call
        {  0,       2,     2,   1,     1,    1,    2,     2,            2 }, // Alarm
        {  0,       2,     2,   1,     1,    1,    2,     2,            2 }, // Notification
        {  0,       2,     2,   1,     1,    1,    2,     2,            2 }, // System
    };

```



#### 4.2.2onAudioFocusRequest 

当有音频焦点请求时，evaluateFocusRequest方法就是策略执行的方法，将执行的结果传给Audiomanager

```java
@Override
public synchronized void onAudioFocusRequest(AudioFocusInfo afi, int requestResult) {
    Log.i(TAG, "onAudioFocusRequest " + afi.getClientId());

    int response = evaluateFocusRequest(afi);

    // Post our reply for delivery to the original focus requester
    mAudioManager.setFocusRequestResult(afi, response, mAudioPolicy);
}
```

evaluateFocusRequest方法比较长也是CarAudioFocus里的核心

总得来说就是就是

**1.维护了两个全局的map和初始化其他变量**

​				mFocusHolders   当前持有音频焦点的实体

​				mFocusLosers     已经短暂丢失音频焦点等待恢复的实体

​                newEntry   此次请求焦点的实体

​                permanentlyLost     存放要丢失长焦点的实体         

**3.遍历mFocusHolders   mFocusLosers这两个map**，得到两个局部的list  losers  blocked

​				losers     即将要丢失焦点的集合，mFocusHolders根据策略表，要丢失焦点的实体集合

​                blocked   已经丢失焦点需要再锁一次的集合，mFocusLosers根据策略表，再次添加一个音频焦点锁定者的集合

**4.遍历blocked**  如果是长时间丢失的音频焦点的，发送丢失焦点的时间，并将实体放入permanentlyLost的list里

​					  如果是短时间丢失的，在实体的 mBlockers里再加一个 当前请求焦点的newEntry实体，需要接收事件的发一个丢失短焦点的事件

**5.遍历losers**     根据请求焦点的类型，发送长、短音频焦点丢失的事件，并在mFocusHolders里删掉当前遍历的实体

​					  如果是长时间丢失，在permanentlyLost的列表里添加当前实体

​                      如果是短时间丢失，将当前实体放入mFocusLosers里，并在当前实体的mBlockers锁定者加上当前请求焦点的newEntry

**6.遍历permanentlyLost**    调用removeFocusEntryAndRestoreUnblockedWaiters方法，removeFocusEntryAndRestoreUnblockedWaiters里处理丢失长音频焦点的是不是其他丢失短音频焦点的锁定者，遍历mFocusLosers的map，在每个被锁定的实体锁定者列表里移除这个丢失长音频实体，如果移除后列表为空了，dispatchFocusGained

```java
int evaluateFocusRequest(AudioFocusInfo afi) {
    Log.i(TAG, "Evaluating " + focusEventToString(afi.getGainRequest()) + " request for client "
          + afi.getClientId());

    // Is this a request for premanant focus?
    // AUDIOFOCUS_GAIN_TRANSIENT_EXCLUSIVE -- Means Notifications should be denied
    // AUDIOFOCUS_GAIN_TRANSIENT -- Means current focus holders should get transient loss
    // AUDIOFOCUS_GAIN_TRANSIENT_MAY_DUCK -- Means other can duck (no loss message from us)
    // NOTE:  We expect that in practice it will be permanent for all media requests and
    //        transient for everything else, but that isn't currently an enforced requirement.
    //两个布尔值，用于判定此次请求焦点的类型
    //是否是长期获取焦点
    final boolean permanent =
        (afi.getGainRequest() == AudioManager.AUDIOFOCUS_GAIN);
    //是否允许其他声音低声播放
    final boolean allowDucking =
        (afi.getGainRequest() == AudioManager.AUDIOFOCUS_GAIN_TRANSIENT_MAY_DUCK);


    // Convert from audio attributes "usage" to HAL level "context"
    //将此次请求焦点的usage转换成Hal层context
    final int requestedContext = mCarAudioService.getContextForUsage(
        afi.getAttributes().getUsage());

    // If we happen to find entries that this new request should replace, we'll store them here.
    // This happens when a client makes a second AF request on the same listener.
    // After we've granted audio focus to our current request, we'll abandon these requests.
    /*
     如果我们碰巧发现这个新请求应该替换的条目，我们将把它们存储在这里。
     当客户端在同一个监听器上发出第二个AF请求时，就会发生这种情况。
     在我们将音频焦点分配给当前请求之后，我们将放弃这些请求。
    */
    FocusEntry replacedCurrentEntry = null;
    FocusEntry replacedBlockedEntry = null;

    // Scan all active and pending focus requests.  If any should cause rejection of
    // this new request, then we're done.  Keep a list of those against whom we're exclusive
    // so we can update the relationships if/when we are sure we won't get rejected.
    Log.i(TAG, "Scanning focus holders...");
    //创建losers list，将要丢失的音频焦点实体存入，之后统一处理，遍历当前所有的音频焦点持有
    final ArrayList<FocusEntry> losers = new ArrayList<FocusEntry>();
    for (FocusEntry entry : mFocusHolders.values()) {
        Log.d(TAG, "Evaluating focus holder: " + entry.getClientId());

        // If this request is for Notifications and a current focus holder has specified
        // AUDIOFOCUS_GAIN_TRANSIENT_EXCLUSIVE, then reject the request.
        // This matches the hardwired behavior in the default audio policy engine which apps
        // might expect (The interaction matrix doesn't have any provision for dealing with
        // override flags like this).
//步骤1. 如果请求的是notification，并且当前的焦点持有者是短暂获取，返回请求失败
        if ((requestedContext == ContextNumber.NOTIFICATION) &&
            (entry.mAfi.getGainRequest() ==
             AudioManager.AUDIOFOCUS_GAIN_TRANSIENT_EXCLUSIVE)) {
            return AudioManager.AUDIOFOCUS_REQUEST_FAILED;
        }

        // We don't allow sharing listeners (client IDs) between two concurrent requests
        // (because the app would have no way to know to which request a later event applied)
        
//步骤2. 不允许连续两次用一个监听器进行申请焦点
        // 如果两次申请的usage （context）相同，则暂存下后处理
        // 如果两次申请的usage（context）不同，直接返回请求失败
        if (afi.getClientId().equals(entry.mAfi.getClientId())) {
            if (entry.mAudioContext == requestedContext) {
                // This is a request from a current focus holder.
                // Abandon the previous request (without sending a LOSS notification to it),
                // and don't check the interaction matrix for it.
                Log.i(TAG, "Replacing accepted request from same client");
                replacedCurrentEntry = entry;
                continue;
            } else {
                // Trivially reject a request for a different USAGE
                Log.e(TAG, "Client " + entry.getClientId() + " has already requested focus "
                      + "for " + entry.mAfi.getAttributes().usageToString() + " - cannot "
                      + "request focus for " + afi.getAttributes().usageToString() + " on "
                      + "same listener.");
                return AudioManager.AUDIOFOCUS_REQUEST_FAILED;
            }
        }

        // Check the interaction matrix for the relationship between this entry and the request
/*步骤3. 查询请求策略表，
		结果为0 INTERACTION_REJECT直接返回请求失败
		结果为1 INTERACTION_EXCLUSIVE独享  加入losers列表
		结果为2 INTERACTION_CONCURRENT共享 
				如果当前请求不允许其他媒体播放，
				或者焦点占用源设置了暂停代替低声播放，
				或者需要监听duck事件（需要添加权限，监听所有音频焦点丢失的事件）
			当前焦点加入loser列表	
*/
        switch (sInteractionMatrix[entry.mAudioContext][requestedContext]) {
            case INTERACTION_REJECT:
                // This request is rejected, so nothing further to do
                return AudioManager.AUDIOFOCUS_REQUEST_FAILED;
            case INTERACTION_EXCLUSIVE:
                // The new request will cause this existing entry to lose focus
                //将当前占有音频焦点的实例存入losers 列表里
                losers.add(entry);
                break;
            case INTERACTION_CONCURRENT:
                // If ducking isn't allowed by the focus requestor, then everybody else
                // must get a LOSS.
                // If a focus holder has set the AUDIOFOCUS_FLAG_PAUSES_ON_DUCKABLE_LOSS flag,
                // they must get a LOSS message even if ducking would otherwise be allowed.
                // If a focus holder holds the RECEIVE_CAR_AUDIO_DUCKING_EVENTS permission,
                // they must receive all audio focus losses.
//判定是不是要让当前音频持有者接收到音频焦点丢失的事件                
                if (!allowDucking
                    || entry.wantsPauseInsteadOfDucking()
                    || entry.receivesDuckEvents()) {
                    losers.add(entry);
                }
                break;
            default:
                Log.e(TAG, "Bad interaction matrix value - rejecting");
                return AudioManager.AUDIOFOCUS_REQUEST_FAILED;
        }
    }
    //到这所有持有音频焦点的实体遍历完毕
    
//同样的步骤在mFocusLosers list在搜索一遍，要丢失放入blocked list里，连续两次的暂存replacedBlockedEntry 
    Log.i(TAG, "Scanning those who've already lost focus...");
    final ArrayList<FocusEntry> blocked = new ArrayList<FocusEntry>();
    for (FocusEntry entry : mFocusLosers.values()) {
        Log.i(TAG, entry.mAfi.getClientId());

        // If this request is for Notifications and a pending focus holder has specified
        // AUDIOFOCUS_GAIN_TRANSIENT_EXCLUSIVE, then reject the request
        if ((requestedContext == ContextNumber.NOTIFICATION) &&
            (entry.mAfi.getGainRequest() ==
             AudioManager.AUDIOFOCUS_GAIN_TRANSIENT_EXCLUSIVE)) {
            return AudioManager.AUDIOFOCUS_REQUEST_FAILED;
        }

        // We don't allow sharing listeners (client IDs) between two concurrent requests
        // (because the app would have no way to know to which request a later event applied)
        if (afi.getClientId().equals(entry.mAfi.getClientId())) {
            if (entry.mAudioContext == requestedContext) {
                // This is a repeat of a request that is currently blocked.
                // Evaluate it as if it were a new request, but note that we should remove
                // the old pending request, and move it.
                // We do not want to evaluate the new request against itself.
                Log.i(TAG, "Replacing pending request from same client");
                replacedBlockedEntry = entry;
                continue;
            } else {
                // Trivially reject a request for a different USAGE
                Log.e(TAG, "Client " + entry.getClientId() + " has already requested focus "
                      + "for " + entry.mAfi.getAttributes().usageToString() + " - cannot "
                      + "request focus for " + afi.getAttributes().usageToString() + " on "
                      + "same listener.");
                return AudioManager.AUDIOFOCUS_REQUEST_FAILED;
            }
        }

        // Check the interaction matrix for the relationship between this entry and the request
        switch (sInteractionMatrix[entry.mAudioContext][requestedContext]) {
            case INTERACTION_REJECT:
                // Even though this entry has currently lost focus, the fact that it is
                // waiting to play means we'll reject this new conflicting request.
                return AudioManager.AUDIOFOCUS_REQUEST_FAILED;
            case INTERACTION_EXCLUSIVE:
                // The new request is yet another reason this entry cannot regain focus (yet)
                blocked.add(entry);
                break;
            case INTERACTION_CONCURRENT:
                // If ducking is not allowed by the requester, or the pending focus holder had
                // set the AUDIOFOCUS_FLAG_PAUSES_ON_DUCKABLE_LOSS flag, or if the pending
                // focus holder has requested to receive all focus events, then the pending
                // holder must stay "lost" until this requester goes away.
                if (!allowDucking
                    || entry.wantsPauseInsteadOfDucking()
                    || entry.receivesDuckEvents()) {
                    // The new request is yet another reason this entry cannot regain focus yet
                    blocked.add(entry);
                }
                break;
            default:
                Log.e(TAG, "Bad interaction matrix value - rejecting");
                return AudioManager.AUDIOFOCUS_REQUEST_FAILED;
        }
    }
//mFocusLosers遍历完成

    // Now that we've decided we'll grant focus, construct our new FocusEntry
    FocusEntry newEntry = new FocusEntry(afi, requestedContext);

    // These entries have permanently lost focus as a result of this request, so they
    // should be removed from all blocker lists.
    ArrayList<FocusEntry> permanentlyLost = new ArrayList<>();
//如果出现连续两次请求的在mFocusHolders和mFocusLosers里都去掉，并放入长时间丢失的permanentlyLost list
    if (replacedCurrentEntry != null) {
        mFocusHolders.remove(replacedCurrentEntry.getClientId());
        permanentlyLost.add(replacedCurrentEntry);
    }
    if (replacedBlockedEntry != null) {
        mFocusLosers.remove(replacedBlockedEntry.getClientId());
        permanentlyLost.add(replacedBlockedEntry);
    }


    // Now that we're sure we'll accept this request, update any requests which we would
    // block but are already out of focus but waiting to come back
    for (FocusEntry entry : blocked) {
        // If we're out of focus it must be because somebody is blocking us
        assert !entry.mBlockers.isEmpty();
//如果当前请求时永久行，发送焦点丢失事件，并将该实体的焦点短暂丢失可以低声播放flag置为false
//在mFocusLosers里删除，在permanentlyLost里添加这个实体        
        if (permanent) {
            // This entry has now lost focus forever
            sendFocusLoss(entry, AudioManager.AUDIOFOCUS_LOSS);
            entry.mReceivedLossTransientCanDuck = false;
            final FocusEntry deadEntry = mFocusLosers.remove(entry.mAfi.getClientId());
            assert deadEntry != null;
            permanentlyLost.add(entry);
        } else {
//如果此次请求的音频焦点不允许其他音频低声播放，并且当前音频焦点实体需要接受所有的焦点丢失事件
//发送短暂丢失音频焦点事件，并将该实体的焦点短暂丢失可以低声播放flag置为false
            if (!allowDucking && entry.mReceivedLossTransientCanDuck) {
                // This entry was previously allowed to duck, but can no longer do so.
                Log.i(TAG, "Converting duckable loss to non-duckable for "
                      + entry.getClientId());
                sendFocusLoss(entry, AudioManager.AUDIOFOCUS_LOSS_TRANSIENT);
                entry.mReceivedLossTransientCanDuck = false;
            }
            // Note that this new request is yet one more reason we can't (yet) have focus
//当前实体，在锁定者里添加，这次请求音频焦点的实体
            entry.mBlockers.add(newEntry);
        }
    }
//遍历losers，这个list里的实体肯定是要丢失焦点的
    // Notify and update any requests which are now losing focus as a result of the new request
    for (FocusEntry entry : losers) {
        // If we have focus (but are about to loose it), nobody should be blocking us yet
        assert entry.mBlockers.isEmpty();
//确定丢失焦点类型
        int lossType;
        if (permanent) {
            lossType = AudioManager.AUDIOFOCUS_LOSS;
        } else if (allowDucking && entry.receivesDuckEvents()) {
            lossType = AudioManager.AUDIOFOCUS_LOSS_TRANSIENT_CAN_DUCK;
//将该实体的焦点短暂丢失可以低声播放flag置为true           
            entry.mReceivedLossTransientCanDuck = true;
        } else {
            lossType = AudioManager.AUDIOFOCUS_LOSS_TRANSIENT;
        }
//发送丢失焦点的事件        
        sendFocusLoss(entry, lossType);

        // The entry no longer holds focus, so take it out of the holders list
//焦点持有者列表里删除该实体，详见4.2.3onAudioFocusAbandon里的短焦点恢复
        mFocusHolders.remove(entry.mAfi.getClientId());

        if (permanent) {
            permanentlyLost.add(entry);
        } else {
            // Add ourselves to the list of requests waiting to get focus back and
            // note why we lost focus so we can tell when it's time to get it back
//如果是长时间请求，将当前实体放入permanentlyLost里，如果是短时间就放入mFocusLosers列表，并把该实体的锁定者里添加当前请求焦点的实体            
            mFocusLosers.put(entry.mAfi.getClientId(), entry);
            entry.mBlockers.add(newEntry);
        }
    }

    // Now that all new blockers have been added, clear out any other requests that have been
    // permanently lost as a result of this request. Treat them as abandoned - if they're on
    // any blocker lists, remove them. If any focus requests become unblocked as a result,
    // re-grant them. (This can happen when a GAIN_TRANSIENT_MAY_DUCK request replaces a
    // GAIN_TRANSIENT request from the same listener.)
//遍历长时间丢失列表，调用removeFocusEntryAndRestoreUnblockedWaiters方法
    for (FocusEntry entry : permanentlyLost) {
        Log.d(TAG, "Cleaning up entry " + entry.getClientId());
        removeFocusEntryAndRestoreUnblockedWaiters(entry);
    }

    // Finally, add the request we're granting to the focus holders' list
//将当前请求实体放入mFocusHolders列表里
    mFocusHolders.put(afi.getClientId(), newEntry);

    Log.i(TAG, "AUDIOFOCUS_REQUEST_GRANTED");
    return AudioManager.AUDIOFOCUS_REQUEST_GRANTED;
}
```

#### 4.2.3onAudioFocusAbandon

放弃音频焦点的处理较获取来说就简单多了

```java
    @Override
    public synchronized void onAudioFocusAbandon(AudioFocusInfo afi) {
        Log.i(TAG, "onAudioFocusAbandon " + afi.getClientId());

        FocusEntry deadEntry = removeFocusEntry(afi);

        if (deadEntry != null) {
            removeFocusEntryAndRestoreUnblockedWaiters(deadEntry);
        }
    }
```

在mFocusHolders列表和mFocusLosers列表里进行删除

```java
    private FocusEntry removeFocusEntry(AudioFocusInfo afi) {
        Log.i(TAG, "removeFocusEntry " + afi.getClientId());

        // Remove this entry from our active or pending list
        FocusEntry deadEntry = mFocusHolders.remove(afi.getClientId());
        if (deadEntry == null) {
            deadEntry = mFocusLosers.remove(afi.getClientId());
            if (deadEntry == null) {
                // Caller is providing an unrecognzied clientId!?
                Log.w(TAG, "Audio focus abandoned by unrecognized client id: " + afi.getClientId());
                // This probably means an app double released focused for some reason.  One
                // harmless possibility is a race between an app being told it lost focus and the
                // app voluntarily abandoning focus.  More likely the app is just sloppy.  :)
                // The more nefarious possibility is that the clientId is actually corrupted
                // somehow, in which case we might have a real focus entry that we're going to fail
                // to remove. If that were to happen, I'd expect either the app to swallow it
                // silently, or else take unexpected action (eg: resume playing spontaneously), or
                // else to see "Failure to signal ..." gain/loss error messages in the log from
                // this module when a focus change tries to take action on a truly zombie entry.
            }
        }
        return deadEntry;
    }
```

拿到被删除的实体deadEntry引用进行，恢复策略处理

遍历暂时丢失焦点的列表的实体，在遍历的实体里的锁定者列表里移除被删除的实体deadEntry。

如果剩下的锁定者列表为空，则将该实体加入mFocusHolders里，并分发获得焦点事件

```java
    private void removeFocusEntryAndRestoreUnblockedWaiters(FocusEntry deadEntry) {
        // Remove this entry from the blocking list of any pending requests
        //遍历暂时丢失焦点的列表的实体
        Iterator<FocusEntry> it = mFocusLosers.values().iterator();
        while (it.hasNext()) {
            FocusEntry entry = it.next();
		//		
            // Remove the retiring entry from all blocker lists
            entry.mBlockers.remove(deadEntry);

            // Any entry whose blocking list becomes empty should regain focus
            if (entry.mBlockers.isEmpty()) {
                Log.i(TAG, "Restoring unblocked entry " + entry.getClientId());
                // Pull this entry out of the focus losers list
                it.remove();

                // Add it back into the focus holders list
                mFocusHolders.put(entry.getClientId(), entry);

                dispatchFocusGained(entry.mAfi);

            }
        }
    }
```

4.2.4 音频焦点例子

举个请求焦点的简单例子，A请求长焦点（无其他焦点），B请求短焦点，A进入等待解锁状态锁定者是B。

此时如果C请求短焦点，B被C锁定，A被BC锁定。C播放完之后，B恢复播放，A被B锁定。

如果C请求的长焦点，A接受长焦点丢失事件，B丢失长焦点丢失事件，并且A从mFocusLoser中移除，B从mFocusHolders移除，然后遍历当前mFocusLoser里的实体删除A B锁定者，如果删除后锁定者列表为空了，恢复实体播放

### 4.3 Uid、ZoneId和AudioFocus的关系

涉及到三个方法，CarAudioManger对外有三分公共方法，对应着CarAudioService的三个同名方法，有一个关键变量

mUidToZoneMap维护uid和zoneId的映射map

```java
    int getZoneIdForUid(int uid);
    boolean setZoneIdForUid(int zoneId, int uid);
    boolean clearZoneIdForUid(int uid);
```

get方法很简单，直接查询列表，如果表里是空的，就把这个UID对应到PrimaryZoneId，调用setZoneIdForUidNoCheckLocked走一遍流程

再看set方法

```java
    @Override
    public boolean setZoneIdForUid(int zoneId, int uid) {
        enforcePermission(Car.PERMISSION_CAR_CONTROL_AUDIO_SETTINGS);
        synchronized (mImplLock) {
            Log.i(CarLog.TAG_AUDIO, "setZoneIdForUid Calling uid "
                    + uid + " mapped to : "
                    + zoneId);

            // Figure out if anything is currently holding focus,
            // This will change the focus to transient loss while we are switching zones、
//uid 对应zoneId是一一对应的  通过UID找到了zoneId 说明之前是设置过得
//一个Uid可能对应着好多focus实体，有正在播的，也可能有被抢等待恢复的         
            Integer currentZoneId = mUidToZoneMap.get(uid);
            ArrayList<AudioFocusInfo> currentFocusHoldersForUid = new ArrayList<>();
            ArrayList<AudioFocusInfo> currentFocusLosersForUid = new ArrayList<>();
            if (currentZoneId != null) {
//把属于这个UID的两种类型的AudioFocus整理成两个列表，移除并发送短音频焦点丢失事件                 
                currentFocusHoldersForUid = mFocusHandler.getAudioFocusHoldersForUid(uid,
                        currentZoneId.intValue());
                currentFocusLosersForUid = mFocusHandler.getAudioFocusLosersForUid(uid,
                        currentZoneId.intValue());
                if (!currentFocusHoldersForUid.isEmpty() || !currentFocusLosersForUid.isEmpty()) {
                    // Order matters here: Remove the focus losers first
                    // then do the current holder to prevent loser from popping up while
                    // the focus is being remove for current holders
                    // Remove focus for current focus losers
                    mFocusHandler.transientlyLoseInFocusInZone(currentFocusLosersForUid,
                            currentZoneId.intValue());
                    // Remove focus for current holders
                    mFocusHandler.transientlyLoseInFocusInZone(currentFocusHoldersForUid,
                            currentZoneId.intValue());
                }
            }

            // if the current uid is in the list
            // remove it from the list
//从mUidToZoneMap移除这个uid
            if (checkAndRemoveUidLocked(uid)) {
//切换新的zoneid，放进mUidToZoneMap里                
                if (setZoneIdForUidNoCheckLocked(zoneId, uid)) {
                    // Order matters here: Regain focus for
                    // Previously lost focus holders then regain
                    // focus for holders that had it last
                    // Regain focus for the focus losers from previous zone
//根据之前的两个表进行音频焦点恢复                    
                    if (!currentFocusLosersForUid.isEmpty()) {
                        regainAudioFocusLocked(currentFocusLosersForUid, zoneId);
                    }
                    // Regain focus for the focus holders from previous zone
                    if (!currentFocusHoldersForUid.isEmpty()) {
                        regainAudioFocusLocked(currentFocusHoldersForUid, zoneId);
                    }
                    return true;
                }
            }
            return false;
        }
    }

```





## 五、外部音源AudioPatch

### 5.1 外部音源

外部音源为除了mic外的其他音源，通过Output进行播放，如FM、USBdevice、HDMI等等。

getExternalSource()调用AudioManger的getDevice获取input

```java
AudioDeviceInfo[] devices = mAudioManager.getDevices(AudioManager.GET_DEVICES_INPUTS);
```

然后将对应类型进行过滤，返回String[]数组

```java
for (AudioDeviceInfo info : devices) {
                switch (info.getType()) {
                    // TODO:  Can we trim this set down? Especially duplicates like FM vs FM_TUNER?
                    case AudioDeviceInfo.TYPE_FM:
                    case AudioDeviceInfo.TYPE_FM_TUNER:
                    case AudioDeviceInfo.TYPE_TV_TUNER:
                    case AudioDeviceInfo.TYPE_HDMI:
                    case AudioDeviceInfo.TYPE_AUX_LINE:
                    case AudioDeviceInfo.TYPE_LINE_ANALOG:
                    case AudioDeviceInfo.TYPE_LINE_DIGITAL:
                    case AudioDeviceInfo.TYPE_USB_ACCESSORY:
                    case AudioDeviceInfo.TYPE_USB_DEVICE:
                    case AudioDeviceInfo.TYPE_USB_HEADSET:
                    case AudioDeviceInfo.TYPE_IP:
                    case AudioDeviceInfo.TYPE_BUS:
                        String address = info.getAddress();
                        if (TextUtils.isEmpty(address)) {
                            Log.w(CarLog.TAG_AUDIO,
                                    "Discarded device with empty address, type=" + info.getType());
                        } else {
                            sourceAddresses.add(address);
                        }
                }
            }
```

### 5.2 createAudioPatch

createAudioPatch 传入了三个参数

String sourceAddress   音源地址

int usage 用法

int  gainInMillibels  音量增益

总的来说就是 

将音频源的sourceConfig和播放sink 的sinkConfig作为参数，调用AudioManger的createAudioPatch

```java
    @Override
    public CarAudioPatchHandle createAudioPatch(String sourceAddress,
            @AudioAttributes.AttributeUsage int usage, int gainInMillibels) {
        synchronized (mImplLock) {
            enforcePermission(Car.PERMISSION_CAR_CONTROL_AUDIO_SETTINGS);
            return createAudioPatchLocked(sourceAddress, usage, gainInMillibels);
        }
    }
```

```java
private CarAudioPatchHandle createAudioPatchLocked(String sourceAddress,
            @AudioAttributes.AttributeUsage int usage, int gainInMillibels) {
        // Find the named source port
//1 从AudioManager getDevice 找到对应AudioDeviceInfo sourceAddress
        AudioDeviceInfo sourcePortInfo = null;
        AudioDeviceInfo[] deviceInfos = mAudioManager.getDevices(AudioManager.GET_DEVICES_INPUTS);
        for (AudioDeviceInfo info : deviceInfos) {
            if (sourceAddress.equals(info.getAddress())) {
                // This is the one for which we're looking
                sourcePortInfo = info;
                break;
            }
        }
//Preconditions.checkNotNull(T) value不为空返回value  为空抛出空指针异常  
        Preconditions.checkNotNull(sourcePortInfo,
                "Specified source is not available: " + sourceAddress);
        // Find the output port associated with the given carUsage
//2 getAudioPort -> usage+PrmaryId getVolumeGroupIdForUsage -> group getContextForUsage得到context
//通过group的getAudioDevicePortForContext得到  AudioDevicePort    
        AudioDevicePort sinkPort = Preconditions.checkNotNull(getAudioPort(usage),
                "Sink not available for usage: " + AudioAttributes.usageToString(usage));

        // {@link android.media.AudioPort#activeConfig()} is valid for mixer port only,
        // since audio framework has no clue what's active on the device ports.
        // Therefore we construct an empty / default configuration here, which the audio HAL
        // implementation should ignore.
//3 构建一个空的sinkConfig，hal应该忽略这个配置，使用当前活跃的配置    
        AudioPortConfig sinkConfig = sinkPort.buildConfig(0,
                AudioFormat.CHANNEL_OUT_DEFAULT, AudioFormat.ENCODING_DEFAULT, null);
        Log.d(CarLog.TAG_AUDIO, "createAudioPatch sinkConfig: " + sinkConfig);

        // Configure the source port to match the output port except for a gain adjustment
//4 创建 audioGainConfig  由 MODE——JOINT和 对应output 的端口号音频增量配置 以及传入的gainInMillibels构建 
// MODE_JONIT就是所有的声道都同时增加一个增益值
        final CarAudioDeviceInfo helper = new CarAudioDeviceInfo(sourcePortInfo);
        AudioGain audioGain = Preconditions.checkNotNull(helper.getAudioGain(),
                "Gain controller not available for source port");

        // size of gain values is 1 in MODE_JOINT
        AudioGainConfig audioGainConfig = audioGain.buildConfig(AudioGain.MODE_JOINT,
                audioGain.channelMask(), new int[] { gainInMillibels }, 0);
        // Construct an empty / default configuration excepts gain config here and it's up to the
        // audio HAL how to interpret this configuration, which the audio HAL
        // implementation should ignore.
        AudioPortConfig sourceConfig = sourcePortInfo.getPort().buildConfig(0,
                AudioFormat.CHANNEL_IN_DEFAULT, AudioFormat.ENCODING_DEFAULT, audioGainConfig);
/*5 创建AudioPatch   调用AudioManger的createAudioPatch
    sourceConfig   外部音源的设置
    sinkConfig     output的sink设置

*/
        // Create an audioPatch to connect the two ports
        AudioPatch[] patch = new AudioPatch[] { null };
        int result = AudioManager.createAudioPatch(patch,
                new AudioPortConfig[] { sourceConfig },
                new AudioPortConfig[] { sinkConfig });
        if (result != AudioManager.SUCCESS) {
            throw new RuntimeException("createAudioPatch failed with code " + result);
        }

        Preconditions.checkNotNull(patch[0],
                "createAudioPatch didn't provide expected single handle");
        Log.d(CarLog.TAG_AUDIO, "Audio patch created: " + patch[0]);
//6 重设下音量 确保音量没有变化
        // Ensure the initial volume on output device port
        int groupId = getVolumeGroupIdForUsage(CarAudioManager.PRIMARY_AUDIO_ZONE, usage);
        setGroupVolume(CarAudioManager.PRIMARY_AUDIO_ZONE, groupId,
                getGroupVolume(CarAudioManager.PRIMARY_AUDIO_ZONE, groupId), 0);

        return new CarAudioPatchHandle(patch[0]);
    }

```

