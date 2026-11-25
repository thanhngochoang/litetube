# Thiết kế: LiteTube — app Android tải video/MP3 từ YouTube, có trim nhạc

Ngày: 2026-11-25 · Trạng thái: bản nháp chờ duyệt

## 1. Mục đích

Một app Android cho phép dán (hoặc chia sẻ) link YouTube rồi tải về dưới dạng
video hoặc âm thanh (MP3, M4A), và cắt một đoạn nhạc (trim) để lưu riêng — ví
dụ làm nhạc chuông hoặc giữ đoạn điệp khúc.

Lấy cảm hứng từ [Tyrrrz/YoutubeDownloader](https://github.com/Tyrrrz/YoutubeDownloader)
(app desktop C#). Không port code; chỉ giữ ý tưởng: một thư viện lấy link
stream trực tiếp từ YouTube + một bước xử lý media cục bộ.

Hai tiêu chí người dùng đặt ra:
- **Phải nhẹ.**
- **Phải có MP3** (không chỉ M4A) và **có trim**.

## 2. Mục tiêu / Không làm / Thất bại nếu

**Mục tiêu (v1)**
- Nhận link YouTube (dán tay, hoặc từ nút Share của app YouTube/trình duyệt).
- Hiện tiêu đề, kênh, thumbnail, thời lượng; cho chọn định dạng:
  - Âm thanh: **MP3** (128 / 192 / 320 kbps), **M4A** (giữ nguyên, không mã hoá lại).
  - Video: MP4 ở các độ phân giải có sẵn, tối đa 1080p.
- Tải nền, có thông báo tiến độ, huỷ được; tải xong lưu vào bộ nhớ chung của máy.
- Màn hình trim: chọn điểm đầu/cuối, nghe thử, xuất ra MP3 hoặc M4A.
- MP3 có tag ID3 (tiêu đề, nghệ sĩ = tên kênh, ảnh bìa = thumbnail).

**Không làm ở v1**
- iOS, Play Store, App Store.
- Playlist / tải hàng loạt, video trên 1080p (VP9/AV1/WebM), phụ đề.
- Video cần đăng nhập: giới hạn tuổi, members-only, private.
- Trình phát nhạc/video đầy đủ, các trang khác ngoài YouTube.
- Trim video.

**Thất bại nếu** (dù vẫn chạy được)
- APK vượt quá ~15 MB.
- Cần root, Termux, hay cài thêm app khác.
- Có quảng cáo, tracking, hoặc gửi dữ liệu người dùng đi bất kỳ đâu. (Kết nối
  mạng chỉ tới YouTube, và tới GitHub để kiểm tra bản cập nhật — xem 5.7.)

*Ba ý trên là đề xuất của mình, suy từ tiêu chí "nhẹ".*

## 3. Ràng buộc

- **Android không có bộ mã hoá MP3 sẵn** — chỉ giải mã được. App phải kèm một bộ.
- **YouTube không cung cấp MP3.** Âm thanh có sẵn là AAC/M4A (itag 140, ~128 kbps)
  và Opus/WebM. MP3 luôn là mã hoá lại từ AAC, nên chọn 320 kbps không làm tiếng
  hay hơn nguồn 128 kbps — UI phải nói rõ điều này (gợi ý mặc định 192 kbps).
- **Video trên 360p là hai luồng riêng** (hình + tiếng), phải ghép lại.
- **YouTube thay đổi thường xuyên** → thư viện lấy link sẽ hỏng định kỳ; phải
  cập nhật app được dễ dàng.
- **Phân phối ngoài store** (APK sideload) vì vi phạm điều khoản YouTube.
- **LAME là LGPL** → link động (`.so` riêng), giữ thông báo bản quyền trong
  màn hình "Giới thiệu". Bằng sáng chế MP3 đã hết hạn (2017).
- **Android 15+ yêu cầu thư viện native căn chỉnh trang 16 KB** → build LAME
  bằng NDK r27 trở lên.

## 4. Các hướng đã cân nhắc

| Hướng | MP3 bằng gì | Dung lượng thêm (ước lượng) | Đánh đổi |
|---|---|---|---|
| **A. LAME + công cụ có sẵn của Android** ✅ | MediaCodec giải mã AAC → LAME mã hoá MP3 | < 1 MB | Nhẹ nhất. Phải tự viết đoạn nối giải mã → mã hoá |
| B. FFmpeg (fork `ffmpegkit-maintained`) | FFmpeg + libmp3lame | ~15–50 MB | Ít code nhất nhưng nặng; FFmpegKit gốc đã ngừng 2025, chưa rõ fork nào có kèm LAME |
| C. `youtubedl-android` (yt-dlp + Python + FFmpeg) | FFmpeg trong gói | ~50–100 MB | Bền nhất trước thay đổi của YouTube, nhưng trái hẳn tiêu chí nhẹ |

**Đã chọn: A** (người dùng chọn). Lý do: là hướng duy nhất giữ APK ở mức vài MB
mà vẫn có MP3. B và C dồn dung lượng vào FFmpeg, trong khi mọi việc khác
(giải mã, ghép video, cắt M4A) Android đã làm được sẵn.

## 5. Thiết kế

### 5.1 Công nghệ

| Phần | Lựa chọn | Ghi chú |
|---|---|---|
| Nền tảng | Android, Kotlin | *Đã chốt* |
| minSdk / targetSdk | 29 (Android 10) / 35 | *Lựa chọn của mình*: từ API 29 lưu file qua MediaStore không cần quyền bộ nhớ |
| UI | Jetpack Compose + Material 3 | *Lựa chọn của mình*: ~1.5–2 MB sau R8, đổi lại viết nhanh hơn nhiều so với View |
| Lấy link YouTube | **NewPipeExtractor** (Java thuần) | Kéo theo Rhino (JS engine) để giải mã chữ ký, ~1–2 MB |
| HTTP | OkHttp | NewPipeExtractor cần một lớp `Downloader` tự cài đặt |
| Mã hoá MP3 | **LAME 3.100**, build bằng NDK, lớp JNI tự viết | Không dùng wrapper cũ (TAndroidLame…) vì đã ngừng cập nhật |
| Giải mã / cắt / ghép | `MediaExtractor`, `MediaCodec`, `MediaMuxer` (có sẵn) | Không thư viện ngoài |
| Nghe thử khi trim | `MediaPlayer` (có sẵn) | Không dùng ExoPlayer để tiết kiệm ~1.5 MB |
| Tải nền | Foreground service + notification | |
| ABI | `arm64-v8a`, `armeabi-v7a` | Chia APK theo ABI |

**Ngân sách dung lượng:** mục tiêu < 10 MB mỗi APK, tối đa 15 MB.

### 5.2 Các khối chính

```
 Link / Share intent
        │
        ▼
 ┌──────────────┐   StreamInfo (tiêu đề, thumbnail, danh sách luồng)
 │  Extractor   │── NewPipeExtractor + OkHttp
 └──────┬───────┘
        ▼
 ┌──────────────┐   tải luồng về file tạm trong cache
 │  Downloader  │── OkHttp, tải theo khối Range
 └──────┬───────┘
        ▼
 ┌──────────────┐   M4A: copy nguyên   MP3: AAC→PCM→LAME   Video: ghép hình+tiếng
 │  Processor   │── MediaExtractor / MediaCodec / MediaMuxer / LAME (JNI)
 └──────┬───────┘
        ▼
 ┌──────────────┐   ghi vào MediaStore (Music/… hoặc Movies/…), xoá file tạm
 │   Storage    │
 └──────────────┘
```

Trim dùng lại `Processor` với khoảng [đầu, cuối] trên một file đã có.

### 5.3 Luồng tải

**Bước 0 — kiểm chứng trước khi làm gì khác.** Từ 2025 YouTube hay đòi PO token
(chặn link stream bằng lỗi 403 hoặc bóp băng thông). App NewPipe xử lý bằng
một bước WebView riêng; bản thân thư viện NewPipeExtractor không làm việc đó.
Việc đầu tiên của kế hoạch là một bản thử nhỏ: NewPipeExtractor + OkHttp tải
trọn một luồng AAC và một luồng video 1080p trên máy thật. Nếu bị chặn, thêm
một khối "PO token" dùng WebView có sẵn của hệ thống (gần như không tăng dung
lượng), làm theo cách của app NewPipe. *Lựa chọn của mình: chấp nhận WebView
nếu cần, vì đó là cách duy nhất còn giữ được app nhẹ.*

1. **Lấy thông tin:** NewPipeExtractor trả về `StreamInfo`. App chọn luồng:
   - Âm thanh: ưu tiên M4A/AAC (itag 140). Opus chỉ dùng khi không có AAC;
     khi đó ẩn lựa chọn M4A, chỉ cho MP3 (Android giải mã được Opus từ API 29).
   - Video: luồng video-only MP4/H.264 (avc1) đúng độ phân giải đã chọn, kèm
     luồng AAC. Nếu chọn 360p và có luồng gộp sẵn (itag 18) thì tải thẳng.
2. **Tải:** chia file thành các khối (~10 MB) bằng header `Range` — YouTube bóp
   băng thông khi tải một request liền. Tiến độ tính theo byte. Huỷ thì xoá file tạm.
3. **Xử lý:**
   - **M4A:** không cần xử lý, chỉ đổi tên/ghi metadata nếu được.
   - **MP3:** xem 5.4.
   - **Video:** `MediaMuxer` (MP4) chép các mẫu H.264 và AAC vào một file, không
     mã hoá lại.
4. **Lưu:** ghi qua MediaStore. Âm thanh vào `Music/<TênApp>/`, video vào
   `Movies/<TênApp>/`. Tên file = tiêu đề video, bỏ ký tự không hợp lệ.

### 5.4 Chuyển sang MP3 (dùng chung cho tải và trim)

```
MediaExtractor (track AAC) → MediaCodec decoder → PCM 16-bit
    → [cắt theo khoảng thời gian, fade nếu có] → LAME (JNI) → file .mp3
    → ghi ID3v2 (tiêu đề, nghệ sĩ, ảnh bìa)
```

- Giữ nguyên sample rate và số kênh của nguồn (thường 44.1 kHz stereo); LAME
  tự xử lý. Mã hoá CBR ở bitrate người dùng chọn.
- Chạy trên một thread nền, báo tiến độ theo thời lượng đã xử lý.
- Lớp JNI chỉ cần ba hàm: `init(sampleRate, channels, bitrate)`,
  `encode(short[] pcm) → byte[]`, `flush() → byte[]`.
- Ghi ID3v2.3 bằng một hàm nhỏ tự viết (vài chục dòng), không thêm thư viện.

### 5.5 Trim

**Đầu vào:** file trong danh sách đã tải của app, hoặc một file âm thanh bất kỳ
chọn từ máy (qua trình chọn file của hệ thống).

**Giao diện:**

```
┌───────────────────────────────────┐
│  Tên bài hát                      │
│  ▁▃▅▇▅▃▂▃▆▇▇▅▃▂▁▂▄▆▇▆▄▂▁▃▅▇▅▃▂   │  ← dạng sóng
│      [■■■■■■■■■■■■]               │  ← vùng chọn, kéo hai đầu
│  Bắt đầu 00:42.300  Kết thúc 01:12.800  (30.5 s) │
│  [−0.1s][+0.1s]      [−0.1s][+0.1s]   │
│  ▶ Nghe thử   ☐ Fade in  ☐ Fade out    │
│  Xuất: (•) MP3 192 kbps  ( ) M4A       │
│              [ Lưu ]                   │
└───────────────────────────────────┘
```

- **Dạng sóng:** giải mã cả file một lần, lấy giá trị đỉnh theo từng ô (~1000
  ô), cache lại. Chỉ để nhìn, không cần chính xác.
- **Nghe thử:** `MediaPlayer` phát từ điểm đầu, dừng ở điểm cuối.
- **Xuất MP3:** chạy đường 5.4 với khoảng [đầu, cuối]; chính xác tới từng mẫu
  âm thanh (bỏ mẫu PCM nằm ngoài khoảng). Fade in/out: tăng/giảm âm lượng tuyến
  tính trong 1–3 giây trên PCM.
- **Xuất M4A:** chép nguyên các khung AAC trong khoảng bằng `MediaMuxer`, không
  mã hoá lại → nhanh, không mất chất lượng, chính xác ~23 ms (một khung AAC).
  Fade không áp dụng cho M4A (cần mã hoá lại); UI ẩn ô fade khi chọn M4A.
- **File nguồn không phải AAC** (MP3, Opus, FLAC… — mọi định dạng Android giải
  mã được): chỉ cho xuất MP3, ẩn lựa chọn M4A. *Lựa chọn của mình: không làm
  thêm đường mã hoá AAC ở v1, để chỉ có một đường mã hoá duy nhất (LAME).*
- File trim được lưu thành file mới (`<tên gốc> (trim).mp3`), không ghi đè gốc.

### 5.6 Các màn hình

1. **Trang chính:** ô dán link + nút "Lấy thông tin"; bên dưới là danh sách đã
   tải (bấm vào → mở / chia sẻ / trim / xoá).
2. **Chọn định dạng** (bottom sheet sau khi lấy thông tin): thumbnail, tiêu đề,
   thời lượng; tab Âm thanh / Video; nút Tải.
3. **Trim** (như trên).
4. **Cài đặt / Giới thiệu:** bitrate MP3 mặc định, phiên bản app + phiên bản
   NewPipeExtractor, thông báo giấy phép (LAME LGPL, NewPipeExtractor GPLv3).

Mở app qua nút Share có link YouTube → vào thẳng bước 2.

### 5.7 Lỗi và cập nhật

- Lỗi lấy link (YouTube đổi cơ chế, video bị chặn, cần đăng nhập): hiện thông
  báo rõ ràng theo loại lỗi, kèm gợi ý "cập nhật app".
- Mất mạng giữa chừng: thử lại khối đang tải tối đa 3 lần, rồi báo lỗi.
- Hết bộ nhớ: kiểm tra dung lượng trống trước khi tải (ước theo kích thước luồng).
- **Cập nhật:** phát hành APK trên GitHub Releases của repo công khai
  `thanhngochoang/litetube`. App gọi API công khai
  `GET /repos/thanhngochoang/litetube/releases/latest` (không cần token)
  khi mở, tối đa 1 lần/ngày, so phiên bản và hiện nút tải bản mới.

### 5.8 Phân phối và giấy phép

App **được chia sẻ cho người khác** (người dùng đã quyết định), mã nguồn công
khai trên GitHub ở tài khoản `thanhngochoang`.

- **Giấy phép GPLv3** cho toàn bộ app (bắt buộc vì NewPipeExtractor là GPLv3).
  LAME (LGPL) tương thích. Màn hình "Giới thiệu" liệt kê giấy phép các thư viện.
- **Khoá ký APK:** một khoá release cố định do chủ repo giữ, không bao giờ đưa
  vào repo. Mọi bản phát hành phải ký cùng khoá này để cài đè được.
- **Phát hành:** GitHub Actions build APK khi đẩy tag `vX.Y.Z`, ký bằng khoá lưu
  trong GitHub Secrets, đính kèm các APK theo ABI vào Release.
- README ghi rõ: app không liên kết với YouTube/Google, người dùng tự chịu trách
  nhiệm với nội dung mình tải.

## 6. Để lại cho người làm

- Cấu trúc package, kiến trúc trong app (ViewModel, DI hay không…), cách tổ chức test.
- Kích thước khối tải, số kết nối song song.
- Cách vẽ dạng sóng và chi tiết giao diện (màu, icon, animation).
- Cách build LAME (CMake trong project hay dùng AAR có sẵn), miễn là căn trang 16 KB.
- Tên app và package id.
