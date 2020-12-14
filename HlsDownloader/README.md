# HlsDownloader
Merupakan custom library yang digunakan untuk mendownload video stream HLS yang berupa .m3u8, namanya juga HLS dan bukan .mp4 ya gaes :') .

Library ini digunakan secara internal (tidak untuk publik) di aplikasi Android RCTI+. Dokumentasi kodingan yang ada di sini ditulis menggunakan bahasa Kotlin (sorry ya kalo kamu masih pake Java, beralih ke Kotlin gih deh hehe \**peace*)

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

## Get Video Resolutions
Pertama-tama, sebelum memulai mengunduh video, kita harus mendapatkan daftar/list resolusi video yang terdapat di dalam file .m3u8, seperti 240p, 360p, 480p, 720p, dll, dan nantinya user diharuskan untuk memilih resolusi video apa yang diinginkan.

```kotlin
val envDir: String? = requireContext().getExternalFilesDir(Environment.DIRECTORY_DOWNLOADS)?.absolutePath
val packageName: String = requireContext().packageName
val videoType: String = "episode"  // bisa berupa: episode, extra, clip
val videoId: String = "1001"
val url: String = "https://abclimadasar.com/.../file.m3u8"

HlsDwld.getVideoRes(
    envDir = envDir,
    packageName = packageName,
    contType = videoType,
    contId = videoId,
    urlHls = url,
    ihd = object : IHlsDwld {
        override fun onSuccess(contentType: String?, contentId: String?) {}
        override fun onFailed(contentType: String?, contentId: String?, reason: String?) {}
        override fun onProgress(contentType: String?, contentId: String?, percent: Int) {}
        override fun onVideoResInfo(contentType: String?, contentId: String?, res: Array<String?>?) {
            // Tampilkan dialog yang menampilkan list resolusi video dari parameter res
        }
    }
}
```
Variable `envDir` merupakan path folder yang akan digunakan untuk menampung seluruh file dan sub-file dari video-video yang akan diunduh. Saya menyarankan sih pakai `getExternalFilesDir(Environment.DIRECTORY_DOWNLOADS)` dikarenakan folder internal aplikasi ini tidak akan muncul di Gallery dan isi foldernya akan terhapus otomatis ketika user meng-uninstall aplikasi sehingga memory storage user akan kembali lega (ini dulu pernah dapet komplen dari user, boss! makanya kita buat seperti ini hehe). LANJUT!

## Save Video
Setelah user memilih resolusi unduhan yang diinginkan, maka langkah selanjutnya adalah melakukan proses download. Berikut contoh potongan kodingannya:

```kotlin
val envDir = // ...
val packageName = // ...
val videoType = // ...
val videoId = // ...
val url = // ...    /* Untuk selebihnya saya ga perlu nulis ulang variable-variable ini ya, capek om! */
val resolutionOption = "240p"
val urlThumbnail = "https://abclimadasar.com/.../thumbnail.jpg"
val title = "Dunia Terbalik"
val extra: String? = null

HlsDwld.saveVideo(
    envDir = envDir,
    packageName = packageName,
    resOpt = resolutionOption,
    contType = videoType,
    contId = videoId,
    urlHls = url,
    urlThumbnail = urlThumbnail,
    title = videoTitle,
    extra = downloadExtraJson,
    ihd = object : IHlsDwld {
        override fun onSuccess(contentType: String?, contentId: String?) {
            // Jika video sudah berhasil terunduh
        }
        
        override fun onFailed(contentType: String?, contentId: String?, reason: String?) {
            // Jika video gagal terunduh, akan dibalikkan `reason` kenapa bisa gagal.
        }
        
        override fun onProgress(contentType: String?, contentId: String?, percent: Int) {
            // Progress dari unduhan video, akan dibalikkan berapa `percent` dari video yang sedang diunduh.
        }
        
        override fun onVideoResInfo(contentType: String?, contentId: String?, res: Array<String?>?) {
            // Ga ngelakuin apa-apa sih di sini, soalnya pake callback interface, jadinya semua functionnya harus di-consume hehe.
        }
    }
)
```
Nah, parameter function `extra` atau variable `downloadExtraJson` itu apa? Ini bisa digunakan untuk menyimpan informasi tambahan untuk video tersebut, seperti object model yang saya gunakan di bawah ini:

