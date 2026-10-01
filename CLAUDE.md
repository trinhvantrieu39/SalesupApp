# CLAUDE.md

Dự án **SalesUp VifonQL**: bản nâng cấp app quản lý bán hàng VifonQL từ **React Native 0.61.5 → 0.75.5**.
Repo này là khung RN 0.75.5 sạch (tạo bằng `@react-native-community/cli`). Code nghiệp vụ được port dần từ dự án cũ.

## Dự án nguồn (bản cũ)

- Đường dẫn: `../VifonQl` (`D:\Work\GESO\Native\VifonQl`). **Chỉ đọc, không sửa** dự án này.
- RN 0.61.5, React 16.9, JavaScript (không TS), entry: `index.js` → `src/App.js` → `App.start()`.
- Cấu trúc `src/`: `app-navigation/` (wrapper cho Wix react-native-navigation, `screenHOC`, `NavigatorMap`), `screens/` (`Geso`, `home`, `login`, `overlay`, `ScreenIDs.js`), `redux/`, `stores/` (account, app, auth, home), `sagas/`, `common-components/`, `theme/`, `assets/` (fonts, icon), `utils/`.
- Native Android cũ: `android/app/src/main/java/geso/salesupvifonqlnew/*.java`, link thủ công trong `settings.gradle`, AGP 4.2.2.
- File `App.js` ở thư mục gốc dự án cũ là file demo, **không dùng**.

## Môi trường & lệnh

- Node 20 (`engines >=18`), **Yarn 3.6.4** (Berry, qua `packageManager`, bật bằng `corepack enable`). Không dùng npm hay yarn 1.
- JDK 17, Gradle 8.8, AGP theo RN 0.75, Kotlin 1.9.25, NDK 26.1.10909125.
- Android: `minSdk 23`, `compileSdk/targetSdk 34`. Google Play yêu cầu target 35, nên cân nhắc nâng lên 35 trước khi phát hành.
- `newArchEnabled=false` (giữ Old Architecture cho tới khi mọi thư viện đều hỗ trợ), `hermesEnabled=true`.

```bash
yarn install
yarn start               # Metro
yarn android             # build + chạy Android
yarn ios                 # (macOS) cần: cd ios && bundle install && bundle exec pod install
yarn lint
yarn test
cd android && ./gradlew clean
cd android && ./gradlew assembleRelease
```

## Định danh app (bắt buộc giữ nguyên để cập nhật đè lên bản cũ trên store)

- `applicationId` / `namespace`: `geso.salesupvifonqlnew`.
- `versionCode` bản cũ là `20250828`, nên bản mới **phải lớn hơn** (giữ format `yyyyMMdd`). Bản cũ có `versionCodeOverride` theo ABI, cần kiểm tra lại khi bật split APK.
- Phải ký bằng **đúng release keystore của bản cũ**. Không commit keystore hoặc mật khẩu: `.gitignore` đã chặn `*.keystore` (chỉ cho phép `debug.keystore`). Mật khẩu để trong `~/.gradle/gradle.properties` hoặc biến môi trường.
- Bundle ID iOS: lấy theo dự án cũ khi port phần iOS.

## Nguyên tắc nâng cấp

1. **Không copy nguyên `android/` và `ios/` từ dự án cũ.** Giữ template của 0.75 và chỉ port thủ công từng phần: permission và `meta-data` (Google Maps API key…) trong `AndroidManifest.xml`, `strings.xml`, icon/splash, font, cấu hình signing, và code native riêng nếu có. Native code mới viết bằng Kotlin (`MainActivity.kt`, `MainApplication.kt`).
2. **Autolinking**: không thêm `include ':lib'` thủ công vào `settings.gradle`, không `new XxxPackage()` trong `MainApplication` (trừ thư viện yêu cầu riêng).
3. **Port theo từng giai đoạn**, và mỗi giai đoạn phải build chạy được trước khi sang giai đoạn tiếp:
   1. Hạ tầng: theme, assets/fonts, utils, redux store/persist, sagas, API (axios).
   2. Navigation và màn hình login.
   3. Home rồi đến các module trong `screens/Geso` (từng module một).
   4. Tính năng native: bản đồ, định vị, camera/ảnh, sinh trắc học, cập nhật OTA.
4. Port code JS nguyên trạng trước, chỉ sửa những gì cần để chạy trên RN 0.75 / React 18. **Không refactor nghiệp vụ** khi chưa được yêu cầu. Có thể giữ `.js`; file mới có thể viết `.tsx`.
5. Mỗi thư viện cũ phải được quyết định rõ: **nâng version**, **thay thế**, hoặc **bỏ** (xem bảng dưới). Ghi lại thay đổi API ở nơi gọi.
6. Kiểm tra dữ liệu người dùng khi cập nhật đè: redux-persist 4 lưu với key dạng `reduxPersist:*`, còn bản 6 dùng `persist:root`. Cần quyết định migrate hay chấp nhận cho người dùng đăng nhập lại.

## Thay đổi breaking cần chú ý khi port code

