# flutter-runner
این یک سری دستور و راه حل برای اجرای کدهای فلاتر با استفاده از github actions و خروجی گرفتنه. این خروجی امضا میشه و قابل انتشاره <br />
اگر لازم داشتید که یک کد فلاتر رو خروجی بگیرید ولی مشکل اینترنت داشتید، یا سیستمتون ضعیف بود و خروجی طول میکشید، یا میخواستید با آیپی خودتون خروجی نگیرید میتونید از این روش استفاده کنید <br />
اول باید یک فایل keystore برای امضای نرم افزار بسازید <br />
توی ویندوز میتونید از این دستور استفاده کنید <br />
```
keytool -genkeypair -v -keystore output.keystore -keyalg RSA -keysize 2048 -validity 10000 -alias YOURALIAS
```
بعدش باید فایل android/app/build.gradle رو به شکل زیر ویرایش کنید <br />
```
signingConfigs {
        release {
            keyAlias System.getenv("KEY_ALIAS")
            keyPassword System.getenv("KEY_PASSWORD")
            storeFile file(System.getenv("KEYSTORE_PATH"))
            storePassword System.getenv("KEYSTORE_PASSWORD")

        }
    }
    buildTypes {
        release {
            signingConfig signingConfigs.release
        }
    }
```
باید چهارتا مقدار KEY_ALIAS , KEY_PASSWORD , KEYSTORE_PATH , KEYSTORE_PASSWORD رو توی گیت هاب اضافه کنید. تنها مورد خاصی که وجود داره اینه که باید فایل keystore ای که ساختید رو تبدیل کنید به base64 
<br />
برای این کار توی ویندوز، powershell رو به شکل ادمین باز کنید و دستور زیر رو توی مسیر فایل keystore بزنید <br />
```
[convert]::ToBase64String((Get-Content -Path "file.keystore" -Encoding byte)) > keystore.b64
```
حالا فایل خروجی رو به عنوان تکست باز کنید و مقدارش رو کپی کنید <br />
بعدش برید توی تنظیمات گیت هاب و بخش settings > Secrets and variables > Actions و چهارتا secret اضافه کنید <br />
```
KEY_ALIAS که موقع ساخت امضا وارد کردید
KEY_PASSWORD که موقع ساخت امضا وارد کردید
KEYSTORE_PASSWORD که موقع ساخت امضا وارد کردید
KEYSTORE فایلی که به base64 تبدیل کردید و کپی کردید
```
حالا به بخش actions برید و تمپلیت دارت رو انتخاب کنید
مرحله بعد یک فایل داره که داخلش میتونید این کدها رو جایگزین کنید:
```

name: runner

on:
  workflow_dispatch:
    inputs:
      tag_name:
        description: 'version'
        required: true
        default: 'v1.0.0'
      branch:
        description: 'Branch to build'
        required: true
        default: 'main'
      output_name:
        description: 'Name for the output file'
        required: true
        default: 'release'
      

jobs:
  build:
    runs-on: ubuntu-latest

    steps:

      - name: Set Run Name
        run: echo "Run name set to ${{ github.event.inputs.tag_name }}"
        if: always()
        env:
          TAG_NAME: ${{ github.event.inputs.tag_name }}
    
      - uses: actions/checkout@v4
      - uses: dart-lang/setup-dart@9a04e6d73cca37bd455e0608d7e5092f881fd603

      - name: Checkout repository
        uses: actions/checkout@v2
        with:
          ref: ${{ github.event.inputs.branch }}

      - name: Set up Flutter
        uses: subosito/flutter-action@v2
        with:
          flutter-version: '3.13.0'

      - name: Cache Flutter SDK
        id: cache-flutter
        uses: actions/cache@v3
        with:
          path: ~/.pub-cache
          key: flutter-sdk-${{ runner.os }}-${{ hashFiles('.github/workflows/flutter-build.yml') }}
          restore-keys: |
            flutter-sdk-${{ runner.os }}-

      - name: Cache Pub Dependencies
        id: cache-pub
        uses: actions/cache@v3
        with:
          path: |
            ~/.pub-cache
            .packages
            .dart_tool
          key: pub-cache-${{ runner.os }}-${{ hashFiles('pubspec.yaml') }}
          restore-keys: |
            pub-cache-${{ runner.os }}-

      - name: Decode keystore
        run: echo "$KEYSTORE" | base64 --decode > /tmp/keystore.jks
        env:
          KEYSTORE: ${{ secrets.KEYSTORE }}

      - name: Verify keystore file
        run: ls -l /tmp/keystore.jks

      - name: Set up environment for signing
        run: |
          echo "KEYSTORE_PATH=/tmp/keystore.jks" >> $GITHUB_ENV
          echo "KEYSTORE_PASSWORD=${{ secrets.KEYSTORE_PASSWORD }}" >> $GITHUB_ENV
          echo "KEY_ALIAS=${{ secrets.KEY_ALIAS }}" >> $GITHUB_ENV
          echo "KEY_PASSWORD=${{ secrets.KEY_PASSWORD }}" >> $GITHUB_ENV

      - name: Debug environment variables
        run: |
          echo "KEYSTORE_PATH=$KEYSTORE_PATH"
          echo "KEYSTORE_PASSWORD=$KEYSTORE_PASSWORD"
          echo "KEY_ALIAS=$KEY_ALIAS"
          echo "KEY_PASSWORD=$KEY_PASSWORD"
        env:
          KEYSTORE_PATH: /tmp/keystore.jks
          KEYSTORE_PASSWORD: ${{ secrets.KEYSTORE_PASSWORD }}
          KEY_ALIAS: ${{ secrets.KEY_ALIAS }}
          KEY_PASSWORD: ${{ secrets.KEY_PASSWORD }}

          
          
      - name: Install dependencies
        run: flutter pub get

      - name: Build APK
        run: flutter build apk --release

      - name: Build App Bundle
        run: flutter build appbundle --release

      # - name: Build Web
      #   run: flutter build web --release --web-renderer canvaskit --no-tree-shake-icons

      - name: Rename and Move APK
        run: |
          mv build/app/outputs/flutter-apk/app-release.apk build/app/outputs/flutter-apk/${{ github.event.inputs.output_name }}-${{ github.event.inputs.tag_name }}.apk

      - name: Rename and Move App Bundle
        run: |
          mv build/app/outputs/bundle/release/app-release.aab build/app/outputs/bundle/release/${{ github.event.inputs.output_name }}-${{ github.event.inputs.tag_name }}.aab

      # - name: Rename and Move Web Build
      #   run: |
      #     mv build/web build/${{ github.event.inputs.output_name }}-${{ github.event.inputs.tag_name }}

      # - name: Create Web Zip
      #   run: |
      #     cd build
      #     zip -r ${{ github.event.inputs.output_name }}-${{ github.event.inputs.tag_name }}.zip ${{ github.event.inputs.output_name }}-${{ github.event.inputs.tag_name }}

      - name: Upload APK
        uses: actions/upload-artifact@v3
        with:
          name: ${{ github.event.inputs.output_name }}-${{ github.event.inputs.tag_name }}-apk
          path: build/app/outputs/flutter-apk/${{ github.event.inputs.output_name }}-${{ github.event.inputs.tag_name }}.apk

      - name: Upload App Bundle
        uses: actions/upload-artifact@v3
        with:
          name: ${{ github.event.inputs.output_name }}-${{ github.event.inputs.tag_name }}-aab
          path: build/app/outputs/bundle/release/${{ github.event.inputs.output_name }}-${{ github.event.inputs.tag_name }}.aab

      # - name: Upload Web Build Zip
      #   uses: actions/upload-artifact@v3
      #   with:
      #     name: ${{ github.event.inputs.output_name }}-${{ github.event.inputs.tag_name }}-web-zip
      #     path: build/${{ github.event.inputs.output_name }}-${{ github.event.inputs.tag_name }}.zip

```

حالا میتونید به بخش actions برید و این رو اجرا کنید <br />
موقع اجرا ازتون 3 تا چیز میپرسه<br />
اسم برنچی که باید کدهاش خروجی گرفته بشه <br />
اسم فایل خروجی <br />
ورژنی که کنار اسم فایل باید بنویسه. این ورژن نرم افزار نیست. فقط اسم فایل خروجیه <br />
بخش مربوط به خروجی وب رو کامنت کردم میتونید از کامنت دربیارید اگر خروجی وب رو هم میخواید
