# شروع پروژه Genius Air

این بسته شامل سورس Ionic با ارتباط واقعی و firmware متناظر ESP-12E است. APK/IPA یا Gerber نیست و هنوز روی برد واقعی آزموده نشده است.

برای ساخت APK روی سرور GitHub، راهنمای `ionic/docs/GITHUB_BUILD.fa.md` و گردش‌کار `.github/workflows/ionic-android.yml` را استفاده کنید. اتصال حساب باید به مخزن پروژه هم دسترسی داشته باشد؛ ساخت موفق هنوز گزارش نشده است.

از راهنمای تصویری `ionic/docs/Genius_Air-build-guide.html` شروع کنید؛ متن فرمان‌ها قابل کپی است. نسخه PDF کنار آن، ۱۲ صفحه نمودار آموزشی دارد. برای ویندوز گام‌های ۱ تا ۷ و برای Mac/iPhone گام‌های ۹ تا ۱۱ هستند؛ اسکریپت اختیاری ویندوز در گام ۸ آمده است.

1. اجرای ویندوز و ساخت اپ: `ionic/README.fa.md`.
2. ساخت برنامه ESP و آماده‌سازی اولیه حافظه: `firmware/esp12e/README.md`.
3. نگاشت پایه‌ها و گیت سخت‌افزاری: `hardware/rev_b/PINOUT.fa.md`.
4. وضعیت دقیق آزمون‌ها و محدودیت ساخت: `REAL_BUILD_STATUS.fa.md`.
5. قرارداد مشترک ارتباط: `DEVICE_PROTOCOL.md`.

فایل‌ها را در `C:\Genius_Air` استخراج کنید؛ سپس از PowerShell:

```powershell
cd C:\Genius_Air\ionic
npm install
npm run dev
```

Node.js LTS لازم است. بدون برد ساخته و برنامه‌ریزی‌شده، رابط باز می‌شود ولی فرمان‌ها فعال نمی‌شوند.
