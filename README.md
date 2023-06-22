
# Cutils (Linux & Android)

System Properties based Cmake file that help you to import it to app project easily and config Kernel services and settings RealTime.






## Usage/Examples IN JNI

in this example, we Read the HDMI_IN status and enabled the audio route or disabled it for Kernel.


Test board :"FriendlyELEC rk3399-android-8.1"


```javascript
#include <jni.h>
#include <cutils/properties.h>

extern "C" {
JNIEXPORT void JNICALL
  Java_com_example_nouri_enableHdmiIn(JNIEnv *env, jclass clazz) {
      property_set("persist.audio.hdmiin.normal", "true");
  }

JNIEXPORT void JNICALL
  Java_com_example_nouri_disableHdmiIn(JNIEnv *env, jclass clazz) {
      property_set("persist.audio.hdmiin.normal", "false");
  }
}
JNIEXPORT jstring JNICALL
  Java_com_example_nouri_getStateHdmiIn(JNIEnv *env, jclass clazz) {
      char value[100];// __BIONIC_FORTIFY_UNKNOWN_SIZE
      property_get("persist.audio.hdmiin.normal", value, "NULL");
      return (*env).NewStringUTF(value);
  }

```