- **React 16.9 → 18**: `componentWillMount/ReceiveProps/Update` phải đổi thành `UNSAFE_*` hoặc viết lại. String refs (`ref="x"`) bỏ, dùng `createRef`/callback ref. `findNodeHandle` hạn chế.
- **Đã bị gỡ khỏi core RN**, phải import từ package riêng: `AsyncStorage` → `@react-native-async-storage/async-storage`; `NetInfo` → `@react-native-community/netinfo`; `Picker`, `Slider`, `DatePickerIOS/Android`, `ViewPagerAndroid`, `ProgressBarAndroid`, `Clipboard`, `CameraRoll`… dùng các package `@react-native-community/*` hoặc `@react-native-picker/picker` tương ứng. `ViewPropTypes` và `PropTypes` của RN bị bỏ, dùng `prop-types` hoặc `deprecated-react-native-prop-types`.
- **Babel/Metro**: `metro-react-native-babel-preset` đổi thành `@react-native/babel-preset` (đã cấu hình sẵn). Thư viện cũ có `.babelrc` riêng có thể gây lỗi.
- **Android 13/14+**: quyền `POST_NOTIFICATIONS`, `READ_MEDIA_IMAGES` thay `READ_EXTERNAL_STORAGE`; foreground service cần khai báo type; `android:exported` bắt buộc cho activity/receiver có intent-filter.
- **redux-saga 0.16 → 1.x**: `delay` import từ `redux-saga/effects`; `takeEvery/takeLatest` chỉ import từ `redux-saga/effects`.

## Bảng thư viện (cũ → đề xuất)

Phải xác minh lại version tương thích RN 0.75 trước khi cài (`yarn info <pkg>`, changelog).

| Thư viện cũ | Đề xuất |
|---|---|
| `react-native-navigation` ^6.7 (Wix) | Nâng lên Wix RNN 7.x bản hỗ trợ RN 0.75 để giữ `src/app-navigation`. Nếu gặp vướng thì chuyển sang React Navigation 6/7 (phải viết lại lớp navigation) |
| `react-native-code-push` ^6.2 | App Center đã ngừng (31/03/2025). **Cần chủ dự án quyết định**: tự host CodePush server, chuyển giải pháp OTA khác, hoặc bỏ |
| `redux-persist` 4.10.2 | `redux-persist` 6 + `@react-native-async-storage/async-storage` |
| `redux-saga` ^0.16 | `redux-saga` 1.x |
| `react-redux` 7 / `redux` 4 | Giữ nguyên hoặc nâng `react-redux` 8/9 |
| `react-native-elements` ^1.1 | `@rneui/themed` + `@rneui/base` 4.x |
| `react-native-paper` 3.9 | `react-native-paper` 5.x (API đổi theo MD3) |
| `react-native-vector-icons` ^6 | Bản mới nhất. Android cần `apply from: fonts.gradle` |
| `react-native-maps` ^0.25 | `react-native-maps` 1.x (giữ Google Maps API key) |
| `react-native-geolocation-service` ^3 | `react-native-geolocation-service` 5.3.x |
| `react-native-geocoder` | `react-native-geocoder-reborn` hoặc gọi Geocoding API qua axios |
| `react-native-image-picker` ^2.3 | `react-native-image-picker` 7.x (API đổi hẳn: `launchCamera`/`launchImageLibrary` trả Promise) |
| `react-native-image-view` | `react-native-image-viewing` |
| `react-native-touch-id` | `react-native-biometrics` |
| `react-native-device-info` 5.5.5 | Bản mới nhất (nhiều hàm chuyển sync/async, cần kiểm tra từng chỗ gọi) |
| `react-native-datepicker` | `@react-native-community/datetimepicker` hoặc `react-native-date-picker` |
| `react-native-material-dropdown` | `react-native-element-dropdown` |
| `react-native-safe-area-context` ^0.7 | 4.x |
| `react-native-linear-gradient` 2.6.2 | 2.8.x |
| `react-native-webview` ^11 | 13.x |
| `react-native-snackbar`, `react-native-calendars`, `react-native-floating-action`, `react-native-swiper` | Nâng bản mới nhất, kiểm tra hoạt động |
| `react-native-progress-bar-animated`, `react-native-xml2js` | Kiểm tra. Nếu hỏng thì thay (`react-native-progress` / `fast-xml-parser`) |
| `intl` | Hermes đã hỗ trợ `Intl`, cân nhắc bỏ |
| `axios`, `moment`, `ramda`, `hoist-non-react-statics` | Giữ nguyên (JS thuần) |
| `jetifier`, `add`, `yarn`, `react-native-rename` | **Bỏ** (không còn cần hoặc cài nhầm) |

## Quy ước làm việc

- Commit nhỏ, mỗi commit một bước port hoặc một thư viện. Chỉ commit/push khi được yêu cầu.
- Sau mỗi thay đổi lớn: `yarn lint` và build thử Android (`yarn android` hoặc `./gradlew assembleDebug`).
- Khi gặp lỗi build native, ưu tiên xem `android/build.gradle`, `gradle.properties` và phiên bản thư viện trước khi patch `node_modules`. Nếu bắt buộc patch thì dùng `yarn patch` (lưu ở `.yarn/patches`).
