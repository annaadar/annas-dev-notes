---
title: "Native Interop in Android: OCR, Tesseract and JNI"
description: Why I bypassed cloud-based OCR for a privacy-focused medical filing app.
publishDate: "2026-04-19T11:23:00Z"
---

### Choosing An OCR Engine

Designing an OCR Android app can be tricky, especially when choosing an OCR solution. An obvious choice may seem to be [Google ML Kit's API](https://developers.google.com/ml-kit/vision/text-recognition/v2/android) - it has a free tier, is easy to use, reliable, and performant. However, some scenarios call for different solutions. If your app handles private medical files or texts, you likely don’t want to send those to a third-party server. Also, if your app targets a non-Latin language such as Hebrew, Greek, or Russian, the Google ML Kit free tier won’t have support for it. Choosing an open-source engine that runs locally will let your app avoid compromising users’ sensitive data and may support a wider variety of languages.

#### The Open Source Option - Tesseract

One of the most established names in the OCR ecosystem is [Tesseract](https://github.com/tesseract-ocr/tesseract). Developed in C++ and originally distributed as proprietary software by Hewlett-Packard in the 1980s, it was released as open-source in 2005. Google led its development for over a decade, modernizing it with an LSTM-based neural network engine. Today, 20+ years after its original development, it remains the go-to engine for free, offline, and highly customizable text recognition. It does have some limitations, such as not being as accurate as AI-based solutions, lacking handwriting recognition, and requiring additional development time to implement. Even so, I chose it for my OCR Privacy-First application.

#### Tesseract On Android

Implementing Tesseract on Android is not as straightforward as just adding a dependency. Since the engine is written in C++, bridging it with Kotlin requires using Android’s Native Development Kit (NDK) and the Java Native Interface (JNI). [Tesseract4Android](https://github.com/adaptech-cz/Tesseract4Android), an open-source Tesseract Android library, can handle this bridge for you - but understanding how it works under the hood is valuable.

### JNI

JNI, which stands for Java Native Interface, is the standard mechanism that allows the bytecode compiled by the Android system from managed code (Java/Kotlin) to interact with native C/C++ code and vice versa. It is a core part of the Android NDK, and allows you to reuse existing C/C++ libraries instead of rewriting them in another, managed language.

#### JNI Implementation

:::note
The following code snippets are minimalistic examples of JNI implementation for kotlin+Cpp bridging, and are not meant for production usage.
:::

To implement JNI and bridge C++ libraries with Kotlin, we start by loading the shared library. This is typically done within a companion object using `System.loadLibrary` to ensure the native code is linked when the class is first loaded.

```kotlin
class NativeLib {
    companion object {
        init {
            // loading "libnative-lib.so"
            System.loadLibrary("native-lib")
        }
    }

    // This method is marked 'external' and has no body since its body is implemented in C++
    external fun sayHello(): String
}
```

- The `external` keyword is used to assure the compiler the implementation of the method is implemented elsewhere, externally, and will be linked at runtime.

After declaring the method in Kotlin, you provide the corresponding implementation in C++. The function name must follow the JNI naming convention: Java_PackageName_ClassName_MethodName.

```cpp
#include <jni.h>
#include <string>

extern "C" JNIEXPORT jstring JNICALL
Java_com_example_myapp_NativeLib_sayHello(JNIEnv* env, jobject /* this */) {
    std::string hello = "Hello from C++";
    return env->NewStringUTF(hello.c_str());
}
```

A few notes -

- the `extern "C"` is required to prevent C++ name mangling. Otherwise the compiler could change it at compile time and the function will be invisible to the Java linker.
- `jstring` is the JNI version of a Java String. You cannot return a standard C++ std::string directly to Java. instead, you wrap it in a jstring. Likewise, there are also jboolean, jint, jlong, etc. We will see them later on in the tesseract4android's source code.
- When the Android app first runs System.loadLibrary the JVM scans the .so library. If a method isn't exported, it won't be seen by the JVM. To essentially export it, you add the `JNIEXPORT` macro.
- The `JNICALL` macro ensures the function is invoked correctly with the appropriate compiler directives, according to different platforms and operating systems. Although it may not be needed for the Android platform (as it does not require any special keywords for invoking methods) It is a best practice to include it.
- The `JNIEnv` pointer provides access to most of the JNI functions needed to manipulate Java objects, classes, and methods. Your native functions receive a JNIEnv as the first argument, which points to a table of function pointers. For example, when you call NewStringUtf your actually performing a lookup. You look for the NewStringUtf in the env functions table and jump to that address. Then, you execute the code that transforms your C++ characters into a Java-compatible object.

Now that I covered the fundamentals I'll deep dive into Tesseract4android source in order to understand the android bridging better.

#### JNI + Tesseract

In the tesseract4android repo, under `Tesseract4Android/tesseract4android/src/main`, there are two interesting directories - /cpp/tessreact, and /java/com/googlecode/tesseract. these two include the JNI implementation for the C++ library and java bridging.
Starting form the Android implementation, let's review the TessBaseApi.java file. as you can infer from its name, it include the base class that will be linked to the cpp code. For the purpose of this article, these are the most important bits of this file -
The libraries being loaded and nativeClassInit called -

```java title="TessBaseApi.java"
// Adapted from tesseract4android by adaptech-cz
// Licensed under the Apache License, Version 2.0
// Original source: https://github.com/adaptech-cz/tesseract4android
public class TessBaseAPI {
	/**
	 * Used by the native implementation of the class.
	 */
	private long mNativeData;

	static {
		System.loadLibrary("jpeg");
		System.loadLibrary("pngx");
		System.loadLibrary("leptonica");
		System.loadLibrary("tesseract");

		nativeClassInit();
	}
```

nativeClassInit is an external method without a body, since its body is implemented in the cpp section.
Inside the TessBaseApi class, all the library`s native, external methods, that are implemented in cpp, are declared. Including nativeClassInit and other methods. Some for library initialization and some are the core methods that allow the developer to perform ocr in an Android app.

```java title="TessBaseApi.java"
// Adapted from tesseract4android by adaptech-cz
// Licensed under the Apache License, Version 2.0
// Original source: https://github.com/adaptech-cz/tesseract4android
	// ******************
	// * Native methods *
	// ******************

	/**
	 * Initializes static native data. Must be called on object load.
	 */
	private static native void nativeClassInit();

	/**
	 * Initializes native data. Must be called on object construction.
	 */
	private native long nativeConstruct();

	/**
	 * Calls End() and finalizes native data. Must be called on object destruction.
	 */
	private native void nativeRecycle(long mNativeData);
	//
    //.. more native methods
    //

	private native String nativeGetUTF8Text(long mNativeData);

	private native int nativeMeanConfidence(long mNativeData);


```

- In this code snippet, multiple native methods are declared. This is essentially the core of the library. In the original source code 36 native methods are declared. Since the method declaration is quite repetitive, I provided only a few examples. In here, you can spot some familiar methods - the `meanConfidence` which returns a mean confidence score of the last OCR operation performed, and the `GetUTF8Text` which returns the extracted text.

Let's focus specifically on `GetUTF8Text`. As an Android developer who added the Tesseract4Android dependency, you would never use it directly. Instead, you would call `getUTF8Text` on your TessBaseApi class instance. Let's check this method's source.

```java title="TessBaseApi.java"
	/**
	 * The recognized text is returned as a String which is coded as UTF8.
	 * This is a blocking operation that will not work with {@link #stop()}.
	 * Call {@link #getHOCRText(int)} before calling this function to
	 * interrupt a recognition task with {@link #stop()}
	 *
	 * @return the recognized text
	 */
	@WorkerThread
	public String getUTF8Text() {
		if (mRecycled)
			throw new IllegalStateException();

		// Trim because the text will have extra line breaks at the end
		String text = nativeGetUTF8Text(mNativeData);

		return text != null ? text.trim() : null;
	}
```

The method acts as a wrapper, or a getter, for the native, private, nativeGetUTF8Text method. It calls it inside a worker thread, handles any errors, and returns the result.
Now, let's see what happens in the cpp code when we call getUTF8Text.

```cpp title=TessBaseApi.cpp
jstring Java_com_googlecode_tesseract_android_TessBaseAPI_nativeGetUTF8Text(JNIEnv *env,
                                                                            jobject thiz,
                                                                            jlong mNativeData) {

  native_data_t *nat = (native_data_t*) mNativeData;
  nat->initStateVariables(env, &thiz);

  char *text = nat->api.GetUTF8Text();

  jstring result = env->NewStringUTF(text);

  free(text);
  nat->resetStateVariables();

  return result;
}
// ...
  void initStateVariables(JNIEnv* env, jobject *object) {
    cancel_ocr = false;
    cachedEnv = env;
    cachedObject = object;
    lastProgress = 0;
  }

```

- the function name is according to the JNI method naming conventions.
- The code casts the jlong back into a native_data_t cpp pointer. This structure holds the Tesseract API instance.
- initStateVariables prepares the environment, setting the app's state variables with default values and the current api object.
- the native version of GetUTF8Text is called, actually triggering OCR on the Tesseract native library! If successful, we get back a string of the extracted text. The string is first stored in a char, then transformed into string using the NewStringUTF from the JNIEnv env reference.
  The jstring tranformation is needed in order to send back the text to the invoker of the method, the jvm thread.

That's a full circle on invoking cpp methods + utilizing native libraries from an Android app! There's still much more i haven't covered, but that's not in the scope of the article.

Nevertheless, enjoy some extra read on another JNI-like mechanism
:::note[Extra Read]
React Native also has its own JNI-like system, which allows JavaScript to interact with C++ or Kotlin/Swift code. This mechanism is part of the new RN Bridgeless Architecture, an approach that aims to simplify communication between JavaScript and native code by removing the traditional async bridge. The system is mainly useful for accessing specific iOS or Android features or running heavier operations that may not perform efficiently in JavaScript alone. JavaScript can hold a reference to a C++ object and vice versa. With this reference, you can directly invoke methods without serialization costs. The interface also uses JNI for its Java interop system.
Likely inspired by JNI, the React Native team named their bridge the [JSI](https://reactnative.dev/architecture/landing-page#fast-javascriptnative-interfacing).
:::

## Resources

- [Tesseract history](https://static.googleusercontent.com/media/research.google.com/iw//pubs/archive/33418.pdf) - "An Overview of the Tesseract OCR Engine." by Ray Smith, 2007.
- [Tesseract on GitHub](https://github.com/tesseract-ocr/tesseract)
- [Tesseract4android on Github](https://github.com/adaptech-cz/Tesseract4Android)
- [JNI on the official Android documentation](https://developer.android.com/ndk/guides/jni-tips) -[JSI on the React Native Docs](https://reactnative.dev/architecture/landing-page)
