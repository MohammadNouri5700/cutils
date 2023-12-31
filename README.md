
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







## Comprehensive Test Class


```javascript
#define LOG_TAG "Properties_test"

#include <limits.h>

#include <iostream>
#include <sstream>
#include <string>

#include <android/log.h>
#include <android-base/macros.h>
#include <cutils/properties.h>
#include <gtest/gtest.h>

namespace android {

#define STRINGIFY_INNER(x) #x
#define STRINGIFY(x) STRINGIFY_INNER(x)
#define ASSERT_OK(x) ASSERT_EQ(0, (x))
#define EXPECT_OK(x) EXPECT_EQ(0, (x))

#define PROPERTY_TEST_KEY "libcutils.test.key"
#define PROPERTY_TEST_VALUE_DEFAULT "<<<default_value>>>"

template <typename T>
static std::string HexString(T value) {
    std::stringstream ss;
    ss << "0x" << std::hex << std::uppercase << value;
    return ss.str();
}

template <typename T>
static ::testing::AssertionResult AssertEqualHex(const char *mExpr,
        const char *nExpr,
        T m,
        T n) {
    if (m == n) {
        return ::testing::AssertionSuccess();
    }

    return ::testing::AssertionFailure()
        << mExpr << " and " << nExpr << " (expected: " << HexString(m) <<
        ", actual: " << HexString(n) << ") are not equal";
}

class PropertiesTest : public testing::Test {
public:
    PropertiesTest() : mValue() {}
protected:
    virtual void SetUp() {
        EXPECT_OK(property_set(PROPERTY_TEST_KEY, /*value*/NULL));
    }

    virtual void TearDown() {
        EXPECT_OK(property_set(PROPERTY_TEST_KEY, /*value*/NULL));
    }

    char mValue[PROPERTY_VALUE_MAX];

    template <typename T>
    static std::string ToString(T value) {
        std::stringstream ss;
        ss << value;

        return ss.str();
    }

    // Return length of property read; value is written into mValue
    int SetAndGetProperty(const char* value, const char* defaultValue = PROPERTY_TEST_VALUE_DEFAULT) {
        EXPECT_OK(property_set(PROPERTY_TEST_KEY, value)) << "value: '" << value << "'";
        return property_get(PROPERTY_TEST_KEY, mValue, defaultValue);
    }

    void ResetValue(unsigned char c = 0xFF) {
        for (size_t i = 0; i < arraysize(mValue); ++i) {
            mValue[i] = (char) c;
        }
    }
};

TEST_F(PropertiesTest, SetString) {

    // Null key -> unsuccessful set
    {
        // Null key -> fails
        EXPECT_GT(0, property_set(/*key*/NULL, PROPERTY_TEST_VALUE_DEFAULT));
    }

    // Null value -> returns default value
    {
        // Null value -> OK , and it clears the value
        EXPECT_OK(property_set(PROPERTY_TEST_KEY, /*value*/NULL));
        ResetValue();

        // Since the value is null, default value will be returned
        size_t len = property_get(PROPERTY_TEST_KEY, mValue, PROPERTY_TEST_VALUE_DEFAULT);
        EXPECT_EQ(strlen(PROPERTY_TEST_VALUE_DEFAULT), len);
        EXPECT_STREQ(PROPERTY_TEST_VALUE_DEFAULT, mValue);
    }

    // Trivial case => get returns what was set
    {
        size_t len = SetAndGetProperty("hello_world");
        EXPECT_EQ(strlen("hello_world"), len) << "hello_world key";
        EXPECT_STREQ("hello_world", mValue);
        ResetValue();
    }

    // Set to empty string => get returns default always
    {
        const char* EMPTY_STRING_DEFAULT = "EMPTY_STRING";
        size_t len = SetAndGetProperty("", EMPTY_STRING_DEFAULT);
        EXPECT_EQ(strlen(EMPTY_STRING_DEFAULT), len) << "empty key";
        EXPECT_STREQ(EMPTY_STRING_DEFAULT, mValue);
        ResetValue();
    }

    // Set to max length => get returns what was set
    {
        std::string maxLengthString = std::string(PROPERTY_VALUE_MAX-1, 'a');

        int len = SetAndGetProperty(maxLengthString.c_str());
        EXPECT_EQ(PROPERTY_VALUE_MAX-1, len) << "max length key";
        EXPECT_STREQ(maxLengthString.c_str(), mValue);
        ResetValue();
    }

    // Set to max length + 1 => set fails
    {
        const char* VALID_TEST_VALUE = "VALID_VALUE";
        ASSERT_OK(property_set(PROPERTY_TEST_KEY, VALID_TEST_VALUE));

        std::string oneLongerString = std::string(PROPERTY_VALUE_MAX, 'a');

        // Expect that the value set fails since it's too long
        EXPECT_GT(0, property_set(PROPERTY_TEST_KEY, oneLongerString.c_str()));
        size_t len = property_get(PROPERTY_TEST_KEY, mValue, PROPERTY_TEST_VALUE_DEFAULT);

        EXPECT_EQ(strlen(VALID_TEST_VALUE), len) << "set should've failed";
        EXPECT_STREQ(VALID_TEST_VALUE, mValue);
        ResetValue();
    }
}

TEST_F(PropertiesTest, GetString) {

    // Try to use a default value that's too long => get truncates the value
    {
        ASSERT_OK(property_set(PROPERTY_TEST_KEY, ""));

        std::string maxLengthString = std::string(PROPERTY_VALUE_MAX - 1, 'a');
        std::string oneLongerString = std::string(PROPERTY_VALUE_MAX, 'a');

        // Expect that the value is truncated since it's too long (by 1)
        int len = property_get(PROPERTY_TEST_KEY, mValue, oneLongerString.c_str());
        EXPECT_EQ(PROPERTY_VALUE_MAX - 1, len);
        EXPECT_STREQ(maxLengthString.c_str(), mValue);
        ResetValue();
    }

    // Try to use a default value that's the max length => get succeeds
    {
        ASSERT_OK(property_set(PROPERTY_TEST_KEY, ""));

        std::string maxLengthString = std::string(PROPERTY_VALUE_MAX - 1, 'b');

        // Expect that the value matches maxLengthString
        int len = property_get(PROPERTY_TEST_KEY, mValue, maxLengthString.c_str());
        EXPECT_EQ(PROPERTY_VALUE_MAX - 1, len);
        EXPECT_STREQ(maxLengthString.c_str(), mValue);
        ResetValue();
    }

    // Try to use a default value of length one => get succeeds
    {
        ASSERT_OK(property_set(PROPERTY_TEST_KEY, ""));

        std::string oneCharString = std::string(1, 'c');

        // Expect that the value matches oneCharString
        int len = property_get(PROPERTY_TEST_KEY, mValue, oneCharString.c_str());
        EXPECT_EQ(1, len);
        EXPECT_STREQ(oneCharString.c_str(), mValue);
        ResetValue();
    }

    // Try to use a default value of length zero => get succeeds
    {
        ASSERT_OK(property_set(PROPERTY_TEST_KEY, ""));

        std::string zeroCharString = std::string(0, 'd');

        // Expect that the value matches oneCharString
        int len = property_get(PROPERTY_TEST_KEY, mValue, zeroCharString.c_str());
        EXPECT_EQ(0, len);
        EXPECT_STREQ(zeroCharString.c_str(), mValue);
        ResetValue();
    }

    // Try to use a NULL default value => get returns 0
    {
        ASSERT_OK(property_set(PROPERTY_TEST_KEY, ""));

        // Expect a return value of 0
        int len = property_get(PROPERTY_TEST_KEY, mValue, NULL);
        EXPECT_EQ(0, len);
        ResetValue();
    }
}

TEST_F(PropertiesTest, GetBool) {
    /**
     * TRUE
     */
    const char *valuesTrue[] = { "1", "true", "y", "yes", "on", };
    for (size_t i = 0; i < arraysize(valuesTrue); ++i) {
        ASSERT_OK(property_set(PROPERTY_TEST_KEY, valuesTrue[i]));
        bool val = property_get_bool(PROPERTY_TEST_KEY, /*default_value*/false);
        EXPECT_TRUE(val) << "Property should've been TRUE for value: '" << valuesTrue[i] << "'";
    }

    /**
     * FALSE
     */
    const char *valuesFalse[] = { "0", "false", "n", "no", "off", };
    for (size_t i = 0; i < arraysize(valuesFalse); ++i) {
        ASSERT_OK(property_set(PROPERTY_TEST_KEY, valuesFalse[i]));
        bool val = property_get_bool(PROPERTY_TEST_KEY, /*default_value*/true);
        EXPECT_FALSE(val) << "Property shoud've been FALSE For string value: '" << valuesFalse[i] << "'";
    }

    /**
     * NEITHER
     */
    const char *valuesNeither[] = { "x0", "x1", "2", "-2", "True", "False", "garbage", "", " ",
            "+1", "  1  ", "  true", "  true  ", "  y  ", "  yes", "yes  ",
            "+0", "-0", "00", "  00  ", "  false", "false  ",
    };
    for (size_t i = 0; i < arraysize(valuesNeither); ++i) {
        ASSERT_OK(property_set(PROPERTY_TEST_KEY, valuesNeither[i]));

        // The default value should always be used
        bool val = property_get_bool(PROPERTY_TEST_KEY, /*default_value*/true);
        EXPECT_TRUE(val) << "Property should've been NEITHER (true) for string value: '" << valuesNeither[i] << "'";

        val = property_get_bool(PROPERTY_TEST_KEY, /*default_value*/false);
        EXPECT_FALSE(val) << "Property should've been NEITHER (false) for string value: '" << valuesNeither[i] << "'";
    }
}

TEST_F(PropertiesTest, GetInt64) {
    const int64_t DEFAULT_VALUE = INT64_C(0xDEADBEEFBEEFDEAD);

    const std::string longMaxString = ToString(INT64_MAX);
    const std::string longStringOverflow = longMaxString + "0";

    const std::string longMinString = ToString(INT64_MIN);
    const std::string longStringUnderflow = longMinString + "0";

    const char* setValues[] = {
        // base 10
        "1", "2", "12345", "-1", "-2", "-12345",
        // base 16
        "0xFF", "0x0FF", "0xC0FFEE",
        // base 8
        "0", "01234", "07",
        // corner cases
        "       2", "2      ", "+0", "-0", "  +0   ", longMaxString.c_str(), longMinString.c_str(),
        // failing cases
        NULL, "", " ", "    ", "hello", "     true     ", "y",
        longStringOverflow.c_str(), longStringUnderflow.c_str(),
    };

    int64_t getValues[] = {
        // base 10
        1, 2, 12345, -1, -2, -12345,
        // base 16
        0xFF, 0x0FF, 0xC0FFEE,
        // base 8
        0, 01234, 07,
        // corner cases
        2, 2, 0, 0, 0, INT64_MAX, INT64_MIN,
        // failing cases
        DEFAULT_VALUE, DEFAULT_VALUE, DEFAULT_VALUE, DEFAULT_VALUE, DEFAULT_VALUE, DEFAULT_VALUE, DEFAULT_VALUE,
        DEFAULT_VALUE, DEFAULT_VALUE,
    };

    ASSERT_EQ(arraysize(setValues), arraysize(getValues));

    for (size_t i = 0; i < arraysize(setValues); ++i) {
        ASSERT_OK(property_set(PROPERTY_TEST_KEY, setValues[i]));

        int64_t val = property_get_int64(PROPERTY_TEST_KEY, DEFAULT_VALUE);
        EXPECT_PRED_FORMAT2(AssertEqualHex, getValues[i], val) << "Property was set to '" << setValues[i] << "'";
    }
}

TEST_F(PropertiesTest, GetInt32) {
    const int32_t DEFAULT_VALUE = INT32_C(0xDEADBEEF);

    const std::string intMaxString = ToString(INT32_MAX);
    const std::string intStringOverflow = intMaxString + "0";

    const std::string intMinString = ToString(INT32_MIN);
    const std::string intStringUnderflow = intMinString + "0";

    const char* setValues[] = {
        // base 10
        "1", "2", "12345", "-1", "-2", "-12345",
        // base 16
        "0xFF", "0x0FF", "0xC0FFEE", "0Xf00",
        // base 8
        "0", "01234", "07",
        // corner cases
        "       2", "2      ", "+0", "-0", "  +0   ", intMaxString.c_str(), intMinString.c_str(),
        // failing cases
        NULL, "", " ", "    ", "hello", "     true     ", "y",
        intStringOverflow.c_str(), intStringUnderflow.c_str(),
    };

    int32_t getValues[] = {
        // base 10
        1, 2, 12345, -1, -2, -12345,
        // base 16
        0xFF, 0x0FF, 0xC0FFEE, 0Xf00,
        // base 8
        0, 01234, 07,
        // corner cases
        2, 2, 0, 0, 0, INT32_MAX, INT32_MIN,
        // failing cases
        DEFAULT_VALUE, DEFAULT_VALUE, DEFAULT_VALUE, DEFAULT_VALUE, DEFAULT_VALUE, DEFAULT_VALUE, DEFAULT_VALUE,
        DEFAULT_VALUE, DEFAULT_VALUE,
    };

    ASSERT_EQ(arraysize(setValues), arraysize(getValues));

    for (size_t i = 0; i < arraysize(setValues); ++i) {
        ASSERT_OK(property_set(PROPERTY_TEST_KEY, setValues[i]));

        int32_t val = property_get_int32(PROPERTY_TEST_KEY, DEFAULT_VALUE);
        EXPECT_PRED_FORMAT2(AssertEqualHex, getValues[i], val) << "Property was set to '" << setValues[i] << "'";
    }
}

} // namespace android
```