```kotlin
class DownloadExtra {
    @SerializedName("program_id")
    var programId: Int = 0

    @SerializedName("time_created")
    var timeCreated: Long = 0

    @SerializedName("share_link")
    var shareLink: String = ""

    @SerializedName("season")
    var season: Int = 0

    @SerializedName("episode")
    var episode: Int = 0
}
```
Jikalau tipe video yang diunduh adalah episode, maka kita bisa memasukkan informasi tambahan seperti `season` ke-berapa dan `episode` ke-berapa, selain dari tipe episode maka kedua variabel tersebut dapat diabaikan.

Selanjutnya model object tersebut dikonversi menjadi bentuk String, kita bisa menggunakan fungsi `Gson()` dari library mbah Google.
```kotlin
val downloadExtraJson = DownloadExtra().apply {
    programId = 12
    timeCreated = System.currentTimeMillis()
    season = 5
    episode = 3132
    shareLink = "https://abclimadasar.com/share/..."
    Gson().toJson(this, object : TypeToken<DownloadExtra>() {}.type)
}
```

## Resume Video
Ada kalanya video yang diunduh terjadi kegagalan, baik karena koneksi yang tiba-tiba mati atau tidak stabil. Oleh karena itu, kita tidak perlu mengulang kembali proses-proses sebelumnya, seperti **Get Video Resolutions** dan lalu **Save Video** kembali, namun cukup dengan memanggil 2 fungsi di bawah ini:
```kotlin
if (HlsDwld.wasSavingPaused(
        envDir = envDir,
        packageName = packageName,
        contType = videoType,
        contId = videoId)) {
    /* Video bisa di-resume. */
    HlsDwld.resumeSaving(
        envDir = envDir,
        packageName = packageName,
        contType = videoType,
        contId = videoId,
        ihd = object : IHlsDwld {
            override fun onSuccess(contentType: String?, contentId: String?) {}
            override fun onFailed(contentType: String?, contentId: String?, reason: String?) {}
            override fun onProgress(contentType: String?, contentId: String?, percent: Int) {}
            override fun onVideoResInfo(contentType: String?, contentId: String?, res: Array<String?>?) {}
        }
    )
} else {
    /* Unduh video seperti biasa. */
    ...
}
```

## Get Saved Videos
Untuk dapat melihat daftar video yang telah berhasil diunduh, maka dapat dilakukan seperti di bawah ini:
```kotlin
val downloadedDataJson: String? = HlsDwld.getSavedVideosInJson(
    envDir = envDir,
    packageName = packageName
)
```

Hasil yang dibalikkan dari fungsi tersebut adalah berupa String JSON dari daftar/list object model seperti di bawah ini:
```kotlin
class HlsDownloadModel {
    @SerializedName("resolution")
    var resolution: String = ""

    @SerializedName("content_type")
    var contentType: String = ""

    @SerializedName("content_id")
    var contentId: String = ""

    @SerializedName("hls_url")
    var hlsUrl: String = ""

    @SerializedName("hls_path")
    var hlsPath: String = ""

    @SerializedName("thumbnail_url")
    var thumbnailUrl: String = ""

    @SerializedName("thumbnail_path")
    var thumbnailPath: String = ""

    @SerializedName("title")
    var title: String = ""

    @SerializedName("extra")
    var extra: String = ""
}
```

Oleh karena itu, variable `downloadedDataJson` di atas harus dikonversi kembali menjadi bentuk ArrayList dari object model `HlsDownloadModel` tersebut.
```kotlin
if (downloadedDataJson?.isNotEmpty() == true) {
    val downloadedList = Gson().fromJson<MutableList<HlsDownloadModel>>(
        downloadedDataJson,
        object : TypeToken<MutableList<HlsDownloadModel>>() {}.type
    )
}
```

## Get Unsaved Videos
Selain melihat daftar video yang berhasil terunduh, ada kalanya kita menginginkan untuk melihat daftar video yang masih belum berhasil terunduh atau daftar video yang masih gagal. Kita dapat menggunakan fungsi seperti di bawah ini:
```kotlin
val resumableDownloadedDataJson: String? = HlsDwld.getResumableVideosInJson(
    envDir = envDir,
    packageName = packageName
)
if (resumableDownloadedDataJson?.isNotEmpty() == true) {
    val resumableDownloadList = Gson().fromJson<MutableList<HlsDownloadModel>>(
        resumableDownloadedDataJson,
        object : TypeToken<MutableList<HlsDownloadModel>>() {}.type
    )
}
```
