# Android 多语言

一般在应用内通过如下方式设置显示的语言类型：

```
public void setLanguage(Resources resources){
        Locale targtLocal = Locale.CHINA;
        if(/**判断选中的是否为美式英文*/){
            targtLocal = Locale.US;
        }

        Configuration configuration = resources.getConfiguration();
        DisplayMetrics displayMetrics = resources.getDisplayMetrics();
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.JELLY_BEAN_MR1) {
            configuration.setLocale(targtLocal);
        } else {
            //noinspection deprecation
            configuration.locale = targtLocal;
        }
        resources.updateConfiguration(configuration,displayMetrics);
    }
```

设置完成后，一般还需要通知相应的页面，根据新语言类型来进行展示。