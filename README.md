![logo_t](https://github.com/lxhAndSmh/HyperLPR/blob/master/demo_images/Android%20%E5%AE%9E%E6%97%B6%E6%89%AB%E6%8F%8F.gif)

## HyperLPR Android 新能源车牌识别、实时识别和识别区域的获取

### Q&A

Q：Android 选择新能源车牌号的图片进行识别，为什么识别不出来

A：可能和车牌号的拍摄角度、分辨率有关，尝试调节分辨率看看

Q：如何验证支持了新能源车牌号识别

A：换用实时识别试试。


###  新能源车牌识别
 ##### 1.将 Prj-Linux 项目下 model 中的文件全部 copy 到 Prj-Android 中 assets 的 pr 目录下
 ##### 2.修改 MainActivity.java 的 initRecognizer() 方法
```
public void initRecognizer()
   {
       String assetPath = "pr";
       String sdcardPath = Environment.getExternalStorageDirectory()
               + File.separator + assetPath;
       copyFilesFromAssets(this, assetPath, sdcardPath);
       String cascade_filename  =  sdcardPath
           + File.separator+"cascade.xml";
       String finemapping_prototxt  =  sdcardPath
               + File.separator+"HorizonalFinemapping.prototxt";
       String finemapping_caffemodel  =  sdcardPath
               + File.separator+"HorizonalFinemapping.caffemodel";
       String segmentation_prototxt =  sdcardPath
               + File.separator+"Segmentation.prototxt";
       String segmentation_caffemodel =  sdcardPath
               + File.separator+"Segmentation.caffemodel";
       String character_prototxt =  sdcardPath
               + File.separator+"CharacterRecognization.prototxt";
       String character_caffemodel=  sdcardPath
               + File.separator+"CharacterRecognization.caffemodel";
      //添加新能源车牌识别
       String segmentationfree_prototxt =  sdcardPath
               + File.separator+"SegmenationFree-Inception.prototxt";
       String segmentationfree_caffemodel=  sdcardPath
               + File.separator+"SegmenationFree-Inception.caffemodel";
       handle  =  PlateRecognition.InitPlateRecognizer(
               cascade_filename,
               finemapping_prototxt,finemapping_caffemodel,
               segmentation_prototxt,segmentation_caffemodel,
               character_prototxt,character_caffemodel,
               segmentationfree_prototxt,segmentationfree_caffemodel
       );
   }
```
##### 3.修改 PlateRecognition.java
```
public class PlateRecognition {
    static {
        System.loadLibrary("hyperlpr");
    }
    static native long InitPlateRecognizer(String casacde_detection,
                                           String finemapping_prototxt,String finemapping_caffemodel,
                                           String segmentation_prototxt,String segmentation_caffemodel,
                                           String charRecognization_proto,String charRecognization_caffemodel,
                                           //添加新能源的参数
                                           String segmentationfree_proto, String segmentationfree_caffemodel);

    static native void ReleasePlateRecognizer(long  object);
    static native String SimpleRecognization(long  inputMat,long object);
}
```
##### 4.修改 javaWrapper.cpp 文件的以下两部分
```
extern "C" {
JNIEXPORT jlong JNICALL
Java_pr_platerecognization_PlateRecognition_InitPlateRecognizer(
        JNIEnv *env, jobject obj,
        jstring detector_filename,
        jstring finemapping_prototxt, jstring finemapping_caffemodel,
        jstring segmentation_prototxt, jstring segmentation_caffemodel,
        jstring charRecognization_proto, jstring charRecognization_caffemodel,
        jstring segmentationfree_proto, jstring segmentationfree_caffemodel) {

    std::string detector_path = jstring2str(env, detector_filename);
    std::string finemapping_prototxt_path = jstring2str(env, finemapping_prototxt);
    std::string finemapping_caffemodel_path = jstring2str(env, finemapping_caffemodel);
    std::string segmentation_prototxt_path = jstring2str(env, segmentation_prototxt);
    std::string segmentation_caffemodel_path = jstring2str(env, segmentation_caffemodel);
    std::string charRecognization_proto_path = jstring2str(env, charRecognization_proto);
    std::string charRecognization_caffemodel_path = jstring2str(env, charRecognization_caffemodel);
    //新能源识别参数
    std::string segmentationfree_proto_path = jstring2str(env, segmentationfree_proto);
    std::string segmentationfree_caffemodel_path = jstring2str(env, segmentationfree_caffemodel);

    pr::PipelinePR *PR = new pr::PipelinePR(detector_path,
                                            finemapping_prototxt_path, finemapping_caffemodel_path,
                                            segmentation_prototxt_path, segmentation_caffemodel_path,
                                            charRecognization_proto_path, charRecognization_caffemodel_path,
                                            //新能源识别参数
                                            segmentationfree_proto_path, segmentationfree_caffemodel_path);
    return (jlong) PR;
}

JNIEXPORT jstring JNICALL
Java_pr_platerecognization_PlateRecognition_SimpleRecognization(
        JNIEnv *env, jobject obj,
        jlong matPtr, jlong object_pr) {
    pr::PipelinePR *PR = (pr::PipelinePR *) object_pr;
    cv::Mat &mRgb = *(cv::Mat *) matPtr;
    cv::Mat rgb;
    cv::cvtColor(mRgb,rgb,cv::COLOR_RGBA2BGR);


    //1表示 SEGMENTATION_BASED_METHOD 在方法里有说明,该方法能够识别新能源车牌
    std::vector<pr::PlateInfo> list_res= PR->RunPiplineAsImage(rgb,pr::SEGMENTATION_FREE_METHOD);
//    std::vector<pr::PlateInfo> list_res= PR->RunPiplineAsImage(rgb,1);
    std::string concat_results;
    for(auto one:list_res)
    {
        //可信度
        if (one.confidence>0.7)
            concat_results+=one.getPlateName()+",";
    }

    concat_results = concat_results.substr(0,concat_results.size()-1);

    return env->NewStringUTF(concat_results.c_str());
}
```
##### 5.打开 cpp.src 下 Pipeline.cpp 中的以下三处的注释
```
const int HorizontalPadding = 4;
PipelinePR::PipelinePR(std::string detector_filename,
                       std::string finemapping_prototxt, std::string finemapping_caffemodel,
                       std::string segmentation_prototxt, std::string segmentation_caffemodel,
                       std::string charRecognization_proto, std::string charRecognization_caffemodel,
                       std::string segmentationfree_proto, std::string segmentationfree_caffemodel) {
    plateDetection = new PlateDetection(detector_filename);
    fineMapping = new FineMapping(finemapping_prototxt, finemapping_caffemodel);
    plateSegmentation = new PlateSegmentation(segmentation_prototxt, segmentation_caffemodel);
    generalRecognizer = new CNNRecognizer(charRecognization_proto, charRecognization_caffemodel);
    //新能源车牌识别
    segmentationFreeRecognizer =  new SegmentationFreeRecognizer(segmentationfree_proto,segmentationfree_caffemodel);
}

PipelinePR::~PipelinePR() {

    delete plateDetection;
    delete fineMapping;
    delete plateSegmentation;
    delete generalRecognizer;
    //新能源车牌识别
    delete segmentationFreeRecognizer;
}
```
##### 6.打开 cpp.include 包下 Pipeline.h 文件中的注释
```
namespace pr{

    const std::vector<std::string> CH_PLATE_CODE{"京", "沪", "津", "渝", "冀", "晋", "蒙", "辽", "吉", "黑", "苏", "浙", "皖", "闽", "赣", "鲁", "豫", "鄂", "湘", "粤", "桂",
                                                 "琼", "川", "贵", "云", "藏", "陕", "甘", "青", "宁", "新", "0", "1", "2", "3", "4", "5", "6", "7", "8", "9", "A",
                                                 "B", "C", "D", "E", "F", "G", "H", "J", "K", "L", "M", "N", "P", "Q", "R", "S", "T", "U", "V", "W", "X",
                                                 "Y", "Z","港","学","使","警","澳","挂","军","北","南","广","沈","兰","成","济","海","民","航","空"};

    const int SEGMENTATION_FREE_METHOD = 0;
    const int SEGMENTATION_BASED_METHOD = 1;

    class PipelinePR{
    public:
        GeneralRecognizer *generalRecognizer;
        PlateDetection *plateDetection;
        PlateSegmentation *plateSegmentation;
        FineMapping *fineMapping;
        SegmentationFreeRecognizer *segmentationFreeRecognizer;

        PipelinePR(std::string detector_filename,
                   std::string finemapping_prototxt,std::string finemapping_caffemodel,
                   std::string segmentation_prototxt,std::string segmentation_caffemodel,
                   std::string charRecognization_proto,std::string charRecognization_caffemodel,
                   //新能源车牌识别
                   std::string segmentationfree_proto,std::string segmentationfree_caffemodel
        );
        ~PipelinePR();

        std::vector<std::string> plateRes;
        std::vector<PlateInfo> RunPiplineAsImage(cv::Mat plateImage,int method);
    };
}
#endif
```

### 获取识别区域的车牌号图片

##### 1.在 cpp 包下的 CMakeLists.txt 配置文件中添加 jnigraphics 依赖库
`target_link_libraries(hyperlpr jnigraphics ${OpenCV_LIBS})`

##### 2.创建包含识别区域图片的对象 PlateInfo
```
public class PlateInfo {
    /**
     * 车牌号
     */
    public String plateName;
    /**
     * 车牌号图片
     */
    public Bitmap bitmap;
    public PlateInfo() {
    }
    public PlateInfo(String plateName, Bitmap bitmap) {
        this.plateName = plateName;
        this.bitmap = bitmap;
    }
}
```

##### 3.在 PlateRecognition 类中添加获取车牌号识别区域图片的接口
```
public class PlateRecognition {
    static {
        System.loadLibrary("hyperlpr");
    }
    static native long InitPlateRecognizer(String casacde_detection,
                                           String finemapping_prototxt,String finemapping_caffemodel,
                                           String segmentation_prototxt,String segmentation_caffemodel,
                                           String charRecognization_proto,String charRecognization_caffemodel,
                                           //添加新能源的参数
                                           String segmentationfree_proto, String segmentationfree_caffemodel);

    static native void ReleasePlateRecognizer(long  object);
    static native String SimpleRecognization(long  inputMat,long object);
    /**
     * 获取识别区域的车牌号图片
     * @param inputMat
     * @param object
     * @return
     */
    static native PlateInfo PlateInfoRecognization(long  inputMat,long object);
}
```

##### 4.在原生代码开发中添加实现方法 javaWrapper.cpp 文件中添加如下方法
```
/**
 * Mat 转 Bitmap
 */
jobject mat_to_bitmap(JNIEnv * env, Mat & src, bool needPremultiplyAlpha, jobject bitmap_config){

    jclass java_bitmap_class = (jclass)env->FindClass("android/graphics/Bitmap");
    jmethodID mid = env->GetStaticMethodID(java_bitmap_class,
                                           "createBitmap", "(IILandroid/graphics/Bitmap$Config;)Landroid/graphics/Bitmap;");

    jobject bitmap = env->CallStaticObjectMethod(java_bitmap_class,
                                                 mid, src.size().width, src.size().height, bitmap_config);
    AndroidBitmapInfo  info;
    void* pixels = 0;

    try {
        //validate
        CV_Assert(AndroidBitmap_getInfo(env, bitmap, &info) >= 0);
        CV_Assert(src.type() == CV_8UC1 || src.type() == CV_8UC3 || src.type() == CV_8UC4);
        CV_Assert(AndroidBitmap_lockPixels(env, bitmap, &pixels) >= 0);
        CV_Assert(pixels);

        //type mat
        if(info.format == ANDROID_BITMAP_FORMAT_RGBA_8888){
            Mat tmp(info.height, info.width, CV_8UC4, pixels);
            if(src.type() == CV_8UC1){
                cvtColor(src, tmp, CV_GRAY2RGBA);
            } else if(src.type() == CV_8UC3){
                cvtColor(src, tmp, CV_RGB2RGBA);
            } else if(src.type() == CV_8UC4){
                if(needPremultiplyAlpha){
                    cvtColor(src, tmp, COLOR_RGBA2mRGBA);
                }else{
                    src.copyTo(tmp);
                }
            }

        } else{
            Mat tmp(info.height, info.width, CV_8UC2, pixels);
            if(src.type() == CV_8UC1){
                cvtColor(src, tmp, CV_GRAY2BGR565);
            } else if(src.type() == CV_8UC3){
                cvtColor(src, tmp, CV_RGB2BGR565);
            } else if(src.type() == CV_8UC4){
                cvtColor(src, tmp, CV_RGBA2BGR565);
            }
        }
        AndroidBitmap_unlockPixels(env, bitmap);
        return bitmap;
    } catch(cv::Exception e){
        AndroidBitmap_unlockPixels(env, bitmap);
        jclass je = env->FindClass("org/opencv/core/CvException");
        if(!je) je = env->FindClass("java/lang/Exception");
        env->ThrowNew(je, e.what());
        return bitmap;
    } catch (...){
        AndroidBitmap_unlockPixels(env, bitmap);
        jclass je = env->FindClass("java/lang/Exception");
        env->ThrowNew(je, "Unknown exception in JNI code {nMatToBitmap}");
        return bitmap;
    }
}

/**
 * 车牌号的详细信息
 * @param env
 * @param obj
 * @param matPtr
 * @param object_pr
 * @return
 */
JNIEXPORT jobject JNICALL
Java_pr_platerecognization_PlateRecognition_PlateInfoRecognization(
        JNIEnv *env, jobject obj,
        jlong matPtr, jlong object_pr) {
    jclass plateInfo_class = env -> FindClass("pr/platerecognization/PlateInfo");
    jmethodID mid = env->GetMethodID(plateInfo_class,"<init>","()V");
    jobject plateInfoObj   = env->NewObject(plateInfo_class,mid);

    pr::PipelinePR *PR = (pr::PipelinePR *) object_pr;
    cv::Mat &mRgb = *(cv::Mat *) matPtr;
    cv::Mat rgb;
    cv::cvtColor(mRgb,rgb,cv::COLOR_RGBA2BGR);

    //1表示SEGMENTATION_BASED_METHOD在方法里有说明
    std::vector<pr::PlateInfo> list_res= PR->RunPiplineAsImage(rgb,pr::SEGMENTATION_FREE_METHOD);
    std::string concat_results;
    pr::PlateInfo plateInfo;
    for(auto one:list_res)
    {
        //可信度
        if (one.confidence>0.7) {
            plateInfo = one;
            //车牌号
            jfieldID fid_plate_name  = env->GetFieldID(plateInfo_class,"plateName","Ljava/lang/String;");
            env->SetObjectField(plateInfoObj,fid_plate_name,env->NewStringUTF(plateInfo.getPlateName().c_str()));

            //识别区域
            Mat src = plateInfo.getPlateImage();

            jclass java_bitmap_class = (jclass)env->FindClass("android/graphics/Bitmap$Config");
            jmethodID bitmap_mid = env->GetStaticMethodID(java_bitmap_class,
                                                          "nativeToConfig", "(I)Landroid/graphics/Bitmap$Config;");
            jobject bitmap_config = env->CallStaticObjectMethod(java_bitmap_class, bitmap_mid, 5);

            jfieldID fid_bitmap = env->GetFieldID(plateInfo_class, "bitmap","Landroid/graphics/Bitmap;");
            jobject _bitmap = mat_to_bitmap(env, src, false, bitmap_config);
            env->SetObjectField(plateInfoObj,fid_bitmap, _bitmap);
            return plateInfoObj;
        }
    }
    return plateInfoObj;
}
```
