# HlsDownloader
Merupakan custom library yang digunakan untuk mendownload video stream HLS yang berupa .m3u8, namanya juga HLS dan bukan .mp4 ya gaes :')

## Get Started
1. Letakkan file AAR library seperti contoh di folder `{PROJECT_KAMU}/app/libs`.
2. Buka file `build.gradle` dari app-level di project kamu, lalu tuliskan seperti di bawah ini
```kotlin
dependencies {
    implementation fileTree(dir: 'libs', include: ['*.jar'])
    implementation(name: 'HlsDwld-2.0.1', ext: 'aar')
}
```
3. Sync project kamu ya gaes, gitu doang kok, udah ngerti pasti lah ya :')
