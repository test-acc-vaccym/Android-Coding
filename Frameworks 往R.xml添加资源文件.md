如果我们需要在framework-res.apk里面添加一个xml文件，并通过com.android.internal.R.xml.car_volume_group来访问
- **添加car_volume_group.xml**
1. 通过添加到frameworks/base/core/res/res/资源文件夹里面

2. 通过overlay的方式去重载资源文件
```
<volumeGroups xmlns:car="http://schemas.android.com/apk/res-auto">
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
- **添加attrs.xml 和 symbols.xml**
frameworks/base/core/res/res/values/attrs.xml
```
    <!-- Defines the attributes and values used in res/xml/car_volume_group.xml -->
    <declare-styleable name="volumeGroups" />

    <declare-styleable name="volumeGroups_group"/>

    <declare-styleable name="volumeGroups_context">
        <!-- Align with hardware/interfaces/automotive/audiocontrol/1.0/types.hal:ContextNumber -->
        <attr name="context">
            <enum name="music" value="1"/>
            <enum name="navigation" value="2"/>
            <enum name="voice_command" value="3"/>
            <enum name="call_ring" value="4"/>
            <enum name="call" value="5"/>
            <enum name="alarm" value="6"/>
            <enum name="notification" value="7"/>
            <enum name="system_sound" value="8"/>
        </attr>
    </declare-styleable>
```
frameworks/base/core/res/res/values/symbols.xml
```
  <!-- Private symbols that we need to reference from framework code.  See
       frameworks/base/core/res/MakeJavaSymbols.sed for how to easily generate
       this.

       Can be referenced in java code as: com.android.internal.R.<type>.<name>
       and in layout xml as: "@*android:<type>/<name>"
  -->
  <java-symbol type="xml" name="car_volume_group" />
```
- **代码调用**
从代码里面解析配置的xml文件
```
class CarVolumeGroupsHelper {

    private static final String TAG_VOLUME_GROUPS = "volumeGroups";
    private static final String TAG_GROUP = "group";
    private static final String TAG_CONTEXT = "context";

    private final Context mContext;
    private final @XmlRes int mXmlConfiguration;

    CarVolumeGroupsHelper(Context context, @XmlRes int xmlConfiguration) {
        mContext = context;
        mXmlConfiguration = xmlConfiguration;
    }

    CarVolumeGroup[] loadVolumeGroups() {
        List<CarVolumeGroup> carVolumeGroups = new ArrayList<>();
        try (XmlResourceParser parser = mContext.getResources().getXml(mXmlConfiguration)) {
            AttributeSet attrs = Xml.asAttributeSet(parser);
            int type;
            // Traverse to the first start tag
            while ((type=parser.next()) != XmlResourceParser.END_DOCUMENT
                    && type != XmlResourceParser.START_TAG) {
            }

            if (!TAG_VOLUME_GROUPS.equals(parser.getName())) {
                throw new RuntimeException("Meta-data does not start with volumeGroups tag");
            }
            int outerDepth = parser.getDepth();
            int id = 0;
            while ((type=parser.next()) != XmlResourceParser.END_DOCUMENT
                    && (type != XmlResourceParser.END_TAG || parser.getDepth() > outerDepth)) {
                if (type == XmlResourceParser.END_TAG) {
                    continue;
                }
                if (TAG_GROUP.equals(parser.getName())) {
                    carVolumeGroups.add(parseVolumeGroup(id, attrs, parser));
                    id++;
                }
            }
        } catch (Exception e) {
            Log.e(CarLog.TAG_AUDIO, "Error parsing volume groups configuration", e);
        }
        return carVolumeGroups.toArray(new CarVolumeGroup[carVolumeGroups.size()]);
    }

    private CarVolumeGroup parseVolumeGroup(int id, AttributeSet attrs, XmlResourceParser parser)
            throws XmlPullParserException, IOException {
        int type;

        List<Integer> contexts = new ArrayList<>();
        int innerDepth = parser.getDepth();
        while ((type=parser.next()) != XmlResourceParser.END_DOCUMENT
                && (type != XmlResourceParser.END_TAG || parser.getDepth() > innerDepth)) {
            if (type == XmlResourceParser.END_TAG) {
                continue;
            }
            if (TAG_CONTEXT.equals(parser.getName())) {
                TypedArray c = mContext.getResources().obtainAttributes(
                        attrs, R.styleable.volumeGroups_context);
                contexts.add(c.getInt(R.styleable.volumeGroups_context_context, -1));
                c.recycle();
            }
        }

        return new CarVolumeGroup(mContext, id,
                contexts.stream().mapToInt(i -> i).filter(i -> i >= 0).toArray());
    }
}

```
```
        final CarVolumeGroupsHelper helper = new CarVolumeGroupsHelper(
                mContext, R.xml.car_volume_groups);
        mCarVolumeGroups = helper.loadVolumeGroups();
```

